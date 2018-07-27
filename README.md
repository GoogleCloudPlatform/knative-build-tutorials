# Knative Build Tutorials

[Knative Build](https://github.com/knative/build) is an open-source project
that provides an implementation of the Build [CRD](https://kubernetes.io/docs/concepts/extend-kubernetes/api-extension/custom-resources/) that runs Builds on-cluster.

It's not a complete standalone product that could be used for CI/CD.
Think of it as a building block to facilitate the expression of Builds as part of larger systems.

**This repository contains a list of tutorials to get started with Knative Build.**

## Hello, World

Learn how to install Knative Build, run a simple Build and inspect its result.

[![Open in Cloud Shell](http://gstatic.com/cloudssh/images/open-btn.svg)](https://console.cloud.google.com/cloudshell/open?git_repo=https%3A%2F%2Fgithub.com%2FGoogleCloudPlatform%2Fknative-build-tutorials&page=editor&tutorial=getting-started/README.md&open_in_editor=.)

## Docker Build with Kaniko

Learn how to build a Docker image from a Dockerfile, using [Kaniko](https://github.com/GoogleContainerTools/kaniko).

[![Open in Cloud Shell](http://gstatic.com/cloudssh/images/open-btn.svg)](https://console.cloud.google.com/cloudshell/open?git_repo=https%3A%2F%2Fgithub.com%2FGoogleCloudPlatform%2Fknative-build-tutorials&page=editor&tutorial=docker-build/README.md&open_in_editor=.)

## Spring Boot Java application with Jib

Learn how to build a Spring Boot application without Docker, using [Jib](https://github.com/GoogleContainerTools/jib).

[![Open in Cloud Shell](http://gstatic.com/cloudssh/images/open-btn.svg)](https://console.cloud.google.com/cloudshell/open?git_repo=https%3A%2F%2Fgithub.com%2FGoogleCloudPlatform%2Fknative-build-tutorials&page=editor&tutorial=spring-boot/README.md&open_in_editor=.)
