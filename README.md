# flux-local-minikube
A repository that provides an introduction to using [Flux](https://docs.fluxcd.io/en/1.21.1/introduction/) on a Kubernetes (K8s) cluster to install/maintain core components.

## Table of Contents
* [Project Directory](#Project-Directory)
* [Requirements](#Requirements)
* [Installation](#Installation)
  * [1. Prepare Deployment Files](#1-Prepare-Deployment-Files)
  * [2. Create Kong Secrets](#2-Create-Kong-Secrets)
  * [3. Install Flux & Helm Operator](#3-Install-Flux-&-Helm-Operator)
  * [4. (Alternatively) Deploy Jenkins & Kong with Kustomize](#4-(Alternatively)-Deploy-Jenkins-&-Kong-with-Kustomize)
* [Commiting Changes for Flux](#Commiting-Changes)
* [Jenkins](#Jenkins)
* [Kong](#Kong)
* [Helpful Links](#Helpful-Links)

## Project Directory
* **/:** The root of this repository holds the `kustomization.yaml` which defines the base deployments to be installed on the cluster.
  * **/base:** This folder contains the core components which will be created and maintained by Flux.
    - **/echo:** The Kubernetes manifests for the echo service.
    - **/jenkins:** The Kubernetes manifests for Jenkins.
    - **/kong:** The HelmRelease for Kong.
    - **/namespaces:** The Kubernetes namespaces to be created.
  * **/flux:** This folder contains the deployment of Flux and the Helm Operator, which are created by Kustomize.

## Requirements
In order to use this Flux repository, we need to spin up a new Kubernetes cluster on our local machine using `minikube`. [This guide](https://github.com/massmutual/intro-to-kubernetes) created by Jon Ellenberger goes over the core concepts of Kubernetes along with [installation instructions](https://github.com/massmutual/intro-to-kubernetes#installing-minikube) for `minikube`.

****It is recommended to disconnect from the VPN while starting minikube to avoid networking issues.***

The remainder of this guide will assume you have a local Kubernetes cluster running and are able to interact with it using `kubectl` commands. This guide will use [Kustomize](https://github.com/kubernetes-sigs/kustomize), which was added to `kubectl` in v1.14. To check the version, run `kubectl version`.

We'll also need [fluxctl](https://docs.fluxcd.io/en/1.21.1/references/fluxctl/) to obtain the SSH key requried our instance of Flux to communicate with the Git repository. You can install it by running:
```
brew install fluxctl
```

## Installation
Before getting started, you'll need to fork this repository, clone the forked repo with git, and change into the project directory:
```
$ git clone https://github.com/<YOUR-GITHUB-USER>/flux-local-minikube.git
$ cd flux-local-minikube
```
Using a fork is required to be able to create a Deploy Key and to test making changes to the Flux repo.

### 1. Prepare Deployment Files
To start, we'll need to fill in some values in our base deployment files. In the `/flux/patch.yaml` file, replace `<YOUR-GITHUB-USER>` with your Github username used to create the fork of this repository. This will ensure Flux is pointing to the correct repo to sync the cluster state.

Next, run the following command to get the local IP that we can use to access `minikube`:
```
minikube ip
```
In the `base/kong/kong.yaml` file, use this IP address to replace `<MINIKUBE-IP>` on lines 24-27. This will ensure that our Kong components are able to communicate with each other upon startup.

**Commit these changes into your fork before continuing so that Flux will sync the correct values once deployed.**

### 2. Create Kong Secrets
Kong will not install properly unless the following secrets are present in the `kong` namespace. Eventually, we can utilize Vault and have Flux create these secrets automatically. For now, we will create these secrets manually.
* `kong-enterprise-license`
* `kong-enterprise-superuser-password`
* `kong-session-config`

First, create the `kong` namespace:
```
kubectl create namespace kong
```
Next, run the following commands to create the required secrets. Note that the `kong-enterprise-license` secret requires a file called `license.json` to be present in the current directory. Contact the API Platform team for help obtaining this license.
```
kubectl create secret generic -n kong kong-enterprise-license --from-file=license=./license.json

kubectl create secret generic -n kong kong-enterprise-superuser-password --from-literal=password=admin

echo '{"cookie_name":"admin_session","cookie_samesite":"off","secret":"<your-password>","cookie_secure":false,"storage":"kong"}' > admin_gui_session_conf

echo '{"cookie_name":"portal_session","cookie_samesite":"off","secret":"<your-password>","cookie_secure":false,"storage":"kong"}' > portal_session_conf

kubectl create secret generic -n kong kong-session-config --from-file=admin_gui_session_conf --from-file=portal_session_conf
```


### 3. Install Flux & Helm Operator
Now we're ready to start deploying to K8s. This first step will utilize Kustomize to deploy Flux and the Helm Operator. Run the following command to apply the Resources declared in `flux/kustomization.yaml`:
```
kubectl apply --kustomize flux
```
Once deployed, you should see all the running components in the `flux` namespace:
```
kubectl get all -n flux
```
Now, we're ready to sync our cluster state with the Flux Git repository, which will create all the resources incuded in our `base` directory and continiously poll for changes. Run the following `fluxctl` command to obtain the SSH key generated by Flux at startup:
```
fluxctl identity --k8s-fwd-ns flux
```
Copy this public key and create a deploy key with write access on your forked repository. To do this, open GitHub, navigate to your fork, go to **Setting > Deploy keys**, click on **Add deploy key**, give it a `Title`, check **Allow write access**, paste the Flux public key and click **Add key**.

Once completed, Flux should automatically pick up the state of the cluster and begin deploying the base images that are not yet present on the cluster (Jenkins and Kong).

As configured in this example, the Flux git pull frequency is set to 1 minute. You can tell Flux to sync changes immediately with:
```
fluxctl sync --k8s-fwd-ns flux
```

Congratulations! You've just installed Flux on a fresh Kubernetes cluster to automatically install and maintain the core deployments of Jenkins and Kong. Now, any changes made to the Flux repository should be picked and applied automatically by Flux running in K8s. We'll test this in the following sections.

### 4. (Alternatively) Deploy Jenkins & Kong with Kustomize
The same way we deployed Flux itself, we can utilize `Kustomize` to deploy the deployments in the `base` directory ourselves if we wanted to. Run the following command to apply the Resources declared in `/kustomization.yaml`:
```
kubectl apply --kustomize .
```

## Commiting Changes
In this section, we'll push some small changes to our Flux repo and observe how it is picked up and applied to the cluster. First, we can confirm that the `demo` namespace was created by Flux:
```
kubectl get ns
```
If we get all the resources, we can see there are none in the `demo` namespace, as expected.
```
kubectl get all -n demo
```
Even though we have the file `base/echo/echo.yaml` present in our `base` directory, we've indicated Flux to ignore these resources through the annotation:
```
annotations:
    fluxcd.io/ignore: "true"
```
Now, we will change this annotation on the Service and Deployment manifests declared in this file to have Flux deploy these resources.
```
annotations:
    fluxcd.io/ignore: "false"
```
Commit and push these changes to the Flux repository. Soon, we should see the `demo` namespace populated with the `echo` Service and Deployment. We can get all the resources to confirm that our app has been deployed:
```
kubectl get all -n demo
```
Now that Flux is syncing this deployment with the repository, any changes we make to it will be picked up by Flux and automatically applied to the app. We can test this out by adding an annotation to the `echo` Service. For example:
```
annotations:
    demo-annotation: testValue
```
Once this is pushed to the repo and picked up by Flux, we can describe the Service to see the annotations:
```
kubectl describe service echo -n demo
```
## Jenkins
This repository contains a base installation of Jenkins which uses a PersistentVolume resource on the Kubernetes cluster. While not recommended for a cloud deployment, this deployment should still allow you test Jenkins builds and pipelines on your local machine.

To access Jenkins we can use the NodePort service deployed in the `jenkins` namespace:
```
kubectl get service jenkins -n jenkins

NAME      TYPE       CLUSTER-IP      EXTERNAL-IP   PORT(S)          AGE
jenkins   NodePort   10.98.213.111   <none>        8080:30645/TCP   64m
```
The port we need is after `8080:` and will be different than the one in the output above. You can access Jenkins in your browser using the `minikube ip` and this NodePort, for example http://192.168.64.43:30645/.
For more information on getting started with Jenkins, visit the [Jenkins documentation](https://www.jenkins.io/doc/book/installing/kubernetes/#access-jenkins-dashboard).

## Kong
Since we injected Kong with the `minikube ip` before it was deployed, our instance of Kong should be ready to use without any additional changes.

Run the following command to view all the Kong services in the `kong` namespace:
```
kubectl get services -n kong
```
This deployment specified the NodePort to use for each component. To access Kong Manager, use the minikube IP and the NodePort for the `kong-kong-kong-manager` service, which is `32445`, for example http://192.168.64.43:32445/. You should see the login page for Kong Manager.

If you have trouble logging in using `kong_admin` and the superuser password we configured earlier, it may because your browser warns of a privacy issue accessing the Admin API. To fix this, visit the Admin API at NodePort `32444` in your browser and allow the connection. Now, you can try logging in again.

***You will need to disconnect from the VPN while running the following `minikube tunnel` command to avoid networking issues.***

This deployment of Kong also has the `kong-kong-kong-proxy` service as a LoadBalancer. Currently, the `EXTERNAL-IP` of this service will show `<pending>`. To populate this with an IP, in a separate Terminal window, run the following command:
```
minikube tunnel
```
You may get promted for your machine's password. Once entered, you can get the Kong Proxy service again to see the populated IP:
```
kubectl get service kong-kong-kong-proxy -n kong

NAME                   TYPE           CLUSTER-IP       EXTERNAL-IP      PORT(S)                      AGE
kong-kong-kong-proxy   LoadBalancer   10.105.137.168   10.105.137.168   80:31562/TCP,443:32724/TCP   94m
```
We can visit this IP directly to access any Kong Routes we create, for example, https://10.105.137.168/rest/echo.

For more information on accessing apps on minikube, visit the [minikube documentation](https://minikube.sigs.k8s.io/docs/handbook/accessing/).
## Helpful Links
Follow these links for more details about all the resources used in this project.
### Kubernetes & minikube
* [Intro to Kubernetes by Jon Ellenberger](https://github.com/massmutual/intro-to-kubernetes)
* [Accessing apps on minikube](https://minikube.sigs.k8s.io/docs/handbook/accessing/)
### Flux
* [Introducing Flux](https://docs.fluxcd.io/en/1.21.1/introduction/)
  * [How to bootstrap Flux using Kustomize](https://docs.fluxcd.io/en/1.21.1/tutorials/get-started-kustomize/)
### Kustomize
* [Overview of Kustomize](https://kubernetes.io/docs/tasks/manage-kubernetes-objects/kustomization/)
* [Kustomize Examples](https://github.com/kubernetes-sigs/kustomize/tree/master/examples)
* [kustomization File Reference](https://kubectl.docs.kubernetes.io/references/kustomize/glossary/#kustomization)
### Jenkins
* [Installing Jenkins on Kubernetes](https://www.jenkins.io/doc/book/installing/kubernetes/)