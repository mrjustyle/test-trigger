# Kubernetes

## ![](http://kubernetes.io/images/nav_logo2.svg) <a id="Kubernetes-"></a>

## Introduction to Kubernetes <a id="Kubernetes-IntroductiontoKubernetes"></a>

to be completed

### Guidelines <a id="Kubernetes-Guidelines"></a>

Reference [fabric8 Guidelines](https://fabric8.io/guide/develop/configuration.html)

## Miscellaneous <a id="Kubernetes-Miscellaneous"></a>

### Provisioning a cluster on AWS <a id="Kubernetes-ProvisioningaclusteronAWS"></a>

To be completed:

* if using kube-up.sh:
  * Note to disable node logging & deploy DaemonSets manually \(upload scripts\)
  * Note on including ephemeral volume in fluentd container
  * Note on Deploying & Configuring Prometheus
  * Note on Upgrading Grafana and importing Prometheus dashboards
* if using Kops.. \(create Terraform templates\)

### Migration <a id="Kubernetes-Migration"></a>

WIP Article on Kubernetes cluster migration for CoreOS docs: [https://github.com/coreos/docs/pull/789](https://github.com/coreos/docs/pull/789)

GitHub ticket: [https://github.com/kubernetes/kubernetes/issues/24229\#issuecomment-209899231](https://github.com/kubernetes/kubernetes/issues/24229#issuecomment-209899231)

### CLI <a id="Kubernetes-CLI"></a>

To get name of first pod matching a label query \(key=value\):

```text
kubectl get po -l key=value -o json | jq -r .items[0].metadata.name
#alternative
kubectl get po -l key=value -o name | sed 's@^.*/@@'


#usage examples:
kubectl logs -f `kubectl get po -l name=kubedeploy -o json | jq -r .items[0].metadata.name`
kubectl describe po `kubectl get po -l name=kubedeploy -o json | jq -r .items[0].metadata.name`
kubectl exec -it `kubectl get po -l name=kubedeploy -o json | jq -r .items[0].metadata.name` /bin/sh

```

Verify image & tags pushed to registry:

```text
#images
curl -sL -u user:password https://registry.honestbee.com/v2/_catalog | jq -r .repositories[]


#tags
curl -sL -u user:password https://registry.honestbee.com/v2/<image>/tags/list | jq -r .tags[]

```

Retrieve Registry secret if it already exists in a cluster

```text
kubectl get secret honestbee-registry -o json | jq -r .data[\".dockercfg\"] | base64 -D | jq
```

### Minikube <a id="Kubernetes-Minikube"></a>

Get InternalIP of nodes

```text
kubectl get no -o json | jq -r '.items[].status.addresses[] | select(.type == "InternalIP") | .address'
```

Get NodePort of service

```text
kubectl get service kubedeploy -o json | jq .spec.ports[].nodePort
```

Simplified container list:

```text
docker ps --format='table{{.Image}}\t{{.Names}}\t{{.Status}}\t{{.Ports}}\t{{.ID}}'
```

Add Image Pull Secrets to Default Service Account [ref](http://kubernetes.io/docs/user-guide/service-accounts/#adding-imagepullsecrets-to-a-service-account) \(this allows to pull images from Private Registry without having to specify the imagePullSecret in every PodSpec\)

```text
$ kubectl get serviceaccounts default -o yaml > ./sa.yaml
$ vim sa.yaml
$ cat sa.yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  creationTimestamp: 2016-09-22T08:13:12Z
  name: default
  namespace: labs
  selfLink: /api/v1/namespaces/labs/serviceaccounts/default
  uid: 67f7ad53-809c-11e6-9ff9-6e3859b59076
secrets:
- name: default-token-g5el0
imagePullSecret:
- name: honestbee-registry


$ kubectl replace serviceaccount default -f ./sa.yaml
```

### User Management <a id="Kubernetes-UserManagement"></a>

#### Basic Auth <a id="Kubernetes-BasicAuth"></a>

```text
ssh k8s-master
sudo -i
cd /srv/kubernetes
vim basic_auth.csv
vim abac-authz-policy.jsonl
docker ps --format="table {{.Image}}\t{{.ID}}"
#can try to use image filter... but tag is hard to get right
docker restart `docker ps -f ancestor=gcr.io/google_containers/kube-apiserver:<git-tag> -q`
```

Testing username & password:

```text
curl -ku <user>:<password> https://<ip>/version
```

### Retrieve Cluster Names and API Endpoints from Kube Config <a id="Kubernetes-RetrieveClusterNamesandAPIEndpointsfromKubeConfig"></a>

```text
kubectl config view -o json | jq -r '.clusters[] | [.name,.cluster.server]'
```

### Get Instance Console log of a Kubernetes node <a id="Kubernetes-GetInstanceConsolelogofaKubernetesnode"></a>

```text
aws ec2 get-console-output --instance-id `kubectl get no -o json | jq -r .items[0].spec.externalID` --output text
```

### Cluster Add-ons <a id="Kubernetes-ClusterAdd-ons"></a>

Cluster add-ons are deployed into the \`kube-system\` namespace by the add-on manager. All configuration is managed through the \`/etc/kubernetes/addons/\` folder on the master node.

Any workload deployed into the \`kube-system\` namespace which does not exist in the \`/etc/kubernetes/addons/\` folder, is automatically purged.

Any changes made to the add-ons without updating the workload definitions in the \`/etc/kubernetes/addons\` folder, are automatically reverted.

To monitor add-on manager actions within the cluster, use the following command from within the Master node:

```text
docker logs -f `docker ps -f ancestor=gcr.io/google-containers/kube-addon-manager:v4 -q`
```

### Extending Ephemeral volume: <a id="Kubernetes-ExtendingEphemeralvolume:"></a>

`m3.medium` only has 4GB Instance storage, which runs out quickly... 

```text
$ df -h
Filesystem                           Size  Used Avail Use% Mounted on
...
/dev/mapper/vg--ephemeral-ephemeral  3.9G  3.0G  710M  82% /mnt/ephemeral

```

Note: [Kubernetes on AWS needs solution for this](https://github.com/kubernetes/kubernetes/issues/14276) - it may be better to use other instance types for cluster...

Create EBS Volume

```text
$ aws ec2 create-volume --size 80 --region ap-southeast-1 --availability-zone ap-southeast-1a --volume-type gp2
```

Attach EBS Volume to instance \([get instance-id from metadata service](http://docs.aws.amazon.com/AWSEC2/latest/UserGuide/ec2-instance-metadata.html)\):

```text
ssh$ curl http://169.254.169.254/latest/meta-data/instance-id && echo #print newline
$ aws ec2 attach-volume --volume-id <volume> --instance-id <instance> --device /dev/sdg
```

Get Device info & extend ephemeral volume through LVM:

```text
$ lsblk
NAME                      MAJ:MIN RM SIZE RO TYPE MOUNTPOINT
xvda                      202:0    0  32G  0 disk
└─xvda1                   202:1    0  32G  0 part /
xvdc                      202:32   0   4G  0 disk
└─vg--ephemeral-ephemeral 254:0    0   4G  0 lvm  /mnt/ephemeral
xvdg                      202:96   0  80G  0 disk
 
#device has no fs
$ file -s /dev/xvdg
/dev/xvdg: data
 
$ pvcreate /dev/xvdg
  Physical volume "/dev/xvdg" successfully created


$ vgextend vg-ephemeral /dev/xvdg
  Volume group "vg-ephemeral" successfully extended
 
$ lvextend -r -l +100%FREE /dev/vg-ephemeral/ephemeral
  Size of logical volume vg-ephemeral/ephemeral changed from 3.99 GiB (1022 extents) to 83.99 GiB (21501 extents).
  Logical volume ephemeral successfully resized


$ df -h
Filesystem                           Size  Used Avail Use% Mounted on
...
/dev/mapper/vg--ephemeral-ephemeral   83G  3.0G   77G   4% /mnt/ephemeral
 
#device now has fs
$ file -s /dev/xvdg
/dev/xvdg: LVM2 PV (Linux Logical Volume Manager), UUID: 0Xzm43-3Ud3-pHbH-WH1A-BpEZ-dI7w-Bgj1F3, size: 85899345920
```

Refer also to [AWS EBS Guides](http://docs.aws.amazon.com/AWSEC2/latest/UserGuide/ebs-using-volumes.html)

### Working with Built-In services <a id="Kubernetes-WorkingwithBuilt-Inservices"></a>

Refer to [kubernetes.io Guides](http://kubernetes.io/docs/user-guide/accessing-the-cluster/#discovering-builtin-services)

