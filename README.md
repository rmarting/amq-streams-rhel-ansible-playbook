# AMQ Streams RHEL Standalone Playbook

## About

This [Ansible Playbook](http://docs.ansible.com/ansible/playbooks.html) includes
a set of different roles:

* **kafka-install**: Deploys a cluster of Red Hat AMQ Streams on several hosts.
* **kafka-uninstall**: Uninstall a cluster of Red Hat AMQ Streams from several hosts.

These roles with the right configuration will allow you to deploy a full complex
Red Hat AMQ Streams on RHEL and automate the most common tasks.

[Change Log](./Changelog.md)

## Ansible Requirements

This playbook has tested with the next Ansible versions:

```txt
ansible 2.9.3
```

The playbook has to be executed with **root permission**, using the **root user** or
via **sudo** because it will install packages and configure the hosts.

## RHEL Subscription Required

This role is prepared to be executed in a RHEL 7 Server or RHEL 8 Server. Subscription is required to
execute the **Grant RHEL repos enabled** task, which enables the required
repositories to install Red Hat AMQ Streams dependencies

## Global and Host Variables

### Global Variables

Each playbook will use a set of Global Variables with values that it will be the same for all of them.

These variables will be defined in **groups_vars/all.yaml** file.

These variables are:

* **binary**: to define AMQ Streams binary location at your ansible control machine.

```yaml
# AMQ Streams Instalation Source
binary:
  folder: '/tmp/'
```

* **kafka**: Define the Red Hat AMQ Streams version to install. This values will
form the path the the binaries: */tmp/amq-streams-{{ kafka['version'] }}.-bin.zip*

```yaml
# AMQ Streams Version
kafka:
  version: '1.3.0'
```

* **user**: OS user to execute the AMQ Streams processes.

```yaml
# OS User to install/execute AMQ Streams
user:
  name: 'kafka'
  shell: '/bin/bash'
  homedir: 'True'
```

* **java_8_home** and **java_8_home**: Path to Java Virtual Machine 8 (for RHEL 7) and
Java Virtual Machine 11 (for RHEL 11) to execute the processes.

```yaml
# Java Home
java_8_home: '/usr/lib/jvm/jre-1.8.0-openjdk'
java_11_home: '/usr/lib/jvm/jre-11-openjdk'
```

* **kafka_base**: AMQ Streams base path where it will be installed everything.

```yaml
kafka_base: '/opt/kafka'
```

* **kafka_home**: AMQ Streams Home path where it will installed the cluster. This
variable will be defined for each host with a name. This allow installs more
instances in the same hosts.

```yaml
kafka_home: "{{ kafka_base }}/{{ kafka_name }}"
```

* **zookeeper_data_folder**: Path to be used by Apache Zookeeper to store its status.

```yaml
zookeeper_data_folder: "/var/lib/{{ kafka_name }}/zookeeper"
```

* **kafka_storage_folder**: Path to be used by Apache Kafka to store the partitions.

```yaml
kafka_storage_folder: "/var/lib/{{ kafka_name }}/kafka"
```

### Host Variables

Each role will use a set of Host Variables defined in the playbook for each
host defined in the inventory.

These variables are:

* **kafka_name**: Logical name to identify each AMQ Cluster Instance. Mandatory.
* **port_offset**: Port offset to be added to default AMQ Cluster ports. Mandatory to install different instances in the same host.

These variables should be defined in the playbook as:

```yaml
  roles:
    - {
        role: kafka-install,
        kafka_name: 'cluster-01',
        port_offset: '0'
      }
```

## Ansible Roles

### kafka-install role

This role deploys a AMQ Cluster in a set of hosts.

The main tasks done are:

* Install AMQ Streams prerequisites and set up host
* Install AMQ Streams binaries
* Create OS users and OS services
* Set up different AMQ Streams features: Clustering, storage folders, ...

Red Hat AMQ Streams binaries should be downloaded from [Red Hat Customer Portal](https://access.redhat.com/jbossnetwork/restricted/listSoftware.html?downloadType=distributions&product=jboss.amq.streams)
(subscription needed). The binaries should be copied into **/tmp** folder from
the host where the playbook will be executed.

#### Configuration parameters

Role's execution could be configured with the following variables.

#### Example playbook

To execute this playbook:

```bash
ansible-playbook -i hosts kakfa-install.yaml
```

Inventory (*host* file):

```txt
[amq_lab_environment]
rh8amq01
rh8amq02
rh8amq03

[zookeepers]
rh8amq01
rh8amq02
rh8amq03

[brokers]
rh8amq01
rh8amq02
rh8amq03
```

Playbook (*kafka-install.yaml* file):

```yaml
---
- name: Install Playbook of an AMQ Streams Environment
  hosts: amq_lab_environment
  serial: 1
  remote_user: root
  gather_facts: true
  become: yes
  become_user: root
  become_method: sudo
  roles:
    - {
        role: kafka-install,
        kafka_name: 'cluster-01',
        port_offset: '0'
      }
```

#### Verify Installation

Once all Zookeeper nodes of the clusters are up and running, verify
that all nodes are Zookeeper members of the cluster by sending
a *stat* command to each of the nodes using ncat utility.

```bash
> echo stat | ncat rh8amq02 2181
Zookeeper version: 3.4.14-redhat-00001-a70f9d20b6373085f9a94183a1324a57cdaead75, built on 05/30/2019 13:26 GMT
Clients:
 /192.168.124.1:33172[0](queued=0,recved=1,sent=0)
 /192.168.124.15:53240[1](queued=0,recved=346,sent=346)

Latency min/avg/max: 0/1/66
Received: 358
Sent: 357
Connections: 2
Outstanding: 0
Zxid: 0x200000034
Mode: leader
Node count: 27
Proposal sizes last/min/max: 133/32/372
```

Once all nodes of the Kafka brokers are up and running, verify that
all nodes are members of the Kafka cluster by sending a *dump* command
to one of the Zookeeper nodes using the ncat utility.
The command prints all Kafka brokers registered in Zookeeper.

```bash
> echo dump | ncat rh8amq02 2181
SessionTracker dump:
Session Sets (3):
0 expire at Thu Jan 01 02:07:57 CET 1970:
0 expire at Thu Jan 01 02:08:00 CET 1970:
3 expire at Thu Jan 01 02:08:03 CET 1970:
	0x1000033f9ab0000
	0x300003467e80000
	0x2000033eebe0001
ephemeral nodes dump:
Sessions with Ephemerals (3):
0x1000033f9ab0000:
	/controller
	/brokers/ids/1
0x2000033eebe0001:
	/brokers/ids/0
0x300003467e80000:
	/brokers/ids/2
```

### kafka-uninstall role

This role uninstall an AMQ Streams Cluster from inventory hosts.

The main tasks done are:

* Stop AMQ Streams services
* Remove OS services
* Remove AMQ Streams binaries

#### Configuration parameters

Role's execution could be configured with the following variables.

#### Example playbook

To execute this playbook:

```bash
ansible-playbook -i hosts kafka-uninstall.yaml
```

Inventory (*host* file):

```txt
[amq_lab_environment]
rh8amq01
rh8amq02
rh8amq03

[zookeepers]
rh8amq01
rh8amq02
rh8amq03

[brokers]
rh8amq01
rh8amq02
rh8amq03
```

Playbook (*kafka-uninstall.yaml* file):

```yaml
---
- name: Uninstall Playbook of an AMQ Streams Environment
  hosts: amq_lab_environment
  remote_user: root
  gather_facts: true
  become: yes
  become_user: root
  become_method: sudo
  roles:
    - {
        role: kafka-uninstall,
        kafka_name: 'cluster-01'
      }
```

## Main References

* [Product Documentation for Red Hat AMQ](https://access.redhat.com/documentation/en-us/red_hat_amq/7.5/)
* [Using AMQ Streams on RHEL](https://access.redhat.com/documentation/en-us/red_hat_amq/7.5/html-single/using_amq_streams_on_rhel/index)
* [Ansible Documentation](http://docs.ansible.com/ansible/)
