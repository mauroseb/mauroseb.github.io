---
layout: posts
title: "Software RAIDs with Red Hat OpenStack Platform"
date: 2022-01-30
categories: [blog]
tags: [ osp, openstack, software-raids, tripleo ]
author: mauroseb
excerpt_separator: "<!-- more -->"
---
### Intro

While helping one of my favourite customers to migrate from an OpenStack cluster deployed with their own tooling to a Director based deployement, they asked me how to get software RAIDs using Director (RHOSP version 16.2). They have a server landscape without RAID controllers therefore they need to rely in tools like the md driver that GNU/Linux offers to setup the RAID and avoid a SPOF like a disk failure to bring down a whole overcloud node.

As you know, Director is the Red Hat's downstream of TripleO deployment framework for OpenStack, and it is based in a broad set of technologies to achieve its extremely versatile deployment options. A typical configuration in Director is achieved by passing environment YAML files and Tripleo Heat Templates to the deployment command. However achieving software raids configuration is quite more complex than that and I will explain the process throughout this article.

{% include note.html content="Using software RAIDs with Red Hat OpenStack Platform at the time of writting this article is not yet officially supported, and you may need to ask for a support exception to run it in production." %} 


<!-- more -->

### Deployment Workflow

Director provisions baremetal nodes by using the great Ironic project.
The greater goal of Ironic is to treat baremetal hardware as instances from a cloud, and Director makes use of it to provision and manage the nodes where it will be installing OpenStack software on.

In a typical very summarized workflow of deploying an overcloud, the following tasks are Ironic's responsibilities.

 1. An administrator enrolls a baremetal node in Ironic (i.e. via an import) and if the power management and some other checks responds well it will be set to **manageable** state.

 2. An administrator triggers the **introspection** of the manageable nodes. Then the ironic-inspector service, boots the node with a discovery ramdisk image to detect the node's hardware properties, network configuration, etc. in order to populate Ironic's database and set the node to available if all went well.

 3. An administrator triggers the **deployment** of the overcloud. The nodes in available state will be booted with an ironic-python-agent ramdisk image that will eventually burn the overcloud image stored in Glance to the node's disk, to finally reboot the node from the image now in its own disk.

Of course the process is a lot more complex than that[^1][^2], but for the interest of this article this is all we need to know for now.

For software RAIDs configuration, when the image is burned to disk, it has to be done directly on the md virtual device.

So this default workflow has to be modified a bit. Before deployment I will invoke the node **clean** operation with some special steps to setup the software RAID, that will create the RAID right before deployment. Afterwards, during deployment, just let Ironic pick the md virtual device to burn the overcloud image. The TripleO documentation[^3] describes slightly the process and limitations.


### Limitations

Before I start, the current limitations of the process I am about to describe should be clear:

 1. The overcloud image must be a **whole-disk** image, as oppossed to the default "partition" image.
 2. The first partition of the whole-disk image to be used must be the **root** partition.
 3. The overcloud image has to have **mdadm** installed in it to be able to work with the software RAID.
 4. The BIOS should be set to Legacy mode (UEFI is not possible at the time of writting)
 5. LVM in the whole-disk image is not supported either

Check Appendix 1 to see the whole-disk image creation process.


### Procedure

Now lets walk through the entire process to set it up.

#### STEP 1. Enable the software RAID interface in Director

By default the software RAID agent is disabled. Include __agent__ in the list of values for  __enabled_raid_interfaces__ in undercloud.conf.

{% highlight bash %}
[DEFAULT]
local_ip = 172.16.0.2/24
local_mtu = 8938
local_interface = eth1
undercloud_public_host = 10.1.0.11
undercloud_admin_host = 172.16.0.4
container_images_file = /home/stack/containers-prepare-parameter.yaml
undercloud_debug = true
undercloud_hostname = undercloud-osp162.hextupleo.lab
overcloud_domain_name = hextupleo.lab
undercloud_nameservers = 10.9.71.7
undercloud_ntp_servers = 10.9.71.7
enabled_hardware_types = ipmi,redfish,ilo,idrac,fake-hardware,manual-management
enabled_raid_interfaces = agent,idrac,no-raid
custom_env_files = /home/stack/templates/undercloud_custom_env.yaml
clean_nodes = true

[ctlplane-subnet]
gateway = 172.16.0.2
cidr = 172.16.0.0/24
masquerade = true
dhcp_start = 172.16.0.10
dhcp_end = 172.16.0.59
inspection_iprange = 172.16.0.60,172.16.0.99
{% endhighlight %}

#### STEP 2. Run undercloud install to deploy the new setting

{% highlight console %}
(undercloud) [stack@undercloud-osp162 ~]$ openstack undercloud install
{% endhighlight %}

#### STEP 3. Set the RAID interface in the node to agent

In this case the overcloud_compute1 node has two disks that I will use to create a software RAID. The rest of the nodes we can assume they have a single disk.

{% highlight console %}
(undercloud) [stack@undercloud-osp162 ~]$ openstack baremetal node set --raid-interface agent overcloud_compute1 
{% endhighlight %}

#### STEP 4. Set target RAID config

There is a specific field in Ironic to set what is the target configuration that we want for the node. This can include, which disks we want to use, the RAID mode, if it is a root disk or not, the size of the disk to use, and some other options.

{% highlight console %}
(undercloud) [stack@undercloud-osp162 ~]$ openstack baremetal node set --target-raid-config '{"logical_disks": [{"raid_level":"1","size_gb":"MAX","controller":"software","is_root_volume":true,"physical_disks":[ "/dev/vda", "/dev/vdb"]}]}' overcloud_compute1

(undercloud) [stack@undercloud-osp162 ~]$ openstack baremetal node show -c raid_config -c raid_interface -c target_raid_config overcloud_compute1                                                                  
+--------------------+----------------------------------------------------------------------------------------------------------------------------------------------------------+                                  
| Field              | Value                                                                                                                                                    |                                  
+--------------------+----------------------------------------------------------------------------------------------------------------------------------------------------------+                                  
| raid_config        | {}                                                                                                                                                       |                                  
| raid_interface     | agent                                                                                                                                                    |                                  
| target_raid_config | {'logical_disks': [{'raid_level': '1', 'size_gb': 'MAX', 'controller': 'software', 'is_root_volume': True, 'physical_disks': ['/dev/vda', '/dev/vdb']}]} |                                  
+--------------------+----------------------------------------------------------------------------------------------------------------------------------------------------------+     
{% endhighlight %}

#### STEP 5. Set the node to manageable state to setup RAID

It is important to know that to run the **clean** operation the node has to be moved from **available** state back to **manageable** state.

{% highlight console %}
(undercloud) [stack@undercloud-osp162 ~]$ openstack baremetal node manage overcloud_compute1
(undercloud) [stack@undercloud-osp162 ~]$ openstack baremetal node list
+--------------------------------------+-----------------------+---------------+-------------+--------------------+-------------+                                                                                 
| UUID                                 | Name                  | Instance UUID | Power State | Provisioning State | Maintenance |                                                                                 
+--------------------------------------+-----------------------+---------------+-------------+--------------------+-------------+                                                                                 
| 5305b322-b7dd-491d-bb84-dca8a4557ed6 | overcloud_ceph3       | None          | power off   | available          | False       |                                                                                 
| 8f6794d0-25a2-4efd-9be7-60a730c856ca | overcloud_ceph2       | None          | power off   | available          | False       |                                                                                 
| 5825b542-65f4-44cc-9775-ffdebcf3edd7 | overcloud_ceph1       | None          | power off   | available          | False       |                                                                                 
| 9ed03c40-2e19-442c-a132-244b7f04c9f9 | overcloud_networker1  | None          | power off   | available          | False       |                                                                                 
| 47f0fea1-3bd6-41df-9bc2-32f196d0aa49 | overcloud_controller1 | None          | power off   | available          | False       |                                                                                 
| 2bcf72a7-082a-4109-bba9-2b6cf914d72b | overcloud_compute1    | None          | power off   | manageable         | False       |                                                                                 
| e112ab32-a9f2-4138-a668-bd9cc83f6364 | overcloud_controller2 | None          | power off   | available          | False       |                                                                                 
| 4564d7f2-0468-4c13-9e82-9105028ca655 | overcloud_controller3 | None          | power off   | available          | False       |                                                                                 
+--------------------------------------+-----------------------+---------------+-------------+--------------------+-------------+                         
{% endhighlight %}

#### STEP 6. Clean baremetal node with RAID creation steps

Now we can safely run the clean operation adding three steps: delete_configuration (deletes the previous RAID), earse_devices_metadata (deletes any partition data), create_configuration (creates the RAID).

{% highlight console %}
(undercloud) [stack@undercloud-osp162 ~]$ cat soft-raid-clean-steps.json
[{
  "interface": "raid",
  "step": "delete_configuration"
 },
 {
  "interface": "deploy",
  "step": "erase_devices_metadata"
 },
 {
  "interface": "raid",
  "step": "create_configuration"
 }]

(undercloud) [stack@undercloud-osp162 ~]$ openstack baremetal node clean --clean-steps soft-raid-clean-steps.json overcloud_compute1

(undercloud) [stack@undercloud-osp162 ~]$ openstack baremetal node show -c raid_config -c raid_interface -c target_raid_config -c capabilities overcloud_compute1
+--------------------+--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| Field              | Value                                                                                                                                                                                                  |
+--------------------+--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| raid_config        | {'logical_disks': [{'raid_level': '1', 'size_gb': 'MAX', 'controller': 'software', 'is_root_volume': True, 'physical_disks': ['/dev/vda', '/dev/vdb']}], 'last_updated': '2022-01-05 09:12:43.279903'} |
| raid_interface     | agent                                                                                                                                                                                                  |
| target_raid_config | {'logical_disks': [{'raid_level': '1', 'size_gb': 'MAX', 'controller': 'software', 'is_root_volume': True, 'physical_disks': ['/dev/vda', '/dev/vdb']}]}                                               |
+--------------------+--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
{% endhighlight %}


As we can observe now the raid_config field matches what I previously set as the target_raid_config.

Lets check the logs of ironic_python_agent to see what happened:

{% highlight bash %}
…
Jan 05 09:03:43 host-172-16-0-35 ironic-python-agent[832]: 2022-01-05 09:03:43.529 832 DEBUG root [-] Creating md device /dev/md0 on ['/dev/vda1', '/dev/vdb1'] create_configuration /usr/lib/python3.6/site-packages/ironic_python_agent/hardware.py:1675
Jan 05 09:03:43 host-172-16-0-35 ironic-python-agent[832]: 2022-01-05 09:03:43.531 832 DEBUG oslo_concurrency.processutils [-] Running cmd (subprocess): mdadm --create /dev/md0 --force --run --metadata=1 --level 1 --raid-devices 2 /dev/vda1 /dev/vdb1 execute /usr/lib/python3.6/site-packages/oslo_concurrency/processutils.py:379
Jan 05 09:03:43 host-172-16-0-35 kernel: md/raid1:md0: not clean -- starting background reconstruction
Jan 05 09:03:43 host-172-16-0-35 kernel: md/raid1:md0: active with 2 out of 2 mirrors
Jan 05 09:03:43 host-172-16-0-35 kernel: md0: detected capacity change from 0 to 64387809280
Jan 05 09:03:43 host-172-16-0-35 kernel: md: resync of RAID array md0
Jan 05 09:03:43 host-172-16-0-35 ironic-python-agent[832]: 2022-01-05 09:03:43.553 832 DEBUG oslo_concurr
ency.processutils [-] CMD "mdadm --create /dev/md0 --force --run --metadata=1 --level 1 --raid-devices 2 /dev/vda1 /dev/vdb1" returned: 0 in 0.022s execute /usr/lib/python3.6/site-packages/oslo_concurrency/processutils.py:416
Jan 05 09:03:43 host-172-16-0-35 ironic-python-agent[832]: 2022-01-05 09:03:43.556 832 DEBUG ironic_lib.utils [-] Execution completed, command line is "mdadm --create /dev/md0 --force --run --metadata=1 --level 1 --raid-devices 2 /dev/vda1 /dev/vdb1" execute /usr/lib/python3.6/site-packages/ironic_lib/utils.py:101
Jan 05 09:03:43 host-172-16-0-35 ironic-python-agent[832]: 2022-01-05 09:03:43.559 832 DEBUG ironic_lib.utils [-] Command stdout is: "" execute /usr/lib/python3.6/site-packages/ironic_lib/utils.py:103
Jan 05 09:03:43 host-172-16-0-35 ironic-python-agent[832]: 2022-01-05 09:03:43.562 832 DEBUG ironic_lib.utils [-] Command stderr is: "mdadm: array /dev/md0 started.
                                                           " execute /usr/lib/python3.6/site-packages/ironic_lib/utils.py:104
Jan 05 09:03:43 host-172-16-0-35 ironic-python-agent[832]: 2022-01-05 09:03:43.565 832 INFO root [-] Successfully created Software RAID
Jan 05 09:03:43 host-172-16-0-35 ironic-python-agent[832]: 2022-01-05 09:03:43.568 832 INFO root [-] Clean step completed: {'interface': 'raid', 'step': 'create_configuration', 'abortable': False, 'priority': 0}, result: {'logical_disks': [{'raid_level': '1', 'size_gb': 'MAX', 'controller': 'software', 'is_root_volume': True, 'physical_disks': ['/dev/vda', '/dev/vdb']}]}
Jan 05 09:03:43 host-172-16-0-35 ironic-python-agent[832]: 2022-01-05 09:03:43.572 832 INFO root [-] Asynchronous command execute_clean_step completed: {'clean_result': {'logical_disks': [{'raid_level': '1', 'size_gb': 'MAX', 'controller': 'software', 'is_root_volume': True, 'physical_disks': ['/dev/vda', '/dev/vdb']}]}, 'clean_step': {'interface': 'raid', 'step': 'create_configuration', 'abortable': False, 'priority': 0}}
…            
{% endhighlight %}

We can see the exact mdadm commmand that was executed, on which disks (/dev/vda and /dev/vdb) and that it succeeded creating /dev/md0.

#### STEP 7. Set the node back to available state

In order to run the deployment we need the node to be back in available.

{% highlight console %}
(undercloud) [stack@undercloud-osp162 ~]$ openstack overcloud node provide overcloud_compute1
Waiting for messages on queue 'tripleo' with no timeout.
1 node(s) successfully moved to the "available" state.
(undercloud) [stack@undercloud-osp162 ~]$ openstack baremetal node list
+--------------------------------------+-----------------------+---------------+-------------+--------------------+-------------+
| UUID                                 | Name                  | Instance UUID | Power State | Provisioning State | Maintenance |
+--------------------------------------+-----------------------+---------------+-------------+--------------------+-------------+
| 5305b322-b7dd-491d-bb84-dca8a4557ed6 | overcloud_ceph3       | None          | power off   | available          | False       |
| 8f6794d0-25a2-4efd-9be7-60a730c856ca | overcloud_ceph2       | None          | power off   | available          | False       |
| 5825b542-65f4-44cc-9775-ffdebcf3edd7 | overcloud_ceph1       | None          | power off   | available          | False       |
| 9ed03c40-2e19-442c-a132-244b7f04c9f9 | overcloud_networker1  | None          | power off   | available          | False       |
| 47f0fea1-3bd6-41df-9bc2-32f196d0aa49 | overcloud_controller1 | None          | power off   | available          | False       |
| 2bcf72a7-082a-4109-bba9-2b6cf914d72b | overcloud_compute1    | None          | power off   | available          | False       |
| e112ab32-a9f2-4138-a668-bd9cc83f6364 | overcloud_controller2 | None          | power off   | available          | False       |
| 4564d7f2-0468-4c13-9e82-9105028ca655 | overcloud_controller3 | None          | power off   | available          | False       |
+--------------------------------------+-----------------------+---------------+-------------+--------------------+-------------+
{% endhighlight %}

#### STEP 8. Deploy the overcloud

Here just use your default deployment script as usual.


#### STEP 9. Lets login to the compute1 and check the RAID config we obtained at the end

{% highlight console %}
[root@com-01 ~]# mdadm --detail /dev/md127
/dev/md127:
           Version : 1.2
     Creation Time : Wed Jan 05 10:32:02 2022
        Raid Level : raid1
        Array Size : 62878720 (55.96 GiB 62.87 GB)
     Used Dev Size : 62878720 (59.96 GiB 62.87 GB)
      Raid Devices : 2
     Total Devices : 2
       Persistence : Superblock is persistent

       Update Time : Wed Jan 05 10:32:02 2022
             State : clean
    Active Devices : 2
   Working Devices : 2
    Failed Devices : 0
     Spare Devices : 0

Consistency Policy : resync

              Name : host-172-16-0-104:0
              UUID : 381d02dc:ffc37cee:f72889dc:fe14dda0
            Events : 133

    Number   Major   Minor   RaidDevice State
       0     252        1        0      active sync   /dev/vda1
       1     252       17        1      active sync   /dev/vdb1
{% endhighlight %}

### Appendix 1: Whole-disk image for the overcloud

#### STEP1. Download the latest RHEL 8.4 KVM image from Red Hat Customer Portal

#### STEP2. Create a file with the environment variables that will be needed for the image builder

{% highlight bash %}
export DIB_LOCAL_IMAGE=./rhel-8.4-x86_64-kvm.qcow2
export REG_METHOD=portal
export REG_USER={{ rhn_user }}
export REG_PASSWORD={{ rhn_password }}
export REG_POOL_ID="{{ pool_id }}"
export REG_RELEASE="8.4"
export REG_REPOS="rhel-8-for-x86_64-baseos-eus-rpms rhel-8-for-x86_64-appstream-eus-rpms rhel-8-for-x86_64-highavailability-eus-rpms ansible-2.9-for-rhel-8-x86_64-rpms fast-datapath-for-rhel-8-x86_64-rpms openstack-16.1-for-rhel-8-x86_64-rpms"
export DIB_BLOCK_DEVICE_CONFIG='''
- local_loop:
    name: image0
- partitioning:
    base: image0
    label: mbr
    partitions:
      - name: root
        flags: [ boot,primary ]
        size: 35G
- mkfs:
    name: fs_root
    base: root
    type: xfs
    label: "img-rootfs"
    mount:
        mount_point: /
        fstab:
            options: "defaults,rw,relatime"
            fsck-passno: 1

{% endhighlight %}

#### STEP3. Copy and edit the overcloud-hardened-images-python3-custom.yaml to match the disk size and add mdadm package.

{% highlight console %}
(undercloud) [stack@undercloud-osp162 ~]$ cp /usr/share/openstack-tripleo-common/image-yaml/overcloud-hardened-images-python3.yaml /home/stack/overcloud-hardened-images-python3-custom.yaml
{% endhighlight %}

Edit overcloud-hardened-images-python3-custom.yaml as follows.

{% highlight bash %}
(undercloud) [stack@undercloud-osp162 ~]$ cat /home/stack/overcloud-hardened-images-python3-custom.yaml
disk_images:
  -
    imagename: overcloud-hardened-full
    type: qcow2
    elements:
      - openvswitch
      - overcloud-agent
      - overcloud-base
      - overcloud-controller
      - overcloud-compute
      - overcloud-ceph-storage
      - puppet-modules
      - stable-interface-names
      - bootloader
      - element-manifest
      - dynamic-login
      - iptables
      - enable-packages-install
      - override-pip-and-virtualenv
      - dracut-regenerate
      - remove-machine-id
      - remove-resolvconf
      - modprobe
      - overcloud-secure
      - openssh
      - disable-nouveau
    packages:
      - python3-psutil
      - python3-debtcollector
      - sos
      - device-mapper-multipath
      - openstack-heat-agents
      - os-net-config
      - jq
      - mdadm
    options:
      - "--min-tmpfs=7"
    environment:
      DIB_PYTHON_VERSION: '3'
      DIB_MODPROBE_BLACKLIST: 'usb-storage cramfs freevxfs jffs2 hfs hfsplus squashfs udf vfat bluetooth'
      DIB_BOOTLOADER_DEFAULT_CMDLINE: 'nofb nomodeset vga=normal console=tty0 console=ttyS0,115200 audit=1 nousb'
      DIB_IMAGE_SIZE: '35'
      COMPRESS_IMAGE: '1'
{% endhighlight %}


#### STEP 4. Run the image build

{% highlight console %}
(undercloud) [stack@undercloud-osp162 ~]$ openstack overcloud image build --image-name overcloud-hardened-full --config-file /home/stack/overcloud-hardened-images-python3-custom.yaml --config-file /usr/share/openstack-tripleo-common/image-yaml/overcloud-hardened-images-rhel8.yaml
Running ['disk-image-create', '-a', 'amd64', '-o', './overcloud-hardened-full', '-t', 'qcow2', '-p', 'python3-psutil,python3-debtcollector,sos,device-mapper-multipath,openstack-heat-agents,os-net-config,jq,mdadm', '--min-tmpfs=7', 'rhel', 'openvswitch', 'overcloud-agent', 'overcloud-base', 'overcloud-controller', 'overcloud-compute', 'overcloud-ceph-storage', 'puppet-modules', 'stable-interface-names', 'bootloader', 'e
lement-manifest', 'dynamic-login', 'iptables', 'enable-packages-install', 'override-pip-and-virtualenv', 'dracut-regenerate', 'remove-machine-id', 'remove-resolvconf', 'modprobe', 'overcloud-secure', 'openssh', 'disable-nouveau', 'interface-names']
Logging output to ./overcloud-hardened-full.log
2021-12-22 15:03:19.725 | diskimage-builder version 3.9.0
2021-12-22 15:03:19.727 | Building elements: base rhel openvswitch overcloud-agent overcloud-base overcloud-controller overcloud-compute overcloud-ceph-storage puppet-modules stable-interface-names bootloader el
ement-manifest dynamic-login iptables enable-packages-install override-pip-and-virtualenv dracut-regenerate remove-machine-id remove-resolvconf modprobe overcloud-secure openssh disable-nouveau interface-names
2021-12-22 15:03:21.029 | Expanded element dependencies to: puppet element-manifest overcloud-compute remove-resolvconf enable-packages-install openvswitch pip-manifest rhel-common source-repositories yum overcl
oud-openstack-clients dracut-regenerate block-device-mbr base rhel install-static install-bin disable-nouveau stable-interface-names overcloud-secure interface-names overcloud-base sysprep iptables rpm-distro ov
ercloud-ceph-storage svc-map cache-url remove-machine-id select-boot-kernel-initrd overcloud-controller pkg-map redhat-common manifests runtime-ssh-host-keys bootloader dib-init-system openssh-server os-svc-inst
all openssh install-types modprobe puppet-modules overcloud-agent package-installs override-pip-and-virtualenv dynamic-login overcloud-opstools
2021-12-22 15:03:21.708 | Building in /home/stack/dib_build.BX2OgKis
...
{% endhighlight %}

#### STEP 5. Replace the previous overcloud image with the new one

{% highlight bash %}
(undercloud) [stack@undercloud-osp162 ~]$ mv overcloud-hardened-full.qcow2 ~/images/overcloud-full.qcow2
(undercloud) [stack@undercloud-osp162 ~]$ openstack image delete overcloud-full
(undercloud) [stack@undercloud-osp162 ~]$ openstack image delete overcloud-full-initrd
(undercloud) [stack@undercloud-osp162 ~]$ openstack image delete overcloud-full-vmlinuz
(undercloud) [stack@undercloud-osp162 ~]$ openstack overcloud image upload --image-path /home/stack/images --whole-disk
Image "overcloud-full" was uploaded.
+--------------------------------------+----------------+-------------+------------+--------+
|                  ID                  |      Name      | Disk Format |    Size    | Status |
+--------------------------------------+----------------+-------------+------------+--------+
| 265b73e5-9be6-41a1-8afc-ba3ced6530ac | overcloud-full |    qcow2    | 1306465280 | active |
+--------------------------------------+----------------+-------------+------------+--------+
Image file "/var/lib/ironic/httpboot/agent.kernel" is up-to-date, skipping.
Image file "/var/lib/ironic/httpboot/agent.ramdisk" is up-to-date, skipping.
{% endhighlight %}


### References

[^1]: <https://docs.openstack.org/ironic-inspector/latest/user/workflow.html>
[^2]: <https://docs.openstack.org/ironic-python-agent/latest/admin/how_it_works.html>
[^3]: <https://docs.openstack.org/ironic/latest/admin/raid.html>
