# What happens when ... Kubernetes

Imagine I want to run nginx in a Kubernetes cluster and make it visible to the outside world. We can do this in a one-liner with kubectl:

```bash
kubectl run --image=nginx --replicas=3 --port=80 --expose
```

But what _really_ happens when you run that command?

One of the beautiful things about Kubernetes is that it offers tremendous power while abstracting complexity through user-friendly interfaces. In order to understand this complexity (and therefore what value Kubernetes offers us), we need to follow the path of a request as it travels through a Kubernetes sytem. This repo seeks to explain that lifecycle.

This is a living document. Contributions that add, edit, or delete content where necessary is definitely welcome!

## api version reconciliation

The first thing that kubectl will do is perform some client-side validation. This ensures that requests that will always fail (e.g. creating a non-supported or outdated resource) will not be sent to kube-apiserver, thereby saving precious bandwidth and CPU cycles. 

To do this, kubectl uses the official kubernetes [client]() to discover all the potential [API groups]() for a given HTTP endpoint. The API server represents its RESTful state using OpenAPI (previously Swagger) documents, so these are retrieved and persisted to disk in the `~/.kube/schema` directory. It then retrieves all supported versions for those groups and saves to `~`. Now that it has a cached representation of the API contract, kubectl can perform validation. 

However, we need to hold our horses because `kubectl run` acts a bit differently. Most commands (like `kubectl create`) to kubectl reference some kind of YAML or JSON file which, to kubectl's perspective, has the potential to contain malformed or just plain wrong data structures. The run command on the other hand collects input through CLI flags, so kubectl has full control over how to use and serialize that data - so it doesn't need to perform schema validations. 

## kubectl generates objects

After the "validation" stage, kubectl then begins assembling the request it'll send to kube-apiserver. To do this it uses a concept called [generators]() to generate a Resource object and then serialize it into JSON. 

What you might not realise is that in the run command you can actually deploy multiple resource types, not just deployments. To make that happen, kubectl will try to infer what type of resource to run if it wasn't explicitly specified with the `--generator` flag. For example, resources that have `--restart-policy=Always` are considered Deployments, those with `--restart-policy=Never` are considered pods. Another part of this inference stage is figuring out whether other actions need to be triggered: for example to record the command (for rollouts or auditing), or whether this command is just a dry run (`--dry-run`). 

The final step is to construct a versioned client that maps with the OpenAPI spec for the specific API group, and then delegate request lifecycle events to the client, such as sending HTTP request and parsing the response from the kube-apiserver.

## client auth

In order to send the request successfully, however, the client needs to be able to authenticate. User credentials are almost always stored in the `kubeconfig` file which resides on disk. kubectl will try to auto-detect the correct path to the file by doing the following:

- if `--kubeconfig` flag is provided, easy peasy, use that.
- if the `$KUBECONFIG` environment variable is defined, use that.
- otherwise look in a predictable directory like `~/.kube`, and use the first file found.

After parsing the file, it then determines the current context to use, the current cluster to point to, and any auth information associated with the current user. If the user provided flag-specific values (such as `--username`) these take precedence and will override kubeconfig. Once it has this information, kubectl populates the client's configuration so that it is able decorate the HTTP request appropriately:

- x509 certificates are sent using [`tls.TLSConfig`]() (this also includes the root CA)
- bearer tokens are sent in the "Authorization" HTTP header
- username and password are sent via HTTP basic authentication
- the OpenID auth process is handled manually by the user beforehand, producing a token which is sent like a bearer token

## server auth

So our request has been sent, hooray! What next? This is where the kube-apiserver enters the picture. In a nutshell, the kube-server is the primary interface that clients use to persist and retrieve cluster state. To do this well, it needs to be able to verify that the client is who they say there are, this is authentication.

How does the apiserver authenticate requests? When the server first starts, it looks at all the CLI flags the user provided and assembles a list of suitable authenticators. Let's take an example: if a `--client-ca-file` has been passed in, it appends the [x509 authenticator](); if it sees `--token-auth-file` provided, it appends the [token authenticator]() to the list. Every time a request is received, it is [run through the authenticator chain until one succeeds](): 

- the x509 handler will verify that the HTTP request is encoded with a TLS key signed by the CA root cert
- the bearer token handler will verify that the provided token (specified in the HTTP Authorization header) exists in the provided file on disk
- the password auth handler will similarly ensure that the HTTP request's basic auth credentials match its own local state.

If every authenticator fails, the request fails and an aggregate error is returned. If authentication succeeds, the `Authorization` header is removed from the request, and user information is added to its context. This provides future validators (such as authorization and admission controllers) the ability to access the previously established identity of the user. 

## authorization

Okay, the request has been sent, and kube-apiserver has successfully verified we are who we say we are. What a relief! However, we're not done yet. We may be who we say we are, but are we _allowed_ to perform this action? Identity and permission are not the same thing, and in order for us to continue, the apiserver needs to authorize us.

The way kube-apiserver handles this is very similar to authentication: based on flag inputs, at start-up it will assemble a chain of authorizers that will be run for every incoming request. If all authorizers deny the request, the request results in a `Forbidden` response and goes no further down the chain. If a single authorizer approves, the request proceeds.

Some examples of authorizers that ship with v1.8 are:

- `AllowAll` and `DenyAll`, which approve and deny all traffic respectively;
- webhook, which interacts with an off-cluster HTTP(S) service;
- ABAC, which enforces policies defined in a static file;
- RBAC, which enforces RBAC roles which are added by the admin as k8s resources
- Node, which ensures that node clients, i.e. the kubelet, can only access resources hosted on itself.

### [side point]: accessing state with informers

As you might have noticed, some authorization controllers like RBAC and Node are dynamic, in that they need to retrieve cluster state to function. To return to the example of the RBAC authorizer, we know that when a request comes in, the authenticator will save an initial representation of user state for later use. The RBAC authorizer will then use this to retrieve all the roles and role bindings that are associated with the user in etcd. How are controllers supposed to access and modify such resources? It turns out this is a common use case and is solved in Kubernetes with informers. 

"A what?!" I hear you ask. An informer is a pattern that allows controllers to subscribe to storage events and easily list resources they're interested in. Apart from providing an abstraction which is nice to work with, it also handles a lot of the nuts and bolts such as caching. Caching is important because it reduces unnecessary kube-apiserver connections, and reduces duplicate serialization costs server- and controller-side. By formalising a model like this, it also allows controllers to interact in a threadsafe manner without having to worry about stepping on anybody else's toes. 

In the case of the RBAC authorizor, it will not register any event handlers, but what it will do is use the informer to list over a collection of roles and retrieve a specific resource in a consistent, supported way. Now that we know what informers are and the basics of how they're used, let's leave the rabbit hole and return to our main journey.

## admission controllers

Okay so we've authenticated and been authorized at this point, awesome sauce. So what's left? From kube-apiserver's perspective, it believes who we are and permits us to continue, but with Kubernetes, other parts of the system have strong opinions about what should and should not be permitted to happen. Cue admission controllers.

Whilst authorization is focused on answering if a _user_ is authorized to perform an action, admission control is focused on if the wider system will permit the action. They are the last bastion of control before an object is persisted to etcd, so they encapsulate the remaining system checks to ensure an action does not produce unexpected or negative results.

The way admission controllers are initialized is very similar to authenticator and authorizer chains. To promote extensibility, they are stored as plugins in the `plugin/pkg/admission` directory, made to satisfy a small interface, and are compiled into kubernetes itself. Unlike other control chains we have mentioned, if a single admission controller fails, the whole chain is broken and the request will fail. 

Admission controllers are usually categorised into resource management, security, defaulting, and referential consistency. Sometimes an admission controller will permit a request, but reconcile cluster state in accordance with its own policy (the `NamespaceExists` controller will create a namespace for example). Commonly used resource ACs are: 

- `InitialResources` which sets default resource limits to the resources for a container based on past usage; 
- `LimitRanger` which sets defaults for container requests and limits, or enforce upper bounds on certain resources (no more than 2GB of memory, default to 512MB); 
- `ResourceQuota` which calculates and denies a number of objects (pods, rc, service load balancers) or total consumed resources (cpu, memory, disk) in a namespace.

## each object save to etcd

By this point, Kubernetes has fully vetted the incoming request and has permitted it to go forth and prosper. The next step is how kube-apiserver deserializes the request, constructs resources from them, and persists them to the datastore. Let's break that down a bit.

How does kube-apiserver know what to do when it accepts our request? Enter our old friend OpenAPI! As we mentioned earlier, all API operations are formalised into an OpenAPI spec, which lists the path, JSON structures and query parameters. These OpenAPI specs are generated into the `pkg/generated/openapi` package when kubernetes is built. This spec then populates the [apiserver's config](https://github.com/kubernetes/apiserver/blob/7001bc4df8883d4a0ec84cd4b2117655a0009b6c/pkg/server/config.go#L149). This spec is then iterated over and each API group is [installed](https://github.com/kubernetes/apiserver/blob/7001bc4df8883d4a0ec84cd4b2117655a0009b6c/pkg/server/genericapiserver.go#L371) into a chain of handlers.

1. When the kube-apiserver binary is run, it [creates a server chain](https://github.com/kubernetes/kubernetes/blob/master/cmd/kube-apiserver/app/server.go#L119), which allows apiserver aggregation
1. When this happens, a [generic apiserver is created](https://github.com/kubernetes/kubernetes/blob/master/cmd/kube-apiserver/app/server.go#L149) that serves as a default implementation. 
1. The generic server then iterates over all the API groups and configures the [storage provider](https://github.com/kubernetes/kubernetes/blob/c7a1a061c3dc5acabcc0c35b3b96a6935dccf546/pkg/master/master.go#L410)
1. For every API group it also iterates over each of the group versions and [installs the REST mappings](https://github.com/kubernetes/apiserver/blob/7001bc4df8883d4a0ec84cd4b2117655a0009b6c/pkg/endpoints/groupversion.go#L92) for each of the group version's routes. 
1. For our specific use case, a [POST handler](https://github.com/kubernetes/apiserver/blob/7001bc4df8883d4a0ec84cd4b2117655a0009b6c/pkg/endpoints/installer.go#L710) is registered, which in turn will delegate to a [create resource handler](https://github.com/kubernetes/apiserver/blob/7001bc4df8883d4a0ec84cd4b2117655a0009b6c/pkg/endpoints/handlers/create.go#L37).

After all this is set up, the server is in a position to respond. By the time a request comes in, this is what will happen:

1. If the handler chain can match the request to a set pattern (i.e. to the routes we registered), it will [dispatch the dedicated handler](https://github.com/kubernetes/apiserver/blob/7001bc4df8883d4a0ec84cd4b2117655a0009b6c/pkg/server/handler.go#L143) that was registered for the route. Otherwise it will use a [path-based handler](https://github.com/kubernetes/apiserver/blob/7001bc4df8883d4a0ec84cd4b2117655a0009b6c/pkg/server/mux/pathrecorder.go#L248). If no paths are registered, a [not found handler](https://github.com/kubernetes/apiserver/blob/7001bc4df8883d4a0ec84cd4b2117655a0009b6c/pkg/server/mux/pathrecorder.go#L254) is invoked.
1. Luckily for us, we have a registered route called [`createHandler`](https://github.com/kubernetes/apiserver/blob/7001bc4df8883d4a0ec84cd4b2117655a0009b6c/pkg/endpoints/handlers/create.go#L37)! What does it do? Well it will first decode the HTTP request and perform basic validation, such as ensuring the JSON they provided correlates with our expectation of the versioned API resource.
1. Auditing and final admission [will occur](https://github.com/kubernetes/apiserver/blob/7001bc4df8883d4a0ec84cd4b2117655a0009b6c/pkg/endpoints/handlers/create.go#L93-L104). 
1. The resource will be [saved to etcd](https://github.com/kubernetes/apiserver/blob/7001bc4df8883d4a0ec84cd4b2117655a0009b6c/pkg/endpoints/handlers/create.go#L111) by [delegating to the storage provider](https://github.com/kubernetes/apiserver/blob/19667a1afc13cc13930c40a20f2c12bbdcaaa246/pkg/registry/generic/registry/store.go#L327). Usually the etcd key will be the form of `<namespace>/<name>`, but this is configurable.
1. Any create errors are caught and finally the storage provider performs a get to ensure the object was created, then invokes any post-create handlers and decorators if additional finalization is required.
1. The HTTP response [is constructed](https://github.com/kubernetes/apiserver/blob/7001bc4df8883d4a0ec84cd4b2117655a0009b6c/pkg/endpoints/handlers/create.go#L131-L142) and sent back

## initializers

after an object is persisted to the datastore, it is not made fully visible by the apiserver or scheduled until a series of "intializers" have run for the specific resource. if no initializers are set for the resource type, it is made visible immediately. an initializer is a controller that is associated with a resource type and performs logic on the resource before it's made available to the outside world. 

as Ahmet Balkan notes in [his great blog post](https://ahmet.im/blog/initializers/), this allows kubernetes to perform some cool bootstrap operations, like:
- Inject a proxy sidecar container to the pod if it has port 80, or has a particular annotation.
- Inject a volume with test certificates to all pods in the test namespace automatically.
- If a Secret is shorter than 20 characters (probably a password), prevent its creation.

intializer configuration objects allow you to declare which initializers should run for certain object types. for example, for every deployment, ensure that MyDeploymentInitializer runs. this would mean that when a Deployment object is received, MyDeploymentInitializer is appended to the object's `metadata.initializers.pending` field. each initializer runs sequentually and removes itself from the list when it's finished processing.

all throughout this process, a pod's status will be `PodInitializing`. when this bootstrap finishes the object will be considered initialized and then allow other controllers to continue the creation process.

one question which you might have asked is, how can a userland controller process resources if they're not made visible by the apiserver? this problem is solved by using the `includeUninitialized` query parameter, which returns unitialized objects. 

## deployment controller creates replicasets

after a deployment record is stored to etcd, it is detected by the deployment controller whose job it is to listen out for changes to deployment records (adds, updates and deletes). it aims to reconcile state by synchronizing deployments with their respective pods and replica sets throughout the system. 

a collection of pods make up a replica set (defined by a replica count), and replica sets make up a deployment. when a rollout to a new container image, or a scale operation, occurs, a new replica set is created.

to return to our create operation: at this point in time, there exists a Deployment object in the datastore, but no associated replica sets or pods. the deployment controller now starts the process of synchronization.

when the controller starts, it users informers to inform it of certain events, firing off specific functions for each event type. when a deployment is added, it's added to an internal work queue and a sync method is called. 

it will first list all replica sets and pods that match the deployments label selector, and no will be returned. it will then begin rolling out its first replica set by creating the resource, assigning it its label selector, and giving it the revision number of 1. its PodSpec is copied from the Deployment's manifest, as well as other metadata including replica set. sometimes the deployment record will need to be updated after this too (for instance if the progress deadline is set).

updates status.

once this task finishes, it enters a wait loop waiting for the deployment to become complete. this is the job of the next controller and happens once all of its desired replicas are updated and available, and no old pods are running. 

once this wait loop finishes, it will perform a cleanup, which usually involves deleting any old replica sets that exceed the revision history limit for a given deployment (by default this is disabled).

## replicaset controller creates pods

the next steps is the replicaset controller which is responsible for monitoring the lifecycle of replicasets and their dependent resources (pods), and acting appropriately on key events. so far, kubectl has sent a Deployment resource to the apiserver, and the deployments controller has created a new replicaset resource with the appropriate owner reference.

when a new replicaset is created, the RS controller inspects the desired state and realizes there is a skew between what exists and what is required. it then seeks to reconcile this state by bumping the number of pods that belong to the replica set. it starts creating them in a careful manner, ensuring that a burst count is always matched. 

the create operations is also batched, meaning that the batch sizes start at SlowStartInitialBatchSize and doubles with each successful iteration in a kind of "slow start". this aims to mitigate the risk of numerous pod bootup failures (due to quotas) that could end up placing unnecessary load on the API server. 

owner references. not only does this ensure that child resources are garbage-collected once a resource managed by the controller is deleted (cascading deletion), it also provides an effective way for parent resources to not fight over their children. having a stateful representation of dependencies allows controller restarts to not affect the state of the system. this enforces a policy of isolation where controllers do not operate on resources they don't explicitly own, nor should it count these resources towards how it reconciles state. in other words, controllers are designed so that they're responsible (only take ownership of their own resources), non-interfering, and non-sharing. 

sometimes, orphaned resources (i.e. those which were not created by the controller but having matching label selectors) are adopted by an owner resource if no ControllerRef already exists for it. multiple parents can race to adopt a child, but only one will be successful (the others will receive a validation error). orphans arise when their parents are deleted, but garbage collection policies prohibit child deletion.

updates the status and also sets the ObservedGeneration, so that clients know the controller has processed the resource successfully.

## scheduler assigns node

The scheduler runs is a controller than runs as part of the control panel among other master components. like all other controllers, it listens out for events and attempts to reconcile state. in this case, it listens out for pods with an empty PodSpec.NodeName and attempts to find a suitable Node that the pod can reside on. 

in order to find a suitable pod, a specific scheduling algorithm is used. the default scheduler registers predicates that are run against all Schedulable Nodes in the system and filters out those which are inappropriate for the workload. for example, if the PodSpec explicitly requests CPU or RAM resources, and a Node cannot meet these requests due to lack of capacity, it will be deselected. rsource capacity is calculated as the total capacity minus the sum of the resource requests of currently running containers.

once appropriate nodes have been selected, a series of priority functions are run against the remaining nodes in order to rank them according to suitability. for example, in order to spread workloads across the system, it will favour nodes with less resources (i.e. containers with resource requests. as it runs these functions, it assigns each node a numerical rank. the highest ranked node is then selected for scheduling.

both predicate and priority functions are extensible and can be defined by using the `--policy-config-file` flag. this introduces a degree of flexibility into the default scheduler. administrators can also run custom schedulers (controllers with custom processing logic) as Deployments. if a PodSpec contains `schedulerName`, Kubernetes will hand over scheduling for that pod to whatever scheduler thas has registered itself under that name.

once the algorithm finds a node, the scheduler then creates a Binding API object whose Name and UID match the Pod, and whose ObjectReference field contains the name of the selected node. this is then POSTed to the apiserver.

when the apiserver receives this Binding object, the registry deserializes the object and updates the following fields on the Pod object: it sets the NodeName to the one in the ObjectReference, it adds any relevant annotations, and sets its `PodScheduled` status condition to `True`.

## kubelet begins pod sync

the next step is handled by the kubelet, which is an agent that runs on every node marked for handling workloads. the kubelet agent is responsible for listening out for new Pod manifests and deploying them as containers on the instance it's running on. it does this by checking the apiserver HTTP API every 20 seconds (this is configurable) for unscheduled pods whose NodeName matches the node the kubelet is running on.

the kubelet pulls all of the relevant pods (i.e. those with the correct NodeName) from the server, and does a quick comparison to see if the current state is new. it then adds all update events to a sync channel and depending on the type of operation (add, update, delete) fires off the correct handler. then `syncPod` is called, which does the following:

the kubelet will now begin to start our pod! however it doesn't really have a concept of "start this pod", all it knows about is syncing the real state of affairs with a desired state. with this in mind, it defines a [`syncPod`](https://github.com/kubernetes/kubernetes/blob/fc8bfe2d8929e11a898c4557f9323c482b5e8842/pkg/kubelet/kubelet.go#L1481) method that performs the following:

- if the pod is being created (ours is!), it registers some startup metrics

- generates a [PodStatus]() API object, which is responsible for indicating the state of a pod through its `phase`. The phase of a Pod is a simple, high-level summary of where the Pod is in its lifecycle. Examples include `Pending`, `Running`, `Succeeded`, `Failed` and `Unknown`. 

    - when the PodStatus is generated, what's interesting is that a chain of PodSyncHandlers is called on the Pod. If any of them decide that the Pod no longer belongs there, the Pod's phase will change to v1.PodFailed and it will be evicted from the Node.

    - the Pod Phase is determined by the status of its init and real containers. Since our containers have not been started yet, the containers are classed as [waiting](https://github.com/kubernetes/kubernetes/blob/fc8bfe2d8929e11a898c4557f9323c482b5e8842/pkg/kubelet/kubelet_pods.go#L1244). Any pod with a waiting container is considered [Pending](https://github.com/kubernetes/kubernetes/blob/fc8bfe2d8929e11a898c4557f9323c482b5e8842/pkg/kubelet/kubelet_pods.go#L1258-L1261), which is the case in our situation!

    - the Pod condition is also dictated by the condition of its containers. Since none of our containers have been created by the runtime yet, it will set the PodReady condition to False](https://github.com/kubernetes/kubernetes/blob/fc8bfe2d8929e11a898c4557f9323c482b5e8842/pkg/kubelet/status/generate.go#L70-L81).

- It will then send this PodStatus to the kubelet's status manager, which will asynchronously update the etcd record via the apiserver 

- a series of admit handlers are run to ensure the pod is allowed to be run. Handlers include AppArmor and enforcing `NO_NEW_PRIVS`. Pods denied at this stage will stay in the `Pending` state indefinitely.

- if the `cgroups-per-qos` flag has been provided, kubelet will create cgroups for the pod and apply resource parameters. This is to enable better Quality of Service (QoS) handling for pods.

- data directories are created for the pod. These include the pod dir (usually `/var/run/kubelet/pods/<podID>`), its volumes dir (`<podDir>/volumes`) and its plugins dir (`<podDir>/plugins`).

- the volume manager will attach and wait for any relevant volumes defined in Pod.Spec.Volumes. Depending on the type of volume being mounted, some pods will need to wait longer (e.g. cloud or NFS volumes).

- all secrets defined in `Pod.Spec.ImagePullSecrets` are retrieved from the apiserver

- the container runtime then runs the container (described in more detail next)

this syncPod callback will be invoked every XXX seconds

## CRI and pause containers

the software responsible for starting and stopping containers is called the container runtime. in an effort to be more extensible, since 1.5 the kubelet has been using a plugin interface called CRI (Container Runtime Interface) to interact with container runtimes. CRI provides an abstracted interface between the kubelet and a specific container runtime, allowing them to communicate via protocol buffers and a gRPC API. the net benenit of using such an abstraction is a clean contract between kubelet and a runtime, allowing new compliant runtimes to be added with minimal overhead, since implementations are no longer tightly coupled.

when a pod is first started, kubelet invokes the `RunPodSandbox` RPC. a "sandbox" is a CRI term to describe a set of containers, which in Kubernetes parlance is a pod. for hypervisor-based runtimes, a sandbox might represent a VM. for the docker service, creating a sandbox involves creating a "pause" container which acts as the "parent container" for all other containers in a pod. it does so by hosting the namespaces that other containers will join (IPC, network, PID) and serving as PID 1 for each pod, allowing it to reap zombie containers. having a dedicated container allows for more efficient and reliable process reaping. 

after the pause container has been created, checkpointed to disk, and started.

## CNI and pod networking

when the kubelet sets up networking, it delegates this task to a CNI plugin. CNI stands for container network interface and is an abstraction that allows different network providers to set up networking and communicate back to the kubelet in a standardised way. CNI works by piping JSON configuration to a CNI binary that is usually tasked with a specific responsibility. 

> A CNI plugin is responsible for inserting a network interface into the container network namespace (e.g. one end of a veth pair) and making any necessary changes on the host (e.g. attaching the other end of the veth into a bridge). It should then assign the IP to the interface and setup the routes consistent with the IP Address Management section by invoking appropriate IPAM plugin.

for the ADD command, the container ID is passed to the ID, along with the path to the network NS file, the interface name to set up inside the container, the path to the CNI binary, and any additional networking information, such as which DNS nameservers to use. kubelet will pass in the cluster's internal DNS server's IP address, which will ensure that the container's `resolv.conf` file is set appropriately. a list of CNI plugins (also defined in the JSON) will then by run in order.

kubelet relies on `--cni-conf-dir` to find the CNI configuration. it will then find the first configuration file in that directory, and send it to the appropriate plugin binary. for example:

```yaml
{
    "cniVersion": "0.3.1",
    "name": "bridge",
    "type": "bridge",
    "bridge": "cnio0",
    "isGateway": true,
    "ipMasq": true,
    "ipam": {
        "type": "host-local",
        "ranges": [
          [{"subnet": "${POD_CIDR}"}]
        ],
        "routes": [{"dst": "0.0.0.0/0"}]
    }
}
```

will set up a local Linux bridge on the host. With bridge plugin, all containers (on the same host) are plugged into a bridge (virtual switch) that resides in the host network namespace. The containers receive one end of the veth pair with the other end connected to the bridge. An IP address is only assigned to one end of the veth pair -- one residing in the container. The bridge itself can also be assigned an IP address, turning it into a gateway for the containers. Alternatively, the bridge can function purely in L2 mode and would need to be bridged to the host network interface (if other than container-to-container communication on the same host is desired). The network configuration specifies the name of the bridge to be used. If the bridge is missing, the plugin will create one on first use and, if gateway mode is used, assign it an IP that was returned by IPAM plugin via the gateway field.

IP allocation is handled by a IPAM plugins, which are invoked by the CNI plugin according to the configuration. similar to main plugins, IPAM ones are invoked via an executable have a standardised interface. The IPAM plugin must determine the interface IP/subnet, Gateway and Routes and return this information to the "main" plugin to apply. host-local IPAM plugin allocates ip addresses out of a set of address ranges. It stores the state locally on the host filesystem, therefore ensuring uniqueness of IP addresses on a single host.

so after this process, the networking for the pause container is set up: it's been allocated a unique IP from a global pod subnet, routes have been set up inside, veth interfaces have been created, and a linux bridge on the host. 

it is also possible to use overlay networking to dynamically connect host. Flannel, for example, is responsible for providing a layer 3 IPv4 network between multiple nodes in a cluster. Flannel does not control how containers are networked to the host, only how the traffic is transported between hosts. However, flannel does provide a CNI plugin for Kubernetes and a guidance on integrating with Docker. 

## containers are started

once the sandbox has finished initializing and is active, the kubelet can begin ceating individual containers for it. it first starts any init containers, then start the main containers themselves. the process for doing this is: 

1. pull the image, 
1. create the container. It does this by populating a `ContainerConfig` struct (command, image, labels, mounts, devices, environment variables etc.) with the PodSpec and then sending that with protobufs to the container runtime. For Docker, it deserializes the payload and populates its own config structures to send to the Daemon API. In the process it applies a few metadata labels (container type, log path, sandbox ID).
1. (alpha feature) register container with CPU manager, which is a new feature in 1.8 that assigns containers to sets of CPUs on the local node by using the `UpdateContainerResources` CRI method
1. start the container, 
1. if post-start container lifecycle hooks are registered, run them. Hooks can either be of the type `Exec` (executes a specific command inside the container) or `HTTP` (performs a HTTP request against a container endpoint). If the PostStart hook takes too long to run, hangs, or fails, the container will never reach a `running` state.