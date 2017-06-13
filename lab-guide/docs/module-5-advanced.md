!!! Summary "Overview"
    In this module a couple of advanced topics are discussed. Some of them are in *Technology Preview* state at the moment and not supported in production.

CNS with other volumes types
----------------------------

The only volume type supported with CNS in production is 3-way replicated.
For demonstration purposes you however can also created distributed volumes (no replication) and dispersed volume (4:2 erasure-coding).

Currently volume type is implemented as a parameter in the *StorageClass*, not the PVC. All volumes of this *StorageClass* will be of the specified type.

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
  volumetype: "dispersed:4:2"
  secretNamespace: "default"
  secretName: "cns-secret"
```

This *StorageClass* called `container-native-storage-dispersed` uses the same GlusterFS pool and the same heketi credentials to create volumes, with the difference of the `volumetype` parameter. It will cause the GlusterFS Dynamic Provisioner (the driver in OpenShift calling heketi's API) to request dispersed volumes.

!!! Caution "Important"
    For a 4:2 erasure-coding volume to be healthy at minimum 6 nodes are needed.

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



The registry will now redeploy.

&#8680; Observe the registry deployment get updated:


With this the OpenShift Registry is based on persistent storage provided by CNS. Since this is shared storage this also allows to scale out the registry pods. 


---

Running heketi on the Master
----------------------------

In this exercise, and by default, the heketi pod runs on OpenShift Application Nodes, co-located with the GlusterFS pods and application pods.

In production deployments facilities like the OpenShift Registry or the OpenShift Router are running in pods on special nodes called `Infrastructure Nodes`. It is a good practice to do the same with heketi.

&#8680; Make sure you are logged in as `operator` in the `default` namespace:

    oc login -u operator -n container-native-storage

&#8680; Display all pods in this namespace

    oc get pods -o wide

You will notice the heketi pod running among the GlusterFS pods:

&#8680; Modify the registry's `DeploymentConfig` as follows:


The `nodeSelector` is basically a search predicate applied when making scheduling decision for pods among nodes.
In this environment the master node also acts as an infra node and therefore carries the label "region=infra".

By modifying the `DeploymentConfig` the heketi pod will be recreated and scheduled on the Master. It's data is persisted in a BoltDB instance hosted outside of the pod on a GlusterFS volume from the first cluster in CNS.
