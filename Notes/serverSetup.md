# Ubuntu Server 22.04.3

## Setup

GNU/Linux 5.10.160-rockchip aarch64
Orange Pi 5 Plus w/NVMe SSD

### Image Installation and Setup
1. Installed Ubuntu Server IAW GitHub Repo for Joshua Riek's Orange Pi Ubuntu
2. Adjusted console font size
3. Performed `sudo apt update` & `sudo apt upgrade`
4. `adduser` and deluser
	1. Added: admin, jonathoncs, navelfuzz
	2. Deleted: ubuntu

#### User Info
See `.env` file

#### Set User as Sudo User
`usermod -aG sudo admin`
### SSH & UFW Setup

#### Change Server Name
1. `hostname`
2. `sudo hostnamectl set-hostname obsidian`

#### Server-Side Configurations
1. `sudo nano /etc/ssh/sshd_config`
	1. `PermitRootLogin no`
	2. `Port ****`
	3. `PasswordAuthentication no`
2. `sudo service ssh restart`
3. Within the user home directory:
	1. `mkdir .ssh`
	2. `touch ~/.ssh/authorized_keys`
	3. Copy the public key generated on local machine to this directory `id_rsa.pub`
		1. `ssh-keygen -b 4096`

#### Client-Side Configurations
1. `nano ~/.ssh/config`

```bash
Host ****
	HostName 192.168.***.***
	Port ****
	User ****
	IdentityFile ~/.ssh/id_rsa
```

#### UFW app Profile
1. Navigate to `etc/ufw/applications.d`
2. `nano openssh-server`

```bash
[OpenSSH]
title=Secure shell server, an rshd replacement
description=OpenSSH is a free implementation of the Secure Shell protocol.
ports=****/tcp
```

Then ensure you re-allow OpenSSH in UFW because if added prior to this change it will still look at Port 22

---
**23FEB2024**
* Ansible installed + Configured
	* See Ansible installation notes

#### Lynis
Commands:
1. `sudo apt install lynis`
2. `lynis audit system`

## 29FEB 2024 Updates

### Ansible

```bash
root@obsidian:/etc/ansible# tree
.
├── ansible.cfg
├── docker_playbook.yml
└── hosts
```

Run playbook command: `ansible-playbook <name of playbook file>`

**ansible.cfg**

```bash
[defaults]
inventory = /etc/ansible/hosts
remote_user = admin
host_key_checking = False
private_key_file = ./ssh/id_rsa

[privilege_escalation]
become = true
become_method = sudo
become_user = root
become_ask_pass = False
```

**hosts**

```bash
[local]
localhost ansible_connection=local
```

**docker_playbook.yml**

```bash
---
- hosts: local
  become: true

  tasks:
    - name: Install aptitude
      apt:
        name: aptitude
        state: latest
        update_cache: true

    - name: Install required system packages
      apt:
        pkg:
          - apt-transport-https
          - ca-certificates
          - curl
          - software-properties-common
          - python3-pip
          - virtualenv
          - python3-setuptools
        state: latest
        update_cache: true

    - name: Add Docker GPG apt Key
      apt_key:
        url: https://download.docker.com/linux/ubuntu/gpg
        state: present

    - name: Add Docker Repository
      apt_repository:
        repo: deb https://download.docker.com/linux/ubuntu jammy stable
        state: present

    - name: Update apt and install docker-ce
      apt:
        name: docker-ce
        state: latest
        update_cache: true

    - name: Install Docker Module for Python
      pip:
        name: docker

    - name: Install Docker Compose plugin
      apt:
        name: docker-compose-plugin
        state: latest
        update_cache: true

    - name: Create Docker group
      group:
        name: docker
        state: present

    - name: Add current user to Docker group
      user:
        name: "{{ ansible_env.SUDO_USER }}"
        groups: docker
        append: yes

    - name: Enable and start docker service
      systemd:
        name: docker.service
        enabled: yes
        state: started

    - name: Enable and start containerd service
      systemd:
        name: containerd.service
        enabled: yes
        state: started

    - name: Download ufw-docker script
      get_url:
        url: https://github.com/chaifeng/ufw-docker/raw/master/ufw-docker
        dest: /usr/local/bin/ufw-docker
        mode: '0755'

    - name: Install ufw-docker to modify after.rules
      command: ufw-docker install

    - name: Install neofetch
      apt:
        name: neofetch
        state: latest
        update_cache: true

    - name: Create necessary directories for Docker and Dockge
      file:
        path: "{{ item }}"
        state: directory
        owner: root
        group: docker
        mode: '0755'
      loop:
        - /srv/docker
        - /srv/dockge
        - /srv/dockge/data
        - /opt/stacks

    - name: Deploy Dockge container
      community.docker.docker_container:
        name: dockge
        image: louislam/dockge:1
        restart_policy: always
        ports:
          - "5001:5001"
        volumes:
          - /var/run/docker.sock:/var/run/docker.sock
          - /srv/dockge/data:/app/data  # Ensure this directory exists or adjust accordingly
          - /opt/stacks:/opt/stacks  # Ensure this directory exists or adjust accordingly
        env:
          DOCKGE_STACKS_DIR: "/opt/stacks"
        state: started
```
### Dockge

All contents are saved within the `/dockge` directory and the dockge container itself is created within the ansible playbook above.

--- 

