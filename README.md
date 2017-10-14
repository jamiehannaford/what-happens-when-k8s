# What happens when ... Kubernetes

Imagine I want to run nginx in a Kubernetes cluster and make it visible to the outside world. We can do this in a one-liner with kubectl:

```bash
kubectl run --image=nginx --replicas=3 --port=80 --expose
```

But what _really_ happens when you run that command?

One of the beautiful things about Kubernetes is that it offers tremendous power while abstracting complexity through user-friendly interfaces. In order to understand this complexity (and therefore what value Kubernetes offers us), we need to follow the path of a request as it travels through a Kubernetes sytem. This repo seeks to explain that lifecycle.

This is a living document. Contributions that add, edit, or delete content where necessary is definitely welcome!

## api version reconciliation

kubectl uses client-go's discovery client to retrieve all API groups, saving this list to a file in ~/.kube/schema. It then retrieves supported versions for those groups and saves to ~. This is a cached representation of the remote server API in its current form and allows kubectl to handle some client-side validation. 

## kubectl generates objects

since the `run` command can actually deploy multiple resource types, kubectl will try to infer what type of resource to run if it wasn't explicitly specified with the `--generator` flag. For example, resources that have `--restart-policy=Always` are considered Deployments, those with `--restart-policy=Never` are considered pods. 

it also determines other actions, e.g. whether to record the command, or whether this command is just a dry run. if not, it send maps this resource to a specific group and version, then sends it to the API

if an `--expose` flag is provided, it also generates a service object to send to the API.

## creds

kubectl will now try to identify the best location to find your credentials. it follows the following algorithm:

- if a path is specified as a flag, use that
- otherwise $KUBECONFIG
- otherwise look in a predictable home dir location, and use the first file found

it then determines the current context, then the cluster to point to, and auth information. command flags take precedence over those defined in kubeconfig.

objects are sent to the API with the necessary auth information encoded into the HTTP request. for the following auth types:

- x509 certificates are sent using golang's tlsconfig
- bearer tokens are sent with the "Authorization" header
- the openid auth process is handled manually by the operator beforehand, producing a token which is sent like a bearer token

if TLS is enabled, the api server's root CA cert is also sent in the request.

## authentication

when the apiserver first starts, it parses the configuration given to it and assembles a list of suitable authenticators. for example, if a client CA has been passed in, it appends the x509 authenticator. ANOTHER EXAMPLE. a union is then performed which takes an incoming request and runs it against the entire authentication until one succeeds. if all fail, an aggregate error is returned.

for example, the x509 auth handler will verify that the HTTP request is encoded with a TLS key signed by the CA root cert provided to the apiserver. the bearer token handler will verify that the provided token (specified in the HTTP Authorization header) exists in the provided file on disk. the password auth handler will similarly ensure that the HTTP request's basic auth credentials match its own local state.

if authenticator approves the request, it results in an Unauthorized response and goes no further down the chain.

if authentication succeeds, the Authorization header is deleted from the request, and user information is added to the request context store. this allows more handlers down the chain from accessing the previously established identity of the user. these handlers include: authorization (described next), in flight limit, impersonation, auditing, authorized, CORS, timeout for non-long running requests, and any admission plugins that are auto-loaded by the controller manager or run separately in userspace.

## authorization

after authentication, the next step is authorization. the apiserver parses configuration and registers authorizers to a union chain, in the same way it did for authenticators. this means that for every request that comes in, it iterates through this list and checks to see whether the request passes an authenticator's own checks. if a single authorizer approves, the request proceeds.  if all authorizers deny the request, the request results in a Forbidden response and goes no further down the chain.

supported authorizers as of v1.8 are AllowAll and DenyAll (which approve and deny all traffic respectively), webhook (interacts with an off-cluster HTTP(S) service), ABAC (enforces policies defined in a static file), RBAC (enforces RBAC roles which are added by the admin as k8s resources) and Node (which ensures that nodes, i.e. the kubelet, can only access resources hosted on itself).

some authorizers as you might have noticed are dynamic (RBAC and Node) and therefore need to retrieve cluster state. for example, in the case of RBAC, when a request comes in and has been authenticated, it retrieves user information set by the authenticator and looks up to see whether that user has any associated roles which permit them to do what they want to do. accessing roles requires a list operation to be performed against the API server. to solve this problem, kubernetes uses the concept of an informer.

an informer is a way for external controllers to subscribe to storage events. it provides an abstraction for subscribers to listen out for API server events (creation, update, delete) in a threadsafe manner, without having to worry about stepping on anybody else's toes. it does this by accessing a local cached representation of the resource it's following. This saves us connections against the API server, duplicate serialization costs server-side, duplicate deserialization costs controller-side, and duplicate caching costs controller-side. informers provide convenience methods to list and retrieve specific resources, which is useful in the case of RBAC.

## admission controllers

whilst authorization is focused on answering if a user is authorized to perform an action, admission Control is focused on if the system will accept an authorized action. Kubernetes may choose to dismiss an authorized action based on any number of admission control strategies. **** CHANGE For example, you can have an admission plug-in enforcing all container images to come from a particular registry, and prevent other images from being deployed in pods. There are quite many admission controllers providing functionality such as enforcing limits, applying pre-create checks, and setting up default values for missing fields. ****

the next stage of the request lifecycle concerns admission controllers. these are hooks that validate a request at the final stages before it's persisted to etcd. admission controllers are defined through a runtime flag to the apiserver. when a name is provided, the apiserver initiliazes it and adds it to a chain. it then does a union and executes each one when a request comes in. 

admission controllers are stored as plugins in `plugin/pkg/admission` and are compiled into kubernetes. they have to satisfy a small interface. if the controller performs validation and decides that the incoming request cannot meet its requirement, it returns an error which is caught by the server and rendered into a HTTP response. if a single admission controller fails, the chain is broken and the whole request will fail.

admission controllers are usually categorised into resource management, security, defaulting, and referential consistency. commonly used resource ACs are: `InitialResources` which sets default resource limits to the resources for a container based on past usage; `LimitRanger` which sets defaults for container requests and limits, or enforce upper bounds on certain resources (no more than 2GB of memory, default to 512MB); and `ResourceQuota` which calculates and denies a number of objects (pods, rc, service load balancers) or total consumed resources (cpu, memory, disk) in a namespace.

## each object save to etcd

when the apiserver first starts, it maps the REST API and registers handlers for every expected API operation. in our case, kubectl will have converted `kubectl run` to a POST request. since this request matches an expected pattern, when the api server receives the request it will map it to a create handler which is responsible for request validation and interacting with the storage abstraction.

the create handler will read the request and deserialize the data. before passing this on the storage adapter, it will invoke performance and audit tasks that are used to gain insight into the system. 

the storage registry then assembles the storage key for the object. this is configurable and is usually done by appending the object's name to the namespace, e.g. `<namespace>/<name>`. it then persists the object to the database and checks for any create errors. it then performs a get to ensure the object was created, then invokes any post-create handlers and decorators if additional finalization is required.

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
1. if post-start container lifecycle hooks are registered, run them. Hooks can either be of the type `Exec` (executes a specific command inside the container) or `HTTP` (performs a HTTP request against a container endpoint). If the PostStart hook takes too long to run, hangs, or fails, the container will never reach a `running` state