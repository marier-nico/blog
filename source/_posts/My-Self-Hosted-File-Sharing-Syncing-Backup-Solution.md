---
title: My Self-Hosted File Sharing/Syncing/Backup Solution
date: 2020-08-02 19:40:28
tags:
  - backup
  - sync
  - file-share
  - self-hosted
---

# Updated 2020-08-23

After having used Restic and Backblaze B2 for some time, I started to see some issues that were not immediately obvious while planning out my solution, which made me reconsider using them as my backup solution. The relevant parts of this post have been updated with what I'm currently using.

The main issue with Restic is that it's very slow, but it's also inconvenient that it doesn't support compression (at the time of writing). This isn't such a massive problem on its own, but it's definitely inconvenient. Having used Borg Backup in the past, it's hard to accept anything that isn't as fast.

For me, Backblaze was even more problematic than Restic, mostly due to the fact that it's not possible to choose the geographic location of a backup after account creation. The goal with this was to come up with a bullet-proof solution for backups, but not being able to easily replicate data in two regions makes this impossible.

Also, Backblaze B2 doesn't have a [Terraform](https://www.terraform.io/) provider, which is more of an inconvenience rather than a deal-breaker, but that certainly factored into the decision to migrate out of Backblaze B2.

# Updated 2021-02-22

I now no longer use this method for backups. Having moved away from Android, I could no longer use syncthing. My current solution is to use [Tresorit](https://tresorit.com/individuals), which is an end-to-end encrypted cloud storage / file sharing service. Tresorit also handles data under Swiss privacy laws and uses non-convergent crypto (which means your data can't be matched to other users' data).

They also support every major platform (Windows, Mac, Linux, iOS, Android), which is the main reason I didn't go with [Sync.com](https://www.sync.com/).

For all the features offered and the privacy guarantees, I find the price to be quite reasonable. I'll leave the rest of this article up in case anyone was interested in replicating my previous setup.

## Motivations

I'm sure most people reading this are well aware that backups are absolutely crucial to recover data in the case of a system failure or ransomware attack, so the motivations behind a robust backup solution should be obvious.

For file sharing and syncing, it's mostly a matter of convenience. I have a laptop I used for university (before the pandemic), a desktop, and a phone, and I don't like manually copying files between devices whenever I need them. Certainly, after having used a commercial solution like Dropbox, it's hard to go back!

The problem with commercial solutions, however, is that they're fairly expensive compared to a self-hosted option. For example, the cheapest paid option for Dropbox is $12.99 CAD per month, but running a small home server costs $3.72 CAD per month in power (assuming an average power draw of 100W and a cost of $0.0588/kWh). Besides the price, privacy and control over your own data might be other reasons to lead you to choose a self-hosted approach.

## Quick Overview

The way I decided to do it is quite simple and only combines a few things : [Syncthing](https://syncthing.net/) for file sync, [Filestash](https://www.filestash.app/) for file sharing, with [BorgBackup](https://www.borgbackup.org/) and [Amazon S3](https://aws.amazon.com/s3/) for backups.

Syncthing is used to sync files between all my devices (and one server, to allow for sharing in any situation and more reliable backups) and it does a great job. I highly recommend reading more into it. It's everything I could hope for to sync files.

Filestash allows viewing files from a nice web interface as well as sharing files with links and even collaborating on documents with OnlyOffice.

BorgBackup is quick, easy to work with, secure and reliable. It's actively developped and has many users, so there's a good community surrounding it.

While Amazon S3 is more expensive than some other services (like Backblaze B2), it offers much more flexibility and makes some very good guarantees regarding the safety and availability of data stored within it. This added reliability makes it more suitable (in my opinion) for backups than Backblaze B2.

## Syncthing

### Installation

Installing Syncthing is super easy because it's packaged for a lot of linux distributions. It's also on the Google Play Store, if syncing with an Android phone is required. If it's not packaged for your distribution, binaries are available on the project's download page ([here](https://syncthing.net/downloads/)).

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

## Amazon S3

### Creating a Bucket

To create a bucket in Amazon S3 is fairly straight-forward, but creating a bucket which meets all the requirements isn't so easy. A bucket which can handle backups should be versioned, encrypted, replicated and private.

I created a Terraform module to conveniently create S3 buckets which will work in this way, which can be found [here](https://gist.github.com/marier-nico/5c7c99c7670162175f2c1a4c03703626). Here is a sample which shows a bucket and its replicated counterpart.

**Note**: When using the module from the gist, an IAM user still needs to be created to upload backups to the S3 bucket.

```terraform
resource "aws_s3_bucket" "main_bucket" {
  provider = aws.main_region
  bucket   = var.bucket_name
  acl      = "private"

  versioning {
    enabled = true
  }

  lifecycle_rule {
    id      = "infrequent_access"
    enabled = true

    transition {
      days          = 30
      storage_class = "STANDARD_IA"
    }

    noncurrent_version_expiration {
      days = 15
    }
  }

  replication_configuration {
    role = aws_iam_role.s3_replication_role.arn

    rules {
      id     = "ReplicateAll"
      status = "Enabled"

      destination {
        bucket        = aws_s3_bucket.replication_bucket.arn
        storage_class = "STANDARD"
      }
    }
  }

  server_side_encryption_configuration {
    rule {
      apply_server_side_encryption_by_default {
        sse_algorithm = "AES256"
      }
    }
  }
}

resource "aws_s3_bucket_public_access_block" "block_main_bucket_public_access" {
  bucket = aws_s3_bucket.main_bucket.id

  block_public_acls       = true
  block_public_policy     = true
  ignore_public_acls      = true
  restrict_public_buckets = true
}

resource "aws_s3_bucket" "replication_bucket" {
  provider = aws.replication_region
  bucket   = "${var.bucket_name}-replicated"
  acl      = "private"

  versioning {
    enabled = true
  }

  lifecycle_rule {
    id      = "infrequent_access"
    enabled = true

    transition {
      days          = 30
      storage_class = "STANDARD_IA"
    }

    noncurrent_version_expiration {
      days = 15
    }
  }

  server_side_encryption_configuration {
    rule {
      apply_server_side_encryption_by_default {
        sse_algorithm = "AES256"
      }
    }
  }
}

resource "aws_s3_bucket_public_access_block" "block_replication_bucket_public_access" {
  provider = aws.replication_region
  bucket   = aws_s3_bucket.replication_bucket.id

  block_public_acls       = true
  block_public_policy     = true
  ignore_public_acls      = true
  restrict_public_buckets = true
}
```

## BorgBackup

### Installation

BorgBackup is packaged for several linux distributions, which makes installation very easy. If it's not packaged for your distribution, binaries can easily be found on the project's installation page ([here](https://borgbackup.readthedocs.io/en/latest/installation.html)).

The important thing is that the binary is in the `$PATH` that your cron jobs execute with, which is usually `/bin` or `/usr/bin`. One way to do it is to download the binary to `/opt` and adding a symbolic link to `/usr/bin` with `ln -s /opt/borg /usr/bin/borg` (run as root).

### Configuration

No configuration is necessary as such for BorgBackup, but to use it effectively, it's a very good idea to rely on a script. The BorgBackup project has script examples. The following is an adapted version of an example which works with Amazon S3.

```bash
#!/bin/bash

export AWS_ACCESS_KEY_ID={access-key}
export AWS_SECRET_ACCESS_KEY={secret-access-key}

export BORG_REPO=~/.backup
export BORG_PASSPHRASE='{backup-passphrase}'

# some helpers and error handling:
info() { printf "\n%s %s\n\n" "$( date )" "$*" >&2; }
trap 'echo $( date ) Backup interrupted >&2; exit 2' INT TERM

info "Starting backup"

# Backup the most important directories into an archive named after
# the machine this script is currently running on:

borg create                         \
    --verbose                       \
    --filter AME                    \
    --stats                         \
    --show-rc                       \
    --compression lz4               \
    --exclude-caches                \
                                    \
    ::'{hostname}-{now}'            \
    ~/important/stuff

backup_exit=$?

info "Pruning repository"

borg prune                          \
    --list                          \
    --prefix '{hostname}-'          \
    --show-rc                       \
    --keep-daily    7               \
    --keep-weekly   5               \
    --keep-monthly  12              \
    --keep-yearly   3

prune_exit=$?

# use highest exit code as global exit code
global_exit=$(( backup_exit > prune_exit ? backup_exit : prune_exit ))

if [ ${global_exit} -eq 0 ]; then
    info "Backup and Prune finished successfully"
else
    info "Backup and/or Prune finished with errors"
    info "Not uploading resulting backup to s3"
    exit ${global_exit}
fi

info "Uploading resulting backup to s3"

aws s3 sync --delete --quiet ~/.backup s3://{s3-bucket-name}
```

To run the script (which should be run on a server), a simple and easy tool is `cron`. You can edit your cron entries with `crontab -e`. The example below will run the script above every day at midnight.

```cron
m h  dom mon dow   command
0 0  *   *   *     /home/fileshare/backup/backup.sh >/dev/null 2>&1
```

## Recap

After completing all these steps, you will have installed : Syncthing to sync files between systems, Filestash to share them (with an SFTP server to make the link between Syncthing and Filestash) and BorgBackup to create encrypted backups.

You will also have Amazon S3 buckets to upload your backups and a script which runs daily and makes a backup of Syncthing files.

Overall, this leaves you with convenient file syncing and sharing, along with at least two on-site backups (thanks to Syncthing with one device and a server) and six off-site (in Amazon S3, because data is redundant in three physical locations within the a bucket's region, and we are using two buckets), which follows (and goes beyond) the 3-2-1 backup principle!