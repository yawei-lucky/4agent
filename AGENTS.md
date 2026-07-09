# Priority SSH Guidance

This is the first rule for connecting from this Windows Codex session to remote project machines.

- Use non-interactive, fail-fast SSH commands so Codex does not hang on prompts.
- Prefer `cmd /c ssh ...` from Windows Codex to reduce PowerShell quoting issues.
- Always add SSH batch and timeout options for remote checks and commands:
  - `-o BatchMode=yes`
  - `-o ConnectTimeout=8`
  - `-o ServerAliveInterval=10`
  - `-o ServerAliveCountMax=2`
- For blackbox, prefer the explicit private-key direct connection from the blackbox notes instead of relying on an interactive alias.

Primary blackbox form:

```bash
cmd /c ssh -o BatchMode=yes -o ConnectTimeout=8 -o ServerAliveInterval=10 -o ServerAliveCountMax=2 -i C:\Users\Yawei\.ssh\blackbox_ed25519 ubuntu@100.66.216.60 "cd /home/ubuntu/remoteAD && git status"
```

For ds_office and other SSH-config hosts, still use the same non-interactive timeout options:

```bash
cmd /c ssh -o BatchMode=yes -o ConnectTimeout=8 -o ServerAliveInterval=10 -o ServerAliveCountMax=2 ds_office "cd /home/stf3/remoteAD && git status"
```

If an SSH command fails because authentication is not available in batch mode, stop and report the failure instead of retrying in a way that waits for an interactive password prompt.

## Environment Scope Rules

Before running any command, classify the execution scope first and state it in updates/results when it matters.

Allowed scopes:

```text
Windows relay
  The current Windows Codex desktop shell. Use only for AGENTS.md, SSH wrappers, browser/client checks, and coordination.

Remote Linux over SSH
  A command wrapped by Windows `cmd /c ssh ... "remote command"` and executed on ds_office, blackbox, or another remote host.

Local Linux direct
  A command executed directly in a Linux shell where Codex is already attached to that Linux environment. Do not add SSH or extra relay layers in this case.

WSL Ubuntu direct
  A command executed inside this Windows machine's WSL Ubuntu. Treat this as local Linux for cloud-side work when the active shell is already WSL.

Browser/client-side check
  Browser access or frontend validation only.
```

Do not confuse these scopes:

- If the active shell is Windows, project commands for ds_office/blackbox/WSL must go through the correct SSH or WSL entry point.
- If the active shell is already the intended Linux environment, run the command directly there; do not wrap it in SSH just because other workflows use SSH.
- Before build, deploy, Git branch, LiveKit, GStreamer, RTP, service, or push commands, verify the target using `pwd`, `hostname`, and when useful `uname -a` or `git remote -v`.
- Never run Linux project scripts directly from local Windows paths.

## Git Management

- If there are committed local changes on a project branch, push them directly to the corresponding remote branch.
- If new work is committed during a task, push the branch after the commit succeeds unless the user explicitly says not to push.
- Run Git and GitHub authentication commands on the corresponding remote Linux project machine, not from local Windows project paths, unless the repository being managed is intentionally local to Windows.
- If push fails because GitHub CLI, credentials, or Git permissions are missing, set up `gh` on that remote Linux host:

```bash
sudo apt install gh
gh auth login
gh auth setup-git
```

Use these `gh auth login` choices:

```text
GitHub.com
HTTPS
Login with a web browser
```

After `gh auth setup-git` succeeds, retry the Git push.

### Current GitHub Auth And Rebase Note

Current known GitHub state:

- Non-sandbox `gh auth status` is working.
- GitHub HTTPS push can use the `gh` credential helper.
- If a push/auth command fails only because the sandbox blocks access to credentials, retry outside the sandbox with a scoped approval request instead of changing repository state.

Current alignment issue to remember:

- Local duplicate commits need to be aligned with the remote API commit.
- Intended order: commit this runbook/AGENTS update first, rebase onto remote commit `873eaef7`, then push.
- Before rebasing, inspect `git status`, current branch, upstream, and recent log. Do not discard user work.


### Missing Push Permission Rule

If Git push fails because permissions, credentials, or GitHub authentication are missing, directly set up GitHub CLI on the same Linux environment that owns the repository, then retry push.

Use this flow:

```bash
sudo apt install gh
gh auth login
gh auth setup-git
```

During `gh auth login`, choose:

```text
GitHub.com
HTTPS
Login with a web browser
```

After authentication succeeds, run `gh auth status`, verify `git remote -v`, and retry `git push`.
## Third-Party Fork Ownership

When working on a third-party official repository with local environment adaptations, treat the user's fork as the writable project repository.

Rules:

- Do not push local environment changes to the third-party official upstream repository.
- If the user has already forked the repository, switch the working repository to the user's fork and push there.
- If the user has not forked it yet and push is needed, create/use a fork under the user's GitHub account, then push to that fork.
- Prefer converting `origin` to the user's fork for active work. Keep the official repository only as `upstream` when upstream sync is needed.
- If the user explicitly says to keep only the fork as `origin`, remove or ignore the official remote for normal work.
- Before changing remotes, inspect `git remote -v`, current branch, and `git status`; do not discard local modifications.

HUGSIM-specific note:

```text
Repository: HUGSIM
User fork: yawei-lucky/HUGSIM
Writable remote: the user's fork
Policy: convert the local working repo so `origin` is the user's fork, then commit/push there.
```

Known HUGSIM local environment adaptation files that may be intentionally uncommitted or ready to commit when requested:

```text
M pixi.lock
M pixi.toml
?? pixi.toml.smoke-backup
```

These are local environment adaptation changes for the user's own fork, not changes to push to the third-party official repository.
# Project Guidance

## Core Rule

- This project is developed and operated on the corresponding remote Linux machines over SSH.
- Do not treat local Windows as the project development environment.
- Local Windows should only be used as a relay/control machine, for example to open terminals, SSH into hosts, transfer notes, or access browser pages.
- Before running build, deploy, network, RTP, WebRTC, LiveKit, or service commands, identify the target host and SSH into that host first.

## Current End-to-End Link

```text
ds_office vehicle side
  -> direct Ethernet RTP raw
blackbox
  -> 4G / Tailscale / WebRTC
WSL cloud side
  -> LiveKit + browser access
```

## 1. ds_office: Vehicle Simulator

```text
Host: ds_office
SSH: stf3@100.97.202.73
Repo: /home/stf3/remoteAD
Branch: codex/rtp-raw-video
Role: vehicle-rtp sender
Direct NIC: enp0s31f6
Direct IP: 192.168.50.2/24
Tailscale IP: 100.97.202.73
```

Preparation status:

```text
vehicle-rtp environment check OK
GStreamer / v4l2-ctl / rtpvrawpay / udpsink OK
```

Start real vehicle RTP raw sender on ds_office:

```bash
ssh stf3@100.97.202.73
cd /home/stf3/remoteAD
VEHICLE_RTP_HOST=192.168.50.10 \
bash stat_img_trans/webrtc/deploy/start_low_latency_role.sh vehicle-rtp
```

For simulated 6-channel data, use `videotestsrc` to send to `192.168.50.10:9301-9306`.

## 2. blackbox: Receiver + WebRTC Publisher

```text
Host: blackbox
SSH: ubuntu@100.66.216.60
Repo: /home/ubuntu/remoteAD
Branch: codex/rtp-raw-video
Role: blackbox-rtp receiver + LiveKit publisher
Direct eth0: 192.168.50.10/24
Tailscale IP: 100.66.216.60
4G interface: wwan0
```

Current network status:

```text
eth0 <-> ds_office: bidirectional ping OK
wwan0: 4G connected
4G operator: China Mobile HK
4G status: OK
Default internet route: wwan0
```

blackbox RTP receiver ports:

```text
cam1 -> UDP 9301
cam2 -> UDP 9302
cam4 -> UDP 9303
cam5 -> UDP 9304
cam6 -> UDP 9305
cam7 -> UDP 9306
```

Start blackbox on blackbox host:

```bash
ssh ubuntu@100.66.216.60
cd /home/ubuntu/remoteAD

LIVEKIT_URL=ws://100.101.151.68:7880 \
RTP_RAW_PORTS="9301 9302 9303 9304 9305 9306" \
RAW_CAMERA_NAMES="cam1 cam2 cam4 cam5 cam6 cam7" \
RTP_RAW_WIDTH=1280 \
RTP_RAW_HEIGHT=720 \
RTP_RAW_FPS=15 \
RTP_RAW_FORMAT=I420 \
LIVEKIT_VIDEO_CODEC=h264 \
FPS=15 \
bash stat_img_trans/webrtc/deploy/start_low_latency_role.sh blackbox-rtp
```

## 3. WSL: Cloud LiveKit + Browser Entry

```text
Host: WSL Ubuntu on Windows
Tailscale IP: 100.101.151.68
Role: cloud / LiveKit server / token server / frontend
Repo: ~/remoteAD_rtp_raw_codex
```

Start cloud side inside WSL Ubuntu, not from Windows PowerShell as a local project:

```bash
cd ~/remoteAD_rtp_raw_codex

PUBLIC_IP=100.101.151.68 \
bash stat_img_trans/webrtc/deploy/start_low_latency_role.sh cloud
```

Browser entry:

```text
http://100.101.151.68:8765/frontend/cameras_webrtc.html
```

## Current Preparation Status

```text
ds_office vehicle environment: OK
blackbox blackbox-rtp environment: OK
blackbox 4G: OK
blackbox eth0 and ds_office: connected
WSL Tailscale cloud IP: 100.101.151.68
```

## Startup Order

```text
1. Start WSL cloud.
2. Start blackbox blackbox-rtp.
3. Start ds_office vehicle-rtp or videotestsrc simulation.
4. Open browser and verify tracks: 6/6, codec: H264, packets increasing.
```

## Operational Discipline

- Run ds_office commands only after SSH into `stf3@100.97.202.73`.
- Run blackbox commands only after SSH into `ubuntu@100.66.216.60`.
- Run cloud commands inside WSL Ubuntu for `~/remoteAD_rtp_raw_codex`.
- Do not run Linux project scripts directly from local Windows paths.
- Use Windows only as an access point for SSH, browser viewing, and coordination.

## SSH Access

The following SSH aliases are configured and have already been tested. Passwordless SSH works. Prefer these aliases instead of spelling out full user and host pairs.

```sshconfig
Host ds_office
  HostName 100.97.202.73
  User stf3

Host vehicle_dg
  HostName 100.64.64.122
  User cityu02

Host blackbox
  HostName 100.66.216.60
  User ubuntu

Host blackbox_wlan
  HostName 100.66.216.60
  User ubuntu
  IdentityFile C:\Users\Yawei\.ssh\blackbox_ed25519
  IdentitiesOnly yes

Host shidi
  HostName 100.116.66.57
  User yawei

Host kylin
  HostName 100.68.76.4
  User kylin

Host carla
  HostName 100.68.223.102
  User testuser

Host aliyun
  HostName 8.129.91.23
  User root
  Port 22
```

Use examples:

```bash
ssh ds_office
ssh blackbox
ssh blackbox_wlan
ssh vehicle_dg
ssh shidi
ssh kylin
ssh carla
ssh aliyun
```

## Current Codex Connection Model

This Codex session is running on Windows desktop and reaches the project machines indirectly through Windows SSH.

```text
Codex -> PowerShell/cmd -> Windows ssh.exe -> remote Linux bash
```

Prefer using `cmd /c ssh ...` for remote commands when issuing commands from this Windows Codex session, because it reduces PowerShell quoting and escaping issues.

Long-term, the most stable approach is to open a Codex SSH connection directly into the target project host, so the execution path becomes:

```text
Codex -> remote bash
```

### blackbox Connection

```bash
ssh -i C:\Users\Yawei\.ssh\blackbox_ed25519 ubuntu@100.66.216.60
```

Connection path:

```text
Windows Codex
  -> Windows ssh.exe
  -> Tailscale 100.66.216.60
  -> blackbox /home/ubuntu/remoteAD
```

blackbox details:

```text
Host: blackbox
User: ubuntu
Tailscale IP: 100.66.216.60
Repo: /home/ubuntu/remoteAD
Main branch: codex/rtp-raw-video
Role: blackbox-rtp receiver + LiveKit publisher
```

Example from Windows Codex:

```bash
cmd /c ssh -i C:\Users\Yawei\.ssh\blackbox_ed25519 ubuntu@100.66.216.60 "cd /home/ubuntu/remoteAD && git status"
```

### ds_office Connection

Windows SSH config includes:

```sshconfig
Host ds_office
  HostName 100.97.202.73
  User stf3
```

Use:

```bash
ssh ds_office
```

Equivalent full command:

```bash
ssh stf3@100.97.202.73
```

ds_office details:

```text
Host: ds_office
User: stf3
Tailscale IP: 100.97.202.73
Repo: /home/stf3/remoteAD
Main branch: codex/rtp-raw-video
Role: vehicle simulator / vehicle-rtp sender
```

Example from Windows Codex:

```bash
cmd /c ssh ds_office "cd /home/stf3/remoteAD && git status"
```

### WSL Connection

WSL is Ubuntu running inside this Windows machine. It can be reached through the previous Windows-to-WSL SSH port mapping:

```bash
ssh -p 2222 -i C:\Users\Yawei\.ssh\blackbox_ed25519 yawei@172.30.236.201
```

WSL now also has Tailscale. If the WSL SSH service is running, it can also be reached via:

```bash
ssh yawei@100.101.151.68
```

WSL details:

```text
Host: WSL Ubuntu on Windows
User: yawei
Tailscale IP: 100.101.151.68
Repo: /home/yawei/remoteAD_rtp_raw_codex
Role: cloud / LiveKit server / frontend
```

When operating cloud-side scripts, run them inside WSL Ubuntu under `/home/yawei/remoteAD_rtp_raw_codex`, not from local Windows project paths.









