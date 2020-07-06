# Introduction to the OpenShift 4.3 Logging Stack

July 6th, 2020 | by Ryan Devlin

![Logging Stack](https://github.com/ryandevlin-redhat/LoggingBlog/blob/master/logging-unsplash.jpg "A typical logging stack.")

*__Figure 1:__ A typical logging stack configuration, courtesy of Unsplash*

## Introduction:
The Openshift logging stack is an OpenShift component that is ubiquitous in most enterprise clusters. It acts as a virtual witness to cluster activity and provides a scalable mechanism for recording everything that happens inside the cluster. The purpose of this post is to introduce the logging stack to administrators new to OpenShift and to explain how several open source technologies link together to provide its flexibility and functionality.

## Topology:
The logging stack consists of five major components. These are:
1. Event Router
2. Fluentd
3. Elasticsearch
4. Kibana
5. Curator

Don’t worry about the names, for now; we’ll cover what they do below. The main point here is that these five open source components chain together to facilitate all of OpenShift’s logging capabilities. Logging in OpenShift is a multistage process where log data is harvested from containers, nodes, and OpenShift itself, transported to a central location and stored/managed for later use. It’s helpful to think of the logging stack as a flow of events that feed into one another. 


*__Figure 2:__ A topological view of the logging stack, in a typical configuration.*


At a very high level, it begins with Event Router and containers handing their logs off to Fluentd. Fluentd aggregates these logs and ships them to Elasticsearch, which acts as the centralized store of the logs and is responsible for managing them. Once the logs are in Elasticsearch, they can be accessed and perused via a GUI provided by Kibana. Finally, Curator acts as the entity responsible for overseeing log deletion and management so that older logs can be rotated out as new ones come in. If we break out this process, we begin to get a sense of each sub-component's purpose in the logging stack.

## Event Router:
The operation begins with event processing. Event Router is a component that is deployed as a pod in the cluster, and its purpose is to ensure that OpenShift events are saved as logs. It does this by collecting any events occurring in the cluster, formatting them as JSON, and then handing them off to Fluentd. This allows processes, such as container creations, or failures when attaching a volume, to be reported in the cluster logs. Over time, this gives administrators a full record of what’s happening in the cluster.



## Fluentd:
The second part of the logging stack is the log collection. As mentioned before, this is performed by Fluentd. There are multiple moving parts involved with Fluentd, so it’s best to start the explanation from the ground up. If we think about a typical OpenShift cluster, there are several nodes hosting many pods, all managed by Kubernetes and OpenShift. These pods encapsulate containers, which are running processes that are usually outputting a steady stream of messages to places like STDOUT and STDERR. The messages printed are often helpful information like errors in Java or connection statuses for an application's HTTP requests. In the OpenShift logging stack, Fluentd comes configured to collect all of these logs, and the event logs produced by Event Router. 
Fluentd is deployed as a daemonset, which allows for a Fluentd pod to be deployed on every node in the cluster. Because of this, each node can aggregate all the logs produced by containers on that node. This ensures that no matter where a pod is deployed, it’s logs will be picked up by Fluentd. In a typical configuration, container logs are located at /var/log/ inside a container, or are collected by journald running in the container. OpenShift comes pre-configured to aggregate these logs, and store them in the following directory on the node:
/var/log/containers/POD-NAME_NAMESPACE-NAME_CONTAINER-NAME-CONTAINER-ID.log 
It is via this directory that a Fluentd pod running on the node picks up the logs and prepares them for transport. Upon collection, Fluentd injects extra metadata into the log files such as the namespace, pod name, and container name where the logs originated. This makes it easier to search through the logs, sorting by pod name or namespace, for instance. After collection, Fluentd automatically handles the process of sending the logs to Elasticsearch pods in the cluster for storage and management. Because of its role in log collection, Fluentd pod failures are among the first things administrators should check if logs are not appearing in Kibana.
*__Figure 3:__ Log collection. Keep in mind that a Fluentd pod runs on every node in the cluster.*

## Elasticsearch:
The next and most central piece of the logging stack is Elasticsearch, the component where logs are stored. Elasticsearch usually consists of three pods, for high availability, each on a different node, for redundancy. These pods are responsible for long term storage of the logs, and things like RBAC policy that protect logs from being read by unauthorized users. The Elasticsearch deployments are configured with their own special storage volume so that they have enough disk space to hold as many logs as the cluster needs. Elasticsearch intakes logs from Fluentd and formats them in what are called “indices.” These indices are organized into a few types. Indices labeled .operations.<date> are indices that hold records of cluster events collected by Event Router. Indices labeled .project.<project_name> are indices that hold logs from containers corresponding to that project, respectively. While it is possible to manually obtain Elasticsearch logs via a REST api, it is a best practice to view them using Kibana.

## Kibana:
At the end of the chain of operations that comprise the logging stack is Kibana, a component that provides a GUI to view the logs. In a typical cluster, Kibana exists as a single pod that serves web pages via a URL provided by an OpenShift route. Under the hood, when a user interacts with the Kibana GUI, the pod issues API requests to the Elasticsearch pods to retrieve and display the logs in the indices. To locate the Kibana URL, an administrator can run oc get routes -n openshift-logging. When the Kibana URL is entered into a browser, the administrator can log into a GUI which presents the logs in a visual format and provides mechanics for sorting and searching the logs.


*__Figure 4:__ An example of a Kibana dashboard, for more information see here*


## Curator:
The final component of the logging stack is Curator, which is responsible for cluster log rotation. Log rotation is the act of periodically deleting older logs from a log store in order to save on disk space. Curator exists as a single pod in the openshift-logging namespace and is configured as a cron job. At the time dictated by the cron job, the curator pod is spun up and will perform deletion actions as determined by its mounted configmap. For instance, Curator’s configmap can be set up so that Curator will delete any logs that are more than seven days old. This prevents older logs from weighing down the disks underpinning Elasticsearch.

## Takeaways:
This post was created to bring transparency to the logging stack so that new administrators can jumpstart their knowledge of how the logging components function. With a high-level understanding of how the logging stack works, administrators can diagnose problems faster and deploy these components more effectively.

## Next Steps:

- Information on how to fully deploy OpenShift logging is located *[here](https://docs.openshift.com/container-platform/4.3/logging/cluster-logging.html)*.
- Documentation on Kibana, including how to create dashboards, can be found *[here](https://www.elastic.co/guide/en/kibana/current/index.html)*.
- A deep dive on configuring Elasticsearch is covered *[here](https://www.elastic.co/guide/en/elasticsearch/reference/current/index.html)*.
- Fluentd documentation is found *[here](https://docs.fluentd.org/)*.
- If you desire unique configuration or want to know more about how components have been integrated into OpenShift, you can find source code available *[here](https://github.com/openshift/origin-aggregated-logging)*.
