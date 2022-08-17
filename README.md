<p align="center">
  <a href="https://www.fathom5.co/" target="_blank"><img src="https://static.wixstatic.com/media/3d35e8_264823a2b86b409e9615664cd09d57da~mv2.png/v1/fill/w_218,h_45,al_c,q_85,usm_0.66_1.00_0.01/3d35e8_264823a2b86b409e9615664cd09d57da~mv2.webp"/></a>
</p>

<br>

<h1 align="center">Developing CRDs for Kubernetes</h1>

<p align="center">
  <img src="https://img.shields.io/badge/shell-bash-yellow"/>
  <img src="https://travis-ci.org/mingrammer/pyreportcard.svg?branch=master"/>
</p>

<p align="center">
  This talk is dedicated to learning how to develop Custom Resource Definitions (CRDs) in Kubernetes using <a href="https://github.com/kubernetes-sigs/kubebuilder">Kubebuilder</a>.
</p>

## Getting Started 🎬

Ensure that you have the following installed on your computer:

- [Golang](https://go.dev/doc/install)
- [Docker](https://docs.docker.com/engine/install/)
- Kubectl ([Linux](https://kubernetes.io/docs/tasks/tools/install-kubectl-linux), [MacOS](https://kubernetes.io/docs/tasks/tools/install-kubectl-macos), [Windows](https://kubernetes.io/docs/tasks/tools/install-kubectl-windows))
- Single node K8s Dev Tool ([K3D](https://github.com/k3d-io/k3d/releases/tag/v5.4.4), [Kind](https://github.com/kubernetes-sigs/kind), [Minikube](https://github.com/kubernetes/minikube), etc...). I'll be using K3D for this demo.
  
## Install Kubebuilder 💻

**On MacOS:**

```
version=1.0.8 # latest stable version
arch=amd64
# download the release
curl -L -O "https://github.com/kubernetes-sigs/kubebuilder/releases/download/v${version}/kubebuilder_${version}_darwin_${arch}.tar.gz"
# extract the archive
tar -zxvf kubebuilder_${version}_darwin_${arch}.tar.gz
mv kubebuilder_${version}_darwin_${arch} kubebuilder && sudo mv kubebuilder /usr/local/
# update your PATH to include /usr/local/kubebuilder/bin
export PATH=$PATH:/usr/local/kubebuilder/bin
```

**On Linux:**

```
version=1.0.8 # latest stable version
arch=amd64
# download the release
curl -L -O "https://github.com/kubernetes-sigs/kubebuilder/releases/download/v${version}/kubebuilder_${version}_linux_${arch}.tar.gz"
# extract the archive
tar -zxvf kubebuilder_${version}_linux_${arch}.tar.gz
mv kubebuilder_${version}_linux_${arch} kubebuilder && sudo mv kubebuilder /usr/local/
# update your PATH to include /usr/local/kubebuilder/bin
export PATH=$PATH:/usr/local/kubebuilder/bin
```

**On Windows:**

<div align="center">WSL??? ¯\_(ツ)_/¯</div>

## Demo Time 😎

<!-- ### Table of Contents

<p>
  <ul>
    <li><a href="https://github.com/FATHOM5/grace2-drp-deploy#getting-started">Getting Started</a></li>
    <li><a href="https://github.com/FATHOM5/grace2-drp-deploy#adding-workflow-to-drp-ecosystem">Adding Workflow to DRP Ecosystem</a></li>
    <li><a href="https://github.com/FATHOM5/grace2-drp-deploy#repository-directory-structure">Repository Directory Structure</a></li>
  </ul>
</p> -->

### As a Developer, What Do You Want?

You've just created a Kubernetes cluster loaded with microservices. It appears to be operating as intented, but you'd like a way to run better resiliency testing. You think to yourself, "What would happen if I killed pods at runtime? Would the overall reliability of the system go down?"
This seems like a good test. How do we implement it?

### The Goblin

Imagine that we deployed a Goblin to our cluster. The role of the Goblin is simply to kill pods in the namespace that it's deployed to. The Goblin has some properties:
- Name
- Age
- Color

The Goblin will be our Custom Resource Definition which will destroy pods during runtime. We'll now use Kubebuilder to generate the boilerplate for our Goblin (don't forget to `go mod init` first):

    # Run go mod init if you are outside of your $GOPATH
    go mod init github.com/FATHOM5/goblin

    # Initialize Kubebuilder project
    kubebuilder init --domain fathom5.co --owner "Fathom5"

    # Create the API that defines the Goblin resource
    kubebuilder create api --group chaos --version v1 --kind Goblin

    # Create a Kubebuilder Webhook
    kubebuilder create webhook --group chaos --version v1 --kind Goblin --defaulting --programmatic-validation

### Temporarily disabling SSL

In `main.go`, you should see a block of code like this:

```
if err = (&chaosv1.Goblin{}).SetupWebhookWithManager(mgr); err != nil {
  setupLog.Error(err, "unable to create webhook", "webhook", "Goblin")
  os.Exit(1)
}
```

Wrap the block of text inside an `if` statement like this:

```
if os.Getenv("ENABLE_WEBHOOKS") != "false" {
  if err = (&chaosv1.Goblin{}).SetupWebhookWithManager(mgr); err != nil {
    setupLog.Error(err, "unable to create webhook", "webhook", "Goblin")
    os.Exit(1)
  }
}
```

### Defining the API
Navigate to `./api/v1/goblin_types.go`. There is a struct called `GoblinSpec`. This is where we include the fields from above. The resulting `GoblinSpec` struct will look like this:

```
// GoblinSpec defines the desired state of Goblin
type GoblinSpec struct {
	// INSERT ADDITIONAL SPEC FIELDS - desired state of cluster
	// Important: Run "make" to regenerate code after modifying this file

	// Name is the name of the Goblin
	Name string `json:"name"`

	// The age of the Goblin, in years
	Age int32 `json:"age"`

	// Color of the Goblin
	Color string `json:"color"`
}
```

`goblin_types.go` also has a struct called `GoblinStatus`, which defines fields that we'll access in order to update the state of the Goblin:

```
// GoblinStatus defines the observed state of Goblin
type GoblinStatus struct {
	// INSERT ADDITIONAL STATUS FIELD - define observed state of cluster
	// Important: Run "make" to regenerate code after modifying this file

	// The mood of the goblin
	Mood string `json:"mood"`
}
```

Now that we have an API, run `make manifests` to take the API definition and turn it into something that Kubernetes can understand.

### Installing CRD to the Cluster

First, start a cluster:

    k3d cluster create

Now run `kubectl get goblins`. You should get output like the following:

```
error: the server doesn't have a resource type "goblins"
```

Let's fix that. When you ran `make manifests`, the CRD for the Goblin type was created under `./config/crd/bases`. We can apply the CRD by running `make install`. You should get output like the following:

```
GOBIN=/home/brent/code/erm/goblin/bin go install sigs.k8s.io/controller-tools/cmd/controller-gen@v0.9.0
/home/brent/code/erm/goblin/bin/controller-gen rbac:roleName=manager-role crd webhook paths="./..." output:crd:artifacts:config=config/crd/bases
curl -s "https://raw.githubusercontent.com/kubernetes-sigs/kustomize/master/hack/install_kustomize.sh" | bash -s -- 3.8.7 /home/brent/code/erm/goblin/bin
{Version:kustomize/v3.8.7 GitCommit:ad092cc7a91c07fdf63a2e4b7f13fa588a39af4f BuildDate:2020-11-11T23:14:14Z GoOs:linux GoArch:amd64}
kustomize installed to /home/brent/code/erm/goblin/bin/kustomize
/home/brent/code/erm/goblin/bin/kustomize build config/crd | kubectl apply -f -
customresourcedefinition.apiextensions.k8s.io/goblins.chaos.fathom5.co created
```

Now when you run `kubectl get goblins`, you'll get the following output:

```
No resources found in default namespace.
```

This means that Kubernetes recognizes the `Goblin` type. We can further verify this by running `kubectl explain goblin.spec`:

```
KIND:     Goblin
VERSION:  chaos.fathom5.co/v1

RESOURCE: spec <Object>

DESCRIPTION:
     GoblinSpec defines the desired state of Goblin

FIELDS:
   age  <integer> -required-
     The age of the Goblin, in years

   color        <string> -required-
     Color of the Goblin

   name <string> -required-
     Name is the name of the Goblin
```

This gives us more information about what it means to deploy a Goblin to the cluster. Let's define one now. Create a file called `goblin.yaml` and copy/paste this block of text into it:

```
apiVersion: chaos.fathom5.co/v1
kind: Goblin
metadata:
  name: goblin-1
spec:
  age: 25
  color: "Purple"
  name: "Steve"
```

Now run `kubectl apply -f goblin.yaml`, you should get output similar to the following:

```
goblin.chaos.fathom5.co/goblin-1 created
```

Now if you run `kubectl get goblins`, you should see the goblin you just wrote:

```
NAME       AGE
goblin-1   52s
```

Currently we only return back the name of the goblin definition and the age of the resource. This is not particularly useful...we'll change this later!

Go ahead and delete the Goblin with `kubectl delete -f goblin.yaml`.

### The Reconciler

The meat of your controller implementation will live inside the `Reconcile` function, located in `./controllers/goblin_controller.go`. The controller will listen against the Kubernetes event stream and then handle those events accordingly. ie - What does it actually MEAN to be a Goblin. That gets defined in this function. Let's put together a workflow:

1. Deploy a Goblin
2. Listen against the Kubernetes event stream
3. Query the K8s API for the Goblin we just created
4. If the Goblin has just been created, it's searching for a Pod to delete. Therefore it's `Lurking`. 
5. Grab a list of running pods
6. If there are no running pods, then the Goblin gets very `Unhappy` since there is nothing to delete
7. If the Goblin finds a Pod, attempt to delete it.
8. If the Goblin successfully deletes the Pod, he becomes a `Happy` Goblin. 

Before we start implementing the function, be sure to add these two comments underneath the clump of 3 rbac rules around line 36/37 of `./controllers/goblin_controller.go`:

```
//+kubebuilder:rbac:groups=chaos.fathom5.co,resources=goblins,verbs=get;list;watch;create;update;patch;delete
//+kubebuilder:rbac:groups=chaos.fathom5.co,resources=goblins/status,verbs=get
```

Run `make manifests`. Now, let's implement the Reconciler together!

Before we run the controller, let's prettify that `kubectl get goblins` command. Add the following comment block to `goblin_types.go` under where it says `//+kubebuilder:subresource:status`:

```
//+kubebuilder:printcolumn:name="Mood",type="string",JSONPath=".status.mood",description="Goblin's current mood"
//+kubebuilder:printcolumn:name="Goblin Name",type="string",JSONPath=".spec.name",description="Name of the Goblin"
//+kubebuilder:printcolumn:name="Age",type="string",JSONPath=".spec.age",description="Age of the Goblin"
//+kubebuilder:printcolumn:name="Color",type="string",JSONPath=".spec.color",description="Color of the Goblin"
```

## Running the Controller

    make run ENABLE_WEBHOOKS=false

You should get output similar to the following:

```
/home/brent/code/erm/goblin-template/bin/controller-gen rbac:roleName=manager-role crd webhook paths="./..." output:crd:artifacts:config=config/crd/bases
/home/brent/code/erm/goblin-template/bin/controller-gen object:headerFile="hack/boilerplate.go.txt" paths="./..."
go fmt ./...
go vet ./...
go run ./main.go
1.6607689361128545e+09  INFO    controller-runtime.metrics      Metrics server is starting to listen    {"addr": ":8080"}
1.6607689361131492e+09  INFO    setup   starting manager
1.6607689361132872e+09  INFO    Starting server {"path": "/metrics", "kind": "metrics", "addr": "[::]:8080"}
1.6607689361133199e+09  INFO    Starting server {"kind": "health probe", "addr": "[::]:8081"}
1.660768936113424e+09   INFO    Starting EventSource    {"controller": "goblin", "controllerGroup": "chaos.fathom5.co", "controllerKind": "Goblin", "source": "kind source: *v1.Goblin"}
1.660768936113556e+09   INFO    Starting Controller     {"controller": "goblin", "controllerGroup": "chaos.fathom5.co", "controllerKind": "Goblin"}
1.6607689362144554e+09  INFO    Starting workers        {"controller": "goblin", "controllerGroup": "chaos.fathom5.co", "controllerKind": "Goblin", "worker count": 1}
```

Deploy some kind of workload. In this case, I'll simply just deploy 3 ubuntu pods all running a `sleep` function:

```
kubectl run ubuntu-1 --image ubuntu:20.04 -- sleep 6000 
kubectl run ubuntu-2 --image ubuntu:20.04 -- sleep 6000 
kubectl run ubuntu-3 --image ubuntu:20.04 -- sleep 6000 
```

Deploy the Goblin and watch as it deletes one of the Ubuntu pods:

```
kubectl apply -f goblin.yaml
```