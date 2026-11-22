# BBRound Trippin’
## Open files with the ```bbedit``` cli-tool from the server.

[BBEdit](https://www.barebones.com/products/bbedit/) is a stalwart commercial text editor for the Macintosh computer.

It offers a [Command-line tool](https://www.barebones.com/products/bbedit/benefitscommand.html#commandline): __bbedit__. This script invokes it over SSH.

Use it in a very similar way as you would with local files:

- ```Server_Prompt$ rtedit file.txt``` --> _opens file.txt._

- ```Server_Prompt$ rtedit .``` --> _opens BBEdit’s sftp browser to the current directory._

- ```Server_Prompt$ rtedit ~``` --> _opens BBEdit’s sftp browser to the home directory._

- ```Server_Prompt$ rtedit /etc``` --> _opens BBEdit’s sftp browser to the etc directory._

Including pipes and flags:

- ```Server_Prompt$ man seq | col -b | rtedit --view-top -m unix-man-page``` --> _Opens the manual for __seq__ in BBedit with the language set to Unix man page and the window scrolled to the top._

## Install and configure.

1. Copy shell script __rtedit__ to a server you can configure. *If you rename the script bbedit the command will look exactly like their local version*

1. Place in a dir accessable by the users __PATH__. Such as __/usr/local/bin__.

1. Make the script executable.

1. Set env variable __BB_USER__ to your username on the client mac.

1. Set env variable __BB_HOST__ to the hostname of the client mac. -*This is optional* 

## Admissions, assumptions, concerns, and more configurations.

I am not a security expert, so weigh my advice and the use of this script accordingly. BBRound Trippin’ exploits remote access to the server and to your client.

There are a lot of scripts like this in forums on the internet, and probably more on GitHub as well. The truth is I worry a little bit about how people are using them and if they are putting enough effort in isolating the users credentials.

I’d like to offer a setup that is at least reasonable, if not diligent.

- I’m assuming you have access to configure SSH on the server, and your client mac of course.

- I’m not going to cover how to call back to your mac client from across the internet or navigate your local firewall, router, vpn etc.

- I’m also betting you know a little about SSH key authentication. 

- Finally, you should be familiar with the command line, and setting env variables.

### The breakdown.
1) Your client computer is a mac (with BBEdit installed) opening an SSH session with a Unix style server.

1) When you open a file with```Server_Prompt$ rtedit file_name.txt``` the script sends a properly formatted command with parameters back to your mac via SSH.

1) Now BBEdit opens __file_name.txt__ via it’s own SSH (sftp) connections, leaving you with two mac to server connections; one from your terminal, the other from BBEdit.

It’s that second step 🤨; keep an eye on it.

Here is what the command would look like typed out manually:
```bash
Server_Prompt$ ssh userC@my_macintosh.local bbedit "sftp://userS@my_server.local"
```

## Setting up your BB environment.
Here are a few options in setting up __BB\_USER__ and __BB\_HOST__ with various tradeoffs.

### Declaring them on the server.
__~/.bash_profile__

```bash
# SSH environment
if [ "$SSH_CONNECTION" ]
then
  export BB_USER="userC"
  export BB_HOST="mymacurl.net" 
fi
```

### Your mac is your safe place, maybe.

You can try and configure as much as possible in your local SSH config and shell environment. This will usually mean having control of the server outside your user environment, like say on your own vps.

This primarily means setting __BB\_USER__ and __BB\_HOST__ localling rather than on the server.

Your hostname/ip can be surmised on from your SSH connection, so __BB\_HOST__ is optional, even though I set it manually in all my examples.

#### First declare them locally:
__~/.bash_profile__

```bash
# Here be rtedit variables for ssh connections
export BB_USER="userC"
export BB_HOST="$(hostname)"	
```

There are a number of ways to setup BB\_HOST. On most macs “$(hostname)” will expand to something like my\_macintosh.local. You can configure your hostname in the Sharing preference panel. This is great because it avoids using your mac’s ip, which is probably changing all the time.

You also might set a domain like back\_to\_me\_domain.net if you want to point back at your mac from outside your network.

#### Send the variables to your SSH session:

You have to first configure the server.
Add this line to your __/etc/ssh/sshd_config__ on the server:

```
AcceptEnv BB_USER BB_HOST
```

> [!TIP]
> Using a drop-in config file is in __/etc/ssh/sshd_config.d/__ is usually preferred over editing __sshd_config__ directly.
> Remember that ssh uses a first come first serve model for it’s configurations. So if you load a `AcceptEnv` setting after another (such as the system default) you won’t get the changes.

Lets setup a __~.ssh/config__ on your mac
```
Host the_server
  HostName my_server.local
  User userS
  SendEnv BB_USER BB_HOST
```

Now when you ```ssh the_server```it will add __BB\_USER__ and __BB\_HOST__ to that sessions environment.

This does increase your server’s surface area for attack. I think this is reasonable as long as you are not leaving the door wide open and declaring the specific variables you will allow. The upside is that you haven’t put any info about your Mac on the server. It lives and dies with that specific SSH connection. Tradeoffs, security is hard.

#### An alternative to declaring __BB\_USER__ and __BB\_HOST__ in your local environment.

OpenSSH added the configuration `SetEnv` in late 2018. You can check `man ssh_config` to see if your version supports it. It’s a better option as you can configure BBRound Trippn’ in __~.ssh/config__ on a host by host basis.

```
Host local_server
  HostName my_server.local
  User userS
  SetEnv BB_USER=userC BB_HOST=my_macintosh.local
  SendEnv BB_USER BB_HOST

Host remote_server
  HostName my_server.net
  User userRS
  SetEnv BB_USER=userC BB_HOST=back_to_me_domain.net
  SendEnv BB_USER BB_HOST
```

#### Authentication and the SSH Agent:

[Don’t use a password](https://medium.com/macoclock/set-up-ssh-on-macos-89e8354d8b63
), and have only one set of keys.

You should have an SSH key pair set up in order to login to your server.

You don’t want to make a set of keys on the server, but you have to make an SSH connection back to your mac. You can use Agent Forwarding to to _safely_ pass your private key from your mac to your server and back to your mac.

Add your public key to __~/.ssh/authorized_keys__

Set up your __~.ssh/config__ like so.

```
Host the_server
  HostName my_server.local
  User userS
  SetEnv BB_USER=userC BB_HOST=my_macintosh.local
  SendEnv BB_USER BB_HOST
  AddKeysToAgent yes
  IdentityFile ~/.ssh/<private_key>
  ForwardAgent yes
```

> [!WARNING]
> Using `ForwardAgent yes` with your primary SSH agent exposes all loaded keys on that agent to anyone with `root` access on the remote server. By default you are likely running one agent on your Mac, so all your SSH keys would be accessable if that server were compromised.
> 
> For a little guidance, see [How I Hardened My Security or Don't get your keys stolen!](security.md).

#### The other BB variable.

I added a variable called _BB\_SSH\_HOST_, it is optional.
By default the script figures the server_name@server_host string on the server. You use _BB\_SSH\_HOST_ if you want _BBEdit_ to use a specific SSH host to call back.

You might want to configure a host to use [_ssh multiplexing_](https://www.cyberciti.biz/faq/linux-unix-reuse-openssh-connection/) so that after you initially open a file, subsequent saves can be faster.

With a configuration something like this:
```
Host remote_server
  HostName my_server.net
  User userRS
  SetEnv BB_USER=userC BB_SSH_HOST=remote_server
  SendEnv BB_USER BB_SSH_HOST
  ControlPath ~/.ssh/control-socket-%C
  ControlMaster auto
  ControlPersist 5m
```

## Testing.

The repository includes a test suite  __Tests/test_rtedit__ that runs a full round-trip check over a local SSH loopback connection (`127.0.0.1`).

### Prerequisites.

Before running the test suite on your Mac, make sure your local environment allows the script to loop back into itself:

1. __Enable Remote Login:__ Go to **System Settings > General > Sharing__ and turn on __Remote Login** (SSH).
1. __Allow Environment Variables:__ Rather than editing __/etc/ssh/sshd_config__ directly, add a drop-in configuration file at __/etc/ssh/sshd_config.d/11-rtedit.conf__ to preserve macOS's default locale variables alongside the `BB_*` variables:
   ```text
   AcceptEnv LANG LC_* BB_USER BB_HOST BB_SSH_HOST
   ```
   *Toggle Remote Login off and back on—or restart `sshd`—after creating the file.*
1. __SSH Key & Agent:__ Add your local SSH public key to __~/.ssh/authorized_keys__ and ensure your key is loaded in your SSH agent `ssh-add` for passwordless loopback authentication.
  
    Optionally, to isolate test authentication from your primary agent, generate a dedicated test key at __~/.ssh/id_ed25519_rtedit_test__ or and append its public key to __~/.ssh/authorized_keys__:
   
     ```sh
     ssh-keygen -t ed25519 -f ~/.ssh/id_ed25519_rtedit_test
     cat ~/.ssh/id_ed25519_rtedit_test.pub >> ~/.ssh/authorized_keys
     ```
   
1. __BBEdit CLI Tool:__ Confirm `bbedit` is installed and accessible in your `PATH`.

### Running the tests.

From the root of the repository, run:

```sh
./Tests/test_rtedit
```

The script dynamically generates a temporary SSH config (__Tests/ssh_config.run__), runs 12 tests covering file opening, directory browsing, piping, flag combinations, and permission/error handling, prints a dated summary, and cleans up after itself.

> [!NOTE]
> Running the test suite will open several temporary documents and SFTP browser windows in BBEdit on your Mac. This is normal, don’t fret.

