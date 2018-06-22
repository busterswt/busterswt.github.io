---
title: "Integrating Tungsten Fabric (OpenContrail) with OpenStack-Ansible"
layout: post
date: 2018-06-22
image: /assets/images/2018-06-22-contrail-osa/tungsten-fabric.png
headerImage: true
tag:
- blog
- openstack
- openstack-ansible
- contrail
- opencontrail
- tungstenfabric
blog: true
author: jamesdenton
description: Integrating Tungsten Fabric (OpenContrail) with OpenStack-Ansible

---

Tungsten Fabric (formerly OpenContrail) is a "multicloud, multistack" SDN solution sponsored by the Linux Foundation - [https://tungsten.io](https://tungsten.io). 

In a nutshell, Tungsten Fabric, and Contrail, the commercial product based on TF, can replace and augment many of the networking components of a standard OpenStack cloud and provides features such as: 

- Distributed virtual routing
- DHCP and Metadata services
- Policy-based access control
- Compatibility with security groups
- ... and more

The forwarding plane supports MPLS over GRE, VXLAN, L2/L3 unicast and L3 multicast for interconnection between virtual and physical networks. A Tungsten Fabric architecture overview can be found at [http://www.opencontrail.org/opencontrail-architecture-documentation/](http://www.opencontrail.org/opencontrail-architecture-documentation/).

<!--more-->
I've recently undertaken the challenge of integrating Tungsten Fabric into OpenStack-Ansible to ease the deployment of TF and associated OpenStack bits in a production-grade OpenStack cloud. This blog will cover, at a high-level, some patches to the master (Rocky) branch of OpenStack-Ansible as well as some tweaks to the contrail-ansible-deployer playbooks provided by Juniper and the TF community for deploying Tungsten Fabric. The process defined here is by no means meant as the final process, and is probably considered a bit clunky and less than ideal. But, it's a start.

I will use the terms Tungsten Fabric, OpenContrail, and Contrail interchangably throughout this blog. My apologies in advance!

# Requirements
A few weeks ago I deployed a standalone 3-node OpenContrail setup with an OpenStack-Ansible all-in-one node based on Queens. After figuring out some of the tweaks necessary to get things to a semi-working state, I decided to try my hand and deploying an AIO node that contained a single instance of Tungsten Fabric services as well as base OpenStack services.

The following minimum specs are recommended:

- System: Ubuntu VM
- OS: 16.04.4 LTS
- RAM: 48GB
- Disk: 300GB
- NIC: Single

A baremetal node may provide less complications, as I will point out later, but a virtual machine on ESXi or some other hypervisor should do the trick.


# Getting started w/ OpenStack-Ansible
To get started, clone the OpenStack-Ansible repository. At time of writing, the `master` branch was tied to Rocky, the 18th release of OpenStack.

```
# git clone https://git.openstack.org/openstack/openstack-ansible /opt/openstack-ansible
# cd /opt/openstack-ansible
# git checkout master
# export ANSIBLE_ROLE_FETCH_MODE=git-clone
```

Next, run the bootstrapping scripts:

```
# scripts/bootstrap-ansible.sh
# scripts/bootstrap-aio.sh
```

The bootstrapping scripts will download the playbooks to deploy OpenStack, and will also prepare the networking on the machine to meet the OpenStack-Ansible architecture.

## Role modifications
Making changes to an OpenStack cloud deployed with OpenStack-Ansible often means making changes to Ansible roles that make up the deployment. This includes changes to tasks, templates, variables, and more.

The roles I needed to modify include:

- os_neutron
- os_nova

Whether all of these role changes were necessary remains to be seen, but they're here for your viewing pleasure.

### os_neutron

Some new files include:
{% raw %}
```
root@aio1:/etc/ansible/roles/os_neutron# git diff --staged
diff --git a/tasks/providers/opencontrail_config.yml b/tasks/providers/opencontrail_config.yml
new file mode 100644
index 0000000..8f5fc7d
--- /dev/null
+++ b/tasks/providers/opencontrail_config.yml
@@ -0,0 +1,99 @@
+---
+# Copyright 2018, Rackspace Hosting, Inc.
+#
+# Licensed under the Apache License, Version 2.0 (the "License");
+# you may not use this file except in compliance with the License.
+# You may obtain a copy of the License at
+#
+#     http://www.apache.org/licenses/LICENSE-2.0
+#
+# Unless required by applicable law or agreed to in writing, software
+# distributed under the License is distributed on an "AS IS" BASIS,
+# WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
+# See the License for the specific language governing permissions and
+# limitations under the License.
+
+- name: Set the packages to install
+  set_fact:
+    neutron_optional_combined_pip_packages: |-
+      {% set packages = neutron_optional_opencontrail_pip_packages %}
+      {{ packages }}
+
+- name: Install OpenContrail pip packages
+  pip:
+    name: "{{ neutron_optional_combined_pip_packages }}"
+    state: "{{ neutron_pip_package_state }}"
+    virtualenv: "{{ neutron_bin | dirname }}"
+    virtualenv_site_packages: "no"
+    extra_args: >-
+      {{ neutron_developer_mode | ternary(pip_install_developer_constraints | default('--constraint /opt/developer-pip-constraints.txt'), '') }}
+      {{ (pip_install_upper_constraints is defined) | ternary('--constraint ' + pip_install_upper_constraints | default(''),'') }}
+      {{ pip_install_options | default('') }}
+  register: install_packages
+  until: install_packages|success
+  retries: 5
+  delay: 2
+  tags:
+    - opencontrail-install
+    - opencontrail-pip-packages
+
+- name: Install git
+  apt:
+    name: git
+    state: present
+  delegate_to: "{{ item }}"
+  with_items:
+    - "{{ groups['neutron_server'] }}"
+  tags:
+    - opencontrail-install
+
+- name: Clone contrail neutron plugin
+  git:
+    repo: "{{ opencontrail_plugin_git_repo }}"
+    version: "{{ opencontrail_plugin_git_install_branch }}"
+    dest: /opt/contrail-neutron-plugin
+    force: yes
+  register: contrail_plugin_git_clone
+  delegate_to: "{{ item }}"
+  with_items:
+    - "{{ groups['neutron_server'] }}"
+  until: contrail_plugin_git_clone|success
+  retries: 5
+  delay: 2
+  tags:
+    - opencontrail-install
+
+# (jamesdenton) Will need to eventually compile and/or extract from Docker container
+# The tasks marked (temp) should be reworked
+
+- name: Download Contrail python libraries (temp)
+  vars:
+  - dlpath: https://github.com/busterswt/contrail-openstack/raw/master
+  get_url:
+    url: "{{ dlpath }}/{{ item }}"
+    dest: /opt
+    mode: 0440
+  with_items:
+    - contrail-openstack-neutron-init.tar
+  tags:
+    - opencontrail-install
+
+- name: Unpack Contrail python libraries (temp)
+  unarchive:
+    remote_src: yes
+    src: /opt/contrail-openstack-neutron-init.tar
+    dest: /openstack/venvs/neutron-{{ neutron_venv_tag }}/lib/python2.7/site-packages
+  when:
+    inventory_hostname == groups['neutron_server'][0]
+  tags:
+    - opencontrail-install
+
+- name: Install contrail neutron plugin into venv
+  command: "/openstack/venvs/neutron-{{ neutron_venv_tag }}/bin/python setup.py install"
+  args:
+    chdir: /opt/contrail-neutron-plugin
+  delegate_to: "{{ item }}"
+  with_items:
+    - "{{ groups['neutron_server'] }}"
+  tags:
+    - opencontrail-install
diff --git a/templates/plugins/opencontrail/ContrailPlugin.ini.j2 b/templates/plugins/opencontrail/ContrailPlugin.ini.j2
new file mode 100644
index 0000000..9d645b0
--- /dev/null
+++ b/templates/plugins/opencontrail/ContrailPlugin.ini.j2
@@ -0,0 +1,23 @@
+# {{ ansible_managed }}
+
+{% if neutron_plugin_type == 'opencontrail' %}
+[APISERVER]
+api_server_ip = {{ opencontrail_api_vip_address }}
+api_server_port = {{ opencontrail_api_vip_port }}
+multi_tenancy = True
+contrail_extensions = ipam:neutron_plugin_contrail.plugins.opencontrail.contrail_plugin_ipam.NeutronPluginContrailIpam,policy:neutron_plugin_contrail.plugins.opencontrail.contrail_plugin_policy.NeutronPluginContrailPolicy,route-table:neutron_plugin_contrail.plugins.opencontrail.contrail_plugin_vpc.NeutronPluginContrailVpc,contrail:None,service-interface:None,vf-binding:None
+
+[COLLECTOR]
+analytics_api_ip = {{ opencontrail_collector_vip_address }}
+analytics_api_port = {{ opencontrail_collector_vip_port }}
+
+[keystone_authtoken]
+auth_host = {{ internal_lb_vip_address }}
+auth_port = {{ keystone_service_port }}
+auth_protocol = {{ keystone_service_proto }}
+admin_user = {{ keystone_admin_user_name }}
+admin_password = {{ keystone_auth_admin_password }}
+admin_tenant_name = {{ keystone_admin_tenant_name }}
+insecure = True
+region_name = {{ keystone_service_region }}
+{% endif %}
```
{% endraw %}
Changes to existing files include:
{% raw %}
```
root@aio1:/etc/ansible/roles/os_neutron# git diff
diff --git a/defaults/main.yml b/defaults/main.yml
index 162e933..7054c96 100644
--- a/defaults/main.yml
+++ b/defaults/main.yml
@@ -63,6 +63,8 @@ networking_bgpvpn_git_repo: https://git.openstack.org/openstack/networking-bgpvp
 networking_bgpvpn_git_install_branch: master
 openstack_ceilometer_git_repo: https://git.openstack.org/openstack/ceilometer
 openstack_ceilometer_git_install_branch: master
+opencontrail_plugin_git_repo: https://github.com/Juniper/contrail-neutron-plugin
+opencontrail_plugin_git_install_branch: master

 # Developer mode
 neutron_developer_mode: false
@@ -164,6 +166,7 @@ neutron_sriov_nic_agent_ini_overrides: {}
 neutron_sriov_nic_agent_init_overrides: {}
 neutron_vpn_agent_init_overrides: {}
 neutron_vpnaas_agent_ini_overrides: {}
+neutron_opencontrail_conf_ini_overrides: {}

 ###
 ### Quotas
@@ -434,3 +437,12 @@ ovs_nsh_support: False

 # Set higher priority to mardim PPA when ovs_nsh_support is True
 ovs_nsh_apt_pinned_packages: [{ package: "*", release: "LP-PPA-mardim-mardim-ppa"}]
+
+###
+### Contrail/OpenContrail/Tungsten Fabric Configuration
+###
+
+opencontrail_api_vip_address: "{{ external_lb_vip_address }}"
+opencontrail_api_vip_port: "8082"
+opencontrail_collector_vip_address: "{{ external_lb_vip_address }}"
+opencontrail_collector_vip_port: "8081"
diff --git a/templates/neutron.conf.j2 b/templates/neutron.conf.j2
index 83d25a7..dd755ca 100644
--- a/templates/neutron.conf.j2
+++ b/templates/neutron.conf.j2
@@ -42,6 +42,10 @@ core_plugin = {{ neutron_plugin_core }}
 {% if neutron_plugin_type.split('.')[0] == 'ml2' %}
 service_plugins = {{ neutron_plugin_loaded_base | join(',') }}
 {% endif %}
+{% if neutron_plugin_type == 'opencontrail' %}
+service_plugins = neutron_plugin_contrail.plugins.opencontrail.loadbalancer.v2.plugin.LoadBalancerPluginV2
+api_extensions_path = /openstack/venvs/neutron-{{ neutron_venv_tag }}/lib/python2.7/site-packages/neutron_plugin_contrail/extensions:/openstack/venvs/neutron-{{ neutron_venv_tag }}/lib/python2.7/site-packages/neutron_lbaas/extensions
+{% endif %}

 # MAC address generation for VIFs
 base_mac = fa:16:3e:00:00:00
@@ -94,8 +98,9 @@ rpc_workers = {{ neutron_rpc_workers }}

 {% set dhcp_agents_max = num_agent if num_agent > 2 else 2 %}
 # DHCP
-{% if neutron_plugin_type == 'ml2.dragonflow' %}
-# In dragonflow, DHCP is fully distributed, and DHCP agents are not used
+{% if neutron_plugin_type == ('ml2.dragonflow' or 'opencontrail') %}
+# In dragonflow and opencontrail, DHCP is fully distributed and DHCP
+# agents are not used
 dhcp_agent_notification = False
 {% else %}
 dhcp_agent_notification = True
diff --git a/vars/main.yml b/vars/main.yml
index cef4ee8..2d1c2a2 100644
--- a/vars/main.yml
+++ b/vars/main.yml
@@ -121,6 +121,10 @@ neutron_plugins:
     plugin_ini: plugins/ml2/ml2_conf.ini
     driver_interface: "openvswitch"
     l3_agent_mode: "legacy"
+  opencontrail:
+    plugin_core: neutron_plugin_contrail.plugins.opencontrail.contrail_plugin.NeutronPluginContrailCoreV2
+    plugin_ini: plugins/opencontrail/ContrailPlugin.ini
+    plugin_conf_ini_overrides: "{{ neutron_opencontrail_conf_ini_overrides }}"

 ###
 ### ML2 Plugin Configuration
diff --git a/vars/source_install.yml b/vars/source_install.yml
index a246a45..24e57ea 100644
--- a/vars/source_install.yml
+++ b/vars/source_install.yml
@@ -96,6 +96,13 @@ neutron_proprietary_nuage_pip_packages:
   - nuage-openstack-neutronclient
   - nuagenetlib

+neutron_optional_opencontrail_pip_packages:
+  - bitarray
+  - bottle
+  - geventhttpclient
+  - psutil>=0.6.0
+  - requests>=1.1.0
+
 neutron_developer_constraints:
   - "git+{{ neutron_git_repo }}@{{ neutron_git_install_branch }}#egg=neutron"
   - "git+{{ neutron_fwaas_git_repo }}@{{ neutron_fwaas_git_install_branch }}#egg=neutron-fwaas"
```
{% endraw %}
### os_nova

```
root@aio1:/etc/ansible/roles/os_nova# git diff
diff --git a/defaults/main.yml b/defaults/main.yml
index 67d92e9..bc44511 100644
--- a/defaults/main.yml
+++ b/defaults/main.yml
@@ -325,6 +325,9 @@ nova_network_services:
   calico:
     use_forwarded_for: True
     metadata_proxy_enabled: False
+  opencontrail:
+    use_forwarded_for: True
+    metadata_proxy_enabled: True

 # Nova quota
 nova_quota_cores: 20
```

### HAproxy changes

In order to provide a single TF API and dashboard endpoint, I made a decision to create a VIP that would balance traffic amongst the TF API and analytics services. Whether this is best practice remains to be seen, but the changes to `group_vars` that facilitate VIP creation are here:
{% raw %}
```
root@aio1:/opt/openstack-ansible# git diff
diff --git a/inventory/group_vars/haproxy/haproxy.yml b/inventory/group_vars/haproxy/haproxy.yml
index b837443..dc53ef4 100644
--- a/inventory/group_vars/haproxy/haproxy.yml
+++ b/inventory/group_vars/haproxy/haproxy.yml
@@ -36,6 +36,7 @@ haproxy_rabbitmq_management_whitelist_networks: "{{ haproxy_whitelist_networks }
 haproxy_repo_git_whitelist_networks: "{{ haproxy_whitelist_networks }}"
 haproxy_repo_cache_whitelist_networks: "{{ haproxy_whitelist_networks }}"
 haproxy_opendaylight_whitelist_networks: "{{ haproxy_whitelist_networks }}"
+haproxy_opencontrail_whitelist_networks: "{{ haproxy_whitelist_networks }}"

 haproxy_default_services:
   - service:
@@ -365,3 +366,23 @@ haproxy_default_services:
       haproxy_backend_httpcheck_options:
         - expect status 405
       haproxy_service_enabled: "{{ (groups['ceph-rgw'] is defined and groups['ceph-rgw'] | length > 0) or (ceph_rgws | length > 0) }}"
+  - service:
+      haproxy_service_name: opencontrail-api
+      haproxy_backend_nodes: "{{ groups['opencontrail-api_hosts'] | default([]) }}"
+      haproxy_bind: "{{ [opencontrail_api_vip_address] }}"
+      haproxy_port: 8082
+      haproxy_balance_type: tcp
+      haproxy_timeout_client: 5000s
+      haproxy_timeout_server: 5000s
+      haproxy_whitelist_networks: "{{ haproxy_opencontrail_whitelist_networks }}"
+      haproxy_service_enabled: "{{ neutron_plugin_type == 'opencontrail' }}"
+  - service:
+      haproxy_service_name: opencontrail-collector
+      haproxy_backend_nodes: "{{ groups['opencontrail-analytics_hosts'] | default([]) }}"
+      haproxy_bind: "{{ [opencontrail_collector_vip_address] }}"
+      haproxy_port: 8081
+      haproxy_balance_type: tcp
+      haproxy_timeout_client: 5000s
+      haproxy_timeout_server: 5000s
+      haproxy_whitelist_networks: "{{ haproxy_opencontrail_whitelist_networks }}"
+      haproxy_service_enabled: "{{ neutron_plugin_type == 'opencontrail' }}"
```
{% endraw %}
Some issues I ran into with this approach on an all-in-one include the ability for HAproxy to bind the VIP to port 8081. Turns out that later on in the process, the Contrail playbooks create a listener on 0.0.0.0:8081 that keeps the VIP from binding on the same port. An alternative here would be to comment out that service, or disable it once HAproxy is deployed. For a multi-node installation where HAproxy is on a different node, it can remain. Load balancing these two services may not work out in the end, but I'll leave it for now.


## Environment changes

The default Neutron `env.d` skeleton will deploy Neutron agent containers that are not necessary as part of the deployment. By overriding the default, we can remove the agent containers and define a few new components:

```
cat <<'EOF' >> /etc/openstack_deploy/env.d/neutron.yml
component_skel:
  neutron_server:
    belongs_to:
      - neutron_all
  opencontrail_vrouter:
    belongs_to:
    - neutron_all
  opencontrail_api:
    belongs_to:
    - neutron_all
  opencontrail_analytics:
    belongs_to:
    - neutron_all

container_skel:
  neutron_agents_container:
    contains: {}
  opencontail-vrouter_container:
    belongs_to:
      - compute_containers
    contains:
      - opencontrail_vrouter
    properties:
      is_metal: true
  opencontrail-api_container:
    belongs_to:
      - opencontrail-api_containers
    contains:
      - opencontrail_api
    properties:
      is_metal: true
  opencontrail-analytics_container:
    belongs_to:
      - opencontrail-analytics_containers
    contains:
      - opencontrail_analytics
    properties:
      is_metal: true
  neutron_server_container:
      belongs_to:
        - network_containers
      contains:
        - neutron_server

physical_skel:
  opencontrail-api_containers:
    belongs_to:
      - all_containers
  opencontrail-api_hosts:
    belongs_to:
      - hosts
  opencontrail-analytics_containers:
    belongs_to:
      - all_containers
  opencontrail-analytics_hosts:
    belongs_to:
      - hosts
EOF
```  
I am not entirely sure at this point if those will all be necessary and if that's the proper approach, but it's there for now.

In the `openstack_user_config.yml` file I defined two new groups with the intention of being able to split out API and Analytics services across hosts. Since this is an AIO, the same IP is defined:

```
opencontrail-api_hosts:
  aio1:
    ip: 172.29.236.100

opencontrail-analytics_hosts:
  aio1:
    ip: 172.29.236.100
```

## Overrides
Changes to the `os_neutron` role resulted in new defaults being added and some overrides being required. In `user_variables.yml` the following was added:

```
neutron_plugin_type: opencontrail
neutron_driver_quota: neutron_plugin_contrail.plugins.opencontrail.quota.driver.QuotaDriver
opencontrail_plugin_git_install_branch: R5.0
```

Some defaults specified in the role include:
{% raw %}
```
opencontrail_api_vip_address: "{{ external_lb_vip_address }}"
opencontrail_api_vip_port: "8082"
opencontrail_collector_vip_address: "{{ external_lb_vip_address }}"
opencontrail_collector_vip_port: "8081"
opencontrail_plugin_git_repo: https://github.com/Juniper/contrail-neutron-plugin
opencontrail_plugin_git_install_branch: master
neutron_opencontrail_conf_ini_overrides: {}
```
{% endraw %}
The final requirements are yet to be determined.

## Running the OpenStack-Ansible playbooks

At this point, most of the changes required to OpenStack-Ansible playbooks have been made and OpenStack can be deployed:

```
# cd /opt/openstack-ansible/playbooks
# openstack-ansible setup-hosts.yml
# openstack-ansible setup-infrastructure.yml
# openstack-ansible setup-openstack.yml
```

While Neutron will be laid down, some additional changes will need to be made and Contrail deployed before networking can be used. Horizon can be used but may not be fully-functional.

Next, we must perform some actions that include cloning the Juniper ansible playbook repo, setting up overrides, and running those playbooks to install Contrail and overlay some Docker containers for Contrail-related services.

# Deploying Tungsten Fabric / OpenContrail / Contrail

At this point of the process, OpenStack has been deployed and is aware of the implemented networking backend but the backend doesn't exist. Juniper provides playbooks that can deploy OpenContrail using Docker containers. Those same playbooks can also deploy an OpenStack release based on Kolla, another OpenStack deployment strategy. We really only need the Contrail-related bits here.

To implement a few changes, I cloned the repo and pushed the required changes needed to deploy on top of an OpenStack-Ansible based cloud.

Clone the repo:

```
# git clone http://github.com/busterswt/contrail-ansible-deployer /opt/openstack-ansible/playbooks/contrail-ansible-deployer
# cd /opt/openstack-ansible/playbooks/contrail-ansible-deployer
# git checkout osa
```

## Overrides

The Juniper playbooks rely on their own inventory and overrides, so performing these next few steps may feel a bit redundant. I have not yet reworked the playbooks to utilize the inventory in place for OpenStack-Ansible.

These overrides can be defined in a new file at `/etc/openstack_deploy/user_opencontrail_vars.yml`:

```
config_file: /etc/openstack_deploy/user_opencontrail_vars.yml
opencontrail_api_vip_address: "{{ external_lb_vip_address }}"
opencontrail_collector_vip_address: "{{ external_lb_vip_address }}"
provider_config:
  bms:
    ssh_pwd:
    ssh_user: root
    ssh_public_key: /root/.ssh/id_rsa.pub
    ssh_private_key: /root/.ssh/id_rsa
    ntpserver: 129.6.15.28
instances:
  aio1:
    provider: bms
    ip: 172.29.236.100
    roles:
      config_database:
      config:
      control:
      analytics_database:
      analytics:
      webui:
      vrouter:
	    VROUTER_GATEWAY: 10.50.0.1
		PHYSICAL_INTERFACE: ens160
global_configuration:
  CONTAINER_REGISTRY: opencontrailnightly
#  CONTAINER_REGISTRY: hub.juniper.net/contrail
#  CONTAINER_REGISTRY_USERNAME: 
#  CONTAINER_REGISTRY_PASSWORD: 
contrail_configuration:
  CLOUD_ORCHESTRATOR: openstack
  CONTRAIL_VERSION: latest
#  CONTRAIL_VERSION: 5.0.0-0.40
# UPGRADE_KERNEL: true
  UPGRADE_KERNEL: false
  KEYSTONE_AUTH_HOST: "{{ internal_lb_vip_address }}"
  KEYSTONE_AUTH_PUBLIC_PORT: 5000
  KEYSTONE_AUTH_PROTO: http
  KEYSTONE_AUTH_URL_VERSION:
  KEYSTONE_AUTH_ADMIN_USER: admin
  KEYSTONE_AUTH_ADMIN_PASSWORD: "{{ keystone_auth_admin_password }}"
  KEYSTONE_AUTH_URL_VERSION: /v3
  CONTROLLER_NODES: 172.29.236.100
  CONTROL_NODES: 172.29.236.100
  ANALYTICSDB_NODES: 172.29.236.100
  WEBUI_NODES: 172.29.236.100
  ANALYTICS_NODES: 172.29.236.100
  CONFIGDB_NODES: 172.29.236.100
  CONFIG_NODES: 172.29.236.100
```

Take special note of the following key/value pairs:

```
VROUTER_GATEWAY: 10.50.0.1
PHYSICAL_INTERFACE: ens160

CONTAINER_REGISTRY: opencontrailnightly
#  CONTAINER_REGISTRY: hub.juniper.net/contrail
#  CONTAINER_REGISTRY_USERNAME: 
#  CONTAINER_REGISTRY_PASSWORD: 
CONTRAIL_VERSION: latest
#  CONTRAIL_VERSION: 5.0.0-0.40

CONTROLLER_NODES: 172.29.236.100
CONTROL_NODES: 172.29.236.100
ANALYTICSDB_NODES: 172.29.236.100
WEBUI_NODES: 172.29.236.100
ANALYTICS_NODES: 172.29.236.100
CONFIGDB_NODES: 172.29.236.100
CONFIG_NODES: 172.29.236.100
```

The first section defines the "physical" interface that will be repurposed/leveraged by the `vhost0` interface required for vRouter. This being an all-in-one node with a single NIC means chances are good the playbooks will automatically determine which interface to use. However, I've gone ahead and called it out here. The IP address of this host is `10.50.0.221` and the gateway is `10.50.0.1`. For a multi-homed host, it may be possible to dedicate an interface to the vRouter along with its own gateway address. I have yet to do a multi-NIC deploy, but look forward to that being the case.

The second section defines the Docker registry from which the containers will be downloaded. The `opencontailnightly` registry contains containers that are build every one to two days, and may or may not work on any given day. If you have access to the GA registry through Juniper, you can define that registry instead and provide access credentials. The only version available on the nightly registry is `latest`, while the Juniper registry may have tagged releases. Use what's appropriate.

The third section defines address for the respective node types, and are used throughout the playbooks to plug values into templates. These are currently required for the playbooks to complete successfully. Note that I've used the 'CONTAINER_NET' address here. The idea is that Contrail and OpenStack would communicate on the existing container network used by LXC. In this AIO, `172.29.236.100` is the IP configured on `br-mgmt`, and happens to also be the internal VIP address. Services deployed in Docker containers will bind ports to this IP for their respective service. How this changes in a multi-node install remains TBD.

## OSA repo changes

OpenStack-Ansible includes a repo server for package management that currently inhibits the installation of certain versions of packages needed to deploy OpenContrail. Here I've disabled the use of the internal repo server by modifying `/root/.pip/pip.conf`:

```
[global]
disable-pip-version-check = true
timeout = 120
#index-url = http://172.29.236.100:8181/simple
#trusted-host =
#        172.29.236.100

[install]
upgrade = true
upgrade-strategy = only-if-needed
pre = true
```

## Running the playbooks

At this point, we should be in a good spot to begin running the Contrail playbooks.

First, the node(s) must be bootstrapped:

```
# cd /opt/openstack-ansible/playbooks/
# openstack-ansible -e orchestrator=openstack contrail-ansible-deployer/playbooks/configure_instances.yml
```

When `UPGRADE_KERNEL` is true, the host may reboot. If this happens, just rerun the playbook again. When the Contrail nodes are not the deploy host, the playbooks have a timer that waits for the host(s) to return.

Next, run the `install_contrail.yml` to deploy OpenContrail:

```
# cd /opt/openstack-ansible/playbooks/
# openstack-ansible -e orchestrator=openstack contrail-ansible-deployer/playbooks/install_contrail.yml
```

## Install additional components

Because the Juniper playbooks expect a Kolla-based deployment, some of the components are not installed on top of our OpenStack-Ansible based cloud. I've taken the liberty of extracting some utilities, modules, and more from Docker containers and wrote some playbooks to lay those down:

```
# cd /opt/openstack-ansible/playbooks/
# openstack-ansible contrail-ansible-deployer/playbooks/install_contrailtools.yml
```

## Reboot

Once OpenContrail and been deployed and the vRouter kernel module compiled and inserted, it may be necessary to reboot the host to clear up some issues encounted inside the vRouter agent container, including:

```
contrail-vrouter-agent: controller/src/vnsw/agent/vrouter/ksync/ksync_memory.cc:107: void KSyncMemory::Mmap(bool): Assertion `0' failed.
/entrypoint.sh: line 304: 28142 Aborted                 (core dumped) $@
```

I found that rebooting the host post-install was enough for the issue to go away.

# Post-Deploy

Now that OpenContrail is (hopefully) installed, use the `contrail-status` command to check the status of the services:

```
root@aio1:/opt/openstack-ansible/playbooks# contrail-status
Pod        Service         Original Name                          State    Status
analytics  api             contrail-analytics-api                 running  Up 21 minutes
analytics  collector       contrail-analytics-collector           running  Up 21 minutes
analytics  nodemgr         contrail-nodemgr                       running  Up 21 minutes
analytics  query-engine    contrail-analytics-query-engine        running  Up 21 minutes
config     api             contrail-controller-config-api         running  Up 24 minutes
config     cassandra       contrail-external-cassandra            running  Up 27 minutes
config     device-manager  contrail-controller-config-devicemgr   running  Up 24 minutes
config     nodemgr         contrail-nodemgr                       running  Up 24 minutes
config     rabbitmq        contrail-external-rabbitmq             running  Up 27 minutes
config     schema          contrail-controller-config-schema      running  Up 24 minutes
config     svc-monitor     contrail-controller-config-svcmonitor  running  Up 24 minutes
config     zookeeper       contrail-external-zookeeper            running  Up 27 minutes
control    control         contrail-controller-control-control    running  Up 23 minutes
control    dns             contrail-controller-control-dns        running  Up 23 minutes
control    named           contrail-controller-control-named      running  Up 23 minutes
control    nodemgr         contrail-nodemgr                       running  Up 23 minutes
database   cassandra       contrail-external-cassandra            running  Up 22 minutes
database   nodemgr         contrail-nodemgr                       running  Up 22 minutes
database   zookeeper       contrail-external-zookeeper            running  Up 22 minutes
vrouter    agent           contrail-vrouter-agent                 running  Up 19 minutes
vrouter    nodemgr         contrail-nodemgr                       running  Up 19 minutes
webui      job             contrail-controller-webui-job          running  Up 23 minutes
webui      web             contrail-controller-webui-web          running  Up 23 minutes

vrouter kernel module is PRESENT
== Contrail control ==
control: active
nodemgr: active
named: active
dns: active

== Contrail database ==
nodemgr: active
zookeeper: active
cassandra: active

== Contrail analytics ==
nodemgr: active
api: initializing (UvePartitions:UVE-Aggregation[Partitions:0] connection down)
collector: initializing (KafkaPub:172.29.236.100:9092 connection down)
query-engine: active

== Contrail webui ==
web: active
job: active

== Contrail vrouter ==
nodemgr: active
agent: active

== Contrail config ==
api: active
zookeeper: active
svc-monitor: active
nodemgr: active
device-manager: active
cassandra: active
rabbitmq: active
schema: active
```

If you see `nodemgr` services in an `Initializing` state with `(NTP state unsynchronized.)`, try restarting NTP on the host. It may be necessary to define multiple NTP servers. I have not yet worked out the Analytics issues, but hope to resolve that soon.

## Issues

I'd like to say that things are completely working at this point, but they're not! Using the `openstack` or `neutron` clients may fail to work, as the `neutron-server` service is likely failing with the following error:

```
Unrecoverable error: please check log for details.: ExtensionsNotFound: Extensions not found: ['route-table']. Related to WARNING neutron.api.extensions [-] Extension file vpcroutetable.py wasn't loaded due to cannot import name attributes.
``` 

In Rocky, one of the deprecated modules that the Contrail plugin replies has finally been removed. Fret not, as it was rolled into another module/file.

In the `neutron_server` container, I modified the `/openstack/venvs/neutron-18.0.0.0b3/lib/python2.7/site-packages/neutron_plugin_contrail/extensions/vpcroutetable.py` file like so:

```
-from neutron.api.v2 import attributes as attr
+from neutron_lib.api import attributes as attr
```

A restart of the `neutron-server` service is needed:

```
# systemctl restart neutron-server
```

# Testing

To validate that OpenContrail has been configured to utilize Keystone authentication, open up the Contrail UI in a browser using the external VIP address and port `8143`:

![](/assets/images/2018-06-22-contrail-osa/contrail-login.png)

Specify the `admin` user and the password defined in the `openrc` file. The domain is `default`. If authentication is successful, the dashboard should appear:

![](/assets/images/2018-06-22-contrail-osa/contrail-dashboard.png)

Switching back to OpenStack, we can perform an `openstack network list` to see the networks:

```
root@aio1-utility-container-ee37a935:~# openstack network list
+--------------------------------------+-------------------------+---------+
| ID                                   | Name                    | Subnets |
+--------------------------------------+-------------------------+---------+
| 723e67c1-8ccd-43ba-a6f4-8b2399c1b8d2 | __link_local__          |         |
| 5a2947ce-0030-4d2a-a06a-76b0d6934102 | ip-fabric               |         |
| 5f4b0153-8146-4e9c-91d4-c60364ece6bc | default-virtual-network |         |
+--------------------------------------+-------------------------+---------+
```

Those networks were all created by the Contrail plugin/driver and should not be removed.

Create a test network:

```
root@aio1-utility-container-ee37a935:~# openstack network create test_network_green
+---------------------------+--------------------------------------+
| Field                     | Value                                |
+---------------------------+--------------------------------------+
| admin_state_up            | UP                                   |
| availability_zone_hints   | None                                 |
| availability_zones        | None                                 |
| created_at                | None                                 |
| description               | None                                 |
| dns_domain                | None                                 |
| id                        | d9e0507f-5ef4-4b62-bf69-176340095053 |
| ipv4_address_scope        | None                                 |
| ipv6_address_scope        | None                                 |
| is_default                | None                                 |
| is_vlan_transparent       | None                                 |
| mtu                       | None                                 |
| name                      | test_network_green                   |
| port_security_enabled     | True                                 |
| project_id                | e565909917a5463b867c5a7594a7612f     |
| provider:network_type     | None                                 |
| provider:physical_network | None                                 |
| provider:segmentation_id  | None                                 |
| qos_policy_id             | None                                 |
| revision_number           | None                                 |
| router:external           | Internal                             |
| segments                  | None                                 |
| shared                    | False                                |
| status                    | ACTIVE                               |
| subnets                   |                                      |
| tags                      |                                      |
| updated_at                | None                                 |
+---------------------------+--------------------------------------+
```

Notice the `provider` attributes are not specified. These are no longer important to us. Other attributes may not be supported by the Contrail plugin.

Create the subnet:

```
root@aio1-utility-container-ee37a935:~# openstack subnet create --subnet-range 172.23.0.0/24 --network test_network_green test_subnet_green
+-------------------+--------------------------------------+
| Field             | Value                                |
+-------------------+--------------------------------------+
| allocation_pools  | 172.23.0.2-172.23.0.254              |
| cidr              | 172.23.0.0/24                        |
| created_at        | None                                 |
| description       | None                                 |
| dns_nameservers   |                                      |
| enable_dhcp       | True                                 |
| gateway_ip        | 172.23.0.1                           |
| host_routes       |                                      |
| id                | cc2d2f56-5c87-49fb-afd5-14e32feccd6a |
| ip_version        | 4                                    |
| ipv6_address_mode | None                                 |
| ipv6_ra_mode      | None                                 |
| name              | test_subnet_green                    |
| network_id        | d9e0507f-5ef4-4b62-bf69-176340095053 |
| project_id        | e565909917a5463b867c5a7594a7612f     |
| revision_number   | None                                 |
| segment_id        | None                                 |
| service_types     | None                                 |
| subnetpool_id     | None                                 |
| tags              |                                      |
| updated_at        | None                                 |
+-------------------+--------------------------------------+
```

IPv6 should be supported, but I ran into issues trying to create an IPv6 subnet. At this point, our network is ready for VMs. For good measure, I created a security group that could be applied to our instance that would allow SSH:

```
root@aio1-utility-container-ee37a935:~# openstack security group create allow_ssh
+-----------------+--------------------------------------+
| Field           | Value                                |
+-----------------+--------------------------------------+
| created_at      | None                                 |
| description     | allow_ssh                            |
| id              | 39a9e241-27c3-452a-b37a-80b6dcbbf783 |
| name            | allow_ssh                            |
| project_id      | e565909917a5463b867c5a7594a7612f     |
| revision_number | None                                 |
| rules           |                                      |
| tags            | []                                   |
| updated_at      | None                                 |
+-----------------+--------------------------------------+

root@aio1-utility-container-ee37a935:~# openstack security group rule create --dst-port 22 allow_ssh

+-------------------+--------------------------------------+
| Field             | Value                                |
+-------------------+--------------------------------------+
| created_at        | None                                 |
| description       | None                                 |
| direction         | ingress                              |
| ether_type        | IPv4                                 |
| id                | b8393e4d-1d9d-47e9-877e-86374f38dca1 |
| name              | None                                 |
| port_range_max    | 22                                   |
| port_range_min    | 22                                   |
| project_id        | e565909917a5463b867c5a7594a7612f     |
| protocol          | tcp                                  |
| remote_group_id   | None                                 |
| remote_ip_prefix  | 0.0.0.0/0                            |
| revision_number   | None                                 |
| security_group_id | 39a9e241-27c3-452a-b37a-80b6dcbbf783 |
| updated_at        | None                                 |
+-------------------+--------------------------------------+
```

I then booted the instance using a tiny flavor and a CirrOS image:

```
root@aio1-utility-container-ee37a935:~# openstack server create --image cirros --flavor test_flavor --nic net-id=test_network_green --security-group allow_ssh test1
+-------------------------------------+----------------------------------------------------+
| Field                               | Value                                              |
+-------------------------------------+----------------------------------------------------+
| OS-DCF:diskConfig                   | MANUAL                                             |
| OS-EXT-AZ:availability_zone         |                                                    |
| OS-EXT-SRV-ATTR:host                | None                                               |
| OS-EXT-SRV-ATTR:hypervisor_hostname | None                                               |
| OS-EXT-SRV-ATTR:instance_name       |                                                    |
| OS-EXT-STS:power_state              | NOSTATE                                            |
| OS-EXT-STS:task_state               | scheduling                                         |
| OS-EXT-STS:vm_state                 | building                                           |
| OS-SRV-USG:launched_at              | None                                               |
| OS-SRV-USG:terminated_at            | None                                               |
| accessIPv4                          |                                                    |
| accessIPv6                          |                                                    |
| addresses                           |                                                    |
| adminPass                           | a8tghwSoTWZP                                       |
| config_drive                        |                                                    |
| created                             | 2018-06-18T14:34:49Z                               |
| flavor                              | test_flavor (5c0600b7-f9fe-46f3-8af5-f8390ee5c6f3) |
| hostId                              |                                                    |
| id                                  | b14d1861-8855-4d17-a2d3-87eb67a3d81c               |
| image                               | cirros (4006fd58-cdc5-4bd8-bc25-ef73be1cd429)      |
| key_name                            | None                                               |
| name                                | test1                                              |
| progress                            | 0                                                  |
| project_id                          | e565909917a5463b867c5a7594a7612f                   |
| properties                          |                                                    |
| security_groups                     | name='39a9e241-27c3-452a-b37a-80b6dcbbf783'        |
| status                              | BUILD                                              |
| updated                             | 2018-06-18T14:34:49Z                               |
| user_id                             | f6aac1aa53294659998aa71838133a1d                   |
| volumes_attached                    |                                                    |
+-------------------------------------+----------------------------------------------------+

root@aio1-utility-container-ee37a935:~# openstack server list
+--------------------------------------+-------+--------+-------------------------------+--------+-------------+
| ID                                   | Name  | Status | Networks                      | Image  | Flavor      |
+--------------------------------------+-------+--------+-------------------------------+--------+-------------+
| b14d1861-8855-4d17-a2d3-87eb67a3d81c | test1 | ACTIVE | test_network_green=172.23.0.3 | cirros | test_flavor |
+--------------------------------------+-------+--------+-------------------------------+--------+-------------+
```

Now, I could attach to the instance's console and attempt outbound connectivity:

![](/assets/images/2018-06-22-contrail-osa/ping-fail.png)

Within the Contrail UI, I was able to enable `snat` on the network to allow the vRouter to snat outbound connections from the VM:

![](/assets/images/2018-06-22-contrail-osa/contrail-snat.png)

A quick test showed ping working:

![](/assets/images/2018-06-22-contrail-osa/ping-snat.png)

Inbound connections to the VM are also possible, but require some additional work within Contrail to advertise the VMs address. I have a Cisco ASA 1001 in my lab that has been configured to peer with the Contrail controller, but will have to save that configuration demonstration for another day.

# Summary

There is still a lot of work to do to not only learn how Tungsten Fabric / OpenContrail / Contrail operates, but to also construct best practices around how it should be deployed within an OpenStack-Ansible based cloud. Some of the components used for the installation are heavily wrapped up in Docker containers and had to be extracted before deploying in the LXC container and/or the host(s). This isn't scalable, but is good enough for now. 

I ran into issues with the `opencontrailnightly` build recently, in that the vRouter was dropping outbound or response traffic from my VM. Resorting to a GA build from Juniper's repo solved the issue, but not everyone may have that access. 

Another problem I encountered was that while pings to/from the VM worked (with the ASR bits in place), SSH connections failed. In fact, any TCP connection failed. The SYNs were seen at the instance, and SYN/ACK was observed going out. However, the SYN/ACK never made it past the vRouter. A packet capture showed that the SYN/ACK had an invalid checksum. Disabling generic IP checksums on the "physical" interface of the host, `ens160` in this case, allowed things to work. This article was super helpful:

[https://kb.juniper.net/InfoCenter/index?page=content&id=KB30500](https://kb.juniper.net/InfoCenter/index?page=content&id=KB30500)

As I get more reps with this I hope to simplify the process and one day get this moved upstream for inclusion into OpenStack-Ansible. Until then, good luck!