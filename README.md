# Project ansible to manage project on Rancher

## Foreword

If you are not familiar with Rancher please read the official documentation http://docs.rancher.com/rancher/latest/en/.

Rancher is used to create segregated environments for each project instances.

Each environment should exist as "environments" in rancher (example: my-uat-project my-production-project ).
Each rancher environment is segregated at network level via Docker Overlay and IPSec tunnels (managed by rancher itself).

Only trusted ops can connect to the machines, unix accounts are managed by ansible.

Developpers will not be allowed to access rancher by itself, rancher master will only be accessible to ops for the moment.
End users will have tools in their project to see their application logs & metrics (elk, prometheus).

### Machine requirements

At the moment this set of playbooks is designed to run on ubuntu. We recommend using latest LTS (16.04 at times of writing).

Each host (including the master) must have at least the following requirements :
* 2 vCPU
* 4 Go ram
* 2 disks with one for the system with 12Go (8Go for / + 4Go for swap) and one for /var/lib/docker with 8Go.

### Example for test environments

3 hosts per Rancher environment :
* one for monitoring (ELK, Prometheus)
* one for the cluster itself

## Playbooks documentation

### Requirements

* Ansible >= 2.3
* Vagrant / Virtualbox (optional) if you want to run this on your own machine
* Configure collectdata/inventory with the list of Host
* Configure collectdata/all/Rancher with the ip of rancher master and docker_registries (if needed)

### Roles dependencies

First you need to fetch roles with ansible galaxy :
```
ansible-galaxy install -r requirements.yml
```

### Rancher platforms

Each Rancher platform will have its own configuration (inventory + group_vars + hosts_vars).
By default wrappers will use the production configuration.

You can use other configurations by changing inventory from the command line (everything is relative to the path of the inventory file).
```
./ansible-playbook_wrapper configure_host.yml -K -i path/to/inventory

```

### SSH configuration

Copy the file `config_ssh.template` to `config_ssh` in this local folder,
then edit `config_ssh` to configure it. The user must match a user declared
in ssh_users list in `<rancher_cluster_name>/group_vars/all/vars`

To prevent typing sudo password when using ansible, create for each cluster
the file `<rancher_cluster_name>/group_vars/all/private` with the following content:

```
ansible_sudo_pass: your_sudo_password
```

Where your_sudo_password is the password declared in ssh_users list in `<rancher_cluster_name>/group_vars/all/vars`

Note that if you are using this mechanism, it will always be used, even if you are using `-k` or `-K`

### Vault

Add the vault password in `ansible/.ansible_vault_pass`

### Workflow to setup rancher from scratch

* run configure_host playbook
* run create_master playbook
* run create_project playbook (everytime you need to setup a new env)

### Configure the Machine
Use the configure_host.yml

You can bootstrap the machine for the first time with the following command

``` ./ansible-playbook_wrapper configure_host.yml -u your_account -Kk ```

This ansible does the following :

* Configures the kernel
* Installs Docker
* Sets machine hostname
* Configures ssh (disable root login, enforce auth by key)
* Removes the former ops account

###  Create a Master

Create a tag for the new Master, for example
```
[rancher-master]
host-1
```

Configure the create_master.yml for [rancher-master]

Launch the playbook
```
./ansible-playbook_wrapper create_master.yml -K
```

This playbook will create a file apiKey in {{ inventory_dir }}/group_vars/all/apikey. This file contains the rancher api key and secrets.
We HIGHLY recommend to vault this file using ansible vault.

### Create a project

Create a tag for your project into the inventory for example
```
[my-first-environment-project]
host-2
host-3
```

The first host of the tag will become the "tools" host (running elk, etc...).

Launch the playbook
```
./ansible-playbook_wrapper create_project.yml -K -e "NAME_PROJECT=my-first-environment-project"
```

This ansible does the following :

* Creates a project into RANCHER
* Creates "API KEY ENVIRONMENT" into rancher and write into group_vars/{{NAME_PROJECT}}
* Adds Host into the project
* Installs some stacks : Janitor

### Create the project "collect-data"

Configure collectdata/inventory
```
[collect-data]
host-2
host-3
```
Launch the playbook
```
./ansible-playbook_wrapper install_skalogs.yml -K -e "NAME_PROJECT=collect-data"
```

This ansible does the following :

* Creates a project into RANCHER
* Creates "API KEY ENVIRONMENT" into rancher and write into group_vars/{{NAME_PROJECT}}
* Adds Host into the project
* Installs stacks :
     * Janitor
     * Elasticsearch
     * Zookeeper
     * Kafka
     * Importer
     * Kibana
     * Packetbeat-dashboard
     * Metricbeat-dashboard
     * Elk-monitoring (to monitor your cluster)
     * Prometheus
     * Endpoints

### Create the project "demo"

This project is a demo for collect data.

Launch the playbook
```
./ansible-playbook_wrapper create_project_demo.yml -K -e "NAME_PROJECT=demo"
```

* Installs stacks :
    * demo-todo : application nodejs
    * demo-petclinic : application springboot
    * packetbeat : collect network informations from Host
    * metricbeat : collect metric informations from Host

## Disaster recovery

If for some reason the host running rancher-master dies, application will remain up and running, so there is no impact.
After running troubleshooting steps, if you don't have any clue we recommend you to wipe the master machine and spawn a new master with "create_master.yml" playbook and run mysql restore script (see above).


### Quick cleanup procedure (optional)
This step can be skipped if you spawn a new master.

Run the following commands on your rancher-master
```
sudo service docker stop
rm -rf /var/lib/docker/*
rm -rf /var/lib/mysql/*
sudo service docker start
```

### Make sure master is configured properly
```
./ansible-playbook_wrapper create_master.yml -K
```

### Restore mysql

On the master enter into the container itself and run the restore script
```
sudo docker exec -ti mysql-backup-s3 bash
./restore.sh
```

This will restore the latest backup find on s3.

If you want to restore a specific backup you can do
```
ID_BUCKET_RESTORE=2016-07-21T133544Z.dump.sql.gz ./restore.sh
```

## Upgrading Rancher

To upgrade Rancher you just need to change the rancher_version version in group vars.
Example: in production/group_vars/all/vars
```
rancher_version: "v1.6.21"
rancher_agent_version: "v1.2.11"
```

We highly recommend to stick to a specific version for production Rancher environment to make sure everything is repeatable.

Then just run again the create_master.yml playbook.
It will upgrade rancher-master smoothly.
When rancher-master is up again it will contact environments host agents and update it if necessary, this operation is done by Rancher itself.

Docker version
--------------
Default docker version is for Ubuntu 18.04 version.
If you want to get this role working on ubuntu 16.04 you may override docker_version variable to an older version :

```
docker_version: "18.02.0~ce-0~ubuntu"
```
Suggested docker version and docker-compose version for Ubuntu 18.04.1 (bionic):

```
docker_version: 5:18.09.0~3-0~ubuntu-bionic
docker_compose_version: 1.23.1
```
See:

docker releases: https://github.com/docker/docker-ce/releases

docker-compose releases: https://github.com/docker/compose/releases/

Reference to localhost present in /etc/resolv.conf
--------------------------------------------------
Some Linux distributions will run a local DNS cache server like dnsmasq. If this is the case, the nameserver entry in /etc/resolv.conf will point to 127.0.0.1 (localhost). This configuration is re-used when running Docker containers, but inside a container you cannot reach dnsmasq on 127.0.0.1. Since rancher/agent:v1.2.7, the agent will fail to register and log:

```
ERROR: DNS Checking loopback IP address 127.0.0.0/8, localhost or ::1 configured as the DNS server on the host file /etc/resolv.conf, can't accept it
```
To fix this, you can either specify DNS servers for Docker or disable dnsmasq. Instructions on both options are provided in the [Docker documentation](https://docs.docker.com/install/linux/linux-postinstall/#dns-resolver-found-in-resolvconf-and-containers-cant-use-it)

This can be done by overriding docker_opts in group_vars/all/.
```
docker_opts: "--dns 8.8.8.8 --dns 8.8.4.4"
```
