# Crossover

[![CircleCI](https://circleci.com/gh/mumoshu/crossover.svg?style=svg)](https://circleci.com/gh/mumoshu/crossover)

[![dockeri.co](https://dockeri.co/image/mumoshu/crossover)](https://hub.docker.com/r/mumoshu/crossover)

`Crossover` is **the minimal and sufficient xDS for [Envoy](https://www.envoyproxy.io/)** that helps Canary Deployment, without sacrificing Envoy's rich feature set.

## Features

- **Minimal Dependencies**
- **Gradual Migration**
- **Understandable**
- **Feature Complete**: Access to every feature Envoy provides. `crossover` makes no leaky abstraction on top.

### Minimal Dependencies

Depends only on the Go standard library and [go-yaml](https://github.com/go-yaml/yaml).

No need for `kubectl`, `client-go`, `apimachineary` or even Envoy's `go-control-plane`.

### Gradual Migration

You always start with a standard Envoy without `crossover`. If you're happy with that, keep using it and don't bother with adding additional moving part to the mix.

Canary deployments with [Flagger](https://github.com/weaveworks/flagger)?

Reconfigure Envoy without time-consuming and unreliable reloads?

Enter `crossover`.

Add `crossover` as a sidecar container to your Envoy. Update configmaps and trafficsplist via kubectl or helm to load changes into Envoy in near-realtime. That's all you need to get started, really!

### Understandable

No gRPC, REST server or serious K8s controller to maintain and debug.

`crossover` is a simple golang program that feeds only necessary parts of the config to Envoy via xDS.

It fetches well-known K8s `ConfigMap` and SMI `TrafficSplit` resources via Kubernetes' REST API, write config files for Envoy, rename the files so that Envoy can atomically reconfigure itself.

From Envoy's perspective, there's just xDS data stored at `/srv/runtime/current/*.yaml` visible from within Envoy, that are read from Envoy's standard `local file` config-source.

### Feature Complete

Access to every feature Envoy provides. `crossover` makes no leaky abstraction on top of Envoy.

You write regular Envoy config files. `crossover` just merges and syncs it to Envoy for you. 

## Design

```

[ Envoy ]--reads-->[ xDS YAML file ]<--writes--[ Crossover ]--reads-->[ ConfigMap, TrafficSplit ]<--write--[ Kubectl, Helm, etc. ]
```

[Envoy](https://www.envoyproxy.io/) is able to reconfigure itself at runtime by reading local files or by querying one or more management servers called xDS servers.

`crossover` is an implementation of xDS that translates configs stored in Kubernetes to Envoy primitives.

It is an init container and a sidecar container for Envoy, deployed to your Kubernetes cluster along with Envoy, serving xDS.

It continuously reads Kubernetes ConfigMap and [SMI TrafficSplit](https://github.com/deislabs/smi-spec/blob/master/traffic-split.md) resources, to generate and feed xDS resources to Envoy via the local filesystem.

As it relies solely on the local filesystem to communicate with Envoy, it is transparent to the operator. You don't need any experience in gRPC, which is typically required for managing famous xDS implementations. Just `kubectl exec` into the Envoy pod and `cat` standard Envoy config files generated by the loader to debug. 

Thanks to Envoy's inotify support and Kubernetes' Watch API, any changes made via ConfigMap and TrafficSplit are still recongnizable by Envoy in near real-time.

## Why not...

[Why not use configmap volumes?](https://github.com/mumoshu/crossover#why-not-use-configmap-volumes) Because it doesn't apply changes to the local disk in [a way Envoy wants](https://www.envoyproxy.io/docs/envoy/latest/intro/arch_overview/operations/runtime.html).

Why not write a gRPC server that translates SMI resources to xDS resources? Because I don't want to introduce a leaky abstraction.

## Use-cases

- Ingress Gateway
- Canary Releases
- In-Cluster Router/Load-Balancer

### Ingress Gateway

Turn [stable/envoy](https://github.com/helm/charts/tree/master/stable/envoy) chart into a dynamically configurable API Gateway, Ingress Gateway or Front Proxy

### Canary Releases

Do weighted load-balancing and canary deployments with zero Envoy restart, redeployment and CRD. [Just distribute RDS(Route Discovery Service) files via configmaps!](https://www.envoyproxy.io/learn/incremental-deploys#weighted-load-balancing).

### In-Cluster Router/Load-Balancer

Wanna add retries, circuit-breakers, tracing, metrics to your traffic? Deploy Envoy paired with `crossover` in front of your apps. No need for service meshes from day 1.

## What's this?

`crossover` is an init-container AND a sidecar for your Envoy proxy to use K8s ConfigMaps as xDS backend.

This works by loading kvs defined within specified configmap(s) and writing files assuming the key is the filename and the value is the content.

You then point your Envoy to read xDS from the directory `/srv/runtime/*.yaml`.

`crossover` writes files read from configmap(s) into the directory, triggers [symlink swap](https://www.envoyproxy.io/docs/envoy/latest/configuration/operations/runtime#updating-runtime-values-via-symbolic-link-swap)
 so that Envoy finally detects and applies changes. 
 
 ## Why not use configmap volumes?
 
You may [already know that K8s supports mounting configmaps as container volumes out of the box](https://kubernetes.io/docs/tasks/configure-pod-container/configure-pod-configmap/#add-configmap-data-to-a-volume).

The downside of using that feature to feed Envoy xDS files is that it takes 1 minute(default, configurable via kubelet `--sync-interval`) a change is reflected to the volume.

And more importantly, Envoy is unable to detect changes made in configmap volumes due to that it relies on `inotify` `MOVE` events to occur, where configmap volume changes only trigger the below events:

```
root@envoy-675dc8d98b-tvw9b:/# inotifywait -m /xds/rds.yaml
Setting up watches.
Watches established.
/xds/rds.yaml OPEN
/xds/rds.yaml ACCESS
/xds/rds.yaml CLOSE_NOWRITE,CLOSE
/xds/rds.yaml ATTRIB
/xds/rds.yaml DELETE_SELF
```

So in nutshell, `crossover` is the minimal and sufficient companion to actually distribute xDS data via configmaps, without using more advanced CRD-based solutions like Istio and VMWare Contour.

## Usage

```console
$ crossover -h
Usage of ./crossover:
  -apiserver string
    	K8s api endpoint (default "https://kubernetes")
  -configmap value
    	the configmap to process.
  -dry-run
    	print processed configmaps and secrets and do not submit them to the cluster.
  -insecure
    	disable tls server verification
  -namespace string
    	the namespace to process.
  -onetime
    	run one time and exit.
  -output-dir string
    	Directory to putput xDS configs so that Envoy can read
  -smi
    	Enable SMI integration
  -sync-interval duration
    	the time duration between template processing. (default 1m0s)
  -token-file string
    	path to serviceaccount token file (default "/var/run/secrets/kubernetes.io/serviceaccount/token")
  -trafficsplit value
    	the trafficsplit to be watched and merged into the configmap
  -watch
    	use watch api to detect changes near realtime
```

## Getting Started

### ConfigMap-Only Mode

Try weighted load-balancing using `crossover`!

Deploy Envoy along with the loader using the `stable/envoy` chart:

```
helm repo add stable https://kubernetes-charts.storage.googleapis.com
helm upgrade --install envoy stable/envoy -f example/values.yaml -f example/values.services.yaml
```

Or by using `crossover/envoy` chart:

```
helm repo add crossover https://mumoshu.github.com/crossover
helm upgrade --install envoy crossover/envoy -f example/values.upstreams.yaml
```

Then install backends - we use @stefanprodan's awesome [podinfo](https://github.com/stefanprodan/podinfo):

```
helm repo add flagger https://flagger.app
helm upgrade --install bold-olm flagger/podinfo --set canary.enabled=false
helm upgrade --install eerie-octopus flagger/podinfo --set canary.enabled=false
```

In another terminal, run the tester pod to watch traffic shifts:

```
kubectl run -it --rm --image alpine:3.9 tester sh

apk add --update curl
watch curl http://envoy:10000
```

Finally, try changing load-balancing weights instantly and without restarting Envoy at all:

```
# 100% bold-olm
helm upgrade --install envoy stable/envoy -f example/values.yaml -f example/values.services.yaml \
  --set services.podinfo.backends.eerie-octopus-podinfo.weight=0 \
  --set services.podinfo.backends.bold-olm-podinfo.weight=100

# 100% eerie-octopus
helm upgrade --install envoy stable/envoy -f example/values.yaml -f example/values.services.yaml \
  --set services.podinfo.backends.eerie-octopus-podinfo.weight=100 \
  --set services.podinfo.backends.bold-olm-podinfo.weight=0
```

See [example/values.yaml](example/values.yaml) for more details on the configuration.

### ConfigMap + SMI TrafficSplit Mode

The setup is mostly similar to that for CofigMap-only mode.

Just add `services.podinfo.smi.enabled=true` while installing Envoy:

```
helm upgrade --install envoy stable/envoy \
  -f example/values.yaml -f example/values.services.yaml \
  --set services.podinfo.smi.enabled=true
```

Or with the `crossover/envoy` chart:

```
helm repo add crossover https://mumoshu.github.com/crossover
helm upgrade --install envoy crossover/envoy -f example/values.services.yaml \
  --set services.podinfo.smi.enabled=true
```

Or to be extra sure, you can do it wihout `example/values.services.yaml`:

```
helm upgrade --install envoy stable/envoy   --namespace test   -f example/values.yaml   -f <(cat <<EOF
services:
  podinfo:
    smi:
      enabled: true
    backends:
      eerie-octopus-podinfo:
        port: 9898
        weight: 50
      bold-olm-podinfo:
        port: 9898
        weight: 50
EOF
)
```

Now you are ready to change weights by creating and modifying TrafficSplit like this:

```
apiVersion: split.smi-spec.io/v1alpha2
kind: TrafficSplit
metadata:
  name: podinfo
spec:
  # The root service that clients use to connect to the destination application.
  service: podinfo
  # Services inside the namespace with their own selectors, endpoints and configuration.
  backends:
  - service: eerie-octopus-podinfo
    weight: 0
  - service: bold-olm-podinfo
    weight: 100
```

Under the hood, `crossover` reads `podinfo` trafficsplit and `envoy-xds` configmap, merges the trafficsplit into the configmap to produce the final configmap `envoy-xds-gen`. It is `envoy-xds-gen` which is loaded into `envoy`. 

For convenience, there are several manifest files each with different set of weights:

```
# 100% bold-olm
kubectl apply -f podinfo-v0.trafficsplit.yaml

# 75% bold-olm 25% eerie-octopus
kubectl apply -f podinfo-v1.trafficsplit.yaml

#...

# 0% bold-olm 100% eerie-octupus
kubectl apply -f podinfo-v4.trafficsplit.yaml
```

## Developing

Bring your own K8s cluster, move to the project root, and run the following commands to give it a ride:

```
sudo mkdir /srv/runtime
sudo chmod -R 777 /srv/runtime
k get secret -o json $(k get secret | grep default-token | awk '{print $1 }') | jq -r .data.token | base64 -D > mytoken
export APISERVER=$(k config view --minify -o json | jq -r .clusters[0].cluster.server)
make build && ./crossover --namespace default --token-file ./mytoken --configmap incendiary-shark-envoy-xds --onetime --insecure --apiserver "http://127.0.0.1:8001"
```

## FAQ

### Why is the init container needed?

`crossover` needs to be included in your pod not only as a sidecar but also as an init container.

That's because Envoy checks for the existence of xDS-managed config files on startup. That is, if any of xDS-managed config files that are
referenced from Envoy's bootstrap config is missing on startup, Envoy fails starting.

Once Kubernetes adds [the first-class support for sidecar containers](https://github.com/kubernetes/enhancements/issues/753), the requisite of the init container is likely to go away.

Anyways, `crossover` is very resource-efficient in terms of image size and memory, it shouldn't be a huge problem.

## References

### Technical information to use Envoy's dynamic runtime config via local files

- [File Based Dynamic Configuration of Routes in Envoy Proxy](https://medium.com/grensesnittet/file-based-dynamic-configuration-of-routes-in-envoy-proxy-6234dae968d2)
- ["How does one atomically change a symlink to a directory in busybox?"](https://unix.stackexchange.com/questions/5093/how-does-one-atomically-change-a-symlink-to-a-directory-in-busybox)
- [Runtime configuration — envoy](https://www.envoyproxy.io/docs/envoy/latest/intro/arch_overview/operations/runtime.html)

### Other Envoy xDS implementations

Kubernetes CRD and gRPC server based implementations

- [Istio](https://istio.io/)
- [Contour](https://github.com/projectcontour/contour)
- [Ambassador](https://github.com/datawire/ambassador)
- [Gloo](https://github.com/solo-io/gloo)
- [flagger-appmesh-gateway](https://github.com/stefanprodan/flagger-appmesh-gateway)

Consul and gRPC server based implementations

- [gojek/consul-envoy-xds](https://github.com/gojek/consul-envoy-xds)
- [tak2siva/Envoy-Pilot](https://github.com/tak2siva/Envoy-Pilot)
