# BBRound Trippin’
## Open files with the ```bbedit``` cli-tool from the server.

Use it in a very similar way as you would with local files:


- ```Server_Prompt$ bbedit file.txt``` --> _opens file.txt._

- ```Server_Prompt$ bbedit .``` --> _opens BBEdit’s sftp browser to the current directory._

- ```Server_Prompt$ bbedit ~``` --> _opens BBEdit’s sftp browser to the home directory._

- ```Server_Prompt$ bbedit /etc``` --> _opens BBEdit’s sftp browser to the etc directory._

Including pipes and flags:

- ```Server_Prompt$ man seq | col -b | bbedit --view-top -m "unix-man-page"``` --> _Opens the manual for __seq__ in BBedit with the language set to Unix man page and the window scrolled to the top._

## Install and configure.

1) Copy shell script __bbedit__ to a server you can configure.

1) Place in a dir accessable by the users __PATH__. Such as __/usr/local/bin__.

1) Make the script executable.

1) Set env variable __BB_user__ to your username on the client mac.

1) Set env variable __BB_host__ to the hostname of the client mac. 

## Admissions, assumptions, concerns, and more configurations.

I am not a security expert, so weigh my advice and the use of this script against that. BBRound Trippin’ exploits remote access to the server and to your client.

There are a lot of scripts like this in forums on the internet, and probably more on GitHub as well. The truth is I worry a little bit about how people are using them and if they are putting enough effort in isolating their credentials.

I’d like to offer a setup that is at least reasonable, if not diligent.

- I’m assuming you have access to configure SSH on the server, and your client mac of course.

- I’m using this for a local server. So I’m not going to cover how to call back to your mac client from across the internet, aka some sort of dynamic dns.

- I’m also betting you know a little about SSH key authentication. 

- Finally, you should be familiar with the command line, and setting env variables.


### The breakdown.
1) Your client computer is a mac (with BBEdit installed) opening an SSH session with a Unix style server.

1) When you open a file with```Server_Prompt$ bbedit file_name.txt``` the script sends that command and properly formatted parameters back to your mac via SSH.

1) Now BBEdit opens __file_name.txt__ via it’s own SSH (sftp) connections, leaving you with two mac to server connections; one from your terminal, the other from BBEdit.

It’s that second step 🤨; keep an eye on it.

Here is what the command would look like typed out manually:
```bash
Server_Prompt$ ssh userC@my_macintosh.local bbedit "sftp://userS@my_server.local"
```

### Your mac is your safe place.

Even hardcoding your username can be avoided. Though your hostname/ip can be surmised on the server I think it’s more flexible to declare it in a variable on your mac, client side, and set it as part of your SSH enviroment.

That is what __BB\_user__ and __BB\_host__ are for.

#### First declare them locally:

__~/.bash_profile__

```bash
export BB_user="userC"
export BB_host="$(hostname)"
```
There are a number of ways to setup BB\_host. On most macs “$(hostname)” will expand to something like my\_macintosh.local. You can configure your hostname in the Sharing preference panel. This is great because it avoids using your mac’s ip, which is probably changing all the time, even on your home network. 

You also might set a domain like my\_registered\_domain.com if you want to point back at your mac from outside your network.

#### Send the variables to your SSH session:

You have to first configure the server.
Add this line to your __/etc/ssh/sshd_config__ on the server:

```
AcceptEnv BB_user BB_host
```

Lets setup a __~.ssh/config__ on your mac

```
Host the_server
	HostName my_server.local
	User userS
	SendEnv BB_user BB_host
```

Now when you ```ssh the_server```it will add __BB\_user__ and __BB\_host__ to that sessions environment.

#### Authentication and the SSH Agent:

[Don’t use a password](https://medium.com/macoclock/set-up-ssh-on-macos-89e8354d8b63
), and have only one set of keys.

You should have an SSH key pair set up in order to login to your server.

You don’t want to make a set of keys on the server, but you have to make an SSH connection back to your mac. You can use Agent Forwarding to to safely pass your private key from your mac to your server and back to your mac.

Add your public key to __~/.ssh/authorized_keys__

Set up your __~.ssh/config__ like so.

```
Host the_server
	HostName my_server.local
	User userS
	SendEnv BB_user BB_host
	AddKeysToAgent yes
	IdentityFile ~/.ssh/<private_key>
	ForwardAgent yes
```

That’s it, everything should work and you haven’t hard coded any sensitive information about your mac on the server.