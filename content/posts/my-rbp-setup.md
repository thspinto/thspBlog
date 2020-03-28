---
title: "My Raspberry Pi Setup"
date: 2020-03-08T20:29:49-03:00
draft: false
screenshot: /rbp.jpg
featured: false
tags: [ansible, rbp, raspberry, auth0, traefik]
---

![Raspberry Pi Logo in front of blue circuit board](/rbp.jpg)

# Tired of configuring my RBP

I've had a raspberry pi for a while and most of the time it rests off in a drawer. From time to time I dust off the SD card to setup the box with dynamic DNS and play with it as my personal server. It always went back to the drawer when my vacations fished and I broke the SO with some crazy experiment.

As an SRE I spend most of my time thinking what flows are more burdensome to the team and how can we automate the process to gain more time to do cool more cool stuff. But with the RBP it was different, I'd never versioned the scripts I executed and never though about creating a proper automation for the box.

This time I started differently to avoid giving up on the Pi again. Every configuration running on the Pi must be replayable so I can fix it just by burning a new SO in the SD card and running the automation to set it up. What I have up to now is [here](https://github.com/thspinto/raspberry-config).

# How I did it

## Provisioning Raspbian

Start off with the (Rasbian Buster Lite image)[https://www.raspberrypi.org/downloads/raspbian/] and burn it on the SD card. Just assure you're pointing to the right disk before running the commands below.


```
unzip -p 2020-02-13-raspbian-buster-lite.zip | sudo dd of=/dev/disk2 bs=4m
touch /Volumes/boot/ssh # to enable ssh
```

## Ansible automation

Ansible [Ansible](https://docs.ansible.com/ansible/latest/index.html) is a grate tool for automating systems configurations. This setup starts with the default user and password.

### Initialize

The `initialize` playbook adds my user and public keys to the RBP and disables password ssh logins.

```
ANSIBLE_HOST_KEY_CHECKING=False ansible-playbook initialize.yaml -i inventories/thspinto
```

Notice that `inventories/thspinto/all.yaml` is unreadable. That's because its encrypted with [Ansible Vault](https://docs.ansible.com/ansible/latest/user_guide/vault.html#file-level-encryption). I don't recommend versioning secrets in public repositories most use cases, even if they are encrypted. In my case I versioned it because I'll certainly lose the file other wise. Also, in my case, the only sensitive information in is a token to alter my personal DNS registry. Since there is virtually no traffic in my domain, it's no big deal.

### Basic setup

The second phase it doing a little hardening and adding docker. This is done on the setup playbook.

```bash
ansible-galaxy install -r requirements.yml
ansible-playbook --vault-password-file /path/to/my/vault-password-file setup.yaml -i inventories/thspinto
```

### Services

The last playbook spins up the services I want to run using docker compose. I separated it from the rest because it is the most frequently changed and deployed.

```bash
ansible-playbook --vault-password-file /path/to/my/vault-password-file services.yaml -i inventories/thspinto
```

Currently I run:

* [Cloudflare DDNS](https://github.com/oznu/docker-cloudflare-ddns) to register the Pi's id the my domain.
* [Traefik 2.1](https://docs.traefik.io) as proxy with docker service discovery
* [ForwardAuth](https://github.com/thomseddon/traefik-forward-auth) to have OpenConnectID login using [Auth0](https://auth0.com/)


# My Pi

Everything runs fine on model B with one 700MHz core and 512MB of ram. Checkout the [raspberry-config repo](https://github.com/thspinto/raspberry-config) for more details and updates.
