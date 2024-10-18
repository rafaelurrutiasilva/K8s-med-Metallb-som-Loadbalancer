### [🇸🇪 Svenska](README.md) / [🇬🇧 English](README_en.md)

# Kubernetes Cluster with MetalLB as LoadBalancer
## Introduction
### Background
I often create a complete lab environment on my laptop using VMware Workstation, where I set up different virtual machines to carry out test projects or implement ideas. Now, I have the opportunity to set up a standalone VMware ESXi server and was therefore forced to build a Kubernetes cluster and configure a LoadBalancer instead of using kubectl port-forward service.

### Summary
Simply following one of these articles would not have helped me achieve my goal, which was to expose the services I create in my Kubernetes cluster externally via a load balancer.

Here, I have compiled the articles I used, along with a short description of how I proceeded. Everything is mainly written to help my forgetful self remember the process—but if it can be useful to you, feel free to use it!

### Lab Environment
The lab environment consisted of a Kubernetes cluster with three nodes, where each node was a virtual machine running Ubuntu 20.04. All of this was hosted on a standalone ESXi server, version 8.

### Update
<aside> 💡 It also worked well with VMware Workstation 17 Pro on Windows 10. The only requirement was that the address pool in the LB configuration did not collide with what DHCP (in VMware Workstation) was assigning. </aside>

### References
https://metallb.universe.tf/installation

https://blog.andreev.it/2023/10/install-metallb-on-kubernetes-cluster-running-on-vmware-vms-or-bare-metal-server

https://akyriako.medium.com/load-balancing-with-metallb-in-bare-metal-kubernetes-271aab751fb8

# Tutorial
## Installation and Configuration
The installation starts with what is published on the official website and then continues with the steps detailed in article 2 in the list. We will configure kube-proxy as needed and then configure MetalLB with a load balancer using the IP address pool I require.

### Enable strict ARP mode
If you are using kube-proxy in IPVS mode, you need to enable strict ARP mode starting from Kubernetes v1.14.2. You can achieve this by editing the kube-proxy configuration in the current cluster.

```bash
kubectl get configmap kube-proxy -n kube-system -o yaml \
 |sed -e "s/strictARP: false/strictARP: true/" | kubectl apply -f - -n kube-system
```
> Note: This is not needed if you are using kube-router as the service proxy, as it enables strict ARP by default.
>
 
### Installation according to metallb.universe.tf
This will install MetalLB in your cluster, under the namespace metallb-system.

```bash
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.14.8/config/manifests/metallb-native.yaml
```
**Objects in the manifest include:**

- `metallb-system/controller deployment`. This is the cluster-wide controller that manages IP address assignments.
- `metallb-system/speaker` daemonset. This is the component that uses the protocol(s) you choose to make services reachable.
- Service accounts for the controller and speaker, along with the necessary RBAC permissions required for these components to function.
> The installation manifest does not include any configuration file. MetalLB components will start but remain inactive until you deploy resources.
>

### Checking the Status of MetalLB Core Objects
You can verify that the different objects from the manifest are running without errors.
```bash
kubectl -n metallb-system get pods
kubectl -n metallb-system get all
```

Configuring the LoadBalancer
Now we can proceed to configure an address pool for our load balancer. You should use an address range that fits your environment. The range must be reserved and not used by the DHCP in your network.
```yaml
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: first-pool
  namespace: metallb-system
spec:
  addresses:
  - 172.29.8.170-172.29.8.180           # Use what fits your environment
---
apiVersion: metallb.io/v1beta1
kind: L2Advertisement
metadata:
  name: example
  namespace: metallb-system   
```
Save this in a YAML file and then run kubectl apply -f <yaml-file>.

Testing the LoadBalancer
We will install two simple applications that we will test both internally from Worker Nodes and externally via a web browser. These applications will help us see which pod in the cluster responds to requests, which helps us verify that the load balancing works.

### App Demo1
This will start a lightweight web server that displays the name of the pod where it's running.
```bash
kubectl create deployment demo1 --image=gcr.io/google-samples/hello-app:1.0 --replicas 3
kubectl expose deployment demo1 --type LoadBalancer --port 80 --target-port 8080
```
### App Demo2
```bash
Copy code
kubectl create deployment demo2 --image=klimenta/serverip --replicas 6 --port 3000
kubectl expose deployment demo2 --type LoadBalancer --port 80 --target-port 3000
```
### Testing Demo1 & Demo2
You can run the following commands multiple times in the CLI or visit the URL shown in a web browser.

```bash
# demo1
curl $(kubectl get svc demo1 |awk '/^demo1/  {print"http://"$4}'); \
echo "Try it yourself from your web client:"; \
echo $(kubectl get svc demo1 |awk '/^demo1/  {print"http://"$4}')

# demo2
curl $(kubectl get svc demo2 |awk '/^demo2/  {print"http://"$4}'); \
echo "Try it yourself from your web client:"; \
echo $(kubectl get svc demo2 |awk '/^demo2/  {print"http://"$4}')
```

### Loop Test
Want to test with a loop in the CLI? Use this example.
```bash
for i in {1..5}
do
 curl $(kubectl get svc demo1 |awk '/^demo1/ {print"http://"$4}') ; echo "" 
done
```

## Clean Up Everything
This can be done in several ways, but the common method is using the CLI commands kubectl delete service and kubectl delete deployment.
```bash
for I in service/demo1 deployment.apps/demo1 service/demo2 deployment.apps/demo2 
do 
   kubectl delete $I
done
```
