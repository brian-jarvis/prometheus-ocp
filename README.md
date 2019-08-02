# Prometheus OCP

This role deploys a customized prometheus-operator to work alongside the cluster-monitoring operator present in OpenShift v3.11

It is intended to be run against an existing OpenShift 3.11 cluster.  You only need one instance in a cluster to monitor multiple applications.  New applications are automatically added to the monitoring by use of labels and creatiion of `ServiceMonitor` in each project.

This project has been modified from the original [github.com/Havilland/prometheus-ocp](https://github.com/Havilland/prometheus-ocp):
  + Removed use of ansible modules not available in Ansible 2.6.
  + Modified to make use of the same inventory as run to deploy cluster
  + Removed any relation to images not published by Red Hat (Thanos, pushgateway)
  + Set image urls to use oreg_url from inventory, allowing disconnected installs to work
  + Changed layout to match the openshift-ansible project
  + Made use of openshift-ansible roles where appropriate


## Role Variables
Below are the variables you might want to customize for your installation.

For defaults see [`private/roles/openshift_application_monitoring/defaults/main.yml`](private/roles/openshift_application_monitoring/defaults/main.yml)

* `cluster_prometheus_namespace`: Namespace where to deploy prometheus operator
* `cluster_prometheus_apiGroup`: apiGroup for the prometheus-operator (don't use monitoring.coreos.com if cluster-monitoring-operator is present)
* `cluster_prometheus_operator_serviceAccount`: Name of the prometheus-operator service account (currently also used for clusterrole)
* `cluster_prometheus_namespace_label`: Namespace label that determines if prometheus and grafana will be deployed.  
* `cluster_prometheus_grafana_storage_type`: What storage type should be used for grafana (none or pvc)
* `cluster_prometheus_default_labelselector:`: Default label selector to be used by the Prometheus Operator to discover Custom Resources such as ServiceMonitors.  All `ServiceMonitor` and application namespaces need to be labeled with to be monitored.
* `cluster_monitoring_operator_node_selector: `: Set nodeSelector for prometheus-operator, prometheus, grafana, alertmanager
* `cluster_monitoring_operator_alertmanager_config`: Default alertmanager configuration.  Note this uses the same variable as the Cluster Monitoring Operator (Openshift Monitoring).
* Use the following variables to control the supplimental groups used for nfs storage:
  * `cluster_prometheus_alertmanager_storage_supplemental_group: "65534"`
  * `cluster_prometheus_grafana_storage_supplemental_group: "65534"`
  * `cluster_prometheus_storage_supplemental_group: "65534"`
  

## Usage
To execute the installation perform the following:

``` bash
  # Copy the repository to the ansible cluster host used to deploy the cluster
  mkdir ~/app-monitoring
  cd ~/app-monitoring
  git clone https://github.com/brian-jarvis/prometheus-ocp.git
  cd prometheus-ocp

  # execute the installation
  ansible-playbook -i [Inventory Path] ./config.yml
```
## Grafana storage
Before running the installation create a `PersistentVolume` that meets the following requirements:
  + Has a storage class named the same as the value of `{{ cluster_prometheus_grafana_serviceAccount }}-app-monitor` variable.  "app-grafana-app-monitor" by default.
  + Has a size of 1Gi

Set `cluster_prometheus_grafana_storage_type=pvc` in your inventory file.

## Prometheus and Alertmanager storage
The following variables are available to set storage for Prometheus and Alertmanager.  Create appropriate persistent volumes to match.

```
cluster_prometheus_storage_enabled: false
cluster_prometheus_storage_capacity: 5Gi
cluster_prometheus_storage_class_name: ""

cluster_prometheus_alertmanager_storage_enabled: false
cluster_prometheus_alertmanager_storage_capacity: 5Gi
cluster_prometheus_alertmanager_storage_class_name: ""
```

## Monitoring applications
To monitor an application perform the following steps:
1. Label the namespace so Prometheus knows to look for `ServiceMonitor` objects there
    ``` bash
      oc label ns [Namespace name] app-prometheus=monitoring
    ```

2. Create a `ServiceMonitor` in the application project.
Note the apiVersion.

    ``` yaml
    apiVersion: app-monitoring.redhat.io/v1
    kind: ServiceMonitor
    metadata:
    labels:
      app-prometheus: monitoring
    name: app-monitor
    namespace: app-namespace-name
    spec:
      endpoints:
        - interval: 10s
          port: web
      selector:
        matchLabels:
          app: example-app
    ```

After creating the ServiceMonitor you should see the target showup in Prometheus.

## License

BSD

## Author Information

Alexander Bartilla (alexander.bartilla@cloudwerkstatt.com)
Brian Jarvis (bjarvis@redhat.com)
