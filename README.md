# What happens when ... Kubernetes edition!

Imagine I want to deploy nginx to a Kubernetes cluster. I'd probably type something like this in my terminal:

```bash
kubectl run nginx --image=nginx --replicas=3
```

and hit enter. After a few seconds, I should see three nginx pods spread across all my worker nodes. It works like magic, and that's great! But what's really going on under the hood?

One of the awesome things about Kubernetes is that it handles the deployment of workloads across infrastructure through user-friendly APIs. Complexity is hidden by simple abstractions. But in order to fully understand the value it offers us, it's also useful to understand its internals. This guide will lead you through the full lifecycle of a request from the client to the kubelet, linking off to the source code where necessary to illustrate what's going on.

This is a living document. If you spot areas that can be improved or rewritten, contributions are welcome!

## contents

1. [kubectl](#kubectl)
   - [validation and generators](#validation-and-generators)
   - [API groups and version negotiation](#api-groups-and-version-negotiation)
   - [client-side auth](#client-auth)
1. [kube-apiserver](#kube-apiserver)
   - [authentication](#authentication)
   - [authorization](#authorization)
   - [admission control](#admission-control)
1. [etcd](#etcd)
1. [initializers](#initializers)
1. [control loops](#control-loops)
   - [deployments controller](#deployments-controller)
   - [replicasets controller](#replicasets-controller)
   - [informers](#informers)
   - [scheduler](#scheduler)
1. [kubelet](#kubelet)
   - [pod sync](#pod-sync)
   - [CRI and pause containers](#cri-and-pause-containers)
   - [CNI and pod networking](#cni-and-pod-networking)
   - [inter-host networking](#inter-host-networking)
   - [container startup](#container-startup)
1. [wrap-up](#wrap-up)

## kubectl

### Validation and generators

Okay, so let's begin. We've just hit enter in our terminal. What now?

The first thing that kubectl will do is perform some client-side validation. This ensures that requests that will _always_ fail (e.g. creating a non-supported resource or using a [malformed image name](https://github.com/kubernetes/kubernetes/blob/v1.14.0/pkg/kubectl/cmd/run/run.go#L264)) will fail fast and not be sent to kube-apiserver. This improves system performance by reducing unnecessary load.

After validation, kubectl begins assembling the HTTP request it will send to kube-apiserver. All attempts to access or change state in the Kubernetes system goes through the API server, which in turns communicates with etcd. The kubectl client is no different. To construct the HTTP request, kubectl uses something called [generators](https://kubernetes.io/docs/user-guide/kubectl-conventions/#generators) which is an abstraction that takes care of serialization.

What may not be obvious is that you can actually specify multiple resource types with `kubectl run`, not just Deployments. To make that work, kubectl will [infer](https://github.com/kubernetes/kubernetes/blob/v1.14.0/pkg/kubectl/cmd/run/run.go#L319-L339) the resource type if the generator name wasn't explicitly specified using the `--generator` flag.

For example, resources that have `--restart-policy=Always` are considered Deployments, and those with `--restart-policy=Never` are considered Pods. kubectl will also figure out whether other actions need to be triggered, such as recording the command (for rollouts or auditing), or whether this command is just a dry run (indicated by the `--dry-run` flag).

After realising that we want to create a Deployment, it will use the `DeploymentAppsV1` generator to generate a [runtime object](https://github.com/kubernetes/kubernetes/blob/v1.14.0/pkg/kubectl/generate/versioned/run.go#L237) from our provided parameters. A "runtime object" is a generic term for a resource.

### API groups and version negotiation

What's worth pointing out before we continue is that Kubernetes uses a _versioned_ API that is categorised into "API groups". An API group is meant to categorise similar resources so that they're easier to reason about. It also provides a better alternative to a singular monolithic API. The API group of a Deployment is named `apps`, and its most recent version is `v1`. This is why you need type `apiVersion: apps/v1` at the top of your Deployment manifests.

Anyhow... After kubectl generates the runtime object, it starts to [find the appropriate API group and version](https://github.com/kubernetes/kubernetes/blob/v1.14.0/pkg/kubectl/cmd/run/run.go#L674-L686) for it and then [assembles a versioned client](https://github.com/kubernetes/kubernetes/blob/v1.14.0/pkg/kubectl/cmd/run/run.go#L705-L708) that is aware of the various REST semantics for the resource. This discovery stage is called version negotiation and involves kubectl scanning the `/apis` path on the remote API to retrieve all possible API groups. Since kube-apiserver exposes its schema document (in OpenAPI format) at this path, it's easy for clients to perform their own discovery.

To improve performance, kubectl also [caches the OpenAPI schema](https://github.com/kubernetes/kubernetes/blob/v1.14.0/staging/src/k8s.io/cli-runtime/pkg/genericclioptions/config_flags.go#L234) to the `~/.kube/cache/discovery` directory. If you want to see this API discovery in action, try deleting that directory and running a command with the `-v` flag turned to the max. You'll see all the HTTP requests which are trying to find those API versions. There are lots!

The final step is to actually [send](https://github.com/kubernetes/kubernetes/blob/v1.14.0/pkg/kubectl/cmd/run/run.go#L709) the HTTP request. Once it does so and gets back a successful response, kubectl will then print out a success message [based on the desired output format](https://github.com/kubernetes/kubernetes/blob/v1.14.0/pkg/kubectl/cmd/run/run.go#L459).

### Client auth

One thing that we didn't mention in the previous step is client authentication (this is handled before the HTTP request is sent), so let's look at that now.

In order to send the request successfully, kubectl needs to be able to authenticate. User credentials are almost always stored in the `kubeconfig` file which resides on disk, but that file can be stored in different locations. To locate it, kubectl does the following:

- if a `--kubeconfig` flag is provided, use that.
- if the `$KUBECONFIG` environment variable is defined, use that.
- otherwise look in a [predictable home directory](https://github.com/kubernetes/client-go/blob/master/tools/clientcmd/loader.go#L52) like `~/.kube`, and use the first file found.

After parsing the file, it then determines the current context to use, the current cluster to point to, and any auth information associated with the current user. If the user provided flag-specific values (such as `--username`) these take precedence and will override those specified in kubeconfig. Once it has this information, kubectl populates the client's configuration so that it is able to decorate the HTTP request appropriately:

- x509 certificates are sent using [`tls.TLSConfig`](https://github.com/kubernetes/client-go/blob/82aa063804cf055e16e8911250f888bc216e8b61/rest/transport.go#L80-L89) (this also includes the root CA)
- bearer tokens are [sent](https://github.com/kubernetes/client-go/blob/c6f8cf2c47d21d55fa0df928291b2580544886c8/transport/round_trippers.go#L314) in the "Authorization" HTTP header
- username and password are [sent](https://github.com/kubernetes/client-go/blob/c6f8cf2c47d21d55fa0df928291b2580544886c8/transport/round_trippers.go#L223) via HTTP basic authentication
- the OpenID auth process is handled manually by the user beforehand, producing a token which is sent like a bearer token

## kube-apiserver

### Authentication

So our request has been sent, hooray! What next? This is where kube-apiserver enters the picture. As we've already mentioned, kube-apiserver is the primary interface that clients and system components use to persist and retrieve cluster state. To perform its function, it needs to be able to verify that the requester is who they say there are. This process is called authentication.

How does the apiserver authenticate requests? When the server first starts, it looks at all the [CLI flags](https://kubernetes.io/docs/admin/kube-apiserver/) the user provided and assembles a list of suitable authenticators. Let's take an example: if a `--client-ca-file` has been passed in, it appends the x509 authenticator; if it sees `--token-auth-file` provided, it appends the token authenticator to the list. Every time a request is received, it is [run through the authenticator chain until one succeeds](https://github.com/kubernetes/apiserver/blob/51bebaffa01be9dc28195140da276c2f39a10cd4/pkg/authentication/request/union/union.go#L54): 

- the [x509 handler](https://github.com/kubernetes/apiserver/blob/51bebaffa01be9dc28195140da276c2f39a10cd4/pkg/authentication/request/x509/x509.go#L60) will verify that the HTTP request is encoded with a TLS key signed by the CA root cert
- the [bearer token handler](https://github.com/kubernetes/apiserver/blob/51bebaffa01be9dc28195140da276c2f39a10cd4/pkg/authentication/request/bearertoken/bearertoken.go#L38) will verify that the provided token (specified in the HTTP Authorization header) exists in the file on disk specified by `--token-auth-file`
- the [basicauth handler](https://github.com/kubernetes/apiserver/blob/51bebaffa01be9dc28195140da276c2f39a10cd4/plugin/pkg/authenticator/request/basicauth/basicauth.go#L37) will similarly ensure that the HTTP request's basic auth credentials match its own local state.

If _every_ authenticator fails, [the request fails](https://github.com/kubernetes/apiserver/blob/20bfbdf738a0643fe77ffd527b88034dcde1b8e3/pkg/authentication/request/union/union.go#L71) and an aggregate error is returned. If authentication succeeds, the `Authorization` header is removed from the request, and [user information is added](https://github.com/kubernetes/apiserver/blob/e30df5e70ef9127ea69d607207c894251025e55b/pkg/endpoints/filters/authentication.go#L71-L75) to its context. This gives future steps (such as authorization and admission controllers) the ability to access the previously established identity of the user. 

### Authorization

Okay, the request has been sent, and kube-apiserver has successfully verified we are who we say we are. What a relief! However, we're not done yet. We may be who we say we are, but do we have the permissions to perform this action? Identity and permission are not the same thing, after all. In order for us to continue, kube-apiserver needs to authorize us.

The way kube-apiserver handles authorization is very similar to authentication: based on flag inputs, it will assemble a chain of authorizers that will be run against every incoming request. If _all_ authorizers deny the request, the request results in a `Forbidden` response and [goes no further](https://github.com/kubernetes/apiserver/blob/e30df5e70ef9127ea69d607207c894251025e55b/pkg/endpoints/filters/authorization.go#L60). If a single authorizer approves, the request proceeds.

Some examples of authorizers that ship with v1.8 are:

- [webhook](https://github.com/kubernetes/apiserver/blob/d299c880c4e33854f8c45bdd7ab599fb54cbe575/plugin/pkg/authorizer/webhook/webhook.go#L143), which interacts with an off-cluster HTTP(S) service;
- [ABAC](https://github.com/kubernetes/kubernetes/blob/77b83e446b4e655a71c315ad3f3890dc2a220ccf/pkg/auth/authorizer/abac/abac.go#L223), which enforces policies defined in a static file;
- [RBAC](https://github.com/kubernetes/kubernetes/blob/8db5ca1fbb280035b126faf0cd7f0420cec5b2b6/plugin/pkg/auth/authorizer/rbac/rbac.go#L43), which enforces RBAC roles which are added by the admin as k8s resources
- [Node](https://github.com/kubernetes/kubernetes/blob/dd9981d038012c120525c9e6df98b3beb3ef19e1/plugin/pkg/auth/authorizer/node/node_authorizer.go#L67), which ensures that node clients, i.e. the kubelet, can only access resources hosted on itself.

Check out the `Authorize` method for each one to see how they work!

### Admission control

Okay, so by this point we've authenticated and have been authorized by the kube-apiserver. So what's left? From kube-apiserver's perspective, it believes who we are and permits us to continue, but with Kubernetes, other parts of the system have strong opinions about what should and should not be permitted to happen. This is where [admission controllers](https://kubernetes.io/docs/admin/admission-controllers/#what-are-they) enter the picture.

Whilst authorization is focused on answering whether a user has permission, admission controllers intercept the request to ensure that it matches the wider expectations and rules of the cluster. They are the last bastion of control before an object is persisted to etcd, so they encapsulate the remaining system checks to ensure an action does not produce unexpected or negative results.

The way admission controllers work is similar to way authenticators and authorizers work, but there is one difference: unlike authenticator and authorizers chains, if a single admission controller fails, the whole chain is broken and the request will fail. 

What's really cool about the design of admission controllers is its focus on promoting extensibility. Each controller is stored as a plugin in the [`plugin/pkg/admission` directory](https://github.com/kubernetes/kubernetes/tree/master/plugin/pkg/admission), and is made to satisfy a small interface. Each one is then compiled into main kubernetes binary itself. 

Admission controllers are usually categorised into resource management, security, defaulting, and referential consistency. Here are some examples of admission controllers that just take care of resource management:

- `InitialResources` sets default resource limits to the resources for a container based on past usage; 
- `LimitRanger` sets defaults for container requests and limits, or enforce upper bounds on certain resources (no more than 2GB of memory, default to 512MB); 
- `ResourceQuota` calculates and denies a number of objects (pods, rc, service load balancers) or total consumed resources (cpu, memory, disk) in a namespace.

## etcd

By this point, Kubernetes has fully vetted the incoming request and has permitted it to go forth and prosper. In the next step, kube-apiserver deserializes the HTTP request, constructs runtime objects from them (kinda like the inverse process of kubectl's generators), and persists them to the datastore. Let's break that down a bit.

How does kube-apiserver know what to do when it accepts our request? Well, there's a pretty complicated series of steps that happen _before_ any requests are served. Let's start at the beginning, from when the binary is first run:

1. When the `kube-apiserver` binary is run, it [creates a server chain](https://github.com/kubernetes/kubernetes/blob/master/cmd/kube-apiserver/app/server.go#L119), which allows apiserver aggregation. This is basically a way of supporting multiple apiservers (we don't need to worry about this).
1. When this happens, a [generic apiserver is created](https://github.com/kubernetes/kubernetes/blob/master/cmd/kube-apiserver/app/server.go#L149) that serves as a default implementation. 
1. The generated OpenAPI schema populates the [apiserver's configuration](https://github.com/kubernetes/apiserver/blob/7001bc4df8883d4a0ec84cd4b2117655a0009b6c/pkg/server/config.go#L149).
1. kube-apiserver then iterates over all the API groups specified in the schema and configures a [storage provider](https://github.com/kubernetes/kubernetes/blob/c7a1a061c3dc5acabcc0c35b3b96a6935dccf546/pkg/master/master.go#L410) for each that serves as a generic storage abstraction. This is what kube-apiserver talks to when it accesses or mutates the state of a resource.
1. For every API group it also iterates over each of the group versions and [installs the REST mappings](https://github.com/kubernetes/apiserver/blob/7001bc4df8883d4a0ec84cd4b2117655a0009b6c/pkg/endpoints/groupversion.go#L92) for every HTTP route. This allows kube-apiserver to map requests and be able to delegate off to the correct logic once it finds a match.
1. For our specific use case, a [POST handler](https://github.com/kubernetes/apiserver/blob/7001bc4df8883d4a0ec84cd4b2117655a0009b6c/pkg/endpoints/installer.go#L710) is registered, which in turn will delegate to a [create resource handler](https://github.com/kubernetes/apiserver/blob/7001bc4df8883d4a0ec84cd4b2117655a0009b6c/pkg/endpoints/handlers/create.go#L37).

By this point, kube-apiserver is fully aware of what routes exist and has an internal mapping of which handlers and storage providers to invoke if a request matches. What a smart cookie. Now let's imagine our HTTP request has flown in:

1. If the handler chain can match the request to a set pattern (i.e. to the routes we registered), it will [dispatch the dedicated handler](https://github.com/kubernetes/apiserver/blob/7001bc4df8883d4a0ec84cd4b2117655a0009b6c/pkg/server/handler.go#L143) that was registered for the route. Otherwise it falls back to a [path-based handler](https://github.com/kubernetes/apiserver/blob/7001bc4df8883d4a0ec84cd4b2117655a0009b6c/pkg/server/mux/pathrecorder.go#L248) (this is what happens when you call `/apis`). If no handlers are registered for that path, a [not found handler](https://github.com/kubernetes/apiserver/blob/7001bc4df8883d4a0ec84cd4b2117655a0009b6c/pkg/server/mux/pathrecorder.go#L254) is invoked which results in a 404.
1. Luckily for us, we have a registered route called [`createHandler`](https://github.com/kubernetes/apiserver/blob/7001bc4df8883d4a0ec84cd4b2117655a0009b6c/pkg/endpoints/handlers/create.go#L37)! What does it do? Well it will first decode the HTTP request and perform basic validation, such as ensuring the JSON they provided correlates with our expectation of the versioned API resource.
1. Auditing and final admission [will occur](https://github.com/kubernetes/apiserver/blob/7001bc4df8883d4a0ec84cd4b2117655a0009b6c/pkg/endpoints/handlers/create.go#L93-L104). 
1. The resource will be [saved to etcd](https://github.com/kubernetes/apiserver/blob/7001bc4df8883d4a0ec84cd4b2117655a0009b6c/pkg/endpoints/handlers/create.go#L111) by [delegating to the storage provider](https://github.com/kubernetes/apiserver/blob/19667a1afc13cc13930c40a20f2c12bbdcaaa246/pkg/registry/generic/registry/store.go#L327). Usually the etcd key will be the form of `<namespace>/<name>`, but this is configurable.
1. Any create errors are caught and, finally, the storage provider performs a `get` call to ensure the object was actually created. It then invokes any post-create handlers and decorators if additional finalization is required.
1. The HTTP response [is constructed](https://github.com/kubernetes/apiserver/blob/7001bc4df8883d4a0ec84cd4b2117655a0009b6c/pkg/endpoints/handlers/create.go#L131-L142) and sent back.

That's a lot of steps! It's pretty amazing to follow the crumbs down the rabbit hole because we realise how much work the apiserver actually does. So to summarise: our Deployment resource now exists in etcd. But just to throw a spanner in the works, you won't be able to see it yet...

## Initializers

After an object is persisted to the datastore, it is not made fully visible by the apiserver or scheduled until a series of [intializers](https://kubernetes.io/docs/admin/extensible-admission-controllers/#initializers) have run. An initializer is a controller that is associated with a resource type and performs logic on the resource before it's made available to the outside world. If a resource type has zero initializers registered, this initialization step is skipped and resources are made visible immediately. 

As [many great blog posts](https://ahmet.im/blog/initializers/) have pointed out, this is a powerful feature because it allows us to perform generic bootstrap operations. Examples could be:

- Injecting a proxy sidecar container into Pod that expose port 80, or feature a particular annotation.
- Injecting a volume with test certificates to all Pods in a specific namespace.
- If a Secret is shorter than 20 characters (e.g. a password), prevent its creation.

`initializerConfiguration` objects allow you to declare which initializers should run for certain resource types. Imagine we want a custom initializer to run every time a Pod is created, we'd do something like this:

```
apiVersion: admissionregistration.k8s.io/v1alpha1
kind: InitializerConfiguration
metadata:
  name: custom-pod-initializer
initializers:
  - name: podimage.example.com
    rules:
      - apiGroups:
          - ""
        apiVersions:
          - v1
        resources:
          - pods
```

After creating this config, it will append `custom-pod-initializer` to every Pod's `metadata.initializers.pending` field. The initializer controller would already be deployed and would be routinely scanning for new Pods. When the initializer detects one with its name in the Pod's pending field, it will perform its logic. After it completes its process, it removes its name from the pending list. Only initializers whose name is first in the list may operate on the resource. When all initializers finish and the `pending` field is empty, the object will be considered initialized.

The eagle-eyed of you may have a spotted a potential problem. How can a userland controller process resources if those resources are not made visible by kube-apiserver? To get around this problem, kube-apiserver exposes a `?includeUninitialized` query parameter which returns _all_ objects, even unitialized ones. 

## Control loops

### Deployments controller

By this stage, our Deployment record exists in etcd and any initialization logic has completed. The next stages involve us setting up the resource topology that Kubernetes relies on. When we think about it, a Deployment is really just a collection of ReplicaSets, and a ReplicaSet is a collection of Pods. So how does Kubernetes go about creating this hierarchy from one HTTP request? This is where Kubernetes' built-in controllers take over.

Kubernetes makes strong use of "controllers" throughout the system. A controller is an asychronous script that works to reconcile the 
current state of the Kubernetes system to a desired state. Each controller has a small responsibility and is run in parallel by the `kube-controller-manager` component. Let's introduce ourselves to the first one that takes over, the Deployment controller.

After a Deployment record is stored to etcd and initialized, it is made visible via kube-apiserver. When this new resource is available, it is detected by the Deployment controller, whose job it is to listen out for changes to Deployment records. In our case, the controller [registers a specific callback](https://github.com/kubernetes/kubernetes/blob/master/pkg/controller/deployment/deployment_controller.go#L122) for create events via an informer (see below for more information about what this is). 

This handler will be executed when our Deployment first becomes available and will start by [adding the object to an internal work queue](https://github.com/kubernetes/kubernetes/blob/master/pkg/controller/deployment/deployment_controller.go#L170). By the time it gets around to processing our object, the controller will [inspect our Deployment](https://github.com/kubernetes/kubernetes/blob/master/pkg/controller/deployment/deployment_controller.go#L572) and [realise](https://github.com/kubernetes/kubernetes/blob/master/pkg/controller/deployment/deployment_controller.go#L633) that there are no ReplicaSet or Pod records associated with it. It does this by querying kube-apiserver with label selectors. What's interesting to note is that this sychronization process is state agnostic: it reconciles new records in the same way as existing ones.

After realising none exist, it will begin a [scaling process](https://github.com/kubernetes/kubernetes/blob/master/pkg/controller/deployment/sync.go#L385) to start resolving state. It does this by rolling out (i.e. creating) a ReplicaSet resource, assigning it a label selector, and giving it the revision number of 1. The ReplicaSet's PodSpec is copied from the Deployment's manifest, as well as other relevant metadata. Sometimes the Deployment record will need to be updated after this as well (for instance if the progress deadline is set). 

The [status is then updated](https://github.com/kubernetes/kubernetes/blob/master/pkg/controller/deployment/sync.go#L70) and it then re-enters the same reconciliation loop waiting for the deployment to match a desired, completed state. Since the Deployment controller is only concerned about creating ReplicaSets, this reconcilation stage needs to be continued by the next controller, the ReplicaSet controller. 

### ReplicaSets controller

In the previous step, the Deployments controller created our Deployment's first ReplicaSet but we still have no Pods. This is where the ReplicaSet controller comes into play! Its job is to monitor the lifecycle of ReplicaSets and their dependent resources (Pods). Like most other controllers, it does this by triggering handlers on certain events.

The event we're interested in is creation. When a ReplicaSet is created (courtesy of the Deployments controller) the RS controller [inspects the state](https://github.com/kubernetes/kubernetes/blob/master/pkg/controller/replicaset/replica_set.go#L583) of the new ReplicaSet and realizes there is a skew between what exists and what is required. It then seeks to reconcile this state by [bumping the number of pods](https://github.com/kubernetes/kubernetes/blob/master/pkg/controller/replicaset/replica_set.go#L460) that belong to the ReplicaSet. It starts creating them in a careful manner, ensuring that the ReplicaSet's burst count (which it inherited from its parent Deployment) is always matched. 

Create operations for Pods [are also batched](https://github.com/kubernetes/kubernetes/blob/master/pkg/controller/replicaset/replica_set.go#L487), starting with `SlowStartInitialBatchSize` and doubling with each successful iteration in a kind of "slow start" operation. This aims to mitigate the risk of swamping kube-apiserver with unnecessary HTTP requests when there are numerous pod bootup failures (for example, due to resource quotas). If we're going to fail, we might as well fail gracefully with minimal impact on other system components! 

Kubernetes enforces object hierarchies through Owner References (a field in the child resource where it references the ID of its parent). Not only does this ensure that child resources are garbage-collected once a resource managed by the controller is deleted (cascading deletion), it also provides an effective way for parent resources to not fight over their children (imagine the scenario where two potential parents think they own the same child!). 

Another subtle benefit of the Owner Reference design is that it's stateful: if any controller were to restart, that downtime would not affect the wider system since resource topology is independent of the controller. This focus on isolation also creeps in to the design of controllers themselves: they should not operate on resources they don't explicitly own. Controllers should instead be selective in its ownership assertions, non-interfering, and non-sharing. 

Anyway, back to owner references! Sometimes there are "orphaned" resources in the system which usually happens when:

1. a parent is deleted but not its children
2. garbage collection policies prohibit child deletion

When this occurs, controllers will ensure that orphans are adopted by a new parent. Multiple parents can race to adopt a child, but only one will be successful (the others will receive a validation error). 

### Informers

As you might have noticed, some controllers like the RBAC authorizer or the Deployment controller need to retrieve cluster state to function. To return to the example of the RBAC authorizer, we know that when a request comes in, the authenticator will save an initial representation of user state for later use. The RBAC authorizer will then use this to retrieve all the roles and role bindings that are associated with the user in etcd. How are controllers supposed to access and modify such resources? It turns out this is a common use case and is solved in Kubernetes with informers. 

An informer is a pattern that allows controllers to subscribe to storage events and easily list resources they're interested in. Apart from providing an abstraction which is nice to work with, it also takes care of a lot of the nuts and bolts such as caching (caching is important because it reduces unnecessary kube-apiserver connections, and reduces duplicate serialization costs server- and controller-side). By using this design, it also allows controllers to interact in a threadsafe manner without having to worry about stepping on anybody else's toes. 

For more information about how informers work in relation to controllers, check out [this blog post](http://borismattijssen.github.io/articles/kubernetes-informers-controllers-reflectors-stores).

### Scheduler

After all the controllers have run, we have a Deployment, a ReplicaSet and three Pods stored in etcd and available through kube-apiserver. Our pods, however, are stuck in a `Pending` state because they have not yet been scheduled to a Node. The final controller that resolves this is the scheduler.

The scheduler runs as a standalone component of the control plane and operates in the same way as other controllers: it listens out for events and attempts to reconcile state. In this case, it [filters pods](https://github.com/kubernetes/kubernetes/blob/master/plugin/pkg/scheduler/factory/factory.go#L190) that have an empty `NodeName` field in their PodSpec and attempts to find a suitable Node that the pod can reside on. 

In order to find a suitable node, a specific scheduling algorithm is used. The way the default scheduling algorithm works is the following:

1. When the scheduler starts, a [chain of default predicates are registered](https://github.com/kubernetes/kubernetes/blob/2d64ce5e8e45e26b02492d2b6c85e5ebfb1e4761/plugin/pkg/scheduler/algorithmprovider/defaults/defaults.go#L65-L81). These predicates are effectively functions that, when evaluated, [filter Nodes](https://github.com/kubernetes/kubernetes/blob/2d64ce5e8e45e26b02492d2b6c85e5ebfb1e4761/plugin/pkg/scheduler/core/generic_scheduler.go#L117) based on their suitability to host a pod. For example, if the PodSpec explicitly requests CPU or RAM resources, and a Node cannot meet these requests due to lack of capacity, it will be deselected for the Pod (resource capacity is calculated as the _total capacity_ minus the _sum of the resource requests_ of currently running containers).

1. Once appropriate nodes have been selected, a series of [priority functions are run](https://github.com/kubernetes/kubernetes/blob/2d64ce5e8e45e26b02492d2b6c85e5ebfb1e4761/plugin/pkg/scheduler/core/generic_scheduler.go#L354-L360) against the remaining Nodes in order to rank their suitability. For example, in order to spread workloads across the system, it will favour nodes that have fewer resource requests than others (since this indicates less workloads running). As it runs these functions, it assigns each node a numerical rank. The highest ranked node is then selected for scheduling.

Once the algorithm finds a node, the scheduler then [creates a Binding](https://github.com/kubernetes/kubernetes/blob/2d64ce5e8e45e26b02492d2b6c85e5ebfb1e4761/plugin/pkg/scheduler/scheduler.go#L336-L342) object whose Name and UID match the Pod, and whose ObjectReference field contains the name of the selected Node. This is then [sent to the apiserver](https://github.com/kubernetes/kubernetes/blob/2d64ce5e8e45e26b02492d2b6c85e5ebfb1e4761/plugin/pkg/scheduler/factory/factory.go#L1095) via a POST request.

When kube-apiserver receives this Binding object, the registry deserializes the object and updates the following fields on the Pod object: it [sets the NodeName](https://github.com/kubernetes/kubernetes/blob/2d64ce5e8e45e26b02492d2b6c85e5ebfb1e4761/pkg/registry/core/pod/storage/storage.go#L170) to the one in the ObjectReference, it [adds any relevant annotations](https://github.com/kubernetes/kubernetes/blob/2d64ce5e8e45e26b02492d2b6c85e5ebfb1e4761/pkg/registry/core/pod/storage/storage.go#L174-L176), and [sets](https://github.com/kubernetes/kubernetes/blob/2d64ce5e8e45e26b02492d2b6c85e5ebfb1e4761/pkg/registry/core/pod/storage/storage.go#L177-L180) its `PodScheduled` status condition to `True`. 

Once the scheduler has scheduled a Pod to a Node, the kubelet on that Node can take over to begin the deployment. Exciting!

**Side note: customising the scheduler:** What's interesting is that both predicate and priority functions are extensible and can be defined by using the `--policy-config-file` flag. This introduces a degree of flexibility. Administrators can also run custom schedulers (controllers with custom processing logic) in standalone Deployments. If a PodSpec contains `schedulerName`, Kubernetes will hand over scheduling for that pod to whatever scheduler thas has registered itself under that name.

## kubelet 

### Pod sync

Okay, the main controller loop has finished, phew! Let's summarise: the HTTP request passed authentication, authorization, and admission control stages; a Deployment, ReplicaSet, and three Pod resources were persisted to etcd; a series of initializers ran; and, finally, each Pod was scheduled to a suitable node. So far, however, the state we've been reasoning about exists purely in etcd. The next steps involve distributing state across the worker nodes, which is the whole point of a distributed system like Kubernetes! The way this happens is through a component called the kubelet. Let's begin!

The kubelet is an agent that runs on every node in a Kubernetes cluster and is responsible for, among other things, managing the lifecycle of Pods. This means it handles all of the translation logic between the abstraction of a "Pod" (which is really just a Kubernetes concept) and its building blocks, containers. It also handles all of the associated logic around mounting volumes, container logging, garbage collection, and many more important things.

A useful way of thinking about the kubelet is again like a controller! It queries Pods from kube-apiserver every 20 seconds (this is configurable), filtering the ones whose `NodeName` [matches the name](https://github.com/kubernetes/kubernetes/blob/3b66adb8bc6929e1205bcb2bc32f380c39be8381/pkg/kubelet/config/apiserver.go#L34) of the node the kubelet is running on. Once it has that list, it detects new additions by comparing against its own internal cache and begins to synchronise state if any discrepencies exist. Let's take a look at what that synchronization process looks like:

1. If the pod is being created (ours is!), it [registers some startup metrics](https://github.com/kubernetes/kubernetes/blob/fc8bfe2d8929e11a898c4557f9323c482b5e8842/pkg/kubelet/kubelet.go#L1519) that is used in Prometheus for tracking pod latency.
1. It then [generates a PodStatus](https://github.com/kubernetes/kubernetes/blob/dd9981d038012c120525c9e6df98b3beb3ef19e1/pkg/kubelet/kubelet_pods.go#L1287) object, which represents the state of a Pod's current Phase. The Phase of a Pod is a high-level summary of where the Pod is in its lifecycle. Examples include `Pending`, `Running`, `Succeeded`, `Failed` and `Unknown`. Generating this state is quite complicated, so let's dive into exactly what happens:
    - first, a chain of `PodSyncHandlers` is executed sequentially. Each handler checks whether the Pod should still reside on the node. If any of them decide that the Pod no longer belongs there, the Pod's phase [will change](https://github.com/kubernetes/kubernetes/blob/dd9981d038012c120525c9e6df98b3beb3ef19e1/pkg/kubelet/kubelet_pods.go#L1293-L1297) to `PodFailed` and it will eventually be evicted from the Node. Examples of these include evicting a Pod after its `activeDeadlineSeconds` has exceeded (used during Jobs).
    - next, the Pod's Phase is determined by the status of its init and real containers. Since our containers have not been started yet, the containers are classed as [waiting](https://github.com/kubernetes/kubernetes/blob/fc8bfe2d8929e11a898c4557f9323c482b5e8842/pkg/kubelet/kubelet_pods.go#L1244). Any Pod with a waiting container has a Phase of [`Pending`](https://github.com/kubernetes/kubernetes/blob/fc8bfe2d8929e11a898c4557f9323c482b5e8842/pkg/kubelet/kubelet_pods.go#L1258-L1261).
    - finally, the Pod Condition is determined by the condition of its containers. Since none of our containers have been created by the container runtime yet, it will [set the `PodReady` condition to False](https://github.com/kubernetes/kubernetes/blob/fc8bfe2d8929e11a898c4557f9323c482b5e8842/pkg/kubelet/status/generate.go#L70-L81).
1. After the PodStatus is generated, it will then be sent to the Pod's status manager, which is tasked with asynchronously updating the etcd record via the apiserver.
1. Next, a series of admission handlers are run to ensure the pod has the correct security permissions. These include enforcing [AppArmor profiles and `NO_NEW_PRIVS`](https://github.com/kubernetes/kubernetes/blob/fc8bfe2d8929e11a898c4557f9323c482b5e8842/pkg/kubelet/kubelet.go#L883-L884). Pods denied at this stage will stay in the `Pending` state indefinitely.
1. If the `cgroups-per-qos` runtime flag has been specified, the kubelet will create cgroups for the pod and apply resource parameters. This is to enable better Quality of Service (QoS) handling for pods.
1. Data directories [are created for the pod](https://github.com/kubernetes/kubernetes/blob/dd9981d038012c120525c9e6df98b3beb3ef19e1/pkg/kubelet/kubelet_pods.go#L772). These include the pod directory (usually `/var/run/kubelet/pods/<podID>`), its volumes directory (`<podDir>/volumes`) and its plugins directory (`<podDir>/plugins`).
1. The volume manager will [attach and wait](https://github.com/kubernetes/kubernetes/blob/2723e06a251a4ec3ef241397217e73fa782b0b98/pkg/kubelet/volumemanager/volume_manager.go#L330) for any relevant volumes defined in `Spec.Volumes`. Depending on the type of volume being mounted, some pods will need to wait longer (e.g. cloud or NFS volumes).
1. All secrets defined in `Spec.ImagePullSecrets` are [retrieved from the apiserver](https://github.com/kubernetes/kubernetes/blob/dd9981d038012c120525c9e6df98b3beb3ef19e1/pkg/kubelet/kubelet_pods.go#L788) so that they can later be injected into the container.
1. The container runtime then runs the container (described in more detail below).

### CRI and pause containers

We're at the point now where most of the set-up is done and the container is ready to be launched. The software that does this launching is called the Container Runtime (`docker` or `rkt` are examples).

In an effort to be more extensible, the kubelet since v1.5.0 has been using a concept called CRI (Container Runtime Interface) for interacting with concrete container runtimes. In a nutshell, CRI provides an abstraction between the kubelet and a specific runtime implementation. Communication happens via [protocol buffers](https://github.com/google/protobuf) (it's like a faster JSON) and a [gRPC API](https://grpc.io/) (a type of API well-suited to performing Kubernetes operations). This is an incredibly cool idea because by using a defined contract between the kubelet and the runtime, the actual implementation details of how containers are orchestrated become largely irrelevant. All that matters is the contract. This allows new runtimes to be added with minimal overhead since no core Kubernetes code needs to change!

Enough digressions, let's get back to deploying our container... When a pod is first started, kubelet [invokes the `RunPodSandbox`](https://github.com/kubernetes/kubernetes/blob/2d64ce5e8e45e26b02492d2b6c85e5ebfb1e4761/pkg/kubelet/kuberuntime/kuberuntime_sandbox.go#L51) remote procedure command (RPC). A "sandbox" is a CRI term to describe a set of containers, which in Kubernetes parlance is, you guessed it, a Pod. The term is deliberately vague so it doesn't lose meaning for other runtimes that may not actually use containers (imagine a hypervisor-based runtime where a sandbox might be a VM). 

In our case, we're using Docker. In this runtime, creating a sandbox involves creating a "pause" container. A pause container serves like a parent for all of the other containers in the Pod since it hosts a lot of the pod-level resources that workload containers will end up using. These "resources" are Linux namespaces (IPC, network, PID). If you're not familiar with how containers work in Linux, let's take a quick refresher. The Linux kernel has the concept of namespaces, which allow the host OS to carve out a dedicated set of resources (CPU or memory for example) and offer it to a process as if it's the only thing in the world using them. Cgroups are also important here, since they're the way that Linux governs resource allocation (it's kinda like a cop that polices resource usage). Docker uses both of these Kernel features to host a process that has guaranteed resources and enforced isolation. For more information, check out b0rk's amazing post: [What even is a Container?](https://jvns.ca/blog/2016/10/10/what-even-is-a-container/). 

The "pause" container provides a way to host all of these namespaces and allow child containers to share them. By being a part of the same network namespace, one benefit is that containers in the same pod can refer to one another using `localhost`. The _second_ role of a pause container is related to how PID namespaces work. In these types of namespaces, processes form a hierarchical tree and the "init" process at the top takes responsibility for "reaping" dead processes. For more information on how this works, check out this [great blog post](https://www.ianlewis.org/en/almighty-pause-container). After the pause container has been created, it is checkpointed to disk, and started.

### CNI and pod networking

Our Pod now has its bare bones: a pause container which hosts all of the namespaces to allow inter-pod communication. But how does networking work and how is it set up? 

When the kubelet sets up networking for a pod it delegates the task to a "CNI" plugin. CNI stands for Container Network Interface and operates in a similar way to the Container Runtime Interface. In a nutshell, CNI is an abstraction that allows different network providers to use different networking implementations for containers. Plugins are registered and the kubelet interacts with them by streaming JSON data (config files are located in `/etc/cni/net.d`) to the relevant CNI binary (located in `/opt/cni/bin`) via stdin. This is an example of the JSON configuration:

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

It also specifies additional metadata for pod, such as its name and namespace via the `CNI_ARGS` environment variable.

What happens next is dependent on the CNI plugin, but let's look at the `bridge` CNI plugin:

1. The plugin will first set up a local Linux bridge in the root network namespace to serve all containers on that host
1. It will then insert an interface (one end of a veth pair) into the pause container's network namespace and attach the other end to the bridge. The best way to think about a veth pair is like a big tube: one side is connected to the container and the other side is in the root network namespace, allowing packets to travel inbetween. 
1. It should then assign an IP to the pause container's interface and set up the routes. This will result in the Pod having its own IP address. IP assignment is delegated to the IPAM providers specified to the JSON configuration. 
    - IPAM plugins are similar to main network plugins: they are invoked via a binary and have a standardised interface. Each must determine the IP/subnet of the container's interface, along with the gateway and routes, and return this information back to the main plugin. The most common IPAM plugin is called `host-local` and allocates IP addresses out of a predefined set of address ranges. It stores the state locally on the host filesystem, therefore ensuring uniqueness of IP addresses on a single host.
1. For DNS, the kubelet will specify the internal DNS server IP address to the CNI plugin, which will ensure that the container's `resolv.conf` file is set appropriately. 

Once the process is complete, the plugin will return JSON data back to the kubelet indicating the result of the operation. 

### Inter-host networking

So far we've described how containers connect to the host, but how do hosts communicate? This will obviously happen if two Pods on different machines want to communicate.

This is usually accomplished using a concept called overlay networking, which is a way to dynamically synchronize routes across multiple hosts. One popular overlay network provider is Flannel. When installed, its core responsibility is to provide a layer-3 IPv4 network between multiple nodes in a cluster. Flannel does not control how containers are networked to the host (this is the job of CNI remember), but rather how the traffic is transported _between_ hosts. To do this, it selects a subnet for the host and registers it with etcd. It then keeps a local representation of the cluster routes and encapsulates outgoing packets in UDP datagrams, ensuring it reaches the right host. For more information, check out [CoreOS's documentation](https://github.com/coreos/flannel).

### Container startup

All the networking shenanigans are done and out of the way. What's left? Well we need to actually start out workload containers. 

Once the sandbox has finished initializing and is active, the kubelet can begin creating containers for it. It first [starts any init containers](https://github.com/kubernetes/kubernetes/blob/5adfb24f8f25a0d57eb9a7b158db46f9f46f0d80/pkg/kubelet/kuberuntime/kuberuntime_manager.go#L690) as defined in the PodSpec, and will then start the main containers themselves. The process for doing this is: 

1. [Pull the image](https://github.com/kubernetes/kubernetes/blob/5f9f4a1c5939436fa320e9bc5973a55d6446e59f/pkg/kubelet/kuberuntime/kuberuntime_container.go#L90) for the container. Any secrets that are defined in the PodSpec are used for private registries.
1. [Create the container](https://github.com/kubernetes/kubernetes/blob/5f9f4a1c5939436fa320e9bc5973a55d6446e59f/pkg/kubelet/kuberuntime/kuberuntime_container.go#L115) via CRI. It does this by populating a `ContainerConfig` struct (in which the command, image, labels, mounts, devices, environment variables etc. are defined) from the parent PodSpec and then sending that via protobufs to the CRI plugin. For Docker, it deserializes the payload and populates its own config structures to send to the Daemon API. In the process it applies a few metadata labels (such container type, log path, sandbox ID) to the container.
1. It then registers the container with CPU manager, which is a new alpha feature in 1.8 that assigns containers to sets of CPUs on the local node by using the `UpdateContainerResources` CRI method.
1. The container is then [started](https://github.com/kubernetes/kubernetes/blob/5f9f4a1c5939436fa320e9bc5973a55d6446e59f/pkg/kubelet/kuberuntime/kuberuntime_container.go#L135).
1. If any post-start container lifecycle hooks are registered, they are [run](https://github.com/kubernetes/kubernetes/blob/5f9f4a1c5939436fa320e9bc5973a55d6446e59f/pkg/kubelet/kuberuntime/kuberuntime_container.go#L156-L170). Hooks can either be of the type `Exec` (executes a specific command inside the container) or `HTTP` (performs a HTTP request against a container endpoint). If the PostStart hook takes too long to run, hangs, or fails, the container will never reach a `running` state.

## Wrap-up

Okay, phew. Done. Finito.

After all this, we should have 3 containers running on one or more worker nodes. All of the networking, volumes and secrets have been populated by the kubelet and made into containers via the CRI plugin. 
