
# prometheus

![Version: 0.0.11](https://img.shields.io/badge/Version-0.0.11-informational?style=flat-square) ![Type: application](https://img.shields.io/badge/Type-application-informational?style=flat-square) ![AppVersion: 0.0.13](https://img.shields.io/badge/AppVersion-0.0.13-informational?style=flat-square)

Prometheus is a monitoring system and time series database.

**Homepage:** <https://prometheus.io/>

## Maintainers

| Name | Email | Url |
| ---- | ------ | --- |
| OpenZiti | <help.openziti.org> |  |

## Source Code

* <https://github.com/prometheus/alertmanager>
* <https://github.com/prometheus/prometheus>
* <https://github.com/prometheus/pushgateway>
* <https://github.com/prometheus/node_exporter>
* <https://github.com/kubernetes/kube-state-metrics>

## Requirements

| Repository | Name | Version |
|------------|------|---------|
| https://prometheus-community.github.io/helm-charts | kube-state-metrics | 4.0.* |

This Helm chart deploys Prometheus with embeded Ziti. Ziti solves the target reachability problem by enabling Prometheus to collect metrics from targets via a Ziti overlay instead of a regular network path.

## Overview

The operation of this chart is described in [part 2 of the PrometheuZ tutorial](https://docs.openziti.io/blog/zitification/prometheus/part2/#deploying-prometheuz-1).

```bash
helm install prometheuz ./charts/prometheus \
     --set-file configmapReload.ziti.id.contents="/ziti/id/to/reload/prometheus/after/change.json" \
     --set configmapReload.ziti.targetService="my.zitified.prometheus.svc" \
     --set configmapReload.ziti.targetIdentity="hosting.ziti.identity" \
     --set-file server.ziti.id.contents="/ziti/id/to/prometheus/ziti.id.json" \
     --set server.ziti.service="my.zitified.prometheus.svc" \
     --set server.ziti.identity="hosting.ziti.identity"
```

A Ziti identity for the Prometheus server is required as part of the deployment.

-----------------------

Configuring the chart

======================

By default the deployment sets up some scrape targets to scrape the system that prometheus server is deployed on. In order to scrape additional targets one of two methods can be used:

-----------------------------------------------
Editing the prometheus.yaml file once deployed:
-----------------------------------------------
jobs can be added to the prometheus.yaml after the service is deployed by running

kubectl edit cm prometheus-server

this opens a text editor that will allow you to modify the config map for the prometheus-server which contains the prometheus.yaml file. From here new jobs can be added. The pod should restart automatically

----------------------------------------
Deploying additional targets on install:
----------------------------------------
create a YAML file that will contain your additional scrape targets. the YAML file will look something like:

- job_name: 'job1'
  scrape_interval: 15s
  honor_labels: true
  scheme: 'ziti'
  params:
  'match[]':
  - '{job!=""}'
  'ziti-config':
  - '/etc/prometheus/{identityFileName}.json'
  static_configs:
    - targets:
      - '{serviceName}-{identityFileName}'

- job_name: 'job2'
  scrape_interval: 15s
  honor_labels: true
  scheme: 'ziti'
  params:
  'match[]':
  - '{job!=""}'
  'ziti-config':
  - '/etc/prometheus/{prometheusIdentityName}.json'

  static_configs:
  - targets:
    - '{serviceName}-{targetIdentityName}'

With the YAML file containing your additional targets, add the following arugment to your install command:

--set-file extraScrapeConfigs=myScrapeConfigs.yaml

where myScrapeConfigs.yaml is the path to the yaml containing the extra scrape configs

This method can also be used when updating scrape targets by running a helm update instead of an install

-----------------------------
Configuring the identity file
-----------------------------
The ziti identity json file must be provided during an install/updgrade in order to scrape ziti targets. This can be done by the use of the following arugment

--set-file prometheusIdentity=prometheus.json

where prometheus.json is the path to the identity file on the local machine.

By default, this helm chart will mount this file to /etc/prometheus/prometheus.json on the kubernetes cluster. Please take care to include the full path when providing your scrape targets

---------------------------------------
Changing the mounted identity file name
---------------------------------------
It is possible to change the name of the identity file that gets mounted on the kubernetes cluster if the prometheus.json identity name isn't desired. This can be done with the following argument:

--set identityFileName=myIdentity

where myIdentity is the name of the identity. This will cause the identity file to be mounted at /etc/prometheus/myIdentity.json instead of /etc/prometheus/prometheus.json

-----------------------------------------------------------------------------------------------------------------------------------

Example Installation Command

Assume that we have an identity file zitiPrometheus.json and we have a yaml file called zitiTargets.yaml which contains the following:

- job_name: 'redis'
  scrape_interval: 15s
  honor_labels: true
  scheme: 'ziti'
  params:
  'match[]':
  - '{job!=""}'
  'ziti-config':
  - '/etc/prometheus/prometheus.json'
  static_configs:
    - targets:
        - 'metrics-minikube'

- job_name: 'traefik'
  scrape_interval: 15s
  honor_labels: true
  scheme: 'ziti'
  params:
  'match[]':
  - '{job!=""}'
  'ziti-config':
  - '/etc/prometheus/prometheus.json'

  static_configs:
    - targets:
        - 'traefikPrometheus-prometheus'

The following command will allow us to install zitified prometheus:
// TODO update once remote
helm install prometheus . --set-file prometheusIdentity=zitiPrometheus.json --set-file extraScrapeConfigs=zitiTargets.yaml

**NOTE: since we did not provide an identityFileName the identity file is mounted as /etc/prometheus/prometheus.json by default

--------------------------

If we wanted to specify the identity file name to match the local identity file's name we would first change

/etc/prometheus/prometheus.json

to

/etc/prometheus/zitiPrometheus.json

in our zitiTargets.yaml file

the helm command would then look like:

helm install prometheus . --set-file prometheusIdentity=zitiPrometheus.json --set-file extraScrapeConfigs=zitiTargets.yaml --set identityFileName=zitiPrometheus

--------------------
Upgrading the Chart
--------------------

If we needed to add a new job to our zitiTargets we could either run

kubectl edit cm prometheus-server

to add the targets directly to the config map

Otherwise, we would run the same command as we did earlier except run a upgrade instead of an install. So for the case where the identityFileName was set it would look like:

helm upgrade prometheus . --set-file prometheusIdentity=zitiPrometheus.json --set-file extraScrapeConfigs=zitiTargets.yaml --set identityFileName=zitiPrometheus
kubectl scale deployment prometheus-server --replicas=0
kubectl scale deployment prometheus-server --replicas=1

by scaling the deployment down and then up this will force a restart so that the new scrapes get picked up

## Values Reference

| Key | Type | Default | Description |
|-----|------|---------|-------------|
| alertRelabelConfigs | string | `nil` |  |
| alertmanager.affinity | object | `{}` |  |
| alertmanager.baseURL | string | `"http://localhost:9093"` |  |
| alertmanager.clusterPeers | list | `[]` |  |
| alertmanager.configFileName | string | `"alertmanager.yml"` |  |
| alertmanager.configFromSecret | string | `""` |  |
| alertmanager.configMapOverrideName | string | `""` |  |
| alertmanager.deploymentAnnotations | object | `{}` |  |
| alertmanager.dnsConfig | object | `{}` |  |
| alertmanager.emptyDir.sizeLimit | string | `""` |  |
| alertmanager.enabled | bool | `false` |  |
| alertmanager.extraArgs | object | `{}` |  |
| alertmanager.extraEnv | object | `{}` |  |
| alertmanager.extraInitContainers | list | `[]` |  |
| alertmanager.extraSecretMounts | list | `[]` |  |
| alertmanager.image.pullPolicy | string | `"IfNotPresent"` |  |
| alertmanager.image.repository | string | `"quay.io/prometheus/alertmanager"` |  |
| alertmanager.image.tag | string | `"v0.23.0"` |  |
| alertmanager.ingress.annotations | object | `{}` |  |
| alertmanager.ingress.enabled | bool | `false` |  |
| alertmanager.ingress.extraLabels | object | `{}` |  |
| alertmanager.ingress.extraPaths | list | `[]` |  |
| alertmanager.ingress.hosts | list | `[]` |  |
| alertmanager.ingress.path | string | `"/"` |  |
| alertmanager.ingress.pathType | string | `"Prefix"` |  |
| alertmanager.ingress.tls | list | `[]` |  |
| alertmanager.name | string | `"alertmanager"` |  |
| alertmanager.nodeSelector | object | `{}` |  |
| alertmanager.persistentVolume.accessModes[0] | string | `"ReadWriteOnce"` |  |
| alertmanager.persistentVolume.annotations | object | `{}` |  |
| alertmanager.persistentVolume.enabled | bool | `true` |  |
| alertmanager.persistentVolume.existingClaim | string | `""` |  |
| alertmanager.persistentVolume.mountPath | string | `"/data"` |  |
| alertmanager.persistentVolume.size | string | `"2Gi"` |  |
| alertmanager.persistentVolume.subPath | string | `""` |  |
| alertmanager.podAnnotations | object | `{}` |  |
| alertmanager.podDisruptionBudget.enabled | bool | `false` |  |
| alertmanager.podDisruptionBudget.maxUnavailable | int | `1` |  |
| alertmanager.podLabels | object | `{}` |  |
| alertmanager.podSecurityPolicy.annotations | object | `{}` |  |
| alertmanager.prefixURL | string | `""` |  |
| alertmanager.priorityClassName | string | `""` |  |
| alertmanager.replicaCount | int | `1` |  |
| alertmanager.resources | object | `{}` |  |
| alertmanager.securityContext.fsGroup | int | `65534` |  |
| alertmanager.securityContext.runAsGroup | int | `65534` |  |
| alertmanager.securityContext.runAsNonRoot | bool | `true` |  |
| alertmanager.securityContext.runAsUser | int | `65534` |  |
| alertmanager.service.annotations | object | `{}` |  |
| alertmanager.service.clusterIP | string | `""` |  |
| alertmanager.service.externalIPs | list | `[]` |  |
| alertmanager.service.labels | object | `{}` |  |
| alertmanager.service.loadBalancerIP | string | `""` |  |
| alertmanager.service.loadBalancerSourceRanges | list | `[]` |  |
| alertmanager.service.servicePort | int | `80` |  |
| alertmanager.service.sessionAffinity | string | `"None"` |  |
| alertmanager.service.type | string | `"ClusterIP"` |  |
| alertmanager.statefulSet.annotations | object | `{}` |  |
| alertmanager.statefulSet.enabled | bool | `false` |  |
| alertmanager.statefulSet.headless.annotations | object | `{}` |  |
| alertmanager.statefulSet.headless.enableMeshPeer | bool | `false` |  |
| alertmanager.statefulSet.headless.labels | object | `{}` |  |
| alertmanager.statefulSet.headless.servicePort | int | `80` |  |
| alertmanager.statefulSet.labels | object | `{}` |  |
| alertmanager.statefulSet.podManagementPolicy | string | `"OrderedReady"` |  |
| alertmanager.tolerations | list | `[]` |  |
| alertmanager.useClusterRole | bool | `true` |  |
| alertmanager.useExistingRole | bool | `false` |  |
| alertmanager.zitified | bool | `false` |  |
| alertmanagerFiles."alertmanager.yml".global | object | `{}` |  |
| alertmanagerFiles."alertmanager.yml".receivers[0].name | string | `"default-receiver"` |  |
| alertmanagerFiles."alertmanager.yml".route.group_interval | string | `"5m"` |  |
| alertmanagerFiles."alertmanager.yml".route.group_wait | string | `"10s"` |  |
| alertmanagerFiles."alertmanager.yml".route.receiver | string | `"default-receiver"` |  |
| alertmanagerFiles."alertmanager.yml".route.repeat_interval | string | `"3h"` |  |
| configmapReload.alertmanager.enabled | bool | `true` |  |
| configmapReload.alertmanager.extraArgs | object | `{}` |  |
| configmapReload.alertmanager.extraConfigmapMounts | list | `[]` |  |
| configmapReload.alertmanager.extraVolumeDirs | list | `[]` |  |
| configmapReload.alertmanager.image.pullPolicy | string | `"Always"` |  |
| configmapReload.alertmanager.image.repository | string | `"openziti/configmap-reloadz"` |  |
| configmapReload.alertmanager.image.tag | string | `"latest-amd64"` |  |
| configmapReload.alertmanager.name | string | `"configmap-reload"` |  |
| configmapReload.alertmanager.resources | object | `{}` |  |
| configmapReload.prometheus.enabled | bool | `true` |  |
| configmapReload.prometheus.extraArgs | object | `{}` |  |
| configmapReload.prometheus.extraConfigmapMounts | list | `[]` |  |
| configmapReload.prometheus.extraVolumeDirs | list | `[]` |  |
| configmapReload.prometheus.image.pullPolicy | string | `"Always"` |  |
| configmapReload.prometheus.image.repository | string | `"openziti/configmap-reloadz"` |  |
| configmapReload.prometheus.image.tag | string | `"latest-amd64"` |  |
| configmapReload.prometheus.name | string | `"configmap-reload"` |  |
| configmapReload.prometheus.resources | object | `{}` |  |
| configmapReload.webhookUrl | string | `"http://127.0.0.1:9090/-/reload"` |  |
| configmapReload.ziti.identityFile | string | `"/run/secrets/ziti.identity.json"` |  |
| extraScrapeConfigs | string | `nil` |  |
| forceNamespace | string | `nil` |  |
| imagePullSecrets | string | `nil` |  |
| kubeStateMetrics.enabled | bool | `true` |  |
| networkPolicy.enabled | bool | `false` |  |
| nodeExporter.dnsConfig | object | `{}` |  |
| nodeExporter.enabled | bool | `true` |  |
| nodeExporter.extraArgs | object | `{}` |  |
| nodeExporter.extraConfigmapMounts | list | `[]` |  |
| nodeExporter.extraHostPathMounts | list | `[]` |  |
| nodeExporter.extraInitContainers | list | `[]` |  |
| nodeExporter.hostNetwork | bool | `true` |  |
| nodeExporter.hostPID | bool | `true` |  |
| nodeExporter.hostRootfs | bool | `true` |  |
| nodeExporter.image.pullPolicy | string | `"IfNotPresent"` |  |
| nodeExporter.image.repository | string | `"quay.io/prometheus/node-exporter"` |  |
| nodeExporter.image.tag | string | `"v1.3.0"` |  |
| nodeExporter.name | string | `"node-exporter"` |  |
| nodeExporter.nodeSelector | object | `{}` |  |
| nodeExporter.pod.labels | object | `{}` |  |
| nodeExporter.podAnnotations | object | `{}` |  |
| nodeExporter.podDisruptionBudget.enabled | bool | `false` |  |
| nodeExporter.podDisruptionBudget.maxUnavailable | int | `1` |  |
| nodeExporter.podSecurityPolicy.annotations | object | `{}` |  |
| nodeExporter.priorityClassName | string | `""` |  |
| nodeExporter.resources | object | `{}` |  |
| nodeExporter.securityContext.fsGroup | int | `65534` |  |
| nodeExporter.securityContext.runAsGroup | int | `65534` |  |
| nodeExporter.securityContext.runAsNonRoot | bool | `true` |  |
| nodeExporter.securityContext.runAsUser | int | `65534` |  |
| nodeExporter.service.annotations."prometheus.io/scrape" | string | `"true"` |  |
| nodeExporter.service.clusterIP | string | `"None"` |  |
| nodeExporter.service.externalIPs | list | `[]` |  |
| nodeExporter.service.hostPort | int | `9100` |  |
| nodeExporter.service.labels | object | `{}` |  |
| nodeExporter.service.loadBalancerIP | string | `""` |  |
| nodeExporter.service.loadBalancerSourceRanges | list | `[]` |  |
| nodeExporter.service.servicePort | int | `9100` |  |
| nodeExporter.service.type | string | `"ClusterIP"` |  |
| nodeExporter.tolerations | list | `[]` |  |
| nodeExporter.updateStrategy.type | string | `"RollingUpdate"` |  |
| podSecurityPolicy.enabled | bool | `false` |  |
| pushgateway.deploymentAnnotations | object | `{}` |  |
| pushgateway.dnsConfig | object | `{}` |  |
| pushgateway.enabled | bool | `true` |  |
| pushgateway.extraArgs | object | `{}` |  |
| pushgateway.extraInitContainers | list | `[]` |  |
| pushgateway.image.pullPolicy | string | `"IfNotPresent"` |  |
| pushgateway.image.repository | string | `"prom/pushgateway"` |  |
| pushgateway.image.tag | string | `"v1.4.2"` |  |
| pushgateway.ingress.annotations | object | `{}` |  |
| pushgateway.ingress.enabled | bool | `false` |  |
| pushgateway.ingress.extraPaths | list | `[]` |  |
| pushgateway.ingress.hosts | list | `[]` |  |
| pushgateway.ingress.path | string | `"/"` |  |
| pushgateway.ingress.pathType | string | `"Prefix"` |  |
| pushgateway.ingress.tls | list | `[]` |  |
| pushgateway.name | string | `"pushgateway"` |  |
| pushgateway.nodeSelector | object | `{}` |  |
| pushgateway.persistentVolume.accessModes[0] | string | `"ReadWriteOnce"` |  |
| pushgateway.persistentVolume.annotations | object | `{}` |  |
| pushgateway.persistentVolume.enabled | bool | `false` |  |
| pushgateway.persistentVolume.existingClaim | string | `""` |  |
| pushgateway.persistentVolume.mountPath | string | `"/data"` |  |
| pushgateway.persistentVolume.size | string | `"2Gi"` |  |
| pushgateway.persistentVolume.subPath | string | `""` |  |
| pushgateway.podAnnotations | object | `{}` |  |
| pushgateway.podDisruptionBudget.enabled | bool | `false` |  |
| pushgateway.podDisruptionBudget.maxUnavailable | int | `1` |  |
| pushgateway.podLabels | object | `{}` |  |
| pushgateway.podSecurityPolicy.annotations | object | `{}` |  |
| pushgateway.priorityClassName | string | `""` |  |
| pushgateway.replicaCount | int | `1` |  |
| pushgateway.resources | object | `{}` |  |
| pushgateway.securityContext.runAsNonRoot | bool | `true` |  |
| pushgateway.securityContext.runAsUser | int | `65534` |  |
| pushgateway.service.annotations."prometheus.io/probe" | string | `"pushgateway"` |  |
| pushgateway.service.clusterIP | string | `""` |  |
| pushgateway.service.externalIPs | list | `[]` |  |
| pushgateway.service.labels | object | `{}` |  |
| pushgateway.service.loadBalancerIP | string | `""` |  |
| pushgateway.service.loadBalancerSourceRanges | list | `[]` |  |
| pushgateway.service.servicePort | int | `9091` |  |
| pushgateway.service.type | string | `"ClusterIP"` |  |
| pushgateway.tolerations | list | `[]` |  |
| pushgateway.verticalAutoscaler.enabled | bool | `false` |  |
| rbac.create | bool | `true` |  |
| server.affinity | object | `{}` |  |
| server.alertmanagers | list | `[]` |  |
| server.baseURL | string | `""` |  |
| server.configMapOverrideName | string | `""` |  |
| server.configPath | string | `"/etc/config/prometheus.yml"` |  |
| server.deploymentAnnotations | object | `{}` |  |
| server.dnsConfig | object | `{}` |  |
| server.dnsPolicy | string | `"ClusterFirst"` |  |
| server.emptyDir.sizeLimit | string | `""` |  |
| server.enableServiceLinks | bool | `true` |  |
| server.enabled | bool | `true` |  |
| server.env | list | `[]` |  |
| server.extraArgs | object | `{}` |  |
| server.extraConfigmapMounts | list | `[]` |  |
| server.extraFlags[0] | string | `"web.enable-lifecycle"` |  |
| server.extraHostPathMounts | list | `[]` |  |
| server.extraInitContainers | list | `[]` |  |
| server.extraSecretMounts | list | `[]` |  |
| server.extraVolumeMounts | list | `[]` |  |
| server.extraVolumes | list | `[]` |  |
| server.global.evaluation_interval | string | `"1m"` |  |
| server.global.scrape_interval | string | `"1m"` |  |
| server.global.scrape_timeout | string | `"10s"` |  |
| server.hostAliases | list | `[]` |  |
| server.hostNetwork | bool | `false` |  |
| server.image.pullPolicy | string | `"IfNotPresent"` |  |
| server.image.repository | string | `"openziti/prometheuz"` |  |
| server.image.tag | string | `"0.0.1"` |  |
| server.ingress.annotations | object | `{}` |  |
| server.ingress.enabled | bool | `false` |  |
| server.ingress.extraLabels | object | `{}` |  |
| server.ingress.extraPaths | list | `[]` |  |
| server.ingress.hosts | list | `[]` |  |
| server.ingress.path | string | `"/"` |  |
| server.ingress.pathType | string | `"Prefix"` |  |
| server.ingress.tls | list | `[]` |  |
| server.livenessProbeFailureThreshold | int | `3` |  |
| server.livenessProbeInitialDelay | int | `30` |  |
| server.livenessProbePeriodSeconds | int | `15` |  |
| server.livenessProbeSuccessThreshold | int | `1` |  |
| server.livenessProbeTimeout | int | `10` |  |
| server.name | string | `"server"` |  |
| server.nodeSelector | object | `{}` |  |
| server.persistentVolume.accessModes[0] | string | `"ReadWriteOnce"` |  |
| server.persistentVolume.annotations | object | `{}` |  |
| server.persistentVolume.enabled | bool | `true` |  |
| server.persistentVolume.existingClaim | string | `""` |  |
| server.persistentVolume.mountPath | string | `"/data"` |  |
| server.persistentVolume.size | string | `"8Gi"` |  |
| server.persistentVolume.subPath | string | `""` |  |
| server.podAnnotations | object | `{}` |  |
| server.podDisruptionBudget.enabled | bool | `false` |  |
| server.podDisruptionBudget.maxUnavailable | int | `1` |  |
| server.podLabels | object | `{}` |  |
| server.podSecurityPolicy.annotations | object | `{}` |  |
| server.prefixURL | string | `""` |  |
| server.priorityClassName | string | `""` |  |
| server.probeHeaders | list | `[]` |  |
| server.probeScheme | string | `"HTTP"` |  |
| server.readinessProbe.enabled | bool | `false` |  |
| server.readinessProbeFailureThreshold | int | `3` |  |
| server.readinessProbeInitialDelay | int | `30` |  |
| server.readinessProbePeriodSeconds | int | `5` |  |
| server.readinessProbeSuccessThreshold | int | `1` |  |
| server.readinessProbeTimeout | int | `4` |  |
| server.remoteRead | list | `[]` |  |
| server.remoteWrite | list | `[]` |  |
| server.replicaCount | int | `1` |  |
| server.resources | object | `{}` |  |
| server.retention | string | `"15d"` |  |
| server.securityContext.fsGroup | int | `65534` |  |
| server.securityContext.runAsGroup | int | `65534` |  |
| server.securityContext.runAsNonRoot | bool | `true` |  |
| server.securityContext.runAsUser | int | `65534` |  |
| server.service.annotations | object | `{}` |  |
| server.service.clusterIP | string | `""` |  |
| server.service.externalIPs | list | `[]` |  |
| server.service.gRPC.enabled | bool | `false` |  |
| server.service.gRPC.servicePort | int | `10901` |  |
| server.service.labels | object | `{}` |  |
| server.service.loadBalancerIP | string | `""` |  |
| server.service.loadBalancerSourceRanges | list | `[]` |  |
| server.service.servicePort | int | `80` |  |
| server.service.sessionAffinity | string | `"None"` |  |
| server.service.statefulsetReplica.enabled | bool | `false` |  |
| server.service.statefulsetReplica.replica | int | `0` |  |
| server.service.type | string | `"ClusterIP"` |  |
| server.sidecarContainers | list | `[]` |  |
| server.sidecarTemplateValues | object | `{}` |  |
| server.startupProbe.enabled | bool | `false` |  |
| server.startupProbe.failureThreshold | int | `30` |  |
| server.startupProbe.periodSeconds | int | `5` |  |
| server.startupProbe.timeoutSeconds | int | `10` |  |
| server.statefulSet.annotations | object | `{}` |  |
| server.statefulSet.enabled | bool | `false` |  |
| server.statefulSet.headless.annotations | object | `{}` |  |
| server.statefulSet.headless.gRPC.enabled | bool | `false` |  |
| server.statefulSet.headless.gRPC.servicePort | int | `10901` |  |
| server.statefulSet.headless.labels | object | `{}` |  |
| server.statefulSet.headless.servicePort | int | `80` |  |
| server.statefulSet.labels | object | `{}` |  |
| server.statefulSet.podManagementPolicy | string | `"OrderedReady"` |  |
| server.storagePath | string | `""` |  |
| server.tcpSocketProbeEnabled | bool | `false` |  |
| server.terminationGracePeriodSeconds | int | `300` |  |
| server.tolerations | list | `[]` |  |
| server.verticalAutoscaler.enabled | bool | `false` |  |
| server.ziti.enabled | bool | `true` |  |
| server.ziti.identity | string | `"prometheus"` |  |
| server.ziti.path | string | `"/etc/prometheus/prometheus.json"` |  |
| server.ziti.service | string | `"prometheus.svc"` |  |
| serverFiles."alerting_rules.yml" | object | `{}` |  |
| serverFiles."prometheus.yml".rule_files[0] | string | `"/etc/config/recording_rules.yml"` |  |
| serverFiles."prometheus.yml".rule_files[1] | string | `"/etc/config/alerting_rules.yml"` |  |
| serverFiles."prometheus.yml".rule_files[2] | string | `"/etc/config/rules"` |  |
| serverFiles."prometheus.yml".rule_files[3] | string | `"/etc/config/alerts"` |  |
| serverFiles."prometheus.yml".scrape_configs[0].bearer_token_file | string | `"/var/run/secrets/kubernetes.io/serviceaccount/token"` |  |
| serverFiles."prometheus.yml".scrape_configs[0].job_name | string | `"kubernetes-apiservers"` |  |
| serverFiles."prometheus.yml".scrape_configs[0].kubernetes_sd_configs[0].role | string | `"endpoints"` |  |
| serverFiles."prometheus.yml".scrape_configs[0].relabel_configs[0].action | string | `"keep"` |  |
| serverFiles."prometheus.yml".scrape_configs[0].relabel_configs[0].regex | string | `"default;kubernetes;https"` |  |
| serverFiles."prometheus.yml".scrape_configs[0].relabel_configs[0].source_labels[0] | string | `"__meta_kubernetes_namespace"` |  |
| serverFiles."prometheus.yml".scrape_configs[0].relabel_configs[0].source_labels[1] | string | `"__meta_kubernetes_service_name"` |  |
| serverFiles."prometheus.yml".scrape_configs[0].relabel_configs[0].source_labels[2] | string | `"__meta_kubernetes_endpoint_port_name"` |  |
| serverFiles."prometheus.yml".scrape_configs[0].scheme | string | `"https"` |  |
| serverFiles."prometheus.yml".scrape_configs[0].tls_config.ca_file | string | `"/var/run/secrets/kubernetes.io/serviceaccount/ca.crt"` |  |
| serverFiles."prometheus.yml".scrape_configs[0].tls_config.insecure_skip_verify | bool | `true` |  |
| serverFiles."prometheus.yml".scrape_configs[1].bearer_token_file | string | `"/var/run/secrets/kubernetes.io/serviceaccount/token"` |  |
| serverFiles."prometheus.yml".scrape_configs[1].job_name | string | `"kubernetes-nodes"` |  |
| serverFiles."prometheus.yml".scrape_configs[1].kubernetes_sd_configs[0].role | string | `"node"` |  |
| serverFiles."prometheus.yml".scrape_configs[1].relabel_configs[0].action | string | `"labelmap"` |  |
| serverFiles."prometheus.yml".scrape_configs[1].relabel_configs[0].regex | string | `"__meta_kubernetes_node_label_(.+)"` |  |
| serverFiles."prometheus.yml".scrape_configs[1].relabel_configs[1].replacement | string | `"kubernetes.default.svc:443"` |  |
| serverFiles."prometheus.yml".scrape_configs[1].relabel_configs[1].target_label | string | `"__address__"` |  |
| serverFiles."prometheus.yml".scrape_configs[1].relabel_configs[2].regex | string | `"(.+)"` |  |
| serverFiles."prometheus.yml".scrape_configs[1].relabel_configs[2].replacement | string | `"/api/v1/nodes/$1/proxy/metrics"` |  |
| serverFiles."prometheus.yml".scrape_configs[1].relabel_configs[2].source_labels[0] | string | `"__meta_kubernetes_node_name"` |  |
| serverFiles."prometheus.yml".scrape_configs[1].relabel_configs[2].target_label | string | `"__metrics_path__"` |  |
| serverFiles."prometheus.yml".scrape_configs[1].scheme | string | `"https"` |  |
| serverFiles."prometheus.yml".scrape_configs[1].tls_config.ca_file | string | `"/var/run/secrets/kubernetes.io/serviceaccount/ca.crt"` |  |
| serverFiles."prometheus.yml".scrape_configs[1].tls_config.insecure_skip_verify | bool | `true` |  |
| serverFiles."prometheus.yml".scrape_configs[2].bearer_token_file | string | `"/var/run/secrets/kubernetes.io/serviceaccount/token"` |  |
| serverFiles."prometheus.yml".scrape_configs[2].job_name | string | `"kubernetes-nodes-cadvisor"` |  |
| serverFiles."prometheus.yml".scrape_configs[2].kubernetes_sd_configs[0].role | string | `"node"` |  |
| serverFiles."prometheus.yml".scrape_configs[2].relabel_configs[0].action | string | `"labelmap"` |  |
| serverFiles."prometheus.yml".scrape_configs[2].relabel_configs[0].regex | string | `"__meta_kubernetes_node_label_(.+)"` |  |
| serverFiles."prometheus.yml".scrape_configs[2].relabel_configs[1].replacement | string | `"kubernetes.default.svc:443"` |  |
| serverFiles."prometheus.yml".scrape_configs[2].relabel_configs[1].target_label | string | `"__address__"` |  |
| serverFiles."prometheus.yml".scrape_configs[2].relabel_configs[2].regex | string | `"(.+)"` |  |
| serverFiles."prometheus.yml".scrape_configs[2].relabel_configs[2].replacement | string | `"/api/v1/nodes/$1/proxy/metrics/cadvisor"` |  |
| serverFiles."prometheus.yml".scrape_configs[2].relabel_configs[2].source_labels[0] | string | `"__meta_kubernetes_node_name"` |  |
| serverFiles."prometheus.yml".scrape_configs[2].relabel_configs[2].target_label | string | `"__metrics_path__"` |  |
| serverFiles."prometheus.yml".scrape_configs[2].scheme | string | `"https"` |  |
| serverFiles."prometheus.yml".scrape_configs[2].tls_config.ca_file | string | `"/var/run/secrets/kubernetes.io/serviceaccount/ca.crt"` |  |
| serverFiles."prometheus.yml".scrape_configs[2].tls_config.insecure_skip_verify | bool | `true` |  |
| serverFiles."prometheus.yml".scrape_configs[3].job_name | string | `"kubernetes-service-endpoints"` |  |
| serverFiles."prometheus.yml".scrape_configs[3].kubernetes_sd_configs[0].role | string | `"endpoints"` |  |
| serverFiles."prometheus.yml".scrape_configs[3].relabel_configs[0].action | string | `"keep"` |  |
| serverFiles."prometheus.yml".scrape_configs[3].relabel_configs[0].regex | bool | `true` |  |
| serverFiles."prometheus.yml".scrape_configs[3].relabel_configs[0].source_labels[0] | string | `"__meta_kubernetes_service_annotation_prometheus_io_scrape"` |  |
| serverFiles."prometheus.yml".scrape_configs[3].relabel_configs[1].action | string | `"replace"` |  |
| serverFiles."prometheus.yml".scrape_configs[3].relabel_configs[1].regex | string | `"(https?)"` |  |
| serverFiles."prometheus.yml".scrape_configs[3].relabel_configs[1].source_labels[0] | string | `"__meta_kubernetes_service_annotation_prometheus_io_scheme"` |  |
| serverFiles."prometheus.yml".scrape_configs[3].relabel_configs[1].target_label | string | `"__scheme__"` |  |
| serverFiles."prometheus.yml".scrape_configs[3].relabel_configs[2].action | string | `"replace"` |  |
| serverFiles."prometheus.yml".scrape_configs[3].relabel_configs[2].regex | string | `"(.+)"` |  |
| serverFiles."prometheus.yml".scrape_configs[3].relabel_configs[2].source_labels[0] | string | `"__meta_kubernetes_service_annotation_prometheus_io_path"` |  |
| serverFiles."prometheus.yml".scrape_configs[3].relabel_configs[2].target_label | string | `"__metrics_path__"` |  |
| serverFiles."prometheus.yml".scrape_configs[3].relabel_configs[3].action | string | `"replace"` |  |
| serverFiles."prometheus.yml".scrape_configs[3].relabel_configs[3].regex | string | `"([^:]+)(?::\\d+)?;(\\d+)"` |  |
| serverFiles."prometheus.yml".scrape_configs[3].relabel_configs[3].replacement | string | `"$1:$2"` |  |
| serverFiles."prometheus.yml".scrape_configs[3].relabel_configs[3].source_labels[0] | string | `"__address__"` |  |
| serverFiles."prometheus.yml".scrape_configs[3].relabel_configs[3].source_labels[1] | string | `"__meta_kubernetes_service_annotation_prometheus_io_port"` |  |
| serverFiles."prometheus.yml".scrape_configs[3].relabel_configs[3].target_label | string | `"__address__"` |  |
| serverFiles."prometheus.yml".scrape_configs[3].relabel_configs[4].action | string | `"labelmap"` |  |
| serverFiles."prometheus.yml".scrape_configs[3].relabel_configs[4].regex | string | `"__meta_kubernetes_service_annotation_prometheus_io_param_(.+)"` |  |
| serverFiles."prometheus.yml".scrape_configs[3].relabel_configs[4].replacement | string | `"__param_$1"` |  |
| serverFiles."prometheus.yml".scrape_configs[3].relabel_configs[5].action | string | `"labelmap"` |  |
| serverFiles."prometheus.yml".scrape_configs[3].relabel_configs[5].regex | string | `"__meta_kubernetes_service_label_(.+)"` |  |
| serverFiles."prometheus.yml".scrape_configs[3].relabel_configs[6].action | string | `"replace"` |  |
| serverFiles."prometheus.yml".scrape_configs[3].relabel_configs[6].source_labels[0] | string | `"__meta_kubernetes_namespace"` |  |
| serverFiles."prometheus.yml".scrape_configs[3].relabel_configs[6].target_label | string | `"namespace"` |  |
| serverFiles."prometheus.yml".scrape_configs[3].relabel_configs[7].action | string | `"replace"` |  |
| serverFiles."prometheus.yml".scrape_configs[3].relabel_configs[7].source_labels[0] | string | `"__meta_kubernetes_service_name"` |  |
| serverFiles."prometheus.yml".scrape_configs[3].relabel_configs[7].target_label | string | `"service"` |  |
| serverFiles."prometheus.yml".scrape_configs[3].relabel_configs[8].action | string | `"replace"` |  |
| serverFiles."prometheus.yml".scrape_configs[3].relabel_configs[8].source_labels[0] | string | `"__meta_kubernetes_pod_node_name"` |  |
| serverFiles."prometheus.yml".scrape_configs[3].relabel_configs[8].target_label | string | `"node"` |  |
| serverFiles."prometheus.yml".scrape_configs[4].job_name | string | `"kubernetes-service-endpoints-slow"` |  |
| serverFiles."prometheus.yml".scrape_configs[4].kubernetes_sd_configs[0].role | string | `"endpoints"` |  |
| serverFiles."prometheus.yml".scrape_configs[4].relabel_configs[0].action | string | `"keep"` |  |
| serverFiles."prometheus.yml".scrape_configs[4].relabel_configs[0].regex | bool | `true` |  |
| serverFiles."prometheus.yml".scrape_configs[4].relabel_configs[0].source_labels[0] | string | `"__meta_kubernetes_service_annotation_prometheus_io_scrape_slow"` |  |
| serverFiles."prometheus.yml".scrape_configs[4].relabel_configs[1].action | string | `"replace"` |  |
| serverFiles."prometheus.yml".scrape_configs[4].relabel_configs[1].regex | string | `"(https?)"` |  |
| serverFiles."prometheus.yml".scrape_configs[4].relabel_configs[1].source_labels[0] | string | `"__meta_kubernetes_service_annotation_prometheus_io_scheme"` |  |
| serverFiles."prometheus.yml".scrape_configs[4].relabel_configs[1].target_label | string | `"__scheme__"` |  |
| serverFiles."prometheus.yml".scrape_configs[4].relabel_configs[2].action | string | `"replace"` |  |
| serverFiles."prometheus.yml".scrape_configs[4].relabel_configs[2].regex | string | `"(.+)"` |  |
| serverFiles."prometheus.yml".scrape_configs[4].relabel_configs[2].source_labels[0] | string | `"__meta_kubernetes_service_annotation_prometheus_io_path"` |  |
| serverFiles."prometheus.yml".scrape_configs[4].relabel_configs[2].target_label | string | `"__metrics_path__"` |  |
| serverFiles."prometheus.yml".scrape_configs[4].relabel_configs[3].action | string | `"replace"` |  |
| serverFiles."prometheus.yml".scrape_configs[4].relabel_configs[3].regex | string | `"([^:]+)(?::\\d+)?;(\\d+)"` |  |
| serverFiles."prometheus.yml".scrape_configs[4].relabel_configs[3].replacement | string | `"$1:$2"` |  |
| serverFiles."prometheus.yml".scrape_configs[4].relabel_configs[3].source_labels[0] | string | `"__address__"` |  |
| serverFiles."prometheus.yml".scrape_configs[4].relabel_configs[3].source_labels[1] | string | `"__meta_kubernetes_service_annotation_prometheus_io_port"` |  |
| serverFiles."prometheus.yml".scrape_configs[4].relabel_configs[3].target_label | string | `"__address__"` |  |
| serverFiles."prometheus.yml".scrape_configs[4].relabel_configs[4].action | string | `"labelmap"` |  |
| serverFiles."prometheus.yml".scrape_configs[4].relabel_configs[4].regex | string | `"__meta_kubernetes_service_annotation_prometheus_io_param_(.+)"` |  |
| serverFiles."prometheus.yml".scrape_configs[4].relabel_configs[4].replacement | string | `"__param_$1"` |  |
| serverFiles."prometheus.yml".scrape_configs[4].relabel_configs[5].action | string | `"labelmap"` |  |
| serverFiles."prometheus.yml".scrape_configs[4].relabel_configs[5].regex | string | `"__meta_kubernetes_service_label_(.+)"` |  |
| serverFiles."prometheus.yml".scrape_configs[4].relabel_configs[6].action | string | `"replace"` |  |
| serverFiles."prometheus.yml".scrape_configs[4].relabel_configs[6].source_labels[0] | string | `"__meta_kubernetes_namespace"` |  |
| serverFiles."prometheus.yml".scrape_configs[4].relabel_configs[6].target_label | string | `"namespace"` |  |
| serverFiles."prometheus.yml".scrape_configs[4].relabel_configs[7].action | string | `"replace"` |  |
| serverFiles."prometheus.yml".scrape_configs[4].relabel_configs[7].source_labels[0] | string | `"__meta_kubernetes_service_name"` |  |
| serverFiles."prometheus.yml".scrape_configs[4].relabel_configs[7].target_label | string | `"service"` |  |
| serverFiles."prometheus.yml".scrape_configs[4].relabel_configs[8].action | string | `"replace"` |  |
| serverFiles."prometheus.yml".scrape_configs[4].relabel_configs[8].source_labels[0] | string | `"__meta_kubernetes_pod_node_name"` |  |
| serverFiles."prometheus.yml".scrape_configs[4].relabel_configs[8].target_label | string | `"node"` |  |
| serverFiles."prometheus.yml".scrape_configs[4].scrape_interval | string | `"5m"` |  |
| serverFiles."prometheus.yml".scrape_configs[4].scrape_timeout | string | `"30s"` |  |
| serverFiles."prometheus.yml".scrape_configs[5].honor_labels | bool | `true` |  |
| serverFiles."prometheus.yml".scrape_configs[5].job_name | string | `"prometheus-pushgateway"` |  |
| serverFiles."prometheus.yml".scrape_configs[5].kubernetes_sd_configs[0].role | string | `"service"` |  |
| serverFiles."prometheus.yml".scrape_configs[5].relabel_configs[0].action | string | `"keep"` |  |
| serverFiles."prometheus.yml".scrape_configs[5].relabel_configs[0].regex | string | `"pushgateway"` |  |
| serverFiles."prometheus.yml".scrape_configs[5].relabel_configs[0].source_labels[0] | string | `"__meta_kubernetes_service_annotation_prometheus_io_probe"` |  |
| serverFiles."prometheus.yml".scrape_configs[6].job_name | string | `"kubernetes-services"` |  |
| serverFiles."prometheus.yml".scrape_configs[6].kubernetes_sd_configs[0].role | string | `"service"` |  |
| serverFiles."prometheus.yml".scrape_configs[6].metrics_path | string | `"/probe"` |  |
| serverFiles."prometheus.yml".scrape_configs[6].params.module[0] | string | `"http_2xx"` |  |
| serverFiles."prometheus.yml".scrape_configs[6].relabel_configs[0].action | string | `"keep"` |  |
| serverFiles."prometheus.yml".scrape_configs[6].relabel_configs[0].regex | bool | `true` |  |
| serverFiles."prometheus.yml".scrape_configs[6].relabel_configs[0].source_labels[0] | string | `"__meta_kubernetes_service_annotation_prometheus_io_probe"` |  |
| serverFiles."prometheus.yml".scrape_configs[6].relabel_configs[1].source_labels[0] | string | `"__address__"` |  |
| serverFiles."prometheus.yml".scrape_configs[6].relabel_configs[1].target_label | string | `"__param_target"` |  |
| serverFiles."prometheus.yml".scrape_configs[6].relabel_configs[2].replacement | string | `"blackbox"` |  |
| serverFiles."prometheus.yml".scrape_configs[6].relabel_configs[2].target_label | string | `"__address__"` |  |
| serverFiles."prometheus.yml".scrape_configs[6].relabel_configs[3].source_labels[0] | string | `"__param_target"` |  |
| serverFiles."prometheus.yml".scrape_configs[6].relabel_configs[3].target_label | string | `"instance"` |  |
| serverFiles."prometheus.yml".scrape_configs[6].relabel_configs[4].action | string | `"labelmap"` |  |
| serverFiles."prometheus.yml".scrape_configs[6].relabel_configs[4].regex | string | `"__meta_kubernetes_service_label_(.+)"` |  |
| serverFiles."prometheus.yml".scrape_configs[6].relabel_configs[5].source_labels[0] | string | `"__meta_kubernetes_namespace"` |  |
| serverFiles."prometheus.yml".scrape_configs[6].relabel_configs[5].target_label | string | `"namespace"` |  |
| serverFiles."prometheus.yml".scrape_configs[6].relabel_configs[6].source_labels[0] | string | `"__meta_kubernetes_service_name"` |  |
| serverFiles."prometheus.yml".scrape_configs[6].relabel_configs[6].target_label | string | `"service"` |  |
| serverFiles."prometheus.yml".scrape_configs[7].job_name | string | `"kubernetes-pods"` |  |
| serverFiles."prometheus.yml".scrape_configs[7].kubernetes_sd_configs[0].role | string | `"pod"` |  |
| serverFiles."prometheus.yml".scrape_configs[7].relabel_configs[0].action | string | `"keep"` |  |
| serverFiles."prometheus.yml".scrape_configs[7].relabel_configs[0].regex | bool | `true` |  |
| serverFiles."prometheus.yml".scrape_configs[7].relabel_configs[0].source_labels[0] | string | `"__meta_kubernetes_pod_annotation_prometheus_io_scrape"` |  |
| serverFiles."prometheus.yml".scrape_configs[7].relabel_configs[1].action | string | `"replace"` |  |
| serverFiles."prometheus.yml".scrape_configs[7].relabel_configs[1].regex | string | `"(https?)"` |  |
| serverFiles."prometheus.yml".scrape_configs[7].relabel_configs[1].source_labels[0] | string | `"__meta_kubernetes_pod_annotation_prometheus_io_scheme"` |  |
| serverFiles."prometheus.yml".scrape_configs[7].relabel_configs[1].target_label | string | `"__scheme__"` |  |
| serverFiles."prometheus.yml".scrape_configs[7].relabel_configs[2].action | string | `"replace"` |  |
| serverFiles."prometheus.yml".scrape_configs[7].relabel_configs[2].regex | string | `"(.+)"` |  |
| serverFiles."prometheus.yml".scrape_configs[7].relabel_configs[2].source_labels[0] | string | `"__meta_kubernetes_pod_annotation_prometheus_io_path"` |  |
| serverFiles."prometheus.yml".scrape_configs[7].relabel_configs[2].target_label | string | `"__metrics_path__"` |  |
| serverFiles."prometheus.yml".scrape_configs[7].relabel_configs[3].action | string | `"replace"` |  |
| serverFiles."prometheus.yml".scrape_configs[7].relabel_configs[3].regex | string | `"([^:]+)(?::\\d+)?;(\\d+)"` |  |
| serverFiles."prometheus.yml".scrape_configs[7].relabel_configs[3].replacement | string | `"$1:$2"` |  |
| serverFiles."prometheus.yml".scrape_configs[7].relabel_configs[3].source_labels[0] | string | `"__address__"` |  |
| serverFiles."prometheus.yml".scrape_configs[7].relabel_configs[3].source_labels[1] | string | `"__meta_kubernetes_pod_annotation_prometheus_io_port"` |  |
| serverFiles."prometheus.yml".scrape_configs[7].relabel_configs[3].target_label | string | `"__address__"` |  |
| serverFiles."prometheus.yml".scrape_configs[7].relabel_configs[4].action | string | `"labelmap"` |  |
| serverFiles."prometheus.yml".scrape_configs[7].relabel_configs[4].regex | string | `"__meta_kubernetes_pod_annotation_prometheus_io_param_(.+)"` |  |
| serverFiles."prometheus.yml".scrape_configs[7].relabel_configs[4].replacement | string | `"__param_$1"` |  |
| serverFiles."prometheus.yml".scrape_configs[7].relabel_configs[5].action | string | `"labelmap"` |  |
| serverFiles."prometheus.yml".scrape_configs[7].relabel_configs[5].regex | string | `"__meta_kubernetes_pod_label_(.+)"` |  |
| serverFiles."prometheus.yml".scrape_configs[7].relabel_configs[6].action | string | `"replace"` |  |
| serverFiles."prometheus.yml".scrape_configs[7].relabel_configs[6].source_labels[0] | string | `"__meta_kubernetes_namespace"` |  |
| serverFiles."prometheus.yml".scrape_configs[7].relabel_configs[6].target_label | string | `"namespace"` |  |
| serverFiles."prometheus.yml".scrape_configs[7].relabel_configs[7].action | string | `"replace"` |  |
| serverFiles."prometheus.yml".scrape_configs[7].relabel_configs[7].source_labels[0] | string | `"__meta_kubernetes_pod_name"` |  |
| serverFiles."prometheus.yml".scrape_configs[7].relabel_configs[7].target_label | string | `"pod"` |  |
| serverFiles."prometheus.yml".scrape_configs[7].relabel_configs[8].action | string | `"drop"` |  |
| serverFiles."prometheus.yml".scrape_configs[7].relabel_configs[8].regex | string | `"Pending|Succeeded|Failed|Completed"` |  |
| serverFiles."prometheus.yml".scrape_configs[7].relabel_configs[8].source_labels[0] | string | `"__meta_kubernetes_pod_phase"` |  |
| serverFiles."prometheus.yml".scrape_configs[8].job_name | string | `"kubernetes-pods-slow"` |  |
| serverFiles."prometheus.yml".scrape_configs[8].kubernetes_sd_configs[0].role | string | `"pod"` |  |
| serverFiles."prometheus.yml".scrape_configs[8].relabel_configs[0].action | string | `"keep"` |  |
| serverFiles."prometheus.yml".scrape_configs[8].relabel_configs[0].regex | bool | `true` |  |
| serverFiles."prometheus.yml".scrape_configs[8].relabel_configs[0].source_labels[0] | string | `"__meta_kubernetes_pod_annotation_prometheus_io_scrape_slow"` |  |
| serverFiles."prometheus.yml".scrape_configs[8].relabel_configs[1].action | string | `"replace"` |  |
| serverFiles."prometheus.yml".scrape_configs[8].relabel_configs[1].regex | string | `"(https?)"` |  |
| serverFiles."prometheus.yml".scrape_configs[8].relabel_configs[1].source_labels[0] | string | `"__meta_kubernetes_pod_annotation_prometheus_io_scheme"` |  |
| serverFiles."prometheus.yml".scrape_configs[8].relabel_configs[1].target_label | string | `"__scheme__"` |  |
| serverFiles."prometheus.yml".scrape_configs[8].relabel_configs[2].action | string | `"replace"` |  |
| serverFiles."prometheus.yml".scrape_configs[8].relabel_configs[2].regex | string | `"(.+)"` |  |
| serverFiles."prometheus.yml".scrape_configs[8].relabel_configs[2].source_labels[0] | string | `"__meta_kubernetes_pod_annotation_prometheus_io_path"` |  |
| serverFiles."prometheus.yml".scrape_configs[8].relabel_configs[2].target_label | string | `"__metrics_path__"` |  |
| serverFiles."prometheus.yml".scrape_configs[8].relabel_configs[3].action | string | `"replace"` |  |
| serverFiles."prometheus.yml".scrape_configs[8].relabel_configs[3].regex | string | `"([^:]+)(?::\\d+)?;(\\d+)"` |  |
| serverFiles."prometheus.yml".scrape_configs[8].relabel_configs[3].replacement | string | `"$1:$2"` |  |
| serverFiles."prometheus.yml".scrape_configs[8].relabel_configs[3].source_labels[0] | string | `"__address__"` |  |
| serverFiles."prometheus.yml".scrape_configs[8].relabel_configs[3].source_labels[1] | string | `"__meta_kubernetes_pod_annotation_prometheus_io_port"` |  |
| serverFiles."prometheus.yml".scrape_configs[8].relabel_configs[3].target_label | string | `"__address__"` |  |
| serverFiles."prometheus.yml".scrape_configs[8].relabel_configs[4].action | string | `"labelmap"` |  |
| serverFiles."prometheus.yml".scrape_configs[8].relabel_configs[4].regex | string | `"__meta_kubernetes_pod_annotation_prometheus_io_param_(.+)"` |  |
| serverFiles."prometheus.yml".scrape_configs[8].relabel_configs[4].replacement | string | `"__param_$1"` |  |
| serverFiles."prometheus.yml".scrape_configs[8].relabel_configs[5].action | string | `"labelmap"` |  |
| serverFiles."prometheus.yml".scrape_configs[8].relabel_configs[5].regex | string | `"__meta_kubernetes_pod_label_(.+)"` |  |
| serverFiles."prometheus.yml".scrape_configs[8].relabel_configs[6].action | string | `"replace"` |  |
| serverFiles."prometheus.yml".scrape_configs[8].relabel_configs[6].source_labels[0] | string | `"__meta_kubernetes_namespace"` |  |
| serverFiles."prometheus.yml".scrape_configs[8].relabel_configs[6].target_label | string | `"namespace"` |  |
| serverFiles."prometheus.yml".scrape_configs[8].relabel_configs[7].action | string | `"replace"` |  |
| serverFiles."prometheus.yml".scrape_configs[8].relabel_configs[7].source_labels[0] | string | `"__meta_kubernetes_pod_name"` |  |
| serverFiles."prometheus.yml".scrape_configs[8].relabel_configs[7].target_label | string | `"pod"` |  |
| serverFiles."prometheus.yml".scrape_configs[8].relabel_configs[8].action | string | `"drop"` |  |
| serverFiles."prometheus.yml".scrape_configs[8].relabel_configs[8].regex | string | `"Pending|Succeeded|Failed|Completed"` |  |
| serverFiles."prometheus.yml".scrape_configs[8].relabel_configs[8].source_labels[0] | string | `"__meta_kubernetes_pod_phase"` |  |
| serverFiles."prometheus.yml".scrape_configs[8].scrape_interval | string | `"5m"` |  |
| serverFiles."prometheus.yml".scrape_configs[8].scrape_timeout | string | `"30s"` |  |
| serverFiles."recording_rules.yml" | object | `{}` |  |
| serverFiles.alerts | object | `{}` |  |
| serverFiles.rules | object | `{}` |  |
| serviceAccounts.alertmanager.annotations | object | `{}` |  |
| serviceAccounts.alertmanager.create | bool | `true` |  |
| serviceAccounts.alertmanager.name | string | `nil` |  |
| serviceAccounts.nodeExporter.annotations | object | `{}` |  |
| serviceAccounts.nodeExporter.create | bool | `true` |  |
| serviceAccounts.nodeExporter.name | string | `nil` |  |
| serviceAccounts.pushgateway.annotations | object | `{}` |  |
| serviceAccounts.pushgateway.create | bool | `true` |  |
| serviceAccounts.pushgateway.name | string | `nil` |  |
| serviceAccounts.server.annotations | object | `{}` |  |
| serviceAccounts.server.create | bool | `true` |  |
| serviceAccounts.server.name | string | `nil` |  |

<!-- generated with helm-docs -->
