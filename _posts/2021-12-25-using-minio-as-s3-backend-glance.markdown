---
title: "Using Minio as S3 Backend for OpenStack Glance"
layout: post
date: 2021-12-25
image: /assets/images/2021-12-25-using-minio-as-s3-backend-glance/s3.png
headerImage: true
tag:
- minio
- s3
- openstack
- glance
category: blog
blog: true
author: jamesdenton
description: "Using Minio as S3 Backend for OpenStack Glance"
---

My homelab consists of a few random devices, including a Synology NAS that doubles as a home backup system. I use NFS to provide shared storage for Glance images and Cinder volumes, and Synology even has Cinder drivers that leverage iSCSI. All-in-all, it's a pretty useful setup to test a myriad of OpenStack functionality.

I recently discovered Minio, which is an open-source object storage solution that provides S3 compatibility. Installable with Docker, I thought I'd give it a go and test OpenStack's *reintroduced* support for S3 backends in Glance. 
<!--more-->

## Configuring Minio

To install Minio in Docker in DSM, I followed a [guide](https://jonaharagon.me/installing-minio-on-synology-diskstation-4823caf600c3) that, while a little old, worked out well enough. In my environment, using `host` networking vs `bridge` worked out better.

Once installed, it requires a minimal amount of configuration to work with Glance. You will need:

- a user with r/w permissions
- a region defined

To create the user, navigate to **Users** -> **Create User** and provide an ACCESS KEY and SECRET KEY and appropriate permissions:

```
ACCESS_KEY: openstack
SECRET_KEY: 0p3nstack
POLICY: readwrite
```

To define a region, navigate to **Settings** -> **Region** and set the region name in the **Server Location** field. I originally set `us-south-lab`, but due to some pre-configured assumptions in the `boto3` python client, I had to change this to `us-east-1` for things to work properly.

## Configuring OpenStack

There are some overides on the OpenStack-Ansible side that must be configured to allow the playbooks to properly configure Glance for the additional backend. Use the `glance_additional_stores` variable, taking care to ensure that any defaults are also specified (since you're overriding the default variable).

The value for `name` is arbitrary, and used as an identifier for specific settings that will also be defined, while `type` is a specific type of Glance backend.

```
glance_additional_stores:
  - http
  - cinder
  - name: minio
    type: s3
```

Addition to `glance_additional_stores`, you must define a new configuration block that maps to the new backend definition. For OpenStack-Ansible, this can be done as a config override:

```
glance_glance_api_conf_overrides:
  minio:
    s3_store_host: http://172.22.0.4:9000
    s3_store_access_key: openstack
    s3_store_secret_key: 0p3nstack
    s3_store_bucket: glance
    s3_store_create_bucket_on_put: True
    s3_store_bucket_url_format: auto
```

In `glance-api.conf`, the override above will be written as this:

```
[minio]
s3_store_host = http://172.22.0.4:9000
s3_store_access_key = openstack
s3_store_secret_key = 0p3nstack
s3_store_bucket = glance
s3_store_create_bucket_on_put = True
s3_store_bucket_url_format = auto
```

## Testing the Backend

If the default Glance backend (file) has not been changed, it is still possible to upload individual images to the new S3 backend using the `glance` client.

In this example, a Cirros image will be uploaded to the `minio` store:

```
root@lab-infra01:~/images# glance image-create --file cirros-0.5.1-x86_64-disk.img --disk-format raw --container-format bare --name cirros3 --store minio --progress
[=============================>] 100%
+------------------+----------------------------------------------------------------------------------+
| Property         | Value                                                                            |
+------------------+----------------------------------------------------------------------------------+
| checksum         | 1d3062cd89af34e419f7100277f38b2b                                                 |
| container_format | bare                                                                             |
| created_at       | 2021-12-24T04:23:19Z                                                             |
| disk_format      | raw                                                                              |
| id               | 53627724-da3e-4b81-9910-55598d9393d4                                             |
| locations        | [{"url": "s3://openstack:0p3nstack@172.22.0.4:9000/glance/53627724-da3e-4b81-991 |
|                  | 0-55598d9393d4", "metadata": {"store": "minio"}}]                                |
| min_disk         | 0                                                                                |
| min_ram          | 0                                                                                |
| name             | cirros3                                                                          |
| os_hash_algo     | sha512                                                                           |
| os_hash_value    | 553d220ed58cfee7dafe003c446a9f197ab5edf8ffc09396c74187cf83873c877e7ae041cb80f3b9 |
|                  | 1489acf687183adcd689b53b38e3ddd22e627e7f98a09c46                                 |
| os_hidden        | False                                                                            |
| owner            | 7a8df96a3c6a47118e60e57aa9ecff54                                                 |
| protected        | False                                                                            |
| size             | 16338944                                                                         |
| status           | active                                                                           |
| stores           | minio                                                                            |
| tags             | []                                                                               |
| updated_at       | 2021-12-24T04:23:21Z                                                             |
| virtual_size     | 16338944                                                                         |
| visibility       | shared                                                                           |
+------------------+----------------------------------------------------------------------------------+
```

Once uploaded, an instance can be created by specifying the new image name or UUID.

## Benchmarking Minio

The Minio team provides a benchmarking utility named Warp, which is available on [Github](https://github.com/minio/warp) as source code of pre-compiled binaries.

To test, you'll need the Minio endpoint along with the access and secret keys:

```
# warp mixed --host=172.22.0.4:9000 --access-key=openstack --secret-key=0p3nstack --autoterm

Throughput 7.3 objects/s within 7.500000% for 25.802s. Assuming stability. Terminating benchmark.
warp: Benchmark data written to "warp-mixed-2021-12-24[050521]-hCzP.csv.zst"
Mixed operations.
Operation: DELETE, 10%, Concurrency: 20, Ran 1m33s.
 * Throughput: 2.39 obj/s

Operation: GET, 44%, Concurrency: 20, Ran 1m33s.
 * Throughput: 104.80 MiB/s, 10.48 obj/s

Operation: PUT, 15%, Concurrency: 20, Ran 1m32s.
 * Throughput: 36.17 MiB/s, 3.62 obj/s

Operation: STAT, 30%, Concurrency: 20, Ran 1m33s.
 * Throughput: 7.00 obj/s

Cluster Total: 139.94 MiB/s, 23.35 obj/s over 1m34s.
```

The NAS hosting this instance of Minio is a DS1815+ with 4x 6TB 6Gbps SATA disks and 1Gbps networking. Things look considerably better with a different NAS (DS1621+) using NVMe and 10Gbps networking:

```
# warp mixed --host=10.22.0.4:9000 --access-key=openstack --secret-key=0p3nstack --autoterm

Throughput 51.6 objects/s within 7.500000% for 13.489s. Assuming stability. Terminating benchmark.
warp: Benchmark data written to "warp-mixed-2021-12-27[153057]-mzH0.csv.zst"
Mixed operations.
Operation: DELETE, 10%, Concurrency: 20, Ran 48s.
 * Throughput: 16.51 obj/s

Operation: GET, 45%, Concurrency: 20, Ran 48s.
 * Throughput: 742.10 MiB/s, 74.21 obj/s

Operation: PUT, 15%, Concurrency: 20, Ran 48s.
 * Throughput: 248.44 MiB/s, 24.84 obj/s

Operation: STAT, 30%, Concurrency: 20, Ran 48s.
 * Throughput: 49.47 obj/s

Cluster Total: 986.46 MiB/s, 164.60 obj/s over 48s.
```

## Summary

I was glad to see that the S3 backend had been re-introduced in Ussuri after being deprecated around the Mitaka timeframe, and having some local object storage options is nice for testing and for eventually setting up Cinder volume backups. Using something like Ceph (for object) is a bit overkill for my usecases, and another administrative headache I don't want to deal with. I might try to implement a Swift proxy to translate Swift -> S3 for Ironic, but will leave that for another day.

---
If you have some thoughts or comments on this article, I'd love to hear 'em. Feel free to reach out on Twitter at @jimmdenton or hit me up on LinkedIn.
