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

## admission controllers

## object topology created (pods -> RS -> deployment)

kubernetes runs a generic web server and mutates its state using the informer pattern. in other words, a server is decorated with a set of api resources (REST mapping) defined by  
????

## each object save to etcd

## scheduler assigns node

## kubelet deploys container

## create service

## steps authn->etcd repeated

## endpoint created

## kube-proxy writes iptables rules

## dns server adds A/SRV records