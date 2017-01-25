# eg-customer
An example playbook that demonstrates how the various application-specific roles under the DataNexus organization (currently `dn-kafka`, `dn-postgresql`, `dn-influxdb`, `dn-telegraf`, and `dn-solr`) can be wrapped as submodules in a larger playbook that is used to deploy multiple applications to multiple nodes.  The current implementation of the eg-customer project simply wraps one of our roles (the `dn-kafka` role) and only supports deployment to a named Virtual Machine instance (by IP address) using Vagrant, but it could be quite easily extended to support other roles and additional cloud types.

# Installation
To install kafka using the `site.yml` playbook in this repository, first clone the contents of this repository to a local directory using a command like the following:

```bash
$ git clone --recursive https://github.com/Datanexus/eg-customer
```

That command will pull down the repository and it's submodules (currently the only dependency embedded as a submodule is the dependency on the `https://github.com/Datanexus/dn-kafka` repository, which itself depends on the `https://github.com/Datanexus/common-roles` repository).

# Defining the parameters necessary for deployment
The `eg-customer` repository includes a playbook (in the `site.yml` file contained at the top-level of the repository) that demonstrates how any of the DataNexus application-specific roles might be used from within a higher-level playbook.  That playbook currently looks like this:

```yaml
# (c) 2016 DataNexus Inc.  All Rights Reserved
---
- name: Setup eg-customer environment
  hosts: "{{host_inventory}}"
  vars_files:
    - dn-kafka/vars/kafka.yml
    - vars/cloud.yml
    - vars/tenant.yml
    - vars/test-project.yml
    - vars/development-domain.yml
  vars:
    - combined_package_list: "{{ (default_packages|default([])) | union(kafka_package_list) | union((install_packages_by_tag|default({})).kafka|default([])) }}"
  roles:
    - role: dn-kafka/common-roles/ensure-interfaces-up
    - role: dn-kafka/common-roles/get-iface-addr
      iface_name: "{{kafka_iface}}"
    - role: dn-kafka/common-roles/setup-web-proxy
    - role: dn-kafka/common-roles/install-packages
      package_list: "{{combined_package_list}}"
    - role: dn-kafka
      kafka_addr: "{{iface_addr}}"
```

As you can see, this playbook (which will be run for all hosts in the host list provided via the `host_inventory` variable) first loads a set of variables into the playbook from a set of variable files, then constructs a set of packages that should be installed on all Kafka nodes, and finally runs the set of roles that are needed to deploy Kafka to each of those nodes.

The configuration used in those roles is broken out across a number of files that are included in the playbook run using the `vars_files` list.  Specifically, at runtime, the playbook loads the following files (in this order):

1. The `dn-kafka/vars/kafka.yml` file, which contains default values for the parameters that are needed to deploy Kafka to a node.
2. The `vars/{{cloud_name}}.yml` file, which contains the default values for the parameters needed instantiate a node (or nodes) based on a specific cloud (by name, where the name is something like `aws`, `vagrant`, `openstack`, etc.).  Currently this file is mostly empty.
3. The `vars/{{tenant_name}}-tenant.yml` file, which contains default values for the parameters that used in deployments for a specific tenant (again, by tenant name).
4. The `vars/{{project_name}}-project.yml` file, which contains the default values for the parameters used in deployments for a specific project.
5. The `vars/{{domain_name}}-domain.yml` file, which contains the default values for the parameters used in deployments to a specific domain (eg. `test`, `dev`, `prod`, etc.)

The order that these files are loaded in defines the precedence if a parameter is defined at more than one level in this hierarchy, and that precedence is tied to how specific the level is at which the parameter value is defined.  If a parameter is defined at both the application and cloud level, for example, then the cloud-level definition for that parameter will be used (since the application-level definitions are default values that can be used by **any** deployment of that application).  Similarly, if a parameter is defined at both the tenant and project level, then the project-level definition will be used.  If a parameter is defined at the domain level, the value set at that level will supercede values for that same parameter that might be defined at any other level.  Graphically, then, the order of precedence is as follows:

```
   +-------------+     +-------+     +--------+     +---------+     +--------+
   | application | ==> | cloud | ==> | tenant | ==> | project | ==> | domain |
   +-------------+     +-------+     +--------+     +---------+     +--------+
```

where parameters defined further to the right override parameters defined further to the left in this diagram.  In the `eg-customer` repository, these 5 files are defined as follows:

1. The `dn-kafka/vars/kafka.yml` file:

	```yaml
	# (c) 2016 DataNexus Inc.  All Rights Reserved
	#
	# Defaults that are necessary for all deployments of
	# kafka
	---
	application: kafka
	# the distribution of Kafka that should be installed (apache or confluent)
	kafka_distro: confluent
	# the interface Kafka should listen on when running
	kafka_iface: eth0
	# the following parameters are only used when provisioning an instance
	# of the apache distribution, but are uncommented here (regardless) to
	# provide reasonable default values when provisioning via Vagrant (where
	# the distribution being provisioned may be different from the default)
	scala_version: "2.11"
	kafka_version: "0.10.1.0"
	kafka_url: "https://www-us.apache.org/dist/kafka/{{kafka_version}}/kafka_{{scala_version}}-{{kafka_version}}.tgz"
	kafka_dir: "/opt/kafka"
	# this value is only used when installing the confluent distribution,
	# but is uncommented here so that it can be used if a confluent distribution
	# is chosen when provisioning via Vagrant
	confluent_version: "3.1"
	# these parameters are used for both confluent and apache distributions
	kafka_topics: ["metrics", "logs"]
	kafka_package_list: ["java-1.8.0-openjdk", "java-1.8.0-openjdk-devel"]
	```

2. The `vars/vagrant.yml` file:

	```yaml
	# (c) 2016 DataNexus Inc.  All Rights Reserved
	#
	# Variables that are used for all deployments
	# (split apart by cloud type: 'aws', 'openstack', etc.)
	---
	cloud: vagrant
	```

3. The `vars/demo-tenant.yml` file:

	```yaml
	# (c) 2016 DataNexus Inc.  All Rights Reserved
	#
	# Variables that are used for all deployments in a tenant
	# environment (proxies, extra packages per node, etc.)
	---
	# proxy-related variables (values are retrieved from the corresponding
	# environment variables, if they are set)
	proxy: "{{ lookup('env','http_proxy') }}"
	no_proxy: "{{ lookup('env','no_proxy') }}"
	proxy_username: "{{ lookup('env','proxy_username') }}"
	proxy_password: "{{ lookup('env','proxy_password') }}"
	proxy_env:
	  http_proxy: "{{proxy}}"
	  no_proxy: "{{no_proxy}}"
	  proxy_username: "{{proxy_username}}"
	  proxy_password: "{{proxy_password}}"
	# the list of packages that should be installed on every node
	default_packages:
	  - java-1.8.0-openjdk
	  - net-tools
	install_packages_by_tag:
	  kafka:
	    - bind-utils
	  solr:
	    - nmap
	    - tomcat
	```

4. The `vars/demo-project.yml` file:

	```yaml
	# (c) 2016 DataNexus Inc.  All Rights Reserved
	#
	# Variables that are used for all deployments in given
	# project (project-greenfield, customer-demo, etc.)
	---
	scala_version: 2.11
	kafka_version: 0.10.1.0
	kafka_distro: apache
	kafka_url: "https://10.0.2.2/dist/kafka/{{kafka_version}}/kafka_{{scala_version}}-{{kafka_version}}.tgz"
	kafka_dir: "/opt/kafka"
	# confluent_version: "3.1"
	kafka_iface: eth1
	kafka_topics: ["metrics", "logs"]
	```

5. The `vars/development-domain.yml` file:

	``` yaml
	# (c) 2016 DataNexus Inc.  All Rights Reserved
	#
	# Variables that are used for all deployments within
	# a given domain ('production', 'test', 'development', etc.)
	---
	domain: development
	```

As you can see, the cloud and domain level files are mostly empty in the `eg-customer` example.  In addition, the `eg-customer` example only shows a cloud-level file for deployment via Vagrant; deployment to an AWS or OpenStack environment would require creation of additional cloud-level configuration files in the `vars` directory.

The `vars/demo-tenant.yml` file is used in this example project to define some variables that are used for all deployments for the Demo tenant, namely a set of variables that are needed to access resources through a proxy firewall, a list of default packages that should be installed on all nodes (regardless of type; in this example we're configuring the `site.yml` playbook so that the `net-tools` package and the `java-1.8.0-openjdk` package, which contains a Java runtime environment, willb e installed on all nodes, regardless of node type), and a hash of the extra packages that should be installed on nodes of a specific type (in this example, we have said that the `bind-utils` package should be installed on all `kafka` nodes while the `solr` nodes should have the `nmap` and `tomcat` packages installed on them).

Finally, in the `vars/demo-project.yml` file, we override many of the defaults that are defined in the `dn-kafka/vars/kafka.yml` file, setting up the underlying `dn-kafka` role so that it will deploy an instance of the Apache Kafka distribution that is downloaded from a local web server (with an IP address of 10.0.2.2) for all Kafka nodes that are provisioned for the Demo project.

This schema can be quite easily extended to support multiple clouds, tenants, projects, or domains, with minimal work on the part of those managing these environments.  Basically, only the parameters that need to be redefined to support deployments at each level will need to be added to each file; all other parameters can be pulled from the default values that are defined for each role.

# Using the playbook to deploy Kafka
With the parameters defined in the five "configuration" files shown above, running the playbook to deploy Kafka to a node becomes quite simple.  For example, the following command will deploy the Apache Kafka distribution to a single node with the IP address "192.168.34.8":

```bash
$ export KAFKA_ADDR="192.168.34.8"
$ ansible-playbook -i "${KAFKA_ADDR}," -e "{ host_inventory: ['${KAFKA_ADDR}']}" site.yml
```

The values for all of the other parameters that are needed to deploy Kafka to this node are taken from the "configuration" files that have been shown, above.  That makes it much simpler to track what has been deployed to each node in a given environment, since the configuration parameters that are needed for the deployments can be managed in files that are maintained under version control in a local clone or fork of this repository.

# Deployment via vagrant
The deployment using Vagrant becomes simple as well, since only the `host_inventory` needs to be defined.  All other parameters are taken from the "configuration" files included in this repository.  The Vagrantfile contained in this repository can easily be used to create a VM with the same IP address shown above and then provision that node with an instance of the Apache Kafka distribution in a single command:

```bash
$ vagrant -k="192.168.34.8" up
```

The Vagrantfile contained in this repository also supports breaking up the VM creation and VM provisioning into two separate commands (something that is necessary to integrate the workflow used to create and provision VMs under Vagrant with a process that also supports creation and provisioning of a VM under a different cloud provider, like AWS.  To create a new VM at the same address that is shown above, you would run a command that looks something like this:

```bash
$ vagrant -k="192.168.34.8" up --no-provision
```

Once that command completed (and the VM was created), you could easily run the same `site.yml` playbook used in the `vagrant ... up` command shown above by running a command that looked something like this:

```bash
$ vagrant -k="192.168.34.8" provision
```
