# gRPC Hello World example in Kubernetes
The [gRPC C++ Quick start](https://grpc.io/docs/languages/cpp/quickstart/) walks through how to compile and run a simple gRPC service (client and server) utilizing a very simple Procol Buffer implementation. In order to test the client and server, you launch the server in one window, then launch the client in another window. This project deploys the same client and server in containers using Kubernetes.

## Prerequsities
This guide assumes that you have a working `docker` installation that can connect to a docker daemon, a working Kubernetes cluster, and that `kubectl` can connect to the cluster. There are hundreds of guides online that are better than what I would write.

## Building gRPC and protoc
In order to build the gRPC client and server, you need `protoc`, the protocol buffer compiler that will generate the stub functions from the protocol buffer specification. You also need the gRPC libraries to link against. These are easily compiled using the instructions in the linked quick start. Because they take some time to compile, I've already built a container image that contains them and pushed it to Docker Hub [here](https://hub.docker.com/repository/docker/crcady/grpc). The Dockerfile for that image just follows the instructions in the quick start to build and install gGRPC with a prefix of `/tmp/grpc_install`.

## Building the example
This example will work out of the box because it points to a public image on Docker Hub. If you want to build the container image yourself, follow the instructions in this section.

To minimize the size of the `.dockerignore` file, the Dockerfile for this example is in the Dockerfile directory. To build and tag the image:

```bash
git clone https://github.com/crcady/grpc-k8s-example
cd grpc-k8s-example/docker
docker build -t <reponame>/grpc-example .
```

This will build the exmaple container. To use your image intstead of my image you need to:
- Push your image to a repository (Docker Hub, GitHub Container Registry, etc)
- Edit `manifests/deployment.yaml` to point to your repository instead of mine

## Running the example
The following steps will create a service called `greeter` that exposes the `grpc-example` container internally within the cluster. Then a new pod will be created in the cluster that runs the `greeter_client` application and then stops. The commands given here assume you've cloned this repo and are in the base directory.

### Create the greeter service
You can create the service or the deployment in any order, but creating the service first means that the service environment variables will be present when you deploy the application, which is sometimes useful. Creating the service with `kubectl` is simple:

```bash
kubectl apply -f manifests/service.yaml
# service/greeter created
```

To verify that the service was created:
```bash
kubectl get service greeter
```

### Deploy the grpc-example pods
That created the service, but there are no running pods for the service to direct traffic towards. Again, use `kubectl`:

```bash
kubectl apply -f manifests/deployment.yaml
# deployment.apps/grpc-deployment created
```

After a few seconds, the pods should be available. To check on the status of them:
```bash
kubectl get pods
```

### Run the client application the imperative way
The simplest way to run the client application inside the cluster is using `kubectl run`:

```bash
kubectl run grpc-client --image crcady/grpc_example --command -i -t --rm --restart=Never -- /bin/sh -c './greeter_client --target=$$GREETER_SERVICE_HOST:$$GREETER_SERVICE_PORT'
# Greeter received: Hello World
# pod "grpc-client" deleted
```

Because of the `-i -t` options, you'll immediately see the output of the client. The `--rm` flag removes the pod as soon as it finishes, and the `--restart=Never` means that when the application exits, Kubernetes won't keep restarting it until a timeout occurs. Note how you don't need to provide the IP address of the greeter service, Kubernetes provides that via an environment variable automatically.

### Run the client application declaratively
One of the basic tenets of working with Kubernetes is that declarive is better than imperative. You can create a pod that runs the client and then exits as follows:

```bash
kubectl apply -f manifests/client-pod.yaml
# pod/grpc-client created
```

It won't be immediately obvious that this was successful. To see that the pod ran and stopped, get the pods:
```bash
kubectl get pods
```

To see the output of the client application:
```bash
kubectl logs grpc-client
# Greeter received: Hello world
```

Now, if you try to apply the yaml file again, you'll get an error because you're applying a resource (the pod) that already exists. You can remove the pod with

```bash
kubectl delete pod grpc-client
# pod "grpc-client" deleted
```

### Clean up
You still have the greeter service and the pod that runs the greeter server running on your cluster. To remove them:

```bash
kubectl delete service greeter
# service "greeter" deleted
kubectl delete deployment grpc-deployment
# deployment.app "grpc-deployment" deleted
```

## That's it
That's all it takes to deploy the C++ gRPC example application as a Kubernetes service, and then access from another pod within the cluster!