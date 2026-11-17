---
title: "One Component to Replace Four: Moving My Talos Homelab to Cilium"
date: 2026-11-15T12:00:00-07:00
lastmod: 2026-11-15T12:00:00-07:00
draft: false
summary: "Replacing flannel, kube-proxy, ingress-nginx and MetalLB with Cilium on a three-node Talos homelab: the final config and what went sideways."

tags: ["Kubernetes", "Cilium", "Linux"]
categories: ["Cloud"]

featuredImage: "/images/3-cilium/cilium_migration.webp"
images: ["/images/3-cilium/cilium_migration.webp"]

lightgallery: true
---

My cluster is three Talos VMs on Proxmox, managed with Flux. For a while the networking looked like this, flannel for pod traffic, kube-proxy for Services, ingress-nginx for HTTP, and MetalLB handing out LAN addresses. Four separate pieces, four sets of docs, four places to look when something broke.

Cilium can do all four jobs, so I replaced them for simplicity sake and for network observability. It took five phases, one real outage, and a few predictions I got wrong. This post covers the final config and what went sideways getting there.


## Step one: Cilium takes over the CNI and kube-proxy

Talos ships flannel and kube-proxy by default. You turn both off in the machine config (`cluster.network.cni.name: none` and `cluster.proxy.disabled: true`), then install Cilium yourself.

Everything below lives in one Flux HelmRelease. This is trimmed, but the settings that matter are all here:

```yaml
apiVersion: helm.toolkit.fluxcd.io/v2
kind: HelmRelease
metadata:
  name: cilium
  namespace: kube-system
  annotations:
    # Never let Flux prune this: deleting the HelmRelease would helm-uninstall the CNI
    kustomize.toolkit.fluxcd.io/prune: disabled
spec:
  interval: 24h
  chart:
    spec:
      chart: cilium
      version: "1.20.4"
      sourceRef:
        kind: HelmRepository
        name: cilium
        namespace: kube-system
  values:
    # Use the pod CIDRs Kubernetes already assigned to nodes
    ipam:
      mode: kubernetes

    # Replace kube-proxy. Talos serves the API locally on every node via KubePrism.
    kubeProxyReplacement: true
    k8sServiceHost: localhost
    k8sServicePort: 7445

    # Talos-specific: cgroups are already mounted by Talos
    cgroup:
      autoMount:
        enabled: false
      hostRoot: /sys/fs/cgroup

    # Cilium owns every class-less LoadBalancer Service
    defaultLBServiceIPAM: lbipam

    l2announcements:
      enabled: true

    operator:
      replicas: 1

    gatewayAPI:
      enabled: true
      gatewayClass:
        create: "true"

    hubble:
      tls:
        auto:
          enabled: true
          method: certmanager
          certManagerIssuerRef:
            group: cert-manager.io
            kind: ClusterIssuer
            name: hubble-ca
      relay:
        enabled: true
      ui:
        enabled: true
```

The `k8sServiceHost: localhost` and port `7445` pair is the Talos-specific bit. KubePrism is a small load balancer Talos runs on every node so the API server is always reachable locally. Cilium needs to talk to the API before it can set up Services, so pointing it at KubePrism avoids a chicken-and-egg problem.

The `prune: disabled` annotation is there because of a scary property of Helm: deleting a HelmRelease runs `helm uninstall`. For a CNI, that takes the cluster network down with it. I'd rather a bad revert get stuck than succeed.

### The cutover hurt

My plan was to install Cilium next to flannel and kube-proxy so there'd be minimal downtime. It doesn't work that way. Once Cilium took over the CNI config on each node, about 28 existing pods kept showing `Running` but went `0/1 Ready`. Their probes timed out, Services lost endpoints, and ingress, Flux, cert-manager and the databases all went dark. No data was touched, and deleting the pods so they got recreated under Cilium fixed everything.

Pods created under flannel sit on flannel's bridge. Once Cilium owns the node datapath, they're on a path it knows nothing about. In a production environment I'd drain and replace nodes one at a time instead of swapping the network out from under running pods.

Regarding a related surprise, my qBittorrent pod, which runs behind a gluetun VPN sidecar, started crash-looping. I found Gluetun has a default-drop firewall and works out which local traffic to allow from the pod's own interface config. Under flannel the pod was associated with /24, so the whole subnet was allowed. Under Cilium the pod gets a /32, so only its own address was allowed, and kubelet's probes hit the DROP rule. The fix was one env var, `FIREWALL_INPUT_PORTS: "8080,9999"`. The lesson is that anything that inspects its own network config, like VPN sidecars or firewalls, is worth testing first after a CNI change.

## Hubble and a certificate detour

Hubble is Cilium's flow visibility layer. It uses mutual TLS, and by default Helm generates the certs at render time, so every upgrade makes a new CA and restarts the agents. Instead I pointed Hubble at a cert-manager ClusterIssuer (`certManagerIssuerRef` above). Now the certs renew on their own, and the agents pick up replaced certs without restarting.

I skipped WireGuard pod encryption. Three VMs on one Proxmox bridge in a home LAN, and anyone who could sniff that traffic would already have bigger targets. The costs (CPU, smaller MTU, one more thing to debug) outweighed the benefit.

## Ingress-nginx becomes a Gateway

Gateway API isn't built into Kubernetes. The resource types are CRDs you install first, and Cilium quietly disables its Gateway controller if any are missing. I pull them straight from upstream with a Flux GitRepository pinned to a tag, so upgrading is a one-line change:

```yaml
apiVersion: source.toolkit.fluxcd.io/v1
kind: GitRepository
metadata:
  name: gateway-api
  namespace: flux-system
spec:
  interval: 24h
  url: https://github.com/kubernetes-sigs/gateway-api
  ref:
    tag: v1.6.3
```

Then one shared Gateway for the whole cluster:

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: homelab
  namespace: infra
spec:
  gatewayClassName: cilium
  addresses:
    - type: IPAddress
      value: 192.168.1.110
  listeners:
    - name: http
      protocol: HTTP
      port: 80
      allowedRoutes:
        namespaces:
          from: All
    - name: https
      protocol: HTTPS
      port: 443
      tls:
        mode: Terminate
        certificateRefs:
          - name: wildcard-alarlab-tls
      allowedRoutes:
        namespaces:
          from: All
```

One wildcard certificate replaces the three per-namespace copies I had before. HTTP-to-HTTPS is a single route for every host:

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: http-redirect
  namespace: infra
spec:
  parentRefs:
    - name: homelab
      sectionName: http
  rules:
    - filters:
        - type: RequestRedirect
          requestRedirect:
            scheme: https
            statusCode: 301
```

And each app ships a tiny route in its own namespace. This is homepage's:

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: homepage
  namespace: utils
spec:
  parentRefs:
    - name: homelab
      namespace: infra
      sectionName: https
  hostnames:
    - homepage.alarlab.dev
  rules:
    - backendRefs:
        - name: homepage-service
          port: 80
```

I moved apps over in two commits each. The first added the HTTPRoute next to the old Ingress, with an annotation telling external-dns to ignore it. Then I tested by sending the same request to nginx and the Gateway and comparing status codes and byte counts. The second commit deleted the Ingress. Staging first meant I could prove the new path worked before anything depended on it.

That process still caught me out once. After the first cutover, Cloudflare had the right DNS record, but homepage returned 404. My router has a split-DNS wildcard that sends every `*.alarlab.dev` name to the old nginx IP, and LAN clients use the router, not Cloudflare. I only found it because the TTL on the answer never counted down, so it wasn't a cache, and a made-up hostname got an answer from the router but NXDOMAIN from Cloudflare. So the lession here is check what your LAN resolver actually returns, not just what Cloudflare says.

The last step swapped the Gateway onto `.110` and deleted nginx. Now a new app needs one HTTPRoute and nothing else, because the router's wildcard already points at the Gateway. Simplify, simplify, simplify.

## MetalLB, and the outage I caused

This was the part I got wrong. Cilium can hand out LoadBalancer IPs and announce them on the LAN with ARP, same as MetalLB's L2 mode. I had a plan to move Services over one at a time using a label, tested it on a throwaway Service, and it worked. Then I flipped `defaultLBServiceIPAM` to `lbipam`, because my pool only selected labelled Services and none were labelled yet.

It took ingress and Plex down for about six minutes. The operator log was full of `Removing Ingress IPs`, about 4,000 lines in three minutes. With that setting on, Cilium takes responsibility for every class-less LoadBalancer Service. A pool's selector only decides who gets addresses from that pool. It doesn't make Cilium leave everyone else alone, so any Service with no matching pool had its IP stripped. MetalLB kept putting them back, and the two controllers fought over them.

My canary had only tested a Service with an explicit class and the label, which never exercised the class-less path. A test has to look like what it's standing in for. I tried to patch the live ConfigMap as a quick fix, the permission check blocked it, and I fell back on a `git revert`. That was slower, but it kept Git and the cluster in agreement, and I'm glad it stopped me.

The retry flipped the order. The final config is a pool with no selector at all, one /32 for each address actually in use:

```yaml
apiVersion: cilium.io/v2
kind: CiliumLoadBalancerIPPool
metadata:
  name: homelab
spec:
  blocks:
    - cidr: 192.168.1.110/32   # Gateway homelab
    - cidr: 192.168.1.115/32   # plex-external
    - cidr: 192.168.1.138/32   # immich-pg-lb
    - cidr: 192.168.1.139/32   # postgres-lb
```

Each Service asks for its own address. The Gateway uses `spec.addresses` (shown above), and the others use `spec.loadBalancerIP`. Then the announcement policy:

```yaml
apiVersion: cilium.io/v2alpha1
kind: CiliumL2AnnouncementPolicy
metadata:
  name: homelab
spec:
  # Announce from the workers only (the control-plane node is excluded)
  nodeSelector:
    matchExpressions:
      - key: node-role.kubernetes.io/control-plane
        operator: DoesNotExist
  interfaces:
    - ^ens18$
  loadBalancerIPs: true
```

The order was the whole fix. I scaled the MetalLB controller to zero first, which left its speakers announcing so traffic kept flowing, but it could no longer fight Cilium over assignments. Then I flipped the default. This time there were zero `Removing Ingress IPs` lines, with the same flag as the first attempt. The difference was that every Service now had a pool and a request. At last, one commit removed MetalLB from Flux.

I watched it from outside with a TCP probe on all four addresses, and no DOWN lines showed up. I then rebooted the nodes one at a time as a failover test. When worker-01 held all four leases and went down, they moved to worker-02 on their own. The Gateway was out about 13 seconds, and the Postgres addresses about 2. Because I run single-instance databases, a reboot of the node hosting a primary is roughly a minute of downtime. I knew that going in, and it's a trade-off I made on purpose. Lets be honest, this is a homelab made for tinkering not production.

## What I'd tell someone doing this

- **Existing pods don't survive a CNI swap.** Plan to recreate them, or drain nodes one at a time.
- **Stage, compare, then cut over.** Same request to old and new, same status and byte count, before DNS moves.
- **Cilium doesn't pick up ConfigMap changes by itself.** After changing Helm values, restart the operator, then the agents one node at a time. The DaemonSet's default lets two go down at once.
- **Order beats the flag.** Freeze the old controller, keep the new one inert, then flip the default.

{{< image src="/images/3-cilium/hubble.webp" alt="hubble" caption="hubble" >}}

Hubble was the payoff. I can see pods talking to eachother, with the real client IP and the HTTP request on each line, which I never had with nginx and flannel. And there's one component to learn instead of four which significantly simplified my netwokring stack. Check out Cilium over at https://cilium.io/ if you're looking for more control and observability into Kubernetes networking.
