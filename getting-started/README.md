<walkthrough-author name="David Gageot" repositoryUrl="https://github.com/GoogleCloudPlatform/knative-build-tutorials" email="dgageot@google.com" tutorialName="knative-build-getting-started"></walkthrough-author>

# Knative Build - Getting started

Welcome! This tutorial will help you get started with [Knative Build](https://github.com/knative/build) extension to Kubernetes.

## What is Knative Build?

It's an [open-source project](https://github.com/knative/build) that provides an implementation
of the Build [CRD](https://kubernetes.io/docs/concepts/extend-kubernetes/api-extension/custom-resources/)
that runs Builds on-cluster.

**What is it NOT?**

It's not a complete standalone product that could be used for CI/CD.
Think of it as a building block to facilitate the expression of Builds as part of larger systems.

**What am I going to learn?**

 1. You'll learn how to install Knative Build and start using it with simple builds.
 2. You'll also learn a bit how it works underneath.

**Time to complete:** <walkthrough-tutorial-duration duration="10"></walkthrough-tutorial-duration>

**Are you ready?** Click the `Continue` button to get started...

## Setup Kubernetes

*Warning: Those instructions assume you have a Kubernetes cluster and `kubectl` pointing at it,
as well as a basic knowledge of Kubernetes and its CLI, `kubectl`.*

One way to check if `kubectl` is configured properly is to run:

```bash
kubectl get nodes
```

It should show a list of nodes.

If you don't have a Kubernetes cluster, follow the instructions below:

<walkthrough-tutorial-card
  url="kubernetes-engine/docs/how-to/creating-a-cluster"
  icon="KUBERNETES_SECTION"
  label="gke">
**Creating a Cluster**
This page explains how to create a cluster on Kubernetes Engine.
</walkthrough-tutorial-card>

**Click the `Continue` button to prepare the cluster...**

## Configure permissions

Before Knative Build is installed, we need to grant `cluster-admin` permissions to the current user:

```bash
kubectl create clusterrolebinding cluster-admin-binding \
  --clusterrole=cluster-admin \
  --user=$(gcloud config get-value core/account)
```

**Install Knative Build**

Next, we install the latest version of [Knative Build](https://github.com/knative/build):

```bash
kubectl apply -f https://storage.googleapis.com/build-crd/latest/release.yaml
```

From now one, you should be good to go!

But let's check that everything was installed correctly. That'll also help you understand which
new components have been installed on the cluster.

**Click the `Continue` button to run those checks...**

## Verify the installation

The installation process created a [Kubernetes namespace](https://kubernetes.io/docs/concepts/overview/working-with-objects/namespaces/)
in which the Knative components run.

```bash
kubectl get namespace knative-build
```

Let's list the components that are running in that namespace and
wait until each component is running:

```bash
kubectl -n knative-build get pods -w
```

CTRL+C when `STATUS` column shows `Running` for both pods.

You can see that the Build [CRDs](https://kubernetes.io/docs/concepts/extend-kubernetes/api-extension/custom-resources/) are installed:

```bash
kubectl get crd
```

Thanks to those CRDs, now Kubernetes recognizes a new type of resource, the `Build` type:

```bash
kubectl get build
```

It should display `No resources found.` for now.

**You are ready to run your first Build!**

**Click the `Continue` button to run a simple build...**

## Hello, World!

Let's start simple with a Build that has no input and produces no artifact.
All it does is print "`Hello, World!`". It's enough to learn the basics!

<walkthrough-spotlight-pointer spotlightId="devshell-web-editor-button">Open the file editor</walkthrough-spotlight-pointer>
and take a look at the Kubernetes <walkthrough-editor-open-file filePath="knative-build-tutorials/getting-started/build.yaml">yaml manifest</walkthrough-editor-open-file>
to express such a build:

```yaml
apiVersion: build.knative.dev/v1alpha1
kind: Build
metadata:
  name: hello
spec:
  steps:
  - image: busybox
    args: ['echo', 'Hello, World!']
```

See, that's easy!

**The `Build` kind**

You can see that Kubernetes was extended with a new kind of resource: the `Build`.
A Build is made of steps that run in sequence and share a common directory called the `workspace`.

Each step is a Docker image that runs with the given arguments.

Our sample has only one step which uses the `busybox` image to run `echo Hello, World!`.

**Click the `Continue` button to start the build...**

## Let's start the build

```bash
kubectl apply -f getting-started/build.yaml
```

The build is running:

```bash
kubectl get builds
```

**Congratulations, you've started your first build with Knative Build!**

<walkthrough-conclusion-trophy></walkthrough-conclusion-trophy>

Running `kubectl apply` triggered quite a few actions on the cluster side.

**Continue to next steps, to understand what happened underneath...**

## Inspect the Build

We can start by inspecting the Build object that was created:

```bash
kubectl get build hello -o yaml
```

It'll give this kind of result:

```yaml
apiVersion: build.knative.dev/v1alpha1
kind: Build
metadata:
  creationTimestamp: 2018-07-06T13:12:43Z
  name: hello
  namespace: default
spec:
  steps:
  - args:
    - echo
    - Hello, World!
    image: busybox
status:
  builder: Cluster
  cluster:
    namespace: default
    podName: hello-klsqq
  conditions:
  - reason: Building
    state: Succeeded
  startTime: 2018-07-06T13:12:43Z
```

**The interesting information are:**

 + The state of the build: **`Succeeded`**
 + The name of the pod that ran the job: **`hello-klsqq`**

The build is described in a `Build` object but its steps actually ran in a standard
[Kubernetes pod](https://kubernetes.io/docs/concepts/workloads/pods/pod/).

**Continue to next step to inspect the pod...**

## Let's inspect the pod to learn more about the build

```bash
kubectl get pod $(kubectl get build hello -ojsonpath={.status.cluster.podName}) -oyaml
```

It'll give you the full description of the pod.

**That's a lot of information!**

The main information is the list of containers (think steps) that are run:

```yaml
initContainers:
- env:
  - name: HOME
    value: /builder/home
  image: gcr.io/build-crd/github.com/knative/build/cmd/creds-init@sha256:a40420845b318318f6649674fc8cc1b23c7528b8be7d1a92a97508c69dac133c
  name: build-step-credential-initializer
  volumeMounts:
  - mountPath: /workspace
    name: workspace
  - mountPath: /builder/home
    name: home
  workingDir: /workspace
- args:
  - echo
  - Hello, World!
  env:
  - name: HOME
    value: /builder/home
  image: busybox
  name: build-step-unnamed-1
  volumeMounts:
  - mountPath: /workspace
    name: workspace
  - mountPath: /builder/home
    name: home
  workingDir: /workspace
```

*Pro Tip: look for the **image** keyword to find the containers.*

**Build steps run in containers:**

We can see that the pod ran two [Init Containers](https://kubernetes.io/docs/concepts/workloads/pods/init-containers/) in sequential order:

 + One to initialize the credentials. More on that later.
 + One to actually run the only step of our build.

**Build steps share a common workspace:**

Containers all share the same two volumes:

 + `/builder/home` - the home folder for all containers where shared configuration is typically installed.
 + `/workspace` - a shared folder where to read inputs from and where to write outputs of each step.

**Continue to next step to get the Build's status...**

## The pod also holds the statuses for each build step:

```yaml
initContainerStatuses:
- containerID: docker://94a639a6b16c69648989ff5c0d8eb94a54b0f6ac50f8484eed803d1f91b1123a
  image: gcr.io/build-crd/github.com/knative/build/cmd/creds-init@sha256:a40420845b318318f6649674fc8cc1b23c7528b8be7d1a92a97508c69dac133c
  name: build-step-credential-initializer
  ready: true
  state:
    terminated:
      containerID: docker://94a639a6b16c69648989ff5c0d8eb94a54b0f6ac50f8484eed803d1f91b1123a
      exitCode: 0
      finishedAt: 2018-07-06T11:51:07Z
      reason: Completed
      startedAt: 2018-07-06T11:51:07Z
- containerID: docker://db1de3e7032da065c7e1a9dcedf6e10372389130d0fcfb1546b1cd44749bed48
  image: busybox:latest
  imageID: docker-pullable://busybox@sha256:74f634b1bc1bd74535d5209589734efbd44a25f4e2dc96d78784576a3eb5b335
  name: build-step-unnamed-1
  ready: true
  state:
    terminated:
      containerID: docker://db1de3e7032da065c7e1a9dcedf6e10372389130d0fcfb1546b1cd44749bed48
      exitCode: 0
      finishedAt: 2018-07-06T11:51:09Z
      reason: Completed
      startedAt: 2018-07-06T11:51:09Z
phase: Succeeded
```

That's useful to understand if the build went well or not.

**Let's see the logs**

More than the status of each step, what you want, as a user, is the logs output.
It's possible to read the logs for each step with the usual `kubectl logs` command.

For example:

```bash
kubectl logs -f $(kubectl get build hello -ojsonpath={.status.cluster.podName}) -c build-step-unnamed-1
```

It should show `Hello, World!`.

**It takes a lot of time!**

*Only the first build will be long to start because Knative needs to pull some Docker images that
are used by each build. Once those images are pulled, the builds will be much faster.*

**I see no logs!**

*Sometimes, Kubernetes deletes the logs for a Build that's terminated. It makes sense in the Kubernetes world.
Not so much in the user world!. Remember that Knative Build is a building block meant to be used in larger systems.
In this kind of system, logs would typically be aggregated and published somewhere else.*

**Continue to next step to improve the logs...**

## Let's improve the Hello, World!

First, we are going to install a CLI utility to read a build's logs. This will be much easier than running
lots of `kubectl` commands!

```bash
go get -u github.com/knative/build/cmd/logs
```

```bash
logs hello
```

That's so much easier!

**Continue to next step to improve the build description...**

## Build steps can have names

When build steps are not given names, they get an auto-generated name such as `build-step-X`.
Naming build steps will give us a more meaningful output in logs.

Let's give a name to the build's unique step. Why not name it `hello-world`?

Open the <walkthrough-editor-open-file filePath="knative-build-tutorials/getting-started/build.yaml">yaml manifest</walkthrough-editor-open-file> and use this content:
```yaml
apiVersion: build.knative.dev/v1alpha1
kind: Build
metadata:
  name: hello
spec:
  steps:
  - name: hello-world
    image: busybox
    args: ['echo', 'Hello, World!']
```

We have to delete the previous build...

```bash
kubectl delete build hello
```

...and start a fresh one.

```bash
kubectl apply -f getting-started/build.yaml
```

The logs are easy to tail:

```bash
logs hello
```

## Congratulations! You know everything about simple builds.

<walkthrough-conclusion-trophy></walkthrough-conclusion-trophy>

**Why not build a Docker image from a Dockerfile now?**

[![Open in Cloud Shell](http://gstatic.com/cloudssh/images/open-btn.svg)](https://console.cloud.google.com/cloudshell/open?git_repo=https%3A%2F%2Fgithub.com%2FGoogleContainerTools%2Fknative-build-tutorials&page=editor&tutorial=docker-build/README.md&open_in_editor=.)

<walkthrough-footnote>
Copyright 2018 Google LLC All Rights Reserved. Licensed under the Apache
License, Version 2.0 (the "License"); you may not use this file except in
compliance with the License. You may obtain a copy of the License at
http://www.apache.org/licenses/LICENSE-2.0.
Unless required by applicable law or agreed to in writing, software
distributed under the License is distributed on an "AS IS" BASIS, WITHOUT
WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied. See the
License for the specific language governing permissions and limitations under
the License.
</walkthrough-footnote>