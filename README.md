## OpenShift ETCD Upgrade Considerations

### Release versions and updates

OpenShift Container Platorm (OCP) is semantically versioned - https://semver.org/

For OpenShift, each release that gets it's own channel is supported in parallel with other channel-based releases. 

The `minor` number is used for the channel distinction e.g `major.minor` = 3.5, 3.6, 3.7

Security and/or bugfix errata for minor releases are called `asyncronous errata` delineated by `major.minor.patch` e.g 3.7.23, 3.7.14-1

Migrating from one of these `minor` releases to another requires a more involved upgrade. Updating `patch` async errata is normally not that involved. 

Always read the release notes for each upgrade to ascertain important upgrade information.

If you're applying more than one errata at a time (e.g. if you upgrade directly from 3.7.9 to 3.7.23) you don't need to follow all the steps of each sub-update one by one; in particular, you only need to run the final upgrade commands at the end of the process (not once for 3.7.9, 3.7.14-1 and 3.7.23)

So, when performing an in-place upgrade from OCP 3.5 -> 3.7 one would upgrade as follows:

```
OCP 3.5 -> 3.6 -> 3.7
```

Current OCP versions for the 3.5 -> 3.7 channels:

```
3.7.23 (latest) - Mon Jan 22 2018
3.7.14-1
3.7.9

3.6.173.0.96 - Mon Jan 22 2018
3.6.173.0.83
3.6.173.0.63
3.6.173.0.49
3.6.173.0.21
3.6.173.0.5

3.5.5.31.48 - Wed Jan 10 2018
3.5.5.31.47
3.5.5.31.36
3.5.5.31.24
3.5.5.31.19
3.5.5.31
3.5.5.26
3.5.5.24
3.5.5.15
3.5.5.8
3.5.5.5
```

It is good practice when eprforming upgrades to an automated upgrade process and test in a non-production environment that matches production to pick up any local changes and issues that need to be factored in.

### ETCD versions, migrations and relationship to OpenShift versions

The stated component table (just showing etcd software version) does not show the underlying etcd `data model` versions supported:

```
Components	3.0	3.1	3.2	3.3	3.4	3.5	3.6	3.7
---------------------------------------------------------------------------
etcd		2.1.1	2.1.1	2.2.5	2.3.7	3.0.x	3.0.x	-	-
etcd3		-	-	-	-	-	3.1.x	3.1.x	3.2.x
```

OpenShift Container Platform Tested Integrations - https://access.redhat.com/articles/2176281

ETCD has changed software versions from 2.X -> 3.X in OCP 3.4. However etcd supports both v2 and v3 `data models` in the 3.1+ software release.

The default etcd `data model` version did not change from v2->v3 until OCP 3.6

By moving to the etcd3 v3 data model, there is now:
- Larger memory space to enable larger cluster sizes.
- Increased stability in adding and removing nodes in general life cycle actions.
- A significant performance boost.

Performance improvement graphs between v2 and v3 etcd data models for the same etcd 3.1.7 version can be found here:

https://docs.openshift.com/container-platform/3.7/scaling_performance/host_practices.html#scaling-performance-capacity-host-practices-etcd

New OCP 3.6 installs will use a v3 etcd database whereas OCP 3.5 and upgraded OCP 3.6 environments use a v2 database. 

OCP 3.6 supports both the v2 and v3 schemas, for OCP 3.7 you need to migrate the schema before the upgrade.

So, as a result, all OCP 3.6 clusters, before they upgrade to OCP 3.7 will be running the v3 schema (if even for only a short period of time). 

There are automated ansible playbooks for this migration. The etcd migration documentation should be read in conjunction with the overall upgrade process. The in-place and blue-green upgrade playbooks take etcd backups immediately before and immediately after the upgrade process.

ETCD Migration Documentation:

https://docs.openshift.com/container-platform/3.7/install_config/upgrading/migrating_etcd.html

The etcd v2 to v3 data migration is performed as an offline migration which means all etcd members and master services are stopped during the migration. Large clusters with up to 600MiB of etcd data can expect a 10 to 15 minute outage of the API, web console, and controllers.

You can only begin the etcd data migration process after upgrading to OpenShift Container Platform 3.6, as previous versions are not compatible with etcd v3 storage. 

Additionally, the upgrade to OpenShift Container Platform 3.6 reconfigures cluster DNS services to run on every node,
rather than on the masters, which ensures that, even when master services are taken down, existing pods continue to function as expected (see DNS section below for more details on this).

Recovering from etcd Migration Issues using backups is Documented here:

https://docs.openshift.com/container-platform/3.7/install_config/upgrading/migrating_etcd.html#etcd-data-migration-recovering


### Embedded vs External ETCD

Until OCP 3.6, it was possible to deploy a cluster with an embedded etcd.

As of 3.7, this is no longer possible. 

The easiest way is to check if your cluster is using embedded etcd or not - is to read the ansible hosts inventory which should have an `[etcd]` host defined for external. You can also check if the master has an etcd service.

If it's an HA cluster it's not embedded, so the vast majority of production environments won't need to perform this migration.

If you attempt to migrate an embedded environment from v2 to v3 it will block and instruct you to migrate to an external etcd first.

There are automated ansible playbooks for this migration. See Migrating Embedded etcd to External etcd for more information.

https://docs.openshift.com/container-platform/3.7/install_config/upgrading/migrating_embedded_etcd.html#install-config-upgrading-etcd-data-migration

Co-hosting etcd with master-api server id advisable as they communicate frequently. Also one process is I/O heavy, the other CPU heavy so they are good to co-host together.

The recommended etcd cluster size is `3` members for the vast majority of HA Cluster use cases.


### DNS Changes between OCP 3.5 and 3.6, HA in-place upgrades, zero downtime

ETCD data migration requires an outage for the control plane. This would have caused issues with service resolution using SkyDNS in earlier releases of OpenShift.

OCP 3.6 dns requests from a pod hit dnsmasq on the node then route internal queries to the node service which maintains a cache of all DNS service information. This means DNS should continue to work even when the master API is down so long as the node service is not restarted.

For HA clusters that require workloads to be highly available, this is advantageous as your business services will remain up and running. Node and existing pods should continue to run without any impact. You cannot make any changes to the environment so builds, etc won't happen during the downtime and the web console will be offline.

A good description of DNS changes between versions can be found here:

https://www.redhat.com/en/blog/red-hat-openshift-container-platform-dns-deep-dive-dns-changes-red-hat-openshift-container-platform-36


### Notable Enhancements and Bug Fixes realted to ETCD for OCP releases 3.5 -> 3.7

For information - here is a list (not guaranteed to be exhaustive!) of fixes and enhancements that relate to ETCD.

##### 3.5.5.31.36
In some failure cases, the etcd client used by OpenShift will not rotate through all the available etcd cluster members - https://bugzilla.redhat.com/show_bug.cgi?id=1490428

##### 3.5.5.26
With this enhancement, the etcd CA certificate may now be replaced independent of the OpenShift CA certificate using the etcd CA certificate redeployment playbook 
```
playbooks/byo/openshift-cluster/redeploy-etcd-ca.yml
```

##### 3.6.x
- Starting with new installations of OpenShift Container Platform 3.6, the etcd3 v3 data model is the default.
- Installation of etcd, docker daemon, and ansible installer as system containers (technology preview)
- New Installer Cluster Variables
```
  # CA, node and master certificate expiry
  openshift_ca_cert_expire_days=1825
  openshift_node_cert_expire_days=730
  openshift_master_cert_expire_days=730
  # Registry certificate expiry
  openshift_hosted_registry_cert_expire_days=730
  # Etcd CA, peer, server and client certificate expiry
  etcd_ca_default_days=1825    
```

##### 3.6.173.0.96

Encryption of secret data at the datastore layer (etcd). While the examples use the secrets resource, any resource can be encrypted, such as configmaps. etcd v3 or later is required in order to use this feature. 

This is fully supported feature in OCP 3.6.1 (back-ported from OCP 3.7) - there were documentation bugs stating it was experimental. The feature is considered alpha (as API's may change), but nonetheless it is fully supported.

https://access.redhat.com/documentation/en-us/openshift_container_platform/3.6/html-single/cluster_administration/#encrypting-data-overview

##### 3.7.9
- Providing storage to an etcd node using pci passthrough with openstack
Guidance on Providing Storage to an etcd Node Using PCI Passthrough with OpenStack is now available.
- Migrate etcd before openshift container platform 3.7 upgrade. Starting in OpenShift Container Platform 3.7, the use of the etcd3 v3 data model is required.
- Additional health checks for etcd
```
https://docs.openshift.com/container-platform/3.7/release_notes/ocp_3_7_release_notes.html#ocp-37-additional-health-checks

ansible-playbook playbooks/byo/openshift-checks/adhoc.yml
                  curator
                  diagnostics
                  disk_availability
                  docker_image_availability
                  docker_storage
                  elasticsearch
                  etcd_imagedata_size
                  etcd_traffic
                  etcd_volume
                  fluentd
                  fluentd_config
                  kibana
                  logging
                  logging_index_time
                  memory_availability
                  ovs_version
                  package_availability
                  package_update
                  package_version
```

https://bugzilla.redhat.com/show_bug.cgi?id=1523814

When running the etcd v2 to v3 migration playbooks as included in the OpenShift Container Platform 3.7 release, the playbooks incorrectly assumed that all services were HA services (atomic-openshift-master-api and atomic-openshift-master-controllers rather than atomic-openshift-master), which is the norm on version 3.7. However, the migration playbooks would be executed prior to upgrading to version 3.7, so this was incorrect. The migration playbooks have been updated to start and stop the correct services ensuring proper migration. (BZ#1523814)

https://bugzilla.redhat.com/show_bug.cgi?id=1475867

The openshift jenkins sync plug-in was updating Jenkins pipeline build status annotations every second, regardless of whether the status changed. The frequency of updates would put unnecessary stress on the etcd instance backing openshift master. Now, Jenkins pipeline build status annotations are only updated if the status actually changes, or 30 seconds have passed. (BZ#1475867)

https://bugzilla.redhat.com/show_bug.cgi?id=1492891

The etcd quota backend was set to 2GB by default. This resulted in a cluster going into a hold state, blocking all writes into the etcd storage. The default quota backend was increased to 4GB by default to encompass the storage needs of bigger clusters. (BZ#1492891)

https://bugzilla.redhat.com/show_bug.cgi?id=1501752

Previously, the etcd v3 data migrated prior to the first etcd v2 snapshot being written. Without a v2 snapshot, the v3 data was not propagated properly to the remaining etcd members, which resulted in a loss of some v3 data. This bug fix checks to see if there is at least one v2 snapshot before etcd data migration proceeds. As a result, etcd v3 data is now properly distributed among all etcd members. (BZ#1501752)
 
https://bugzilla.redhat.com/show_bug.cgi?id=1504515

When trying to upgrade OpenShift Container Platform with dedicated etcd from v3.6 to v3.7, the upgrade failed at the [Stop atomic-openshift-master-controllers] task due to the wrong hosts group. This bug fix corrected the host group to specify the masters group for controller restart. As a result, the upgrade now succeeds. (BZ#1504515)


### Other Anonymous Gotchas from ETCD upgrade proces 

After a slightly-botched OCP 3.4 -> 3.5 -> 3.6 upgrade, the etcd datastore migration playbook was not run after the upgrade so the v2 on-disk format was still being used and existing secrets did not appear to have been encrypted. Resolved once v3 format complete and manual removal of v2 secrets was required.

Upgraded to OCP 3.6 and migratd etcd from v2 to v3 schema. When they run the migrate.yml playbook, it consistently fails on the first etcd_cluster_health task. etcdctl commands were failing when running the ansible playbook. `NO_PROXY` entries were required for masters `and` etcd hosts to avoid using a proxy when connecting to etcd.


### Summary - order of upgrade steps

For in place upgrades of an existing OCP 3.5 Cluster:

- Upgrade OCP 3.5 to latest patch version
```
https://docs.openshift.com/container-platform/3.5/install_config/upgrading/automated_upgrades.html
```  
- Upgrade OCP 3.5 -> 3.6 using existing v2 etcd data model.
```
https://docs.openshift.com/container-platform/3.6/install_config/upgrading/automated_upgrades.html
```
- Check OCP 3.6 etcd embedded vs external. Migrate to external etcd if required.
```
https://docs.openshift.com/container-platform/3.7/install_config/upgrading/migrating_embedded_etcd.html#install-config-upgrading-etcd-data-migration
```
- Migrate OCP 3.6 etcd v2 to v3 data model.
```
https://docs.openshift.com/container-platform/3.7/install_config/upgrading/migrating_etcd.html 
  https://docs.openshift.com/container-platform/3.7/install_config/upgrading/migrating_etcd.html#etcd-data-migration-recovering
```
- Upgrade OCP 3.6 to 3.7.
```
https://docs.openshift.com/container-platform/3.7/install_config/upgrading/automated_upgrades.html
```
- Configure Secret encryption if required.
```
https://docs.openshift.com/container-platform/3.7/admin_guide/encrypting_data.html
```
