## GitOps with Kubernetes and ArgoCD workshop

Follow the instructions below to setup your environment for the workshop.

The instructions assume you are using Linux but they should be easy to translate to MacOS or Windows.

### Docker

This is a pre-requisite for the workshop and you should have it installed on your system already.
The commands below assume you can run docker without `sudo`, check [this link](https://docs.docker.com/engine/install/linux-postinstall/#manage-docker-as-a-non-root-user) if necessary. You can also add `sudo` to the commands when necessary.

### kubectl

`kubectl` is the CLI for Kubernetes. You can use it to get, create, modify or delete resources in the cluster.

```sh
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
mv kubectl ${HOME}/bin/kubectl
chmod +x ${HOME}/bin/kubectl
kubectl version
```

### k3d

Install a release for your OS from https://github.com/rancher/k3d/releases.
Download the binary, copy it to a place in your PATH and give it execution permission.

```sh
curl -LO "https://github.com/k3d-io/k3d/releases/download/v5.4.1/k3d-linux-amd64"
mv k3d-linux-amd64 ${HOME}/bin/k3d
chmod +x ${HOME}/bin/k3d
```

Now create your first cluster:

```sh
k3d cluster create gitops
kubectl get nodes
```

### ArgoCD

ArgoCD is installed as a set of Kubernetes Custom Resource Definitions (CRDs), an API server,
a Repository Server, an Application Controller and a Web UI.

```sh
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

Wait for all pods to be ready:
```sh
kubectl -n argocd get pods
```

Now create a port-forward to map a local port to the service in the cluster:
```sh
kubectl -n argocd get service
kubectl port-forward svc/argocd-server -n argocd 8443:443
```

The ArgoCD installation automatically creates an `admin` user and password. Get the password with:

```sh
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d; echo
```

Open https://localhost:8443/ in your browser and login.

### ArgoCD CLI

ArgoCD has a CLI which you can download from the help page: https://localhost:8443/help

You can login with:
```sh
argocd login localhost:8443 --insecure --username admin --password $(kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d)
```

### Guestbook

We will use the Go version of the [Guestbook sample application](https://github.com/kubernetes/examples/tree/master/guestbook-go) 
to create a CI pipeline which publishes a docker image to GitHub Packages and triggers a deployment 
to a PR environment using Git.

This repository contains the application source code and the [guestbook-go-config](../../../guestbook-go-config) repository contains 
the Kubernetes and ArgoCD manifests to deploy the application. Using a separate repository for configuration is one of the best practices of GitOps.

Now go ahead and fork both repositories into your GitHub account. After forking, clone them to your local environment.

### Change github URLs

There are some files in the `guestbook-go-config` repo with URLs for the `isacssouza` GitHub user. You will need to change those to use your username.
You can use the command below on the root of the `guestbook-go-config` repo. Don't forget to change `<github-user>` to the actual username.

```sh
find . -name '*.yaml' | xargs sed -i'' 's/isacssouza/<github-user>/g'
```

Now review, commit and push the changes.

### Setup Personal Access Token (PAT)

In order to checkout and push to the config repo, we will need a GitHub PAT configured as a secret, which will be used by GitHub Actions.

First, create the PAT using [these instructions](https://docs.github.com/en/github/authenticating-to-github/keeping-your-account-and-data-secure/creating-a-personal-access-token). The PAT only needs the `repo` permissions.

Copy the token and create a new secret in the [Settings page](https://github.com/isacssouza/guestbook-go/settings/secrets/actions) of the `guestbook-go` repo. Name it `CONFIG_GITHUB_TOKEN`.

### Continuous Integration

For this application we have a very simple CI pipeline that uses GitHub Actions to build a docker image and publishes it to GitHub Packages. Take a look at the workflow definition in the [.github/workflows](.github/workflows) directory.

To setup our first guestbook image we will manually trigger a workflow run at [the Actions tab](../../actions/workflows/ci.yaml)

### ArgoCD Applications

From https://argo-cd.readthedocs.io/en/stable/operator-manual/declarative-setup/#applications:

The Application CRD is the Kubernetes resource object representing a deployed application instance in an environment. It is defined by two key pieces of information:

- `source` reference to the desired state in Git (repository, revision, path, environment)
- `destination` reference to the target cluster and namespace. For the cluster one of server or name can be used, but not both (which will result in an error). Under the hood when the server is missing, it is calculated based on the name and used for any operations.

A minimal Application spec is as follows:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: guestbook
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/argoproj/argocd-example-apps.git
    targetRevision: HEAD
    path: guestbook
  destination:
    server: https://kubernetes.default.svc
    namespace: guestbook
```

### Let's create our first Application

Head over to https://localhost:8443/ and create your first application using the Web UI.

After the application is deployed, setup a port-forward for it:

```sh
kubectl -n guestbook-pr port-forward service/guestbook 8080:3000
```

And access the guestbook at http://localhost:8080/

### App of Apps

With ArgoCD you can create an app that creates other apps, which in turn can create other apps. This allows you to declaratively manage a group of apps that can be deployed and configured in concert.

Let's create an app of apps to manage our guestbook environments. Go to the `guestbook-go-config` repository and execute:
```sh
kubectl apply -f guestbook-apps.yaml
```

Now check the ArgoCD dashboard to see the new apps.
