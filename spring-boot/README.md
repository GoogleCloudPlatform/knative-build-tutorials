# Knative Build

The previous tutorial taught you how to build a Docker image from a Dockerfile,
using Knative Build.

**Build a Docker image without Docker**

Let's try something different and build a Java Spring Boot web application, without using
Docker. It should still produce a Docker image at the end, though.

To achieve that goal, we are going to use Knative Build with [Jib](https://github.com/GoogleContainerTools/jib),
another open-source project from Google.

Here's the Kubernetes <walkthrough-editor-open-file filePath="build-crd/spring-boot/build.yaml">yaml manifest</walkthrough-editor-open-file>
to express such a build:

```yaml
apiVersion: build.knative.dev/v1alpha1
kind: Build
metadata:
  name: jib
spec:
  serviceAccountName: knative-build
  source:
    git:
      url: https://github.com/dgageot/hello-jib.git
      revision: master
 
  steps:
  - name: build-and-push
    image: dgageot/jib@sha256:a98d9adace11285b2344b588407cc25ac57a0ab6b55a4534b2ac97f0b0ed8609
    args: ["compile", "jib:build"]
```

### Git source

Like for the previous example, the build takes a git repository as an input.

```yaml
source:
  git:
    url: https://github.com/dgageot/hello-jib.git
    revision: master
```

### Jib

This time, we are using Jib to do the actual build.

*Pro tip: we use the fully qualified name of the jib docker image to make sure our build is reproducible.*

```yaml
- name: build-and-push
image: dgageot/jib@sha256:a98d9adace11285b2344b588407cc25ac57a0ab6b55a4534b2ac97f0b0ed8609
args: ["compile", "jib:build"]
```

Once the image is built, it'll be pushed to [Google Container Registry](https://cloud.google.com/container-registry/),
so, we are going to use the `knative-build` service account we've setup for previous tutorial.

**Click the `Continue` button to run the build.**

## Run the Build

TODO: Configure the image name

Let's run the build:

```bash
kubectl apply -f spring-boot/build.yaml
```

The build is running.

```bash
kubectl get builds
```

Tail the logs with:

```bash
logs jib
```

**Congratulations! You've had your first Java Application built with Knative Build.**

<walkthrough-conclusion-trophy></walkthrough-conclusion-trophy>

**Continue to next step, to improve the build file...**

## Persist cache across builds

If you run previous build a second time, you migh see that it will download files
that it has already downloaded during the first build.

That makes the build more reproducible but also slower.

It's most of the time safe to share the artifacts that Maven downloads across builds.
And it usually makes a build much faster.

Because Knative Build is native to Kubernetes, it can leverage [Persistent Volumes](https://kubernetes.io/docs/concepts/storage/persistent-volumes/)
to share files across builds.

### Update the Build manifest

We are going to use a more elaborate version of the Build manifest that looks like that:

```yaml
apiVersion: build.knative.dev/v1alpha1
kind: Build
metadata:
  name: jib-cache
spec:
  serviceAccountName: knative-build
  source:
    git:
      url: https://github.com/dgageot/hello-jib.git
      revision: master
 
  steps:
  - name: build-and-push
    image: dgageot/jib@sha256:a98d9adace11285b2344b588407cc25ac57a0ab6b55a4534b2ac97f0b0ed8609
    args: ["compile", "jib:build"]
    volumeMounts:
    - name: mvn-cache
      mountPath: /root/.m2

  volumes:
  - name: mvn-cache
    persistentVolumeClaim:
      claimName: cache
---
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: cache
spec:
  accessModes:
    - ReadWriteOnce
  volumeMode: Filesystem
  resources:
    requests:
      storage: 8Gi
```

This configuration does two things:

 + It creates a Persistent Volume to be shared by builds
 + It mounts this volume in `/root/.m2` during a build so that files written there will be available next time we run a build.

**Continue to next step, to give it a try...**

### Run with a cache

Let's run the build:

```bash
kubectl apply -f spring-boot/build-cache.yaml
```

The build is running.

```bash
kubectl get builds
```

Tail the logs with:

```bash
logs jib-cache
```

You should see the files beging downloded.

Let's run the same build a second time.

```bash
kubectl delete build jib-cache
```

```bash
kubectl apply -f spring-boot/build-cache.yaml
```

```bash
logs jib-cache
```

Now, the build should be a bit faster and you see in the logs that no file
was downloaded from Maven Central!

**The more dependencies your application has, the bigger the gain**

<walkthrough-conclusion-trophy></walkthrough-conclusion-trophy>

Congratulations! You've used a cache to make your builds much faster. You are an expert
user now!

---------------
Copyright 2018 Google LLC All Rights Reserved. Licensed under the Apache
License, Version 2.0 (the "License"); you may not use this file except in
compliance with the License. You may obtain a copy of the License at
http://www.apache.org/licenses/LICENSE-2.0

Unless required by applicable law or agreed to in writing, software
distributed under the License is distributed on an "AS IS" BASIS, WITHOUT
WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied. See the
License for the specific language governing permissions and limitations under
the License.