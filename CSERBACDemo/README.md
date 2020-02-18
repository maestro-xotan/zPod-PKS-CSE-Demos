# CSE Tenant On-Boarding and RBAC Functionality Demo

<img src="Images/rbac-diagram.png">

In this demo, we'll walk through the process of onboarding a tenant in vCD to deploy Enterprise PKS clusters via the Container Service Extension. We'll also utilize the RBAC funcitonality provided by CSE to ensure only certain users within the org can provision clusters. We are going to utilize the `enterprise-dev-org` tenant for this demo.

For future demos, there is a cluster (`demo-cluster`) provisioned with 3 worker nodes that you can use to deploy applications. This demo showcases the workflow for deploying a cluster via CSE. This cluster will only have 1 worker node to ensure the cluster provisions quickly. This cluster should not be used for the app demos as it may not have enough resources to support multiple applications.

**Note:** Ensure that you are using your Horizon instance (access instruction detailed [here](https://confluence.eng.vmware.com/display/CPCSA/CSE+zPod+Lab+Access+and+Demo+Scripts)) to access the demo environment.

## Step 1: Prepare Tenant Org

**1.1** Before starting the demo, access the `cse-client` server from your Horizon instance via putty (pw is `VMware1!`):

<img src="Images/putty-ss.png">

**1.2** Once logged in to the `cse-client` server, run the clean-up script to ensure the `enterprise-dev-org` is not enabled to provision Enterprise PKS clusters:

(**Note**: please reach out to Joe Mann or ping in the `#zpod-feedback` slack channel for access credentials for the system admin user for this demo.)

~~~
$ ./zPod-PKS-CSE-Demos/CSERBACDemo/onboarding-demo-cleanup.sh
~~~

**1.3** Log in with the system admin user of the vCloud Director environment: 

~~~
$ vcd login director.vcd.zpod.io system administrator -iw
~~~

**1.4** Verify the `enterprise-dev-org` and it's corresponding OvDC is NOT enabled with a `k8 provider`

~~~
$ vcd cse ovdc list
name                org                 k8s_provider
------------------  ------------------  --------------
standard-demo-ovdc  cse-demo-org        native
ent-demo-ovdc       cse-demo-org        ent-pks
dev-ovdc            dev-org             native
ent-dev-ovdc        enterprise-dev-org  none
base-ovdc           base-org            none
prod-ovdc           prod-org            ent-pks
~~~

## Step 2: Enable the enterprise-dev-org for Enterprise PKS k8 Cluster Creation via CSE

**2.1** Add the right that allows an org to support Enterprise PKS cluster creation to our `enterprise-dev-org`:
~~~
$ vcd right add "{cse}:PKS DEPLOY RIGHT" -o enterprise-dev-org
~~~
**2.2** Ensure you are using the correct org and then enable the OvDC to support Enterprise PKS cluster creation with the following command:
~~~
$ vcd cse ovdc enable ent-dev-ovdc -o enterprise-dev-org -k ent-pks --pks-plan "1worker" --pks-cluster-domain "pks.zpod.io"
~~~
where `-k` is the k8 provider in question, `--pks-plan` is the PKS cluster [plan](https://docs.pivotal.io/pks/1-4/installing-pks-vsphere.html#plans) CSE will reference when a user provisions a cluster in this envrionment, and `--pks-cluster-domain` is the subdomain that we'll use for the hostname for Kubernetes master API access when a cluster is created.

Note: the `1worker` plan is just 1 master/1 worker, used for cluster creation demo as it only takes about 8 minutes to deploy.

**2.3** Verify the `enterprise-dev-org` tenant and the OvDC are enabled for Enterprise PKS cluster creation:
~~~
$ vcd cse ovdc list
name                org                 k8s_provider
------------------  ------------------  --------------
standard-demo-ovdc  cse-demo-org        native
ent-demo-ovdc       cse-demo-org        ent-pks
dev-ovdc            dev-org             native
ent-dev-ovdc        enterprise-dev-org  ent-pks
base-ovdc           base-org            none
prod-ovdc           prod-org            ent-pks
~~~

## Step 3: Add PKS Cluster Deploy Rights to Role

If we look at users in the `enterprise-dev-org` in vCloud Director, we can see that two users (dev1 and dev2) have the custom `k8deploy` role while another user (dev3), has the standard `vApp Author` role:

<img src="Images/vcd-dev-users.png">

We need to add the `"{cse}:PKS DEPLOY RIGHT"` right to the `k8deploy` role in order for our dev1 and 2 users to be able to deploy k8 clusters in this org.

**3.1** Instruct the `vcd-cli` to "use" the `enterprise-dev-org`:
~~~
$ vcd org use enterprise-dev-org
~~~
**3.2** Add the Enterprise PKS cluster deploy right to the custom role:
~~~
$ vcd role add-right "k8deploy" "{cse}:PKS DEPLOY RIGHT"
~~~
**3.3** Login to the `vcd-cli` with our `dev1` user to test a cluster creation:
~~~
$ vcd login director.vcd.zpod.io enterprise-dev-org dev1 -iwp VMware1!
~~~
**Note:** There is currently a bug in CSE 2.5.1 where `--nodes` has to be defined for CSE Enterprise clusters, see command below
~~~
$ vcd cse cluster create dev1-cluster --nodes=1
property                     value
---------------------------  ------------------------
kubernetes_master_host       dev1-cluster.pks.zpod.io
kubernetes_master_ips        In Progress
kubernetes_master_port       8443
kubernetes_worker_instances  1
last_action                  CREATE
last_action_description      Creating cluster
last_action_state            in progress
name                         dev1-cluster
worker_haproxy_ip_addresses
~~~

**3.4** Use the following command to monitor the progress of the cluster creation. Wait until the `last_action_state` equals `succeeded` and you have an IP listed for the `kubernetes_master_ips` value to proceed to step 3.5:
~~~
$ vcd cse cluster info dev1-cluster
property                     value
---------------------------  ---------------------------------
k8s_provider                 ent-pks
kubernetes_master_host       dev1-cluster.pks.zpod.io
kubernetes_master_ips        <Kubernetes Master IP Address here>
kubernetes_master_port       8443
kubernetes_worker_instances  1
last_action                  CREATE
last_action_description      Instance provisioning completed
last_action_state            succeded
name                         dev1-cluster
worker_haproxy_ip_addresses
~~~

**3.5** If you'd like to access this cluster via `kubectl` on the cse server, you'll need to add the `kubernetes_master_ips` (will only be available after the cluster provision is complete) and `kubernetes_master_host` values into the `/etc/hosts` file of the cse server so the hostname will resolve correctly.

**Note**: If there is an existing entry for `dev1-cluster` in the `/etc/hosts` file, delete or change it to reflect the IP address you obtained from step 3.4

~~~
$ sudo vi /etc/hosts

#add the following line to the file
<kubernetes_master_ip>  dev1-cluster.pks.zpod.io
~~~
~~~
$ vcd cse cluster config dev1-cluster > ~/.kube/config
~~~
~~~
$ kubectl get nodes
NAME                                   STATUS   ROLES    AGE    VERSION
0faf789a-18db-4b3f-a91a-a9e0b213f310   Ready    <none>   10m    v1.13.5
~~~

**3.6** After the `dev1-cluster` has been provisioned, make sure RBAC is working as expected by logging into the org with our `dev3` user and trying to provision a cluster:
~~~
$ vcd login director.vcd.zpod.io enterprise-dev-org dev3 -iwp VMware1!
~~~
~~~
$ vcd cse cluster create rbac-test
Usage: vcd cse cluster create [OPTIONS] NAME
Try "vcd cse cluster create -h" for help.

Error: Access Forbidden. Missing required rights.
~~~
Perfect! So our `dev3` user does not have the `"{cse}:PKS DEPLOY RIGHT"` granted to their role so they aren't able to deploy clusters within the org.

## Step 4: CSE and NSX-T Integration

Whenever a PKS Kubernetes cluster is created via CSE, the CSE server reaches out directly to the NSX-T API upon cluster creation completion to create a Distributed Firewall rule to prevent any traffic from entering cluster from other clusters deployed in the environment.

**4.1** To verify this, after the `dev1-cluster` cluster creation is completed, log in to the [NSX-T manager](https://nsx.pks.zpod.io) and navigate to the **Advanced Network and Security** tab and then the **Security** > **Distrubuted Firewall Rule** tab on the left hand menu. Locate the `isolate-dev1-cluster...` DFW stanza and expand it to examine the 2 rules created:

<img src="Images/dfw-rule.png">

**4.2** Examine the rules to understand what each one specifies. The first rule (Allow cluster node-pod to cluster node-pod communication), ensures that all pods within the `dev1-cluster` can communicate with each other. The second rule (Block cluster node-pod to all-node-pod communication) ensures that pods running in other clusters (`ALL_NODES_PODS`) can not reach the pods running in the `dev1-cluster`. We can examine the target groups these rules are applied to by selecting the hyperlink for each group within the rule:

<img src="Images/source.png">

<img src="Images/target.png">

**4.3** Please delete the cluster after you finish this demo (you will use the `demo-cluster` for subsequest demo workflows):

~~~
$ vcd cse cluster delete dev1-cluster
~~~
CSE will delete the DFW rule create to isolate the cluster while the PKS control plane will handle deleting all additional NSX-T resources created to support the cluster.

**4.4** After deleting the cluster, please run the `onboarding-demo-cleanup.sh` script and clear out the hostname entry for the `dev1-cluster` to clean the environment up for the next user:

~~~
$ ./zPod-PKS-CSE-Demos/CSERBACDemo/onboarding-demo-cleanup.sh
~~~

~~~
$ sudo vi /etc/hosts

127.0.0.1   localhost localhost.localdomain localhost4 localhost4.localdomain4
::1         localhost localhost.localdomain localhost6 localhost6.localdomain6
~~~

## Conclusion

In this demo, we walked through the onboarding of a new organization that wishes to deploy Enterprise PKS clusters via the Container Service Extention, granted users within the tenant org access to provision clusters, and demoed the creation of a cluster via CSE.

We also examined the work CSE does to isolate network traffic per cluster via NSX-T DFW rules any time a cluster is created via CSE. Move on to the [next demo](https://github.com/mann1mal/zPod-PKS-CSE-Demos/tree/master/GuestbookDemo) to start to deploy applications in Enterprise PKS Kubernetes clusters.
