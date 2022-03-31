# Argo CD kbld Plugin Example

This repository contains an exampe of using [Argo CD](https://argo-cd.readthedocs.io/en/stable/) with [kbld](https://carvel.dev/kbld/). 
Argo CD's plugin features and kbld's configuration for resolving container images to a digest format can help assure that images deployed 
to a Kubernetes cluster are exactly as expected. 

kbld uses an [ImagesLock file](https://carvel.dev/kbld/docs/v0.32.0/resolving/#generating-resolution-imgpkg-lock-output) to map container 
image tags to a digest format. In this example, kbld will make sure the images deployed by Argo CD match what is defined in the kbld ImagesLock 
file to provide greater assurances around what is being deployed to a cluster.

### Prerequisites

1. A Kubernetes cluster 
2. Carvel tool suite (i.e. kbld, kapp, imgpkg, ytt): https://carvel.dev/#install
3. Argo CD CLI: https://argo-cd.readthedocs.io/en/stable/cli_installation/

### kbld Intro

To start, let's show how kbld works. In the `kbld-example-app-plugin` directory, you will see a file called [sample-app.yml](kbld-example-app-plugin/sample-app.yml). 
The manifest contains a Deployment and a Service, and the Deployment uses the following container image: `danielhelfand/go-web-server:latest`. 
This image has a tag (`latest`), but tags are not immutable and can change over time. The tag has a corresponding digest `1fda93b8cd881440ad41ea94413aef25e850e60403003fdef30cec384bb6c32f`. 

This digest is immutable and will always pull back this exact version of the container image unlike the latest tag. kbld is able to translate this 
latest tag into a digest by running the following command:

```
kbld -f kbld-example-app-plugin/sample-app.yml
```

In the output, you can see the latest tag for the deployment image is changed to the following:

```
- image: index.docker.io/danielhelfand/go-web-server@sha256:1fda93b8cd881440ad41ea94413aef25e850e60403003fdef30cec384bb6c32f
```

So instead of referencing an image by its tag, kbld makes it easy to translate this image reference into an immutable reference that can't be updated.

kbld also helps users resolve image tags to digests in a declarative fashion by using an ImagesLock file. An example of this is shown [here](kbld-example-app-plugin/images.yml).

The ImagesLock in this repository references a different digest from a previous push of the latest tag. Using it with kbld, you will see how you can use this 
config file to lock in which versions of a container are deployed:

```
kbld -f kbld-example-app-plugin/sample-app.yml -f kbld-example-app-plugin/images.yml
```

This time the output will show the following digest for the deployment's image:

```
- image: index.docker.io/danielhelfand/go-web-server@sha256:6c0b0765617ea48ab434f58dd7b3481f50539bbc31de2d8a38c67754ef232a74
```

### Argo CD kbld Plugin

In order to add kbld as a plugin to Argo CD, we will make some minor edits to the Argo CD manifests using `ytt`. In the [plugin-overlay.yml](plugin-overlay.yml) 
file, you will see that a sidecar container is defined. This will be added to the Argo CD Deployment. This sidecar uses an image with kbld installed on it and 
also has a ConfigMap mounted to it named `cmp-plugin`. 

This ConfigMap contains information used by Argo CD on when to use the kbld plugin and how to run kbld before applying manifests to a Kubernetes cluster. Using 
the `spec.discover.fileName` property, this lets Argo CD know to run the kbld plugin on any application that has an `images.yml` file present. Assuming this 
file is present, the following kbld command will be run:

```
kbld -f images.yml -f .
```

This will allow us to resolve container image tags to the digest format before Argo CD deploys resources to a Kubernetes cluster.

To deploy Argo CD with this plugin, run the following commands against a Kubernetes cluster:

```
kubectl create ns argocd
kapp deploy --app argocd --namespace argocd -f <(ytt -f plugin-overlay.yml -f https://raw.githubusercontent.com/argoproj/argo-cd/v2.3.2/manifests/core-install.yaml)
```

Once the command completes, Argo CD should be set up to use kbld as part of its deployment process.

### Deploy the kbld Sample App

Using the Argo CD CLI, let's deploy the sample app using Argo CD and the kbld plugin:

```
kubectl config set-context --current --namespace=argocd
argocd login --core
argocd app create go-web-server --repo https://github.com/danielhelfand/argocd-kbld-plugin-example --path kbld-example-app-plugin --dest-name in-cluster --dest-namespace default
```

With the Application created, we are all set to use kbld with Argo CD and deploy the sample application. Let's first take a look at what the sample app looks like when we 
don't use kbld:

```
kapp deploy -a go-web-server -f https://raw.githubusercontent.com/danielhelfand/argocd-kbld-plugin-example/main/kbld-example-app-plugin/sample-app.yml
```

Once the command completes, let's view the application at localhost:8080/you after running the command below:

```
kubectl port-forward svc/go-web-server-service 8080:8080
```

The latest tag of the app displays a not nice message. Let's see what the earlier version of the app displayed, which is made possible by using kbld and its ImagesLock file.
Stop the port-forward and run the following:

```
argocd app sync go-web-server
```

Run the following to view the sample app:

```
kubectl port-forward svc/go-web-server-service 8080:8080 -n default
```

Visit the link at localhost:8080/you. Hopefully this time the message is a little nicer. 

Let's check what image reference the deployment uses:

```
kubectl get deployment/go-web-server-deployment -n default -o yaml
```

You should see the image reference is `index.docker.io/danielhelfand/go-web-server@sha256:6c0b0765617ea48ab434f58dd7b3481f50539bbc31de2d8a38c67754ef232a74`, which contains 
the same digest found in the ImagesLock file of the repository.

### Clean Up

To clean up this example, run the following commands:

```
kapp delete -a argocd
kapp delete -a go-web-server
```