# Ingress Controller and NSX-T

As you may remember from our [first demo](https://github.com/mann1mal/zPod-PKS-CSE-Demos/blob/master/GuestbookDemo/README.md), if we use the Kubernetes service type of `LoadBalancer` to expose an app to external connections in a PKS cluster, we would still need to manually add DNS records for each NSX-T Load Balancer IP created so that external connections could resolve on a hostname. 

If we use a service type of `Ingress` instead, in conjunction with a DNS wildcard record, we can manage hostname resolution directly at the Kubernetes layer via the Ingress controller (NSX-T layer 7 load balancer) that is automatically deployed in NSX-T upon cluster creation. This allows developers to fully manager the hostnames that resolve to multiple apps without having to create DNS records for each service they are looking to expose externally.

### Accessing the `demo-cluster`

Before starting the demo, access the `cse-client` server with the `cse` user (`cse@cse-client.vcd.zpod.io`) from your Horizon instance via putty (pw is `VMware1!`):

<img src="Images/putty-ss.png">

Ensure you are accessing the `demo-cluster` via kubectl by using the `cse` CLI extension to pull down the cluster config file and store it in the default location, if you haven't done so in a previous lab. Use the `cse-ent-user` with password `VMware1!` to log in to the `vcd-cli`:

~~~
$ vcd login director.vcd.zpod.io cse-demo-org cse-ent-user -iw
~~~
~~~
$ vcd cse cluster config demo-cluster > ~/.kube/config
~~~
~~~
$ kubectl get nodes
NAME                                   STATUS   ROLES    AGE     VERSION
0faf789a-18db-4b3f-a91a-a9e0b213f310   Ready    <none>   5d9h    v1.13.5
713d03dc-a5de-4c0f-bbfe-ed4a31044465   Ready    <none>   5d10h   v1.13.5
8aa79ec7-b484-4451-aea8-cb5cf2020ab0   Ready    <none>   5d10h   v1.13.5
~~~

Ensure the context is set to the `appspace` namespace:
~~~
$ kubectl config set-context --current --namespace=appspace
~~~

## Prereqs

The `guestbook` app is deployed in your cluster, as detailed in the [Guestbook Demo](https://github.com/mann1mal/zPod-PKS-CSE-Demos/tree/master/GuestbookDemo) lab

## Step 1: Configure Ingress Service for Guestbook App

**1.1** After performing the [first demo](https://github.com/mann1mal/zPod-PKS-CSE-Demos/blob/master/GuestbookDemo/README.md), you'll need to change the `frontend` service from `LoadBalancer` type to `ClusterIP` type by editing the service via `kubectl` and making the changes notated below to the `spec` section (Note: your `ClusterIP` may be different than the one in the screenshot):
~~~
$ kubectl edit svc frontend
~~~

**Note**: `type: ClusterIP` should be the last line in the file. Also note, `kubectl edit` utilizes `vi` as it's editor by default. After opening the yaml file for the serivce, hit `i` to enter insert mode, then make your changes. Hit `esc` to leave insert mode and enter `:wq` then `enter` to commit your changes.

<img src="Images/clusterip-service.png">

**1.2** Verify the "frontend" service is now type "ClusterIP"
~~~
$ kubectl get svc
~~~
**Note:** Because we changed the "frontend" service to `ClusterIP` type, the NSX Container Plugin (NCP) will automatically delete the virtual server created within the NSX-T L4 load balancer to support the `Loadbalancer` servce. You can verify this in the NSX-T Management Portal.

**1.3** Create the ingress service to expose the guestbook app via FQDN. In the lab environment, we have set up a DNS wildcard that resolves `*.demo.pks.zpod.io` to the IP address of the NSX-T load balancer that is automatically created by PKS to serve as the ingress controller.

Navigate to the `~/zPod-PKS-CSE-Demos/Ingress\&NSX-T/` directory on the cse-client server:
~~~
$ cd ~/zPod-PKS-CSE-Demos/Ingress\&NSX-T/
~~~

**1.4** Review the frontend-ingress.yaml file and note the `host:` entry. This is the hostname (`guestbook.app.pks.zpod.io`) the ingress controller will redirect to the service offering up the web UI (`frontend` service) for our guestbook portal. Create the ingress controller from the frontend-ingress.yaml file in the guestbook directory and verify the hostname of the app is accessible via the Web UI:
~~~
$ kubectl create -f guestbook-ingress.yaml 
~~~
~~~
$ kubectl get ingress
NAME               HOSTS                        ADDRESS                     PORTS   AGE
guestbook-ingress   guestbook.demo.pks.zpod.io   10.96.59.100,100.64.32.15   80      32m
~~~
**1.5** Navigate to the URL displayed in the output of the above command to verify connectivity:

<img src="Images/guestbook-page.png">

**1.6** Navigate to the NSX-T Manager to view the L7 load balancer serving as our ingress controller for this cluster. Navigate to https://nsx.pks.zpod.io/ and login with the `audit` user's credentials (`audit/VMware1!VMware1!`).

Navigate to the **Advanced Networking and Security** tab. Navigate to **Load Balancing** in the left hand menu and choose the **Server Pools** tab on the right side of the UI. Here, we have (at least) 2 NSX-T load balancers per k8 cluster. The UUID of the demo-cluster is `6e92c1a9-c8f2-4774-ba8b-7786e7fc8d50`. NSX-T assigns the UUID of the cluster to each load balancer it provisions for said. Locate the `pks-6e92c1a9-c8f2-4774-ba8b-7786e7fc8d50-default...` server pool, and click on the integer in the **Members/NSGroups** section:

<img src="Images/frontend-L7.png">

**1.7** Verify these are the pod names of the pods that are serving out our frontend web UI for the guestbook app via the CLI of the `cse-client` server:
~~~
$ kubectl get pods -l tier=frontend,app=guestbook
NAME                        READY   STATUS    RESTARTS   AGE
frontend-7fb9745b88-8nx5k   1/1     Running   0          7m11s
frontend-7fb9745b88-9zx6r   1/1     Running   0          7m11s
frontend-7fb9745b88-gcjfd   1/1     Running   0          7m11s
~~~

## Step 2: Add a Second App (and Ingress Resource) to the Cluster

For our second scenario, our developers want to deploy an additional app in the cluster and expose the UI of the application to external users via an FQDN. Instead of reaching out to the infrastructure team to request a DNS A record creation for the new app, they can use the ingress controller provided to the cluster by NSX-T to serve out the app via FQDN.

**2.1** On the `cse-client` server, let's navigate to the `~/zPod-PKS-CSE-Demos/Ingress\&NSX-T/` directory and review the `yelb-ingress.yaml` configuration file which houses the configuration of our new application components as well as an ingress resource.
~~~
$ ~/zPod-PKS-CSE-Demos/Ingress\&NSX-T/
$ less yelb-ingress.yaml
~~~

**2.2** Deploy the yelb application:
~~~
$ kubectl create -f yelb-ingress.yaml
~~~
**2.3** Review the resources created to support the app:
~~~
$ kubectl get all -l app=yelb-appserver
$ kubectl get all -l app=yelb-ui
$ kubectl get all -l app=yelb-db
~~~
**2.4** Review the ingress resources available in the environment:
~~~
$ kubectl get ingress
NAME               HOSTS                        ADDRESS                     PORTS   AGE
guestbook-ingress  guestbook.demo.pks.zpod.io   10.96.59.100,100.64.32.15   80      87m
yelb-ingress       yelb.demo.pks.zpod.io        10.96.59.100,100.64.32.15   80      75m
~~~
Note, `10.96.59.100` is the same public IP for both hostnames. This is the IP of the L7 NSX-T load balancer that acts as the ingress controller for the cluster.

**2.5** Navigate to the hostname of our new app to ensure it is available:

<img src="Images/yelb-page.png">

**2.6** Voila! Now let's navigate back to the NSX-T manager to see what's happening on the NSX-T side. Again, navigate to the **Advanced Networking and Security** tab. Navigate to **Load Balancing** in the left hand menu and choose the **Server Pools** tab on the right side of the UI. We should now see a new server pool entry ending in `...default-yelb-ui-80` with our `yelb-ui` pod name listed in the **Members/NSGroups** column:

<img src="Images/yelb-L7.png">

~~~
$ kubectl get pods -l app=yelb-ui
NAME                      READY   STATUS    RESTARTS   AGE
yelb-ui-fc74d567f-vh2qb   1/1     Running   0          96m
~~~

There is now an additional server pool within our virtual server tied to our L7 loadbalancer that will except request for the yelb UI at `yelb.demo.pks.zpod.io` and direct those connections to the `yelb-ui-fc74d567f-vh2qb` pod while the existing server pool will continue to direct requests coming in at `guestbook.demo.pks.zpod.io` to the `frontend-xxx` pods.

## Step 3 (Optional): Cleanup

 **3.1** If continuing on to the next lab, leave the `guestbook` and `yelb` apps running in the cluster. If you are not proceeding, please delete the applications and their resources:
~~~
$ kubectl delete -f ~/zPod-PKS-CSE-Demos/Guestbook/guestbook-aio-clusterip.yaml
$ kubectl delete -f ~/zPod-PKS-CSE-Demos/Guestbook/redis-master-claim.yaml
$ kubectl delete -f ~/zPod-PKS-CSE-Demos/Guestbook/redis-slave-claim.yaml
$ kubectl delete ingress guestbook-frontend
~~~
~~~
$ kubectl delete -f ~/zPod-PKS-CSE-Demos/Ingress\&NSX-T/yelb-ingress.yaml
~~~

**3.2** Log back in to the vCloud Director environment via the `vcd-cli` and pull a fresh `demo-cluster` config file to reset cluster contexts:

~~~
$ vcd login director.vcd.zpod.io cse-demo-org <username> -iw
~~~
~~~
$ vcd cse cluster config demo-cluster > ~/.kube/config
~~~

## Conclusion

So there we have it, the integration between NSX-T and Enterprise PKS provides a resource for our developers to easily expose their applications to external users via an FQDN without having to work with the infrastructure team to create resources to support access every time they spin up a new application.

In our [next demo](https://github.com/mann1mal/zPod-PKS-CSE-Demos/tree/master/NetworkPolicy), we will take a deeper dive into the integration between PKS and NSX-T to showcase the automatic creation of DFW rules to allow for microsegmentation of Kubernetes workloads.
