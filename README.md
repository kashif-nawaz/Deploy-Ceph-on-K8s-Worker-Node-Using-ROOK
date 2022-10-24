# Deploy Ceph on K8s Worker Nodes Using ROOK

## Problem Statment
* K8s is becoming a de-fecto standard to host Containerized Network Function (CNFs) in teleco / edge cloud.
* Application data storage should be accessible in case if a worker node is broken or application/ container is broken.
* It implies that data storage should be central so that failure of any component should not impact its availability, but data storage should also be distributed as well for robust access and to achieve resiliency. 
## Proposed Solution
* Ceph is 1st choice when it comes to Software Defined Storage for Telco Cloud Solution.
* ROOK is K8s operator which provides a very easy methodology to deploy Ceph in K8s Cluster. 
* Please [watch video](https://www.openstack.org/videos/summits/denver-2019/rook-a-new-and-easy-way-to-run-your-ceph-storage-on-kubernetes) to learn more about ROOK or vist ROOK [git hompage](https://github.com/rook/rook)
## Work Flow
* Deploy K8s with your preferred tool.
* Deploy your favourite CNI.
* Ensure that k8s worker nodes; intended to have Ceph OSDs should have spare hard available (unpartationed and unformatted).
* Clone the ROOK Git repo and amend  the relevant files it as per your setup. 
* Deploy the ROOK. 
* Create Ceph (shared file syste, Block or object storage) as per use case requriments. 
* Create Storage Class (SC).
* Create the Persistent Volume Claim (PVC) and Persistent Volume (PV) will be created dynamically.
* Create a POD to use already created PVC.
## Implementation Details
* It is supposed that K8s cluster is running. 
* If worker nodes are Bare metal, then add the 2nd HDD by following hardware addition/ replacement procedure from respective vendor.
* If worker nodes are virtual machines, then add the 2nd HDD while creating the VM or it can be added during run time.
* Get KVM VMs list 
  ```
    sudo virsh list | grep worker
    12    worker1                        running
    13    worker2                        running
    14    worker3                        running
  ```
* Verify hdd inside each VM / worker node
  ```
    lsblk
    NAME   MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
    sr0     11:0    1  368K  0 rom
    vda    253:0    0  100G  0 disk
    `-vda1 253:1    0  100G  0 part /
  ```
* Add 2nd hdd in KVM VM (k8s worker node)
  ```
    for domain in worker{1..3}; do
     sudo virsh vol-create-as images ${domain}-disk-2.qcow2 50G
    done

    for domain in worker{1..3}; do   
      sudo virsh attach-disk --domain ${domain}     --source /var/lib/libvirt/images/${domain}-disk-2.qcow2      --persistent --target vdb; 
    done
  ```
* Verify if 2nd hdd is present inside KVM VMs (k8s worker nodes)

  ```
   lsblk
    NAME   MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
    sr0     11:0    1  368K  0 rom
    vda    253:0    0  100G  0 disk
    `-vda1 253:1    0  100G  0 part /
    vdb    253:16   0   50G  0 disk
  ```
* Clone the ROOK git repo in your jump host
* [Ref](https://rook.io/docs/rook/v1.10/Getting-Started/quickstart/)

  ```
   git clone --single-branch --branch v1.10.4 https://github.com/rook/rook.git
   cd rook/deploy/examples
   kubectl create -f crds.yaml -f common.yaml -f operator.yaml
  ```

* Verify if ceph-operator is deployed and please don't proceed until the following are completed.

  ```
  kubectl get all -n rook-ceph | grep 'ceph-operator'
  NAME                                                  READY   STATUS      RESTARTS   AGE
  pod/rook-ceph-operator-758ddc869f-5lk94               1/1     Running     0           2m

  NAME                                                 READY   UP-TO-DATE   AVAILABLE   AGE 
  deployment.apps/rook-ceph-operator                    1/1     1           1           2m

  NAME                                                  DESIRED   CURRENT   READY      AGE
  replicaset.apps/rook-ceph-operator-758ddc869f         1       1           1           2m
  ```
* Create Ceph Cluster on K8s 

  ```
   kubectl config set-context --current --namespace rook-ceph
   Context "kubernetes-admin@cluster.local" modified.
  ```
* Amend the cluster cluster.yaml file to match your setup or leave it as it is if you want to deploy Ceph on all worker nodes.

  ```
   vim ~/rook/deploy/examples/cluster.yaml

  storage: # cluster level storage configuration and selection
    useAllNodes: false
    useAllDevices: false
    #deviceFilter:
    nodes:
    - name: "node2"
      devices: # specific devices to use for storage can be specified for each node
      - name: "vdb"
    - name: "node3"
      devices:
      - name: "vdb"
    - name: "node4"
      devices:
      - name: "vdb"
  kubectl apply -f cluster.yaml
  ```
* Watch Ceph deployment
  ```
  kubectl get pods -n rook-ceph --watch
  Wait until all pods are in running state 
  ```
* Verify if Ceph OSDs are deployed on k8s worker nodes

  ```
  kubectl get -n rook-ceph jobs.batch
  NAME                          COMPLETIONS   DURATION   AGE
  rook-ceph-osd-prepare-node2   1/1           6s         170m
  rook-ceph-osd-prepare-node3   1/1           22s        170m
  rook-ceph-osd-prepare-node4   1/1           20s        170m

  kubectl describe job.batch rook-ceph-osd-prepare-node2
  Name:             rook-ceph-osd-prepare-node2
  Namespace:        rook-ceph
  Selector:         controller-uid=2fcbcbf8-990f-4632-81f2-326199d0755c
  Labels:           app=rook-ceph-osd-prepare
                  ceph-version=16.2.9-0
                  rook-version=v1.8.10
                  rook_cluster=rook-ceph
  Annotations:      batch.kubernetes.io/job-tracking:
  Controlled By:    CephCluster/rook-ceph
  Parallelism:      1
  Completions:      1
  Completion Mode:  NonIndexed
  Start Time:       Mon, 24 Oct 2022 07:31:24 +0000
  Completed At:     Mon, 24 Oct 2022 07:31:30 +0000
  Duration:         6s
  Pods Statuses:    0 Active / 1 Succeeded / 0 Failed
  Pod Template:
  Labels:           app=rook-ceph-osd-prepare
                    ceph.rook.io/pvc=
                    controller-uid=2fcbcbf8-990f-4632-81f2-326199d0755c
                    job-name=rook-ceph-osd-prepare-node2
                    rook_cluster=rook-ceph
  Service Account:  rook-ceph-osd

  kubectl -n rook-ceph get cephcluster
  NAME        DATADIRHOSTPATH   MONCOUNT   AGE   PHASE   MESSAGE                        HEALTH      EXTERNAL
  rook-ceph   /var/lib/rook     3          21h   Ready   Cluster created successfully   HEALTH_OK
  ```
* Deploy Ceph toolbox to play around with Ceph directly

  ```
  kubectl apply -f ~/rook/deploy/examples/toolbox.yaml
  wait untill toolbox is deployed 
  ```
* Login to toolbox POD and verify Ceph Status 

  ```
  kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- bash
  [rook@rook-ceph-tools-6f7bb4b67-gvvrt /]$ceph status
  cluster:
    id:     f1fbdc34-b88e-4187-bf5c-028777547eab
    health: HEALTH_OK

  services:
    mon: 3 daemons, quorum a,b,c (age 82m)
    mgr: a(active, since 81m)
    osd: 3 osds: 3 up (since 80m), 3 in (since 80m)

  data:
    pools:   1 pools, 1 pgs
    objects: 0 objects, 0 B
    usage:   15 MiB used, 150 GiB / 150 GiB avail
    pgs:     1 active+clean

  [rook@rook-ceph-tools-6f7bb4b67-gvvrt /]$ ceph osd status
   ID  HOST    USED  AVAIL  WR OPS  WR DATA  RD OPS  RD DATA  STATE
   0  node4  5160k  49.9G      0        0       0        0   exists,up
   1  node2  5096k  49.9G      0        0       0        0   exists,up
   2  node3  5096k  49.9G      0        0       0        0   exists,up

  [rook@rook-ceph-tools-6f7bb4b67-gvvrt /]$ ceph df
  --- RAW STORAGE ---
  CLASS     SIZE    AVAIL    USED  RAW USED  %RAW USED
  hdd    150 GiB  150 GiB  15 MiB    15 MiB          0
  TOTAL  150 GiB  150 GiB  15 MiB    15 MiB          0

  --- POOLS ---
  POOL                   ID  PGS  STORED  OBJECTS  USED  %USED  MAX AVAIL
  device_health_metrics   1    1     0 B        0   0 B      0     47 GiB
  [rook@rook-ceph-tools-6f7bb4b67-gvvrt /]$ rados df
  POOL_NAME              USED  OBJECTS  CLONES  COPIES  MISSING_ON_PRIMARY  UNFOUND  DEGRADED  RD_OPS   RD  WR_OPS   WR  USED COMPR  UNDER COMPR
  device_health_metrics   0 B        0       0       0                   0        0         0       0  0 B       0  0 B         0 B          0 B

  total_objects    0
  total_used       15 MiB
  total_avail      150 GiB
  total_space      150 GiB

  [rook@rook-ceph-tools-6f7bb4b67-gvvrt /]$ ceph fs ls
  No filesystems enabled
  ```
* With ROOK we can either deploy shared file system, block storage or object storage as per use case requirements.
  - [shard file system](https://rook.io/docs/rook/v1.7/ceph-filesystem.html)
  - [block storage](https://rook.io/docs/rook/v1.7/ceph-block.html)
  - [object storage](https://rook.io/docs/rook/v1.7/ceph-object.html)
* In this article I am only covering how to create a shared file system and how to use it.
* Create Ceph Filesystem
* [Ref](https://rook.io/docs/rook/v1.10/Storage-Configuration/Shared-Filesystem-CephFS/filesystem-storage/#quotas)

  ```
  vim ~/rook/deploy/examples/filesystem.yaml
  apiVersion: ceph.rook.io/v1
  kind: CephFilesystem
  metadata:
    name: k8sfs #Name changed from myfs to k8sfs 
  kubectl apply -f  ~/rook/deploy/examples/filesystem.yaml
  ```
* Login to Ceph toolbox and verify of metadata and data pools are created 

  ```
  [rook@rook-ceph-tools-6f7bb4b67-gvvrt /]$ ceph fs ls
  name: k8sfs, metadata pool: k8sfs-metadata, data pools: [k8sfs-replicated ]

  [rook@rook-ceph-tools-6f7bb4b67-gvvrt /]$ ceph osd lspools
  1 device_health_metrics
  2 k8sfs-metadata
  3 k8sfs-replicated
  ```
* Create Storage Class (SC).
* [Ref](https://rook.io/docs/rook/v1.10/Storage-Configuration/Shared-Filesystem-CephFS/filesystem-storage/#provision-storage)

  ```
  vim ~/rook/deploy/examples/csi/cephfs/storageclass.yaml

  fsName: k8sfs #changed fsName from myfs to k8sf
  pool: k8sfs-replicated #changed pool name from myfs-replicated to k8sfs-replicated

  kubectl apply -f ~/rook/deploy/examples/csi/cephfs/storageclass.yaml

  kubectl get sc
  NAME              PROVISIONER                     RECLAIMPOLICY   VOLUMEBINDINGMODE   ALLOWVOLUMEEXPANSION   AGE
  rook-cephfs       rook-ceph.cephfs.csi.ceph.com   Delete          Immediate           true                   20h
  ```
* Create Presistent Volume and Presistent Volume Claim (PVC).

  ```
  kubectl apply -f  ~/rook/deploy/examples/csi/cephfs/pvc.yaml

  kubectl get pvc
  NAME             STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS      AGE
  cephfs-pvc       Bound    pvc-1816c01a-a05a-4a49-bad0-2c19db47c0ea   1Gi        RWO            rook-cephfs       19h

  kubectl get pv 
  NAME                                       CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM                      STORAGECLASS      REASON   AGE
  pvc-1816c01a-a05a-4a49-bad0-2c19db47c0ea   1Gi        RWO            Delete           Bound    rook-ceph/cephfs-pvc       rook-cephfs                19h
  ```
* Persistent Volume (PV) is dynamically provisioned over storage class.
* Create the POD by referring above defined  Presisten Volume Claim (PVC).
  ```
  vim ~/rook/deploy/examples/csi/cephfs/pod.yaml

  kubectl create -f  ~/rook/deploy/examples/csi/cephfs/pod.yaml

  [contrail@deployer examples]$ kubectl get pods csicephfs-demo-pod
  NAME                 READY   STATUS    RESTARTS   AGE
  csicephfs-demo-pod   1/1     Running   0          19h

  [contrail@deployer examples]$ kubectl describe  pods csicephfs-demo-pod | grep Volumes -A4
   Volumes:
     mypvc:
       Type:       PersistentVolumeClaim (a reference to a PersistentVolumeClaim in the same namespace)
       ClaimName:  cephfs-pvc
      ReadOnly:   false
  ```
