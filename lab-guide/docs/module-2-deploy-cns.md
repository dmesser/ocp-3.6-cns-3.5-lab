!!! Summary "Overview"
    In this module you will set up container-native storage (CNS) in your OpenShift environment. You will use this to dynamically provision storage to be available to workloads in OpenShift. It is provided by GlusterFS running in containers. GlusterFS in turn is backed by local storage available to the OpenShift nodes.

&#8680; Make sure you are logged on as the `ec2-user` to the master node:

    [ec2-user@master ~]$ hostname -f
    master.lab

!!! Caution
    All of the following tasks are carried out as the ec2-user from the master node. For Copy & Paste convenience we will omit the shell prompt unless necessary.

    All files created can be stored in root’s home directory unless a particular path is specified.

&#8680; First ensure the CNS deployment tool is available (it should already be installed)

    yum list installed cns-deploy

Configure the firewall with Ansible
----------------------------------------------

!!! Hint
    In the following we will use Ansible's configuration management capabilities in order to make sure all the OpenShift nodes have the right firewall settings. This is for your convenience.

---
&#8680; You should be able to ping all hosts using Ansible:

    ansible nodes -m ping

All 6 OpenShift application nodes should respond


    node-4.lab | SUCCESS => {
        "changed": false,
        "ping": "pong"
    }
    node-1.lab | SUCCESS => {
        "changed": false,
        "ping": "pong"
    }
    master.lab | SUCCESS => {
        "changed": false,
        "ping": "pong"
    }
    node-2.lab | SUCCESS => {
        "changed": false,
        "ping": "pong"
    }
    node-3.lab | SUCCESS => {
        "changed": false,
        "ping": "pong"
    }
    node-5.lab | SUCCESS => {
        "changed": false,
        "ping": "pong"
    }
    node-6.lab | SUCCESS => {
        "changed": false,
        "ping": "pong"
    }


&#8680; Next, create a file called `configure-firewall.yml` and copy&paste the following contents:

<kbd>configure-firewall.yml:</kbd>
```yaml
---

- hosts: nodes

  tasks:

    - name: insert iptables rules required for GlusterFS
      blockinfile:
        dest: /etc/sysconfig/iptables
        block: |
          -A OS_FIREWALL_ALLOW -p tcp -m state --state NEW -m tcp --dport 24007 -j ACCEPT
          -A OS_FIREWALL_ALLOW -p tcp -m state --state NEW -m tcp --dport 24008 -j ACCEPT
          -A OS_FIREWALL_ALLOW -p tcp -m state --state NEW -m tcp --dport 2222 -j ACCEPT
          -A OS_FIREWALL_ALLOW -p tcp -m state --state NEW -m multiport --dports 49152:49664 -j ACCEPT
        insertbefore: "^COMMIT"

    - name: reload iptables
      systemd:
        name: iptables
        state: reloaded

...
```

This small playbook will save us some work in configuring the firewall to open required ports for CNS on each individual node.

&#8680; Execute it with the following command:

    ansible-playbook configure-firewall.yml

Your output should look like this.

    PLAY [nodes]****************************************************************

    TASK [Gathering Facts]******************************************************
    ok: [node-4.lab]
    ok: [node-2.lab]
    ok: [node-1.lab]
    ok: [node-3.lab]
    ok: [master.lab]
    ok: [node-5.lab]
    ok: [node-6.lab]

    TASK [insert iptables rules required for GlusterFS]*************************
    changed: [node-1.lab]
    changed: [node-4.lab]
    changed: [node-2.lab]
    changed: [node-3.lab]
    changed: [master.lab]
    changed: [node-5.lab]
    changed: [node-6.lab]

    TASK [reload iptables]******************************************************
    changed: [node-4.lab]
    changed: [node-1.lab]
    changed: [node-2.lab]
    changed: [node-3.lab]
    changed: [master.lab]
    changed: [node-6.lab]
    changed: [node-5.lab]

    PLAY RECAP*****************************************************************
    master.lab                 : ok=3    changed=2    unreachable=0    failed=0
    node-1.lab                 : ok=3    changed=2    unreachable=0    failed=0
    node-2.lab                 : ok=3    changed=2    unreachable=0    failed=0
    node-3.lab                 : ok=3    changed=2    unreachable=0    failed=0
    node-4.lab                 : ok=3    changed=2    unreachable=0    failed=0
    node-5.lab                 : ok=3    changed=2    unreachable=0    failed=0
    node-6.lab                 : ok=3    changed=2    unreachable=0    failed=0

With this we checked the requirement for additional firewall ports to be opened on the OpenShift app nodes.

---

Prepare OpenShift for CNS
-------------------------

Next we will create a namespace (also referred to as a *Project*) in OpenShift. It will be used to group the GlusterFS pods.


!!! Warning "Important"

    If you skipped Module 1 you need give the `operator` user cluster admin privileges first:

    &#8680; Log in the built-in system admin

    ```
    oc login -u system:admin
    ```

    &#8680; Grant the user `operator` cluster admin privileges in OpenShift

    ```
    oadm policy add-cluster-role-to-user cluster-admin operator
    ```

&#8680; For the deployment you need to be logged as the `operator` user in OpenShift.

    [ec2-user@master ~]# oc whoami
    operator

&#8680; If you are for some reason not the operator, login to the default namespace like this:

    oc login -u operator -n default

&#8680; Create a namespace with the designation `container-native-storage`:

    oc new-project container-native-storage

GlusterFS pods need access to the physical block devices on the host. Hence they need elevated permissions.

&#8680; Enable containers to run in privileged mode:

    oadm policy add-scc-to-user privileged -z default

Build Container-native Storage Topology
---------------------------------------

CNS will virtualize locally attached block storage on the OpenShift App nodes. In order to deploy you will need to supply the installer with information about where to find these nodes and what network and which block devices to use.  
This is done using a JSON file describing the topology of your OpenShift deployment.

We'll start with the first 3 OpenShift app nodes. For this purpose, create the file topology.json like below.

!!! Warning "Important"
    The deployment tool always expects fully-qualified domains names for the `manage` property and always IP addresses for the `storage` property for the hostnames.

<kbd>topology.json:</kbd>

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
                }
            ]
        }
    ]
}
```

!!! Tip "What is the zone ID for?"

    Next to the obvious information like fully-qualified hostnames, IP address and device names required for Gluster the topology contains an additional property called `zone` per node.

    A zone identifies a failure domain. In CNS data is always replicated 3 times. Reflecting these by zone IDs as arbitrary but distinct numerical values allows CNS to ensure that two copies are never stored on nodes in the same failure domain.

    This information is considered when building new volumes, expanding existing volumes or replacing bricks in degraded volumes.
    An example for failure domains are AWS Availability Zones or physical servers sharing the same PDU.

---

Deploy Container-native Storage
-------------------------------

You are now ready to deploy CNS. Alongside GlusterFS pods the API front-end known as **heketi** is deployed. By default it runs without any authentication layer.
To protect the API from unauthorized access we will define passwords for the `admin` and `user` role in heketi like below. We will refer to these later again.

|Heketi Role | Password |
|------------| -------- |
| admin      | myS3cr3tpassw0rd |
|user        | mys3rs3cr3tpassw0rd |


&#8680; Next start the deployment routine with the following command:

    cns-deploy -n container-native-storage -g topology.json --admin-key 'myS3cr3tpassw0rd' --user-key 'mys3rs3cr3tpassw0rd'

Answer the interactive prompt with **Y**.

!!! Note:
    The deployment will take several minutes to complete. You may want to monitor the progress in parallel also in the OpenShift UI.

    Log in as the `operator` user to the UI and select the `container-native-storage` project.

On the command line the output should look like this:

~~~~ hl_lines="27 40 41 42 44 50 52 54 69 71"
Welcome to the deployment tool for GlusterFS on Kubernetes and OpenShift.

Before getting started, this script has some requirements of the execution
environment and of the container platform that you should verify.

The client machine that will run this script must have:
 * Administrative access to an existing Kubernetes or OpenShift cluster
 * Access to a python interpreter 'python'
 * Access to the heketi client 'heketi-cli'

Each of the nodes that will host GlusterFS must also have appropriate firewall
rules for the required GlusterFS ports:
 * 2222  - sshd (if running GlusterFS in a pod)
 * 24007 - GlusterFS Daemon
 * 24008 - GlusterFS Management
 * 49152 to 49251 - Each brick for every volume on the host requires its own
   port. For every new brick, one new port will be used starting at 49152. We
   recommend a default range of 49152-49251 on each host, though you can adjust
   this to fit your needs.

In addition, for an OpenShift deployment you must:
 * Have 'cluster_admin' role on the administrative account doing the deployment
 * Add the 'default' and 'router' Service Accounts to the 'privileged' SCC
 * Have a router deployed that is configured to allow apps to access services
   running in the cluster

Do you wish to proceed with deployment? Y
[Y]es, [N]o? [Default: Y]:

Using OpenShift CLI.
NAME                       STATUS    AGE
container-native-storage   Active    28m
Using namespace "container-native-storage".
Checking that heketi pod is not running ... OK
template "deploy-heketi" created
serviceaccount "heketi-service-account" created
template "heketi" created
template "glusterfs" created
role "edit" added: "system:serviceaccount:container-native-storage:heketi-service-account"
node "node1.example.com" labeled
node "node2.example.com" labeled
node "node3.example.com" labeled
daemonset "glusterfs" created
Waiting for GlusterFS pods to start ... OK
service "deploy-heketi" created
route "deploy-heketi" created
deploymentconfig "deploy-heketi" created
Waiting for deploy-heketi pod to start ... OK
Creating cluster ... ID: 307f708621f4e0c9eda962b713272e81
Creating node node1.example.com ... ID: f60a225a16e8678d5ef69afb4815e417
Adding device /dev/vdc ... OK
Creating node node2.example.com ... ID: 13b7c17c541069862d7e66d142ab789e
Adding device /dev/vdc ... OK
Creating node node3.example.com ... ID: 5a6fbe5eb1864e711f8bd9b0cb5946ea
Adding device /dev/vdc ... OK
heketi topology loaded.
Saving heketi-storage.json
secret "heketi-storage-secret" created
endpoints "heketi-storage-endpoints" created
service "heketi-storage-endpoints" created
job "heketi-storage-copy-job" created
deploymentconfig "deploy-heketi" deleted
route "deploy-heketi" deleted
service "deploy-heketi" deleted
job "heketi-storage-copy-job" deleted
pod "deploy-heketi-1-599rc" deleted
secret "heketi-storage-secret" deleted
service "heketi" created
route "heketi" created
deploymentconfig "heketi" created
Waiting for heketi pod to start ... OK
heketi is now running.
Ready to create and provide GlusterFS volumes.
~~~~

In order of the appearance of the highlighted lines above in a nutshell what happens here is the following:

-   Enter **Y** and press Enter.

-   OpenShift nodes are labeled. Labels are used as a selector in a `DaemonSet`.

-   GlusterFS `DaemonSet` is started. `DaemonSet` means: start exactly **one** pod per node.

-   All nodes will be referenced in heketi’s database by a UUID. Node block devices are formatted for mounting by GlusterFS.

-   A public route is created for the heketi pod to expose it's API.

-   heketi is deployed in a pod as well.

During the deployment the UI output will look like this:

[![CNS Deployment](img/openshift_cns_deploy_1.png)](img/openshift_cns_deploy_1.png)

When done it should look like this:

[![CNS Deployment](img/openshift_cns_deploy_2.png)](img/openshift_cns_deploy_2.png)

---

Verifying the deployment
------------------------

You now have deployed CNS. Let’s verify all components are in place.

&#8680; If not already there on the CLI change back to the `container-native-storage` namespace:

    oc project container-native-storage

&#8680; List all running pods:

    oc get pods -o wide

You should see all pods up and running. Highlighted below are pods that run GlusterFS containerized sharing the IP of the OpenShift node they are running on.

~~~~ hl_lines="2 3 4"
NAME              READY     STATUS    RESTARTS   AGE       IP           NODE
glusterfs-5rc2g   1/1       Running   0          3m        10.0.2.101   node-1.lab
glusterfs-jbvdk   1/1       Running   0          3m        10.0.3.102   node-2.lab
glusterfs-rchtr   1/1       Running   0          3m        10.0.4.103   node-3.lab
heketi-1-tn0s9    1/1       Running   0          2m        10.130.2.3   node-6.lab
~~~~

!!! Note
    The exact pod names will be different in your environment, since they are auto-generated.

The GlusterFS pods use the hosts network and disk devices to run the software-defined storage system. Hence they attached to the host’s network. See schematic below for a visualization.

[![GlusterFS pods in CNS in detail.](img/cns_diagram_pod.svg)](img/cns_diagram_pod.svg)

*heketi* is a component that will expose an API for GlusterFS to OpenShift. This allows OpenShift to dynamically allocate storage from CNS in a programmatic fashion. See below for a visualization. Note that for simplicity, in our lab environment heketi runs on the OpenShift App nodes, not on the Infra node.

[![heketi pod running in CNS](img/cns_diagram_heketi.svg)](img/cns_diagram_heketi.svg)

To expose heketi’s API a `service` named *heketi* has been generated in OpenShift.

&#8680; Check the service with:

    oc get service/heketi

The output should look similar to the below:

    NAME      CLUSTER-IP     EXTERNAL-IP   PORT(S)    AGE
    heketi    172.30.5.231   <none>        8080/TCP   31m

To also use heketi outside of OpenShift in addition to the service a route has been deployed.

&#8680; Display the route with:

    oc get route/heketi

The output should look similar to the below:

    NAME      HOST/PORT                                                        PATH      SERVICES   PORT      TERMINATION   WILDCARD
    heketi    heketi-container-native-storage.cloudapps.34.252.58.209.nip.io             heketi     <all>                   None

!!! Note:
    In your environment the URL will be slightly different. It will contain the public IPv4 address of your deployment, dynamically resolved by the nip.io service.

Based on this *heketi* will be available on this URL:
http://*heketi-container-native-storage.cloudapps.34.252.58.209.nip.io*

&#8680; You may verify this with a trivial health check:

    curl http://heketi-container-native-storage.cloudapps.34.252.58.209.nip.io/hello

This should say:

    Hello from Heketi

It appears heketi is running. To ensure it's functional and has been set up with authentication we are going to query it with the heketi CLI client.
The client needs to know the heketi service URL above and the password for the `admin` noted in the [deployment step](#deploy-container-native-storage).

<a name="heketi-env-setup"></a>
&#8680; Configure the heketi client with environment variables.

    export HEKETI_CLI_SERVER=http://heketi-container-native-storage.cloudapps.34.252.58.209.nip.io
    export HEKETI_CLI_USER=admin
    export HEKETI_CLI_KEY=myS3cr3tpassw0rd

!!! Caution "Important"
    Replace the FQDN in the `HEKETI_CLI_SERVER` variable above with the one specific to your environment obtained via the `oc get route/heketi` command.

!!! Tip
    It's probably a good idea to copy&paste this information somewhere for the duration of the lab. You'll need this info again later multiple times.

&#8680; You are now able to use the heketi CLI tool:

    heketi-cli cluster list

This should list at least one cluster by it's UUID:

    Clusters:
    fb67f97166c58f161b85201e1fd9b8ed

!!! Note:
    The UUID is auto-generated and will be different for you.

&#8680; Use the UUID unique to your environment and obtain more information about it:

    heketi-cli cluster info fb67f97166c58f161b85201e1fd9b8e

There should be 3 nodes and 1 volume, again displayed with their UUIDs.

    Cluster id: fb67f97166c58f161b85201e1fd9b8ed
    Nodes:
    22cbcd136fa40ffe766a13f305cc1e3b
    bfc006b571e85a083118054233bfb16d
    c5979019ac13b9fe02f4e4e2dc6d62cb
    Volumes:
    2415fba2b9364a65711da2a8311a663a

&#8680; To display a comprehensive overview of everything heketi knows about query it's topology:

    heketi-cli topology info

You will get lengthy output that shows what nodes and disk devices CNS has used to deploy a containerised GlusterFS cluster.

    Cluster Id: fb67f97166c58f161b85201e1fd9b8ed

    Volumes:

    Name: heketidbstorage
    Size: 2
    Id: 2415fba2b9364a65711da2a8311a663a
    Cluster Id: fb67f97166c58f161b85201e1fd9b8ed
    Mount: 10.0.2.101:heketidbstorage
    Mount Options: backup-volfile-servers=10.0.3.102,10.0.4.103
    Durability Type: replicate
    Replica: 3
    Snapshot: Disabled

    Bricks:
      Id: 55851d8ab270112c07ab7a38d55c8045
      Path: /var/lib/heketi/mounts/vg_41b8a921f8e6d31cb04c7dd35b6b4cf2/brick_55851d8ab270112c07ab7a38d55c8045/brick
      Size (GiB): 2
      Node: bfc006b571e85a083118054233bfb16d
      Device: 41b8a921f8e6d31cb04c7dd35b6b4cf2

      Id: 67161e0e607c38677a0ef3f617b8dc1e
      Path: /var/lib/heketi/mounts/vg_8ea71174529a35f41fc0d1b288da6299/brick_67161e0e607c38677a0ef3f617b8dc1e/brick
      Size (GiB): 2
      Node: 22cbcd136fa40ffe766a13f305cc1e3b
      Device: 8ea71174529a35f41fc0d1b288da6299

      Id: a8bf049dcea2d5245b64a792d4b85e6b
      Path: /var/lib/heketi/mounts/vg_2a49883a5cb39c3b845477ff85a729ba/brick_a8bf049dcea2d5245b64a792d4b85e6b/brick
      Size (GiB): 2
      Node: c5979019ac13b9fe02f4e4e2dc6d62cb
      Device: 2a49883a5cb39c3b845477ff85a729ba


    Nodes:

    Node Id: 22cbcd136fa40ffe766a13f305cc1e3b
    State: online
    Cluster Id: fb67f97166c58f161b85201e1fd9b8ed
    Zone: 2
    Management Hostname: node-2.lab
    Storage Hostname: 10.0.3.102
    Devices:
    Id:8ea71174529a35f41fc0d1b288da6299   Name:/dev/xvdc           State:online    Size (GiB):49      Used (GiB):2       Free (GiB):47
      Bricks:
        Id:67161e0e607c38677a0ef3f617b8dc1e   Size (GiB):2       Path: /var/lib/heketi/mounts/vg_8ea71174529a35f41fc0d1b288da6299/brick_67161e0e607c38677a0ef3f617b8dc1e/brick

    Node Id: bfc006b571e85a083118054233bfb16d
    State: online
    Cluster Id: fb67f97166c58f161b85201e1fd9b8ed
    Zone: 3
    Management Hostname: node-3.lab
    Storage Hostname: 10.0.4.103
    Devices:
    Id:41b8a921f8e6d31cb04c7dd35b6b4cf2   Name:/dev/xvdc           State:online    Size (GiB):49      Used (GiB):2       Free (GiB):47
      Bricks:
        Id:55851d8ab270112c07ab7a38d55c8045   Size (GiB):2       Path: /var/lib/heketi/mounts/vg_41b8a921f8e6d31cb04c7dd35b6b4cf2/brick_55851d8ab270112c07ab7a38d55c8045/brick

    Node Id: c5979019ac13b9fe02f4e4e2dc6d62cb
    State: online
    Cluster Id: fb67f97166c58f161b85201e1fd9b8ed
    Zone: 1
    Management Hostname: node-1.lab
    Storage Hostname: 10.0.2.101
    Devices:
    Id:2a49883a5cb39c3b845477ff85a729ba   Name:/dev/xvdc           State:online    Size (GiB):49      Used (GiB):2       Free (GiB):47
      Bricks:
        Id:a8bf049dcea2d5245b64a792d4b85e6b   Size (GiB):2       Path: /var/lib/heketi/mounts/vg_2a49883a5cb39c3b845477ff85a729ba/brick_a8bf049dcea2d5245b64a792d4b85e6b/brick

This information should correspond to the topology.json file you supplied to the installer. With this we successfully verified the CNS deployment.
