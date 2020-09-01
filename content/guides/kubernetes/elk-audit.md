---
title: "Setting up Kubernetes ApiServer auditing with ELK stack"
date: 2019-02-12
description: "Simple tutorial on how to set up Kubernetes apiserver auditing, with ELK backend"
featured_image: "/images/elk-audit/ELK.png"
categories:
- Kubernetes
- Tutorial
- ELK
tags: ["kubernetes", "elk", "audit", "logstash", "kibana", "elasticsearch"]
include_toc: true
---
# What is Kubernetes auditing?

[Kubernetes auditing][audit-manpage] provides a set of records or log entries documenting
what actions have affected the system. Basically, we have access to every API call, along
with some metadata.

This can be really useful:

- It provides you with security-relevant data, you can see WHO changed WHAT and WHEN.
- It can be used to detect bad configurations or possible attacks if your API server
  is exposed to the internet (if your apiserver shows a large ammount of 401 Unauthorized
  responses, someone might be trying to access your cluster with invalid credentials).
- You can detect patterns of usage based on user agents, who is making a lot of requests...
- Sometimes this is a hard requirement for organization-wide certifications that force
  you to have an audit log of every system.
  

# What is ELK?

[ELK][what-is-elk] stands for [Elasticsearch][elasticsearch-elastic], [Logstash][logstash-elastic]
and [Kibana][kibana-elastic]. Basically, Elasticsearch is a NoSQL database, Logstash is a log
pipeline (moves loges from sources to destinations) and Kibana is a visualization tool (think of Grafana).

They are all developed by Elastic, and are used together to perform log collection, aggregation,
storage, visualization and analysis. Sounds cool right? Let's see a very simple example on how
to setup this with docker-compose and a working K8s 1.10 cluster.

## Actual tutorial

You will need the following ingredients:
- A working kubernetes cluster. I will use v1.10, so some flags might change between versions.
- Somewhere to deploy your test ELK stack (I will use a different host with docker-compose).
- Some time.

### Setting up ELK

To set up ELK you can use almost any guide, but I deeply recommend 
[github/deviantony/docker-elk][deviantony/docker-elk], since it will let you set up a very
simple ELK stack with literally **docker-compose up**.

```bash
git clone https://github.com/deviantony/docker-elk --branch=master --depht=1
cd docker-elk
docker-compose up
```

We should be able to see all of the components running, and the Kibana interface at port 5601.

{{< figure src="/images/elk-audit/kibana-ui.png" title="Kibana UI after installing" >}}

For the sake of the guide, the deployed stack will have an accessible Logstash in:
- logstash.local


At the same time, we create a pipeline file for logstash, with the following contents:

```
# Instruct logstash to listen in port 5000. The K8s Apiserver will then send
# the logs here via webhook
input{
    http{
        port => 5000
    }
}
filter{
    split{
        # Webhook audit backend sends several events together with EventList
        # split each event here.
        field=>[items]
        # We only need event subelement, remove others.
        remove_field=>[headers, metadata, apiVersion, "@timestamp", kind, "@version", host]
    }
    mutate{
        rename => {items=>event}
    }
}

output {
    # In this case, we send the output to elasticsearch, in the Kubernetes index.
    # Since we are using docker, we can refer to elasticsearch with "elasticsearch",
    # and it will correctly resolve the elasticsearch container IP. 
	elasticsearch {
		hosts => "elasticsearch:9200"
		index => "kubernetes"
	}
}
```
### Setting up Kubernetes
We will then setup the following files in the Kubernetes nodes that have an API server. We have
to create a new file that the API server will mount.

Let's say we create a file in ***/etc/kubernetes/policy-webhook.yml*** with the following contents:

```yaml
apiVersion: v1
clusters:
- cluster:
    # Send the API server events to logstash, via the webhook plugin
    server: http://logstash.local:5000
  name: logstash
contexts:
- context:
    cluster: logstash
    user: ""
  name: default-context
current-context: default-context
kind: Config
preferences: {}
users: []

```
The API Server receives a LOT of traffic. You have to understand that the API server is the
main entrypoint for all other nodes and all kinds of internal controllers. This means that 
if you record every single request (or "event") in the API server, you will need a LOT of storage,
and it will cause a decent ammount of extra load in the API server.
To select what kind of events you want to log or record, you can use an Audit policy file.
As an example, we will use the GCE recommended audit policy file, that we will copy
to ***/etc/kubernetes/audit-policy.yml*** in every node that hosts the API server:


```yaml
apiVersion: audit.k8s.io/v1beta1
kind: Policy
rules:
  - level: None
    users: ["system:kube-proxy"]
    verbs: ["watch"]
    resources:
      - group: "" # core
        resources: ["endpoints", "services", "services/status"]
  - level: None
    users: ["system:unsecured"]
    namespaces: ["kube-system"]
    verbs: ["get"]
    resources:
      - group: "" # core
        resources: ["configmaps"]
  - level: None
    users: ["kubelet"] # legacy kubelet identity
    verbs: ["get"]
    resources:
      - group: "" # core
        resources: ["nodes", "nodes/status"]
  - level: None
    userGroups: ["system:nodes"]
    verbs: ["get"]
    resources:
      - group: "" # core
        resources: ["nodes", "nodes/status"]
  - level: None
    users:
      - system:kube-controller-manager
      - system:kube-scheduler
      - system:serviceaccount:kube-system:endpoint-controller
    verbs: ["get", "update"]
    namespaces: ["kube-system"]
    resources:
      - group: "" # core
        resources: ["endpoints"]
  - level: None
    users: ["system:apiserver"]
    verbs: ["get"]
    resources:
      - group: "" # core
        resources: ["namespaces", "namespaces/status", "namespaces/finalize"]
  # Don't log HPA fetching metrics.
  - level: None
    users:
      - system:kube-controller-manager
    verbs: ["get", "list"]
    resources:
      - group: "metrics.k8s.io"
  # Don't log these read-only URLs.
  - level: None
    nonResourceURLs:
      - /healthz*
      - /version
      - /swagger*
  # Don't log events requests.
  - level: None
    resources:
      - group: "" # core
        resources: ["events"]
  # node and pod status calls from nodes are high-volume and can be large, don't log responses for expected updates from nodes
  - level: Request
    users: ["kubelet", "system:node-problem-detector", "system:serviceaccount:kube-system:node-problem-detector"]
    verbs: ["update","patch"]
    resources:
      - group: "" # core
        resources: ["nodes/status", "pods/status"]
    omitStages:
      - "RequestReceived"
  - level: Request
    userGroups: ["system:nodes"]
    verbs: ["update","patch"]
    resources:
      - group: "" # core
        resources: ["nodes/status", "pods/status"]
    omitStages:
      - "RequestReceived"
  # deletecollection calls can be large, don't log responses for expected namespace deletions
  - level: Request
    users: ["system:serviceaccount:kube-system:namespace-controller"]
    verbs: ["deletecollection"]
    omitStages:
      - "RequestReceived"
  # Secrets, ConfigMaps, and TokenReviews can contain sensitive & binary data,
  # so only log at the Metadata level.
  - level: Metadata
    resources:
      - group: "" # core
        resources: ["secrets", "configmaps"]
      - group: authentication.k8s.io
        resources: ["tokenreviews"]
    omitStages:
      - "RequestReceived"
  # Get responses can be large; skip them.
  - level: Request
    verbs: ["get", "list", "watch"]
    resources:
      - group: "" # core
      - group: "admissionregistration.k8s.io"
      - group: "apiextensions.k8s.io"
      - group: "apiregistration.k8s.io"
      - group: "apps"
      - group: "authentication.k8s.io"
      - group: "authorization.k8s.io"
      - group: "autoscaling"
      - group: "batch"
      - group: "certificates.k8s.io"
      - group: "extensions"
      - group: "metrics.k8s.io"
      - group: "networking.k8s.io"
      - group: "policy"
      - group: "rbac.authorization.k8s.io"
      - group: "scheduling.k8s.io"
      - group: "settings.k8s.io"
      - group: "storage.k8s.io"
    omitStages:
      - "RequestReceived"
  # Default level for known APIs
  - level: RequestResponse
    resources: 
      - group: "" # core
      - group: "admissionregistration.k8s.io"
      - group: "apiextensions.k8s.io"
      - group: "apiregistration.k8s.io"
      - group: "apps"
      - group: "authentication.k8s.io"
      - group: "authorization.k8s.io"
      - group: "autoscaling"
      - group: "batch"
      - group: "certificates.k8s.io"
      - group: "extensions"
      - group: "metrics.k8s.io"
      - group: "networking.k8s.io"
      - group: "policy"
      - group: "rbac.authorization.k8s.io"
      - group: "scheduling.k8s.io"
      - group: "settings.k8s.io"
      - group: "storage.k8s.io"
    omitStages:
      - "RequestReceived"
  # Default level for all other requests.
  - level: Metadata
    omitStages:
      - "RequestReceived"

```
We also need to modify the API Server configuration file and add the following flags:

```yaml
    - --audit-policy-file=/etc/kubernetes/audit-policy.yml
    - --audit-webhook-config-file=/etc/kubernetes/policy-webhook.yml
    - --feature-gates=AdvancedAuditing=true
```

With these commands we basically tell the API server to use the ***audit-policy.yml*** rules
to select what events are worthy of being recorded.

At the same time, we select where we want to send these events with the ***policy-webhook.yml***
file. In this case, we want the events to be sent to logstash.

Once we have this setup, Logstash will start to receive events, and save them in Elasticsearch. We
can then go to Kibana and see that we have a new Index Pattern that we can use then to build
visualizations:
{{< figure src="/images/elk-audit/kibana-kubernetes-index.png" title="Kibana UI showing the Kubernetes Index coming from Logstash" >}}


We can then access the Kibana UI, and add the "Kubernetes" index pattern (we configure
this in Logstash), to start building our dashboards.



[deviantony/docker-elk]: https://github.com/deviantony/docker-elk
[audit-manpage]: https://kubernetes.io/docs/tasks/debug-application-cluster/audit/
[what-is-elk]: https://logz.io/learn/complete-guide-elk-stack/
[elasticsearch-elastic]: https://www.elastic.co/products/elasticsearch
[kibana-elastic]: https://www.elastic.co/products/kibana
[logstash-elastic]: https://www.elastic.co/products/logstash