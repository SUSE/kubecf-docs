---
title: "Deploy KubeCF on Kind"
date: 2020-02-28
weight: 3
description: >
  Step-by-step guide to deploy KubeCF on top of Kind
---

## Before you begin

This guideline will provide a quick way to deploy KubeCF with Kind and should be used only for evaluation purposes.

Here is a list of all the tools and versions used when creating these instructions:

```
> kind version
kind v0.7.0 go1.13.6 darwin/amd64
```

```
> helm version
version.BuildInfo{Version:"v3.1.1", GitCommit:"afe70585407b420d0097d07b21c47dc511525ac8", GitTreeState:"clean", GoVersion:"go1.13.8"}
```

```
> docker version
Client: Docker Engine - Community
 Version:           19.03.5
 API version:       1.40
 Go version:        go1.12.12
 Git commit:        633a0ea
 Built:             Wed Nov 13 07:22:34 2019
 OS/Arch:           darwin/amd64
 Experimental:      false
```

```
> kubectl version
Client Version: version.Info{Major:"1", Minor:"17", GitVersion:"v1.17.3", GitCommit:"06ad960bfd03b39c8310aaf92d1e7c12ce618213", GitTreeState:"clean", BuildDate:"2020-02-13T18:08:14Z", GoVersion:"go1.13.8", Compiler:"gc", Platform:"darwin/amd64"}
```

```
> cf version
cf version 6.46.1+4934877ec.2019-08-23z
```

## Installing Kind

To install Kind please follow the official instructions [here](https://kind.sigs.k8s.io/docs/user/quick-start/).

## Create a Kind cluster

Run the following commands:
``` 
> export KUBECONFIG=kubeconfig-kubecf
> kind create cluster --name kubecf
> kubectl cluster-info --context kind-kubecf
```

## Installing cf-operator

Before we can deploy the [cf-operator](https://github.com/cloudfoundry-incubator/cf-operator) we need to create the namespace:

```
> kubectl create ns cfo
```

and after we can install by running the helm command:

```
> helm install cf-operator \
    --namespace cfo \
    --set "global.operator.watchNamespace=kubecf" \
    https://s3.amazonaws.com/cf-operators/helm-charts/cf-operator-v2.0.0-0.g0142d1e9.tgz
```

Notes:

1. The *watchNamespace* property is set to watch the **kubecf** namespace for changes
2. cf-operator version may differ between KubeCF versions

Check if the pods are up and running before moving to the next section:

```
> kubectl get pods -n cfo
```

## Installing KubeCF

First let's get Kind node IP address:

```
node_ip=$(kubectl get node kubecf-control-plane \
  --output jsonpath='{ .status.addresses[?(@.type == "InternalIP")].address }')
```

and then set the properties correctly:
```
cat << _EOF_  > values.yaml
system_domain: ${node_ip}.nip.io
services:
  router:
    externalIPs:
    - ${node_ip}
kube:
  service_cluster_ip_range: 0.0.0.0/0
  pod_cluster_ip_range: 0.0.0.0/0
_EOF_
```

On this example, we will use Diego instead Eirini but you can easily switch by adding the following 
lines into your **values.yaml** file:

```
...
features:
  eirini:
    enabled: true
...
```

and, you need to trust the kubernetes root CA on the kind docker container:

```
> docker exec -it "kubecf-control-plane" bash -c 'cp /etc/kubernetes/pki/ca.crt /etc/ssl/certs/ && update-ca-certificates && (systemctl list-units | grep containerd > /dev/null && systemctl restart containerd)'
```

the **values.yaml** file should be similiar to the snippet:

```
system_domain: 172.17.0.3.nip.io

services:
  router:
    loadBalancerIP:
    - 172.17.0.3

kube:
  service_cluster_ip_range: 0.0.0.0/0
  pod_cluster_ip_range: 0.0.0.0/0
```

Now is time to install KubeCF by running the helm command:

```
> helm install kubecf \
    --namespace kubecf \
    --values values.yaml \
https://github.com/cloudfoundry-incubator/kubecf/releases/download/v0.2.0/kubecf-0.2.0.tgz
```

Notes:

1. the namespace property value is the same as cf-operator watchNamespace one

Be aware that it takes a couple of minutes to see the pods showing up on the kubecf namespace and the installation process may take 20-25 minutes depending on your
internet connection speed.

Run the following command to watch the pods progress:

```
> watch kubectl get pods -n kubecf
```

After all the pods are running you can check by running the *cf* cli command:

```
> cf api --skip-ssl-validation api.172.17.0.3.nip.io
```

get the admin password:

```
> admin_pass=$(kubectl get secret --namespace kubecf kubecf.var-cf-admin-password -o jsonpath='{.data.password}' | base64 --decode)
```

and login with: `cf auth admin "${admin_pass}"`

## What's next

After the deployment finishes with success it's time to give it a try by pushing an app using the cf-push cli command.

## Cleaning up

```
> kind delete cluster --name kubecf
```
