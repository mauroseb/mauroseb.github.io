---
layout: posts
title: "OpenStack Director Best Practices"
date: 2021-11-29
categories: [blog]
tags: [ openstack, tripleo, director, osp, networking ]
author: mauroseb
excerpt_separator: "<!-- more -->"
---
### Intro

Not to long ago I had to help one of my favourite customers to migrate an OpenStack deployment from an RPM based deployment where they used their own tooling to deploy into a Director (Triple-O) based deployment.
Part of the work was purely educational about the features, the dos and donts, and the best practices using this great tool.
After I delivered this content to a wider audience and realized some of this points are not obvious for many, I decided to create this post based on that information in case it can help someone else.




<!-- more -->


### Director Best Practices

So lets begin with some best practices and in a later section I will cover some common mistakes when using Director.

#### 1. Version Control for the stack home files

As you know the undercloud where Director runs usually has a stack user from where all the platform operations are run. This is a regular user with sudo privileges, and within its home dire you will find:

 * The undercloud.conf
 * The instack-env.json (or YAML)
 * The Tripleo Heat Templates and environment files for the overcloud
 * Scripts that you may need to deploy, upgrade, scale out, etc.
 * Other

This valuable set of data, should be under strict version control, to make sure no intended changes are going through to overcloud. For instance I have witnessed environments where multiple groups were accesing this files in production (enigneering teams and operation teams) and due to some changes made by one team to these files, the other made a deployment in the overcloud and an unintended change.


#### 2. Sandbox and Testing Environments

Believe it or not, some customers I found in the wild were only using a production environment, meaning that any changes to the enviornment were not tested at all. The consequences were obvious, deployment issues, upgrade issues, changes in configuration generating unintenteded issues, issues during scale outs, basically any operation is a problem.

For a real production environment this is unacceptable, specially complex products like openstack need thorough testing in a pre-production, testing or sandbox environment, to rehearse any change before running it in production (multiple times!).

Creating a testing environment is not cheap, and requires in the best case, similar hardware to production, in order to avoid surprises during an upgrade like for instance, the deprecation of a NIC card at OS (RHEL) could affect a compute node availability.

For a cheaper environment for sandbox it is normal to use a virtual environment, virtualized on top of even in a phyisical OpenStack deployment or KVM VMs.

You can check Hextuple-O project mainly maintained by Chris Janiszewski, that holds a set of Ansible playbooks and roles to deloy OpenStack on top of OpenStack:

 * https://github.com/OOsemka/hextupleo/
 * https://www.youtube.com/watch?v=NPZon911V5A



#### 3. Use Validation Framework

Red Hat OpenStack Platform inclues a validation framework that you can use to verify the requirements and functionality of the undercloud and overcloud.

The target is to take over the whole validations we might want to run before, during and after a deployment, an update or an upgrade task.

There are two types of validations.

 1. **Manual:** through the command set:

{% highlight console %}
$ openstack tripleo validator
{% endhighlight %}

 2. **In-flight:** which execute automatically during the deployment process


In the following example I list the validations available and do a manual run of a single validation.
After I can search the history of runs to see previous outputs.

{% highlight console %}

$ source ~/stackrc
$ openstack tripleo validator list
$ openstack tripleo validator run --validation check-ram
$ openstack tripleo validator run --group prep
$ openstack tripleo validator show history --validation ntp
$ openstack tripleo validator show run --full 7380fed4-2ea1-44a1-ab71-aab561b44395

{% endhighlight %}


#### 4. Post Deployment Testing

On top of the common functional validations, after running a deployment, and upgrade or other operational activities like configuration changes, it is a great practice to run some non-functional tests to measure the platform overall performance and response.

In an ideal world, you may have a pipeline that gets triggered after a change in the triple-o configuration git repository, new deployment of a virtual openstack is created, and funcional and non-functional tests are run against this new environment. If no errors are detected this changes can be admitted in a production.

For that there are a few projects that can help:

 * [Browbeat Project (Rally and Shaker)](https://browbeat.readthedocs.io/): Browbeat is a project that automates the use of Rally and Shaker projects. Mainly, Rally is a testing project aimed to profiling and detecting peformance changes, by running hundreds of parallel requests to the OpenStack API and measuring the response in time. It is normal to include this tests as part of a CI/CD pipeline triggered by changes in the OpenStack configuration in a test environment of course.

{% highlight console %}
$ git clone https://github.com/openstack/browbeat.git
$ cd browbeat/ansible/
$ ./generate_tripleo_inventory.sh -l
$ vi install/group_vars/all.yml
$ git diff install/group_vars/all.yml
-overcloudrc: "{{home_dir}}/overcloudrc"
+overcloudrc: "{{home_dir}}/maurorc"
-dns_server: 8.8.8.8
+dns_server: 10.9.71.7
$ ansible-playbook -i hosts.yml install/browbeat.yml 
$ ansible-playbook -i hosts.yml install/shaker_build.yml
{% endhighlight %}

 * [Tempest](https://docs.openstack.org/tempest/latest/): The OpenStack integration test suite. It has a battery of test for OpenStack API, even though it may be overkill in some ocations, if you can afford to run a smoke test after important operations and changes can help to devise potential issues even in your CI/CD pipelines.

{% highlight console %}
$ sudo yum -y install openstack-tempest
$ sudo yum install python-keystone-tests-tempest python-horizon-tests-tempest python-neutron-tests-tempest python-cinder-tests-tempest python-telemetry-tests-tempest
$ . admin_rc
$ tempest init mytempest
$ cd mytempest/
$ ls -ltr
total 0
drwxrwxr-x. 2 stack stack   6 Nov 22 11:43 tempest_lock
drwxrwxr-x. 2 stack stack   6 Nov 22 11:43 logs
drwxr-xr-x. 2 stack stack 130 Nov 22 11:43 etc
$ tempest workspace list
+-----------+-----------------------+
| Name      | Path                  |
+-----------+-----------------------+
| mytempest | /home/stack/mytempest |
+-----------+-----------------------+
$ openstack network list
+--------------------------------------+------------+--------------------------------------+
| ID                                   | Name       | Subnets                              |
+--------------------------------------+------------+--------------------------------------+
| 57fbc7f2-7e13-48ae-84d2-28656c9c710d | tempestnet | 524efcda-e986-4b77-83f0-a3624bf37f63 |
| c989185b-3a17-4ca2-8f1d-d3691c242d20 | external   | f4ef2ed1-cea7-462a-a212-c295f22e0e2d |
+--------------------------------------+------------+--------------------------------------+
$ discover-tempest-config --deployer-input ~/tempest-deployer-input.conf --debug --create --network-id c989185b-3a17-4ca2-8f1d-d3691c242d20
$ openstack extention list
$ openstack extension list --network
$ tempest verify-config -o output.txt
$ tempest run -l
$ tempest run --smoke
{3} tempest.api.compute.flavors.test_flavors.FlavorsV2TestJSON.test_get_flavor [0.643206s] ... ok
{3} tempest.api.compute.flavors.test_flavors.FlavorsV2TestJSON.test_list_flavors [0.145820s] ... ok
{0} tempest.api.compute.security_groups.test_security_group_rules.SecurityGroupRulesTestJSON.test_security_group_rules_create [3.934657s] ... ok
{0} tempest.api.compute.security_groups.test_security_group_rules.SecurityGroupRulesTestJSON.test_security_group_rules_list [3.741193s] ... ok
{1} tempest.api.compute.servers.test_attach_interfaces.AttachInterfacesTestJSON.test_add_remove_fixed_ip [28.768181s] ... ok
{0} tempest.api.compute.security_groups.test_security_groups.SecurityGroupsTestJSON.test_security_groups_create_list_delete [8.184534s] ... ok
{2} tempest.api.compute.servers.test_create_server.ServersTestBootFromVolume.test_list_servers [0.311029s] ... ok
...
======
Totals
======
Ran: 130 tests in 758.0000 sec.
- Passed: 125
- Skipped: 4
- Expected Fail: 0
- Unexpected Success: 0
- Failed: 1
Sum of execute time for each test: 927.6400 sec.

==============
Worker Balance
==============
- Worker 0 (27 tests) => 0:08:43.900337
- Worker 1 (32 tests) => 0:10:07.943554
- Worker 2 (37 tests) => 0:09:25.850420
- Worker 3 (34 tests) => 0:12:18.011336
{% endhighlight %}





#### 5. Keep Config Simple

Keep a clean stack home.  Only add or change configuration based on what is really necessary to meet the platform objectives.
Split configuration in smaller environment files grouped by purpose to simplify the day to day management of them.
Try to stick to documentation conventions like file names as much as possible so Red Hat support can quickly understand where to find what configuration.


#### 6. Respect Tripleo Heat Templates Order and Location

##### RECOMMENDED APPROACH ADDING TEMPLATES

 A. Include the original env file
{% highlight console %}
$ openstack overcloud deploy --templates \
  -e [your-environment-files] \
  -e /usr/share/openstack-tripleo-heat-templates/environments/services/octavia.yaml
{% endhighlight %}

 B. Create/Copy env file with your customizations
{% highlight console %}
parameter_defaults:
  OctaviaLoadBalancerTopology: "ACTIVE_STANDBY"
{% endhighlight %}

 C. Include customized env file in deployment script AFTER original env file
{% highlight console %}
$ openstack overcloud deploy --templates \
  -e [your-environment-files] \
  -e /usr/share/openstack-tripleo-heat-templates/environments/services/octavia.yaml \
  -e /home/stack/templates/octavia.yaml
{% endhighlight %}

 D. Run deployment script

##### RECOMMENDED APPROACH FOR INITIALIZING TEMPLATES

 A. Create the templates dir
{% highlight console %}
$ mkdir ~/templates
{% endhighlight %}

 B. Generate Roles YAML file
{% highlight console %}
$ openstack overcloud roles generate Controller Compute Networker -o ~/templates/roles_data.yaml
{% endhighlight %}

 C. Generate Networks YAML file
{% highlight console %}
$ cp /usr/share/openstack-tripleo-heat-templates/network_data.yaml ~/templates
(modify ~/templates/network_data.yaml with your specific subnets, vlans, etc.)
{% endhighlight %}

 D. Process roles and network param files to create your templates
{% highlight console %}
$ cd  /usr/share/openstack-tripleo-heat-templates
$ mkdir /tmp/templates_generated
$ ./tools/process-templates.py -n ~/templates/network_data.yaml -r ~/templates/roles_data.yaml -o ~/templates
{% endhighlight %}

 E. Run deployment script


##### COMMON MISTAKES MANAGING TEMPLATES

 * **Common mistake #1:** Copy the default environment file from /usr/share/openstack-tripleo-heat-templates/ to the ~stack/templates/ directory and edit it there. On updates these environment files may change and create problems.

 * **Common mistake #2:** Use a non default templates directory (/usr/share/openstack-tripleo-heat-templates) which would force to merge changes from the software and the manual ones every time there is an update.

 * **Common mistake #3:** Include the stack home version of the modified template before the default environment file in /usr/share/openstack-tripleo-heat-templates. The last file (default) takes precedence, undoing any changes of previously included file.

 * **Common mistake #4:** Order of the roles_data.yaml, network_data.yaml, container-images-prepare.yaml and the rest of the environment files.

 * **Common mistake #5:** Introduce changes manually in the overcloud without reflecting them in Director configuration.

 * **Common mistake #6:** Incorrect server role definitions. Too many roles.


#### 7. Director on a VM

The main advantage of running Director on a VM is the ability to take snapshots and return to a previously known state for the same reason.
Still a snapshot may not save you from restoring the MySQL DB if there is data corruption or it is needed to return to a previous stage.
Follow the [B&R](https://access.redhat.com/documentation/en-us/red_hat_openstack_platform/16.2/html/backing_up_and_restoring_the_undercloud_and_control_plane_nodes) procedure in any case.


#### 8. Take Advantage of TripleO Ansible Collection

Director can run Ansible-based automation in your Red Hat OpenStack Platform (RHOSP) environment. Director uses the **tripleo-ansible-inventory** command to generate a dynamic inventory of nodes in your environment.

 * **Dynamic Inventory**

{% highlight console %}
$ source ~/stackrc
$ tripleo-ansible-inventory --list --stack mauro
{"_meta": {"hostvars": {"ctl-01": {"ansible_host": "172.16.0.100", "deploy_server_id": "d3139bf2-5a5f-449c-8a64-8fac8696bfd2", "ctlplane_ip": "172.16.0.100", "internal_api_ip": "10.20.0.100", "storage_ip": "10.40.0.100", "storage_mgmt_ip": "10.50.0.100", "storage_nfs_ip": "10.80.0.100", "tenant_ip": "10.30.0.100", "external_ip": "10.1.0.90", "internal_api_hostname": "ctl-01.internalapi.lab.local", "storage_hostname": "ctl-01.storage.lab.local", "storage_mgmt_hostname": "ctl-01.storagemgmt.lab.local", "storage_nfs_hostname": "ctl-01.storagenfs.lab.local", "tenant_hostname": "ctl-01.tenant.lab.local", "external_hostname": "ctl-01.external.lab.local"
..._"
{% endhighlight %}

 * **Create Static Inventory and Run Playbooks**
{% highlight console %}
$ tripleo-ansible-inventory --static-yaml-inventory hosts.yaml --stack mauro
$ ansible all -i hosts.yaml -m shell -a "uname -a"
undercloud | CHANGED | rc=0 >>
Linux undercloud-osp162.hextupleo.lab 4.18.0-305.19.1.el8_4.x86_64 #1 SMP Tue Sep 7 07:07:31 EDT 2021 x86_64 x86_64 x86_64 GNU/Linux
com-01 | CHANGED | rc=0 >>
Linux com-01 4.18.0-305.12.1.el8_4.x86_64 #1 SMP Mon Jul 26 08:06:24 EDT 2021 x86_64 x86_64 x86_64 GNU/Linux
ceph-01 | CHANGED | rc=0 >>
Linux ceph-01 4.18.0-305.12.1.el8_4.x86_64 #1 SMP Mon Jul 26 08:06:24 EDT 2021 x86_64 x86_64 x86_64 GNU/Linux
ctl-03 | CHANGED | rc=0 >>
Linux ctl-03 4.18.0-305.12.1.el8_4.x86_64 #1 SMP Mon Jul 26 08:06:24 EDT 2021 x86_64 x86_64 x86_64 GNU/Linux
ctl-02 | CHANGED | rc=0 >>
Linux ctl-02 4.18.0-305.12.1.el8_4.x86_64 #1 SMP Mon Jul 26 08:06:24 EDT 2021 x86_64 x86_64 x86_64 GNU/Linux
ctl-01 | CHANGED | rc=0 >>
Linux ctl-01 4.18.0-305.12.1.el8_4.x86_64 #1 SMP Mon Jul 26 08:06:24 EDT 2021 x86_64 x86_64 x86_64 GNU/Linux
ceph-02 | CHANGED | rc=0 >>
Linux ceph-02 4.18.0-305.12.1.el8_4.x86_64 #1 SMP Mon Jul 26 08:06:24 EDT 2021 x86_64 x86_64 x86_64 GNU/Linux
ceph-03 | CHANGED | rc=0 >>
Linux ceph-03 4.18.0-305.12.1.el8_4.x86_64 #1 SMP Mon Jul 26 08:06:24 EDT 2021 x86_64 x86_64 x86_64 GNU/Linux
{% endhighlight %}

 * **[Collection](https://docs.openstack.org/tripleo-operator-ansible/latest/) available for many TripleO tasks**:

### Common Mistakes

#### 1. Unsupported Hardware

Always validate hardware (servers and components) against: <https://catalog.redhat.com/hardware>


#### 2. Undercloud


 * Undersized undercloud for large deployments
 * Full-disk or out-of-memory conditions
 * Telemetry enabled. Disable unless required otherwise it can create problems in the long term.
 * No room for containers in /var/lib/image-serve if undercloud is the registry
 * Preferably do not use undercloud as registry
 * Short timeouts for deployment
 * Short timeout for keystone token
 * Undercloud CA not injected in overcloud nodes.


#### 3. Network Configuration

 * Overlappings among network allocation pools and VIPs or predictable IPs
 * H/W to config mismatch
 * Not specifying all NICs in the nodes in nic-configs
 * Bonding Options mixed (using OvS bonding options for Linux bonds or viceversa)
 * OvS bonds for control plane and storage networks
 * Using OvS bond in mode balance-tcp on DPDK ports
 * Not using first NICs for control plane network
 * Capacity planning (Storage, Tenant network)
 * Small MTU sizes
 * NTP unset or misconfigured
 * Duplicated IPs with external equipment


#### 4. Scheduler Issues: "No valid host was found"

This is a typical error that can represent many different causes during a deployment.
Following are some examples.

 * Nodes healthy and active (not in maintenance)
 * TripleoCapabiltiesFilter mismatch of scheduler hints and properties in the nodes
 * Resources available presented by Ironic vs what scheduler wants
 * Flavor properties mismatch with node’s
 * Nova Placement complaining
 * Node Blacklisted
 * Deployment interface issues
 * Power management timeouts (IPMI)
 * Compute service down


#### 5. Log forwarding and/or Monitoing Disabled

 * For a production environment there should be proper monitoing and log forwarding in place.
 * Red Hat OpenStack provides Service Telemetry Framework (STF) for monitoring purposes.
 * Red Hat OpenStack provides rsyslog to configure log shipping to an external facility if needed.


#### 6. Storage

 * Allowing qcow2 images for Ceph backends is a common mistake. As it is known format should be raw for exploit Ceph Copy-On-Write capabilities.
 * Not using node cleaning in Director may bring issues during a redeployment of Ceph for instance.
 * Not setting root disk hints for multi disk nodes like Ceph OSD nodes, the consequence is that a root disk could be picked up randomly from the available ones.


#### 7. Telemetry

 * Legacy telemetry services enabled (gnocchi, aodh, panko, ceilometer) without proper capacity planning and resources can create serious problems for the storage backend after some time the platform is up and running.


#### 8. Other Mistakes

 * Executing deployment script detached. Never do this.
 * Fencing not enabled. This is a requirement to receive support.
 * No backup and restore procedures in place. It is a must for a production environment.
 * Inconsistent RPM and container image sources. All software in use by the platform should be aligned in release date as much as possible: either Containers, RPMs, ISO images. Many issues can arise for instance if you are using old contianer images and newer packages, or the RHEL packages (like kernel or OpenvSwitch) are newer than the OpenStack RPMs. Careful management of software is recommeneded, this can be achieved by the use of Red Hat Satellite.



That's all I have for now.

Cheers!








