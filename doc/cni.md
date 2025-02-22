# Container Network Interface (CNI) - Cilium

This stack will use Cilium for traffic from outside-to-inside the cluster (North/South traffic), and for traffic from inside-to-inside the cluster (East/West traffic). Cilium implements the [Gateway API](https://gateway-api.sigs.k8s.io/) for L4 and L7 routing in Kubernetes, and as the Gateway API is the future of North/South traffic to a cluster we will use this for our North/South traffic.

## Node-to-Node Encryption

Cilium supports encrypting all traffic between nodes. This is important as it may be the case that one node is in a geographically different location and traffic may need to cross the public internet to get to the next node. Traffic like this would be unencrypted by the CNI by default, however Cilium supports encryption by IPSec or Wireguard. This encryption is called [Transparent Encryption](https://docs.cilium.io/en/stable/security/network/encryption/) because all traffic _within_ a node is unencrypted, but all traffic _between_ nodes is encrypted.

We will be using WireGuard as it is simpler to setup and automatically handles encryption keys for us.

What traffic is encrypted by cilium? Node-to-Node, Node-to-Pod, Pod-to-Pod?

## Gateway API

### Overview

The Kubernetes Gateway API requires a CNI plugin be used to implement the network behind the API, this is where Cilium comes in. Cilium provides us with a `GatewayClass`, which is a type of `Gateway` that can be deployed - this is a template for the `Gateway` class. This is done in a way to let infrastructure providers offer different types of `Gateways` and Users can choose the `Gateway` they like.

For instance, an infrastructure provider may create two `GatewayClasses` named `internet` and `private` to reflect Gateways that define Internet-facing vs private, internal applications. In our case, the Cilium Gateway API `io.cilium/gateway-controller` will be instantiated.

The schema below represents the various components used by Gateway APIs. When using `Ingress` (prior North/South API), all the functionalities were defined in one API. By deconstructing the ingress routing requirements into multiple APIs, users benefit from a more generic, flexible and role-oriented model.

![Gateway API Model](./files/cni.md/gateway-api-model.png)

When using the Cilium Gateway API the use of a `LoadBalancer` is necessary or else an EXTERNAL-IP won't be exposed.

THe Cilium Service Mesh Gateway API Controller requires the ability to create `LoadBalancer` Kubernetes services. FINISH DOING THE OTHER TURORIAL TO COMPLETE THIS THOUGHT.
If we look at the `kubectl get gateway` we can see what address was assigned if any.

### HTTPS

Using HTTPS just requires supplying a `certificateRefs` list and updating the protocol used. This does not require the `TLSRoute` CRD (custom resource definition):

```yaml
apiVersion: gateway.networking.k8s.io/v1beta1
kind: Gateway
metadata:
  name: tls-gateway
spec:
  gatewayClassName: cilium
  listeners:
    - name: https-1
      protocol: HTTPS
      port: 443
      hostname: "bookinfo.cilium.rocks"
      tls:
        certificateRefs:
          - kind: Secret
            name: demo-cert
    - name: https-2
      protocol: HTTPS
      port: 443
      hostname: "hipstershop.cilium.rocks"
      tls:
        certificateRefs:
          - kind: Secret
            name: demo-cert
```

However, with this prior example, the Gateway is terminating the TLS traffic - so the env Pod is not receiving the encrypted traffic! We get encryption from the world to our Gateway, and then from the Gateway to the Pod it will be unencrypted. We can use `TLSRoute` to passthrough TLS traffic from the client all the way to the Pods - meaning the traffic is encrypted end-to-end:

```yaml
apiVersion: gateway.networking.k8s.io/v1beta1
kind: Gateway
metadata:
  name: cilium-tls-gateway
spec:
  gatewayClassName: cilium
  listeners:
    - name: https
      hostname: "nginx.cilium.rocks"
      port: 443
      protocol: TLS
      tls:
        mode: Passthrough
      allowedRoutes:
        namespaces:
          from: All
---
apiVersion: gateway.networking.k8s.io/v1alpha2
kind: TLSRoute
metadata:
  name: nginx
spec:
  parentRefs:
    - name: cilium-tls-gateway
  hostnames:
    - "nginx.cilium.rocks"
  rules:
    - backendRefs:
        - name: my-nginx
          port: 443
```

The TLDR between the `HTTPRoute` and `TLSRoute` is that the `Gateway` can be used with two TLS modes, `Terminate` and `Passthrough`. The `Terminate` is the default, but when using `TLSRoute` we can specify the passthrough:

In `Terminate`:

- Client -> Gateway: HTTPS
- Gateway -> Pod: HTTP

In `Passthrough`:

- Client -> Gateway: HTTPS
- Gateway -> Pod: HTTPS

### Blue Green / Canary Deployments

The `HTTPRoute` and `TLSRoute` both let us distribute traffic to different backends on the same route differently. Suppose you want to rollout a new feature to about 10% of new users, you can use the built-in weight mechanism to do this:

```yaml
apiVersion: gateway.networking.k8s.io/v1beta1
kind: HTTPRoute
metadata:
  name: load-balancing-route
spec:
  parentRefs:
    - name: my-gateway
  rules:
    - matches:
        - path:
            type: PathPrefix
            value: /echo
      backendRefs:
        - kind: Service
          name: echo-1
          port: 8080
          weight: 90 # <-- Sending traffic here 90% of the time
        - kind: Service
          name: echo-2
          port: 8090
          weight: 10 # <-- Sending traffic here 10% of the time
```

### L2 Service Announcement

By default Cilium has provided a way to create North/South Load Balancer services in the cluster and announce them to the underlying networking using BGP. However, not everyone with an on-premise Kubernetes cluster has a BGP-compatible infrastructure. For this reason, Cilium allows to use ARP (address resolution protocol) in order to announce service IP addresses on Layer 2.

To enable L2 Service Announcement we need to ensure Cilium is installed with the following features:

```bash
cilium install \
  --version v1.16.2 \
  --set kubeProxyReplacement=true \
  --set k8sServiceHost="kind-control-plane" \
  --set k8sServicePort=6443 \
  --set l2announcements.enabled=true \
  --set l2announcements.leaseDuration="3s" \
  --set l2announcements.leaseRenewDeadline="1s" \
  --set l2announcements.leaseRetryPeriod="500ms" \
  --set devices="{eth0,net0}" \
  --set externalIPs.enabled=true \
  --set operator.replicas=2
```

---

# Installation

1. Install all of the required CRDs (custom resource definitions) to extend the Kubernetes API. We can install these individually:

```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes-sigs/gateway-api/v1.2.0/config/crd/standard/gateway.networking.k8s.io_gatewayclasses.yaml
kubectl apply -f https://raw.githubusercontent.com/kubernetes-sigs/gateway-api/v1.2.0/config/crd/standard/gateway.networking.k8s.io_gateways.yaml
kubectl apply -f https://raw.githubusercontent.com/kubernetes-sigs/gateway-api/v1.2.0/config/crd/standard/gateway.networking.k8s.io_httproutes.yaml
kubectl apply -f https://raw.githubusercontent.com/kubernetes-sigs/gateway-api/v1.2.0/config/crd/standard/gateway.networking.k8s.io_referencegrants.yaml
kubectl apply -f https://raw.githubusercontent.com/kubernetes-sigs/gateway-api/v1.2.0/config/crd/standard/gateway.networking.k8s.io_grpcroutes.yaml
kubectl apply -f https://raw.githubusercontent.com/kubernetes-sigs/gateway-api/v1.2.0/config/crd/experimental/gateway.networking.k8s.io_tlsroutes.yaml
```

2. Install Cilium CLI
   > We could also install Cilium via Helm, but Cilium CLI provides built-in check that the network is setup correctly and makes setup extremely simple.

```bash
CILIUM_CLI_VERSION=$(curl -s https://raw.githubusercontent.com/cilium/cilium-cli/main/stable.txt)
CLI_ARCH=amd64
if [ "$(uname -m)" = "aarch64" ]; then CLI_ARCH=arm64; fi
curl -L --fail --remote-name-all https://github.com/cilium/cilium-cli/releases/download/${CILIUM_CLI_VERSION}/cilium-linux-${CLI_ARCH}.tar.gz{,.sha256sum}
sha256sum --check cilium-linux-${CLI_ARCH}.tar.gz.sha256sum
sudo tar xzvfC cilium-linux-${CLI_ARCH}.tar.gz /usr/local/bin
rm cilium-linux-${CLI_ARCH}.tar.gz{,.sha256sum}
```

3. Install Cilium onto the Kubernetes cluster

```bash
# install cilium
sudo cilium install --version 1.17.1 \
    # setup Gateway API
    --namespace kube-system \
    --set kubeProxyReplacement=true \
    --set gatewayAPI.enabled=true
    # allow for host networking & binding to privileged ports (port # < 1023)
    --set gatewayAPI.hostNetwork.enabled=true
    --set envoy.securityContext.capabilities.keepCapNetBindService=true
    --set envoy.securityContext.capabilities.envoy[0]=NET_BIND_SERVICE
    # setup node-to-node encryption
    --set encryption.enabled=true \
    --set encryption.type=wireguard \
    --set encryption.nodeEncryption=true


# ensure that the installation is setup correctly (NOTE: it will look like there are errors right away but after 30 seconds everything should settle!)
cilium status --wait
```

4.  (Optional) If Cilium is already installed into your cluster, you can upgrade it and set the necessary features:

    - Upgrade the existing installation with the new settings:

    ```bash
    cilium upgrade --version 1.17.1 -n kube-system \
        --set kubeProxyReplacement=true \
        --set gatewayAPI.enabled=true
    ```

    - Restart Cilium components to apply changes

    ```bash
    kubectl -n kube-system rollout restart daemonset/cilium
    kubectl -n kube-system rollout restart deployment/cilium-operator
    ```

    - Confirm upgrade is successful and started

    ```bash
    # NOTE: It will look like there are errors righ away, but after 30 seconds everything should settle!
    cilium status --wait
    ```

5.  Confirm the Cilium `GatewayClass` has been deployed during installation/upgrade.

    The `GatewayClass` is a type of `Gateway` that can be deplyed: in other words, it is a template. This is done in a way to let infrastructure providers offer different types of Gateways. For instance, an infrastructure provider may create two `GatewayClasses` named `internet` and `private` to reflect Gateways that define Internet-facing vs private, internal applications.

```bash
kubectl get GatewayClass
```

---

# References

- https://blog.stonegarden.dev/articles/2023/12/cilium-gateway-api/
- https://isovalent.com/labs/gateway-api/
- https://docs.cilium.io/en/stable/network/servicemesh/gateway-api/gateway-api/
- https://gateway-api.sigs.k8s.io/

# check for docker

sudo iptables -t nat -L -n -v --line-numbers

# disable docker

sudo systemctl stop docker
sudo systemctl disable docker
docker network ls

# remove docker rules

sudo iptables -F DOCKER
sudo iptables -F DOCKER-INGRESS
sudo iptables -X DOCKER
sudo iptables -X DOCKER-INGRESS

# remove old silium rules

sudo iptables -t nat -F OLD_CILIUM_POST_nat
sudo iptables -t nat -X OLD_CILIUM_POST_nat

sudo iptables -D INPUT 3 && sudo iptables -D FORWARD 2 && sudo iptables -D OUTPUT 3
sudo iptables -F OLD_CILIUM_INPUT && sudo iptables -X OLD_CILIUM_INPUT
sudo iptables -F OLD_CILIUM_FORWARD && sudo iptables -X OLD_CILIUM_FORWARD
sudo iptables -F OLD_CILIUM_OUTPUT && sudo iptables -X OLD_CILIUM_OUTPUT

# flush existing iptables rules

sudo iptables -t nat -F && sudo iptables -t filter -F && sudo iptables -t mangle -F && sudo iptables -t raw -F
sudo iptables -t nat -X && sudo iptables -t filter -X && sudo iptables -t mangle -X && sudo iptables -t raw -X

kubectl logs -n kube-system -l k8s-app=cilium
