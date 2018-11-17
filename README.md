Cluster Builder
===============

[Ansible](https://www.ansible.com/) and [Packer](https://www.packer.io) IaC() scripts to configure [DC/OS](https://dcos.io/), [Docker Swarm](https://www.docker.com/), [Tectonic CoreOS](https://coreos.com) and [KubeAdm Stock Kubernetes](https://kubernetes.io/docs/setup/independent/) container orchestration clusters and deploy them into VMware environments using simple Ansible inventory host file declarations and a minimal toolset.

> Deploy a production ready container orchestration cluster to VMware in minutes while you read [hacker news](https://news.ycombinator.com/)...

![cluster-builder Overview](docs/images/cluster-builder-overview.png)

Using freely available tools and only an annotated Ansible inventory file __Cluster Builder__ enables the configuration and deployment of fleets of VMware VMs to ESXi, Workstation and Fusion hypervisors.

__Cluster Builder__ is a Packer and Ansible based infrastructure as codebase that can deploy _identical cluster VM images in both local development and production VMware environments_, supporting both _operational_ and _development_ team workflows.  This enables advanced local stack development, as well as meta infrastructure orchestration cluster development.  Clusters can be deployed and re-deployed locally, _and_ into production, identically, _in minutes!_

__Cluster Builder__ follows an [immutable infrastructure](https://www.digitalocean.com/community/tutorials/what-is-immutable-infrastructure) philosophy even at the cluster node level.  Container orchestration clusters are defined in a simple text file and then deployed using a single command.  Always repeatable and documented, this single re-usable toolset can deploy numerous and varied orchestration clusters with a clear separation of configuration and deployment artifacts, while offering a mechanism for managing the various cluster definitions packages.

__Cluster Builder__ was designed to handle ~all~ most of the complexity associated with on-prem deployments of [DC/OS](https://dcos.io/), [Docker Swarm](https://www.docker.com/) and __Kubernetes__/[Tectonic CoreOS](https://coreos.com) container orchestration clusters.

1. [Supported Clusters](#supported-clusters)
2. [Deployment Options](#deployment-options)
3. [Required Software](#required-software)
4. [General Preparation](#general-preparation)
5. [Cluster Definition Packages](#cluster-definition-packages)
6. [Cluster Builder Usage](#cluster-builder-usage)
7. [Deploying a Cluster](#deploying-a-cluster)
8. [Change Cluster Password](#change-cluster-password)
9. [Patching a Cluster](#patching-a-cluster)
10. [Adding a Node to a Cluster](#adding-a-node-to-a-cluster)
11. [Controlling Cluster VM Nodes](#controlling-cluster-vm-nodes)
12. [VMware Docker Volume Storage Driver](#vmware-docker-volume-storage-driver)
13. [Kubernetes iSCSI Provisioner and Targetd Storage Appliance](#kubernetes-iscsi-provisioner-and-targetd-storage-appliance)
14. [Kubernetes CI Job Service Accounts](#kubernetes-ci-job-service-accounts)
15. [Kubernetes Load Testing Sample Stack](#kubernetes-load-testing-sample-stack)
16. [Host Mounted NFS Storage](#host-mounted-nfs-storage)
17. [Swarm Prometheus Monitoring](#swarm-prometheus-monitoring)
18. [System Profile](#system-profile)

## Supported Clusters
The **cluster-builder** currently supports building __Swarm__, __DC/OS__, __Tectonic CoreOS__ and tock __CentOS 7.5__ and __Fedora 29 Kubernetes__ clusters for several platforms:

* CentOS 7 Docker CE
* CentOS 7 DC/OS
* RedHat Enterprise 7 Docker CE
* CoreOS Tectonic Kubernetes (see see the [CoreOS Readme](docs/README_CoreOS.md))
* CentOS 7.5 Kubernetes (Stock `kubeadm`)
* Fedora 29 Kubernetes (Stock `kubeadm`)

## Deployment Options
There are currently two types of deployment, __local machine__ and remote __ESXI__ hypervisor (or vSphere):

Local deployments are supported for:
* VMware Fusion 10+ Pro for macOS
* VMware Workstation 10+ Pro for Windows
* Vmware Workstation 10+ Pro for Linux

and for production usage:

* VMware ESXi (vSphere)

> __Note__ that __DRS__ must be turned __off__ when deploying with __cluster-builder__ to a vSphere/ESXi environment as the toolset currently expects VMs to be on the ESXi hosts specified in the deployment configuration file.  A future version will support a vSphere API based deployment option that will leverage and enable functionality such as DRS.  While DRS must be turned _off_ during current deployments, it can be turned back on when cluster deployment is complete (which usually only takes a few minutes).  This may result in the loss of post-deployment _cluster-control_ capabilities after VMs have been relocated, but should not affect cluster operations or management that relies on SSH.  On the up side, you don't need vCenter to perform __cluster-builder__ deployments.  Free ESXi will do nicely.

> Each variant starts in the **node-packer** and uses _packer_ to build a base VMX/OVA template image from distribution iso.

### Docker Swarm Cluster Types

* centos-swarm
* rhel-swarm

### DC/OS Cluster Types

* centos-dcos

### Kubernetes Cluster Types

There are two special builds in support of __Tectonic CoreOS__:

* coreos-provisioner
* coreos-pxe

For more information on these [see the CoreOS Readme](docs/README_CoreOS.md)

There are also two experimental custom built Kubernetes variants:

* centos-k8s
* fedora-k8s

__centos-k8s__ and __fedora-k8s__ are custom __Kubernetes 1.10__ clusters that implement:

* Canal/Flannel CNI Networking with Network Policy support
* Traefik for Ingress
* iSCSI Provisioner and Targetd integration for PVC storage

The __CentOS7 K8s__ cluster has been load tested and performs as well as CoreOS and with similar stability.

The __Fedora K8s__ cluster is considered experimental, but has the latest 4.x kernel and most up to date binaries.

### Extras

__cluster-builder__ can also deploy a special __Targetd Storage Appliance__ to supply persistent volume storage to Kubernetes clusters.

* targetd-server

For more information on Targetd [see the Kubernetes Storage Readme](docs/kubernetes-iscsi-storage.md)

> __Note__ it is best to deploy the __Targetd Storage Appliance__ prior to installing the Kubernetes cluster as the cluster deployment process will deploy and configure an __iscsi-provisioner__ deployment configured for the Targetd server when it already exists - and when the cluster _hosts_ file contains the necessary configuration information for the Targetd.

## Required Software

### macOS / Linux

* VMware Fusion Pro 8+ / Workstation Pro 12+
* VMware ESXi 6.5+ (optional)
* VMware's [ovftool](https://my.vmware.com/web/vmware/details?productId=614&downloadGroup=OVFTOOL420) in $PATH
* Ansible 2.3+ `brew install/upgrade ansible`
* Hashicorp [Packer 1.04+](https://www.packer.io/downloads.html)
* __kubectl__ 1.9+ (CoreOS Only - `brew install/upgrade kubernetes-cli`)

__Linux Setup Notes__

* For local machine deployments, configure _VMnet8_ (the NAT interface) with correct subnet and DHCP settings for the host file configuration and DNS names you plan to use.  The examples are given using either a __192.168.100.0__ or __192.168.101.0__ subnets (which map to the demo.idstudios.io and vm.idstudios.io example DNS names in the example cluster definition packages), but these can be adjusted as needed.  They simply need to align with the deployed __fusion_net__ and __fusion_net_type__ specified in the configuration, which in the case of Windows should be __vmnet8__.
* Ensure all VMware tools and Packer are in __PATH__.
* Ensure that the hosts configuration for local deployments uses __vmnet8__ and __nat__ for the __fusion_net__ and __fusion_net_type__ settings respectively.

> __Note__ Make sure and use _VMware's Virtual Network Editor_ that comes with the Pro version of Fusion/Workstation.  Trying to adjust the interface subnets by hand can be problematic.

__macOS Setup Notes__

* The locally deployed examples use a custom VMware Fusion host-only network that maps to **vmnet2** with the network **192.168.100.0**.  This should be created in Fusion Pro before attempting to deploy the fusion demos.

### Windows

* VMware Workstation Pro 12+
* VMware ESXi 6.5+ (optional)
* Windows Subsystem for Linux by Ubuntu (WSL)
* __Ansible__ installed in the WSL via apt-get
* __kubectl__ installed via the [xtras/wsl/install-kubectl](xtras/wsl/install-kubectl) script
* __docker-ce__ installed via the [xtras/wsl/install-kubectl](xtras/wsl/install-kubectl) script
* Hashicorp [Packer 1.04+](https://www.packer.io/downloads.html)

__Windows Setup Notes__

* When starting the WSL Bash shell make sure to start it with __Run as Administrator__.
* For local machine deployments, configure VMnet8 (the NAT interface) with correct subnet and DHCP settings for the host file configuration and DNS names you plan to use.  The examples are given using either a __192.168.100.0__ or __192.168.101.0__ subnet, but these can be adjusted as needed.  They simply need to align with the deployed __fusion_net__ and __fusion_net_type__ specified, which in the case of Windows should be __vmnet8__.
* Ensure all VMware tools and Packer are in PATH for __bash__ and __cmd__.  This can be done by adding them to the Windows system path.
  * vmrun.exe
  * ovftool.exe
* For local deployments ensure that the hosts configuration uses __vmnet8__ and __nat__ for the __fusion_net__ and __fusion_net_type__ settings respectively.

Unlike Fusion where the host-only network is NAT'd by default, host-only on VMware Workstation for Windows does not have internet access.  Through experimentation it has been found that for local machine deployments cluster builder works best on the NAT'd interface, which is __VMnet8__ by default.

### Cluster Builder Control Station

The [Cluster Builder Control](https://github.com/ids/cluster-builder-control) is also an alternative.  It is a CentOS7 desktop with all the tools required for running **cluster-builder**.

It can be used:

* Running locally on a Windows or Linux VMware Workstation, or VMware Fusion for macOS
* Running remotely on an ESXi server

It can even be built remotely directly on an ESXi server, which is the intended purpose.  For production deployments it can form the foundation for a control station that operates within the ESX environment and is used to centralize management of the clusters.

For instructions see the [Cluster Builder Control](https://github.com/ids/cluster-builder-control) README.

## General Preparation and Deployment Guides

* For all cluster types ensure that the host names specified in the inventory file also resolve.  For ESXi deployments these should resolve via DNS.  For Fusion deployments you can use __/etc/hosts__ on the host, but DNS resolution still works best.

* It is necessary that the **id_rsa.pub** value of the **cluster-builder** operator account be set in the **node-packer/keys/authorized_keys**. This is required as the scripts use passwordless SSH to access the 
VMs for provisioning.

* You will like want to adjust your __sudo__ timeout if you are doing local workstation deployments with minimal resources.  This can be done with __visudo__ and setting the [timestamp_timeout](https://lifehacker.com/make-sudo-sessions-last-longer-in-linux-1221545774) value.

* The cluster provisioning scripts rely on a **VM template OVA** that corresponds to the cluster type.  These are built by packer and located in **node-packer/output_ovas**.  See the cluster node packer [readme](https://github.com/ids/cluster-builder/blob/master/node-packer/README.md).  The **cluster-deploy** script will attempt to build the ova if it isn't found where expected.

__Note for Red Hat Deployments__
The cluster definition package (folder) you create in the __clusters__ folder will need to contain a valid __rhel7-setup.sh__ file and __rhel.lic__ file. Additionally, the ISO needs to be manually downloaded and place in **node-packer/iso**.

See the [Fusion Deployment Guide](docs/fusion_deployment_guide.md) for details on deploying on VMware Fusion.

See the [ESXi Deployment Guide](docs/esxi_deployment_guide.md) for details on deploying to ESXi hypervisor(s).

## Cluster Definition Packages

Everything is based on the **Ansible inventory file**, which defines the cluster specifications. These are defined in **hosts** files located in a folder given the cluster name:

Eg. In the **clusters/eg** folder there is:

		demo-centos-swarm
			|_ hosts

Sample cluster packages are located in the **clusters/eg** folder and can be copied into the **clusters** folder.

#### VMware Fusion Examples

* [DC/OS in VMware Fusion](clusters/eg/demo-centos-dcos/hosts)
* [Docker CE in VMware Fusion](clusters/eg/demo-centos-swarm/hosts)
* [Tectonic CoreOS in VMware Fusion - Provisioner](clusters/eg/demo-core-provisioner/hosts) and [Clusters](clusters/eg/demo-core/hosts).


#### VMware ESXi Examples

* [DC/OS on ESXi](clusters/eg/esxi-centos-dcos/hosts)
* [Docker CE on ESXi](clusters/eg/esxi-centos-swarm/hosts)
* [Tectonic CoreOS on ESXi - Provisioner](clusters/eg/core-provisioner/hosts) and [Clusters](clusters/eg/core/hosts).

## Cluster Builder Usage

The __cluster-builder__ project is designed as a generic toolset for deployment.  All user specific configuration information is stored in the cluster definition packages which are kept in the __clusters__ folder.

It is recommended that an organization establish a base folder git repository within the __clusters__ folder to store their cluster definition packages.  Anything kept in __clusters__ will be ignored by the parent cluster-builder git repository.

Eg.

	cluster-builder
		|_ clusters
			|_ ids (my organization - git repo - named anything - shorter is better)
				|_ swarm-dev (my cluster definition package)
					|_ hosts (the cluster inventory hosts file)

All resulting artifacts from __cluster-builder__ are then stored within the cluster definition package.

## Deploying a Cluster
To deploy a cluster use **cluster-deploy**:

    $ bash cluster-deploy <inventory-package | cluster-name>

Eg.

    $ bash cluster-deploy eg/demo-centos-swarm

## Change Cluster Password
Change password is now integrated into the cluster deployment process.

For __CentOS__ deployments, both the __root__ and __admin__ passwords are prompted for change at the end of the cluster deployment.

> A bit of an annoyance, but it is integrated to ensure that clusters are never deployed into production with default root passwords.  TODO: Enhance to support prompt-free deployments.

This functionality is also available as as top level script:

	bash cluster-passwd <cluster package> [user to change]

Eg.

	bash cluster-passwd esxi-centos-swarm admin

It is intended to be run on a regular basis as per the standard operating procedures for password change management.

## Patching a Cluster
To update the nodes on a deployed cluster, use **cluster-update**:

    $ bash cluster-deploy <inventory-package | cluster-name>

Eg.

    $ bash cluster-deploy eg/esxi-rhel-swarm

## Adding a Node to a Cluster
To add a new node to an existing cluster, update the original hosts file with the new node.
Then use **cluster-add**:

    $ bash cluster-add <inventory-package | cluster-name>

Eg.

    $ bash cluster-add esxi-rhel-swarm

## Controlling Cluster VM Nodes
There are ansible tasks that use the inventory files to execute VM control commands.

Use **cluster-control**:

    bash cluster-control <inventory-package | cluster-name> <action: one of stop|suspend|start|destroy>

Eg.

    $ bash cluster-control demo-centos-swarm suspend


## VMware Docker Storage Volume Driver Plugin

In order to make use of the VMware Volume Driver Plugin (vDVS) for persistent data volume management the VIB must be installed on all of the ESX servers used for the cluster.

Download the [VIB](https://bintray.com/vmware/vDVS/download_file?file_path=VMWare_bootbank_esx-vmdkops-service_0.19.641f741-0.0.1.vib) for the vDVS.


It is easiest to download the VIB to a shared datastore accessible to all of the ESX servers, and then copy it locally to the /tmp folder during each install.

> Make sure to put the VIB in tmp on the ESX server and reference it absolutely:

Eg.

	esxcli software vib install --no-sig-check -v /tmp/VMWare_bootbank_esx-vmdkops-service_0.20.15f5e1d-0.0.1.vib

The plugin is automatically installed as part of the cluster-builder swarm provisioning process. However, this can also be manually done on cluster nodes with the following command:

	docker plugin install --grant-all-permissions --alias vsphere vmware/docker-volume-vsphere:latest

## Kubernetes iSCSI Provisioner and Targetd Storage Appliance

As Kubernetes provides native storage support for __iSCSI__ and __NFS__, the cleanest most efficient path to providing __persistent volume ReadWriteOnce__ storage is to leverage iSCSI.

The __cluster-builder__ CoreOS deployment is paired with a __Targetd Server Appliance__ VM that can provide dynamically provisioned __PVCs__ to Kubernetes deployments using the __open-iscsi__ platform.

For details see the [Kubernetes iSCSI Storage Guide](docs/kubernetes-iscsi-storage.md)

## Host Mounted NFS Storage

Place the following file in the cluster definition package folder:

  nfs_shares.yml

In the format:

  nfs_shares:
    - folder: the name of the local mount folder
      fstab: the fstab entry
      group: an inventory group of hosts on which to setup this mount

Eg.

  nfs_shares:
    - folder: /mnt/nfs/shared
      fstab: "192.168.1.10:/Users/seanhig/NFS_SharedStorage  /mnt/nfs/shared   nfs      rw,sync,hard,intr  0     0"
      group: "docker_swarm_worker"
    - folder: /mnt/nfs/shared
      fstab: "192.168.1.10:/Users/seanhig/Google\\040Drive/Backups/NFS_Backups  /mnt/nfs/backups   nfs      rw,sync,hard,intr  0     0"
      group: "docker_swarm_worker"

And then run the ansible playbook for the platform:

Eg.

  ansible-playbook -i clusters/esxi-centos-swarm/hosts ansible/centos-nfs-shares.yml

And it will setup the mounts according to host group membership specified in the nfs_shares.yml configuration.

## Kubernetes CI Job Service Accounts

Kubernetes RBAC and service accounts offer a popular model for granting controlled access to CI/CD processes.  It involves creating a `ClusterRole` with the necessary object/verb permission ACLs, and then associating it via `ClusterRoleBinding` to a Kubernetes __service account__, authenticated via an __access token__, stored as a `Secret`.

### Step 1 - Create the Service Account

  kubectl create serviceaccount ci-runner

### Step 2 - Get the Service Account Secret Tokens & Build Kube Config

  kubectl get secrets
  kubectl describe secret ci-runner-<hash>

This will show two tokens, the CA and the user token.  Use them to construct a kube-config for your cluster using the __ci-runner__ service account.

Eg.

```
apiVersion: v1
clusters:
- cluster:
    certificate-authority-data: LS0tL<blah>
    server: https://pks-k8s-01.onprem.idstudios.io:8443
  name: k8s-01-runner
contexts:
- context:
    cluster: k8s-01-runner
    user: ci-runner
  name: k8s-01-runner
current-context: k8s-01-runner
kind: Config
preferences: {}
users:
- name: ci-runner
  user:
    token: eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJrdWJlcm5ldGVzL3NlcnZpY2VhY2NvdW50Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9uYW1lc3BhY2UiOiJkZWZhdWx0Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9zZWNyZXQubmFtZSI6ImNpLXJ1bm5lci10b2tlbi05N3dycCIsImt1YmVybmV0ZXMuaW8vc2VydmljZWFjY291bnQvc2VydmljZS1hY2NvdW50Lm5hbWUiOiJjaS1ydW5uZXIiLCJrdWJlcm5ldGVzLmlvL3NlcnZpY2VhY2NvdW50L3NlcnZpY2UtYWNjb3VudC51aWQiOiI3ZTY4OTMyYS00M2FjLTExZTgtYjM1Zi0wMDUwNTY4YTVkNjciLCJzdWIiOiJzeXN0ZW06c2VydmljZWFjY291bnQ6ZGVmYXVsdDpjaS1ydW5uZXIifQ.c6nkA8PK1-NJ2ObOwHpaARpDLddPlAgzHcyEh0xM1F88UbpTBh3DdkA_xc0dtJUOeTOn4CrUYgBOPgbFfurweSix53G4wOeOnYnxJrA7PtPJjXUn54peGse_LFp6UCaufPEPcCvVgc2UcRL4DSLPZWwziGhxhm4p-qsTbl_r9SQhvAC_CKYyrYX00q_vcZQS-cdqvo1e34YVIb7W7neWCmzEitKwslMz0IkFYkgJbrQU2RvkmVDEhzBTm0qf6DthzvnEzRXTkMPBvuIAZd6AMCKffzF-XKRWkkV9HTRc2Muu0rZEWkSsPqd_hEMxfrPCOhu2l8n9AVAZ4GrkWOC2_w
```

Save this as `ci-runner-kube-config`.

### Step 3 - Switch to Service Account Context and Verify No Access to Namespace

		kubectl --kubeconfig ci-runner-kube-config config use-context k8s-01-runner
		kubectl --kubeconfig ci-runner-kube-config get pods

You will see a message indicating that the ci-runner service account does not have access.

### Step 4 - Create ClusterRole and ClusterRoleBinding to Grant Access to Namespace

		apiVersion: rbac.authorization.k8s.io/v1
		kind: ClusterRole
		metadata:
			name: ci-runner-role
		rules:
		- apiGroups: [""]
			resources: ["pods"]
			verbs: ["get", "list", "watch"]  
		---
		apiVersion: rbac.authorization.k8s.io/v1
		kind: RoleBinding
		metadata:
			name: ci-runner-role-binding
			namespace: default
		roleRef:
			apiGroup: rbac.authorization.k8s.io
			kind: ClusterRole
			name: ci-runner-role
		subjects:
		- kind: ServiceAccount
			name: ci-runner
			namespace: default  

In this example __ci-runner__ has access to __get__,__list__ and __watch__ pods in the default namespace.

For a __CI/CD__ deployment your ACLs may look more like the following:

		apiVersion: rbac.authorization.k8s.io/v1
		kind: ClusterRole
		metadata:
			name: ci-runner-role
		rules:
			- apiGroups: [""]
				resources: ["*"]
				verbs: ["*"]

Which grants full access to the service account namespace (in this case _default_).

### Step 5 - Base64 the Service Account kube-config into a Gitlab CI/CD Secret

		cat ci-runner-kube-config | base64 | pbcopy

And paste it in the secrets stored in __Gitlab Project > Settings > CI/CD > Secret Variables__.

Then use the codeified kube-config to access the target Kubernetes cluster in Gitlab CI/CD:

		deploy:
			stage: deploy
			image: lwolf/helm-kubectl-docker:v152_213
			before_script:
				- mkdir -p /etc/deploy
				- echo ${kube_config} | base64 -d > ${KUBECONFIG}
				- kubectl config use-context k8s-01
			script:
				- kubectl get pods -n kube-system
				- kubectl do some deployment stuff here
			only:
			- master

## Kubernetes Load Testing Sample Stack

A sample application stack has been included with cluster-builder that can be used to perform basic performance and load testing on deployed Kubernetes clusters.

It is comprised of a __MariaDB Galera Active/Active 3 or 5 Node Database Cluster__ paired with a __Drupal 7__ web front-end, which accesses the database through a dedicated __HA Proxy__ load balancer.

For guidance on generating the manifests for your __cluster-builder__ cluster, deploying the stack, and performing load tests, see the [Drupal K8s Test Stack Guide](docs/drupal-k8s-test-stack.md).

## Swarm Prometheus Monitoring

Currently, __cAdvisor__ and __node-exporter__ are installed on CentOS Swarms, with metrics enabled by default.

When the following is added to a cluster package hosts file:

docker_prometheus_server=<ansible inventory hostname>

Eg.

	docker_prometheus_server=swarm-m1

Prometheus and Grafana containers will be installed on the specified node.

Promethus can then be reached at: http://<cluster node>:9090

Grafana at: http://<cluster node>:3000

> TODO: These need to be TLS secured and made production ready

## System Profile

A general overview of the highlights:

### Docker Versions

__Docker CE:__ 17.09.1-ce (or later)
centos-swarm
rhel-swarm

__DC/OS__: 1.11 (or latest)
centos-dcos

__Tectonic CoreOS__: v1.9.6 (or latest)
coreos-provisioner
coreos-pxe

__Stock Kubernetes__: v1.12.1
coreos-k8s
fedora-k8s


### CentOS Based Clusters

* CentOS base VM image OVA template is based on the CentOS 7 Minimal 1708 iso and is  __1.1GB__, and contains one thinly provisioned SCSI based VMDK disk using __overlay2__, which is now supported on 1708 (CentOS/RHEL 7.4+).
* CentOS base VM image is a fully functioning and ready worker node.
* Default linux kernel is 3.10.x
* Use __packer centric__ approach for provisioning, VM OVA based nodes are ready to be added to swarms
* The __VMware Docker Volume Service__ Docker Volume Plugin has been pre-installed on all Swarm based cluster-builder VMs.
* Time synchronization of all the cluster nodes is done as part of the deployment process, and __chronyd__ or __ntpd__ services are configured and verified.
* Deployments can include configurable options for log shipping to ELK, using logstash.  
* Metrics are enabled (a configurable option), and cAdvisor/node-exporter options are available for deployment in support of Prometheus/Grafana monitoring for Docker Swarm.  Tectonic CoreOS has built in Prometheus/Grafana integrations for Kubernetes that are quite a bit more advanced then the Swarm implementation.
* Remote API and TLS certificates are installed and configured on Docker CE deployments.

> Note that all details pertaining to the above exist within this codebase. The cluster-builder starts with the distribution iso file in the initial [node-packer](node-packer) phase, and everything from the initial __kickstart__ install through to the final __ansible playbook__ are documented here and available for review.
