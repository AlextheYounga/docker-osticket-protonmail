# Docker OSTicket Help Desk With Proton Mail Via Dockerized ProtonMailBridge


- [Docker OSTicket Help Desk With Proton Mail Via Dockerized ProtonMailBridge](#docker-osticket-help-desk-with-proton-mail-via-dockerized-protonmailbridge)
	- [About](#about)
		- [Good to Know](#good-to-know)
	- [How It Works](#how-it-works)
	- [Run the Containers](#run-the-containers)
	- [OSTicket Configuration](#osticket-configuration)
		- [Remote Mailbox](#remote-mailbox)
		- [Outgoing (SMTP)](#outgoing-smtp)
	- [Proton Mail Bridge Configuration](#proton-mail-bridge-configuration)
	- [License](#license)


## About
This is a fork of the extremely helpful repo [Docker OSTicket](https://github.com/tiredofit/docker-osticket) by [tiredofit](https://github.com/tiredofit) that has been updated and adapted to use a Dockerized [Proton Mail Bridge CLI](https://proton.me/support/bridge-cli-guide) implementation in order to work with ProtonMail.

### Good to Know
- I have only tested this on MacOS, Debian Linux and RaspberryPi OS. 
- A similar setup can also work on MacOS, Proton Mail Bridge MacOS app, without needing the Proton Mail Bridge Docker container.

## How It Works
I use the [shenxn/protonmail-bridge-docker](https://github.com/shenxn/protonmail-bridge-docker) Docker image to launch the [Proton Mail Bridge CLI](https://proton.me/support/bridge-cli-guide) and create a network interface between the OSTicket app and the Mail Bridge.

By default, no mailports are exposed to the open network, and it is not necessary to do so. These ports only need to be available to the 
OSTicket container. 

## Run the Containers
```sh
cd osticket-protonmail
docker compose up -d
```

## OSTicket Configuration
Provided is an `.env.example` env file. You will still need to do some manual configuration in OSTicket Settings.

Go to Admin Panel -> Emails -> Default Email (Add new if necessary)

### Remote Mailbox
**Mailbox Settings**
- Hostname:	`protonmail-bridge`
- Port Number: `143`
- Mail Folder: INBOX
- Protocol: IMAP
- Authentication: Basic Authentication (Legacy)
  - This is where you will enter your Proton Mail key from the Mail Bridge

**Email Fetching**
- Status: Enable
- Fetch Frequency: 1 minute (or however frequently you would like)
- Emails per Fetch: (Up to you)
- Fetched Emails: Delete Emails (I thought this was a crazy idea at first but it actually is the best solution)


### Outgoing (SMTP)
- Hostname:	`protonmail-bridge`
- Port Number: `25`
- Authentication: Basic Authentication (Legacy)
  - This is where you will enter your Proton Mail key from the Mail Bridge

## Proton Mail Bridge Configuration
The best way I found to run the Proton Mail Bridge CLI on the server was using [tmux](https://github.com/tmux/tmux/wiki). You might have a better solution, but tmux will work and I can't be convinced it's a bad solution. I can attach to the running Mail Bridge at any time and see what is going on, see any logs, etc. 

```sh
sudo apt update
sudo apt install tmux
tmux new -s mailbridge
```

The [shenxn/protonmail-bridge-docker](https://github.com/shenxn/protonmail-bridge-docker) has some slight usabilty issues but nothing we can't fix. Out of the box, the image automatically starts the Proton Mail Bridge, which becomes a problem when you're running Docker in detached mode using `docker compose up -d` - as you'll never be able to attach to the running instance. 

So you will need to create a shell in the running protonmail-bridge Docker container.
```sh
# Create a tmux instance so we can run this in the background later
tmux new -s mailbridge
# Create a shell into the protonmail-bridge container
docker exec -it protonmail-bridge bash
```

From within the container, we need to kill all instances of Proton Mail Bridge.
```sh
sudo apt update
# Installing process tools and nano text editor 
sudo apt install -y nano procps
pkill bridge 
pkill protonmail
```

Now we will need to restart the Proton Mail Bridge CLI using the provided `entrypoint.sh` file shipped with the image. Feel free to inspect this file beforehand using the nano editor we installed earlier.
```sh
# Make file executable
chmod +x /protonmail/entrypoint.sh
# Move into the protonmail folder
cd protonmail
# Start proton mail bridge with the init arg
./entrypoint.sh init
```

This should start the Proton Mail Bridge CLI, where you can enter your login by typing `login`.

It will begin syncing your emails down. There are still some further configurations you will want to make before leaving.

**Ensure Proton Mail Bridge is running in Split Addressses Mode**

Type: `change mode` to show the settings about which mode it is running in. Ensure it is running on Split Addresses mode, or you will run into issues on OSTicket. 

You can always type `help` to see a list of all commands.

Grab your mail bridge encryption key from the CLI. You can always see this by typing: `info`. This is the key you will use to authenticate on OSTicket under `Authentication: Basic Authentication (Legacy)`. (See [OSTicket Configuration](#osticket-configuration) section for more info)

You will want to leave this running. To detach the tmux shell, use `CTRL + B` (then depress both of those keys) then hit `D` to detach. This will keep that shell running in the background... forever.

To reattach, use `tmux a -t mailbridge`


## License
MIT. See [LICENSE](LICENSE) for more details.
