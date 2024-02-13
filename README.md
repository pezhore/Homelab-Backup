# Pez's Homelab Restic Backup Solution

## Overview

I needed a way to backup my ever growing homelab environment, with the following requirements:

**1. File-level backups with include/exclude lists**

I have image/VM level backups that are replicated on-site to a secondary storage platform through Proxmox, I'm less
concerned about that level of recovery and more recoverability of specific file content (e.g. documents,
media, etc). To that end, I need a way to specify which files/folders are included in a backup for a host, and which 
sub-folders/files should be excluded.

**2. Encrypted transit/storage**

The backup transfer should be encrypted to secure the transit, and the storage should be encrypted to prevent data
compromise at rest. Ideally, both should be encrypted with keys that I own and am responsible for securing.

**3. Automatable**

I dislike running things manually. I forget, and things get missed. Backups are something that should be set it and 
forget it.

**4. Standarized tooling**

I don't want to create some bespoke script that tries to re-invent the wheel. Standarized backup tooling with a vibrant
community is essential.


## Solution

I chose to go with [Restic](https://restic.net/) - it hits on all of the above requirements:

- It can accept include/exclude lists from either the command line or through text files
- All data in the restic repo is encrypted with AES-256 and authenticated with Poly1305-AES
- Ansible can do the heavy lifting for installation/management/triggering backups
- The community is relatively robust. The [Github repo](https://github.com/restic/restic) has 1.4k forks and 22.8k Stars.
 Releases are fairly consistent.

## Requirements

- Ansible (tested on 2.14.1) and Python `hvac` package (for Vault integration)
- Hashicorp Vault for secrets management
- SSH access (or localhost?) to target systems
- Wasabi S3 (or alternative backend repo)

## Installation/Getting Started

TL;DR - Install ansible, configure your own `lab-inventory.yml` file, setup Vault to house your secrets, configure your
S3, and run the playbook.

### Inventory File

The playbook does several checks to see if the host (e.g. homelab server) is in a specific inventory group to ensure the
appropriate include/exclude files are used for that host. For example, I run a PowerDNS cluster for redundant internal
DNS - I want to make sure both systems in the `pdns` inventory group have the appropriate files backed up. by putting
both `pdns-1` and `pdns-2` in the `[pdns]` inventory group, I can target it with my playbook with the following stanza:

```
when: inventory_hostname in groups['pdns']
```

### The Playbook

The playbook has four main sections:

1. Restic installation portion (and pre-requisites)
2. Repeating sections for each of my target inventory groups to set the exclude/include file content
3. Running Restic on each system
4. Cleaning up old backups

### Vault Integration

I could just hardcode my Restic/S3 Access Key, Secret Key, Repository, and Restic Password in the playbook, but

1. That's bad practice and a great way (if I pushed it to Github) to have my accounts compromised.
2. If I have to rotate my credentials, I would have to touch this playbook (and all the other playbooks that use those
credentials).

Setting up and managing [Vault](https://www.vaultproject.io/) is out of scope for this project, but it is highly
recommended for secrets management. By placing my credentials in a Vault path (in this case `homelab/wasabi`), I can 
directly reference the key/value pairs in that path (e.g. `restic_access_key`).

If you do not have a Vault instance stood up, or do not wish to have that external dependency, you can just hardcode
your secrets (but do not share/upload/commit your altered playbook), or look into
[alternatives](https://docs.ansible.com/ansible/latest/vault_guide/index.html).