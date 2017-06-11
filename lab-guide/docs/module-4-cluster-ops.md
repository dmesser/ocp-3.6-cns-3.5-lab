!!! Summary "Overview"
    In this module you be introduced to some standard operational procedures. You will learn how to run multiple GlusterFS *Trusted Storage Pools* on OpenShift and how to expand and maintain deployments.

    Herein, we will use the term pool (GlusterFS terminology) and cluster (heketi terminology) interchangeably.

Running multiple GlusterFS pools
--------------------------------

In the previous modules a single GlusterFS clusters was used to supply *PersistentVolume* to applications. CNS allows for multiple clusters to run in a single OpenShift deployment.

There are several use cases for this:

1. Provide data isolation between clusters of different tenants

1. Provide multiple performance tiers of CNS, i.e. HDD-based vs. SSD-based

1. Overcome the maximum amount of 300 PVs per cluster, currently imposed in version 3.5

To deploy a second GlusterFS pool follow these steps:

&#8680; Log in as `operator` to namespace `container-native-storage`

    oc login -u operator -n container-native-storage

Your deployment has 6 OpenShift Application Nodes in total, `node-1`, `node-2` and `node-3` currently setup running CNS. We will now set up a second cluster using `node-4`, `node-5` and `node-6`.

&#8680; Apply additional labels (user-defined key-value pairs) to the remaining 3 OpenShift Nodes:

    oc label node/node-4.lab storagenode=glusterfs
    oc label node/node-5.lab storagenode=glusterfs
    oc label node/node-6.lab storagenode=glusterfs

The label will be used to control GlusterFS pod placement and availability.

&#8680; Wait for all pods to show `1/1` in the `READY` column:

     oc get pods -o wide -n container-native-storage

For bulk import of new nodes into a new cluster, the topology JSON file can be updated to include a second cluster with a separate set of nodes.

&#8680; Create a new file named updated-topology.json with the content below:

<kbd>updated-topology.json:</kbd>

```json hl_lines="54"
{
    "clusters": [
        {
            "nodes": [
                {
                    "node": {
                        "hostnames": {
                            "manage": [
                                "node-1.lab"
                            ],
                            "storage": [
                                "10.0.2.101"
                            ]
                        },
                        "zone": 1
                    },
                    "devices": [
                        "/dev/xvdc"
                    ]
                },
                {
                    "node": {
                        "hostnames": {
                            "manage": [
                                "node-2.lab"
                            ],
                            "storage": [
                                "10.0.3.102"
                            ]
                        },
                        "zone": 2
                    },
                    "devices": [
                        "/dev/xvdc"
                    ]
                },
                {
                    "node": {
                        "hostnames": {
                            "manage": [
                                "node-3.lab"
                            ],
                            "storage": [
                                "10.0.4.103"
                            ]
                        },
                        "zone": 3
                    },
                    "devices": [
                        "/dev/xvdc"
                    ]
                }
            ]
        },
        {
            "nodes": [
                {
                    "node": {
                        "hostnames": {
                            "manage": [
                                "node-4.lab"
                            ],
                            "storage": [
                                "10.0.2.104"
                            ]
                        },
                        "zone": 1
                    },
                    "devices": [
                        "/dev/xvdc"
                    ]
                },
                {
                    "node": {
                        "hostnames": {
                            "manage": [
                                "node-5.lab"
                            ],
                            "storage": [
                                "10.0.3.105"
                            ]
                        },
                        "zone": 2
                    },
                    "devices": [
                        "/dev/xvdc"
                    ]
                },
                {
                    "node": {
                        "hostnames": {
                            "manage": [
                                "node-6.lab"
                            ],
                            "storage": [
                                "10.0.4.106"
                            ]
                        },
                        "zone": 3
                    },
                    "devices": [
                        "/dev/xvdc"
                    ]
                }
            ]
        }
    ]
}
```

The file contains the same content as `topology.json` with a second cluster specification (beginning at the highlighted line).

When loading this topology to heketi, it will recognize the existing cluster (leaving it unchanged) and start creating the new one, with the same bootstrapping process as seen with `cns-deploy`.

&#8680; Prepare the heketi CLI tool like previously in [Module 2](module-2-deploy-cns.md#heketi-env-setup).

    export HEKETI_CLI_SERVER=http://heketi-container-native-storage.cloudapps.34.252.58.209.nip.io
    export HEKETI_CLI_USER=admin
    export HEKETI_CLI_KEY=myS3cr3tpassw0rd

&#8680; Verify there is currently only a single cluster known to heketi

    heketi-cli cluster list

Example output:

    Clusters:
    fb67f97166c58f161b85201e1fd9b8ed

**Note this cluster's id** for later reference.

&#8680; Load the new topology with the heketi client

    heketi-cli topology load --json=updated-topology.jso

You should see output similar to the following:

    Found node node-1.lab on cluster fb67f97166c58f161b85201e1fd9b8ed
    Found device /dev/xvdc
    Found node node-2.lab on cluster fb67f97166c58f161b85201e1fd9b8ed
    Found device /dev/xvdc
    Found node node-3.lab on cluster fb67f97166c58f161b85201e1fd9b8ed
    Found device /dev/xvdc
    Creating cluster ... ID: 46b205a4298c625c4bca2206b7a82dd3
    Creating node node-4.lab ... ID: 604d2eb15a5ca510ff3fc5ecf912d3c0
    Adding device /dev/xvdc ... OK
    Creating node node-5.lab ... ID: 538b860406870288af23af0fbc2cd27f
    Adding device /dev/xvdc ... OK
    Creating node node-6.lab ... ID: 7736bd0cb6a84540860303a6479cacb2
    Adding device /dev/xvdc ... OK

As indicated from above output a new cluster got created.

&#8680; List all clusters:

    heketi-cli cluster list

You should see a second cluster in the list:

    Clusters:
    46b205a4298c625c4bca2206b7a82dd3
    fb67f97166c58f161b85201e1fd9b8ed

The second cluster with the ID `46b205a4298c625c4bca2206b7a82dd3` is an entirely independent GlusterFS deployment. **Note this second cluster's ID** as well for later reference.

Now we have two independent GlusterFS clusters managed by the same heketi instance:

| | Nodes | Cluster UUID |
|------------|--------| -------- |
| First Cluster | node-1, node-2, node-3    | fb67f97166c58f161b85201e1fd9b8ed |
| Second Cluster  | node-4, node-5, node-6     | 46b205a4298c625c4bca2206b7a82dd3  |

&#8680; Query the updated topology:

    heketi-cli topology info

Abbreviated output:

    Cluster Id: 46b205a4298c625c4bca2206b7a82dd3

        Volumes:

        Nodes:

          Node Id: 538b860406870288af23af0fbc2cd27f
          State: online
          Cluster Id: 46b205a4298c625c4bca2206b7a82dd3
          Zone: 2
          Management Hostname: node-5.lab
          Storage Hostname: 10.0.3.105
          Devices:
          	Id:e481d022cea9bfb11e8a86c0dd8d3499   Name:/dev/xvdc           State:online    Size (GiB):499     Used (GiB):0       Free (GiB):499
          		Bricks:

          Node Id: 604d2eb15a5ca510ff3fc5ecf912d3c0
          State: online
          Cluster Id: 46b205a4298c625c4bca2206b7a82dd3
          Zone: 1
          Management Hostname: node-4.lab
          Storage Hostname: 10.0.2.104
          Devices:
          	Id:09a25a114c53d7669235b368efd2f8d1   Name:/dev/xvdc           State:online    Size (GiB):499     Used (GiB):0       Free (GiB):499
          		Bricks:

          Node Id: 7736bd0cb6a84540860303a6479cacb2
          State: online
          Cluster Id: 46b205a4298c625c4bca2206b7a82dd3
          Zone: 3
          Management Hostname: node-6.lab
          Storage Hostname: 10.0.4.106
          Devices:
          	Id:cccadb2b54dccd99f698d2ae137a22ff   Name:/dev/xvdc           State:online    Size (GiB):499     Used (GiB):0       Free (GiB):499
          		Bricks:

    Cluster Id: fb67f97166c58f161b85201e1fd9b8ed

    [...output omitted for brevity...]

heketi formed an new, independent 3-node GlusterFS cluster on those nodes.

&#8680; Check running GlusterFS pods

    oc get pods -o wide

From the output you can spot the pod names running on the new cluster's nodes:

``` hl_lines="2 3 5"
NAME              READY     STATUS    RESTARTS   AGE       IP           NODE
glusterfs-1nvtj   1/1       Running   0          23m       10.0.4.106   node-6.lab
glusterfs-5gvw8   1/1       Running   0          24m       10.0.2.104   node-4.lab
glusterfs-5rc2g   1/1       Running   0          4h        10.0.2.101   node-1.lab
glusterfs-b4wg1   1/1       Running   0          24m       10.0.3.105   node-5.lab
glusterfs-jbvdk   1/1       Running   0          4h        10.0.3.102   node-2.lab
glusterfs-rchtr   1/1       Running   0          4h        10.0.4.103   node-3.lab
heketi-1-tn0s9    1/1       Running   0          4h        10.130.2.3   node-6.lab
```

!!! Note:
    Again note that the pod names are dynamically generated and will be different. Use the FQDN of your hosts to determine one of new cluster's pods.

&#8680; Log on to one of the new cluster's pods:

    oc rsh glusterfs-1nvtj

&#8680; Query this GlusterFS node for it's peers:

    gluster peer status

As expected this node only has 2 peers, evidence that it's running in it's own GlusterFS pool separate from the first cluster in deployed in Module 2.

    Number of Peers: 2

    Hostname: node-5.lab
    Uuid: 0db9b5d0-7fa8-4d2f-8b9e-6664faf34606
    State: Peer in Cluster (Connected)
    Other names:
    10.0.3.105

    Hostname: node-4.lab
    Uuid: 695b661d-2a55-4f94-b22e-40a9db79c01a
    State: Peer in Cluster (Connected)
    sh-4.2# gluster peer status
    Number of Peers: 2

    Hostname: node-5.lab
    Uuid: 0db9b5d0-7fa8-4d2f-8b9e-6664faf34606
    State: Peer in Cluster (Connected)
    Other names:
    10.0.3.105

    Hostname: node-4.lab
    Uuid: 695b661d-2a55-4f94-b22e-40a9db79c01a
    State: Peer in Cluster (Connected)

Before you can use the second cluster two tasks have to be accomplished:

1. The *StorageClass* for the first cluster has to be updated to point the first cluster's UUID,

1. A second *StorageClass* for the second cluster has to be created, pointing to the same heketi

!!! Tip "Why do we need to update the first *StorageClass*"?
    By default, when no cluster UUID is specified, heketi serves volume creation requests from any cluster currently registered to it.
    In order to request a volume from specific cluster you have to supply the cluster's UUID to heketi. The first *StorageClass* has no UUID specified so far.

&#8680; Edit the existing *StorageClass* definition by updating the `cns-storageclass.yml` file from [Module 3](module-3-using-cns.md#storageclass-setup) like below:

<kbd>cns-storageclass.yml:</kbd>
```yaml hl_lines="15"
apiVersion: storage.k8s.io/v1beta1
kind: StorageClass
metadata:
  name: container-native-storage
  annotations:
    storageclass.beta.kubernetes.io/is-default-class: "true"
provisioner: kubernetes.io/glusterfs
parameters:
  resturl: "http://heketi-container-native-storage.cloudapps.34.252.58.209.nip.io"
  restauthenabled: "true"
  restuser: "admin"
  volumetype: "replicate:3"
  secretNamespace: "default"
  secretName: "cns-secret"
  clusterid: "fb67f97166c58f161b85201e1fd9b8ed"
```

Note the additional `clusterid` parameter highlighted. It's the cluster's UUID as known by heketi. Don't change anything else.

&#8680; Delete the existing *StorageClass* definition in OpenShift

    oc delete storageclass/container-native-storage

&#8680; Add the *StorageClass* again:

    oc create -f cns-storageclass.yml

&#8680; Create a new file called cns-storageclass-slow.yml with the following contents:

<kbd>cns-storageclass-slow.yml:</kbd>
```yaml hl_lines="13"
apiVersion: storage.k8s.io/v1beta1
kind: StorageClass
metadata:
  name: container-native-storage-slow
provisioner: kubernetes.io/glusterfs
parameters:
  resturl: "http://heketi-container-native-storage.cloudapps.34.252.58.209.nip.io"
  restauthenabled: "true"
  restuser: "admin"
  volumetype: "replicate:3"
  secretNamespace: "default"
  secretName: "cns-secret"
  clusterid: "46b205a4298c625c4bca2206b7a82dd3"
```

Again note the `clusterid` in the `parameters` section referencing the second cluster's UUID

&#8680; Add the new *StorageClass*:

    oc create -f cns-storageclass-slow.yml

Let's verify both *StorageClasses* are working as expected:

&#8680; Create the following two files containing PVCs issued against either of both GlusterFS pools via their respective *StorageClass*:

<kbd>cns-pvc-fast.yml:</kbd>
```yaml
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: my-container-storage-fast
  annotations:
    volume.beta.kubernetes.io/storage-class: container-native-storage
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 5Gi
```

<kbd>cns-pvc-slow.yml:</kbd>
```yaml
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: my-container-storage-slow
  annotations:
    volume.beta.kubernetes.io/storage-class: container-native-storage-slow
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 7Gi
```

&#8680; Create both PVCs:

    oc create -f cns-pvc-fast.yml
    oc create -f cns-pvc-slow.yml

&#8680; They should both be in bound state after a couple of seconds:

    NAME                        STATUS    VOLUME                                     CAPACITY   ACCESSMODES   AGE
    my-container-storage        Bound     pvc-3e249fdd-4ec0-11e7-970e-0a9938370404   5Gi        RWO           32s
    my-container-storage-slow   Bound     pvc-405083e6-4ec0-11e7-970e-0a9938370404   7Gi        RWO           29s

&#8680; If you check again one of the pods of the second cluster...

    oc rsh glusterfs-1nvtj

&#8680; ...you will see a new volume has been created

    sh-4.2# gluster volume list
    vol_0d976f357a7979060a4c32284ca8e136


This is how you use multiple, parallel GlusterFS pools/clusters on a single OpenShift cluster with a single heketi instance. Whereas the first pool is created with `cns-deploy` subsequent pools/cluster are created with the `heketi-cli` client.

Clean up the PVCs and the second *StorageClass* in preparation for the next section.

&#8680; Delete both PVCs (and therefore their volume)

    oc delete pvc/my-container-storage-fast
    oc delete pvc/my-container-storage-slow

&#8680; Delete the second *StorageClass*

    oc delete storageclass/container-native-storage-slow

---

Deleting a GlusterFS pool
-------------------------

Since we want to re-use `node-4`, `node-5` and `node-6` for the next section we need to delete it the GlusterFS pools on top of them first.

This is a process that involves multiple steps of manipulating the heketi topology with the `heketi-cli` client.

&#8680; Make sure the client is still properly configured via environment variables:

    export HEKETI_CLI_SERVER=http://heketi-container-native-storage.cloudapps.34.252.58.209.nip.io
    export HEKETI_CLI_USER=admin
    export HEKETI_CLI_KEY=myS3cr3tpassw0rd

&#8680; First list the topology and identify the cluster we want to delete:

    heketi-cli topology info

The portions of interest are highlighted:

``` hl_lines="1 7 14 17 24 27 34"
Cluster Id: 46b205a4298c625c4bca2206b7a82dd3

    Volumes:

    Nodes:

    	Node Id: 538b860406870288af23af0fbc2cd27f
    	State: online
    	Cluster Id: 46b205a4298c625c4bca2206b7a82dd3
    	Zone: 2
    	Management Hostname: node-5.lab
    	Storage Hostname: 10.0.3.105
    	Devices:
    		Id:e481d022cea9bfb11e8a86c0dd8d3499   Name:/dev/xvdc           State:online    Size (GiB):499     Used (GiB):0       Free (GiB):499
    			Bricks:

    	Node Id: 604d2eb15a5ca510ff3fc5ecf912d3c0
    	State: online
    	Cluster Id: 46b205a4298c625c4bca2206b7a82dd3
    	Zone: 1
    	Management Hostname: node-4.lab
    	Storage Hostname: 10.0.2.104
    	Devices:
    		Id:09a25a114c53d7669235b368efd2f8d1   Name:/dev/xvdc           State:online    Size (GiB):499     Used (GiB):0       Free (GiB):499
    			Bricks:

    	Node Id: 7736bd0cb6a84540860303a6479cacb2
    	State: online
    	Cluster Id: 46b205a4298c625c4bca2206b7a82dd3
    	Zone: 3
    	Management Hostname: node-6.lab
    	Storage Hostname: 10.0.4.106
    	Devices:
    		Id:cccadb2b54dccd99f698d2ae137a22ff   Name:/dev/xvdc           State:online    Size (GiB):499     Used (GiB):0       Free (GiB):499
			Bricks:
```

The hierachical dependencies in this topology works as follows: Clusters > Nodes > Devices.
Assuming there are no volumes present these need to be deleted in reverse order.

Note that specific values highlighted above in your environment and carry out the following steps in strict order:

&#8680; Delete all devices of all nodes:

    heketi-cli device delete e481d022cea9bfb11e8a86c0dd8d349
    heketi-cli device delete 09a25a114c53d7669235b368efd2f8d1
    heketi-cli device delete cccadb2b54dccd99f698d2ae137a22ff

&#8680; Delete all nodes of the cluster in question:

    heketi-cli node delete 538b860406870288af23af0fbc2cd27f
    heketi-cli node delete 604d2eb15a5ca510ff3fc5ecf912d3c0
    heketi-cli node delete 7736bd0cb6a84540860303a6479cacb2

&#8680; Finally delete the cluster:

    heketi-cli cluster delete 46b205a4298c625c4bca2206b7a82dd3

&#8680; Confirm the cluster is gone:

    heketi-cli cluster list

&#8680; Verify the clusters new topology back to it's state from Module 3.

This deleted all heketi database entries about the cluster. However the GlusterFS pods are still running, since they are controlled directly by OpenShift instead of heketi:

&#8680; List all the pods in the `container-native-storage` namespace:

    oc get pods -o wide -n container-native-storage

The pods formerly running the second GlusterFS pool are still there:

    NAME              READY     STATUS    RESTARTS   AGE       IP           NODE
    glusterfs-1nvtj   1/1       Running   0          1h       10.0.4.106   node-6.lab
    glusterfs-5gvw8   1/1       Running   0          1h       10.0.2.104   node-4.lab
    glusterfs-5rc2g   1/1       Running   0          4h       10.0.2.101   node-1.lab
    glusterfs-b4wg1   1/1       Running   0          1h       10.0.3.105   node-5.lab
    glusterfs-jbvdk   1/1       Running   0          4h       10.0.3.102   node-2.lab
    glusterfs-rchtr   1/1       Running   0          4h       10.0.4.103   node-3.lab
    heketi-1-tn0s9    1/1       Running   0          4h       10.130.2.3   node-6.lab

The can be stopped by removing the labels OpenShift uses to determine GlusterFS pod placement for CNS.

&#8680; Remove the labels from the last 3 OpenShift nodes like so:

    oc label node/node-4.lab storagenode-
    oc label node/node-5.lab storagenode-
    oc label node/node-6.lab storagenode-

Contrary to the output of these commands the label `storagenode` is actually removed.

&#8680; Verify that all GlusterFS pods running on `node-4`, `node-5` and `node-6` are indeed terminated:

    oc get pods -o wide -n container-native-storage

!!! Note:
    It can take up to 2 minutes for the pods to terminate.

You should be back down to 3 GlusterFS pods.

    NAME              READY     STATUS    RESTARTS   AGE       IP           NODE
    glusterfs-5rc2g   1/1       Running   0          5h        10.0.2.101   node-1.lab
    glusterfs-jbvdk   1/1       Running   0          5h        10.0.3.102   node-2.lab
    glusterfs-rchtr   1/1       Running   0          5h        10.0.4.103   node-3.lab
    heketi-1-tn0s9    1/1       Running   0          5h        10.130.2.3   node-6.lab

---

Expanding a GlusterFS pool
--------------------------

Instead of creating additional GlusterFS pools in CNS on OpenShift it is also possible to expand existing pools.

This works similar to creating additional pools, with bulk-import via the topology file. Only this time with nodes added to the existing cluster structure in JSON.

Since manipulating JSON can be error-prone create a new file called `expanded-topology.json` with contents as below:

<kbd>expanded-topology.json:</kbd>

```json
{
    "clusters": [
        {
            "nodes": [
                {
                    "node": {
                        "hostnames": {
                            "manage": [
                                "node-1.lab"
                            ],
                            "storage": [
                                "10.0.2.101"
                            ]
                        },
                        "zone": 1
                    },
                    "devices": [
                        "/dev/xvdc"
                    ]
                },
                {
                    "node": {
                        "hostnames": {
                            "manage": [
                                "node-2.lab"
                            ],
                            "storage": [
                                "10.0.3.102"
                            ]
                        },
                        "zone": 2
                    },
                    "devices": [
                        "/dev/xvdc"
                    ]
                },
                {
                    "node": {
                        "hostnames": {
                            "manage": [
                                "node-3.lab"
                            ],
                            "storage": [
                                "10.0.4.103"
                            ]
                        },
                        "zone": 3
                    },
                    "devices": [
                        "/dev/xvdc"
                    ]
                },
                {
                    "node": {
                        "hostnames": {
                            "manage": [
                                "node-4.lab"
                            ],
                            "storage": [
                                "10.0.2.104"
                            ]
                        },
                        "zone": 1
                    },
                    "devices": [
                        "/dev/xvdc"
                    ]
                },
                {
                    "node": {
                        "hostnames": {
                            "manage": [
                                "node-5.lab"
                            ],
                            "storage": [
                                "10.0.3.105"
                            ]
                        },
                        "zone": 2
                    },
                    "devices": [
                        "/dev/xvdc"
                    ]
                },
                {
                    "node": {
                        "hostnames": {
                            "manage": [
                                "node-6.lab"
                            ],
                            "storage": [
                                "10.0.4.106"
                            ]
                        },
                        "zone": 3
                    },
                    "devices": [
                        "/dev/xvdc"
                    ]
                }
            ]
        }
    ]
}
```

&#8680; Again, apply the expected labels to the remaining 3 OpenShift Nodes:

    oc label node/node-4.lab storagenode=glusterfs
    oc label node/node-5.lab storagenode=glusterfs
    oc label node/node-6.lab storagenode=glusterfs

&#8680; Wait for all pods to show `1/1` in the `READY` column:

     oc get pods -o wide -n container-native-storage

This confirms all GlusterFS pods are ready to receive remote commands:

    NAME              READY     STATUS    RESTARTS   AGE       IP           NODE
    glusterfs-0lr75   1/1       Running   0          4m        10.0.4.106   node-6.lab
    glusterfs-1dxz3   1/1       Running   0          4m        10.0.3.105   node-5.lab
    glusterfs-5rc2g   1/1       Running   0          5h        10.0.2.101   node-1.lab
    glusterfs-8nrn0   1/1       Running   0          4m        10.0.2.104   node-4.lab
    glusterfs-jbvdk   1/1       Running   0          5h        10.0.3.102   node-2.lab
    glusterfs-rchtr   1/1       Running   0          5h        10.0.4.103   node-3.lab
    heketi-1-tn0s9    1/1       Running   0          5h        10.130.2.3   node-6.lab

&#8680; If you logged out of the session meanwhile re-instate the heketi-cli configuration with the environment variables;

    export HEKETI_CLI_SERVER=http://heketi-container-native-storage.cloudapps.34.252.58.209.nip.io
    export HEKETI_CLI_USER=admin
    export HEKETI_CLI_KEY=myS3cr3tpassw0rd

&#8680; Now load the new topology:

    heketi-cli topology load --json=expanded-topology.json

The output indicated that the existing cluster was expanded, rather than creating a new one:

    Found node node-1.lab on cluster fb67f97166c58f161b85201e1fd9b8ed
        Found device /dev/xvdc
    Found node node-2.lab on cluster fb67f97166c58f161b85201e1fd9b8ed
        Found device /dev/xvdc
    Found node node-3.lab on cluster fb67f97166c58f161b85201e1fd9b8ed
        Found device /dev/xvdc
    Creating node node-4.lab ... ID: 544158e53934a3d351b874b7d915e8d4
        Adding device /dev/xvdc ... OK
    Creating node node-5.lab ... ID: 645b6edd4044cb1dd828f728d1c3eb81
        Adding device /dev/xvdc ... OK
    Creating node node-6.lab ... ID: 3f39ebf3c8c82531a7ba447135742776
        Adding device /dev/xvdc ... OK


&#8680; Log on to one of the GlusterFS pods and confirm their new peers:

    oc rsh glusterfs-0lr75

&#8680; Run the peer status command inside the container remote session:

    gluster peer status

You should now have a GlusterFS consisting of 6 nodes:

---
