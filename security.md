# How I Hardened My Security

 I used SSH agent forwarding (`ForwardAgent yes`) to use the same SSH key to access my mac that allowed access to the server. While secure in the sense that a key is not saved on the server, forwarding your primary SSH agent exposes all loaded keys on that agent to anyone with `root` access on the remote server, and an unrestricted callback key would allow a compromised server to execute arbitrary commands on your Mac.

Once again I am not a security expert. You may want to use this setup or just some of it. You might use something like [tailscale](https://tailscale.com) or setup your own VPN rather then using a reverse SSH tunnel (see step 4). I would argue that steps 1-3 are best practices.

1. __Dedicated Callback Key:__ Separates `rtedit` authentication from your primary SSH identity.
2. __Command-Restricted Wrapper:__ Forces incoming callback connections to run *only* `bbedit` and blocks shell injection.
3. __Isolated ssh-agent Socket:__ Ensures remote servers only ever see the restricted callback key—never your primary keys.
4. __Reverse SSH Tunneling:__ Routes callbacks back to your Mac over your existing encrypted SSH connection with zero open ports on your router or firewall.

---

## Step 1: Generate Dedicated SSH Keys

Create two separate Ed25519 key pairs on your Mac so your primary `~/.ssh/id_ed25519` key never leaves your machine:

```sh
# 1. Restricted key used by remote servers to call back into BBEdit
ssh-keygen -t ed25519 -f ~/.ssh/id_ed25519_rtedit -C "rtedit-callback"

# 2. Unrestricted loopback key used strictly by Tests/test_rtedit (no passphrase for automated runs)
ssh-keygen -t ed25519 -f ~/.ssh/id_ed25519_rtedit_test -N "" -C "rtedit-loopback-test"
```

---

## Step 2: Create the Command-Restricted `bbedit` Wrapper

Create __bbedit-ssh-wrapper__ on your Mac, make it executable and available on your PATH. When tied to `restrict,command="..."` in __~/.ssh/authorized_keys__, OpenSSH ignores the remote client's requested command, places it in `$SSH_ORIGINAL_COMMAND`, and runs this script instead:

```sh
#!/bin/sh

# Reject empty commands or shell chaining/redirection characters
# Created to harden the security when using rtedit script.

export PATH="/usr/local/bin:/opt/homebrew/bin:$PATH"

case "$SSH_ORIGINAL_COMMAND" in
    "" | *[\;\&\|\`\$\(\)\<\>
]*)
        printf 'Blocked unsafe command\n' >&2
        exit 1
        ;;
    bbedit | "bbedit "*)
        # Execute bbedit while preserving stdin for piped commands
        exec /bin/sh -c "$SSH_ORIGINAL_COMMAND"
        ;;
    *)
        printf 'Only bbedit is permitted\n' >&2
        exit 1
        ;;
esac
```

---

## Step 3: Authorize Keys on Your Mac and Remote Servers

### 1. Client Mac
Append both public keys to __~/.ssh/authorized_keys__ on your Mac, scoping __id_ed25519_rtedit__ strictly to your wrapper:

```text
# Restricted callback key: can ONLY run bbedit via your wrapper
restrict,command="/Users/<YOUR_MAC_USER>/.local/bin/bbedit-ssh-wrapper" ssh-ed25519 <CONTENT_OF_id_ed25519_rtedit.pub>

# Loopback-only test key: never copy this public key to any remote server
ssh-ed25519 <CONTENT_OF_id_ed25519_rtedit_test.pub>
```

### 2. Remote Servers
Install *only* __id_ed25519_rtedit.pub__ on each remote server you want to use `rtedit` with:

```sh
ssh-copy-id -i ~/.ssh/id_ed25519_rtedit.pub <REMOTE_HOST>
```

*(Note: If you already use `ControlMaster` for `<REMOTE_HOST>`, pass `-f` to `ssh-copy-id` once so it appends the new key without skipping it.)*

---

## Step 4: Set Up an Isolated `ssh-agent` Socket on your Mac

Run a dedicated `ssh-agent` bound to __~/.ssh/rtedit-agent.sock__ via `launchd` so your primary macOS agent remains completely isolated.

### 1. Create __~/Library/LaunchAgents/com.user.ssh-agent-rtedit.plist__
Replace `<YOUR_MAC_USER>` with your macOS username:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://apple.com">
<plist version="1.0">
<dict>
    <key>Label</key>
    <string>com.user.ssh-agent-rtedit</string>
    <key>ProgramArguments</key>
    <array>
        <string>/usr/bin/ssh-agent</string>
        <string>-D</string>
        <string>-a</string>
        <string>/Users/<YOUR_MAC_USER>/.ssh/rtedit-agent.sock</string>
    </array>
    <key>RunAtLoad</key>
    <true/>
    <key>KeepAlive</key>
    <true/>
</dict>
</plist>
```

### 2. Start the Agent and Load Only id_ed25519_rtedit
```sh
launchctl bootstrap gui/$(id -u) ~/Library/LaunchAgents/com.user.ssh-agent-rtedit.plist
SSH_AUTH_SOCK="$HOME/.ssh/rtedit-agent.sock" ssh-add --apple-use-keychain ~/.ssh/id_ed25519_rtedit

# Ensure neither new key is loaded in your default agent
ssh-add -d ~/.ssh/id_ed25519_rtedit 2>/dev/null
ssh-add -d ~/.ssh/id_ed25519_rtedit_test 2>/dev/null
```

Verify separation:
* `SSH_AUTH_SOCK="$HOME/.ssh/rtedit-agent.sock" ssh-add -l` should show _only_ `rtedit-callback`.
* `ssh-add -l` should show _only_ your primary key(s).

---

## Step 5: Configure Reverse SSH Tunneling

Instead of opening port `22` on your home router or firewall, use `RemoteForward` to tunnel callbacks from each remote server back to your Mac over your existing SSH session. Edit __~/.ssh/config__ or __/root/.ssh/config__ to assign a unique loopback port (e.g., `2222`, `2223`) per server so simultaneous connections never collide.

### 1. Remote Server
On each remote server, map a `mac-callback` alias to its assigned reverse tunnel port:

```text
Host mac-callback
	HostName 127.0.0.1
	Port 2222
	StrictHostKeyChecking accept-new
```

*Ensure __/etc/ssh/sshd_config__ on the remote server permits `AcceptEnv BB_` and `AllowTcpForwarding yes`.)*

### 2. Mac Client
Point your remote hosts to `rtedit-agent.sock`, enable `RemoteForward`, and disable `ForwardAgent` globally under `Host *`:

```text
Host server-one
	HostName <SERVER_ONE_IP>
	User <REMOTE_USER>
	IdentityFile ~/.ssh/id_ed25519_rtedit
	IdentitiesOnly yes
	IdentityAgent ~/.ssh/rtedit-agent.sock
	ForwardAgent yes
	RemoteForward 2222 127.0.0.1:22
	SetEnv BB_USER=<YOUR_MAC_USER> BB_SSH_HOST=server-one BB_HOST=mac-callback
	SendEnv BB_USER BB_SSH_HOST BB_HOST

Host server-two
	HostName <SERVER_TWO_IP>
	User <REMOTE_USER>
	IdentityFile ~/.ssh/id_ed25519_rtedit
	IdentitiesOnly yes
	IdentityAgent ~/.ssh/rtedit-agent.sock
	ForwardAgent yes
	RemoteForward 2223 127.0.0.1:22
	SetEnv BB_USER=<YOUR_MAC_USER> BB_SSH_HOST=server-two BB_HOST=mac-callback
	SendEnv BB_USER BB_SSH_HOST BB_HOST

Host *
	UseKeychain yes
	AddKeysToAgent yes
	ForwardAgent no
	IdentityFile ~/.ssh/id_ed25519
```

---

## Step 6: Verify Security Controls

Run these three checks from your Mac after connecting to a configured server:

1. __Agent Isolation:__ Confirm the server sees only the callback key:
   ```sh
   ssh server-one 'ssh-add -l'
   # Expected: 256 SHA256:... rtedit-callback (ED25519)
   ```
2. __Command Restriction:__ Confirm arbitrary commands are blocked over the callback tunnel:
   ```sh
   ssh server-one 'ssh <YOUR_MAC_USER>@mac-callback whoami'
   # Expected: Only bbedit is permitted
   ```
3. __End-to-End File Editing:__ Confirm `rtedit` opens remote files cleanly in BBEdit:
   ```sh
   ssh -t server-one 'rtedit /etc/hosts'
   ```

---

## Step 7: Isolate Local Integration Tests

Because `id_ed25519_rtedit` is restricted to `bbedit`, `Tests/test_rtedit` uses `id_ed25519_rtedit_test` inside a temporary, self-cleaning `ssh-agent` socket (`/tmp/rtedit-test-agent.<PID>.sock`) with `IdentitiesOnly yes`. This ensures automated tests run over `127.0.0.1` without touching either your primary agent or `rtedit-agent.sock`.
