# Installing the Linkerd Service Mesh on Civo Managed Kubernetes

## Introduction
`k3s`, the Kubernetes service underlying the Civo Kubernetes platform, automatically bundles `Traefik` as the default ingress controller in its installation. One of the most commonly-asked questions from our community users of our managed Kubernetes service is about replacing `Traefik`, with a controller or service mesh of their choice. 

Why would you want to replace `Traefik` on your cluster? If you are looking to experiment with features of a particular service mesh is one reason (and the reason I started writing this guide!), but another one is to get native observability of inter-service communication provided by specialised service mesh applications.

This guide will cover the removal of Traefik, and replacement of it as the Ingress Controller and Service Mesh by Linkerd, using the Civo command-line interface. You can, of course, use another service mesh of your choice instead - popular choices for this are [Istio](https://istio.io/), [Consul](https://www.hashicorp.com/products/consul/service-mesh) and the Traefik-based [Maesh](https://mae.sh/). See their respective official documentation for installation instructions. You can install Maesh on your cluster with one click straight from the Civo Kubernetes App Store.

## Pre-Requisites
You will need the following to get started:
- A Civo account and access to the managed Kubernetes beta. You can [apply for the beta here](https://www.civo.com/kube100) - mention this guide in your application!
- The [Civo Command-Line tool installed](https://www.civo.com/learn/kubernetes-cluster-administration-using-civo-cli) with your API key added.
- `kubectl` [downloaded and set up](https://kubernetes.io/docs/tasks/tools/install-kubectl/).
- `kubectx` [to handle context-switching](https://github.com/ahmetb/kubectx) if you are managing multiple clusters.

## What is a Service Mesh?
Typically, a Service Mesh uses processes known as sidecars deployed alongside and containerised separately from service processes. Effectively mini-proxies, these sidecars watch traffic to and from services on your nodes, and provide visibility and control. The notable exception to this is Maesh, mentioned above, which does not function via sidecars. The broad functionality remains the same, though.

## Setting Up Our Cluster

Let's get started by creating a cluster and saving its KUBECONFIG to our ~/.kube/config file using the Civo CLI:
```
$ civo k8s create --nodes=2 --size=g2.medium --wait --save --switch
Building new Kubernetes cluster extra-spiral: Done
Created Kubernetes cluster extra-spiral in 01 min 37 sec
Merged config into ~/.kube/config and switched context to extra-spiral
```
I already have a Kubernetes cluster on my account, so the tool has *merged* my configuration. Handily, using the combination of options `--wait --save --switch` will automatically change our cluster management context to the one that was just created. If you prefer to switch cluster contexts manually, use `kubectx <cluster-name>` after you have downloaded the KUBECONFIG.

Let's see what we have running on our cluster straight out of the box. We shouldn't have anything except for what `k3s` bundles automatically:
```
$ kubectl get pods --all-namespaces
NAMESPACE     NAME                         READY   STATUS      RESTARTS   AGE
kube-system   coredns-66f496764-flbsm      1/1     Running     0           6m
kube-system   helm-install-traefik-77dx7   0/1     Completed   0           6m
kube-system   svclb-traefik-l5bff          3/3     Running     0           5m
kube-system   svclb-traefik-dwnr8          3/3     Running     0           5m
kube-system   traefik-d869575c8-hrfcv      1/1     Running     0           5m
```

The above output shows that `traefik` has been installed (the process has completed) and it is successfully running.

## Removing Traefik
We will need to remove the *job*, *deployment* and then the *service* that run Traefik on our cluster. This is done through three `kubectl` commands on our cluster. 
```
$ kubectl delete pod --namespace kube-system helm-install-traefik-77dx7
pod "helm-install-traefik-77dx7" deleted
$ kubectl delete -n kube-system deploy/traefik
deployment.extensions "traefik" deleted
$ kubectl delete -n kube-system svc traefik
service "traefik" deleted
```

Give it a minute or so to wind down the service, and run the `get pods --all-namespaces` command again:
```
$ kubectl get pods --all-namespaces
NAMESPACE     NAME                         READY   STATUS      RESTARTS   AGE
kube-system   coredns-66f496764-flbsm      1/1     Running     0          33m
```
Sweet! We have successfully uninstalled Traefik. Now to deploy the service mesh we want.

**Note:** As this guide removes a default application from a cluster, the cluster information page on your dashboard, as well as the cluster details command on the Civo CLI may still show `Traefik` as installed even after these steps. The service has been entirely removed, however.

## Installing Linkerd
I will reproduce the steps I took to install Linkerd for this guide below, but you should always refer to the [official Linkerd documentation](https://linkerd.io/2/getting-started/) for the latest way to deploy it, as updates may introduce changes.

### Installing Linkerd CLI
We will need to obtain the Linkerd Command-line interface to then install the control plane onto our Kubernetes cluster. Run the following code in your terminal on your computer: `curl -sL https://run.linkerd.io/install | sh`

On my Mac, I saw the following:
```
$ curl -sL https://run.linkerd.io/install | sh
Downloading linkerd2-cli-stable-2.6.0-darwin...
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100   623    0   623    0     0   2298      0 --:--:-- --:--:-- --:--:--  2307
100 41.1M  100 41.1M    0     0  1041k      0  0:00:40  0:00:40 --:--:-- 1090k
Download complete!

Validating checksum...
Checksum valid.

Linkerd stable-2.6.0 was successfully installed 🎉


Add the linkerd CLI to your path with:

  export PATH=$PATH:/Users/kai/.linkerd2/bin

Now run:

  linkerd check --pre                     # validate that Linkerd can be installed
  linkerd install | kubectl apply -f -    # install the control plane into the 'linkerd' namespace
  linkerd check                           # validate everything worked!
  linkerd dashboard                       # launch the dashboard

Looking for more? Visit https://linkerd.io/2/next-steps
```
These are exactly the steps we are going to take, starting with adding Linkerd to our `PATH`. On a Mac/Linux the following command will work in any shell, in Windows you will have to be using the PowerShell:
```
$ export PATH=$PATH:$HOME/.linkerd2/bin
```
Note that if you want the PATH setting to persist between sessions, you will have to add it to your shell configuration file.

Let's validate our Linkerd installation:
```
$ linkerd check --pre
kubernetes-api
--------------
√ can initialize the client
√ can query the Kubernetes API

kubernetes-version
------------------
√ is running the minimum Kubernetes API version
√ is running the minimum kubectl version

pre-kubernetes-setup
--------------------
√ control plane namespace does not already exist
√ can create Namespaces
√ can create ClusterRoles
√ can create ClusterRoleBindings
√ can create CustomResourceDefinitions
√ can create PodSecurityPolicies
√ can create ServiceAccounts
√ can create Services
√ can create Deployments
√ can create CronJobs
√ can create ConfigMaps
√ no clock skew detected

pre-kubernetes-capability
-------------------------
√ has NET_ADMIN capability
√ has NET_RAW capability

pre-linkerd-global-resources
----------------------------
√ no ClusterRoles exist
√ no ClusterRoleBindings exist
√ no CustomResourceDefinitions exist
√ no MutatingWebhookConfigurations exist
√ no ValidatingWebhookConfigurations exist
√ no PodSecurityPolicies exist

linkerd-version
---------------
√ can determine the latest version
√ cli is up-to-date

Status check results are √
```
Nice, we're ready to roll.

### Installing Linkerd onto our cluster

By default, Linkerd is installed to a namespace called `linkerd`. We are not going to be changing this, so the basic installation command will be simple:
```
$ linkerd install | kubectl apply -f -
```
The above command generates the `yaml` manifests required by `linkerd` and the pipe assigns the resulting files to be applied onto your cluster by `kubectl`.

You should see a number of roles, role bindings and deployments being created in the output.

To verify our installation is complete and `linkerd` is running, let's check the pods again:
```
$ kubectl get pods --all-namespaces
NAMESPACE     NAME                                      READY   STATUS      RESTARTS   AGE
kube-system   coredns-66f496764-flbsm                   1/1     Running     0          48m
kube-system   helm-install-traefik-77dx7                0/1     Completed   0          48m
linkerd       linkerd-identity-859ffb78db-xknxl         2/2     Running     0          2m12s
linkerd       linkerd-destination-74b988cccc-9gddg      2/2     Running     0          2m12s
linkerd       linkerd-sp-validator-c8cf79984-2rsvq      2/2     Running     0          2m11s
linkerd       linkerd-tap-67f6f4d94f-8gqjj              2/2     Running     0          2m11s
linkerd       linkerd-proxy-injector-7687b99bd7-kxrx8   2/2     Running     0          2m11s
linkerd       linkerd-grafana-74cfb44b49-j8xbp          2/2     Running     0          2m11s
linkerd       linkerd-controller-58457d4fd6-pff2n       3/3     Running     0          2m12s
linkerd       linkerd-web-57d95dcbb-kb8db               2/2     Running     0          2m12s
linkerd       linkerd-prometheus-c6fd999fc-vl45z        2/2     Running     0          2m12s
```
That's them running alright!

### Basic Linkerd Usage
Linkerd provide a great tutorial on using the service mesh on their site with an example application called `emojivoto`. All their documentation references this app, so it's a handy reference guide. [Follow the steps here](https://linkerd.io/2/getting-started/#step-5-install-the-demo-app) to deploy it and watch Linkerd do its thing.

Linkerd includes `Grafana` to display data collected by `Prometheus` that will allow you to visualise traffic and identify issues on your cluster. You can start it at any time from your terminal by running the following command, which will transparently open port-forwarding from your cluster and open your browser to the dashboard:
```
$ linkerd dashboard &
```

## Conclusion
By following this guide, you have easily removed the `Traefik` installation that is bundled by default to `k3s` installations, and installed `Linkerd` as a Service Mesh for runtime debugging, observability and reliability. If you would want to use another Service Mesh, you would just diverge from the guide after the Removing Traefik section and follow the official documentation for the Service Mesh you want.

Let us know on Twitter [@civocloud](https://twitter.com/civocloud) if you have a favourite service mesh solution - and why!