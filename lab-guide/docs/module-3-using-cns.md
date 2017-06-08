Creating a StorageClass
------------------------

OpenShift uses Kubernetes' PersistentStorage facility to dynamically allocate storage for applications. This is a fairly simple framework in which only 3 components exists: the storage provider, the storage volume and the request for a storage volume.

![OpenShift Storage Lifecycle](cns_diagram_pvc.svg)

OpenShift knows non-ephemeral storage as "persistent" volumes. This is storage that is decoupled from pod lifecycles. Users can request such storage by submitting a **PersistentVolumeClaim** to the system, which carries aspects like desired capacity or access mode (shared, single, read-only).+ A storage provider in the system is represented by a **StorageClass** and is referenced in the claim. Upon receiving the claim it talks to the API of the actual storage system to provision the storage.  
The storage is represented in OpenShift as a **PersistentVolume** which can directly be used by pods to mount it.

With these basics defined we can configure our system for CNS. First we will set up the credentials for CNS in OpenShift.

Create an encoded value for the CNS admin user like below:

    [root@master ~]# echo -n "myS3cr3tpassw0rd" | base64
    bXlTM2NyM3RwYXNzdzByZA==

We will store this encoded value in an OpenShift secret. Create a file called `cns-secret.yml` as per below:

**cns-secret.yml.**

    apiVersion: v1
    kind: Secret
    metadata:
      name: cns-secret
      namespace: default
    data:
      key: bXlTM2NyM3RwYXNzdzByZA==
    type: kubernetes.io/glusterfs

-   *key* contains base64-encoded version of *myS3cr3tpassw0rd*

Create the secret in OpenShift with the following command:

    [root@master ~]# oc create -f cns-secret.yml

To represent CNS as a storage provider in the system you first have to create a StorageClass. Define by creating a file called `cns-storageclass.yml` which references the secret and the heketi URL shown earlier with the contents as below:

**cns-storageclass.yml.**

    apiVersion: storage.k8s.io/v1beta1
    kind: StorageClass
    metadata:
      name: container-native-storage
      annotations:
        storageclass.beta.kubernetes.io/is-default-class: "true"
    provisioner: kubernetes.io/glusterfs
    parameters:
      resturl: "http://heketi-container-native-storage.cloudapps.example.com"
      restauthenabled: "true"
      restuser: "admin"
      volumetype: "replicate:3"
      secretNamespace: "default"
      secretName: "cns-secret"

Create the StorageClass in OpenShift with the following command:

    [root@master ~]# oc create -f cns-storageclass.yml

With these components in place the system is ready to dynamically provision storage capacity from Container-native Storage.

### Requesting Storage

To get storage provisioned as a user you have to "claim" storage. The *PersistentVolumeClaim* (PVC) basically acts a request to the system to provision storage with certain properties, like a specific capacity.  
Also the access mode is set here, where *ReadWriteOnce* allows one container at a time to mount this storage.

Create a claim by specifying a file called `cns-pvc.yml` with the following contents:

**cns-pvc.yml.**

    kind: PersistentVolumeClaim
    apiVersion: v1
    metadata:
      name: my-container-storage
      annotations:
        volume.beta.kubernetes.io/storage-class: container-native-storage
    spec:
      accessModes:
      - ReadWriteOnce
      resources:
        requests:
          storage: 10Gi

With above PVC we are requesting 10 GiB of non-shared storage. Instead of *ReadWriteOnce* you could also have specified *ReadWriteOnly* (for read-only) and *ReadWriteMany* (for shared storage).

Submit the PVC to the system like so:

    [root@master ~]# oc create -f cns-pvc.yml
    persistentvolumeclaim "my-container-storage" created

Look at the requests state with the following command:

    [root@master ~]# oc get pvc
    NAME                   STATUS    VOLUME                                     CAPACITY   ACCESSMODES   AGE
    my-container-storage   Bound     pvc-382ac13d-4a9f-11e7-b56f-2cc2602a6dc8   10Gi       RWO           16s

> **Note**
>
> It may take up to 15 seconds for the claim to be in **bound**.

> **Caution**
>
> If the PVC is stuck in *PENDING* state you will need to investigate. Run `oc describe pvc/my-container-storage` to see a more detailed explanation. Typically there are two root causes - the StorageClass is not properly setup (wrong name, wrong credentials, incorrect secret name, wrong heketi URL, heketi service not up, heketi pod not up…) or the PVC is malformed (wrong StorageClass, name already taken …)

> **Tip**
>
> You can also do this step with the UI. If you like you can switch to an arbitrary project you have access to and go to the "Storage" tab. Select "Create" storage and make selections accordingly to the PVC described before.

When the claim was fulfilled successfully it is in the **Bound** state. That means the system has successfully (via the StorageClass) reached out to the storage backend (in our case GlusterFS). The backend in turn provisioned the storage and provided a handle back OpenShift. In OpenShift the provisioned storage is then represented by a *PersistentVolume* (PV) which is *bound* to the PVC.  
Look at the PVC for these details:

    [root@master ~]# oc describe pvc/my-container-storage
    Name:           my-container-storage
    Namespace:      container-native-storage
    StorageClass:   container-native-storage
    Status:         Bound
    Volume:         pvc-382ac13d-4a9f-11e7-b56f-2cc2602a6dc8
    Labels:         <none>
    Capacity:       10Gi
    Access Modes:   RWO
    No events.

-   The StorageClass against which the PVC was submitted.

-   The name of PV that has been created.

> **Note**
>
> The PV name will be different in your environment since it’s automatically generated.

Let’s look at the corresponding PV by it’s name:

    [root@master ~]# oc describe pv/pvc-382ac13d-4a9f-11e7-b56f-2cc2602a6dc8
    Name:           pvc-382ac13d-4a9f-11e7-b56f-2cc2602a6dc8
    Labels:         <none>
    StorageClass:   container-native-storage
    Status:         Bound
    Claim:          container-native-storage/my-container-storage
    Reclaim Policy: Delete
    Access Modes:   RWO
    Capacity:       10Gi
    Message:
    Source:
        Type:               Glusterfs (a Glusterfs mount on the host that shares a pod's lifetime)
        EndpointsName:      glusterfs-dynamic-my-container-storage
        Path:               vol_304670f0d50bf5aa4717a69652bd48ff
        ReadOnly:           false
    No events.

-   The StorageClass which provisioned this PV.

-   The claim that initiated the provisioning.

-   What happens to the storage when the PV object is deleted: here it’s deleted as well.

-   The desired access mode. RWO = ReadWriteOnce.

-   The capacity of the provisioned storage.

-   The type of storage: in our case GlusterFS as part of CNS.

> **Tip**
>
> Note that in earlier documentation you may also find references about administrators actually **pre-provisioning** PVs. Later PVCs would "pick up" a suitable PV by looking at it’s capacity. This was needed for storage like NFS that does not have an API and therefore does not support **dynamic provisioning**.  
> This kind of storage should not be used anymore as it requires manual intervention, risky capacity planning and incurs inefficient storage utilization.

Let’s release this storage capacity again, since it’s in the wrong namespace anyway.  
Storage is freed up by deleting the **PVC**. The PVC controls the lifecycle of the storage, not the PV.

> **Important**
>
> Never delete PVs that are dynamically provided. They are only handles for pods mounting the storage. Storage lifecycle is entirely controlled via PVCs.

Delete the storage by deleting the PVC like this:

    [root@master ~]# oc delete pvc/my-container-storage

### Using non-shared storage for databases

Normally a user doesn’t request storage with a PVC directly. Rather the PVC is integrated in a larger template that describe the entire application. Such examples ship with OpenShift out of the box.

> **Tip**
>
> The following steps can again also be done with the UI. For this purpose follow these steps:

1.  Log on to a project you have access to and quota available

2.  next to the project’s name select *Add to project*

3.  In the *Browse Catalog* view select *Ruby* from the list of programming languages

4.  Select the example app entitled *Rails + PostgreSQL (Persistent)*

5.  Optionally change the *Volume Capacity* parameter to something greater than 1GiB, e.g. 15 GiB

6.  Select *Create* to start deploying the app

7.  Select *Continue to Overview* in the confirmation screen

8.  Back on the overview page select the deploymentconfig *postgresql*

9.  On the following page select *Actions* &gt; *Edit Health Checks*

10. In the settings menu change the *Initial Delay* values for both *Readiness Probe* and *Liveliness Probe* to 180 seconds

Log on to the system as `marina` und create a project with an arbitrary name.

    [root@master ~]# oc login -u marina --insecure-skip-tls-verify --server=https://master.example.com:8443
    [root@master ~]# oc new-project my-test-project

To use some of the examples that ship with OpenShift enter the following command to export the template for a sample Ruby on Rails with PostgreSQL application:

    [root@master ~]# oc export template/rails-pgsql-persistent -n openshift -o yaml > rails-app-template.yml

In the file `rails-app-template.yml` you can now review the template for this entire application stack in all it’s glory. In essence it creates Rails Application instance which mimics a very basic blogging application. The articles are saved in a PostgreSQL database which runs in another pod. In addition a PVC is issued (line 194) to supply this pod with persistent storage below the mount point /var/lib/pgsql/data (line 275).

We need to modify this template now. Open it in your favorite editor and increase the values for `initialDelaySeconds` in both sections (`livenessProbe` and `readinessProbe`), around lines 255 - 270:

**rails-app-template.yml.**

    [...omitted...]

              livenessProbe:
                initialDelaySeconds: 180
                tcpSocket:
                  port: 5432
                timeoutSeconds: 1
              name: postgresql
              ports:
              - containerPort: 5432
              readinessProbe:
                exec:
                  command:
                  - /bin/sh
                  - -i
                  - -c
                  - psql -h 127.0.0.1 -U ${POSTGRESQL_USER} -q -d ${POSTGRESQL_DATABASE}
                    -c 'SELECT 1'
                initialDelaySeconds: 180
                timeoutSeconds: 1
              resources:

    [...omitted...]

-   Set the *initialDelaySeconds* value to 180 in both the livenessProbe and readinessProbe section

> **Important**
>
> In production you don’t have to change these values. Your test environment however is using nested virtualization and therefore has much lower performance than a production environment in the cloud or on-premise. Therefore the postgres container takes longer to initialize and would be declared unhealthy by OpenShift with the default delays when checking the container health.

Next we are going to create all the resources from the templates while passing in an additional parameter to override the default storage capacity requested from the PVC.

> **Tip**
>
> To list all available parameters from this template run `oc process -f rails-app-template.yml --parameters`

The parameter in the template is called `VOLUME_CAPACITY`. We will process the template with the CLI client and override this parameter with a value of *15Gi* as follows:

    [root@master ~]# oc process -f rails-app-template.yml -o yaml -p VOLUME_CAPACITY=15Gi > my-rails-app.yml

The `oc process` command parses the template and replaces any parameters with their default values if not supplied explicitly like we did for the volume capacity.

The result `my-rails-app.yml` file contains all resources for this application ready to deploy, like so:

    [root@master ~]# oc create -f my-rails-app.yml
    secret "rails-pgsql-persistent" created
    service "rails-pgsql-persistent" created
    route "rails-pgsql-persistent" created
    imagestream "rails-pgsql-persistent" created
    buildconfig "rails-pgsql-persistent" created
    deploymentconfig "rails-pgsql-persistent" created
    persistentvolumeclaim "postgresql" created
    service "postgresql" created
    deploymentconfig "postgresql" created

You can now use the OpenShift UI (while being logged in as *marina* in the newly created project) to follow the deployment process. Alternatively watch the containers deploy like this:

    [root@master ~]# oc get pods -w
    NAME                             READY     STATUS              RESTARTS   AGE
    postgresql-1-deploy              0/1       ContainerCreating   0          11s
    rails-pgsql-persistent-1-build   0/1       ContainerCreating   0          11s
    NAME                  READY     STATUS    RESTARTS   AGE
    postgresql-1-deploy   1/1       Running   0          14s
    postgresql-1-81gnm   0/1       Pending   0         0s
    postgresql-1-81gnm   0/1       Pending   0         0s
    rails-pgsql-persistent-1-build   1/1       Running   0         19s
    postgresql-1-81gnm   0/1       Pending   0         15s
    postgresql-1-81gnm   0/1       ContainerCreating   0         16s
    postgresql-1-81gnm   0/1       Running   0         47s
    postgresql-1-81gnm   1/1       Running   0         4m
    postgresql-1-deploy   0/1       Completed   0         4m
    postgresql-1-deploy   0/1       Terminating   0         4m
    postgresql-1-deploy   0/1       Terminating   0         4m
    rails-pgsql-persistent-1-deploy   0/1       Pending   0         0s
    rails-pgsql-persistent-1-deploy   0/1       Pending   0         0s
    rails-pgsql-persistent-1-deploy   0/1       ContainerCreating   0         0s
    rails-pgsql-persistent-1-build   0/1       Completed   0         11m
    rails-pgsql-persistent-1-deploy   1/1       Running   0         6s
    rails-pgsql-persistent-1-hook-pre   0/1       Pending   0         0s
    rails-pgsql-persistent-1-hook-pre   0/1       Pending   0         0s
    rails-pgsql-persistent-1-hook-pre   0/1       ContainerCreating   0         0s
    rails-pgsql-persistent-1-hook-pre   1/1       Running   0         6s
    rails-pgsql-persistent-1-hook-pre   0/1       Completed   0         15s
    rails-pgsql-persistent-1-dkj7w   0/1       Pending   0         0s
    rails-pgsql-persistent-1-dkj7w   0/1       Pending   0         0s
    rails-pgsql-persistent-1-dkj7w   0/1       ContainerCreating   0         0s
    rails-pgsql-persistent-1-dkj7w   0/1       Running   0         1m
    rails-pgsql-persistent-1-dkj7w   1/1       Running   0         1m
    rails-pgsql-persistent-1-deploy   0/1       Completed   0         1m
    rails-pgsql-persistent-1-deploy   0/1       Terminating   0         1m
    rails-pgsql-persistent-1-deploy   0/1       Terminating   0         1m
    rails-pgsql-persistent-1-hook-pre   0/1       Terminating   0         1m
    rails-pgsql-persistent-1-hook-pre   0/1       Terminating   0         1m

Exit out of the watch mode with kbd:\[Ctrl + c\]

> **Note**
>
> It may take up to 10 minutes for the deployment to complete.

You should also see a PVC being issued and in the *Bound* state.

    [root@master ~]# oc get pvc
    NAME         STATUS    VOLUME                                     CAPACITY   ACCESSMODES   AGE
    postgresql   Bound     pvc-9bb84d88-4ac6-11e7-b56f-2cc2602a6dc8   15Gi       RWO           4m

> **Tip**
>
> Why did this even work? If you paid close attention you likely noticed that the PVC in the template does not specify a particular *StorageClass*. This still yields a PV deployed because our *StorageClass* has actually been defined as the system-wide default.

Now go ahead and try out the application. The overview page in the OpenShift UI will tell you the `route` which has been deployed as well. Otherwise get it on the CLI like this:

    [root@master ~]# oc get route
    NAME                     HOST/PORT                                                      PATH      SERVICES                 PORT      TERMINATION   WILDCARD
    rails-pgsql-persistent   rails-pgsql-persistent-my-test-project.cloudapps.example.com             rails-pgsql-persistent   <all>                   None

Following this output, point your browser to <http://rails-pgsql-persistent-my-test-project.cloudapps.example.com/articles>.  
The username/password to create articles and comments is by default *openshift*/*secret*.

You should be able to successfully create articles and comments. When they are saved they are actually saved in the PostgreSQL database which stores it’s table spaces on a GlusterFS volume provided by CNS.

Now let’s take a look at how this was actually achieved. First you need to acquire necessary permissions:

    [root@master ~]# oc login -u system:admin

Select the example project of the user `marina` if not already/still selected:

    [root@master ~]# oc project my-test-project

Look at the PVC to determine the PV:

    [root@master ~]# oc get pvc
    NAME         STATUS    VOLUME                                     CAPACITY   ACCESSMODES   AGE
    postgresql   Bound     pvc-9bb84d88-4ac6-11e7-b56f-2cc2602a6dc8   15Gi       RWO           17m

> **Note**
>
> Your PV name will be different as it’s dynamically generated.

Look at the details of this PV:

    [root@master ~]# oc describe pv/pvc-9bb84d88-4ac6-11e7-b56f-2cc2602a6dc8
    Name:           pvc-9bb84d88-4ac6-11e7-b56f-2cc2602a6dc8
    Labels:         <none>
    StorageClass:   container-native-storage
    Status:         Bound
    Claim:          my-test-project/postgresql
    Reclaim Policy: Delete
    Access Modes:   RWO
    Capacity:       15Gi
    Message:
    Source:
        Type:               Glusterfs (a Glusterfs mount on the host that shares a pod's lifetime)
        EndpointsName:      glusterfs-dynamic-postgresql
        Path:               vol_e8fe7f46fedf7af7628feda0dcbf2f60
        ReadOnly:           false
    No events.

-   The unique name of this PV in the system OpenShift refers to

-   The unique volume name backing the PV known to GlusterFS

Note the GlusterFS volume name, in this case **vol\_e8fe7f46fedf7af7628feda0dcbf2f60**.

Now let’s switch to the namespace we used for CNS deployment:

    [root@master ~]# oc project container-native-storage

Look at the GlusterFS pods running and pick one (which one is not important):

    [root@master ~]# oc get pods -o wide
    NAME              READY     STATUS    RESTARTS   AGE       IP              NODE
    glusterfs-37vn8   1/1       Running   1          15m       192.168.0.102   node1.example.com
    glusterfs-cq68l   1/1       Running   1          15m       192.168.0.103   node2.example.com
    glusterfs-m9fvl   1/1       Running   1          15m       192.168.0.104   node3.example.com
    heketi-1-cd032    1/1       Running   1          13m       10.130.0.5      node3.example.com

Remember the IP address of the pod you select. Log on to GlusterFS pod with a remote terminal session like so:

    [root@master ~]# oc rsh glusterfs-37vn8
    sh-4.2#

You have now access to this container’s namespace which has the GlusterFS CLI utilities installed.  
Let’s list all known volumes:

    sh-4.2# gluster volume list
    heketidbstorage
    vol_e8fe7f46fedf7af7628feda0dcbf2f60

-   A special volume dedicated to heketi’s internal database.

-   The volume backing the PV of the PostgreSQL database deployed earlier.

Interrogate GlusterFS about the topology of this volume:

    sh-4.2# gluster volume info vol_e8fe7f46fedf7af7628feda0dcbf2f60

    Volume Name: vol_e8fe7f46fedf7af7628feda0dcbf2f60
    Type: Replicate
    Volume ID: c2bedd16-8b0d-432c-b9eb-4ab1274826dd
    Status: Started
    Snapshot Count: 0
    Number of Bricks: 1 x 3 = 3
    Transport-type: tcp
    Bricks:
    Brick1: 192.168.0.103:/var/lib/heketi/mounts/vg_63b05bee6695ee5a63ad95bfbce43bf7/brick_aa28de668c8c21192df55956a822bd3c/brick
    Brick2: 192.168.0.102:/var/lib/heketi/mounts/vg_0246fd563709384a3cbc3f3bbeeb87a9/brick_684a01f8993f241a92db02b117e0b912/brick
    Brick3: 192.168.0.104:/var/lib/heketi/mounts/vg_5a8c767e65feef7455b58d01c6936b83/brick_25972cf5ed7ea81c947c62443ccb308c/brick
    Options Reconfigured:
    transport.address-family: inet
    performance.readdir-ahead: on
    nfs.disable: on

-   According to the output of `oc get pods -o wide` this is the container we are logged on to.

> **Note**
>
> Identify the right brick by looking at the host IP of the pod you have just logged on to. `oc get pods -o wide` will give you this information.

GlusterFS created this volume as a 3-way replica set across all GlusterFS pods, in therefore across all your OpenShift App nodes running CNS.  
Each pod/node exposes his local storage via the GlusterFS protocol. This local storage is known as a **brick** in GlusterFS and is usually backed by a local SAS disk or NVMe device. The brick is simply formatted with XFS and thus made available to GlusterFS.

You can even look at this yourself:

    sh-4.2# ls -ahl /var/lib/heketi/mounts/vg_0246fd563709384a3cbc3f3bbeeb87a9/brick_684a01f8993f241a92db02b117e0b912/brick
    total 16K
    drwxrwsr-x.   5 root       2001   57 Jun  6 14:44 .
    drwxr-xr-x.   3 root       root   19 Jun  6 14:44 ..
    drw---S---. 263 root       2001 8.0K Jun  6 14:46 .glusterfs
    drwxr-sr-x.   3 root       2001   25 Jun  6 14:44 .trashcan
    drwx------.  20 1000080000 2001 8.0K Jun  6 14:46 userdata

    sh-4.2# ls -ahl /var/lib/heketi/mounts/vg_0246fd563709384a3cbc3f3bbeeb87a9/brick_684a01f8993f241a92db02b117e0b912/brick/userdata

    total 68K
    drwx------. 20 1000080000 2001 8.0K Jun  6 14:46 .
    drwxrwsr-x.  5 root       2001   57 Jun  6 14:44 ..
    -rw-------.  2 1000080000 root    4 Jun  6 14:44 PG_VERSION
    drwx------.  6 1000080000 root   54 Jun  6 14:46 base
    drwx------.  2 1000080000 root 8.0K Jun  6 14:47 global
    drwx------.  2 1000080000 root   18 Jun  6 14:44 pg_clog
    drwx------.  2 1000080000 root    6 Jun  6 14:44 pg_commit_ts
    drwx------.  2 1000080000 root    6 Jun  6 14:44 pg_dynshmem
    -rw-------.  2 1000080000 root 4.6K Jun  6 14:46 pg_hba.conf
    -rw-------.  2 1000080000 root 1.6K Jun  6 14:44 pg_ident.conf
    drwx------.  2 1000080000 root   32 Jun  6 14:46 pg_log
    drwx------.  4 1000080000 root   39 Jun  6 14:44 pg_logical
    drwx------.  4 1000080000 root   36 Jun  6 14:44 pg_multixact
    drwx------.  2 1000080000 root   18 Jun  6 14:46 pg_notify
    drwx------.  2 1000080000 root    6 Jun  6 14:44 pg_replslot
    drwx------.  2 1000080000 root    6 Jun  6 14:44 pg_serial
    drwx------.  2 1000080000 root    6 Jun  6 14:44 pg_snapshots
    drwx------.  2 1000080000 root    6 Jun  6 14:46 pg_stat
    drwx------.  2 1000080000 root   84 Jun  6 15:16 pg_stat_tmp
    drwx------.  2 1000080000 root   18 Jun  6 14:44 pg_subtrans
    drwx------.  2 1000080000 root    6 Jun  6 14:44 pg_tblspc
    drwx------.  2 1000080000 root    6 Jun  6 14:44 pg_twophase
    drwx------.  3 1000080000 root   60 Jun  6 14:44 pg_xlog
    -rw-------.  2 1000080000 root   88 Jun  6 14:44 postgresql.auto.conf
    -rw-------.  2 1000080000 root  21K Jun  6 14:46 postgresql.conf
    -rw-------.  2 1000080000 root   46 Jun  6 14:46 postmaster.opts
    -rw-------.  2 1000080000 root   89 Jun  6 14:46 postmaster.pid

> **Note**
>
> The exact path name will be different in your environment as it has been automatically generated.

You are looking at the PostgreSQL internal data file structure from the perspective of the GlusterFS server side. It’s a normal local filesystem here.

Clients, like the OpenShift nodes and their application pods talk to this storage with the GlusterFS protocol. Which abstracts the 3-way replication behind a single FUSE mount point.  
When a pod starts that mounts storage from a PV backed by GlusterFS OpenShift will mount the GlusterFS volume on the App Node and then *bind-mount* this directory to the right pod.  
This is happen transparently to the application inside the pod and looks like a normal local filesystem.

You may exit your remote session to the GlusterFS pod.

    sh-4.2# exit

### Providing shared storage to multiple application instances

So far only very few options, like the basic NFS support existed, to provide a PersistentVolume to more than one container at once. The access mode used for this is **ReadWriteMany**.

With CNS this capabilities is now available to all OpenShift deployments, no matter where they are deployed. To demonstrate this capability with an application we will deploy a PHP file uploader that has multiple front-end instances sharing a common storage repository.

First log back in as `marina`

    [root@master ~]# oc login -u marina --insecure-skip-tls-verify --server=https://master.example.com:8443

Next deploy the example application:

    [root@master ~]# oc new-app openshift/php:7.0~https://github.com/christianh814/openshift-php-upload-demo --name=file-uploader
    --> Found image a1ebebb (6 weeks old) in image stream "openshift/php" under tag "7.0" for "openshift/php:7.0"

        Apache 2.4 with PHP 7.0
        -----------------------
        Platform for building and running PHP 7.0 applications

        Tags: builder, php, php70, rh-php70

        * A source build using source code from https://github.com/christianh814/openshift-php-upload-demo will be created
          * The resulting image will be pushed to image stream "file-uploader:latest"
          * Use 'start-build' to trigger a new build
        * This image will be deployed in deployment config "file-uploader"
        * Port 8080/tcp will be load balanced by service "file-uploader"
          * Other containers can access this service through the hostname "file-uploader"

    --> Creating resources ...
        imagestream "file-uploader" created
        buildconfig "file-uploader" created
        deploymentconfig "file-uploader" created
        service "file-uploader" created
    --> Success
        Build scheduled, use 'oc logs -f bc/file-uploader' to track its progress.
        Run 'oc status' to view your app.

Wait for the application to be deployed with the suggest command:

    [root@master ~]# oc logs -f bc/file-uploader
    Cloning "https://github.com/christianh814/openshift-php-upload-demo" ...
            Commit: 7508da63d78b4abc8d03eac480ae930beec5d29d (Update index.html)
            Author: Christian Hernandez <christianh814@users.noreply.github.com>
            Date:   Thu Mar 23 09:59:38 2017 -0700
    ---> Installing application source...
    Pushing image 172.30.120.134:5000/my-test-project/file-uploader:latest ...
    Pushed 0/5 layers, 2% complete
    Pushed 1/5 layers, 20% complete
    Pushed 2/5 layers, 40% complete
    Push successful

Again kbd:\[Ctrl + c\] out of the tail mode. When the build is completed ensure the pods are running:

    [root@master ~]# oc get pods
    NAME                             READY     STATUS      RESTARTS   AGE
    file-uploader-1-build            0/1       Completed   0          2m
    file-uploader-1-k2v0d            1/1       Running     0          1m
    ...

Note the name of the single pod currently running the app: **file-uploader-1-k2v0d**. The container called `file-uploader-1-build` is the builder container and is not relevant for us. A service has been created for our app but not exposed yet. Let’s fix this:

    [root@master ~]# oc expose svc/file-uploader

Check the route that has been created:

    [root@master ~]# oc get route
    NAME                     HOST/PORT                                                      PATH      SERVICES                 PORT       TERMINATION   WILDCARD
    file-uploader            file-uploader-my-test-project.cloudapps.example.com                      file-uploader            8080-tcp                 None
    ...

Point your browser the the URL advertised by the route (<http://file-uploader-my-test-project.cloudapps.example.com>)

The application simply lists all file previously uploaded and offers the ability to upload new ones as well as download the existing data. Right now there is nothing.

Select an arbitrary from your local system and upload it to the app.

![A simple PHP-based file upload tool](uploader_screen_upload.png)

After uploading a file validate it has been stored locally in the container by following the link *List uploaded files* in the browser or logging into it via a remote session (using the name noted earlier):

    [root@master ~]# oc rsh file-uploader-1-k2v0d

    sh-4.2$ cd uploaded
    sh-4.2$ pwd
    /opt/app-root/src/uploaded
    sh-4.2$ ls -lh
    total 16K
    -rw-r--r--. 1 1000080000 root 16K May 26 09:32 cns-deploy-4.0.0-15.el7rhgs.x86_64.rpm.gz

> **Note**
>
> The exact name of the pod will be different in your environment.

The app should also list the file in the overview:

![The file has been uploaded and can be downloaded again](uploader_screen_list.png)

This pod currently does not use any persistent storage. It stores the file locally.

> **Caution**
>
> Never store data in a pod. It’s ephemeral by definition and will be lost as soon as the pod terminates.

Let’s see when this become a problem. Exit out of the container shell:

    sh-4.2$ exit

Let’s scale the deployment to 3 instances of the app:

    [root@master ~]# oc scale dc/file-uploader --replicas=3

Watch the additional pods getting spawned:

    [root@master ~]# oc get pods
    NAME                             READY     STATUS      RESTARTS   AGE
    file-uploader-1-3cgh1            1/1       Running     0          20s
    file-uploader-1-3hckj            1/1       Running     0          20s
    file-uploader-1-build            0/1       Completed   0          4m
    file-uploader-1-k2v0d            1/1       Running     0          3m
    ...

> **Note**
>
> The pod names will be different in your environment since they are automatically generated.

When you log on to one of the new instances you will see they have no data.

    [root@master ~]# oc rsh file-uploader-1-3cgh1
    sh-4.2$ cd uploaded
    sh-4.2$ pwd
    /opt/app-root/src/uploaded
    sh-4.2$ ls -hl
    total 0

Similarly, other users of the app will sometimes see your uploaded files and sometimes not - whenever the load balancing service in OpenShift points to the pod that has the file stored locally. You can simulate this with another instance of your browser in "Incognito mode" pointing to your app.

The app is of course not usable like this. We can fix this by providing shared storage to this app.

First create a PVC with the appropriate setting in a file called `cns-rwx-pvc.yml` with below contents:

**cns-rwx-pvc.yml.**

    kind: PersistentVolumeClaim
    apiVersion: v1
    metadata:
      name: my-shared-storage
      annotations:
        volume.beta.kubernetes.io/storage-class: container-native-storage
    spec:
      accessModes:
      - ReadWriteMany
      resources:
        requests:
          storage: 10Gi

Submit the request to the system:

    [root@master ~]# oc create -f cns-rwx-pvc.yml

Let’s look at the result:

    [root@master ~]# oc get pvc
    NAME                STATUS    VOLUME                                     CAPACITY   ACCESSMODES   AGE
    my-shared-storage   Bound     pvc-62aa4dfe-4ad2-11e7-b56f-2cc2602a6dc8   10Gi       RWX           22s
    ...

Notice the ACCESSMODE being set to **RWX** (short for *ReadWriteMany*, synonym for "shared storage").

We can now update the *DeploymentConfig* of our application to use this PVC to provide the application with persistent, shared storage for uploads.

    [root@master ~]# oc volume dc/file-uploader --add --name=shared-storage --type=persistentVolumeClaim --claim-name=my-shared-storage --mount-path=/opt/app-root/src/uploaded

Our app will now re-deploy (in a rolling fashion) with the new settings - all pods will mount the volume identified by the PVC under /opt/app-root/src/upload (the path is predictable so we can hard-code it here).

You can watch it like this:

    [root@master ~]# oc logs dc/file-uploader -f
    --> Scaling up file-uploader-2 from 0 to 3, scaling down file-uploader-1 from 3 to 0 (keep 3 pods available, don't exceed 4 pods)
        Scaling file-uploader-2 up to 1
        Scaling file-uploader-1 down to 2
        Scaling file-uploader-2 up to 2
        Scaling file-uploader-1 down to 1
        Scaling file-uploader-2 up to 3
        Scaling file-uploader-1 down to 0
    --> Success

The new config `file-uploader-2` will have 3 pods all sharing the same storage.

    [root@master ~]# oc get pods
    NAME                             READY     STATUS      RESTARTS   AGE
    file-uploader-1-build            0/1       Completed   0          18m
    file-uploader-2-jd22b            1/1       Running     0          1m
    file-uploader-2-kw9lq            1/1       Running     0          2m
    file-uploader-2-xbz24            1/1       Running     0          1m
    ...

Try it out in your application: upload new files and watch them being visible from within all application pods. In the browser the application behaves fluently as it circles through the pods between browser requests.

    [root@master ~]# oc rsh file-uploader-2-jd22b
    sh-4.2$ ls -lh uploaded
    total 16K
    -rw-r--r--. 1 1000080000 root 16K May 26 10:21 cns-deploy-4.0.0-15.el7rhgs.x86_64.rpm.gz
    sh-4.2$ exit
    exit
    [root@master ~]# oc rsh file-uploader-2-kw9lq
    sh-4.2$ ls -lh uploaded
    -rw-r--r--. 1 1000080000 root 16K May 26 10:21 cns-deploy-4.0.0-15.el7rhgs.x86_64.rpm.gz
    sh-4.2$ exit
    exit
    [root@master ~]# oc rsh file-uploader-2-xbz24
    sh-4.2$ ls -lh uploaded
    -rw-r--r--. 1 1000080000 root 16K May 26 10:21 cns-deploy-4.0.0-15.el7rhgs.x86_64.rpm.gz
    sh-4.2$ exit

That’s it. You have successfully provided shared storage to pods throughout the entire system, therefore avoiding the need for data to be replicated at the application level to each pod.

With CNS this is available wherever OpenShift is deployed with no external dependency.
