# Provisioning Compute Resources

Kubernetes requires a set of machines to host the Kubernetes control plane and the worker nodes where containers are ultimately run. In this lab you will provision the compute resources required for running a secure and highly available Kubernetes cluster across a single [compute zone](https://cloud.google.com/compute/docs/regions-zones/regions-zones).

> Ensure a default compute zone and region have been set as described in the [Prerequisites](01-prerequisites.md#set-a-default-compute-region-and-zone) lab.

## Networking

The Kubernetes [networking model](https://kubernetes.io/docs/concepts/cluster-administration/networking/#kubernetes-model) assumes a flat network in which containers and nodes can communicate with each other. In cases where this is not desired [network policies](https://kubernetes.io/docs/concepts/services-networking/network-policies/) can limit how groups of containers are allowed to communicate with each other and external network endpoints.

> Setting up network policies is out of scope for this tutorial.

### Virtual Private Cloud Network

In this section a dedicated [Virtual Private Cloud](https://cloud.google.com/compute/docs/networks-and-firewalls#networks) (VPC) network will be setup to host the Kubernetes cluster.

Create the `kubernetes-the-hard-way` custom VPC network:

```bash
gcloud compute networks create kubernetes-the-hard-way --subnet-mode custom
```

---
#### My steps:
> Output:
```bash
Created [https://www.googleapis.com/compute/v1/projects/sb-sandbox4cv/global/networks/kubernetes-the-hard-way].
NAME                     SUBNET_MODE  BGP_ROUTING_MODE  IPV4_RANGE  GATEWAY_IPV4
kubernetes-the-hard-way  CUSTOM       REGIONAL

Instances on this network will not be reachable until firewall rules
are created. As an example, you can allow all internal traffic between
instances as well as SSH, RDP, and ICMP by running:

$ gcloud compute firewall-rules create <FIREWALL_NAME> --network kubernetes-the-hard-way --allow tcp,udp,icmp --source-ranges <IP_RANGE>
$ gcloud compute firewall-rules create <FIREWALL_NAME> --network kubernetes-the-hard-way --allow tcp:22,tcp:3389,icmp
```
---

A [subnet](https://cloud.google.com/compute/docs/vpc/#vpc_networks_and_subnets) must be provisioned with an IP address range large enough to assign a private IP address to each node in the Kubernetes cluster.

Create the `kubernetes` subnet in the `kubernetes-the-hard-way` VPC network:

```bash
gcloud compute networks subnets create kubernetes \
  --network kubernetes-the-hard-way \
  --range 10.240.0.0/24
```

---
#### My steps:
> Output:
```bash
Created [https://www.googleapis.com/compute/v1/projects/sb-sandbox4cv/regions/us-east4/subnetworks/kubernetes].
NAME        REGION    NETWORK                  RANGE          STACK_TYPE  IPV6_ACCESS_TYPE  INTERNAL_IPV6_PREFIX  EXTERNAL_IPV6_PREFIX
kubernetes  us-east4  kubernetes-the-hard-way  10.240.0.0/24  IPV4_ONLY
```
---

> The `10.240.0.0/24` IP address range can host up to 254 compute instances.

### Firewall Rules

Create a firewall rule that allows internal communication across all protocols:

```bash
gcloud compute firewall-rules create kubernetes-the-hard-way-allow-internal \
  --allow tcp,udp,icmp \
  --network kubernetes-the-hard-way \
  --source-ranges 10.240.0.0/24,10.200.0.0/16
```

---
#### My steps:
> Output:
```bash
Creating firewall...⠹Created [https://www.googleapis.com/compute/v1/projects/sb-sandbox4cv/global/firewalls/kubernetes-the-hard-way-allow-internal].
Creating firewall...done.
NAME                                    NETWORK                  DIRECTION  PRIORITY  ALLOW         DENY  DISABLED
kubernetes-the-hard-way-allow-internal  kubernetes-the-hard-way  INGRESS    1000      tcp,udp,icmp        False
```
---

Create a firewall rule that allows external SSH, ICMP, and HTTPS:

```bash
gcloud compute firewall-rules create kubernetes-the-hard-way-allow-external \
  --allow tcp:22,tcp:6443,icmp \
  --network kubernetes-the-hard-way \
  --source-ranges 0.0.0.0/0
```

---
#### My steps:
> Output:
```bash
Creating firewall...⠹Created [https://www.googleapis.com/compute/v1/projects/sb-sandbox4cv/global/firewalls/kubernetes-the-hard-way-allow-external].
Creating firewall...done.
NAME                                    NETWORK                  DIRECTION  PRIORITY  ALLOW                 DENY  DISABLED
kubernetes-the-hard-way-allow-external  kubernetes-the-hard-way  INGRESS    1000      tcp:22,tcp:6443,icmp        False
```
---

> An [external load balancer](https://cloud.google.com/compute/docs/load-balancing/network/) will be used to expose the Kubernetes API Servers to remote clients.

List the firewall rules in the `kubernetes-the-hard-way` VPC network:

```bash
gcloud compute firewall-rules list --filter="network:kubernetes-the-hard-way"
```

> output

```
NAME                                    NETWORK                  DIRECTION  PRIORITY  ALLOW                 DENY  DISABLED
kubernetes-the-hard-way-allow-external  kubernetes-the-hard-way  INGRESS    1000      tcp:22,tcp:6443,icmp        False
kubernetes-the-hard-way-allow-internal  kubernetes-the-hard-way  INGRESS    1000      tcp,udp,icmp                Fals
```

### Kubernetes Public IP Address

Allocate a static IP address that will be attached to the external load balancer fronting the Kubernetes API Servers:

```bash
gcloud compute addresses create kubernetes-the-hard-way \
  --region $(gcloud config get-value compute/region)
```

Verify the `kubernetes-the-hard-way` static IP address was created in your default compute region:

```bash
gcloud compute addresses list --filter="name=('kubernetes-the-hard-way')"
```

> output

```
NAME                     ADDRESS/RANGE   TYPE      PURPOSE  NETWORK  REGION    SUBNET  STATUS
kubernetes-the-hard-way  XX.XXX.XXX.XXX  EXTERNAL                    us-west1          RESERVED
```

## Compute Instances

The compute instances in this lab will be provisioned using [Ubuntu Server](https://www.ubuntu.com/server) 20.04, which has good support for the [containerd container runtime](https://github.com/containerd/containerd). Each compute instance will be provisioned with a fixed private IP address to simplify the Kubernetes bootstrapping process.

### Kubernetes Controllers

Create three compute instances which will host the Kubernetes control plane:

```
for i in 0 1 2; do
  gcloud compute instances create controller-${i} \
    --async \
    --boot-disk-size 200GB \
    --can-ip-forward \
    --image-family ubuntu-2004-lts \
    --image-project ubuntu-os-cloud \
    --machine-type e2-standard-2 \
    --private-network-ip 10.240.0.1${i} \
    --scopes compute-rw,storage-ro,service-management,service-control,logging-write,monitoring \
    --subnet kubernetes \
    --tags kubernetes-the-hard-way,controller
done
```

---
#### My steps:
The above for loop gave me the below error:
```bash
ERROR: (gcloud.compute.instances.create) HTTPError 412: Constraint constraints/compute.vmExternalIpAccess violated for project 938803283717. Add instance projects/sb-sandbox4cv/zones/us-east4-c/instances/controller-0 to the constraint to use external IP with it.
```
[GCP org policies for my project](https://console.cloud.google.com/iam-admin/orgpolicies/compute-vmExternalIpAccess?project=sb-sandbox4cv&supportedpurview=project,folder,organizationId)
![[Pasted image 20220827150515.png]]

So I manually created a VM in the console to get the equivalent command:
```bash
for i in 0 1 2; do
  gcloud compute instances create controller-${i}  \
    --async \
    --project=sb-sandbox4cv  \
    --zone=us-east4-c  \
    --machine-type=e2-standard-2  \
    --network-interface=private-network-ip=10.240.0.1${i},subnet=kubernetes,no-address  \
    --can-ip-forward  \
    --scopes=https://www.googleapis.com/auth/devstorage.read_only,https://www.googleapis.com/auth/logging.write,https://www.googleapis.com/auth/monitoring.write,https://www.googleapis.com/auth/servicecontrol,https://www.googleapis.com/auth/service.management.readonly,https://www.googleapis.com/auth/trace.append  \
    --create-disk=auto-delete=yes,boot=yes,device-name=controller-${i},image=projects/ubuntu-os-cloud/global/images/ubuntu-2004-focal-v20220823,mode=rw,size=200,type=projects/sb-sandbox4cv/zones/us-east4-c/diskTypes/pd-balanced  \
    --tags kubernetes-the-hard-way,controller
done
```
```bash
# This works as well and is more similar to the instuctions
# The key here is to add the 'no-address' parameter to the 'network-interface' option
for i in 0 1 2; do
  gcloud compute instances create controller-${i} \
    --async \
    --boot-disk-size 200GB \
    --can-ip-forward \
    --image-family ubuntu-2004-lts \
    --image-project ubuntu-os-cloud \
    --machine-type e2-standard-2 \
    --network-interface=private-network-ip=10.240.0.1${i},subnet=kubernetes,no-address \
    --scopes compute-rw,storage-ro,service-management,service-control,logging-write,monitoring \
    --tags kubernetes-the-hard-way,controller
done
```
---

### Kubernetes Workers

Each worker instance requires a pod subnet allocation from the Kubernetes cluster CIDR range. The pod subnet allocation will be used to configure container networking in a later exercise. The `pod-cidr` instance metadata will be used to expose pod subnet allocations to compute instances at runtime.

> The Kubernetes cluster CIDR range is defined by the Controller Manager's `--cluster-cidr` flag. In this tutorial the cluster CIDR range will be set to `10.200.0.0/16`, which supports 254 subnets.

Create three compute instances which will host the Kubernetes worker nodes:

```
for i in 0 1 2; do
  gcloud compute instances create worker-${i} \
    --async \
    --boot-disk-size 200GB \
    --can-ip-forward \
    --image-family ubuntu-2004-lts \
    --image-project ubuntu-os-cloud \
    --machine-type e2-standard-2 \
    --metadata pod-cidr=10.200.${i}.0/24 \
    --private-network-ip 10.240.0.2${i} \
    --scopes compute-rw,storage-ro,service-management,service-control,logging-write,monitoring \
    --subnet kubernetes \
    --tags kubernetes-the-hard-way,worker
done
```

---
#### My Steps:
Similar to creating the controllers, I got an error so I modified the controller script and modified for the worker nodes
```bash
for i in 0 1 2; do
  gcloud compute instances create worker-${i}  \
    --async \
    --project=sb-sandbox4cv  \
    --zone=us-east4-c  \
    --machine-type=e2-standard-2  \
    --network-interface=private-network-ip=10.240.0.2${i},subnet=kubernetes,no-address  \
    --can-ip-forward  \
    --scopes=https://www.googleapis.com/auth/devstorage.read_only,https://www.googleapis.com/auth/logging.write,https://www.googleapis.com/auth/monitoring.write,https://www.googleapis.com/auth/servicecontrol,https://www.googleapis.com/auth/service.management.readonly,https://www.googleapis.com/auth/trace.append  \
    --create-disk=auto-delete=yes,boot=yes,device-name=worker-${i},image=projects/ubuntu-os-cloud/global/images/ubuntu-2004-focal-v20220823,mode=rw,size=200,type=projects/sb-sandbox4cv/zones/us-east4-c/diskTypes/pd-balanced  \
    --metadata pod-cidr=10.200.${i}.0/24 \
    --tags kubernetes-the-hard-way,controller
done
```
---

### Verification

List the compute instances in your default compute zone:

```bash
gcloud compute instances list --filter="tags.items=kubernetes-the-hard-way"
```

> output

```
NAME          ZONE        MACHINE_TYPE   PREEMPTIBLE  INTERNAL_IP  EXTERNAL_IP    STATUS
controller-0  us-west1-c  e2-standard-2               10.240.0.10  XX.XX.XX.XXX   RUNNING
controller-1  us-west1-c  e2-standard-2               10.240.0.11  XX.XXX.XXX.XX  RUNNING
controller-2  us-west1-c  e2-standard-2               10.240.0.12  XX.XXX.XX.XXX  RUNNING
worker-0      us-west1-c  e2-standard-2               10.240.0.20  XX.XX.XXX.XXX  RUNNING
worker-1      us-west1-c  e2-standard-2               10.240.0.21  XX.XX.XX.XXX   RUNNING
worker-2      us-west1-c  e2-standard-2               10.240.0.22  XX.XXX.XX.XX   RUNNING
```

---
#### My Steps:
```bash
# I don't have EXTERNAL_IPs
NAME          ZONE        MACHINE_TYPE   PREEMPTIBLE  INTERNAL_IP  EXTERNAL_IP  STATUS
controller-0  us-east4-c  e2-standard-2               10.240.0.10               RUNNING
controller-1  us-east4-c  e2-standard-2               10.240.0.11               RUNNING
controller-2  us-east4-c  e2-standard-2               10.240.0.12               RUNNING
worker-0      us-east4-c  e2-standard-2               10.240.0.20               RUNNING
worker-1      us-east4-c  e2-standard-2               10.240.0.21               RUNNING
worker-2      us-east4-c  e2-standard-2               10.240.0.22               RUNNING
```
---

## Configuring SSH Access

SSH will be used to configure the controller and worker instances. When connecting to compute instances for the first time SSH keys will be generated for you and stored in the project or instance metadata as described in the [connecting to instances](https://cloud.google.com/compute/docs/instances/connecting-to-instance) documentation.

Test SSH access to the `controller-0` compute instances:

```bash
gcloud compute ssh controller-0
```

If this is your first time connecting to a compute instance SSH keys will be generated for you. Enter a passphrase at the prompt to continue:

```
WARNING: The public SSH key file for gcloud does not exist.
WARNING: The private SSH key file for gcloud does not exist.
WARNING: You do not have an SSH key for gcloud.
WARNING: SSH keygen will be executed to generate a key.
Generating public/private rsa key pair.
Enter passphrase (empty for no passphrase):
Enter same passphrase again:
```

At this point the generated SSH keys will be uploaded and stored in your project:

```
Your identification has been saved in /home/$USER/.ssh/google_compute_engine.
Your public key has been saved in /home/$USER/.ssh/google_compute_engine.pub.
The key fingerprint is:
SHA256:nz1i8jHmgQuGt+WscqP5SeIaSy5wyIJeL71MuV+QruE $USER@$HOSTNAME
The key's randomart image is:
+---[RSA 2048]----+
|                 |
|                 |
|                 |
|        .        |
|o.     oS        |
|=... .o .o o     |
|+.+ =+=.+.X o    |
|.+ ==O*B.B = .   |
| .+.=EB++ o      |
+----[SHA256]-----+
Updating project ssh metadata...-Updated [https://www.googleapis.com/compute/v1/projects/$PROJECT_ID].
Updating project ssh metadata...done.
Waiting for SSH key to propagate.
```

After the SSH keys have been updated you'll be logged into the `controller-0` instance:

```
Welcome to Ubuntu 20.04.2 LTS (GNU/Linux 5.4.0-1042-gcp x86_64)
...
```

Type `exit` at the prompt to exit the `controller-0` compute instance:

```
$USER@controller-0:~$ exit
```
> output

```
logout
Connection to XX.XX.XX.XXX closed
```

---
#### My steps:
```bash
o2146@lnx33361 [15:40:55] [~]
-> % gcloud compute ssh controller-0
WARNING: The private SSH key file for gcloud does not exist.
WARNING: The public SSH key file for gcloud does not exist.
WARNING: You do not have an SSH key for gcloud.
WARNING: SSH keygen will be executed to generate a key.
Generating public/private rsa key pair.
Enter passphrase (empty for no passphrase):
Enter same passphrase again:
Your identification has been saved in /home/o2146/.ssh/google_compute_engine.
Your public key has been saved in /home/o2146/.ssh/google_compute_engine.pub.
The key fingerprint is:
SHA256:hQFb3w7Skb0PPZPjOTpo/MhcBT4H/Uo3QawkHU89NHk o2146@lnx33361
The keys randomart image is:
+---[RSA 3072]----+
|      ..o .o..+=o|
|       o =.+.+++E|
|      . o = *oo+o|
|         o +o+*..|
|        S   +++*o|
|             ==o.|
|         . ..... |
|         o+oo    |
|         .+...   |
+----[SHA256]-----+
External IP address was not found; defaulting to using IAP tunneling.
Updating project ssh metadata...⠏Updated [https://www.googleapis.com/compute/v1/projects/sb-sandbox4cv].
Updating project ssh metadata...done.
Waiting for SSH key to propagate.
WARNING:

To increase the performance of the tunnel, consider installing NumPy. For instructions,
please see https://cloud.google.com/iap/docs/using-tcp-forwarding#increasing_the_tcp_upload_bandwidth

Warning: Permanently added 'compute.1996056319939307837' (ECDSA) to the list of known hosts.
Enter passphrase for key '/home/o2146/.ssh/google_compute_engine':
WARNING:

To increase the performance of the tunnel, consider installing NumPy. For instructions,
please see https://cloud.google.com/iap/docs/using-tcp-forwarding#increasing_the_tcp_upload_bandwidth

Enter passphrase for key '/home/o2146/.ssh/google_compute_engine':
Welcome to Ubuntu 20.04.4 LTS (GNU/Linux 5.15.0-1016-gcp x86_64)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/advantage

  System information as of Sat Aug 27 19:40:51 UTC 2022

  System load:  0.0                Processes:             102
  Usage of /:   0.9% of 193.65GB   Users logged in:       0
  Memory usage: 2%                 IPv4 address for ens4: 10.240.0.10
  Swap usage:   0%


0 updates can be applied immediately.

Failed to connect to https://changelogs.ubuntu.com/meta-release-lts. Check your Internet connection or proxy settings


Last login: Sat Aug 27 18:49:52 2022 from 35.235.241.33
o2146@controller-0:~$ logout
Connection to compute.1996056319939307837 closed.
```
---

Next: [Provisioning a CA and Generating TLS Certificates](04-certificate-authority.md)
