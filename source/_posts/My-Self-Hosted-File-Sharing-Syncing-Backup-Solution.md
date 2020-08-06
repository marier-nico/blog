---
title: My Self-Hosted File Sharing/Syncing/Backup Solution
date: 2020-08-02 19:40:28
tags:
  - backup
  - sync
  - file-share
  - self-hosted
---

## Motivations

I'm sure most people reading this are well aware that backups are absolutely crucial to recover data in the case of a system failure or ransomware attack, so the motivations behind a robust backup solution should be obvious.

For file sharing and syncing, it's mostly a matter of convenience. I have a laptop I used for university (before the pandemic), a desktop, and a phone, and I don't like manually copying files between devices whenever I need them. Certainly, after having used a commercial solution like Dropbox, it's hard to go back!

The problem with commercial solutions, however, is that they're fairly expensive compared to a self-hosted option. For example, the cheapest paid option for Dropbox is $12.99 CAD per month, but running a small home server costs $3.72 CAD per month in power costs (assuming an average power draw of 100W and a cost of $0.0588/kWh). Besides the price, privacy and control over your own data might be other reasons to lead you to choose a self-hosted approach.

## Quick Overview

The way I decided to do it is quite simple and only combines a few things : [Syncthing](https://syncthing.net/) for file sync, [Filestash](https://www.filestash.app/) for file sharing, with [Restic](https://restic.net/) and [Backblaze B2](https://www.backblaze.com/b2/cloud-storage.html) for backups.

Syncthing is used to sync files between all my devices (and one server, to allow for sharing in any situation and more reliable backups) and it does a great job. I highly recommend reading more into it, but it's everything I could hope for to sync files.

Filestash allows viewing files from a nice web interface as well as sharing files with links and even collaborating on documents with OnlyOffice.

Restic seemed to be a good choice because it's gaining a lot of traction lately and natively supports Backblaze B2, even if it's quite slow compared to [Borg](https://www.borgbackup.org/).

Backblaze B2 is hard to beat in terms of pricing while still offering a reliable service. Certainly, distributing backups as much as possible is a good idea, but this is a good starting point to at least have one off-site backup.

## Syncthing

### Installation

Installing Syncthing is super easy because it's packaged for a lot of linux distributions. It's also on the Google Play Store, if syncing with an Android phone is required.

#### Arch Linux
```
# pacman -S syncthing
```

#### Ubuntu
```
# apt install syncthing
```

### Configuration

Configuring Syncthing is a bit trickier than installing it, but it's still not too difficult. The web GUI can be accessed at `http://localhost:8384`. For a quick guide to using the GUI, see the Syncthing docs [here](https://docs.syncthing.net/intro/gui.html). When installing Syncthing on a headless system, like a server, an SSH tunnel can be used to access the remote web GUI instead of exposing it (which isn't really necessary for normal operation).

#### Creating an SSH tunnel
```
$ ssh -L 9999:localhost:8384 you@example.com
```

In this example, the Syncthing GUI from the host `example.com` will be available at `http://localhost:9999`.

At this point, you should add all the folders you want to share to Syncthing, add all your devices, and share the folders with all your devices. To do that, follow the Syncthing documentation.

## Filestash

### Installation

The only way to install Filestash is with Docker, if you want to use the free hobby license (but there are paid options if you don't want to host it yourself). The steps to install it are documented [here](https://www.filestash.app/docs/install-and-upgrade/) and are quite easy to follow. I was already using Docker Compose, so it was just a matter of dropping the provided service definitions into my own compose file (and updating the format of the provided configuration to version 3). The only other difference with the suggested installation method in my configuration was the use of [Caddy](https://caddyserver.com/v2) instead of Nginx.

### Configuration

Configuration isn't really documented so well, but that's fine because it's all very intuitive. The tweaks I made were to change the `Host` value to my FQDN for Filestash, to switch the `Editor` to vim, to remove the `Fork button` (it looks a bit obnoxious on the login page). Also, I didn't enable the Syncthing integration because I don't want to make that available through Filestash directly.

After Filestash itself is configured, at least one backend has to be configured to be able to access files. I decided to use the SFTP backend, because I wanted to access local files (managed by Syncthing) through Filestash. To do that, I used the `atmoz/sftp:alpine` image to have an SFTP server. I then bind-mounted my Syncthing-managed folders into the SFTP container, which made them available to Filesatsh.

A full `docker-compose.yml` configuration is shown below.

```yml
# docker-compose.yml

version: "3"

services:
  filestash:
    image: machines/filestash
    restart: unless-stopped
    environment:
      APPLICATION_URL: "files.example.com"
      ONLYOFFICE_URL: "http://onlyoffice"
    ports:
      - 8334:8334

  onlyoffice:
    image: onlyoffice/documentserver
    restart: unless-stopped

  sftp:
    image: atmoz/sftp:alpine
    restart: unless-stopped
    volumes:
      - /home/fileshare/syncthing:/home/fileshare
      - ./sftp/users.conf:/etc/sftp/users.conf:ro
```

There are a few important considerations for all this to work seamlessly. First of all, since the `sftp` server uses a `chroot`, all the paths leading up to the folder that is mounted inside it must be owned by `root`. In this example, `/home`, `/home/fileshare` and `/home/fileshare/syncthing` all have to be owned by `root`. This can be very inconvenient if you're not using a dedicated user account for Syncthing. Then, the `sftp` server also has a configuration file for users, which contains a few important bits. Here is a full example.

```conf
# users.conf

fileshare:some-password:1005:1005
```

The username and password (first two fields) don't matter *too* much. The user with the given username will only have access to their home in the container, so files must be mounted in it. The two following fields are critical. They are the UID and GID that the SFTP user will have. These values *must* match the UID and GID from the user that owns the files which are bind-mounted into the container. If they don't match, Filestash won't be able to access any files mounted into the `sftp` container.

After having configured the `sftp` server, it's possible to add an `SFTP` source in Filestash. The only setting that is required is the `Hostname`, which should be set to the hostname of the `sftp` server. In the context of the `docker-compose.yml` file above, that would be simply `sftp`, because docker-compose puts all the containers in the same network by default.

With this backend configured, the Filestash instance should now prompt the user for credentials. If all went well, entering the same credentials as in `users.conf` should grant access to whatever is accessible inside the user's home in the `sftp` container.

## Backblaze B2

### Creating a Bucket

This is very simple, just use the Backblaze website to create your backup bucket. Though, keep its name in mind for the next step.

## Restic

### Installation

Once again, Restic is widely packaged by distributions, but the packaged version might not be recent enough to use some features that are really useful (on Ubuntu, for example), so for some distributions, it might be necessary to download the binary manually. The minimum version is (0.9.2), which allows using non-master keys for Backblaze B2 (using master keys for a backup script like we're making here is just a bad idea in general).

The important thing is that the binary is in the `$PATH` that your cron jobs execute with, which is usually `/bin` or `/usr/bin`. One way to do it is to download the binary to `/opt` and adding a symbolic link to `/usr/bin` with `ln -s /opt/restic /usr/bin/restic` (run as root).

#### Arch Linux
```
# pacman -S restic
```

#### Ubuntu

Follow the instructions [here](https://restic.readthedocs.io/en/stable/020_installation.html), there should be a link to the official binaries on GitHub.

### Configuration

For Restic, there isn't really anything to configure, but to use it, we'll need a script to run our backups. For reference, this is stored in `/home/fileshare/backup/backup.sh`.

```bash
#!/bin/bash

export B2_ACCOUNT_ID=<Key ID>
export B2_ACCOUNT_KEY=<Key Secret>

export RESTIC_PASSWORD=<Your backup password>
export RESTIC_REPOSITORY="b2:<Your B2 bucket name>"

# This is created manually because `/home/fileshare` is owned by `root` (see Filestash instructions)
export RESTIC_CACHE_DIR="/home/fileshare/backup/.cache"

backup_output=$(restic -v backup ~/syncthing 2>&1) # Here we backup `~/syncthing`, but that's really up to you
backup_result=$?

# Send a ping to healthchecks.io. I'm not going to go into this, but it's a good idea to have
# some way to know whether or not a backup ran and to get an alert if it didn't.
curl -m 10 --retry 5 https://hc-ping.com/<Check ID>

# This will keep a number of snapshots, but not too many. At some point, it's just not useful to keep them.
prune_output=$(restic -v forget --prune \
    --keep-daily 7 \
    --keep-weekly 5 \
    --keep-monthly 12 \
    --keep-yearly 3 2>&1)
prune_result=$?

# This checks that everything is fine with the snapshots after pruning. If this fails, it's important to
# quickly make a new backup, because your current ones might be useless.
check_output=$(restic -v check)
check_result=$?

# I only have this on my Ubuntu servers, because I want to update Restic (and I'll forget since it wasn't
# installed through the package manager).
restic self-update

# If any of the previous commands fails, an email is sent to my personal email containing the output
# from all the commands, which will be very useful if a backup ever fails.
global_result=$(echo "$backup_result + $prune_result + $check_result" | bc)
if [ $global_result -ne 0 ]; then
    curl --user "api:<Your API key>" "https://api.mailgun.net/v3/<Your mg domain>/messages" \
        -d from="<From Address>" \
        -d to="<To Address>" \
	-d subject="Syncthing backup failed on $(hostname). Backup: $backup_result. Prune: $prune_result" \
	-d html=$'Backup Output\n'"<pre>$backup_output</pre>"$'\n\nPrune Output\n'"<pre>$prune_output</pre>"$'\n\nCheck Output\n'"<pre>$check_output</pre>"
fi
```

Then, to run the script (which should be run on a server), a simple and easy tool is `cron`. You can edit your cron entries with `crontab -e`. The example below will run the script above every day at midnight.

```cron
m h  dom mon dow   command
0 0  *   *   *     /home/fileshare/backup/backup.sh >/dev/null 2>&1
```

## Recap

After completing all these steps, you will have installed : Syncthing to sync files between systems, Filestash to share them (with an SFTP server to make the link between Syncthing and Filestash) and Restic to create encrypted backups.

You will also have a Backblaze B2 bucket to upload your backups and a script which runs daily and makes a backup of Syncthing files.

Overall, this leaves you with convenient file syncing and sharing, along with at least two on-site backups (thanks to Syncthing with one device and a server) and one off-site (in Backblaze), which follows the 3-2-1 backup principle!