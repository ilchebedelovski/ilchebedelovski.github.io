---
layout: post
title: Tekton CI/CD on Kubernetes
---

## Introduction

The Tekton Pipelines project provides k8s-style resources for declaring CI/CD-style pipelines, it is an open-source, standalone project that provides an additional layer of abstraction by introducing the pipeline mentality in Kubernetes. Tekton Pipelines are Cloud Native:

- Run on Kubernetes
- Have Kubernetes clusters as a first class type
- Use containers as their building blocks

In this article you I am going to present you how you can set up your first pipeline in Tekton, if you need more specific info about Tekton you can find it [here](https://tekton.dev/).

### Bsic Tekon resources

The Tekton Pipeline project extends the Kubernetes API by five additional custom resource definitions (CRDs):

- `tasks`
- `taskrun`
- `pipeline`
- `pipelinerun`
- `pipelineresource`

I will go through some of the concepts with short details.

A `step` is an operation in a CI/CD workflow, such as running some unit tests for a Python web app, or the compilation of a Java program. Tekton performs each step with a container image you provide. For example, you may use the official Go image to compile a Go program in the same manner as you would on your local workstation (go build).

A `task` is a collection of steps in order. Tekton runs a task in the form of a Kubernetes pod, where each step becomes a running container in the pod. This design allows you to set up a shared environment for a number of related steps; for example, you may mount a Kubernetes volume in a task, which will be accessible inside each step of the task.

A `pipeline` is a collection of tasks in order. Tekton collects all the tasks, connects them in a directed acyclic graph (DAG), and executes the graph in sequence. In other words, it creates a number of Kubernetes pods and ensures that each pods complete running successfully as desired. Tekton grants developers full control of the process: one may set up a fan-in/fan-out scenario of task completion, ask Tekton to retry automatically should a flaky test exists, or specify a condition that a task must meet before proceeding.

## Requirements

