---
title: "[OVN] 'Chassis_Private' object has no attribute 'hostname'"
layout: post
date: 2022-03-25
image: /assets/images/2022-03-25-neutron-ovn-private-chassis/dawson-crying-animated.gif
headerImage: true
tag:
- ovn
- neutron
- openstack
category: blog
blog: true
author: jamesdenton
description: "Fixing the 'Chassis_Private' object has no attribute 'hostname' issue in Neutron"
---

On more than one occasion I have turned to this blog to fix issues that reoccur weeks/months/years after the initial post is born, and this post will serve as one of those reference points in the future, I'm sure. In my OpenStack-Ansible Xena lab running OVN, I've twice now come across the following error when performing a `openstack network agent list` command:

```
'Chassis_Private' object has no attribute 'hostname'
```

What does that even mean?!
<!--more-->

What `chassis_private` is referring to is a table in the OVN Southbound database. Not to be confused with the `chassis` table, a row in the `chassis_private` table is used by `ovn-northd` and the owning chassis to store *private* data about that chassis, including:

- uuid
- name
- chassis
- nb_cfg
- nb_cfg_timestamp
- external_ids

The manpage does better service of describing its purpose:

```
These data are stored in this separate table instead of the Chassis
table for performance considerations:

the  rows  in  this table can be conditionally monitored by chassises
so that each chassis only get update notifications for its own row,
to avoid unnecessary chassis  private data update flooding in a large
scale deployment.
```

My environment consists of 3x controller nodes and 3x compute nodes running a variety of services, including OVN, OVN Metadata Agent, Legacy DHCP Agent (for Ironic), and the SR-IOV Agent. The catalyst for this particular post was an error when trying to retrieve a list of those agents:

```
root@lab-infra01:~# openstack network agent list
HttpException: 500: Server Error for url: http://10.20.0.11:9696/v2.0/agents, Request Failed: internal server error while processing your request.
```

A look at the `neutron-server` log revealed the following traceback:

```
Mar 24 19:37:30 lab-infra03 neutron-server[3148184]: 
2022-03-24 19:37:30.625 3148184 ERROR neutron.api.v2.resource [req-82bf64ab-d8d4-4678-abdb-de439c392e71 34f3cf48b24f41c097555c07961f139e 7a8df96a3c6a47118e60e57aa9ecff54 - default default] index failed: No details.: AttributeError: 'Chassis_Private' object has no attribute 'hostname'
2022-03-24 19:37:30.625 3148184 ERROR neutron.api.v2.resource Traceback (most recent call last):
2022-03-24 19:37:30.625 3148184 ERROR neutron.api.v2.resource   File "/openstack/venvs/neutron-24.0.1/lib/python3.8/site-packages/neutron/api/v2/resource.py", line 98, in resource
2022-03-24 19:37:30.625 3148184 ERROR neutron.api.v2.resource     result = method(request=request, **args)
2022-03-24 19:37:30.625 3148184 ERROR neutron.api.v2.resource   File "/openstack/venvs/neutron-24.0.1/lib/python3.8/site-packages/neutron_lib/db/api.py", line 139, in wrapped
2022-03-24 19:37:30.625 3148184 ERROR neutron.api.v2.resource     setattr(e, '_RETRY_EXCEEDED', True)
2022-03-24 19:37:30.625 3148184 ERROR neutron.api.v2.resource   File "/openstack/venvs/neutron-24.0.1/lib/python3.8/site-packages/oslo_utils/excutils.py", line 227, in __exit__
2022-03-24 19:37:30.625 3148184 ERROR neutron.api.v2.resource     self.force_reraise()
2022-03-24 19:37:30.625 3148184 ERROR neutron.api.v2.resource   File "/openstack/venvs/neutron-24.0.1/lib/python3.8/site-packages/oslo_utils/excutils.py", line 200, in force_reraise
2022-03-24 19:37:30.625 3148184 ERROR neutron.api.v2.resource     raise self.value
2022-03-24 19:37:30.625 3148184 ERROR neutron.api.v2.resource   File "/openstack/venvs/neutron-24.0.1/lib/python3.8/site-packages/neutron_lib/db/api.py", line 135, in wrapped
2022-03-24 19:37:30.625 3148184 ERROR neutron.api.v2.resource     return f(*args, **kwargs)
2022-03-24 19:37:30.625 3148184 ERROR neutron.api.v2.resource   File "/openstack/venvs/neutron-24.0.1/lib/python3.8/site-packages/oslo_db/api.py", line 154, in wrapper
2022-03-24 19:37:30.625 3148184 ERROR neutron.api.v2.resource     ectxt.value = e.inner_exc
2022-03-24 19:37:30.625 3148184 ERROR neutron.api.v2.resource   File "/openstack/venvs/neutron-24.0.1/lib/python3.8/site-packages/oslo_utils/excutils.py", line 227, in __exit__
2022-03-24 19:37:30.625 3148184 ERROR neutron.api.v2.resource     self.force_reraise()
2022-03-24 19:37:30.625 3148184 ERROR neutron.api.v2.resource   File "/openstack/venvs/neutron-24.0.1/lib/python3.8/site-packages/oslo_utils/excutils.py", line 200, in force_reraise
2022-03-24 19:37:30.625 3148184 ERROR neutron.api.v2.resource     raise self.value
2022-03-24 19:37:30.625 3148184 ERROR neutron.api.v2.resource   File "/openstack/venvs/neutron-24.0.1/lib/python3.8/site-packages/oslo_db/api.py", line 142, in wrapper
2022-03-24 19:37:30.625 3148184 ERROR neutron.api.v2.resource     return f(*args, **kwargs)
2022-03-24 19:37:30.625 3148184 ERROR neutron.api.v2.resource   File "/openstack/venvs/neutron-24.0.1/lib/python3.8/site-packages/neutron_lib/db/api.py", line 183, in wrapped
2022-03-24 19:37:30.625 3148184 ERROR neutron.api.v2.resource     LOG.debug("Retry wrapper got retriable exception: %s", e)
2022-03-24 19:37:30.625 3148184 ERROR neutron.api.v2.resource   File "/openstack/venvs/neutron-24.0.1/lib/python3.8/site-packages/oslo_utils/excutils.py", line 227, in __exit__
2022-03-24 19:37:30.625 3148184 ERROR neutron.api.v2.resource     self.force_reraise()
2022-03-24 19:37:30.625 3148184 ERROR neutron.api.v2.resource   File "/openstack/venvs/neutron-24.0.1/lib/python3.8/site-packages/oslo_utils/excutils.py", line 200, in force_reraise
2022-03-24 19:37:30.625 3148184 ERROR neutron.api.v2.resource     raise self.value
2022-03-24 19:37:30.625 3148184 ERROR neutron.api.v2.resource   File "/openstack/venvs/neutron-24.0.1/lib/python3.8/site-packages/neutron_lib/db/api.py", line 179, in wrapped
2022-03-24 19:37:30.625 3148184 ERROR neutron.api.v2.resource     return f(*dup_args, **dup_kwargs)
2022-03-24 19:37:30.625 3148184 ERROR neutron.api.v2.resource   File "/openstack/venvs/neutron-24.0.1/lib/python3.8/site-packages/neutron/api/v2/base.py", line 369, in index
2022-03-24 19:37:30.625 3148184 ERROR neutron.api.v2.resource     return self._items(request, True, parent_id)
2022-03-24 19:37:30.625 3148184 ERROR neutron.api.v2.resource   File "/openstack/venvs/neutron-24.0.1/lib/python3.8/site-packages/neutron/api/v2/base.py", line 304, in _items
2022-03-24 19:37:30.625 3148184 ERROR neutron.api.v2.resource     obj_list = obj_getter(request.context, **kwargs)
2022-03-24 19:37:30.625 3148184 ERROR neutron.api.v2.resource   File "/openstack/venvs/neutron-24.0.1/lib/python3.8/site-packages/neutron/plugins/ml2/drivers/ovn/mech_driver/mech_driver.py", line 1165, in fn
2022-03-24 19:37:30.625 3148184 ERROR neutron.api.v2.resource     return op(results, new_method(*args, _driver=self, **kwargs))
2022-03-24 19:37:30.625 3148184 ERROR neutron.api.v2.resource   File "/openstack/venvs/neutron-24.0.1/lib/python3.8/site-packages/neutron/plugins/ml2/drivers/ovn/mech_driver/mech_driver.py", line 1229, in get_agents
2022-03-24 19:37:30.625 3148184 ERROR neutron.api.v2.resource     agent_dict = agent.as_dict()
2022-03-24 19:37:30.625 3148184 ERROR neutron.api.v2.resource   File "/openstack/venvs/neutron-24.0.1/lib/python3.8/site-packages/neutron/plugins/ml2/drivers/ovn/agent/neutron_agent.py", line 59, in as_dict
2022-03-24 19:37:30.625 3148184 ERROR neutron.api.v2.resource     'host': self.chassis.hostname,
2022-03-24 19:37:30.625 3148184 ERROR neutron.api.v2.resource AttributeError: 'Chassis_Private' object has no attribute 'hostname'
2022-03-24 19:37:30.625 3148184 ERROR neutron.api.v2.resource
```

Most importantly:

```
AttributeError: 'Chassis_Private' object has no attribute 'hostname'
```

## A Look at OVN

To understand what the 'Chassis_Private' object was and what its structure was expected to be, I took a visit to the Neutron source code; specifically `neutron/plugins/ml2/drivers/ovn/agent/neutron_agent.py` line 59:

[https://github.com/openstack/neutron/blob/stable/xena/neutron/plugins/ml2/drivers/ovn/agent/neutron_agent.py](https://github.com/openstack/neutron/blob/stable/xena/neutron/plugins/ml2/drivers/ovn/agent/neutron_agent.py)

```
    def as_dict(self):
        return {
            'binary': self.binary,
            'host': self.chassis.hostname,
            'heartbeat_timestamp': timeutils.utcnow(),
            'availability_zone': ', '.join(
                ovn_utils.get_chassis_availability_zones(self.chassis)),
            'topic': 'n/a',
            'description': self.description,
            'configurations': {
                'chassis_name': self.chassis.name,
                'bridge-mappings':
                    self.chassis.external_ids.get('ovn-bridge-mappings', '')},
            'start_flag': True,
            'agent_type': self.agent_type,
            'id': self.agent_id,
            'alive': self.alive,
            'admin_state_up': True}
```

In the above snippet, we can see in `as_dict` that `host` references `self.chassis.hostname`, and `chassis` itself is defined here:

```
@property
def chassis(self):
        return self.chassis_from_private(self.chassis_private)
```

If we take a look at `chassis_from_private`, we get this:

```
@staticmethod
    def chassis_from_private(chassis_private):
        try:
            return chassis_private.chassis[0]
        except (AttributeError, IndexError):
            # No Chassis_Private support, just use Chassis
            return chassis_private
```

I don't proclaim to be a Python expert, or even a developer for that matter, but in following along I can see that it's returning the 1st element ([0]) of the list `chassis` for this `chassis_private` object.

Using some OVN tools, I was able to list both the `chassis_private` and `chassis` tables from the Southbound DB:

#### chassis table

```
root@lab-infra02:~# ovn-sbctl list chassis
_uuid               : 6c90b020-be8e-4b7c-9aa8-0f4a9f826e6d
encaps              : [4c1cae4d-36a4-4541-af8c-fc02758fab4e, ac4dd143-10db-48c3-b4dd-8f42d0d6efd0]
external_ids        : {datapath-type=system, iface-types="bareudp,erspan,geneve,gre,gtpu,internal,ip6erspan,ip6gre,lisp,patch,stt,system,tap,vxlan", is-interconn="false", ovn-bridge-mappings="physnet1:br-rpn,vlan:br-ex", ovn-chassis-mac-mappings="", ovn-cms-options=enable-chassis-as-gw, ovn-enable-lflow-cache="true", ovn-limit-lflow-cache="", ovn-memlimit-lflow-cache-kb="", ovn-monitor-all="false", ovn-trim-limit-lflow-cache="", ovn-trim-wmark-perc-lflow-cache="", port-up-notif="true"}
hostname            : lab-compute01
name                : "0c9b25a6-3760-4b57-ba71-49e7091730bb"
nb_cfg              : 0
other_config        : {datapath-type=system, iface-types="bareudp,erspan,geneve,gre,gtpu,internal,ip6erspan,ip6gre,lisp,patch,stt,system,tap,vxlan", is-interconn="false", ovn-bridge-mappings="physnet1:br-rpn,vlan:br-ex", ovn-chassis-mac-mappings="", ovn-cms-options=enable-chassis-as-gw, ovn-enable-lflow-cache="true", ovn-limit-lflow-cache="", ovn-memlimit-lflow-cache-kb="", ovn-monitor-all="false", ovn-trim-limit-lflow-cache="", ovn-trim-wmark-perc-lflow-cache="", port-up-notif="true"}
transport_zones     : []
vtep_logical_switches: []

_uuid               : 478e3679-f4af-4a2d-a986-85323c840620
encaps              : [1e5060c3-a6ce-41bd-b54a-2ba3907f7092, 3177060c-bcdc-4c02-bf31-ff359c666538]
external_ids        : {datapath-type=system, iface-types="bareudp,erspan,geneve,gre,gtpu,internal,ip6erspan,ip6gre,lisp,patch,stt,system,tap,vxlan", is-interconn="false", ovn-bridge-mappings="vlan:br-ex", ovn-chassis-mac-mappings="", ovn-cms-options=enable-chassis-as-gw, ovn-enable-lflow-cache="true", ovn-limit-lflow-cache="", ovn-memlimit-lflow-cache-kb="", ovn-monitor-all="false", ovn-trim-limit-lflow-cache="", ovn-trim-wmark-perc-lflow-cache="", port-up-notif="true"}
hostname            : lab-compute03
name                : "1f318a3c-f607-4272-814c-b0c4d813daa5"
nb_cfg              : 0
other_config        : {datapath-type=system, iface-types="bareudp,erspan,geneve,gre,gtpu,internal,ip6erspan,ip6gre,lisp,patch,stt,system,tap,vxlan", is-interconn="false", ovn-bridge-mappings="vlan:br-ex", ovn-chassis-mac-mappings="", ovn-cms-options=enable-chassis-as-gw, ovn-enable-lflow-cache="true", ovn-limit-lflow-cache="", ovn-memlimit-lflow-cache-kb="", ovn-monitor-all="false", ovn-trim-limit-lflow-cache="", ovn-trim-wmark-perc-lflow-cache="", port-up-notif="true"}
transport_zones     : []
vtep_logical_switches: []

_uuid               : eed64f71-9cd1-4e1a-a891-e8bbb9049c41
encaps              : [3932bdff-e3a2-425b-9bf5-8d05fffbd171, a6f63e7b-a59a-41a3-9fdf-a2b6fae892cd]
external_ids        : {datapath-type=system, iface-types="bareudp,erspan,geneve,gre,gtpu,internal,ip6erspan,ip6gre,lisp,patch,stt,system,tap,vxlan", is-interconn="false", ovn-bridge-mappings="physnet2:br-rpn,vlan:br-ex", ovn-chassis-mac-mappings="", ovn-cms-options=enable-chassis-as-gw, ovn-enable-lflow-cache="true", ovn-limit-lflow-cache="", ovn-memlimit-lflow-cache-kb="", ovn-monitor-all="false", ovn-trim-limit-lflow-cache="", ovn-trim-wmark-perc-lflow-cache="", port-up-notif="true"}
hostname            : lab-compute02
name                : "6c2a75b1-482a-40e3-91f8-3e449986f5b6"
nb_cfg              : 174
other_config        : {datapath-type=system, iface-types="bareudp,erspan,geneve,gre,gtpu,internal,ip6erspan,ip6gre,lisp,patch,stt,system,tap,vxlan", is-interconn="false", ovn-bridge-mappings="physnet2:br-rpn,vlan:br-ex", ovn-chassis-mac-mappings="", ovn-cms-options=enable-chassis-as-gw, ovn-enable-lflow-cache="true", ovn-limit-lflow-cache="", ovn-memlimit-lflow-cache-kb="", ovn-monitor-all="false", ovn-trim-limit-lflow-cache="", ovn-trim-wmark-perc-lflow-cache="", port-up-notif="true"}
transport_zones     : []
vtep_logical_switches: []

_uuid               : 8a07dfc5-1e52-49aa-aa97-ec0515334fc6
encaps              : [63a84cd2-cb93-485e-aaa4-e6701dbb9a7d, a67c8b1c-da4e-4178-b0eb-58315983ca68]
external_ids        : {datapath-type=system, iface-types="bareudp,erspan,geneve,gre,gtpu,internal,ip6erspan,ip6gre,lisp,patch,stt,system,tap,vxlan", is-interconn="false", ovn-bridge-mappings="vlan:br-ex", ovn-chassis-mac-mappings="", ovn-cms-options=enable-chassis-as-gw, ovn-enable-lflow-cache="true", ovn-limit-lflow-cache="", ovn-memlimit-lflow-cache-kb="", ovn-monitor-all="false", ovn-trim-limit-lflow-cache="", ovn-trim-wmark-perc-lflow-cache="", port-up-notif="true"}
hostname            : lab-infra01
name                : "900595a5-a02a-4566-b6dc-0c1e0e2cb392"
nb_cfg              : 0
other_config        : {datapath-type=system, iface-types="bareudp,erspan,geneve,gre,gtpu,internal,ip6erspan,ip6gre,lisp,patch,stt,system,tap,vxlan", is-interconn="false", ovn-bridge-mappings="vlan:br-ex", ovn-chassis-mac-mappings="", ovn-cms-options=enable-chassis-as-gw, ovn-enable-lflow-cache="true", ovn-limit-lflow-cache="", ovn-memlimit-lflow-cache-kb="", ovn-monitor-all="false", ovn-trim-limit-lflow-cache="", ovn-trim-wmark-perc-lflow-cache="", port-up-notif="true"}
transport_zones     : []
vtep_logical_switches: []

_uuid               : 8f4829ad-746d-4125-8561-363adbbc4dce
encaps              : [ce7a5ab6-534a-4667-a7c0-5f112b0f4507, fcaf8226-fa90-4848-8c1d-d3c975276e05]
external_ids        : {datapath-type=system, iface-types="bareudp,erspan,geneve,gre,gtpu,internal,ip6erspan,ip6gre,lisp,patch,stt,system,tap,vxlan", is-interconn="false", "neutron:ovn-metadata-id"="344341e0-8e69-5e00-979c-d59fee1b9b27", "neutron:ovn-metadata-sb-cfg"="173", ovn-bridge-mappings="vlan:br-ex", ovn-chassis-mac-mappings="", ovn-cms-options=enable-chassis-as-gw, ovn-enable-lflow-cache="true", ovn-limit-lflow-cache="", ovn-memlimit-lflow-cache-kb="", ovn-monitor-all="false", ovn-trim-limit-lflow-cache="", ovn-trim-wmark-perc-lflow-cache="", port-up-notif="true"}
hostname            : lab-infra03
name                : "d50d391d-910f-40d6-8aa7-24fbfda018ff"
nb_cfg              : 171
other_config        : {datapath-type=system, iface-types="bareudp,erspan,geneve,gre,gtpu,internal,ip6erspan,ip6gre,lisp,patch,stt,system,tap,vxlan", is-interconn="false", ovn-bridge-mappings="vlan:br-ex", ovn-chassis-mac-mappings="", ovn-cms-options=enable-chassis-as-gw, ovn-enable-lflow-cache="true", ovn-limit-lflow-cache="", ovn-memlimit-lflow-cache-kb="", ovn-monitor-all="false", ovn-trim-limit-lflow-cache="", ovn-trim-wmark-perc-lflow-cache="", port-up-notif="true"}
transport_zones     : []
vtep_logical_switches: []

_uuid               : ada8b169-bd60-490b-8520-7e621cbbb84e
encaps              : [072fa878-8849-4b06-acfd-4e889ff308b0, 8b6e5992-a4bf-46e3-b1c2-d5494765ca62]
external_ids        : {datapath-type=system, iface-types="bareudp,erspan,geneve,gre,gtpu,internal,ip6erspan,ip6gre,lisp,patch,stt,system,tap,vxlan", is-interconn="false", "neutron:ovn-metadata-id"="83641d9c-6244-564c-b67c-d5b3298adc85", "neutron:ovn-metadata-sb-cfg"="574", ovn-bridge-mappings="vlan:br-ex", ovn-chassis-mac-mappings="", ovn-cms-options=enable-chassis-as-gw, ovn-enable-lflow-cache="true", ovn-limit-lflow-cache="", ovn-memlimit-lflow-cache-kb="", ovn-monitor-all="false", ovn-trim-limit-lflow-cache="", ovn-trim-wmark-perc-lflow-cache="", port-up-notif="true"}
hostname            : lab-infra02
name                : "30757b96-cb1b-4512-bfdd-df6df50f2f4c"
nb_cfg              : 171
other_config        : {datapath-type=system, iface-types="bareudp,erspan,geneve,gre,gtpu,internal,ip6erspan,ip6gre,lisp,patch,stt,system,tap,vxlan", is-interconn="false", ovn-bridge-mappings="vlan:br-ex", ovn-chassis-mac-mappings="", ovn-cms-options=enable-chassis-as-gw, ovn-enable-lflow-cache="true", ovn-limit-lflow-cache="", ovn-memlimit-lflow-cache-kb="", ovn-monitor-all="false", ovn-trim-limit-lflow-cache="", ovn-trim-wmark-perc-lflow-cache="", port-up-notif="true"}
transport_zones     : []
vtep_logical_switches: []
```

There I see 6 chassis defined in the Southbound DB, which is to be expected.

#### chassis_private table

```
root@lab-infra02:~# ovn-sbctl list chassis_private
_uuid               : 38940279-b953-4ec8-9069-9fc2e7b7fe3d
chassis             : eed64f71-9cd1-4e1a-a891-e8bbb9049c41
external_ids        : {"neutron:ovn-metadata-id"="6645f143-2dc0-5f03-b7bb-681bc3e8b969", "neutron:ovn-metadata-sb-cfg"="662"}
name                : "6c2a75b1-482a-40e3-91f8-3e449986f5b6"
nb_cfg              : 662
nb_cfg_timestamp    : 1648150649984

_uuid               : 521bfe75-5a10-4928-8158-b15df2cb0c5d
chassis             : ada8b169-bd60-490b-8520-7e621cbbb84e
external_ids        : {"neutron:ovn-metadata-id"="83641d9c-6244-564c-b67c-d5b3298adc85", "neutron:ovn-metadata-sb-cfg"="662"}
name                : "30757b96-cb1b-4512-bfdd-df6df50f2f4c"
nb_cfg              : 662
nb_cfg_timestamp    : 1648150649989

_uuid               : 4c614b3d-1e26-40d7-8d92-00d1a1e77243
chassis             : 8f4829ad-746d-4125-8561-363adbbc4dce
external_ids        : {"neutron:ovn-metadata-id"="344341e0-8e69-5e00-979c-d59fee1b9b27", "neutron:ovn-metadata-sb-cfg"="662"}
name                : "d50d391d-910f-40d6-8aa7-24fbfda018ff"
nb_cfg              : 662
nb_cfg_timestamp    : 1648150649988

_uuid               : 73b1096a-b38f-4e6a-960c-4a99e93735d6
chassis             : 6c90b020-be8e-4b7c-9aa8-0f4a9f826e6d
external_ids        : {"neutron:ovn-metadata-id"="2864488c-c9a8-5cf1-b1c0-184c295493b6", "neutron:ovn-metadata-sb-cfg"="662"}
name                : "0c9b25a6-3760-4b57-ba71-49e7091730bb"
nb_cfg              : 662
nb_cfg_timestamp    : 1648150649984

_uuid               : 3b580d16-2896-489f-8f1a-10d2cd13e1ae
chassis             : []
external_ids        : {"neutron:ovn-metadata-id"="bdc50d9c-42b1-5f20-8737-baba108b2f67", "neutron:ovn-metadata-sb-cfg"="425"}
name                : "5236f154-4a73-44ab-a588-b602a0b56bd5"
nb_cfg              : 425
nb_cfg_timestamp    : 1643778233179

_uuid               : a65846ba-67dc-49eb-9558-17ca0db09e0f
chassis             : 478e3679-f4af-4a2d-a986-85323c840620
external_ids        : {"neutron:ovn-metadata-id"="4d9e06dc-69c0-5ea7-8a6d-e750d11ebb9f", "neutron:ovn-metadata-sb-cfg"="662"}
name                : "1f318a3c-f607-4272-814c-b0c4d813daa5"
nb_cfg              : 662
nb_cfg_timestamp    : 1648150649985

_uuid               : cb281eea-a02c-44c7-81e9-19aab7637c12
chassis             : 8a07dfc5-1e52-49aa-aa97-ec0515334fc6
external_ids        : {"neutron:ovn-metadata-id"="64b68ff2-b068-5e64-a1cd-9c95afadd0b7", "neutron:ovn-metadata-sb-cfg"="662"}
name                : "900595a5-a02a-4566-b6dc-0c1e0e2cb392"
nb_cfg              : 662
nb_cfg_timestamp    : 1648150649986
```

In listing the `chassis_private` table, however, I see 7 entries. And wouldn't you know, **one** of those entries has an empty `chassis` list:

```
_uuid               : 3b580d16-2896-489f-8f1a-10d2cd13e1ae
chassis             : []
external_ids        : {"neutron:ovn-metadata-id"="bdc50d9c-42b1-5f20-8737-baba108b2f67", "neutron:ovn-metadata-sb-cfg"="425"}
name                : "5236f154-4a73-44ab-a588-b602a0b56bd5"
nb_cfg              : 425
nb_cfg_timestamp    : 1643778233179
```

That could explain, then, that as the agent code attempted to reference the `hostname` of a *null* chassis, a traceback would be encountered:

`AttributeError: 'Chassis_Private' object has no attribute 'hostname'`

On a whim, I deleted the errant row:

```
ovn-sbctl destroy chassis_private 3b580d16-2896-489f-8f1a-10d2cd13e1ae
```

Running the command again, I confirmed there were only six entries and that they lined up to corresponding chassis:

```
root@lab-infra02:~# ovn-sbctl list chassis
_uuid               : 6c90b020-be8e-4b7c-9aa8-0f4a9f826e6d
encaps              : [4c1cae4d-36a4-4541-af8c-fc02758fab4e, ac4dd143-10db-48c3-b4dd-8f42d0d6efd0]
external_ids        : {datapath-type=system, iface-types="bareudp,erspan,geneve,gre,gtpu,internal,ip6erspan,ip6gre,lisp,patch,stt,system,tap,vxlan", is-interconn="false", ovn-bridge-mappings="physnet1:br-rpn,vlan:br-ex", ovn-chassis-mac-mappings="", ovn-cms-options=enable-chassis-as-gw, ovn-enable-lflow-cache="true", ovn-limit-lflow-cache="", ovn-memlimit-lflow-cache-kb="", ovn-monitor-all="false", ovn-trim-limit-lflow-cache="", ovn-trim-wmark-perc-lflow-cache="", port-up-notif="true"}
hostname            : lab-compute01
name                : "0c9b25a6-3760-4b57-ba71-49e7091730bb"
nb_cfg              : 0
other_config        : {datapath-type=system, iface-types="bareudp,erspan,geneve,gre,gtpu,internal,ip6erspan,ip6gre,lisp,patch,stt,system,tap,vxlan", is-interconn="false", ovn-bridge-mappings="physnet1:br-rpn,vlan:br-ex", ovn-chassis-mac-mappings="", ovn-cms-options=enable-chassis-as-gw, ovn-enable-lflow-cache="true", ovn-limit-lflow-cache="", ovn-memlimit-lflow-cache-kb="", ovn-monitor-all="false", ovn-trim-limit-lflow-cache="", ovn-trim-wmark-perc-lflow-cache="", port-up-notif="true"}
transport_zones     : []
vtep_logical_switches: []

_uuid               : 478e3679-f4af-4a2d-a986-85323c840620
encaps              : [1e5060c3-a6ce-41bd-b54a-2ba3907f7092, 3177060c-bcdc-4c02-bf31-ff359c666538]
external_ids        : {datapath-type=system, iface-types="bareudp,erspan,geneve,gre,gtpu,internal,ip6erspan,ip6gre,lisp,patch,stt,system,tap,vxlan", is-interconn="false", ovn-bridge-mappings="vlan:br-ex", ovn-chassis-mac-mappings="", ovn-cms-options=enable-chassis-as-gw, ovn-enable-lflow-cache="true", ovn-limit-lflow-cache="", ovn-memlimit-lflow-cache-kb="", ovn-monitor-all="false", ovn-trim-limit-lflow-cache="", ovn-trim-wmark-perc-lflow-cache="", port-up-notif="true"}
hostname            : lab-compute03
name                : "1f318a3c-f607-4272-814c-b0c4d813daa5"
nb_cfg              : 0
other_config        : {datapath-type=system, iface-types="bareudp,erspan,geneve,gre,gtpu,internal,ip6erspan,ip6gre,lisp,patch,stt,system,tap,vxlan", is-interconn="false", ovn-bridge-mappings="vlan:br-ex", ovn-chassis-mac-mappings="", ovn-cms-options=enable-chassis-as-gw, ovn-enable-lflow-cache="true", ovn-limit-lflow-cache="", ovn-memlimit-lflow-cache-kb="", ovn-monitor-all="false", ovn-trim-limit-lflow-cache="", ovn-trim-wmark-perc-lflow-cache="", port-up-notif="true"}
transport_zones     : []
vtep_logical_switches: []

_uuid               : eed64f71-9cd1-4e1a-a891-e8bbb9049c41
encaps              : [3932bdff-e3a2-425b-9bf5-8d05fffbd171, a6f63e7b-a59a-41a3-9fdf-a2b6fae892cd]
external_ids        : {datapath-type=system, iface-types="bareudp,erspan,geneve,gre,gtpu,internal,ip6erspan,ip6gre,lisp,patch,stt,system,tap,vxlan", is-interconn="false", ovn-bridge-mappings="physnet2:br-rpn,vlan:br-ex", ovn-chassis-mac-mappings="", ovn-cms-options=enable-chassis-as-gw, ovn-enable-lflow-cache="true", ovn-limit-lflow-cache="", ovn-memlimit-lflow-cache-kb="", ovn-monitor-all="false", ovn-trim-limit-lflow-cache="", ovn-trim-wmark-perc-lflow-cache="", port-up-notif="true"}
hostname            : lab-compute02
name                : "6c2a75b1-482a-40e3-91f8-3e449986f5b6"
nb_cfg              : 174
other_config        : {datapath-type=system, iface-types="bareudp,erspan,geneve,gre,gtpu,internal,ip6erspan,ip6gre,lisp,patch,stt,system,tap,vxlan", is-interconn="false", ovn-bridge-mappings="physnet2:br-rpn,vlan:br-ex", ovn-chassis-mac-mappings="", ovn-cms-options=enable-chassis-as-gw, ovn-enable-lflow-cache="true", ovn-limit-lflow-cache="", ovn-memlimit-lflow-cache-kb="", ovn-monitor-all="false", ovn-trim-limit-lflow-cache="", ovn-trim-wmark-perc-lflow-cache="", port-up-notif="true"}
transport_zones     : []
vtep_logical_switches: []

_uuid               : 8a07dfc5-1e52-49aa-aa97-ec0515334fc6
encaps              : [63a84cd2-cb93-485e-aaa4-e6701dbb9a7d, a67c8b1c-da4e-4178-b0eb-58315983ca68]
external_ids        : {datapath-type=system, iface-types="bareudp,erspan,geneve,gre,gtpu,internal,ip6erspan,ip6gre,lisp,patch,stt,system,tap,vxlan", is-interconn="false", ovn-bridge-mappings="vlan:br-ex", ovn-chassis-mac-mappings="", ovn-cms-options=enable-chassis-as-gw, ovn-enable-lflow-cache="true", ovn-limit-lflow-cache="", ovn-memlimit-lflow-cache-kb="", ovn-monitor-all="false", ovn-trim-limit-lflow-cache="", ovn-trim-wmark-perc-lflow-cache="", port-up-notif="true"}
hostname            : lab-infra01
name                : "900595a5-a02a-4566-b6dc-0c1e0e2cb392"
nb_cfg              : 0
other_config        : {datapath-type=system, iface-types="bareudp,erspan,geneve,gre,gtpu,internal,ip6erspan,ip6gre,lisp,patch,stt,system,tap,vxlan", is-interconn="false", ovn-bridge-mappings="vlan:br-ex", ovn-chassis-mac-mappings="", ovn-cms-options=enable-chassis-as-gw, ovn-enable-lflow-cache="true", ovn-limit-lflow-cache="", ovn-memlimit-lflow-cache-kb="", ovn-monitor-all="false", ovn-trim-limit-lflow-cache="", ovn-trim-wmark-perc-lflow-cache="", port-up-notif="true"}
transport_zones     : []
vtep_logical_switches: []

_uuid               : 8f4829ad-746d-4125-8561-363adbbc4dce
encaps              : [ce7a5ab6-534a-4667-a7c0-5f112b0f4507, fcaf8226-fa90-4848-8c1d-d3c975276e05]
external_ids        : {datapath-type=system, iface-types="bareudp,erspan,geneve,gre,gtpu,internal,ip6erspan,ip6gre,lisp,patch,stt,system,tap,vxlan", is-interconn="false", "neutron:ovn-metadata-id"="344341e0-8e69-5e00-979c-d59fee1b9b27", "neutron:ovn-metadata-sb-cfg"="173", ovn-bridge-mappings="vlan:br-ex", ovn-chassis-mac-mappings="", ovn-cms-options=enable-chassis-as-gw, ovn-enable-lflow-cache="true", ovn-limit-lflow-cache="", ovn-memlimit-lflow-cache-kb="", ovn-monitor-all="false", ovn-trim-limit-lflow-cache="", ovn-trim-wmark-perc-lflow-cache="", port-up-notif="true"}
hostname            : lab-infra03
name                : "d50d391d-910f-40d6-8aa7-24fbfda018ff"
nb_cfg              : 171
other_config        : {datapath-type=system, iface-types="bareudp,erspan,geneve,gre,gtpu,internal,ip6erspan,ip6gre,lisp,patch,stt,system,tap,vxlan", is-interconn="false", ovn-bridge-mappings="vlan:br-ex", ovn-chassis-mac-mappings="", ovn-cms-options=enable-chassis-as-gw, ovn-enable-lflow-cache="true", ovn-limit-lflow-cache="", ovn-memlimit-lflow-cache-kb="", ovn-monitor-all="false", ovn-trim-limit-lflow-cache="", ovn-trim-wmark-perc-lflow-cache="", port-up-notif="true"}
transport_zones     : []
vtep_logical_switches: []

_uuid               : ada8b169-bd60-490b-8520-7e621cbbb84e
encaps              : [072fa878-8849-4b06-acfd-4e889ff308b0, 8b6e5992-a4bf-46e3-b1c2-d5494765ca62]
external_ids        : {datapath-type=system, iface-types="bareudp,erspan,geneve,gre,gtpu,internal,ip6erspan,ip6gre,lisp,patch,stt,system,tap,vxlan", is-interconn="false", "neutron:ovn-metadata-id"="83641d9c-6244-564c-b67c-d5b3298adc85", "neutron:ovn-metadata-sb-cfg"="574", ovn-bridge-mappings="vlan:br-ex", ovn-chassis-mac-mappings="", ovn-cms-options=enable-chassis-as-gw, ovn-enable-lflow-cache="true", ovn-limit-lflow-cache="", ovn-memlimit-lflow-cache-kb="", ovn-monitor-all="false", ovn-trim-limit-lflow-cache="", ovn-trim-wmark-perc-lflow-cache="", port-up-notif="true"}
hostname            : lab-infra02
name                : "30757b96-cb1b-4512-bfdd-df6df50f2f4c"
nb_cfg              : 171
other_config        : {datapath-type=system, iface-types="bareudp,erspan,geneve,gre,gtpu,internal,ip6erspan,ip6gre,lisp,patch,stt,system,tap,vxlan", is-interconn="false", ovn-bridge-mappings="vlan:br-ex", ovn-chassis-mac-mappings="", ovn-cms-options=enable-chassis-as-gw, ovn-enable-lflow-cache="true", ovn-limit-lflow-cache="", ovn-memlimit-lflow-cache-kb="", ovn-monitor-all="false", ovn-trim-limit-lflow-cache="", ovn-trim-wmark-perc-lflow-cache="", port-up-notif="true"}
transport_zones     : []
vtep_logical_switches: []
```

## Testing

So now, the moment of truth!

```
root@lab-infra01:~# openstack network agent list
HttpException: 500: Server Error for url: http://10.20.0.11:9696/v2.0/agents, Request Failed: internal server error while processing your request.
```

Dang.

I spent another few minutes mulling this over before considering a restart of the `neutron-server` service might be warranted. After restarting `neutron-server` across the three controller nodes, the following attempt worked:

```
root@lab-infra01:~# openstack network agent list
+--------------------------------------+------------------------------+--------------------------------------+-------------------+-------+-------+----------------------------+
| ID                                   | Agent Type                   | Host                                 | Availability Zone | Alive | State | Binary                     |
+--------------------------------------+------------------------------+--------------------------------------+-------------------+-------+-------+----------------------------+
| 1591b8ad-8a59-47f8-b1cf-53c4375eea5c | NIC Switch agent             | lab-infra03                          | None              | :-)   | UP    | neutron-sriov-nic-agent    |
| 16355a23-b872-4ec2-995e-208094f2057c | Baremetal Node               | 8919cf4d-a9dd-4985-ae70-835ba024e7b7 | None              | :-)   | UP    | ironic-neutron-agent       |
| 258a10ff-1090-4e90-a32c-4c6f8d01c938 | DHCP agent                   | lab-infra01                          | nova              | :-)   | UP    | neutron-dhcp-agent         |
| 29d9376e-dee5-41a1-9e86-e3d9607f4a59 | NIC Switch agent             | lab-infra02                          | None              | :-)   | UP    | neutron-sriov-nic-agent    |
| 346ba9ea-1c2d-4dc8-ba61-4cde37bbeaf9 | Metering agent               | lab-infra03                          | None              | :-)   | UP    | neutron-metering-agent     |
| 416d7511-3ef2-4bda-9b5c-157d2bef182a | Baremetal Node               | f7945b37-f43f-4b69-b987-1277d0a5777f | None              | :-)   | UP    | ironic-neutron-agent       |
| 467885c9-539e-4b9b-8bde-69405bf0597d | Baremetal Node               | 97c9e327-9b72-4566-a345-ca0544e28d14 | None              | :-)   | UP    | ironic-neutron-agent       |
| 57175ad2-02be-4d05-a9ee-08643a6393c8 | NIC Switch agent             | lab-infra01                          | None              | :-)   | UP    | neutron-sriov-nic-agent    |
| 594fdaab-d0be-4c69-8081-b293009b4808 | Metering agent               | lab-infra01                          | None              | :-)   | UP    | neutron-metering-agent     |
| 8533ec17-f8f5-4240-a085-98f158a981df | NIC Switch agent             | lab-compute03                        | None              | :-)   | UP    | neutron-sriov-nic-agent    |
| 868a1ae9-3f3b-4574-9fce-0ff1762df160 | Metering agent               | lab-infra02                          | None              | :-)   | UP    | neutron-metering-agent     |
| 8bc8691c-064b-4661-b4d0-f2ca778012ee | DHCP agent                   | lab-infra02                          | nova              | :-)   | UP    | neutron-dhcp-agent         |
| b126376b-e253-47f8-b22e-fce5ffb87f94 | Baremetal Node               | 1ff24bbc-6058-41f9-aad5-7d4e78c81695 | None              | :-)   | UP    | ironic-neutron-agent       |
| b589a112-0877-4968-a6f5-04a3e3a383b6 | NIC Switch agent             | lab-compute01                        | None              | :-)   | UP    | neutron-sriov-nic-agent    |
| b5c326c6-cbe3-42b2-a78c-bf3008272dc1 | NIC Switch agent             | lab-compute02                        | None              | :-)   | UP    | neutron-sriov-nic-agent    |
| c2b0c5e4-9499-4d97-8ecf-f09c7496b0bd | DHCP agent                   | lab-infra03                          | nova              | :-)   | UP    | neutron-dhcp-agent         |
| da06498a-fc06-45a0-bbba-1568f700cca6 | Baremetal Node               | eac40a3f-3854-426c-b232-7ae7df4ab549 | None              | :-)   | UP    | ironic-neutron-agent       |
| 900595a5-a02a-4566-b6dc-0c1e0e2cb392 | OVN Controller Gateway agent | lab-infra01                          |                   | :-)   | UP    | ovn-controller             |
| 64b68ff2-b068-5e64-a1cd-9c95afadd0b7 | OVN Metadata agent           | lab-infra01                          |                   | :-)   | UP    | neutron-ovn-metadata-agent |
| 1f318a3c-f607-4272-814c-b0c4d813daa5 | OVN Controller Gateway agent | lab-compute03                        |                   | :-)   | UP    | ovn-controller             |
| 4d9e06dc-69c0-5ea7-8a6d-e750d11ebb9f | OVN Metadata agent           | lab-compute03                        |                   | :-)   | UP    | neutron-ovn-metadata-agent |
| d50d391d-910f-40d6-8aa7-24fbfda018ff | OVN Controller Gateway agent | lab-infra03                          |                   | :-)   | UP    | ovn-controller             |
| 344341e0-8e69-5e00-979c-d59fee1b9b27 | OVN Metadata agent           | lab-infra03                          |                   | :-)   | UP    | neutron-ovn-metadata-agent |
| 0c9b25a6-3760-4b57-ba71-49e7091730bb | OVN Controller Gateway agent | lab-compute01                        |                   | :-)   | UP    | ovn-controller             |
| 2864488c-c9a8-5cf1-b1c0-184c295493b6 | OVN Metadata agent           | lab-compute01                        |                   | :-)   | UP    | neutron-ovn-metadata-agent |
| 6c2a75b1-482a-40e3-91f8-3e449986f5b6 | OVN Controller Gateway agent | lab-compute02                        |                   | :-)   | UP    | ovn-controller             |
| 6645f143-2dc0-5f03-b7bb-681bc3e8b969 | OVN Metadata agent           | lab-compute02                        |                   | :-)   | UP    | neutron-ovn-metadata-agent |
| 30757b96-cb1b-4512-bfdd-df6df50f2f4c | OVN Controller Gateway agent | lab-infra02                          |                   | :-)   | UP    | ovn-controller             |
| 83641d9c-6244-564c-b67c-d5b3298adc85 | OVN Metadata agent           | lab-infra02                          |                   | :-)   | UP    | neutron-ovn-metadata-agent |
+--------------------------------------+------------------------------+--------------------------------------+-------------------+-------+-------+----------------------------+
```

## Summary

This was not the first time I'd come across this issue, and unfortunately, can neither understand *why* it happens and what I did last time to fix it. It's probably obvious I did something similar, but I've slept since then and don't recall.

The following links were helpful in gaining a better understanding of what is/was happening and upstream changes being put in place to either keep it from happening in the future or more gracefully recover:

[https://review.opendev.org/c/openstack/neutron/+/797796/](https://review.opendev.org/c/openstack/neutron/+/797796/)
[https://bugzilla.redhat.com/show_bug.cgi?id=1975264](https://bugzilla.redhat.com/show_bug.cgi?id=1975264)
[https://review.opendev.org/c/openstack/neutron/+/818132](https://review.opendev.org/c/openstack/neutron/+/818132)

---
If you have some thoughts or comments on this post, I'd love to hear 'em. Feel free to reach out on Twitter at @jimmdenton or hit me up on LinkedIn.
