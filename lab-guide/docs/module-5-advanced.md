!!! Summary "Overview"
    In this module a couple of advanced topics are discussed. Note that other volume types are currently not supported in production.

CNS with other volumes types
----------------------------

The only volume type supported with CNS in production is 3-way replicated.
For demonstration purposes you however can also created distributed volumes (no replication) and dispersed volume (4:2 erasure-coding).

Currently volume type is implemented as a parameter in the *StorageClass*, not the PVC. All volumes of this *StorageClass* will be of the specified type.

&#8680; Make sure you are logged in as `operator` in the `default` namespace:

    oc login -u operator -n default

&#8680; Create a file called `cns-storageclass-dispersed.yml` with the following content:

<kbd>cns-storageclass-dispersed.yml:</kbd>

```yaml
apiVersion: storage.k8s.io/v1beta1
kind: StorageClass
metadata:
  name: container-native-storage-dispersed
provisioner: kubernetes.io/glusterfs
parameters:
  resturl: "http://heketi-container-native-storage.cloudapps.34.252.58.209.nip.io"
  restauthenabled: "true"
  restuser: "admin"
  volumetype: "disperse:4:2"
  secretNamespace: "default"
  secretName: "cns-secret"
```

This *StorageClass* called `container-native-storage-dispersed` uses the same GlusterFS pool and the same heketi credentials to create volumes, with the difference of the `volumetype` parameter. It will cause the GlusterFS Dynamic Provisioner (the driver in OpenShift calling heketi's API) to request dispersed volumes.

!!! Caution "Important"
    For a 4:2 erasure-coding volume to be healthy at minimum 6 nodes are needed.

Go ahead and try it.

Create a file called `cns-pvc-dispersed.yml` with contents below:

<kbd>cns-pvc-dispersed.yml:</kbd>

```yaml
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: my-dispersed-container-storage
  annotations:
    volume.beta.kubernetes.io/storage-class: container-native-storage-dispersed
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi
```

When you submit via `oc create -f ...` it should successfully create a PV. If you check the backing GlusterFS volume it should look like this:

```
sh-4.2# gluster volume info vol_8d6053d817c6d54452dc31f662cfa401

Volume Name: vol_8d6053d817c6d54452dc31f662cfa401
Type: Disperse
Volume ID: 53bbeed9-4104-465c-932c-fd51866650c2
Status: Started
Snapshot Count: 0
Number of Bricks: 1 x (4 + 2) = 6
Transport-type: tcp
Bricks:
Brick1: 10.0.4.103:/var/lib/heketi/mounts/vg_ad277c699260433a6581825c3a8ea473/brick_2f2ead6e4155bf9d9f22ece3907acc34/brick
Brick2: 10.0.3.105:/var/lib/heketi/mounts/vg_84b531cdc5caae5d77796c43c8b7a4e2/brick_015564bbb1f050ba0f9333f313acd038/brick
Brick3: 10.0.2.101:/var/lib/heketi/mounts/vg_ce16710b894168772e2eb233ee38954f/brick_b5465eb7d31637e11e71ecd55c9569d4/brick
Brick4: 10.0.4.106:/var/lib/heketi/mounts/vg_05c69209a9b7f811760f5ce6044d2761/brick_cfe699588b5e73f73751e34592d9bec8/brick
Brick5: 10.0.3.102:/var/lib/heketi/mounts/vg_10fb35929089146e553336a03684afa5/brick_04dee99af2ad858054c10da6292858ff/brick
Brick6: 10.0.2.104:/var/lib/heketi/mounts/vg_9c487d4fd77d9fa0ee530ccb367e9593/brick_e67015b2aa9522a10682508e23e2837d/brick
Options Reconfigured:
transport.address-family: inet
performance.readdir-ahead: on
nfs.disable: on
```

!!! Note
    For distributed volumes you would set the `volumetype` property in the *StorageClass* to `none`.

---

OpenShift Registry on CNS
-------------------------

The Registry in OpenShift is used to store all images that result of Source-to-Image deployments as well as general purpose container image storage.
It runs as one or more containers in specific Infrastructure Nodes or Master Nodes in OpenShift.

By default the registry uses a hosts local storage which makes it prone to outages. Also, multiple registry pods need shared storage.

This can be achieved with CNS simply by making the registry pods refer to a PVC in access mode *RWX* based on CNS.

!!! Caution "Important"
    This method will be disruptive. All data stored in the registry so far will become unavailable.
    Migration scenarios exist but are beyond the scope of this lab.

&#8680; Make sure you are logged in as `operator` in the `default` namespace:

    oc login -u operator -n default

&#8680; Create a PVC for shared storage like the file `cns-registry-pvc.yml` below:

<kbd>cns-registry-pvc.yml:</kbd>
```yaml
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: registry-storage
  annotations:
    volume.beta.kubernetes.io/storage-class: container-native-storage
spec:
  accessModes:
  - ReadWriteMany
  resources:
    requests:
      storage: 20Gi
```

Create the PVC and ensure it's *BOUND*

    oc create -f cns-registry-pvc.yml

In your environment a registry is already running. This will be the case for most environments. So the existing registry configuration needs to be adjusted to include a PVC and make the pods mount it's volume.
This is done by modifying the `DeploymentConfig` of the registry.

&#8680; Update the registry's `DeploymentConfig` to refer to the PVC created before:

    oc volume deploymentconfigs/docker-registry --add --name=registry-storage -t pvc  --claim-name=registry-storage --overwrite

The registry will now redeploy.

&#8680; Observe the registry deployment get updated:

    oc logs -f deploymentconfig/docker-registry

After a couple of seconds a new deployment of the registry should be running:

&#8680; Verify a new version of the registry's `DeploymentConfig` is running:

    oc get deploymentconfig/docker-registry

It should say:

    NAME              REVISION   DESIRED   CURRENT   TRIGGERED BY
    docker-registry   2          1         1         config

With this the OpenShift Registry is based on persistent storage provided by CNS. Since this is shared storage this also allows to scale out the registry pods.

---
