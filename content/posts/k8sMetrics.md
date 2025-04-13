---
title: "Demystifying Kubernetes Metrics"
date: "2025-03-10"
tags: ["Metrics", "Observability", "Kubernetes"]
series: ["Themes Guide"]
ShowToc: false
TocOpen: true   
---

### Intro

When setting up observability for Kubernetes, I kept running into the same question again and again:

**What metrics are actually available and where do they come from?** 

This post is my attempt to document all default and optional metric sources in a Kubernetes cluster, grouped by system layer. Also i want to provide a brief overview of each metric type an endpoint exposes.

### Core Components Recap

Before diving into metrics, here’s a quick reminder of some key Kubernetes components that either generate or expose metrics:

- **Kubelet**  
    Runs on every node. Manages pods and containers. Exposes various metrics including resource usage and probe status.

- **API Server**  
  Central access point for all Kubernetes operations. Tracks requests, authentication, and cluster object interactions.

- **Controller Manager**  
  Reconciles the desired and actual state of cluster resources like Deployments and Nodes. Emits control loop metrics.

- **Scheduler**  
  Assigns unscheduled pods to nodes. Provides insights into scheduling decisions and latency.

- **etcd**  
  The distributed key-value store backing all cluster data. Emits storage and leader election metrics.

- **kube-state-metrics**  
  Optional component that exposes the current state of Kubernetes resources such as Pods, Deployments, and Nodes. Focuses on metadata, not resource usage.