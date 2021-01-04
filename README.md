# balenablocks/duplicati

A guide for adding [Duplicati](https://www.duplicati.com/) as a backup service to an existing app.

## Goals

- Perform encrypted off-site backups in case of data loss like memory card corruption
- Easy to restore partial or complete data in case of emergency
- Optionally use backups to migrate data to a new memory card or device

## Why Duplicati

From the [Duplicati](https://www.duplicati.com/) homepage:

> Many Backends

> Duplicati works with standard protocols like FTP, SSH, WebDAV as well as popular services like Backblaze B2, Tardigrade, Microsoft OneDrive, Amazon S3, Google Drive, box.com, Mega, hubiC and many others.

> Features

> Backup files and folders with strong AES-256 encryption. Save space with incremental backups and data deduplication. Run backups on any machine through the web-based interface or via command line interface. Duplicati has a built-in scheduler and auto-updater.

> Free software

> Duplicati is free software and open source. You can use Duplicati for free even for commercial purposes. Source code is licensed under LGPL. Duplicati runs under Windows, Linux, MacOS. It requires .NET 4.5 or Mono.

> Strong encryption

> Duplicati uses strong AES-256 encryption to protect your privacy. You can also use GPG to encrypt your backup.

> Built for online

> Duplicati was designed for online backups from scratch. It is not only data efficient but also handles network issues nicely. E.g. interrupted backups can be resumed and Duplicati tests the content of backups regularly. That way broken backups on corrupt storage systems can be detected before it’s too late.

> Web-based user interface

> Duplicati is configured by a web interface that runs in any browser (even mobile) and can be accessed - if you like - from anywhere. This also allows to run Duplicati on headless machines like a NAS (network attached storage).

## Deployment

Duplicati is meant to be used alongside other services, so you will need to create a service in your `docker-compose.yml` file.

```yaml
version: "2.1"

volumes:
  duplicati:
  # ... other volumes

services:
  duplicati:
    image: linuxserver/duplicati:<tag>
    environment:
      PUID: "0"
      PGID: "0"
      CLI_ARGS: --webservice-interface=any
    ports:
      - 8200:8200/tcp
    volumes:
      - duplicati:/config
      - <exampleVolumeA>:/source/<volumeA>
      - <exampleVolumeB>:/source/<volumeB>

  # ... other services
```

Note that for each existing named volume we want backed up, we need to mount that volume to the duplicati service under `/source/<name>`.

So for example to add Duplicati to an existing [Pi-hole](https://github.com/klutchell/balena-pihole) app:

```yaml
version: "2.1"

volumes:
# volumes we want to back up
  pihole_config:
  dnsmasq_config:
# duplicati database volume
  duplicati:

services:

  pihole:
    build: ./pihole
    privileged: true
    volumes:
      - "pihole_config:/etc/pihole"
      - "dnsmasq_config:/etc/dnsmasq.d"
    dns:
      - "127.0.0.1"
      - "1.1.1.1"
    environment:
      DNSMASQ_LISTENING: eth0
      INTERFACE: eth0
      DNS1: 127.0.0.1#5053
      DNS2: 127.0.0.1#5053
    network_mode: host
    labels:
      io.balena.features.dbus: "1"

  unbound:
    build: ./unbound
    ports:
      - "5053:5053/udp"

# duplicati service definition
  duplicati:
    image: linuxserver/duplicati:latest
    environment:
      PUID: "0"
      PGID: "0"
      CLI_ARGS: --webservice-interface=any
    ports:
      - 8200:8200/tcp
    volumes:
      - duplicati:/config
      - pihole_config:/source/pihole_config
      - dnsmasq_config:/source/dnsmasq_config
```

## Usage

Once the service is defined and your volumes are shared we can use the Duplicati Web interface
to define our backup source, destination, passphrase, and schedule.

Connect to `http://<device-ip>:8200` to begin setup.

The first time you start the Duplicati Web interface, you'll be prompted to set a password.

It is strongly recommended to set a password to the Web interface by clicking the Yes button.

### Creating a new backup job

Here we'll refer to the official documentation as it will guide you through the wizard.

<https://duplicati.readthedocs.io/en/latest/03-using-the-graphical-user-interface/#creating-a-new-backup-job>

However here are the quick steps:

1. Add backup: Configure a new backup
2. Give this backup a name (like the app/device name) and set a secure passphrase
3. Backup destination has many options that we won't go into here
but the **Path on server** value should be somewhat unique (like the app/device name)
4. For **Source data** just **Add a path directly** and specify `/source/`
5. Add filters or exclusions if needed
6. Choose a schedule and save!

For more info here's the official documention:

- <https://duplicati.readthedocs.io/en/latest/>

## Acknowledgements

- <https://github.com/duplicati/duplicati>
- <https://github.com/linuxserver/docker-duplicati>
