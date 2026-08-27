# Building a StrongDM Proxy Cluster with Ansible

An example Ansible playbook that provisions EC2 instances and installs
**StrongDM proxy cluster workers natively** — the `sdm` binary managed by
systemd, no Docker, no Kubernetes.

The cluster's workers come from a list in [`workers.yml`](workers.yml). Add an
entry, re-run, and the new worker joins the existing cluster. That works because
the playbook encrypts the cluster's key pair on the first run and reads it back
on every run after — see [Credential persistence](#credential-persistence),
which is the one genuinely non-obvious part of this repo.

StrongDM requires a network load balancer for any cluster with more than one
worker, so the playbook builds one for you by default and uses its AWS-supplied
hostname as the cluster address. Nothing to register in DNS. If you already have
a load balancer, [`manual` mode](#22-choose-how-clients-reach-the-cluster) skips
that and prints what to register with yours.

Everything runs from a disposable EC2 control node, so nothing needs to be
installed on your laptop and the whole exercise is reproducible from scratch.

---

## Terminology, because it matters here

StrongDM has two proxy deployment models and they are not interchangeable:

| | Proxy cluster worker | Gateway / relay |
|---|---|---|
| Systemd unit | `sdm-worker` | `sdm-proxy` |
| Env file | `/etc/sysconfig/sdm-worker` | `/etc/sysconfig/sdm-proxy` |
| Install flag | `sdm install --worker` | `sdm install --node` |
| Credential | access key + secret key | JWT token |
| Default port | 8443 behind an LB, 443 direct | 5000 |
| Status | Recommended deployment model | Older "active networking" model |

This playbook builds the **proxy cluster worker**. Both come out of the same
`sdm` binary, which is why the flags are the only thing that distinguishes them.

---

## What gets built

```
                    ┌─────────────────────────┐
                    │  StrongDM control plane │
                    │    app.strongdm.com     │
                    └────────────▲────────────┘
                                 │ 443 (outbound from every node)
         ┌───────────────────────┼───────────────────────┐
         │                       │                       │
┌────────┴──────────┐  ┌─────────┴─────────┐  ┌──────────┴────────┐
│  Control node     │  │ sdm-proxy-worker- │  │ sdm-proxy-worker- │
│  (EC2, AL2023)    │  │ 01  (us-east-2a)  │  │ 02  (us-east-2b)  │
│                   │  │                   │  │                   │
│  ansible-core     │  │  sdm binary       │  │  sdm binary       │
│  sdm CLI          ├─►│  systemd:         │  │  systemd:         │
│  boto3            │ssh  sdm-worker       │  │  sdm-worker       │
│                   │22│  :8443  :9090     │  │  :8443  :9090     │
└───────────────────┘  └────▲───────▲──────┘  └────▲───────▲──────┘
                            │       │              │       │
                    traffic │  live │      traffic │  live │
                      8443  │  9090 │        8443  │  9090 │
                            └───┬───┴──────────────┴───┬───┘
                                │                      │
                        ┌───────┴──────────────────────┴───────┐
                        │   Network Load Balancer (TCP :443)   │
                        │   target group :8443                 │
                        │   health check  :9090/liveness → 200 │
                        └──────────────────▲───────────────────┘
                                           │ 443
                        dc-ansible-cluster-nlb-<hash>
                              .elb.us-east-2.amazonaws.com
                                           ▲
                                           │
                                   StrongDM clients
```

Resources created in AWS, tagged `Project=strongdm-ansible-demo` and
`Cluster=<sdm_cluster_name>`:

- one Amazon Linux 2023 instance **per entry in `workers.yml`** — `t3.small`
  with an 8 GB root volume by default, sized for a demo (see
  [Sizing](#sizing) if you're adapting this for real use)
- an internet-facing **network load balancer** and a TCP target group, in
  `nlb` mode only
- one security group shared by the whole cluster: `8443` from your client
  CIDRs, `9090` from the subnet CIDRs for health checks, `22` from the control
  node
- one Elastic IP per worker
- an EC2 keypair, imported from a key generated on the control node

Created in StrongDM:

- a proxy cluster named `dc-ansible-cluster` advertising the load balancer's
  hostname on port 443
- one authentication key pair for that cluster, shared by every worker in it,
  encrypted to `credentials/dc-ansible-cluster.yml`

Three things about that diagram are load-bearing rather than incidental:

- **The listener is TCP, not TLS.** Proxy workers establish mutual TLS directly
  with clients and validate client certificates themselves. Anything that
  terminates TLS on their behalf breaks the handshake — which is also why an
  ALB cannot front a proxy cluster at all.
- **The health check hits `:9090/liveness`, not the traffic port.** During a
  maintenance-window rolling upgrade a worker keeps serving traffic on 8443 but
  deliberately shuts `9090` down; that is how it asks the load balancer to
  drain it. Health-checking 8443 would keep feeding new connections to a worker
  that is about to sever them.
- **Client IP preservation is on.** Without it the workers see the load
  balancer as the client, and per-client audit data — the reason StrongDM is in
  the path — becomes worthless.

---

## Part 1 — Stand up the control node

> ### Where do I type this?
>
> **Steps 1.1 – 1.3 happen in the AWS console, in your browser.** No terminal.
>
> **Steps 1.4 onward happen inside the EC2 instance**, in the Session Manager
> shell that opens in a browser tab. *Nothing in this guide runs on your laptop.*
> If you find yourself typing `sudo` into Terminal on your Mac, you're in the
> wrong place — you'll get `su: unknown login: ec2-user`, because `ec2-user`
> exists on Amazon Linux, not on macOS.
>
> How to tell you're on the right box:
>
> ```bash
> hostname      # ip-10-0-1-42       (not SDM-XXXXXX-LT)
> uname -s      # Linux              (not Darwin)
> ```
>
> **The fastest tell is the error prefix.** macOS uses zsh, Amazon Linux uses
> bash, so a missing command names the shell that failed:
>
> | Error you see | Where you are |
> |---|---|
> | `zsh: command not found: ansible-vault` | Your Mac — wrong tab |
> | `bash: ansible-vault: command not found` | The instance — real problem |
>
> On the instance, that second one usually just means the venv isn't active:
> `source ~/.venv/ansible/bin/activate`
>
> That separation is deliberate. The control node exists precisely so you don't
> need admin rights, Homebrew, or a security exception on a managed laptop.

Three things in order, because each one is selected by the next: policy → role →
instance.

### 1.1 Create the customer-managed policy

IAM → **Policies** → **Create policy** → **JSON** tab. Paste the contents of
[`controlnode/iam-policy.json`](controlnode/iam-policy.json) and name it
`ansible-strongdm-demo`.

This grants the permissions the playbook needs: an unconditional
`ec2:Describe*` for reads, plus `ec2:*` and `elasticloadbalancing:*` for writes
**fenced to a single region** by an `aws:RequestedRegion` condition — currently
`us-east-2`. There's also a narrowly scoped `iam:CreateServiceLinkedRole`, which
AWS requires the first time an account creates a load balancer.

**That condition must match `aws_region` in `group_vars/all.yml`**, or every EC2
call returns `UnauthorizedOperation`.

> **Why `ec2:*` and not an enumerated list?** An earlier version of this policy
> listed individual actions. It kept failing one action at a time — the
> `amazon.aws` modules make calls that aren't obvious from the task names, like
> `DescribeVpcAttribute` when looking up a VPC or
> `UpdateSecurityGroupRuleDescriptionsEgress` when a rule carries a description.
> For a sandbox demo, a region fence is the useful boundary; enumerating actions
> just moves the failure later. If you need tight scoping, generate the real list
> from CloudTrail after one successful run rather than guessing — that's the only
> way to get it right.

The region fence is doing real work here: this role cannot touch EC2 anywhere
outside `us-east-2`, so a mistake stays contained to your sandbox region.

> Create the policy as its own object rather than inline on the role. The Create
> Role wizard can only *attach* policies that already exist — if you start with
> the role you end up bouncing out to a second tab and refreshing the list.

### 1.2 Create the role

IAM → **Roles** → **Create role** → trusted entity **AWS service** → **EC2**.
Attach two policies:

1. **`AmazonSSMManagedInstanceCore`** (AWS managed) — this is what makes Session
   Manager work.
2. **`ansible-strongdm-demo`** — the policy you just created in 1.1.

Name the role `ansible-control-node`. The console creates the matching instance
profile automatically when the role's trusted entity is EC2.

Because the control node carries this role, boto3 picks up credentials
automatically. **No access keys anywhere.**

### 1.3 Launch the instance

In the AWS console, launch an instance in your sandbox account:

| Setting | Value |
|---|---|
| Name | `ansible-control-node` |
| AMI | Amazon Linux 2023 |
| Instance type | `t3.small` (plenty for a control node) |
| Key pair | **Proceed without a key pair** — you'll use SSM |
| Subnet | Same VPC you want the worker in |
| Auto-assign public IP | Enable (needed to reach Galaxy and StrongDM) |
| IAM instance profile | `ansible-control-node` (from 1.2) |

The control node needs no inbound rules at all. SSM works entirely outbound.

### 1.4 Connect

AWS console → EC2 → select the instance → **Connect** → **Session Manager** → **Connect**.

A shell opens **in the browser tab**. Nothing installed locally, no SSH key, no
inbound port 22, no corporate approval prompts.

Everything from here to the end of Part 3 is typed into *that* shell:

```bash
# Sanity check: am I actually on the instance?
hostname && uname -s        # expect ip-10-x-x-x and Linux

# Session Manager drops you in as ssm-user, sitting in /usr/bin.
# Switch to a real login shell before doing anything else.
sudo su - ec2-user

# Confirm it worked
pwd && id -un               # expect /home/ec2-user and ec2-user
```

> **Use `su -`, with the dash.** Without it you keep Session Manager's starting
> directory of `/usr/bin`, which `ssm-user` cannot write to — the first `curl -O`
> in the next step then fails with `Permission denied`, and every command after
> it fails in confusing ways. The dash is what gives you a login shell in
> `/home/ec2-user`.

> The `ec2-user` switch is not cosmetic either. StrongDM's installer requires a
> user that exists in `/etc/passwd`, and running the playbook as a real user
> keeps the SSH keys and Galaxy collections somewhere predictable.

### 1.5 Install the toolchain

```bash
cd ~     # never run the download from /usr/bin — see the warning in 1.4

# Build deps and Python
sudo dnf install -y git python3.11 python3.11-pip unzip

# Ansible + AWS SDK in a venv, so nothing fights the system Python
python3.11 -m venv ~/.venv/ansible
source ~/.venv/ansible/bin/activate
echo 'source ~/.venv/ansible/bin/activate' >> ~/.bashrc

pip install --upgrade pip
pip install ansible-core boto3 botocore ansible-lint yamllint
```

The `sdm` CLI is a single static binary. Download it into a scratch directory so
a failed download can't leave junk in your home:

```bash
mkdir -p ~/sdm-dl && cd ~/sdm-dl

curl -fJOL https://app.strongdm.com/releases/cli/linux
ls -l                      # expect sdmcli_<version>_linux_amd64.zip

unzip -o sdmcli_*_linux_amd64.zip
sudo install -m 0755 sdm /usr/local/bin/sdm

cd ~ && rm -rf ~/sdm-dl
```

`-f` makes curl exit non-zero on an HTTP error instead of silently writing an
error page to disk, so the `ls` either shows you a real zip or you stop there.

```bash
# Verify all three
ansible --version && sdm --version && aws sts get-caller-identity
```

That last command should show the instance-profile role ARN. If it errors,
the instance profile isn't attached — fix that before continuing.

### 1.6 Get the playbook onto the box

```bash
cd ~
git clone https://github.com/davidchase-prog/strongdm-proxy-worker.git
cd strongdm-proxy-worker
ansible-galaxy collection install -r requirements.yml
```

The repo is public, so the clone needs no credentials on the instance — nothing
to paste, nothing to leave behind when you terminate it.

---

## Part 2 — Configure

### 2.1 StrongDM admin token

Admin UI → **Settings → Admin Tokens** → create a token.

On the control node, start from the example file:

```bash
cp vault.yml.example vault.yml
```

Set the token without it landing in your shell history — `read -rs` doesn't echo,
and the token never appears as a command argument:

```bash
read -rs -p "StrongDM admin token: " SDM_TOKEN && echo
sed -i "s|^sdm_admin_token:.*|sdm_admin_token: \"$SDM_TOKEN\"|" vault.yml && unset SDM_TOKEN

grep -q '^sdm_admin_token: "REPLACE_ME"' vault.yml && echo "NOT SET" || echo "token set"
```

The `^sdm_admin_token:` anchor matters. A plain `s/REPLACE_ME/…/g` would also
match the commented-out `sdm_proxy_cluster_secret_key` line and leave your token
sitting in a comment. (`vi vault.yml` works too, if you prefer an editor.)

Then create the vault password and encrypt:

```bash
openssl rand -base64 32 > ~/.ansible-vault-pass
chmod 600 ~/.ansible-vault-pass

ansible-vault encrypt vault.yml
```

No password prompt — `ansible.cfg` already points at `~/.ansible-vault-pass`.

### 2.2 Choose how clients reach the cluster

[`workers.yml`](workers.yml) is the file you'll actually edit day to day:

```yaml
sdm_cluster_name: dc-ansible-cluster
sdm_cluster_endpoint: nlb        # nlb | manual
sdm_cluster_hostname: ""         # required in manual mode only
sdm_cluster_port: 443            # what clients connect to
sdm_worker_bind_port: 8443       # what the LB forwards to

sdm_workers:
  - name: sdm-proxy-worker-01
  - name: sdm-proxy-worker-02
```

| Mode | What happens | Use it when |
|---|---|---|
| `nlb` | Playbook creates an internet-facing NLB and uses its `*.elb.amazonaws.com` hostname as the cluster address | Demos, and any AWS-native deployment. **No DNS setup at all** |
| `manual` | No load balancer created. You set `sdm_cluster_hostname` to a name pointing at *your* LB; the playbook prints what to register with it | You already run a load balancer — the usual client-site shape |

The cluster's address is **fixed at creation and cannot be changed**, which is
why `manual` mode hard-fails on an empty or placeholder hostname rather than
guessing.

> **`manual` does not mean round-robin A records pointing at the workers.** That
> looks like it should work, and StrongDM doesn't support it: a worker draining
> for a rolling upgrade stays in DNS rotation, so clients keep landing on a node
> that's shutting down. The playbook warns if you configure more than one worker
> in `manual` mode, because the only correct answer there is a real load
> balancer.

Per-worker keys, all optional except `name`:

| Key | Default | Notes |
|---|---|---|
| `name` | — | Required. EC2 `Name` tag and the instance hostname. Unique per cluster. |
| `instance_type` | `instance_type` from `group_vars` | Honoured at creation only |
| `root_volume_gb` | `root_volume_gb` from `group_vars` | Honoured at creation only |
| `subnet_id` | round-robin across discovered subnets | Set it to pin a worker to one subnet |

Left alone, workers are distributed round-robin across the subnets the playbook
discovers — which spreads them over availability zones, the reason to run more
than one in the first place.

### 2.3 Review the rest of the variables

Everything else lives in [`group_vars/all.yml`](group_vars/all.yml). The ones
you're most likely to touch:

| Variable | Default | Notes |
|---|---|---|
| `aws_region` | `us-east-2` | Must match the IAM policy condition (1.1) |
| `sdm_app_domain` | `app.strongdm.com` | Change for a UK or EU **control plane** |
| `instance_type` | `t3.small` | Default for workers that don't override it |
| `client_ingress_cidrs` | `0.0.0.0/0` | Narrow it if your account has guardrails. Client IP is preserved through the NLB, so these are real client addresses |
| `vpc_id` / `subnet_id` | empty | Empty means default VPC, auto-discovered. Setting `subnet_id` pins **every** worker to one subnet — which breaks `nlb` mode, since an NLB needs two AZs |
| `sdm_credentials_prompt` | `true` | Set `false` for unattended runs — see [Credential persistence](#credential-persistence) |
| `worker_assign_eip` | `true` | Set `false` only if the subnets already give workers egress via NAT or auto-assign |
| `nlb_cross_zone` | `true` | Off is the AWS default and splits traffic per-AZ, not per-worker |
| `nlb_health_check_protocol` | `HTTP` | Change to `TCP` only as a fallback — you lose graceful draining |

---

## Part 3 — Run it

```bash
# Dry run first
ansible-playbook site.yml --check --diff

# Real run
ansible-playbook site.yml
```

Expect roughly five to seven minutes for two workers. The workers are built in
parallel so a third adds little; most of the time is the load balancer
provisioning and then the target group's first health checks passing.

Per-worker success looks like:

```
TASK [Report this worker]
ok: [sdm-proxy-worker-01] =>
  msg: 'sdm-proxy-worker-01 (us-east-2a): sdm-worker active, traffic HTTP 404 (404 = healthy), liveness HTTP 200'
ok: [sdm-proxy-worker-02] =>
  msg: 'sdm-proxy-worker-02 (us-east-2b): sdm-worker active, traffic HTTP 404 (404 = healthy), liveness HTTP 200'
```

**HTTP 404 is the healthy state on the traffic port.** A proxy worker is a TLS
proxy, not a web server — 404 on plain HTTPS means it's up and listening.
StrongDM's own docs use this as the verification step. The liveness endpoint is
ordinary HTTP and answers 200.

Both are probed on the worker itself before the load balancer is asked to depend
on them, because a target group whose health check never passes is a slow and
confusing failure.

Then the workers are registered and the run waits for the load balancer to call
them healthy — the real end-to-end proof that the worker is up, the security
group lets the health check through, and traffic will actually be forwarded:

```
TASK [Register the workers with the load balancer]
changed: [localhost] => (item=sdm-proxy-worker-01)
changed: [localhost] => (item=sdm-proxy-worker-02)

TASK [Report the finished cluster]
  - 'Cluster:  dc-ansible-cluster'
  - 'Address:  dc-ansible-cluster-nlb-6a4a71090d9df84a.elb.us-east-2.amazonaws.com:443'
  - 'Workers:  2 in us-east-2a, us-east-2b'
```

That address is the cluster's, permanently. Nothing to register in DNS.

### Verify independently

```bash
# The cluster address resolves (AWS publishes it; nothing for you to do)
dig +short dc-ansible-cluster-nlb-6a4a71090d9df84a.elb.us-east-2.amazonaws.com

# Through the load balancer — 404 means a worker answered
curl -k https://dc-ansible-cluster-nlb-6a4a71090d9df84a.elb.us-east-2.amazonaws.com

# Target health, straight from AWS
aws elbv2 describe-target-health \
  --target-group-arn "$(aws elbv2 describe-target-groups \
      --names dc-ansible-cluster-tg \
      --query 'TargetGroups[0].TargetGroupArn' --output text)" \
  --query 'TargetHealthDescriptions[].{id:Target.Id,state:TargetHealth.State}'

# Across the whole fleet
ansible sdm_workers -m command -a 'systemctl is-active sdm-worker' -b
```

And in the Admin UI under **Networking → Proxy Clusters**, the cluster should
show every worker connected.

### Rebuilding in nlb mode

The NLB's hostname embeds a generated hash, so **destroying and recreating the
load balancer produces a different address** — and a proxy cluster's address
can't be changed. A rebuilt NLB in front of an existing cluster means every
client is still dialling an address that no longer exists.

The playbook stores the address alongside the credentials and warns on a
mismatch, but it can't fix it. Practical consequences:

- `teardown.yml` deletes the cluster *and* the NLB together, so a full
  teardown-and-rebuild is clean — you get a new cluster and new credentials.
- Deleting only the NLB and re-running is **not** recoverable. Delete the
  cluster too (and its credentials file), or put a CNAME you control in front of
  the NLB from the start and use `manual` mode with that name.

For anything long-lived, `manual` mode with your own DNS name pointed at the NLB
is the better shape — that's the layer of indirection this trade-off is asking
for.

---

## Part 4 — Use it

The cluster proxies nothing until resources are attached to it:

- **Admin UI**: edit a resource, set its **Proxy Cluster** field
- **CLI**: pass `--proxy-cluster-id plc-...` (find the id with `sdm admin nodes list`)

Resources attach to the *cluster*, not to a worker, so this is a one-time step
no matter how many workers you run.

**Every worker in one cluster must be able to reach the same set of resources.**
This is the constraint that decides your topology: workers in a cluster are
interchangeable, and a client landing on any of them must get the same answer.
Different network segments need different clusters, not more workers.

---

## Part 5 — Clean up

```bash
ansible-playbook teardown.yml
```

Scoped by the `Cluster` tag, so it only removes the cluster named in
`workers.yml`: the StrongDM cluster, the load balancer and its target group,
every worker instance, their EIPs, the security group, and the keypair. Then
terminate the control node in the console.

The load balancer goes first, deliberately — an ENI it still holds would block
the security group delete, and the target group can't go while a listener
references it.

The encrypted credentials file is **kept** by default — once the cluster is
deleted its key pair is worthless, but deleting the wrong file is
unrecoverable, so removing it is opt-in:

```bash
ansible-playbook teardown.yml -e delete_credentials=true
```

Skip the confirmation prompt in CI with `-e auto_approve=true`.

---

## Scaling the cluster

In `nlb` mode, adding a worker is two steps:

```bash
# 1. append to workers.yml
#      - name: sdm-proxy-worker-03

# 2. re-run
ansible-playbook site.yml
```

That's the whole point of fronting the fleet with a load balancer: the cluster
address doesn't move, so there's no third step. The new worker is provisioned,
installed with the stored credentials, registered as a target, and the run waits
for it to report healthy before finishing. In `manual` mode there *is* a third
step — register the new worker with your load balancer, using the addresses the
run prints.

Runs are **add-only**. Anything already provisioned comes back unchanged;
`ec2_instance` matches on the `Name` tag, so idempotence is a property of the
module rather than something the playbook has to track. Removing an entry from
`workers.yml` destroys nothing — that's `teardown.yml`'s job.

Two consequences worth knowing:

- **`instance_type` and `root_volume_gb` are honoured at creation only.**
  Changing them for an existing worker does nothing. Resize in the console, or
  tear that worker down and let the playbook rebuild it.
- **The new worker joins the existing cluster** rather than forming a new one,
  because the run reads the stored key pair instead of calling
  `create-proxy-cluster`. If the credentials file is missing, that's the one
  situation the playbook can't recover from on its own — read on.

---

## Credential persistence

This is the constraint everything else in this repo bends around:

> A proxy cluster's access key and secret key are displayed **exactly once**, at
> creation. There is no CLI command to read them back, and no CLI command to
> mint a replacement for an existing cluster.

Every worker in a cluster authenticates with that same pair. So if the pair is
lost, adding a worker means either minting a new key in the Admin UI by hand or
rebuilding the cluster — and rebuilding the cluster means a new advertised
address, new DNS, and re-pointing every resource attached to it.

The playbook therefore treats the key pair as the durable artifact of a run, not
a transient value:

1. On the first run it creates the cluster and parses the pair out of the CLI
   output.
2. It writes the pair to `credentials/<cluster>.yml` and encrypts it in place
   with `ansible-vault`, using the same `~/.ansible-vault-pass` that
   `ansible.cfg` already points at. Nothing extra to manage.
3. This happens **before any instance is launched**, so a failure later in the
   run leaves the credentials intact and the next run is a resume, not a
   restart.
4. Every subsequent run loads that file via `include_vars` — which decrypts
   vault files transparently — and skips cluster creation entirely.

Precedence, highest first:

| Source | When |
|---|---|
| `sdm_proxy_cluster_*` in `vault.yml` | You supplied a pair explicitly — see below |
| `credentials/<cluster>.yml` | A previous run stored it |
| `create-proxy-cluster` | First run for this cluster |
| Interactive prompt | The above failed and `sdm_credentials_prompt` is `true` |

### Back it up

`credentials/` is in `.gitignore` as the conservative default. That's a real
tradeoff, not an obvious win: the file is vault-encrypted, so committing it is
safe — and it's the only copy of something StrongDM will never reissue.
Committing it means a teammate can add a worker to the cluster; not committing
it means only the machine that created the cluster can.

Decide deliberately. If you leave it ignored, back the directory up somewhere
your team can reach, along with the vault password.

### The prompt fallback

`create-proxy-cluster`'s output format isn't documented and has changed between
CLI releases. The playbook matches both the JSON and human-readable shapes, but
if parsing fails — or if the cluster already exists and no credentials were ever
stored — it prompts for the pair instead of dying:

```
The cluster key pair could not be obtained automatically.

Get the pair from: Admin UI > Networking > Proxy Clusters >
"dc-ansible-cluster" > Keys > Add authentication key

Cluster access key (pk-...):
Cluster secret key (input hidden):
```

Paste it once and it's encrypted and stored like any other run, so every future
run is unattended again. Set `sdm_credentials_prompt: false` in CI, where a
prompt would hang — the playbook then fails with the same instructions.

---

## Bring your own cluster

To adopt a cluster this playbook didn't create, or to use a key you minted by
hand:

1. Admin UI → **Networking → Proxy Clusters → Add proxy cluster**
2. Set **Advertised Address** to `<your-cluster-hostname>:443` and use the same
   value for `sdm_cluster_hostname` in `workers.yml`
3. **Keys** tab → **Add authentication key** → copy both values
4. Put them in `vault.yml`:

   ```yaml
   sdm_proxy_cluster_access_key: "pk-0123456789abcdef"
   sdm_proxy_cluster_secret_key: "..."
   ```

5. Run `site.yml`. It detects both values, skips cluster creation, installs the
   workers — and stores the pair to `credentials/<cluster>.yml` so you can
   remove it from `vault.yml` afterwards if you'd rather not keep two copies.

Clusters allow four authentication keys by default, so minting one for Ansible
while leaving your existing keys alone is fine.

---

## How the unattended install works

The documented install is interactive — it prompts for the access key, then the
secret key. Two mechanisms make it work without a TTY, and the playbook uses
both so it succeeds either way:

```yaml
- name: Install the StrongDM proxy worker as a systemd service
  ansible.builtin.command:
    cmd: >-
      {{ sdm_download_dir }}/sdm install --worker
      --worker-bind-addr :{{ sdm_worker_bind_port }}
      --app-domain {{ sdm_app_domain }}
    stdin: "{{ worker_access_key }}\n{{ worker_secret_key }}"
    creates: "{{ sdm_env_file }}"
  environment:
    SDM_PROXY_CLUSTER_ACCESS_KEY: "{{ worker_access_key }}"
    SDM_PROXY_CLUSTER_SECRET_KEY: "{{ worker_secret_key }}"
```

1. The keys are exported as `SDM_PROXY_CLUSTER_*`, which the worker reads natively.
2. The same pair is piped on stdin, satisfying the prompts if the installer asks anyway.

`creates:` pointing at `/etc/sysconfig/sdm-worker` is what makes re-runs a no-op.
The `command` module has no idempotence of its own, so without it every run
would reinstall.

---

## Design notes

**Why a load balancer and not worker IPs?** StrongDM's docs are explicit: a
cluster with more than one worker requires a network load balancer. The reason
is worth understanding rather than just complying with — a cluster's advertised
address is fixed at creation, so pointing it at a worker means that adding,
replacing, or losing that worker requires a whole new cluster. DNS round-robin
across worker IPs looks like the cheap way out and isn't: DNS has no idea a
worker is draining, so clients keep landing on a node that's shutting down. The
load balancer is the layer that knows.

**Why an Elastic IP per worker if the LB fronts them?** Not for the cluster
address any more — for egress. Workers must reach `app.strongdm.com` and
`downloads.strongdm.com`, and a dynamic public IP that changes on stop/start
also costs you Ansible's stable SSH target. Set `worker_assign_eip: false` if
your subnets already provide egress via NAT; the failure mode if you get that
wrong is the install hanging on the StrongDM download, which is why it defaults
on.

**Why is the health check on a separate port?** Because it's the only signal
that supports graceful upgrades. Workers coordinate a rolling restart during
their maintenance window, and each one announces its turn by shutting down
`:9090` while continuing to serve traffic on `:8443` — waiting 90 seconds for
the load balancer to notice before severing connections. A check against the
traffic port sees nothing wrong and keeps sending new work to a worker that is
about to disappear. This is enabled by `SDM_ORCHESTRATOR_PROBES=:9090` in the
worker's environment file, which the playbook writes.

**Why one security group for the whole cluster?** Every worker has identical
ingress requirements, so a shared group means adding a worker needs no firewall
change at all. It's named after the cluster, not a worker.

**Why is the admin token only on the control node?** The workers never see it.
Each receives a cluster-scoped key pair and nothing more, so compromising a
worker doesn't yield StrongDM admin access.

**Why is the cluster resolved before any AWS spend?** A cluster whose key pair
can't be obtained is a cluster no worker can join. Failing at task five is
cheaper than failing after launching three instances.

**Why `lineinfile` instead of a template for the env file?** The installer
writes `/etc/sysconfig/sdm-worker` itself. Templating the whole file would
clobber whatever it put there, so extra settings are layered in.

**Why 443 in and 8443 out?** StrongDM's documented production shape: clients
expect 443, and binding 8443 on the worker needs no root privilege. The port is
part of the cluster's advertised address, so like the hostname it can't be
changed on an existing cluster — `sdm_cluster_port` and `sdm_worker_bind_port`
are separate variables for exactly that reason. Only collapse them to 443 for a
single-worker cluster with no load balancer at all.

**Why TCP and not a TLS listener?** The workers terminate mutual TLS themselves
and validate client certificates directly (`SDM_TLS_CERT_SOURCE=strongdm`,
`SDM_TLS_CLIENT_AUTH=direct`). A load balancer that terminates TLS on their
behalf breaks the handshake — which is also why an ALB can't front a proxy
cluster and the docs name network load balancers specifically. Terminating at
the LB is possible, but it means `SDM_TLS_CERT_SOURCE=none` plus
`SDM_TLS_CLIENT_AUTH=none`, giving up mutual TLS. Not what you want in a demo of
a privileged access product.

**Why a regex over the CLI output instead of `--json`?** `create-proxy-cluster`
doesn't document its output format and it has changed between CLI releases. One
pattern covers both the JSON and human-readable shapes, and an unparseable
result falls through to the prompt rather than failing. One subtlety worth
flagging if you touch that code: don't assign `cluster_create.stdout` to an
intermediate variable first. Ansible runs `literal_eval` over template results,
so JSON output silently becomes a dict, after which the patterns are matching a
Python repr with single quotes and never fire.

**SELinux.** StrongDM's docs say to disable it before installing. AL2023 ships
it enabled in *permissive* mode, which is already compatible — the playbook only
intervenes if something set it to enforcing.

### Sizing

The defaults are demo-sized: `t3.small` with an 8 GB root volume. That is
deliberately below StrongDM's documented minimum of 2 vCPU / 4 GB, because the
point of this repo is the Ansible workflow, not proxy throughput — a worker with
no resources attached and no client sessions barely uses the box.

If you adapt this for real traffic, change two variables:

```yaml
instance_type: t3.medium   # 2 vCPU / 4 GB — StrongDM's documented minimum
root_volume_gb: 20         # leaves room for journald over the long haul
```

### Running cost

Roughly, for a two-worker cluster in `us-east-2`:

| Item | Approx. |
|---|---|
| 3 × `t3.small` (2 workers + control node) | $0.062/hr |
| Network load balancer | $0.0225/hr + LCU |
| 2 × Elastic IP | $0.010/hr |

Call it **$0.10/hr**, or about $70/month if you leave it running. A few hours of
demo is pennies; the NLB is the piece worth remembering to tear down. `manual`
mode drops the load balancer charge if you're pointing at one that already
exists.

Default account quotas are not a constraint at this size: 5 Elastic IPs and 50
load balancers per region.

---

## Troubleshooting

**`su: unknown login: ec2-user`**
You're on your laptop, not the instance. `ec2-user` is an Amazon Linux account.
Confirm with `uname -s` — if it says `Darwin`, open the AWS console and connect
via Session Manager instead. See the callout at the top of Part 1.

**`Permission denied` from `curl -O` or `unzip` on the control node**
First check you're actually on the instance (`uname -s` → `Linux`). If you are,
you're still in Session Manager's starting directory, `/usr/bin`, which isn't
writable. Check with `pwd && id -un` — if it says `/usr/bin` and `ssm-user`, run
`sudo su - ec2-user` (with the dash) and `cd ~`, then retry. See 1.4.

**`unzip: cannot find or open sdmcli_*_linux_amd64.zip`**
The download didn't produce a zip. Either `curl` wrote an error page instead of
the file — re-run it with `-f` so it fails loudly — or `unzip` isn't installed
yet (`sudo dnf install -y unzip`).

**`zsh: command not found: ansible-vault` (or ansible-playbook, sdm, aws)**
You're on your Mac. macOS uses zsh; the instance uses bash. None of the Part 2
or Part 3 commands run locally — switch to the Session Manager tab.

**`bash: ansible-vault: command not found` on the instance**
The venv isn't active in this shell. Run
`source ~/.venv/ansible/bin/activate`. The `.bashrc` line added in 1.5 covers
future logins but not the shell that created it.

**`UnauthorizedOperation`, or any AWS module failing on a specific API call**
The instance profile's policy is missing that action, or the region fence
doesn't match `aws_region`. Symptoms look unrelated to permissions — e.g.
`Unable to describe VPC attribute enableDnsSupport` or `Failed to update
security group rule descriptions egress`. Re-apply
[`controlnode/iam-policy.json`](controlnode/iam-policy.json) (IAM → Policies →
Edit → paste → Save creates a new active version) and confirm its
`aws:RequestedRegion` matches your region.

**`No help topic for 'version'` (rc 3)**
`--version` is a global flag on the `sdm` binary, not a subcommand. Use
`sdm --version`.

**`sdm: command not found` on the control node**
`/usr/local/bin` missing from `PATH`, or step 1.5 didn't complete. Re-run the
`install -m 0755` line.

**`No proxy cluster authentication key pair is available`**
Either the CLI output format differs from what the regex looks for, or the
cluster already exists and no credentials file was ever stored. With
`sdm_credentials_prompt: true` (the default) the playbook prompts for the pair
instead of failing; you only see this message if prompting is off. Either way the
fix is the same — get the pair from the Admin UI and paste it in. See
[Credential persistence](#credential-persistence).

**`workers.yml is incomplete`**
`sdm_cluster_endpoint` must be exactly `nlb` or `manual`, every worker needs a
`name`, and the names must be unique.

**`sdm_cluster_endpoint is "manual", so sdm_cluster_hostname must name…`**
Either set a real hostname for your load balancer, or switch to
`sdm_cluster_endpoint: nlb` and let the playbook build one. The placeholder is
rejected deliberately: the address can't be changed after the cluster is
created.

**`only 1 availability zone(s) were resolved`**
An NLB requires subnets in at least two AZs. Usually caused by the global
`subnet_id` override in `group_vars/all.yml` pinning everything to one subnet —
clear it, or set `vpc_id` to a VPC with subnets in two or more zones.

**`couldn't resolve module/action 'community.aws.elb_network_lb'`**
`community.aws` isn't installed. Re-run
`ansible-galaxy collection install -r requirements.yml`. It was added for the
NLB modules, which haven't moved into `amazon.aws` yet.

**Load balancer creation fails with `AccessDenied` or `UnauthorizedOperation`**
The IAM policy predates the NLB support — it needs
`elasticloadbalancing:*` and the scoped `iam:CreateServiceLinkedRole`. Re-apply
[`controlnode/iam-policy.json`](controlnode/iam-policy.json).

**Targets stuck `unhealthy`, or the run times out registering them**
The health check is HTTP on `:9090/liveness`, so check in this order:

```bash
# 1. Is the liveness endpoint actually up on the worker?
ansible sdm_workers -m uri -a 'url=http://localhost:9090/liveness' -b

# 2. Is SDM_ORCHESTRATOR_PROBES set?
ansible sdm_workers -m command -a 'grep ORCHESTRATOR /etc/sysconfig/sdm-worker' -b

# 3. Does the security group allow 9090 from the subnet CIDRs?
#    Health checks come from the LB's own addresses, not from clients.
```

As a last resort set `nlb_health_check_protocol: TCP` and drop
`health_check_path` — but you lose graceful draining during rolling upgrades,
so treat it as a diagnostic rather than a fix.

**`curl` through the NLB hangs instead of returning 404**
Almost always no healthy targets — an NLB with an empty target group blackholes
connections rather than refusing them. Check target health before suspecting the
workers.

**A new worker in `workers.yml` didn't get built**
Check the `Report the planned fleet` task — it prints each resolved worker. If
yours isn't listed, the entry isn't being parsed (indentation, or a stray
duplicate `name`). If it is listed but nothing launched, look at
`Launch the proxy worker instances` — an existing instance with the same `Name`
tag comes back `ok` rather than `changed`.

**Existing workers show as skipped on a re-run**
That's the expected add-only behaviour. `creates: /etc/sysconfig/sdm-worker`
makes the install a no-op on a worker that already has it, so a run that adds
one worker to a fleet of three should show one `changed` and three `ok`.

**`Attempting to decrypt but no vault secrets found`**
`ansible.cfg`'s `vault_password_file` and `sdm_vault_password_file` in
`group_vars/all.yml` have to name the same file. The credentials file is
encrypted with the latter and decrypted with the former.

**Playbook hangs on `Wait for SSH to accept connections`**
The security group's port 22 rule allows the control node's private IP. If your
control node is in a different VPC, that won't route. Put both in the same VPC,
or set `ssh_ingress_cidrs` explicitly.

**Worker installs but the Admin UI shows it disconnected**
Egress problem. From the worker:

```bash
curl -sv https://app.strongdm.com 2>&1 | head
curl -sv https://downloads.strongdm.com 2>&1 | head
```

Both must reach on 443. If your sandbox egresses through a corporate proxy, set
`sdm_http_proxy` / `sdm_https_proxy` in `group_vars/all.yml`.

**`curl -k https://<EIP>` times out from your laptop**
`client_ingress_cidrs` doesn't include your address, or your corporate network
blocks outbound 443 to arbitrary hosts.

**Service starts then dies**
```bash
journalctl -u sdm-worker -n 100 --no-pager
```
Usually a bad or already-consumed key pair. Clusters allow four keys by default;
mint a fresh one and re-run.

---

## Files

| File | Purpose |
|---|---|
| `workers.yml` | **Cluster topology — the file you edit to add a worker** |
| `site.yml` | The playbook — resolve credentials, build the LB and fleet, install workers, put them in rotation |
| `teardown.yml` | Destroy one cluster, its load balancer, and its workers |
| `group_vars/all.yml` | All other non-secret configuration |
| `vault.yml.example` | Template for the encrypted secrets file |
| `credentials/<cluster>.yml` | Vault-encrypted cluster key pair, written by `site.yml`. Gitignored — [back it up](#back-it-up) |
| `inventory/aws_ec2.yml` | Dynamic inventory for re-runs and ad-hoc commands |
| `requirements.yml` | Galaxy collections |
| `controlnode/iam-policy.json` | IAM policy for the control node instance profile |
| `ansible.cfg`, `.yamllint` | Tooling config |

---

## Reference

- [Proxy Clusters](https://docs.strongdm.com/admin/networking/proxy-clusters) — the load balancer requirement, the 443→8443 recommendation, and the rolling-upgrade sequence
- [Liveness Check](https://docs.strongdm.com/admin/networking/gateways-and-relays#liveness-check) — `SDM_ORCHESTRATOR_PROBES`, the port the LB should health-check
- [Environment Variables](https://docs.strongdm.com/admin/deployment/environment-variables)
- [Ports Guide](https://docs.strongdm.com/admin/networking/ports-guide)
- [Maintenance Windows](https://docs.strongdm.com/admin/networking/maintenance-windows) — why graceful draining matters
- [Deploy ECS Fargate Proxy Cluster](https://docs.strongdm.com/admin/networking/proxy-clusters/ecs-proxy-clusters) — the same NLB topology, containerised
- [`sdm admin nodes create-proxy-cluster`](https://docs.strongdm.com/references/cli/admin/nodes/create-proxy-cluster)
- [Gateways and Relays](https://docs.strongdm.com/admin/networking/gateways-and-relays) — the other model, for contrast
