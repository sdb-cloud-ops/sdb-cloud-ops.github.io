---
title: Monitoring page
tags:
 - Monitoring
#description: Monitoring page
---

# Monitoring SingleStore on Kubernetes
This document will show you how to install Prometheus, Grafana, and Grafana Loki on a GKE cluster. It will also provide basics on Grafana use including adding the Loki data source to Grafana in order to view and query logs and exploring the default dashboards provided by the kube-prometheus stack.

# Prometheus
> Note we are using the [kube-prometheus](https://github.com/prometheus-operator/kube-prometheus) stack. Please refer to this doc for reference.

#### Install dependencies
> On MacOS you can use `brew` to install these packages

1. [jsonnet-bundler](https://github.com/jsonnet-bundler/jsonnet-bundler)

2. [jsonnet](https://github.com/google/jsonnet)

3. [go](https://go.dev/doc/install)

4. gojsontoyaml library
```
  go install github.com/brancz/gojsontoyaml@latest
  go install github.com/google/go-jsonnet/cmd/jsonnet@latest
```
5. wget


#### Create the install directory
```
mkdir kube-prometheus
```
```
cd kube-prometheus
```

#### Initialize jb and install kube-prometheus
```
jb init
```
```
jb install github.com/prometheus-operator/kube-prometheus/jsonnet/kube-prometheus@main
```

#### Pull the example.jsonnet and build.sh files
```
wget https://raw.githubusercontent.com/prometheus-operator/kube-prometheus/main/example.jsonnet -O example.jsonnet
```
```
wget https://raw.githubusercontent.com/prometheus-operator/kube-prometheus/main/build.sh -O build.sh
```

#### Update jb
```
jb update
```

#### Make build.sh executable
```
chmod +x build.sh
```

> Note: If you need to update GOPATH in the build.sh file edit line as below:
>
> ```jsonnet -J vendor -m manifests "${1-example.jsonnet}" | xargs -I{} sh -c 'cat {} | $(go env GOPATH)/bin/gojsontoyaml > {}.yaml; rm -f {}' -- {}```
>

#### Build the customization file

```
vim memsql.jsonnet
```
```
local kp = (import 'kube-prometheus/main.libsonnet') + {
  values+:: {
    common+: {
      namespace: 'monitoring',
    },
    prometheus+:: {
      namespaces+: ['singlestore'],
    },
  },
  memsqlMetrics: {
    serviceMonitorMyNamespace: {
      apiVersion: 'monitoring.coreos.com/v1',
      kind: 'ServiceMonitor',
      metadata: {
        name: 'memsql-metrics',
        namespace: 'singlestore',
      },
      spec: {
        endpoints: [
          {
            path: '/metrics',
            targetPort: 'metrics',
          },
        ],
        jobLabel: 'app.kubernetes.io/name',
        selector: {
          matchLabels: {
            'app.kubernetes.io/name': 'memsql-cluster',
            'app.kubernetes.io/component': 'cluster',
          },
        },
      },
    },
  },
  memsqlClusterMetrics: {
    serviceMonitorMyNamespace: {
      apiVersion: 'monitoring.coreos.com/v1',
      kind: 'ServiceMonitor',
      metadata: {
        name: 'memsql-cluster-metrics',
        namespace: 'singlestore',
      },
      spec: {
        endpoints: [
          {
            path: '/cluster-metrics',
            targetPort: 'metrics',
          },
        ],
        jobLabel: 'app.kubernetes.io/name',
        selector: {
          matchLabels: {
            'app.kubernetes.io/name': 'memsql-cluster',
            'app.kubernetes.io/component': 'master',
          },
        },
      },
    },
  },
};

{ 'setup/0namespace-namespace': kp.kubePrometheus.namespace } +
{
  ['setup/prometheus-operator-' + name]: kp.prometheusOperator[name]
  for name in std.filter((function(name) name != 'serviceMonitor' && name != 'prometheusRule'), std.objectFields(kp.prometheusOperator))
} +
// serviceMonitor and prometheusRule are separated so that they can be created after the CRDs are ready
{ 'prometheus-operator-serviceMonitor': kp.prometheusOperator.serviceMonitor } +
{ 'prometheus-operator-prometheusRule': kp.prometheusOperator.prometheusRule } +
{ 'kube-prometheus-prometheusRule': kp.kubePrometheus.prometheusRule } +
{ ['alertmanager-' + name]: kp.alertmanager[name] for name in std.objectFields(kp.alertmanager) } +
{ ['blackbox-exporter-' + name]: kp.blackboxExporter[name] for name in std.objectFields(kp.blackboxExporter) } +
{ ['grafana-' + name]: kp.grafana[name] for name in std.objectFields(kp.grafana) } +
{ ['kube-state-metrics-' + name]: kp.kubeStateMetrics[name] for name in std.objectFields(kp.kubeStateMetrics) } +
{ ['kubernetes-' + name]: kp.kubernetesControlPlane[name] for name in std.objectFields(kp.kubernetesControlPlane) }
{ ['node-exporter-' + name]: kp.nodeExporter[name] for name in std.objectFields(kp.nodeExporter) } +
{ ['prometheus-' + name]: kp.prometheus[name] for name in std.objectFields(kp.prometheus) } +
{ ['prometheus-adapter-' + name]: kp.prometheusAdapter[name] for name in std.objectFields(kp.prometheusAdapter) } +
{ ['memsql-metrics-' + name]: kp.memsqlMetrics[name] for name in std.objectFields(kp.memsqlMetrics) } +
{ ['memsql-cluster-metrics-' + name]: kp.memsqlClusterMetrics[name] for name in std.objectFields(kp.memsqlClusterMetrics) }
```
> Note: Only change the namespace inside of this file where it says `singlestore` to the namespace that you have SingleStore deployed.


#### Apply the build to the manifest files
```
./build.sh memsql.jsonnet
```

#### Create the kubernetes objects
```
kubectl apply --server-side -f manifests/setup
```
```
kubectl apply -f manifests/
```

#### To access Prometheus locally
Run the command below and then go to http://localhost:9090 in your web browser to access Prometheus
```
kubectl --namespace monitoring port-forward svc/prometheus-k8s 9090
```
# Grafana Loki

> Note: further installation methods are provided in the Reference section below

## GKE

For this installation, we will be creating a Google cloud storage bucket for our backend log store.

### Prerequisites

#### Install [Tanka](https://github.com/grafana/tanka/releases)
> MacOS users can use `brew install tanka`

#### Create a Google cloud bucket and service account
Go to the Google cloud storage browser, and click `create a bucket`. We will need to create a service account in order for Loki to access the bucket, so next, go to IAM & Admin > Service Accounts, and `create a service account`. I gave my service account `Storage Admin` and `Storage Object Admin` roles. Once the service account is created, click into the account and go to the `Keys` section. There you can click `add key` and make sure to save the downloaded file.  

#### Create a secret based off the google key
```
kubectl -n monitoring create secret generic google-key --from-file=key.json=<path/to/downloaded/key.json>
```


### Install Grafana Loki with Tanka
> Please refer to [this documentation](https://grafana.com/docs/loki/latest/installation/tanka/) for reference.

#### Create a local directory for the Loki environment
```
mkdir loki
```
```
cd loki
```
#### Initialize Tanka
```
tk init
```
#### Find your kubernetes api server
```
vim ~/.kube/config
```
Find the `- cluster` section of the cluster you are using, and locate the `server:` section. It will be right below the cluster certificate, and right above the cluster name. This is your kubernetes api server.
```
tk env add environments/loki --namespace=monitoring --server=<Kubernetes API server>
```
#### Download and install the Loki and Promtail module using jb
```
jb install github.com/grafana/loki/production/ksonnet/loki@main
jb install github.com/grafana/loki/production/ksonnet/promtail@main
```
#### Create the htpasswd file
This example creates a `.loki` file with the `loki` username
```
htpasswd -c .loki loki
```
It will prompt for a new password. Enter this twice. Then view the contents of the file for input into the `htpasswd_contents:` section of the `main.jsonnet`

#### Edit the environments/loki/main.jsonnet file
Make sure to replace the commented items with values that are specific to your deployment.
```
vim environments/loki/main.jsonnet
```
```
local gateway = import 'loki/gateway.libsonnet';
local loki = import 'loki/loki.libsonnet';
local promtail = import 'promtail/promtail.libsonnet';
local k = import 'ksonnet-util/kausal.libsonnet';
local gcsBucket = import 'gcsBucket.libsonnet';

loki + promtail + gateway + gcsBucket {

  _config+:: {
    namespace: 'loki',
    htpasswd_contents: 'loki:$apr1$i.G5yqP8$LGo1ANcwfGH87unfpIr6m.', //content of your .loki file created above

    // GCS variables -- Remove if not using gcs
    storage_backend: 'gcs',
    gcs_bucket_name: 'sdb-loki', //name of the google bucket

    //Set this variable based on the type of object storage you're using.
    boltdb_shipper_shared_store: 'gcs',

    //Update the object_store and from fields
    loki+: {
      schema_config: {
        configs: [{
          from: '2022-02-02', //set this date to the date exactly 2 weeks ago from today
          store: 'boltdb-shipper',
          object_store: 'gcs',
          schema: 'v11',
          index: {
            prefix: '%s_index_' % $._config.table_prefix,
            period: '%dh' % $._config.index_period_hours,
          },
        }],
      },
    },

    //Update the container_root_path if necessary
    promtail_config+: {
      clients: [{
        scheme:: 'http',
        hostname:: 'gateway.%(namespace)s.svc' % $._config,
        username:: 'loki',
        password:: 'loki', //same password that was used for the loki username in the htpasswd file
        container_root_path:: '/var/lib/docker',
      }],
    },

    replication_factor: 3,
    consul_replicas: 1,
  },
}
```
#### Create a new file in the lib directory that edits the yaml files for GKE specific variables
```
vim lib/gcsBucket.libsonnet
```
```
//gcsBucket.libsonnet

// Import libs
{
  local volumeMount = $.core.v1.volumeMount,
  local container = $.core.v1.container,
  local statefulSet = $.apps.v1.statefulSet,
  local volume = $.core.v1.volume,
  local secret = $.core.v1.secret,
  local deployment = $.apps.v1.deployment,
  local pvc = $.core.v1.persistentVolumeClaim,
  local spec = $.core.v1.spec,
  local storageClass = $.storage.v1.storageClass,

  //distributor
  distributor_container+::
    container.withEnvMap({GOOGLE_APPLICATION_CREDENTIALS:'/var/secrets/google/key.json'}),

  distributor_deployment+:
    $.util.secretVolumeMount('google', '/var/secrets/google'),

  distributor_args+:: {
    'config.expand-env':'true',
  },


  //querier
  querier_container+::
    container.withEnvMap({GOOGLE_APPLICATION_CREDENTIALS:'/var/secrets/google/key.json'}),

  querier_statefulset+::
    $.util.secretVolumeMount('google', '/var/secrets/google'),

  querier_args+:: {
    'config.expand-env':'true',
  },

  querier_data_pvc+:
    pvc.mixin.spec.withStorageClassName('standard'),


  //ingester
  ingester_container+::
    container.withEnvMap({GOOGLE_APPLICATION_CREDENTIALS:'/var/secrets/google/key.json'}),

  ingester_statefulset+:
    $.util.secretVolumeMount('google', '/var/secrets/google'),

  ingester_args+:: {
    'config.expand-env':'true',
  },

  ingester_data_pvc+::
    pvc.mixin.spec.withStorageClassName('standard'),

  ingester_wal_pvc+::
    pvc.mixin.spec.withStorageClassName('standard'),

  //compactor
  compactor_container+::
    container.withEnvMap({GOOGLE_APPLICATION_CREDENTIALS:'/var/secrets/google/key.json'}),

  compactor_statefulset+:
    $.util.secretVolumeMount('google', '/var/secrets/google'),

  compactor_args+:: {
    'config.expand-env':'true',
  },

  compactor_data_pvc+::
    pvc.mixin.spec.withStorageClassName('standard'),


  //table-manager
  table_manager_container+::
    container.withEnvMap({GOOGLE_APPLICATION_CREDENTIALS:'/var/secrets/google/key.json'}),

  table_manager_deployment+:
    $.util.secretVolumeMount('google', '/var/secrets/google'),

  table_manager_args+:: {
    'config.expand-env':'true',
  },


  //query-frontend
  query_frontend_container+::
    container.withEnvMap({GOOGLE_APPLICATION_CREDENTIALS:'/var/secrets/google/key.json'}),

  query_frontend_deployment+:
    $.util.secretVolumeMount('google', '/var/secrets/google'),

  query_frontend_args+:: {
    'config.expand-env':'true',
  },

}
```
#### Use tanka to create the yaml files
```
tk export manifests environments/loki
```
This will give you a `manifests` directory of yaml files for review.

#### Apply the yaml files to the kubernetes cluster
```
kubectl --namespace monitoring create -f manifests
```

## Add the Loki data source to Grafana

Run the command below and then go to http://localhost:3000 in your web browser to access Grafana
```
kubectl --namespace monitoring port-forward svc/grafana 3000:3000
```

Go to Configuration > Data Sources  
<img width="252" alt="Screen Shot 2022-02-01 at 2 35 32 PM" src="https://user-images.githubusercontent.com/16610646/152038532-d4a3b68a-da39-4b94-b45c-a31401386b93.png">

Click add data source and then type in 'Loki'  
<img width="190" alt="Screen Shot 2022-02-01 at 2 36 44 PM" src="https://user-images.githubusercontent.com/16610646/152038758-e6f8d6fc-d802-4a22-a069-16c5dc95d63e.png">

Enter the Loki URL  
<img width="690" alt="Screen Shot 2022-02-01 at 2 37 45 PM" src="https://user-images.githubusercontent.com/16610646/152038840-e74cef93-b044-4edd-bca4-8dcc443f9787.png">

Click 'Save & Test' and you should see the message below  
<img width="570" alt="Screen Shot 2022-02-01 at 2 38 21 PM" src="https://user-images.githubusercontent.com/16610646/152038985-3d2c9fcf-92cd-4867-9e64-ec516af68d2e.png">

Now you can explore SingleStore logs through Grafana + Loki  
<img width="237" alt="Screen Shot 2022-02-01 at 2 42 01 PM" src="https://user-images.githubusercontent.com/16610646/152039628-dfbc60e1-5add-4d1c-95d4-fff0139572cf.png">

Expand the Log Browser in order to have a helpful UI  
<img width="128" alt="Screen Shot 2022-02-01 at 2 42 27 PM" src="https://user-images.githubusercontent.com/16610646/152039715-46109465-a4a0-403a-922a-af8e363202bd.png">

Example query with SingleStore logs  
<img width="1368" alt="Screen Shot 2022-02-01 at 2 42 46 PM" src="https://user-images.githubusercontent.com/16610646/152039764-446028e1-9fb8-4445-ab33-0d646031d550.png">

> Here is further information on [LogQL Queries](https://grafana.com/docs/loki/latest/logql/)

## Access Grafana Dashboards
> Kube-prometheus added 24 dashboards to Grafana to get you started with dashboard monitoring

Click on Dashboards > Browse  
<img width="247" alt="Screen Shot 2022-02-01 at 2 45 32 PM" src="https://user-images.githubusercontent.com/16610646/152040633-dfc416c5-ae0a-4036-b62e-b43243c590f3.png">

Expand the Default folder  
<img width="1373" alt="Screen Shot 2022-02-01 at 2 45 53 PM" src="https://user-images.githubusercontent.com/16610646/152040686-5a7c0dea-419a-45d9-9eab-fb5bc31b805b.png">

Click on a dashboard to try it out!

Now we have installed a monitoring stack consisting of Prometheus and Grafana Loki. Further reading:
[Dashboard overview](https://grafana.com/docs/grafana/latest/dashboards/),
[Grafana alerting](https://grafana.com/docs/grafana/latest/alerting/unified-alerting/)

## Reference

## Further installation methods of Grafana Loki

### Install Grafana Loki with Helm with a persistant volume claim

> Further installation methods are detailed [here](https://grafana.com/docs/loki/latest/installation/)

Prereqs: Install [Helm](https://helm.sh/docs/intro/install/), then run the commands below

```
helm repo add grafana https://grafana.github.io/helm-charts
```
```
helm repo update
```

#### Install Loki + Promtail
```
helm upgrade --install loki --namespace=monitoring grafana/loki-stack  --set loki.persistence.enabled=true,loki.persistence.storageClassName=standard,loki.persistence.size=5Gi
```
> Note: you can change the size of the volume claim

Grafana is accessed the same way, but not that the Loki URL is http://loki:3100 when adding the Loki datasource to Grafana after installing with Helm
