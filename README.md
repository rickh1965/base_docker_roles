# Base and Docker Server roles
## Edit the inventory file first
You'll notice that the sample inventory file is not a simple list of hosts. 

### Variables used in the inventory file
- host_purpose: this variables denotes the purpose of the machine. It is displayed as part of the MOTD banner
- container_path: The mount point that contains all of the docker compose files for this environment.
- docker_env: denotes a group of containers that will run together on a given server. In my example, I use "dev" and "prod".
 a snippet from my inventory (hosts) file is as follows:

 ```yaml
 proxmox:
      hosts:
        pve01.home.lan:
          host_purpose: "Proxmox VE Cluster Node 1"
        pve02.home.lan:
          host_purpose: "Proxmox VE Cluster Node 2"
        pve03.home.lan:
          host_purpose: "Proxmox VE Cluster Node 3"
        pve04.home.lan:
          host_purpose: "Proxmox VE Standalone"
    pihole:
      hosts:
        ns0[1:2].home.lan:
          host_purpose: "Pihole DNS Server"
    docker_hosts:
      hosts:
        dev-docker.home.lan:
          host_purpose: "Dev Docker Server"
          container_path: "{{ mountpoint }}/dev"
          docker_env: "dev"
        prod-docker.home.lan:
          host_purpose: "Prod Docker Server"
          container_path: "{{ mountpoint }}/prod"
          docker_env: "prod"
```

## Base Role
### Requirements
As with all of my playbooks, I test on Rocky Linux only. This should run on other Redhat variants but if you run Ubuntu --- not so much. 

An NFS server that exports to your servers and contains all of the container docker-compose files in seperate directories. If the mountpoint is "/docker/dev", it is expected you will have directories for each container under that location i.e.

```
/docker/dev/heimdall
/docker/dev/portainer
/docker/dev/speedtest
/docker/dev/traefik
/docker/dev/uptime-kuma
```

Given that strucure this playbook will automatically cd into each directory and bring up the conatiner with docker compose.

### Description

This is a general playbook that ensure that your servers are up to date on all packages in the installation. This should be run on all of your servers.

Included in this playbook is a **chrony** installation and a custom **MOTD** script for each of the servers. Both of these additional plays are optional and default to not being applied. The chrony package and config assume that your NTP time server is the local gateway on your network. The **MOTD** setup is to my taste and may not be for everyone.

To enable these additional plays, simply edit ```roles/base/vars/main.yml``` This file is where you will also find the administrative username and the timezone. Edit this file to you needs. The default file looks like this:

```yaml
# vars file for base
with_motd: false
with_chrony: false
site_name: "Your Site Name"
TZ: "America/Chicago"
```

## Docker Server Role
This role installs, configured a shared storage and brings up all your docker containers. The variables in the inventory  (hosts) file should be set for your environment for this to work properly. In my environment, I use and NFS share to mount all of my container files. Make sure you have it set up as described in the **requirements** section. The admin_user variable denotes the user that will perform docker functions on the host. This should only be run on the servers that will run docker.

The vars file is here:
```yaml
# vars file for docker
mynfs: "nas01.home.lan:/volume1/docker"
mountpoint: "/docker"
permission: '0777'
myopts: 'rw,sync'
admin_user: "admin"
```
# p410i Controller to HBAmode

If you are running an old HP Proliant Gx.  Chances are you have an internal raid controller based on the p410i.  HBAmode support is somehow not supported by HP. In proxmox, you ZFS and CEPH recomend configuring your server in a way where you are talking directly to the disks not via a RAID controller.  You can try to set up RAID 0 LUNS and do it that way or you can disable the RAID controller's dictatorial control over your storage.

I found this:

https://github.com/im-0/hpsahba.git

hpsahba was written by Ivan Mironov <mironov.ivan@gmail.com>

This executable will give you the ability to flip your controller into HBA mode.

To get this working under proxmox, I found this link in a post on their forum:
https://forum.proxmox.com/threads/proxmox-7-on-hp-g7-and-how-to-enable-hba-mode-p410-hpsahba.105320/

The post goes through a very manual set of instructions on how to do this. I simply tightened it up and made this playbook to apply hbamode in a more automated fashion.

<h1 style="text-align: center;">Caution!</h1>
Failure to pay attention to the directions here or lack of knowledge regarding your controller can render you server unbootable. I take no responsibility for what yu do on your server.


## Prerequisites:

Once you execute this playbook, the disks behind your controller will lose the ability to be booted from. You must set up  bootiing **PERMANENTLY** via USB (I boot on the internal USB port) or reconfigure the DVD controller port to boot *independently* of the disks behind the controller. This is discussed in the forum post I referenced above.

Again, for emphasis. **Do not run this playbook until you are booting outside of the RAID controller.**

Also, you *must* have the controller updated to version 6.64. Check HPE's website for what you need to update the controller firmware. The playbook checks for this firmware version and fails if this requirement is not met.

Edit the host section in p410_hbamode.yml before executing so that the playbook knows where to go. I seperated out my OLD HP servers into their own group for clarity called p410_servers. All the normal requirements apply here. Must be a valid user who can become root on the target and the target server *must* have a working internet connection. This playbook is intended for a fully configured Proxmox 7.x server.

## Execution
```bash
ansible-playbook p410_hbamode.yml
```
License
-------

BSD

