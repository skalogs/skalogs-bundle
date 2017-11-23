# Project ansible to manage project on Rancher

## Foreword

If you don't know yet rancher please read the official documentation http://docs.rancher.com/rancher/latest/en/.

Rancher will be used to create segregated environments for each project instances.

Each environments should exists as "environments" in rancher (example: my-uat-project my-production-project ).
Each rancher environment is segregated at network level via Docker Overlay and IPSec tunnels (managed by rancher itself).

Only trusted ops can connect to the machines, unix accounts are managed by ansible.

Developpers will not be allowed to access rancher by itself, rancher master will be accessible only for ops for the moment.
End users will have tools in their project to see their application logs & metrics (elk, prometheus).

### Machine requirements

At the moment this set of playbooks is designed to be run on ubuntu. We recommend using latest LTS (16.04 at times of writing).

Each host (including the master) must have at least the following requirements :
* 2 vcpu
* 4 Go ram
* 2 disks with one for the system with 12Go (8Go for / + 4Go for swap) and one for /var/lib/docker with 8Go.

### Exemple for test environments

3 hosts per rancher environment :
* one for monitoring (elk, prometheus)
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

Each rancher platform will have it's own configuration (inventory + group_vars + hosts_vars).
By default wrappers will use production one.

But you can use other ones changing inventory from the command line (everything is relative to the path of the inventory file).
```
./ansible-playbook_wrapper configure_host.yml -K -i path/to/inventory

```

### SSH configuration

Copy the file `config_ssh.template` to `config_ssh` in this local folder
then edit `config_ssh` to configure it. The user must match a user declared
in ssh_users list in `<rancher_cluster_name>/group_vars/all/vars`

To prevent typing sudo password when using ansible, create for each cluster
the file `<rancher_cluster_name>/group_vars/all/private` with the following content:

```
ansible_sudo_pass: your_sudo_password
```

Where your_sudo_password is the password declared in ssh_users list in `<rancher_cluster_name>/group_vars/all/vars`

Please note that if you are using this mechanism, it will always be used, even if you are using `-k` or `-K`

### Vault

Please add the vault password in `ansible/.ansible_vault_pass`

### Workflow to setup rancher from scratch

* run configure_host playbook
* run create_master playbook
* run create_project playbook (everytime you need to setup a new env)

### Configure the Machine
Use the configure_host.yml

You can bootstrap the machine for the first time with the following command

``` /ansible-playbook_wrapper configure_host.yml -u your account -Kk ```

This ansible install

* Configure the kernel
* Docker
* Set machine hostname
* Configure ssh (disable root login, enforce auth by key)
* Remove the former ops account

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

This ansible create :

* A project into RANCHER
* Create "API KEY ENVIRONMENT" into rancher and write into group_vars/{{NAME_PROJECT}}
* Add Host into the project
* Install some stacks : Janitor

### Create the project "collect-data"

Configure collectdata/inventory
```
[collect-data]
host-2
host-3
```
Launch the playbook
```
./ansible-playbook_wrapper create_collect_data.yml -K -e "NAME_PROJECT=collect-data"
```

This ansible create :

* A project into RANCHER
* Create "API KEY ENVIRONMENT" into rancher and write into group_vars/{{NAME_PROJECT}}
* Add Host into the project
* Install some stacks : 
     * Janitor
     * Elasticsearch
     * Zookeeper
     * Kafka
     * Importer
     * Kibana
     * Packetbeat-dashboard
     * Metricbeat-dashboard
     * Elk-monitoring : for monitor your cluster
     * Prometheus
     * Endpoints
          
### Create the project "demo"

This project is a demo for collect data from different stacks:
* demo-todo : application nodejs
* demo-petclinic : application springboot
* packetbeat : collect network informations from Host
* metricbeat : collect metric informations from Host

## Disaster recovery

If for some reason the host running rancher-master dies, application will remain up and running, so there is no impact.
If after running troubleshooting steps you don't have any clue we recommend you to wipe the master machine spawn a new master with "create_master.yml" playbook and run mysql restore script (see above).


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

## Upgrading rancher

To upgrade rancher you just need to change the rancher_version version in group vars.
Exemple: in production/group_vars/all/vars
```
rancher_version: "v1.6.10"
rancher_agent_version: "v1.2.6"
```

We highly recommend to stick to a specific version for production rancher environment to make sure everything is repeatable.

Then just run again the create_master.yml playbook.
It will upgrade rancher-master smoothly.
When rancher-master is up again it will contact environments host agents and update it if necessary, this operation is done by rancher itself.
