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

## kubelet deploys container

## create service

## steps authn->etcd repeated

## endpoint created

## kube-proxy writes iptables rules

## dns server adds A/SRV records