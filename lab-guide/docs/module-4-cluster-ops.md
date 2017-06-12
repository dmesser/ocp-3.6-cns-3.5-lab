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

!!! Note:
    It may take up to 3 minutes for the GlusterFS pods to transition into `READY` state.

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

Before you can use the second cluster two tasks have to be accomplished:

1. The *StorageClass* for the first cluster has to be updated to point the first cluster's UUID,

1. A second *StorageClass* for the second cluster has to be created, pointing to the same heketi

!!! Tip "Why do we need to update the first *StorageClass*?"
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

They should both be in bound state after a couple of seconds:

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

They can be stopped by removing the labels OpenShift uses to determine GlusterFS pod placement for CNS.

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

!!! Note:
    It may take up to 3 minutes for the GlusterFS pods to transition into `READY` state.

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

    Number of Peers: 5

    Hostname: 10.0.3.102
    Uuid: c6a6d571-fd9b-4bd8-aade-e480ec2f8eed
    State: Peer in Cluster (Connected)

    Hostname: 10.0.4.103
    Uuid: 46044d06-a928-49c6-8427-a7ab37268fed
    State: Peer in Cluster (Connected)

    Hostname: 10.0.2.104
    Uuid: 62abb8b9-7a68-4658-ac84-8098a1460703
    State: Peer in Cluster (Connected)

    Hostname: 10.0.3.105
    Uuid: 5b44b6ea-6fb5-4ea9-a6f7-328179dc6dda
    State: Peer in Cluster (Connected)

    Hostname: 10.0.4.106
    Uuid: ed39ecf7-1f5c-4934-a89d-ee1dda9a8f98
    State: Peer in Cluster (Connected)


With this you have expanded the existing pool. New PVCs will start to use capacity from the additional nodes.

!!! Important
    You now have a GlusterFS pool with mixed media types (both size and speed). It is recommended to have the same media type per pool.
    If you like to offer multiple media types for CNS in OpenShift, use separate pools and separate `StorageClass` objects as described in the [previous section](#running-multiple-glusterfs-pools).

---

Adding a device to a node
-------------------------

Instead of adding entirely new nodes you can also add new storage devices for CNS to use on existing nodes.

It is again possible to do this by loading an updated topology file. Alternatively to bulk-loading via JSON you are also able to do this directly with the `heketi-cli` utility. This also applies to the previous sections in this module.

For this purpose `node-6.lab` has an additional, so far unused block device `/dev/xvdd`.

&#8680; To use the heketi-cli make sure the environment variables are still set:

    export HEKETI_CLI_SERVER=http://heketi-container-native-storage.cloudapps.34.252.58.209.nip.io
    export HEKETI_CLI_USER=admin
    export HEKETI_CLI_KEY=myS3cr3tpassw0rd

&#8680; Determine the UUUI heketi uses to identify `node-6.lab` in it's database:

    heketi-cli topology info | grep -B4 node-6.lab

**Note the UUID**:

``` hl_lines="1"
Node Id: ae03d2c5de5cdbd44ba27ff5320d3438
State: online
Cluster Id: eb909a08c8e8fd0bf80499fbbb8a8545
Zone: 3
Management Hostname: node-6.lab
```

&#8680; Query the node's available devices:

    heketi-cli node info ae03d2c5de5cdbd44ba27ff5320d343

The node has one device available:

    Node Id: ae03d2c5de5cdbd44ba27ff5320d3438
    State: online
    Cluster Id: eb909a08c8e8fd0bf80499fbbb8a8545
    Zone: 3
    Management Hostname: node-6.lab
    Storage Hostname: 10.0.4.106
    Devices:
    Id:cc594d7f5ce59ab2a991c70572a0852f   Name:/dev/xvdc           State:online    Size (GiB):499     Used (GiB):0       Free (GiB):499

&#8680; Add the device `/dev/xvdd` to the node using the UUID noted earlier.

    heketi-cli device add --node=ae03d2c5de5cdbd44ba27ff5320d3438 --name=/dev/xvdd

The device is registered in heketi's database.

    Device added successfully

&#8680; Query the node's available devices again and you'll see a second device.

    heketi-cli node info ae03d2c5de5cdbd44ba27ff5320d343

    Node Id: ae03d2c5de5cdbd44ba27ff5320d3438
    State: online
    Cluster Id: eb909a08c8e8fd0bf80499fbbb8a8545
    Zone: 3
    Management Hostname: node-6.lab
    Storage Hostname: 10.0.4.106
    Devices:
    Id:62cbae7a3f6faac38a551a614419cca3   Name:/dev/xvdd           State:online    Size (GiB):499     Used (GiB):0       Free (GiB):499
    Id:cc594d7f5ce59ab2a991c70572a0852f   Name:/dev/xvdc           State:online    Size (GiB):499     Used (GiB):0       Free (GiB):499

---

Replacing a failed device
-------------------------

One of heketi's advantages is the automation of otherwise tedious manual tasks, like replacing a faulty brick in GlusterFS to repair degraded volumes.
We will simulate this use case now.

&#8680; For convenience you can remain operator for now:

    [ec2-user@master ~]$ oc whoami
    operator

&#8680; Use any available project to submit some PVCs.

    oc project playground

&#8680; Create the file `cns-large-pvc.yml` with content below:

<kbd>cns-large-pvc.yml:</kbd>

```yaml
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: my-large-container-store
  annotations:
    volume.beta.kubernetes.io/storage-class: container-native-storage
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 200Gi
```

&#8680; Create this request for a large volume:

    oc create -f cns-large-pvc.yml

The requested capacity in this PVC is larger than any single brick on nodes `node-1.lab`, `node-2.lab` and `node-3.lab` so it will be created from the bricks of the other 3 nodes which have larger bricks (500 GiB).

Where are now going to determine a PVCs physical backing device on CNS. This is done with the following relationships between the various entities of GlusterFS, heketi and OpenShift in mind:

PVC -> PV -> heketi volume -> GlusterFS volume -> GlusterFS brick -> Physical Device

&#8680; Get the PV

    oc describe pvc/my-large-container-store

Note the PVs name:
``` hl_lines="5"
    Name:		my-large-container-store
    Namespace:	container-native-storage
    StorageClass:	container-native-storage
    Status:		Bound
    Volume:		pvc-078a1698-4f5b-11e7-ac96-1221f6b873f8
    Labels:		<none>
    Capacity:	200Gi
    Access Modes:	RWO
    No events.
```

&#8680; Get the GlusterFS volume name of this PV

    oc describe pv/pvc-078a1698-4f5b-11e7-ac96-1221f6b873f8

The GlusterFS volume name as it used by GlusterFS:

``` hl_lines="13"
    Name:		pvc-078a1698-4f5b-11e7-ac96-1221f6b873f8
    Labels:		<none>
    StorageClass:	container-native-storage
    Status:		Bound
    Claim:		container-native-storage/my-large-container-store
    Reclaim Policy:	Delete
    Access Modes:	RWO
    Capacity:	200Gi
    Message:
    Source:
        Type:		Glusterfs (a Glusterfs mount on the host that shares a pod's lifetime)
        EndpointsName:	glusterfs-dynamic-my-large-container-store
        Path:		vol_3ff9946ddafaabe9745f184e4235d4e1
        ReadOnly:		false
    No events.
```

&#8680; Get the volumes topology directly from GlusterFS

    oc rsh glusterfs-52hkc

    gluster volume info vol_3ff9946ddafaabe9745f184e4235d4e1

The output indicates this volume is indeed backed by, among others, `node-6.lab`

``` hl_lines="11"
Volume Name: vol_3ff9946ddafaabe9745f184e4235d4e1
Type: Replicate
Volume ID: 774ae26f-bd3f-4c06-990b-57012cc5974b
Status: Started
Snapshot Count: 0
Number of Bricks: 1 x 3 = 3
Transport-type: tcp
Bricks:
Brick1: 10.0.3.105:/var/lib/heketi/mounts/vg_e1b93823a2906c6758aeec13930a0919/brick_b3d5867d2f86ac93fce6967128643f85/brick
Brick2: 10.0.2.104:/var/lib/heketi/mounts/vg_3c3489a5779c1c840a82a26e0117a415/brick_6323bd816f17c8347b3a68e432501e96/brick
Brick3: 10.0.4.106:/var/lib/heketi/mounts/vg_62cbae7a3f6faac38a551a614419cca3/brick_a6c92b6a07983e9b8386871f5b82497f/brick
Options Reconfigured:
transport.address-family: inet
performance.readdir-ahead: on
nfs.disable: on
```

&#8680; Using the full path of `Brick3` we cross-check with heketi's topology on which device it is based on:

    heketi-cli topology info | grep -B2 '/var/lib/heketi/mounts/vg_62cbae7a3f6faac38a551a614419cca3/brick_a6c92b6a07983e9b8386871f5b82497f/brick'

Among other results grep will show the physical backing device of this brick's mount path:

``` hl_lines="1"
Id:62cbae7a3f6faac38a551a614419cca3   Name:/dev/xvdd           State:online    Size (GiB):499     Used (GiB):201     Free (GiB):298
			Bricks:
				Id:a6c92b6a07983e9b8386871f5b82497f   Size (GiB):200     Path: /var/lib/heketi/mounts/vg_62cbae7a3f6faac38a551a614419cca3/brick_a6c92b6a07983e9b8386871f5b82497f/brick
```

In this case it's `/dev/vdd` of `node-6.lab`.

!!! Note:
    The device might be different for you. This is subject to heketi's dynamic scheduling.

Let's assume `/dev/vdd` on `node-6.lab` has failed and needs to be replaced.

In such a case you'll take the device's ID and go through the following steps:

&#8680; First, disable the device in heketi

    heketi-cli device disable 62cbae7a3f6faac38a551a614419cca3

This will take the device offline and exclude it from future volume creation requests.

&#8680; Now delete the device in heketi

    heketi-cli device delete 62cbae7a3f6faac38a551a614419cca3

This will trigger a brick-replacement in GlusterFS. heketi will transparently create new bricks for each brick on the device to be deleted. The replacement operation will be conducted with the new bricks replacing all bricks on the device to be deleted.
The new bricks, if possible, will automatically be created in zones different from the remaining bricks to maintain equal balancing and cross-zone availability.

&#8680; Check again the volumes topology directly from GlusterFS

    oc rsh glusterfs-52hkc

    gluster volume info vol_3ff9946ddafaabe9745f184e4235d4e1

You will notice that `Brick3` is now a different mount path, but on the same node.

If you cross-check again the new bricks mount path with the heketi topology you will see it's indeed coming from a different device. The remaining device in `node-6.lab`, in this case `/dev/vdc`

!!! Note
    Device removal while maintaining volume health is possible in heketi as well. Simply delete all devices of the node in question as discussed above. Then the device can be deleted from heketi with `heketi-cli device delete <device-uuid>`

---
