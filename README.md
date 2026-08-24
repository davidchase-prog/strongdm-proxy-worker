# Installing a StrongDM Proxy Worker with Ansible

An example Ansible playbook that provisions an EC2 instance and installs a
**StrongDM proxy cluster worker natively** — the `sdm` binary managed by
systemd, no Docker, no Kubernetes.

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
| Default port | 8443 (443 for single-worker) | 5000 |
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
                          443 ▲  │  ▲ 443
              ┌───────────────┘  │  └────────────────┐
              │                  │                   │
    ┌─────────┴─────────┐        │        ┌──────────┴───────────┐
    │  Control node     │        │        │  Proxy worker        │
    │  (EC2, AL2023)    │        │        │  (EC2, AL2023)       │
    │                   │  ssh   │        │                      │
    │  ansible-core     ├────────┼───────►│  sdm binary          │
    │  sdm CLI          │   22   │        │  systemd: sdm-worker │
    │  boto3            │        │        │  binds :443          │
    └───────────────────┘        │        └──────────▲───────────┘
                                 │                   │ 443
                                 │            StrongDM clients
```

Resources created in AWS, all tagged `Project=strongdm-ansible-demo`:

- one `t3.small` Amazon Linux 2023 instance with an 8 GB root volume — sized for
  a demo (see [Sizing](#sizing) if you're adapting this for real use)
- a security group: inbound 443 from your client CIDRs, inbound 22 from the control node
- an Elastic IP
- an EC2 keypair, imported from a key generated on the control node

Created in StrongDM:

- a proxy cluster named `dc-ansible-cluster` advertising `<EIP>:443`
- one authentication key pair for that cluster

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

This grants the EC2 permissions the playbook needs to build the worker — split
into a read-only statement (describe VPCs, subnets, AMIs) and a write statement
(run/terminate instances, manage security groups and Elastic IPs).

The write statement is scoped by an `aws:RequestedRegion` condition, currently
`us-east-2`. **It must match `aws_region` in `group_vars/all.yml`** or every EC2
call the playbook makes returns `UnauthorizedOperation`.

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

### 2.2 Review the variables

Everything tunable lives in [`group_vars/all.yml`](group_vars/all.yml). The ones
you're most likely to touch:

| Variable | Default | Notes |
|---|---|---|
| `aws_region` | `us-east-2` | Must match the IAM policy condition (1.1) |
| `sdm_app_domain` | `app.strongdm.com` | Change for a UK or EU **control plane** |
| `sdm_cluster_name` | `dc-ansible-cluster` | Name shown in the Admin UI |
| `sdm_worker_bind_port` | `443` | Use `8443` behind a load balancer |
| `client_ingress_cidrs` | `0.0.0.0/0` | Narrow it if your sandbox has guardrails |
| `vpc_id` / `subnet_id` | empty | Empty means default VPC, auto-discovered |

---

## Part 3 — Run it

```bash
# Dry run first
ansible-playbook site.yml --check --diff

# Real run
ansible-playbook site.yml
```

Expect roughly three minutes, most of it waiting on the instance.

Success looks like:

```
TASK [Done]
ok: [sdm-proxy-worker-01] =>
  msg:
  - Proxy worker installed natively via binary + systemd.
  - 'Cluster:    dc-ansible-cluster'
  - 'Advertised: 54.x.x.x:443'
  - 'Service:    sdm-worker (active)'
  - 'Probe:      HTTP 404 (404 = healthy)'
```

**HTTP 404 is the healthy state.** A proxy worker is a TLS proxy, not a web
server — 404 on plain HTTPS means it's up and listening. StrongDM's own docs
use this as the verification step.

### Verify independently

```bash
# From the control node
curl -k https://<EIP>          # -> 404 Not Found

# On the worker
ansible sdm_workers -m command -a 'systemctl status sdm-worker' -b
```

And in the Admin UI under **Networking → Proxy Clusters**, the cluster should
show as connected.

---

## Part 4 — Use it

The worker proxies nothing until resources are attached to its cluster:

- **Admin UI**: edit a resource, set its **Proxy Cluster** field
- **CLI**: pass `--proxy-cluster-id plc-...` (find the id with `sdm admin nodes list`)

Every worker in one cluster must be able to reach the same set of resources.
Different network segments need different clusters.

---

## Part 5 — Clean up

```bash
ansible-playbook teardown.yml
```

Removes the StrongDM cluster, the instance, the EIP, the security group, and
the keypair. Then terminate the control node in the console.

Skip the confirmation prompt in CI with `-e auto_approve=true`.

---

## Bring your own cluster

The playbook creates the cluster and mints its authentication key via
`sdm admin nodes create-proxy-cluster`. **StrongDM does not document the output
format of that command**, so the playbook parses it defensively — and if the key
pair can't be found, it stops with instructions instead of failing obscurely.

If that happens, or if you'd rather manage the cluster by hand:

1. Admin UI → **Networking → Proxy Clusters → Add proxy cluster**
2. Set **Advertised Address** to `<worker-public-ip>:443`
3. **Keys** tab → **Add authentication key** → copy both values
4. Put them in `vault.yml`:

   ```yaml
   sdm_proxy_cluster_access_key: "pk-0123456789abcdef"
   sdm_proxy_cluster_secret_key: "..."
   ```

5. Re-run `site.yml`. It detects both values, skips cluster creation, and goes
   straight to installing the worker.

This is worth knowing about before you demo the playbook to anyone.

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

**Why an Elastic IP?** StrongDM ties a node's advertised address to its IP, and
there is no way to change it after creation. A reboot that reassigned a dynamic
public IP would orphan the cluster. The EIP is what makes the demo survive a
stop/start.

**Why is the admin token only on the control node?** The worker never sees it.
It receives a cluster-scoped key pair and nothing more, so compromising the
worker doesn't yield StrongDM admin access. The cluster-creation tasks use
`delegate_to: localhost` for exactly this reason.

**Why `lineinfile` instead of a template for the env file?** The installer
writes `/etc/sysconfig/sdm-worker` itself. Templating the whole file would
clobber whatever it put there, so extra settings are layered in.

**Why 443 and not 8443?** This is a single-worker sandbox cluster, so clients
connect to the worker directly. StrongDM's production recommendation is workers
on 8443 behind a network load balancer that remaps 443. Set
`sdm_worker_bind_port: 8443` and add the LB when you outgrow one worker.

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

**`No help topic for 'version'` (rc 3)**
`--version` is a global flag on the `sdm` binary, not a subcommand. Use
`sdm --version`.

**`sdm: command not found` on the control node**
`/usr/local/bin` missing from `PATH`, or step 1.5 didn't complete. Re-run the
`install -m 0755` line.

**`Could not determine the proxy cluster authentication key pair`**
Expected if the CLI output format differs from what the regex looks for. Follow
**Bring your own cluster** above — the cluster itself was probably created fine.

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
| `site.yml` | The playbook — provision, create cluster, install worker |
| `teardown.yml` | Destroy everything |
| `group_vars/all.yml` | All non-secret configuration |
| `vault.yml.example` | Template for the encrypted secrets file |
| `inventory/aws_ec2.yml` | Dynamic inventory for re-runs and ad-hoc commands |
| `requirements.yml` | Galaxy collections |
| `controlnode/iam-policy.json` | IAM policy for the control node instance profile |
| `ansible.cfg`, `.yamllint` | Tooling config |

---

## Reference

- [Proxy Clusters](https://docs.strongdm.com/admin/networking/proxy-clusters)
- [Environment Variables](https://docs.strongdm.com/admin/deployment/environment-variables)
- [Ports Guide](https://docs.strongdm.com/admin/networking/ports-guide)
- [Maintenance Windows](https://docs.strongdm.com/admin/networking/maintenance-windows)
- [`sdm admin nodes create-proxy-cluster`](https://docs.strongdm.com/references/cli/admin/nodes/create-proxy-cluster)
- [Gateways and Relays](https://docs.strongdm.com/admin/networking/gateways-and-relays) — the other model, for contrast
