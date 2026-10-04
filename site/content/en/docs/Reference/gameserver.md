---
title: "GameServer Specification"
linkTitle: "Gameserver"
date: 2019-01-03T03:58:48Z
weight: 10
description: >
  Like any other Kubernetes resource you describe a GameServer's desired state via a specification written in YAML or JSON to the Kubernetes API. The Agones controller will then change the actual state to the desired state.
---

A full GameServer specification is available below and in the {{< ghlink href="examples/gameserver.yaml" >}}example folder{{< /ghlink >}} for reference :

```yaml
apiVersion: "agones.dev/v1"
kind: GameServer
# GameServer Metadata
# {{< k8s-api-version href="#objectmeta-v1-meta" >}}
metadata:
  # generateName: "gds-example" # generate a unique name, with the given prefix
  name: "gds-example" # set a fixed name
spec:
  # if there is more than one container, specify which one is the game server
  container: example-server
  # Array of ports that can be exposed as direct connections to the game server container
  ports:
    # name is a descriptive name for the port
  - name: default
    # range is the optional port range name from which to select a port when using a 'Dynamic' or 'Passthrough' port policy.
    range: default
    # portPolicy has four options:
    # - "Dynamic" (default) the system allocates a free hostPort for the gameserver, for game clients to connect to
    # - "Static", user defines the hostPort that the game client will connect to. Then onus is on the user to ensure that the
    # port is available. When static is the policy specified, `hostPort` is required to be populated
    # - "Passthrough" dynamically sets the `containerPort` to the same value as the dynamically selected hostPort.
    #      This will mean that users will need to lookup what port has been opened through the server side SDK.
    # - "None" means the `hostPort` is ignored and if defined, the `containerPort` (optional) is used to set the port on the GameServer instance.
    portPolicy: Static
    # The name of the container to open the port on. Defaults to the game server container if omitted or empty.
    container: simple-game-server
    # the port that is being opened on the game server process
    containerPort: 7654
    # the port exposed on the host, only required when `portPolicy` is "Static". Overwritten when portPolicy is "Dynamic".
    hostPort: 7777
    # protocol being used. Defaults to UDP. TCP and TCPUDP are other options
    # - "UDP" (default) use the UDP protocol
    # - "TCP", use the TCP protocol
    # - "TCPUDP", uses both TCP and UDP, and exposes the same hostPort for both protocols.
    #       This will mean that it adds an extra port, and the first port is set to TCP, and second port set to UDP
    protocol: UDP
  # Health checking for the running game server
  health:
    # Disable health checking. defaults to false, but can be set to true
    disabled: false
    # Number of seconds after the container has started before health check is initiated. Defaults to 5 seconds
    initialDelaySeconds: 5
    # If the `Health()` function doesn't get called at least once every period (seconds), then
    # the game server is not healthy. Defaults to 5
    periodSeconds: 5
    # Minimum consecutive failures for the health probe to be considered failed after having succeeded.
    # Defaults to 3. Minimum value is 1
    failureThreshold: 3
  # Parameters for game server sidecar
  sdkServer:
    # sdkServer log level parameter has three options:
    #  - "Info" (default) The SDK server will output all messages except for debug messages
    #  - "Debug" The SDK server will output all messages including debug messages
    #  - "Error" The SDK server will only output error messages
    #  - "Trace" The SDK server will output all messages, including detailed tracing information
    logLevel: Info
    # grpcPort and httpPort control what ports the sdkserver listens on.
    # Starting with Agones 1.2 the default grpcPort is 9357 and the default
    # httpPort is 9358. In earlier releases, the defaults were 59357 and 59358
    # respectively but as these were in the ephemeral port range they could
    # conflict with other TCP connections.
    grpcPort: 9357
    httpPort: 9358
  # [Stage:Alpha]
  # [FeatureFlag:PlayerTracking]
  # Players provides the configuration for player tracking features.
  # Commented out since Alpha, and disabled by default
  # players:
  #   # set this GameServer's initial player capacity
  #   initialCapacity: 10
  #
  # [Stage:Beta]
  # [FeatureFlag:CountsAndLists]
  # Counts and Lists provides the configuration for generic (player, room, session, etc.) tracking features.
  # Now in Beta, and enabled by default
  counters: # counters are int64 counters that can be incremented and decremented by set amounts. Keys must be declared at GameServer creation time.
    rooms: # arbitrary key.
      count: 1 # initial value can be set.
      capacity: 100 # (Optional) Defaults to 1000 and setting capacity to max(int64) may lead to issues and is not recommended. See GitHub issue https://github.com/agones-dev/agones/issues/3636 for more details.
  lists: # lists are lists of values stored against this GameServer that can be added and deleted from. Keys must be declared at GameServer creation time.
    players: # an empty list, with a capacity set to 10.
      capacity: 10 # capacity value, defaults to 1000.
    rooms: # note that it is allowed to have the same key name with one used in counters
      capacity: 333
      values: # initial values can also be set for lists
        - room1
        - room2
        - room3
  # Pod template configuration
  # {{< k8s-api-version href="#podtemplate-v1-core" >}}
  template:
    # pod metadata. Name & Namespace is overwritten
    metadata:
      labels:
        myspeciallabel: myspecialvalue
    # Pod Specification
    spec:
      containers:
      - name: simple-game-server
        image:  {{< example-image >}}
        imagePullPolicy: Always
      # nodeSelector is a label that can be used to tell Kubernetes which host
      # OS to use. For Windows game servers uncomment the nodeSelector
      # definition below.
      # Details: https://kubernetes.io/docs/setup/production-environment/windows/user-guide-windows-containers/#ensuring-os-specific-workloads-land-on-the-appropriate-container-host
      # nodeSelector:
      #   kubernetes.io/os: windows
```
Since Agones defines a new [Custom Resources Definition (CRD)](https://kubernetes.io/docs/concepts/api-extension/custom-resources/) we can define a new resource using the kind `GameServer` with the custom group `agones.dev` and API version `v1`.

You can use the metadata field to target a specific [namespaces](https://kubernetes.io/docs/concepts/overview/working-with-objects/namespaces/)
but also attach specific [annotations](https://kubernetes.io/docs/concepts/overview/working-with-objects/annotations/) and [labels](https://kubernetes.io/docs/concepts/overview/working-with-objects/labels/) to your resource. This is a very common pattern in the Kubernetes ecosystem.

The length of the `name` field of the Gameserver should not exceed 63 characters.

The `spec` field is the actual GameServer specification and it is composed as follow:

- `container` is the name of container running the GameServer in case you have more than one container defined in the [pod](https://kubernetes.io/docs/concepts/workloads/pods/pod-overview/). If you do,  this is a mandatory field. For instance this is useful if you want to run a sidecar to ship logs.
- `ports` are an array of ports that can be exposed as direct connections to the game server container
  - `name` is an optional descriptive name for a port.
{{% feature publishVersion="1.62.0" %}}
    The names `agones-sdk-grpc`, `agones-sdk-http` and `agones-sdk-health` are reserved: they set the ports the SDK server sidecar binds to, instead of opening a port on the game server container. See [Running GameServers with hostNetwork]({{< relref "#running-gameservers-with-hostnetwork" >}}).
{{% /feature %}}
  - `range` is the optional port range name from which to select a port when using a 'Dynamic' or 'Passthrough' port policy.
  - `portPolicy` has four options:
    - `Dynamic` (default) the system allocates a random free hostPort for the gameserver, for game clients to connect to.
    - `Static`, user defines the hostPort that the game client will connect to. Then onus is on the user to ensure that the port is available. When static is the policy specified, `hostPort` is required to be populated.
    - `Passthrough` dynamically sets the `containerPort`  to the same value as the dynamically selected hostPort. This will mean that users will need to lookup what port to open through the server side SDK before starting communications.
    - `None` means the `hostPort` is ignored and if defined, the `containerPort` (optional) is used to set the port on the GameServer instance.
  - `containerPort` the port that is being opened on the game server process, this is a required field for `Dynamic` and `Static` port policies, and should not be included in <code>Passthrough</code> configuration.
  - `protocol` the protocol being used. Defaults to UDP. TCP and TCPUDP are other options.
- `health` to track the overall healthy state of the GameServer, more information available in the [health check documentation]({{< relref "../Guides/health-checking.md" >}}).
- `sdkServer` defines parameters for the game server sidecar
  - `logging` field defines log level for SDK server. Defaults to "Info". It has three options:
    - "Info" (default) The SDK server will output all messages except for debug messages
    - "Debug" The SDK server will output all messages including debug messages
    - "Error" The SDK server will only output error messages
    - "Trace" The SDK server will output all messages, including detailed tracing information
  - `grpcPort` the port that the SDK Server binds to for gRPC connections
  - `httpPort` the port that the SDK Server binds to for HTTP gRPC gateway connections
- `players` (Alpha, behind "PlayerTracking" feature gate), sets this GameServer's initial player capacity
- `counters` (Beta, requires "CountsAndLists" feature flag) are int64 counters with a default capacity of 1000 that can be incremented and decremented by set amounts. Keys must be declared at GameServer creation time. Note that setting the capacity to max(int64) may lead to issues.
- `lists` (Beta, requires "CountsAndLists" feature flag) are lists of values stored against this GameServer that can be added and deleted from. Key must be declared at GameServer creation time.
- `template` the [pod spec template]({{% k8s-api-version href="#podtemplatespec-v1-core" %}}) to run your GameServer containers, [see](https://kubernetes.io/docs/concepts/workloads/pods/pod-overview/#pod-templates) for more information.

{{% alert title="Note" color="info" %}}
The GameServer resource does not support updates. If you need to make regular updates to the GameServer spec, consider using a [Fleet]({{< ref "/docs/Reference/fleet.md" >}}).
{{% /alert %}}

## Selectable fields

Agones supports [selectable fields](https://kubernetes.io/docs/concepts/overview/working-with-objects/field-selectors/) for querying `GameServer` resources based on spec field values. This feature enables efficient server-side filtering of `GameServer` resources.

By default, the following spec fields are available for field selectors in `GameServer` resources:
- status.state - The lifecycle of the `GameServer` (See [GameServer State Diagram]({{< relref "#gameserver-state-diagram" >}}))
- status.nodeName - The node which the `GameServer` is running on

If you want to configure which fields are selectable, you can configure this through [Helm GameServer configuration variables]({{< ref "docs/installation/install-agones/helm.md#gameservers" >}}).

## Sidecar Containers

Agones supports the use of [Sidecar Containers](https://kubernetes.io/docs/concepts/workloads/pods/sidecar-containers/) with exposed ports.
This requires the `RestartPolicy` to be set to `Always` and can be referenced in the `ports.*.container` field. Typically they used to run
services that need to run alongside the GameServer container, such as log shippers, monitoring agents, allocators and other supporting services.

{{% alert title="Warning" color="warning" %}}
Whether a port on a sidecar container is reachable depends on your CNI (Container Network Interface). When the
kubelet creates the Pod sandbox it only passes the `hostPort` mappings declared on regular `containers` to the CNI, so
with the standard `portmap` plugin a `hostPort` on an `initContainers` entry is unreachable, even though the
`GameServer` reports it as allocated. It only works on CNIs that read the Pod spec themselves to implement `hostPort`.
Check your CNI for compatibility before relying on this, and otherwise keep any container that needs a `GameServer`
port in `containers`.
{{% /alert %}}

{{% feature publishVersion="1.62.0" %}}
## Running GameServers with hostNetwork

By default every `GameServer` Pod gets its own network namespace, and the SDK server sidecar binds fixed ports
(`sdkServer.grpcPort`, `sdkServer.httpPort`, and 8080 for health checks). With `hostNetwork: true` all Pods on a
Node share the Node's network namespace, so a second `GameServer` on the same Node would fail to bind those ports.

To give each `GameServer` its own SDK server ports, add ports with the following reserved names to `spec.ports`,
usually with the `Passthrough` port policy so that Agones allocates a free port on the Node for each of them:

| Port name           | Sidecar flag    | Replaces                               | Also used for                                                           |
|---------------------|-----------------|----------------------------------------|-------------------------------------------------------------------------|
| `agones-sdk-grpc`   | `--grpc-port`   | `sdkServer.grpcPort` (default 9357)    | `AGONES_SDK_GRPC_PORT` in the game server containers                    |
| `agones-sdk-http`   | `--http-port`   | `sdkServer.httpPort` (default 9358)    | `AGONES_SDK_HTTP_PORT` in the game server containers                    |
| `agones-sdk-health` | `--health-port` | 8080                                   | the sidecar's liveness probe, and the game server container's `/gshealthz` liveness probe |

Each of them is optional, and a `GameServer` without them is not changed. The SDKs read the `AGONES_SDK_*_PORT`
environment variables, so game server code does not need to change.

The reserved ports:

- are declared on the SDK server sidecar container, not on the game server container, so `container` must be empty;
- default to, and must use, the `TCP` protocol;
- must not use the `None` port policy. With `Passthrough` the allocated `hostPort` is used; with `Static` or `Dynamic`
  the `containerPort` is used (with `hostNetwork: true`, Kubernetes requires `hostPort` and `containerPort` to be equal,
  so `Dynamic` does not work there);
- are listed in `status.ports` (and in allocation results) like any other port, next to your game ports.

The gRPC and HTTP servers listen on `localhost`, but the health check server listens on all interfaces (the kubelet
probes it on the Pod IP, which is the Node IP with `hostNetwork: true`), so it is reachable from outside the Node unless
a firewall blocks the port range.

For example, a `Fleet` of `GameServers` that share the Node's network, several per Node:

```yaml
apiVersion: agones.dev/v1
kind: Fleet
metadata:
  name: simple-game-server-hostnetwork
spec:
  replicas: 4
  template:
    spec:
      ports:
        # the game port. simple-game-server reads the first port from the SDK with PASSTHROUGH=TRUE
        - name: default
          portPolicy: Passthrough
          protocol: UDP
        # the SDK server sidecar binds these, a different set for each GameServer
        - name: agones-sdk-grpc
          portPolicy: Passthrough
        - name: agones-sdk-http
          portPolicy: Passthrough
        - name: agones-sdk-health
          portPolicy: Passthrough
      template:
        spec:
          hostNetwork: true
          dnsPolicy: ClusterFirstWithHostNet
          containers:
            - name: simple-game-server
              image: us-docker.pkg.dev/agones-images/examples/simple-game-server:0.44
              env:
                - name: PASSTHROUGH
                  value: "TRUE"
```

With `hostNetwork: true` the game server process must also bind the port Agones allocated to it (for example by using
`Passthrough` and reading the port with the SDK's `GameServer()`), and any other sidecar containers you add must not
use fixed ports either.

{{% /feature %}}

## Stable Network ID


If you want to connect to a `GameServer` from within your Kubernetes cluster via a convention based
DNS entry, each Pod attached to a `GameServer` automatically derives its hostname from the name of the `GameServer`.

To create internal DNS entries within the cluster, a group of `Pods` attached to `GameServers` can use a 
[Headless Service](https://kubernetes.io/docs/concepts/services-networking/service/#headless-services) to control 
the domain of the Pods, along with providing 
a [`subdomain` value to the `GameServer` `PodTemplateSpec`](https://kubernetes.io/docs/concepts/services-networking/dns-pod-service/#pod-s-hostname-and-subdomain-fields)
to provide all the required details such that Kubernetes will create a DNS record for each Pod behind the Service.

You are also responsible for setting the labels on the `GameServer.Spec.Template.Metadata` to set the labels on the
created Pods and creating the Headless Service responsible for the network identity of the pods, Agones will not do
this for you, as a stable DNS record is not required for all use cases.

To ensure that the `hostName` value matches
[RFC 1123](https://kubernetes.io/docs/concepts/overview/working-with-objects/names/#dns-label-names), any `.` values 
in the `GameServer` name are replaced by `-` when setting the underlying `Pod.Spec.HostName` value.

## GameServer State Diagram

The following diagram shows the lifecycle of a `GameServer`. 

Game Servers are created through Kubernetes API (either directly or through a [Fleet]({{< ref "fleet.md" >}})) and their state transitions are orchestrated by:

- GameServer controller, which allocates ports, launches Pods backing game servers and manages their lifetime
- Allocation controller, which marks game servers as `Allocated` to handle a game session
- SDK, which manages health checking and shutdown of a game server session

![GameServer State Diagram](../../../diagrams/gameserver-states.dot.png)

## Primary Address vs Addresses

[`GameServer.Status`][gss] has two fields which reflect the network address of the `GameServer`: `address` and `addresses`.
The `address` field is a policy-based choice of "primary address" that will work for many use cases,
and will always be one of the `addresses`. The `addresses` field contains every address in the [`Node.Status.addresses`][addresses] and [`Pod.Status.podIPs`][podIPs] (to allow a direct pod access),
representing all known ways to reach the `GameServer` over the network.

To choose `address` from `addresses`, [Agones looks for the following address types][addressFunc], in highest to lowest priorty:
* `ExternalDNS`
* `ExternalIP`
* `InternalDNS`
* `InternalIP`

e.g. if any `ExternalDNS` address is found in the respective `Node`, it is used as the `address`. (`PodIP` is not considered
for `address`.)

The policy for `address` will work for many use-cases, but for some advanced cases, such as IPv6 enablement, you may need
to evaluate all `addresses` and pick the addresses that best suits your needs.

[addresses]: {{% k8s-api-version href="#nodeaddress-v1-core" %}}
[podIPs]: {{% k8s-api-version href="#podip-v1-core" %}}
[addressFunc]: https://github.com/agones-dev/agones/blob/a59c5394c7f5bac66e530d21446302581c10c225/pkg/gameservers/gameservers.go#L37-L71
[gss]: {{% ref "/docs/Reference/agones_crd_api_reference.html#agones.dev/v1.GameServerStatus"  %}}
