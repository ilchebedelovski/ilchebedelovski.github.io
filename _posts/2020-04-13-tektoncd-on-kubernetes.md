---
layout: post
title: Tekton CI/CD on Kubernetes
---

The Tekton Pipelines project provides k8s-style resources for declaring CI/CD-style pipelines, it is an open-source, standalone project that provides an additional layer of abstraction by introducing the pipeline mentality in Kubernetes. Tekton Pipelines are Cloud Native:

- Run on Kubernetes
- Have Kubernetes clusters as a first class type
- Use containers as their building blocks

In this article I am going to present to you how you can set up your first pipeline in Tekton, if you need more specific info about Tekton you can find it [here](https://tekton.dev/).

### Basic Tekton resources

The Tekton Pipeline project extends the Kubernetes API by five additional custom resource definitions (CRDs):

- `tasks`
- `taskrun`
- `pipeline`
- `pipelinerun`
- `pipelineresource`

I will go through some of the concepts with short details.

A `step` is an operation in a CI/CD workflow, such as running some unit tests for a Python web app, or the compilation of a Java program. Tekton performs each step with a container image you provide. For example, you may use the official Go image to compile a Go program in the same manner as you would on your local workstation (go build).

A `task` is a collection of steps in order. Tekton runs a task in the form of a Kubernetes pod, where each step becomes a running container in the pod. This design allows you to set up a shared environment for a number of related steps; for example, you may mount a Kubernetes volume in a task, which will be accessible inside each step of the task.

A `pipeline` is a collection of tasks in order. Tekton collects all the tasks, connects them in a directed acyclic graph (DAG), and executes the graph in sequence. In other words, it creates a number of Kubernetes pods and ensures that each pod complete running successfully as desired. Tekton grants developers full control of the process: one may set up a fan-in/fan-out scenario of task completion, ask Tekton to retry automatically should a flaky test exists, or specify a condition that a task must meet before proceeding.

## Requirements

- A running Kubernetes cluster is required where Tekton will be installed.
- Container Registry where you'll push your Docker image.

## Installing Tekton on Kubernetes

Installing Tekton on Kubernetes means that I will deploy the required resources on the cluster that will allow me to execute pipelines inside that cluster.

Using the `kubectl` client I will create a Kubernetes Secret from my Registry's credentials (I am using Docker Hub as Container Registry):
```
$ kubectl -n tekton-pipelines create secret docker-registry docker-hub --docker-server=$REGISTRY_HOST --docker-username=$REGISTRY_USERNAME --docker-password=$REGISTRY_PASSWORD --docker-email=$REGISTRY_EMAIL
```

Creating ServiceAccount for our Tekton Pipelines:
```
$ kubectl -n tekton-pipelines create serviceaccount tekton-pipelines
```

Creating ClusterRole and ClusterRoleBinding with specific roles for the previously created ServiceAccount:
```
$ kubectl create clusterrole tekton-pipelines-role --verb=get,list,watch,create,update,patch,delete --resource=deployments,deployments.apps,services,services,pods,secrets,ingress,pvc
$ kubectl create clusterrolebinding tekton-pipelines-binding --clusterrole=tekton-pipelines-role --serviceaccount=tekton-pipelines:tekton-pipelines
```

We just satisfied all dependencies before installing Tekton on our Kubernetes cluster, now we can install the latest Tekton release:
```
$ kubectl apply -f https://storage.googleapis.com/tekton-releases/pipeline/latest/release.yaml
```

## Creating a sample Pipeline

In this example I will present how you can create simple pipeline for Python Flask application. In my workflow I have:

- `PipelineResource`
- `Task`
- `Pipeline`
- `PipelineRun`

### PipelineResource

There are 2 resources:

- `git-repo` the repository from where you will clone the application.
- `image-registry` the Container Registry destination where the Docker Image will be pushed.

```
apiVersion: tekton.dev/v1alpha1
kind: PipelineResource
metadata:
  name: git-repo
  namespace: tekton-pipelines
spec:
  type: git
  params:
    - name: revision
      value: master
    - name: url
      value: ''
---
apiVersion: tekton.dev/v1alpha1
kind: PipelineResource
metadata:
  name: image-registry
  namespace: tekton-pipelines
spec:
  type: image
  params:
    - name: url
      value: ''
```

### Task

There is only one task named as `build-and-deploy`, in the main section I have `inputs`, `outputs` and `steps`. Inside that task there are few steps:

- `build-and-push` - where the image is built and pushed to my Container Registry. I am using [Kaniko](https://github.com/GoogleContainerTools/kaniko) for building and pushing the image.
- `run-helm-add` - my Helm Chart is publicly available, I am adding that repo to my local Helm setting.
- `run-helm-update` - a step for updating the local config with the recently added repository.
- `run-helm-upgrade` - a step for installing/upgrading the Helm Chart, in this step as you can see, several Helm Values are replaced, it is the application requirement for supplying the required variables to the Helm Chart and pass them to the scheduled Kubernetes Pod.

```
apiVersion: tekton.dev/v1alpha1
kind: Task
metadata:
  name: build-and-deploy
  namespace: tekton-pipelines
spec:
  inputs:
    resources:
      - name: git-repo
        type: git
    params:
      - name: pathToDockerFile
        type: string
      - name: pathToContext
        type: string
      - name: appChartRepo
        type: string
      - name: appName
        type: string
        .
        .
        .
  outputs:
    resources:
      - name: image-registry
        type: image
  steps:
    - name: build-and-push
      image: gcr.io/kaniko-project/executor:v0.18.0
      securityContext:
        runAsUser: 0
      env:
        - name: DOCKER_CONFIG
          value: /data/.docker/
      command:
        - /kaniko/executor
      args:
        - --dockerfile=$(inputs.params.pathToDockerFile)
        - --destination=$(inputs.params.imageRepository)
        - --context=$(inputs.params.pathToContext)
        - --skip-tls-verify
        - --verbosity=debug
      volumeMounts:
        - name: kaniko-secret
          mountPath: /data
    - name: run-helm-add
      image: alpine/helm:3.1.1
      command: ["helm"]
      args:
        - repo
        - add
        - $(inputs.params.appName)
        - $(inputs.params.appChartRepo)
    - name: run-helm-update
      image: alpine/helm:3.1.1
      command: ["helm"]
      args:
        - repo
        - update
    - name: run-helm-upgrade
      image: alpine/helm:3.1.0
      command: ["helm"]
      args:
        - upgrade
        - --install
        - --namespace=$(inputs.params.appName)
        - $(inputs.params.appName)
        - api/api
        - --set
        - namespace=$(inputs.params.appName)
        - --set
        - name=$(inputs.params.appName)
        - --set
        - image.repository=$(inputs.params.imageRepository)
        .
        .
        .
  volumes:
    - name: kaniko-secret
      secret:
        secretName: docker-hub
        items:
          - key: .dockerconfigjson
            path: .docker/config.json
```

### Pipeline

As I mentioned earlier, a `pipeline` is a collection of tasks in order, in my `pipeline` I have `resources`, `params` and `tasks`. The vital in this `pipeline` is the moment of calling my previous task named as `build-and-deploy` with the already defined resources.

```
kind: Pipeline
metadata:
  name: api-pipeline
  namespace: tekton-pipelines
spec:
  resources:
    - name: git-repo
      type: git
    - name: image-registry
      type: image
  params:
    - name: pathToDockerFile
      type: string
    - name: pathToContext
      type: string
    - name: appChartRepo
      type: string
    - name: appName
      type: string
      .
      .
      .
  tasks:
    - name: build-and-deploy
      taskRef:
        name: build-and-deploy
      params:
        - name: pathToDockerFile
          value: "$(params.pathToDockerFile)"
        - name: pathToContext
          value: "$(params.pathToContext)"
        - name: appChartRepo
          value: "$(params.appChartRepo)"
        - name: appName
          value: "$(params.appName)"
          .
          .
          .
      resources:
        inputs:
          - name: git-repo
            resource: git-repo
        outputs:
          - name: image-registry
            resource: image-registry
```

### PipelineRun

This deployment is crucial for running the previous pipeline, I am referencing the `pipeline` name where I am sending the resources and the values of the required parameters.

```
apiVersion: tekton.dev/v1alpha1
kind: PipelineRun
metadata:
  name: api-pipeline-run
  namespace: tekton-pipelines
spec:
  serviceAccountName: tekton-pipelines
  pipelineRef:
    name: api-pipeline
  resources:
    - name: git-repo
      resourceRef:
        name: git-repo
    - name: image-registry
      resourceRef:
        name: image-registry
  params:
    - name: "appName"
      value: ""
    - name: imageRepository
      .
      .
      .
```

## Conclusion

Tekton is a perfect CD solution, it provides open-source components for standardizing your CI/CD tooling and processes across different vendors, programming languages, and deployment environments. Industry specifications around pipelines, releases, workflows, and other CI/CD components available with Tekton will work well with existing CI/CD tools such as Jenkins, Jenkins X, Skaffold, and Knative, among others. Tekton provides a number of interactive tutorials at [try](https://try.tekton.dev) for developers to get hands-on experience with the project.

## References

- Tekton git repository, https://github.com/tektoncd/pipeline
- Official Tekton documentation, http://tekton.dev/docs
