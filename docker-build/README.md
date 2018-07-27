<walkthrough-author name="David Gageot" repositoryUrl="https://github.com/GoogleContainerTools/knative-build-tutorials" email="dgageot@google.com" tutorialName="knative-build-docker-build"></walkthrough-author>

# Knative Build - Docker build

The previous tutorial taught you how to setup Knative Build on a Kubernetes cluster. You ran a simple Build
that prints "Hello, World!".

Let's try a more concrete build: **building a Docker image**.

## What am I going to learn?

 1. This new tutorial, will show you how to build and push a Docker image described by a **Dockerfile**.
 2. It'll show you how to extract common build configuration into a Build Template

**Are you ready?** Then click the `Continue` button to get started...

## Docker build with Knative Build

We are going to build a very simple website with [nginx](https://www.nginx.com/).
The sources are on [GitHub](https://github.com/dgageot/hello).
A [Dockerfile](https://github.com/dgageot/hello/blob/master/Dockerfile)
describes the image to be built.

<walkthrough-spotlight-pointer spotlightId="devshell-web-editor-button">Open the file editor</walkthrough-spotlight-pointer>.
Here's the Kubernetes <walkthrough-editor-open-file filePath="knative-build-tutorials/getting-started/build.yaml">yaml manifest</walkthrough-editor-open-file> to express such a build:

```yaml
apiVersion: build.knative.dev/v1alpha1
kind: Build
metadata:
  name: docker-build
spec:
  serviceAccountName: knative-build
  source:
    git:
      url: https://github.com/dgageot/hello.git
      revision: master
  steps:
  - name: build-and-push
    image: gcr.io/kaniko-project/executor:v0.2.0
    args: ["--dockerfile=/workspace/Dockerfile",
           "--destination=gcr.io/[PROJECT-NAME]/hello-nginx"]
```

A few things look different than the simpler "Hello, World!" build.

**Click the `Continue` button to understand those differences...**

## Git source

This time, the build will need sources to build an artifact.

```yaml
source:
  git:
    url: https://github.com/dgageot/hello.git
    revision: master
```

**Kaniko**

The Build will use [Kaniko](https://github.com/GoogleContainerTools/kaniko)
to build a Docker image because just running `docker build` [is unsafe on a
shared cluster](https://github.com/kubernetes/kubernetes/issues/1806).

```yaml
- name: build-and-push
  image: gcr.io/kaniko-project/executor:v0.2.0
```

**Service Account**

Once the image is built, it'll be pushed to [Google Container Registry](https://cloud.google.com/container-registry/).
Or any registry in fact, but we'll demonstrate with GCR.
The build requires a service account to be granted the right to push.

```yaml
serviceAccountName: knative-build
```

This will done by creating a Kubernetes [Service Account](https://kubernetes.io/docs/reference/access-authn-authz/service-accounts-admin/)
that has access to a Google Cloud [Service Account](https://cloud.google.com/iam/docs/understanding-service-accounts).

**That's a lot of Service Accounts! Click the `Continue` button to do this step by step...**

## Configure access to Google Container Registry

Giving a Build the permisison to push to [Google Container Registry](https://cloud.google.com/container-registry/)
requires 4 steps:

 + Create a [Service Account](https://cloud.google.com/iam/docs/understanding-service-accounts) in your Google Cloud project. It needs to be be granted push access to GCR.
 + Create a [JSON key](https://cloud.google.com/container-registry/docs/advanced-authentication#json_key_file) for that Service Account.
 + Store this key in a [Kubernetes Secret](https://kubernetes.io/docs/concepts/configuration/secret/).
 + Create a Kubernetes [Service Account](https://cloud.google.com/iam/docs/understanding-service-accounts) that has access to this secret.

Once those steps are done, we can create as many builds as we want that will share those permissions.

**Click the `Continue` button to follow those steps...**

## Create a Google Cloud Service Account

The first part of the configuration is on the Google Cloud side.

### Create a Service Account

```bash
gcloud iam service-accounts create knative-build --display-name "Knative Build"
```

### Allow it to push to GCR

```bash
gcloud projects add-iam-policy-binding {{project-id}} --member serviceAccount:knative-build@{{project-id}}.iam.gserviceaccount.com --role roles/storage.admin
```

### Create a JSON key

```bash
gcloud iam service-accounts keys create knative-key.json --iam-account knative-build@{{project-id}}.iam.gserviceaccount.com
```

This creates a <walkthrough-editor-open-file filePath="knative-build-tutorials/docker-build/knative-key.json">knative-key.json</walkthrough-editor-open-file> file on your drive.

**You are almost there!** You completed the Google Cloud part.

**Click the `Continue` button to complete the Kubernetes part...**

## Create a Kubernetes Service Account

### Create a secret for the JSON Key

First, create a Kubernetes secret from the JSON key:

```bash
kubectl create secret generic knative-build-auth --type="kubernetes.io/basic-auth" --from-literal=username="_json_key" --from-file=password=knative-key.json
```

To tell the Build to use those parameters when pushing to `gcr.io`, we
need to add an annotation to the secret. That's called [Guided credential selection](https://github.com/knative/docs/blob/master/build/auth.md#guiding-credential-selection).

```bash
kubectl annotate secret knative-build-auth build.knative.dev/docker-0=https://gcr.io
```

### Create a Service Account

We can eventually create a Service Account in Kubernetes:

```bash
kubectl apply -f docker-build/service-account.yaml
```

**You are done! Continue to next step to run the build...**

## Run the Build

OK, we've got everything configured to let the Build push images to a registry.

One last step is to edit <walkthrough-editor-open-file filePath="knative-build-tutorials/docker-build/build.yaml">docker-build/build.yaml</walkthrough-editor-open-file>
and replace `[PROJECT-NAME]` with your project id (**{{project-id}}**).
This way, the build will push the image to the Google Container Registry linked to your project.

Run the build:

```bash
kubectl apply -f docker-build/build.yaml
```

The build is running:

```bash
kubectl get builds
```

Tail the logs with:

```bash
logs docker-build
```

**Congratulations! You have built a Docker image with Knative Build.**

<walkthrough-conclusion-trophy></walkthrough-conclusion-trophy>

**Continue to next step, to improve the build file...**

## Build Templates

Now that you became familiar with expressing Docker builds, you will
want to avoid copy/pasting the same boilerplate yaml each time a new
image needs to be built.

[Build Templates](https://github.com/knative/docs/blob/master/build/build-templates.md)
solve that problem by extracting the common yaml to a shared parameterized template.

### Create a Template

A Build Template for a Docker build looks like that:

```yaml
apiVersion: build.knative.dev/v1alpha1
kind: BuildTemplate
metadata:
  name: docker-build
spec:
  parameters:
  - name: IMAGE
    description: Where to publish the resulting image.
  - name: DIRECTORY
    description: The directory containing the build context.
    default: "/workspace"
  - name: DOCKERFILE_NAME
    description: The name of the Dockerfile
    default: Dockerfile
  steps:
  - name: build-and-push
    image: gcr.io/kaniko-project/executor:v0.2.0
    args: ["--dockerfile=/${DIRECTORY}/{DOCKERFILE_NAME}",
           "--destination=${IMAGE}"]
```

It's a list of steps and a list of parameters, some of which have default
values that can be overriden by a build.

Let's register this template into Kubernetes.

```bash
kubectl apply -f docker-build/template.yaml
```

### Use the Template

Now, the Build can be simplified by referencing the template and by
providing the right values for each parameter.

```yaml
apiVersion: build.knative.dev/v1alpha1
kind: Build
metadata:
  name: docker-build-hello
spec:
  serviceAccountName: knative-build
  source:
    git:
      url: https://github.com/dgageot/hello.git
      revision: master
  template:
    name: docker-build
    arguments:
    - name: IMAGE
      value: gcr.io/[PROJECT-NAME]/hello-nginx
```

**Click the `Continue` button to run the templated build...**

## Run the Templated Build

*Warning*: don't forget to replace `[PROJECT-NAME]` with you actual
project name (**{{project-id}}**) in
<walkthrough-editor-open-file filePath="knative-build-tutorials/docker-build/build-hello.yaml">docker-build/build-hello.yaml</walkthrough-editor-open-file>

Run the build:

```bash
kubectl apply -f docker-build/build-hello.yaml
```

The build is running:

```bash
kubectl get builds
```

Tail the logs with:

```bash
logs docker-build-hello
```

## Congratulations!

<walkthrough-conclusion-trophy></walkthrough-conclusion-trophy>

Woot! You've used a Build Template for the first time. You are an advanced
user now!

**Why not build a more complicated Java application?**

[![Open in Cloud Shell](http://gstatic.com/cloudssh/images/open-btn.svg)](https://console.cloud.google.com/cloudshell/open?git_repo=https%3A%2F%2Fgithub.com%2FGoogleContainerTools%2Fknative-build-tutorials&page=editor&tutorial=spring-boot/README.md&open_in_editor=.)

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