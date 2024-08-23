<img src='grafana-lgtm.png' height="480">

## 💡 Description
The Grafana LGTM stack is a comprehensive set of open-source tools designed for monitoring, observability, and visualization. It includes several key components, each serving a specific purpose to provide a complete solution for monitoring applications and infrastructure. If you are interested in setting up a Grafana LGTM stack and working with it on a kubernetes clsuter, please follow the instructions provided in this repository.

## :wrench: What tools do we want to use in this repository
  - **Loki**: a log aggregation system designed to store and query logs from all your applications and infrastructure. https://grafana.com/oss/loki/
  - **Grafana**: allows you to query, visualize, alert on and understand your metrics no matter where they are stored. https://grafana.com/oss/grafana/
  - **Tempo**: an open source, easy-to-use, and high-scale distributed tracing backend. https://grafana.com/oss/tempo/
  - **Mimir**: an open source, horizontally scalable, highly available, multi-tenant TSDB for long-term storage for Prometheus. https://grafana.com/oss/mimir/
  - **Agent**: is a batteries-included, open source telemetry collector for collecting metrics, logs, and traces. https://grafana.com/oss/agent/

## 🔎 What do we want to do
  - **Traces with Tempo**:
     - We want to write a simple Python app, collect its spans and traces using the `OpenTelemetry Kubernetes operator`, forward these traces to `Tempo`, and finally visualize the traces using `Grafana`.
  - **Metrics with Mimir and Logs with Loki**:
     - Using the `agent operator` to collect kubelet and cAdvisor metrics exposed by the kubelet service. Each node in your cluster exposes kubelet metrics at /metrics and cAdvisor metrics at /metrics/cadvisor. Finally, remotely writing these metrics to `Mimir` and visualizing them with `Grafana`
     - Using the `agent operator` to collect logs from the cluster nodes, shipping them to a remote `Loki` endpoint and visualizing them with `Grafana`.


## :one: Scenario-1: Traces with Tempo

Create a unique Kubernetes namespace for tempo and grafana:

    kubectl create namespace tempo
    kubectl create namespace grafana

Set up a Helm repository using the following commands:

    helm repo add grafana https://grafana.github.io/helm-charts
    helm repo update

Use the following command to install Tempo using the configuration options we’ve specified in the `helm-values/tempo-values.yml` file:

    helm -n tempo install tempo grafana/tempo-distributed -f helm-values/tempo-values.yml

Use the following command to install Grafana using the configuration options we’ve specified in the `helm-values/grafana-values-scenario-1.yml` file:

    helm -n grafana install grafana grafana/grafana -f helm-values/grafana-values-scenario-1.yml

To install the Opentelemetry Operator in an existing cluster, make sure you have cert-manager installed and run:

    kubectl apply -f https://github.com/open-telemetry/opentelemetry-operator/releases/latest/download/opentelemetry-operator.yaml

Create an OpenTelemetry `Collector` and an `Instrumentation` resource with the configuration for the SDK and instrumentation.

    kubectl apply -f open-telemetry-operator/colllector.yml
    kubectl apply -f open-telemetry-operator/instrument.yml

Go to `trace-python-app` directory and build the docker image using this command: `docker build -t myapp .` and then:

    kubectl apply -f trace-python-app-manifests/deployment.yml
    kubectl apply -f trace-python-app-manifests/service.yml

Use `port-forward` to connect to `my-app`, like this: `kubectl port-forward POD_NAME 8080` and then Go to your browser and type:

    http://localhost:8080/rolldice

Refresh the page multiple times. if you want to see the traces, use `port-forward` to connect to `grafana` and Go to `Explore`. You'll see the traces of my-app


## :two: Scenario-2: Metrics with Mimir and Logs with Loki, Collecting Metrics and Logs using Grafana Agent

We're going to use grafana/agent operator to collect metrics and logs from the kubernetes cluster and forward metrics to `Mimir` and pod logs to `loki` . Let's deploy an Nginx deployment with its related service and collect its logs using Grafana Agent. As you know, the log format of Nginx includes HTTP Method and Status Code, etc. Therefore, we will use pipelines in our Grafana Agent configuration to extract the HTTP Method and Status Code from Nginx access logs and add them as additional label to Loki, showcasing the power of Grafana Agent . Ready ? let's GO!

First of all, you need to seutp MiniO as an external object storage to store loki index and chunks, check the `helm-values/loki-values.yml` for the `loki.storage` section to find out how we can integrate MiniO with Loki. We also need to stote `mimir tsdb blocks` inside minio. I highly recommend you to check both `helm-values/loki-values.yml` and `helm-values/mimir-values.yml` because you need to replace some existing data with your own appropriate data.

Install the charts of agent-operator, loki, mimir, grafana as well as the nginx manifests

    helm install collector grafana/grafana-agent-operator
    helm install -n loki loki grafana/loki -f helm-values/loki-values.yml
    helm install -n mimir mimir grafana/mimir-distributed -f helm-values/mimir-values.yml
    helm install -n grafana grafana grafana/grafana -f helm-values/grafana-values-scenario-2.yml
    kubectl apply -f nginx/

Deploy the GrafanaAgent resource, The root of the custom resource hierarchy is the `GrafanaAgent` resource—the primary resource Agent Operator looks for. `GrafanaAgent` is called the root because it discovers other sub-resources, `MetricsInstance` and `LogsInstance`

    kubectl apply -f agent-operator/grafana-agent.yml

Deploy a MetricsInstance resource, Defines where to ship collected metrics. This rolls out a Grafana Agent StatefulSet that will scrape and ship metrics to a remote_write endpoint.

    kubectl apply -f agent-operator/metric-instance.yml

Create ServiceMonitors for kubelet and cAdvisor endpoints, Collects cAdvisor and kubelet metrics. This configures the MetricsInstance / Agent StatefulSet

    kubectl apply -f agent-operator/kubelet-svc-monitor.yml
    kubectl apply -f agent-operator/cadvisor-svc-monitor.yml

Deploy LogsInstance resource, Defines where to ship collected logs. This rolls out a Grafana Agent DaemonSet that will tail log files on your cluster nodes.

    kubectl apply -f agent-operator/log-instance.yml

Deploy PodLogs resource, Collects container logs from Kubernetes Pods. This configures the LogsInstance / Agent DaemonSet.

    kubectl apply -f agent-operator/pod-logs.yml
This example (`agent-operator/pod-logs.yml`) tails container logs for all Pods in the `nginx` namespace. You can restrict the set of matched Pods by using the `matchLabels` selector.

If you want to see the metrics and logs, use `port-forward` to connect to grafana and then Go to `Explore` and choose `mimir` as the datasource to see the metrics and choose `loki` to see the pods logs. you can see the pods logs of `nginx` namespace. if you want to add/remove any namespaces, Modify `matchLabels` selector in `agent-operator/pod-logs.yml`

For instance, you can use LogQL to extract log lines that include the status code `4.*`: `{namespace="nginx", status_code=~"4.*"}`



## :three: Scenario-3: Migrating from Grafana Agent to Grafana Alloy

Enter Grafana Alloy, the spiritual successor to Grafana Agent. Alloy is the new open source distribution of the OpenTelemetry Collector that is 100% OTLP compatible and offers native pipelines for OpenTelemetry and Prometheus telemetry formats, supporting metrics, logs, traces, and profiles. It does everything you’d expect of the Grafana Agent project and other agents — plus so much more. If you are currently using Grafana Agent in your observability stack, we encourage you to migrate to Alloy. both Grafana Agent and Grafana Agent Operator will no longer receive any new feature updates.

The Monitor types (`PodMonitor`, `ServiceMonitor`, `Probe`, and `PodLogs`) are all supported natively by Alloy. The parts of Grafana Agent Operator that deploy Grafana Agent, `GrafanaAgent`, `MetricsInstance`, and `LogsInstance` CRDs, are deprecated.

First of all, Let's start with `Metrics` (forwarding metrics to mimir) and then We'll move on to `Logs` (forwarding logs to loki)

## Convert `MetricsInstance` to Alloy components

Uninstall `GrafanaAgent` and `MetricsInstance` resources inside `default` namespace and then deploy the below configmap as well as `cadvisor` and `kubelet` servicemonitors inside `alloy-metrics` namespace and finally install alloy-metrics using helm:

    kubectl apply -f migrate-from-agent-to-alloy/metrics/alloy-configMap.yml
    migrate-from-agent-to-alloy/metrics/cadvisor-serviceMonitor.yml
    migrate-from-agent-to-alloy/metrics/kubelet-serviceMonitor.yml
    helm install alloy-metrics grafana/alloy -n alloy-metrics -f migrate-from-agent-to-alloy/metrics/alloy-helmValues.yml

## Collecting logs

Uninstall `GrafanaAgent`, `LogsInstance` and `PodLogs` resources inside `default` namespace and then deploy the below configmap as well inside `alloy-logs` namespace and finally install alloy-logs using helm:

    kubectl apply -f migrate-from-agent-to-alloy/logs/alloy-configMap.yml
    kubectl apply -f migrate-from-agent-to-alloy/logs/alloy-helmValues.yml
If you pay attention to `loki.process` inside `migrate-from-agent-to-alloy/logs/alloy-configMap.yml`, you find out that i use an additional `stage` which adds `status_code` and `method` labels to the pods inside `nginx` namespace. if you take a look at `Explore` section of grafana, All logs within `nginx` namespace, have those added labels.
