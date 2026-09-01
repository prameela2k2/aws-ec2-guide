# AWS EC2: From Zero to a Working Remote Dev Box

A practical guide built from an actual session on 2026-08-31 — including every
failure and the error text it produced. Written for Windows 10 with Git Bash and
PowerShell.

> **Use PowerShell for `aws` commands.** `aws.exe` is a Windows binary and won't
> understand Git Bash paths like `/c/Users/...`. It installs to
> `C:\Program Files\Amazon\AWSCLIV2\aws.exe`.

## Fill these in first

Run this once per PowerShell session — the commands below reference these variables:

```powershell
$ACCT = "123456789012"           # aws sts get-caller-identity --query Account --output text
$KEY  = "my-laptop"              # name for your key pair in AWS
$IID  = "i-0123456789abcdef0"    # instance id, once you have one
$SG   = "sg-0123456789abcdef0"   # security group id
$VOL  = "vol-0123456789abcdef0"  # root volume id
```

Find them any time with:

```powershell
aws sts get-caller-identity
aws ec2 describe-instances `
  --query 'Reservations[].Instances[].{ID:InstanceId,State:State.Name,IP:PublicIpAddress,SG:SecurityGroups[0].GroupId}' `
  --output table
```

---

## 0. What EC2 is, and when it's worth it

EC2 is **a computer you rent by the second in a data center**. That's the whole
idea. You SSH in and it's an ordinary Linux box — except it has a public IP and
keeps running when your laptop is closed.

Use it when you need one of these:

1. **It must stay up when your laptop sleeps** — web server, API, bot, cron job
2. **It must be publicly reachable** — webhooks, a demo URL others can open
3. **It must be bigger than your laptop** — lots of RAM, or a GPU, for a few hours
4. **It must be disposable** — break it, terminate it, relaunch clean in 60 seconds
5. **It must be a consistent Linux env** — the same box for the whole team

Otherwise, run it locally. It's just someone else's computer.

**Reality check:** a t3.micro has 1 GB RAM and 2 vCPU. That is *less capable than
free Google Colab*. It's an excellent web server and Linux practice box; it is not
an ML machine. For that you want a GPU instance (`g4dn.xlarge`, ~$0.53/hr), which
requires a quota-increase request before you can launch one.

---

## 1. Generate an SSH key pair

**Git Bash:**

```bash
mkdir -p ~/.ssh && chmod 700 ~/.ssh
ssh-keygen -t ed25519 -f ~/.ssh/ec2-key -C "my-laptop" -N ""
```

**PowerShell** — note the quoting differs; `-N ""` is not empty in PowerShell:

```powershell
New-Item -ItemType Directory -Force "$env:USERPROFILE\.ssh" | Out-Null
ssh-keygen -t ed25519 -f "$env:USERPROFILE\.ssh\ec2-key" -C "my-laptop" -N '""'
```

- `ec2-key` — **private key, never share it**
- `ec2-key.pub` — public key, safe to paste anywhere

Omit `-N` entirely to be prompted for a passphrase (more secure; you type it on
each connect).

### Fix the key permissions — Windows-specific, and it WILL bite you

SSH refuses to use a private key other accounts can read. On Windows, `chmod 600`
from Git Bash often does **not** stick — you must set the NTFS ACL.

**Git Bash:**

```bash
export MSYS_NO_PATHCONV=1      # stop Git Bash rewriting /flags into paths
KEYPATH="C:\Users\$USERNAME\.ssh\ec2-key"
WINUSER=$(/c/Windows/System32/whoami.exe | tr -d '\r')   # -> machine\user
icacls "$KEYPATH" /reset
icacls "$KEYPATH" /inheritance:r /grant:r "${WINUSER}:R"
icacls "$KEYPATH"              # verify: exactly ONE entry, your account
```

**PowerShell:**

```powershell
$k = "$env:USERPROFILE\.ssh\ec2-key"
icacls $k /reset
icacls $k /inheritance:r /grant:r "$env:USERDOMAIN\$($env:USERNAME):R"
icacls $k
```

⚠️ **Use the full `machine\user`, not the bare username.** Git Bash's own
`whoami` and PowerShell's `$env:USERNAME` both return only the username.
`icacls` accepts it, resolves it to a **bogus principal**, and combined with
`/inheritance:r` locks *you* out of your own key:

```
Load key "C:\Users\...\ec2-key": Permission denied
ubuntu@1.2.3.4: Permission denied (publickey).
```

Recover by re-running `/reset` then `/grant:r` with the full `machine\user`.

---

## 2. Launch an instance

### Register your public key with AWS

```powershell
aws ec2 import-key-pair --key-name $KEY `
  --public-key-material "fileb://$env:USERPROFILE/.ssh/ec2-key.pub"
```

`fileb://` (binary) rather than `file://` — the text form can mangle the key on
Windows.

### Look up the AMI — never hardcode it

AMI IDs are **region-specific and change constantly**. The instance in this guide
was built on `ami-0f8a61b66d1accaee`; a few weeks later the current Ubuntu 24.04
image was `ami-0d7f022123f8ff19d`. Always resolve it from an SSM public parameter:

```powershell
# Ubuntu 24.04 LTS
$AMI = aws ssm get-parameters `
  --names "/aws/service/canonical/ubuntu/server/24.04/stable/current/amd64/hvm/ebs-gp3/ami-id" `
  --query "Parameters[0].Value" --output text

# Amazon Linux 2023 alternative
# --names "/aws/service/ami-amazon-linux-latest/al2023-ami-kernel-default-x86_64"

echo $AMI
```

> In **Git Bash**, set `export MSYS_NO_PATHCONV=1` first or the leading `/aws/...`
> gets rewritten to `C:/Program Files/Git/aws/...` and returns `InvalidParameters`.

### Create a security group

```powershell
$myip = (Invoke-RestMethod https://checkip.amazonaws.com).Trim()

$SG = aws ec2 create-security-group --group-name dev-box-sg `
  --description "SSH from my IP, HTTP/HTTPS public" `
  --query "GroupId" --output text

# SSH: your IP only
aws ec2 authorize-security-group-ingress --group-id $SG `
  --protocol tcp --port 22 --cidr "$myip/32"

# HTTP/HTTPS: public (skip these if you aren't serving anything)
aws ec2 authorize-security-group-ingress --group-id $SG --protocol tcp --port 80  --cidr 0.0.0.0/0
aws ec2 authorize-security-group-ingress --group-id $SG --protocol tcp --port 443 --cidr 0.0.0.0/0
```

### Launch it

```powershell
$IID = aws ec2 run-instances `
  --image-id $AMI `
  --instance-type t3.micro `
  --key-name $KEY `
  --security-group-ids $SG `
  --block-device-mappings '[{\"DeviceName\":\"/dev/sda1\",\"Ebs\":{\"VolumeSize\":20,\"VolumeType\":\"gp3\"}}]' `
  --tag-specifications "ResourceType=instance,Tags=[{Key=Name,Value=dev-box}]" `
  --query "Instances[0].InstanceId" --output text

aws ec2 wait instance-status-ok --instance-ids $IID

aws ec2 describe-instances --instance-ids $IID `
  --query "Reservations[0].Instances[0].PublicIpAddress" --output text
```

**Use 20 GB, not the 8 GB default.** An 8 GB root volume hits 95% full from just
the OS, VS Code's remote server, and JupyterLab. The extra 12 GB costs about
$1/month. See §7.

`wait instance-status-ok` blocks until the OS has actually booted — `running`
alone means the VM exists, not that sshd is listening.

---

## 3. Rescue an instance that has no key pair

If you launched from the console without selecting a key pair, you normally can't
log in at all. **EC2 Instance Connect** rescues it by injecting a temporary key
(valid **60 seconds**) authorised by your AWS credentials. Ubuntu and Amazon Linux
AMIs ship with the agent.

```powershell
aws ec2 start-instances --instance-ids $IID
aws ec2 wait instance-status-ok --instance-ids $IID

$ip = aws ec2 describe-instances --instance-ids $IID `
  --query "Reservations[0].Instances[0].PublicIpAddress" --output text
$az = aws ec2 describe-instances --instance-ids $IID `
  --query "Reservations[0].Instances[0].Placement.AvailabilityZone" --output text
```

Inject, then SSH **immediately**:

```powershell
aws ec2-instance-connect send-ssh-public-key `
  --instance-id $IID `
  --instance-os-user ubuntu `
  --availability-zone $az `
  --ssh-public-key "file://$env:USERPROFILE/.ssh/ec2-key.pub"

ssh -i "$env:USERPROFILE\.ssh\ec2-key" ubuntu@$ip
```

**Then make it permanent** so you never repeat the 60-second dance:

```bash
# ON the instance
mkdir -p ~/.ssh && chmod 700 ~/.ssh
echo "ssh-ed25519 AAAA...your-public-key... my-laptop" >> ~/.ssh/authorized_keys
chmod 600 ~/.ssh/authorized_keys
```

This requires the security group to allow SSH from the EC2 Instance Connect
service range — an open `0.0.0.0/0` rule satisfies it, but once you lock port 22
to your own IP (§5), use the CLI form above rather than the console button.

Default SSH usernames: `ubuntu` (Ubuntu), `ec2-user` (Amazon Linux),
`admin` (Debian).

---

## 4. SSH shortcut

`C:\Users\<you>\.ssh\config`:

```
Host ec2
    HostName <PUBLIC_IP>
    User ubuntu
    IdentityFile C:\Users\<you>\.ssh\ec2-key
    IdentitiesOnly yes
    ServerAliveInterval 60
    StrictHostKeyChecking accept-new
```

Now just `ssh ec2`.

⚠️ **The public IP changes on every stop/start** (a plain `reboot` keeps it).
Update `HostName` each time. One-liner from Git Bash:

```bash
NEWIP=$(aws ec2 describe-instances --instance-ids $IID \
  --query 'Reservations[0].Instances[0].PublicIpAddress' --output text | tr -d '\r')
sed -i "s/^\( *HostName \).*/\1$NEWIP/" ~/.ssh/config
```

To pin the address permanently, allocate an Elastic IP. Since AWS began charging
for all public IPv4 in Feb 2024 an EIP costs the same ~$3.65/mo as the
auto-assigned address while running — but unlike the auto-assigned one, it keeps
billing when the instance is stopped.

---

## 5. Lock down the firewall

A launch-wizard security group opens SSH to `0.0.0.0/0`. Bots find that within
minutes. Restrict port 22 to your IP:

```powershell
$myip = (Invoke-RestMethod https://checkip.amazonaws.com).Trim()

aws ec2 authorize-security-group-ingress --group-id $SG `
  --ip-permissions "IpProtocol=tcp,FromPort=22,ToPort=22,IpRanges=[{CidrIp=$myip/32,Description=my-laptop}]"

aws ec2 revoke-security-group-ingress --group-id $SG `
  --protocol tcp --port 22 --cidr 0.0.0.0/0
```

Verify:

```powershell
aws ec2 describe-security-groups --group-ids $SG `
  --query 'SecurityGroups[0].IpPermissions[].{Port:FromPort,Allowed:IpRanges[].CidrIp}' --output json
```

**Home IPs rotate.** When `ssh ec2` starts timing out for no reason, that's the
first thing to check — re-run the `authorize` command with your new IP.

**The security group is the firewall.** Same machine, same network interface:
port 80 open to the world, port 22 open only to you. Getting this wrong is how
instances get compromised.

---

## 6. VS Code Remote-SSH

Files, terminal, Python interpreter and extensions all run **on the server**;
only the UI is local.

```bash
code --install-extension ms-vscode-remote.remote-ssh
```

Connect: <kbd>Ctrl</kbd>+<kbd>Shift</kbd>+<kbd>P</kbd> →
**Remote-SSH: Connect to Host** → `ec2`. Or from a terminal:

```bash
code --remote ssh-remote+ec2 /home/ubuntu/projects
```

Look for a green **`SSH: ec2`** badge bottom-left. <kbd>Ctrl</kbd>+<kbd>`</kbd>
opens a terminal *on EC2*.

### THE most common Windows failure: `powershell: command not found`

```
[Remote - SSH] remote.SSH.remotePlatform = {}
Running script with connection command: ssh -T -D 60235 ec2 powershell
> bash: line 1: powershell: command not found
> Resolver error
```

VS Code doesn't know the remote OS, so on a Windows client it **assumes the remote
is Windows** and types `powershell` at your bash prompt. Normally it shows a
"Select the platform of the remote host" quick-pick — but launching from the CLI
with `--remote` skips that prompt entirely, so it never gets answered.

Fix permanently in `%APPDATA%\Code\User\settings.json`:

```json
{
    "remote.SSH.remotePlatform": { "ec2": "linux" },
    "remote.SSH.connectTimeout": 60,
    "remote.SSH.defaultExtensions": ["ms-python.python", "ms-toolsai.jupyter"]
}
```

Success looks like this — note `sh`, not `powershell`:

```
remote.SSH.remotePlatform = {"ec2":"linux"}
Running script with connection command: ssh -T -D 60312 ec2 sh
Remote server is listening on port 46219
```

**Debug any Remote-SSH problem here:** Output panel → **"Remote - SSH"** in the
dropdown. On disk:
`%APPDATA%\Code\logs\<timestamp>\window<N>\exthost\output_logging_*\*Remote - SSH.log`

---

## 7. Prepare the server

1 GB RAM is tight — **add swap first**, or pip installs get OOM-killed:

```bash
sudo fallocate -l 2G /swapfile
sudo chmod 600 /swapfile
sudo mkswap /swapfile
sudo swapon /swapfile
echo '/swapfile none swap sw 0 0' | sudo tee -a /etc/fstab   # survives reboot
free -h
```

Python tooling and a virtualenv:

```bash
sudo apt-get update -qq
sudo apt-get install -y python3-pip python3-venv
mkdir -p ~/projects && cd ~/projects
python3 -m venv venv
source venv/bin/activate
pip install --upgrade pip
pip install jupyterlab pandas
```

### The 8 GB default disk is a trap

Real numbers from this build, on a fresh 8 GB volume:

| | |
|---|---|
| `/usr` (the OS) | 1.8 GB |
| `.vscode-server` (server 700 MB + extensions 284 MB) | ~1.0 GB |
| `projects/venv` (JupyterLab + pandas) | 367 MB |
| `/var` | 600 MB |

That reached **95% full**, at which point kernels die mid-execution and pip
installs fail halfway. Reclaim space:

```bash
sudo apt-get clean && sudo apt-get autoremove -y
rm -rf ~/.cache/pip
sudo journalctl --vacuum-size=50M
du -sh ~/.vscode-server/cli/servers/*     # one ~700 MB dir PER VS Code version
df -h /
```

VS Code keeps a full server download for every version it has ever connected
with. Delete all but the current commit — it's named in the Remote-SSH log as
`Using commit id <hash> for server`.

### Growing the volume

```powershell
aws ec2 modify-volume --volume-id $VOL --size 20
```

Then on the instance — **check the device name first, do not assume it**:

```bash
lsblk
```

Nitro-based instances (t3, t4g, m5, c5, and newer) show `nvme0n1`; older
Xen-based families (t2, m4) show `xvda`. Use whichever `lsblk` actually reports:

```bash
# if lsblk shows nvme0n1 with partition nvme0n1p1
sudo growpart /dev/nvme0n1 1 && sudo resize2fs /dev/nvme0n1p1

# if lsblk shows xvda with partition xvda1
sudo growpart /dev/xvda 1 && sudo resize2fs /dev/xvda1

df -h /
```

It resizes live, with no downtime and no data loss. ⚠️ **EBS volumes cannot be
shrunk** — this is one-directional.

---

## 8. Jupyter notebooks in VS Code — the real Colab replacement

Register your venv as a kernel:

```bash
source ~/projects/venv/bin/activate
pip install ipykernel
python -m ipykernel install --user --name projects-venv \
       --display-name "Python (projects venv)"
jupyter kernelspec list      # verify it appears
```

Ensure the Python and Jupyter extensions install **on the remote** by adding
`remote.SSH.defaultExtensions` to settings.json (see §6).

Open any `.ipynb` in the `SSH: ec2` window → **Select Kernel** →
**Jupyter Kernel** → **Python (projects venv)**.

The notebook file, the kernel, the interpreter and every variable in memory live
on EC2. VS Code is a viewport. No session timeouts, files persist between
sessions, and you control the machine.

**Proof you're really on EC2** — the instance metadata service answers only from
inside an instance, so this hangs forever on a laptop:

```python
import urllib.request

def imds(path):
    tok = urllib.request.urlopen(urllib.request.Request(
        "http://169.254.169.254/latest/api/token",
        headers={"X-aws-ec2-metadata-token-ttl-seconds": "60"},
        method="PUT")).read()
    return urllib.request.urlopen(urllib.request.Request(
        "http://169.254.169.254/latest/meta-data/" + path,
        headers={"X-aws-ec2-metadata-token": tok})).read().decode()

for f in ["instance-id", "instance-type", "placement/availability-zone"]:
    print(f"{f:32} {imds(f)}")
```

`169.254.169.254` is a link-local address routable only inside EC2. The two-step
token exchange is IMDSv2, required by default on new instances.

**For most work this is strictly better than Colab.** Use the actual Colab UI
only if you specifically want it — install `jupyter_http_over_ws`, run JupyterLab
on the instance, tunnel it with `ssh -N -L 8888:localhost:8888 ec2`, then use
Colab's "Connect to a local runtime" with the `localhost:8888/?token=...` URL.
Colab only accepts `localhost`, which is why the tunnel is mandatory.

---

## 9. Serve something to the public internet

This is the part your laptop cannot do.

`/etc/systemd/system/demo-site.service`:

```ini
[Unit]
Description=Demo static site on port 80
After=network.target

[Service]
ExecStart=/usr/bin/python3 -m http.server 80 --directory /home/ubuntu/site
Restart=always
User=root

[Install]
WantedBy=multi-user.target
```

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now demo-site
systemctl is-active demo-site                                  # -> active
curl -s -o /dev/null -w '%{http_code}\n' http://localhost/      # -> 200
```

Then open `http://<PUBLIC_IP>` **on your phone with wifi off**. It loads. Close
your laptop — it still loads.

- `--now` starts it; `enable` brings it back **after reboot**
- `Restart=always` relaunches it on crash — test with `sudo pkill -f http.server`
- **Type `http://` explicitly.** Phone browsers auto-upgrade to HTTPS (nothing
  listens on 443), and a bare IP often becomes a search query. Both look exactly
  like "the server is down."

### Debugging gotchas that cost real time

**`Address already in use` means it's WORKING.** Running the command by hand
while the service already holds the port gives:

```
OSError: [Errno 98] Address already in use
```

That error proves the port is bound. It is not evidence of a fault.

**Don't race the socket.** Checking `ss`/`curl` in the same command as
`systemctl enable --now` reports "nothing listening" because Python hasn't
finished binding yet. Check as a separate step.

**Run server commands on the SERVER.** A prompt reading `PS C:\Users\you>` is
your laptop; `ubuntu@ip-172-31-17-135:~$` is EC2. Pasting `sudo ...` into
PowerShell just errors — and a staged unit file that was never moved into
`/etc/systemd/system/` means nothing is running at all.

Full sweep when something is unreachable:

```bash
systemctl is-active demo-site      # is the service up?
ss -tln | grep ':80'               # is anything bound?
curl http://localhost/             # does it work locally?
sudo ufw status                    # host firewall blocking?
# then: security group rules, and confirm the public IP hasn't changed
```

`python -m http.server` is a **toy** — single-threaded, no HTTPS, no access
control. Real deployments use nginx in front of gunicorn/uvicorn, plus certbot
for TLS (which needs a domain name pointed at the IP).

---

## 10. Cost control and budget alarms

`us-east-1` pricing:

| Item | Cost | Billed while STOPPED? |
|---|---|---|
| t3.micro compute | $0.0104/hr (~$7.60/mo) | **No** |
| Public IPv4 address | $0.005/hr (~$3.65/mo) | No (released on stop) |
| EBS root disk, gp3 | $0.08/GB-month → $0.64 for 8 GB, $1.60 for 20 GB | **Yes — always** |

**Stop the instance when you're done.** This one habit keeps it near-free:

```powershell
aws ec2 stop-instances --instance-ids $IID
```

### Set a budget alarm on day one

`budget.json`:

```json
{
  "BudgetName": "monthly-5-usd",
  "BudgetLimit": { "Amount": "5", "Unit": "USD" },
  "TimeUnit": "MONTHLY",
  "BudgetType": "COST"
}
```

`notifications.json`:

```json
[
  { "Notification": { "NotificationType": "ACTUAL", "ComparisonOperator": "GREATER_THAN",
      "Threshold": 80, "ThresholdType": "PERCENTAGE" },
    "Subscribers": [ { "SubscriptionType": "EMAIL", "Address": "you@example.com" } ] },
  { "Notification": { "NotificationType": "FORECASTED", "ComparisonOperator": "GREATER_THAN",
      "Threshold": 100, "ThresholdType": "PERCENTAGE" },
    "Subscribers": [ { "SubscriptionType": "EMAIL", "Address": "you@example.com" } ] }
]
```

```powershell
aws budgets create-budget --account-id $ACCT `
  --budget file://budget.json `
  --notifications-with-subscribers file://notifications.json

aws budgets describe-budgets --account-id $ACCT --output table
```

**FORECASTED alerts are the valuable ones** — they warn you on day 3 that you're
on track to blow the budget, rather than telling you after the money is gone.

`aws ce get-cost-and-usage` (Cost Explorer) returns
`AccessDeniedException: User not enabled for cost explorer access` until it is
switched on once from the root account in the Billing console. Budgets works
without it.

Check which free tier applies under **Billing → Free Tier**. Accounts created
before ~July 2025 get 750 hrs/month of t2/t3.micro for 12 months; newer accounts
get a credit-based free plan instead.

---

## 11. stop vs terminate, and auditing with CloudTrail

| | Compute billing | Your data | Public IP |
|---|---|---|---|
| **stop** | ends | **survives** (EBS volume kept) | lost |
| **terminate** | ends | **destroyed forever** | lost |

`DeleteOnTermination` defaults to **true** on the root volume. Terminating an
instance silently deletes the disk — code, venvs, installed packages, config,
everything. There is no undo and no recycle bin.

Protect a volume you care about:

```powershell
aws ec2 modify-instance-attribute --instance-id $IID `
  --block-device-mappings '[{\"DeviceName\":\"/dev/sda1\",\"Ebs\":{\"DeleteOnTermination\":false}}]'
```

Use the device name from `describe-instances`, which may be `/dev/sda1` or
`/dev/xvda` depending on the AMI. Or just snapshot before anything risky:

```powershell
aws ec2 create-snapshot --volume-id $VOL --description "before experiment"
```

### Find out who did what

CloudTrail records every API call, searchable for 90 days at no cost:

```powershell
aws cloudtrail lookup-events --start-time 2026-08-31T16:00:00Z --max-results 40 `
  --query 'Events[].{Time:EventTime,Event:EventName,User:Username}' --output table
```

The `Username` column is the useful part — `root` means someone was clicking in
the console, your IAM username means it came from a script or the CLI. That's how
you establish whether a resource died from a console click or a command:

```
23:33:27  StopInstances       prameela_m   <- CLI
23:33:46  TerminateInstances  root         <- console
```

**Don't operate as root.** AWS recommends the root account only for account-level
tasks (billing settings, closing the account). Do daily work as an IAM user with
MFA enabled. Root has no permission boundaries and cannot be restricted.

---

## Command cheat sheet

```powershell
# --- lifecycle ---
aws ec2 start-instances     --instance-ids $IID
aws ec2 stop-instances      --instance-ids $IID
aws ec2 terminate-instances --instance-ids $IID   # PERMANENT, deletes the disk

# wait until genuinely ready (not merely "running")
aws ec2 wait instance-status-ok --instance-ids $IID

# current public IP
aws ec2 describe-instances --instance-ids $IID `
  --query "Reservations[0].Instances[0].PublicIpAddress" --output text

# everything you own, at a glance
aws ec2 describe-instances `
  --query 'Reservations[].Instances[].{ID:InstanceId,Type:InstanceType,State:State.Name,IP:PublicIpAddress}' `
  --output table

# what else is still billing?
aws ec2 describe-volumes   --query 'Volumes[].{ID:VolumeId,GiB:Size,State:State}' --output table
aws ec2 describe-addresses --query 'Addresses[].{IP:PublicIp,Assoc:InstanceId}'   --output table
aws ec2 describe-snapshots --owner-ids $ACCT --query 'Snapshots[].{ID:SnapshotId,GiB:VolumeSize}' --output table

# re-allow SSH after your home IP changes
$myip = (Invoke-RestMethod https://checkip.amazonaws.com).Trim()
aws ec2 authorize-security-group-ingress --group-id $SG `
  --protocol tcp --port 22 --cidr "$myip/32"
```

```bash
# --- on the instance ---
free -h                                 # memory + swap
df -h /                                 # disk
lsblk                                   # block devices (before growpart!)
systemctl is-active <svc>               # is it up?
sudo journalctl -u <svc> -n 50 --no-pager   # why did it die?
ss -tlnp                                # what's listening
```

---

## Hard-won lessons

1. **`icacls` needs the full `machine\user`** — Git Bash's `whoami` and
   PowerShell's `$env:USERNAME` both give you only half of it.
2. **`export MSYS_NO_PATHCONV=1`** before any Git Bash command with a `/leading/path`
   argument, or it silently becomes `C:/Program Files/Git/...`.
3. **`remote.SSH.remotePlatform` must be set**, or VS Code tries `powershell` on Linux.
4. **`Address already in use` = success**, not failure.
5. **Never check a socket in the same breath as starting it.**
6. **Know which machine your prompt is on** before pasting commands.
7. **Add swap on any 1 GB instance** before pip installing anything.
8. **Run `lsblk` before `growpart`** — device names differ between Nitro and Xen.
9. **Never hardcode an AMI id.** Resolve it from an SSM parameter; they change constantly.
10. **The public IP changes on stop/start** (a plain `reboot` keeps it). If SSH
    times out, check the IP before anything else.
11. **`terminate` deletes your disk.** `stop` keeps it. Learn this on a box you
    don't care about.
12. **CloudTrail tells you who did what** — `root` means the console, your IAM
    username means a script.
13. **Set the budget alarm on day one**, not after the first surprise bill.
14. **Provision 20 GB, not 8.** The default is smaller than a working dev box needs.
