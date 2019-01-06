# OpenShift Logging: Spring Boot Application multiline logs handler and custom log fields parser
There is a [long discussion about the missing support of OpenShift Logging](https://itnext.io/multiline-logs-in-openshift-efk-stack-7a7bda4ed055) (Elasticsearch-Fluentd-Kibana) of multiline logs. This is my appoach for handling multiline application logs , along with applying custom field parsing for Spring Boot custom application logs as a reference of how to handle you custom application logs in OpenShift Container Platform(tested with version 3.9 and 3.10).
This document is also [published at Medium](https://medium.com/@sbourazanis/openshift-logging-spring-boot-application-multiline-logs-handler-and-custom-log-fields-parser-e4e1a64cdc01).
## Introduction
OpenShift Fluentd configuration is exposed in logging-fluentd ConfigMap in openshift-logging project.The ConfigMap is populated with data from files mounted from the Fluentd pods image directory /etc/fluent/configs.d/user/. The main Fluentd configuration file in a Fluentd pod /etc/fluent/fluent.conf  is a symbolic link to /etc/fluent/configs.d/user/fluentd.conf. The idea is to add new Fluentd configuration file(s) as new ConfigMap entries and modify slightly the main fluentd.conf to apply the new pipeline. Furthermore to handle multiline logs, an additional Fluentd plugin is needed which is not part of the Fluent image, so the idea is to create a new ConfigMap for the plugins including not only the existing ones but also the additional plugin needed.

## Spring Boot logs
```
Spring Boot logs are Logback formatted as the following samples:
service1.log:2016-02-26 11:15:47.561  INFO [service1,2485ec27856c56f4,2485ec27856c56f4,true] 68058 --- [nio-8081-exec-1] i.s.c.sleuth.docs.service1.Application   : Hello from service1. Calling service2
service2.log:2016-02-26 11:15:47.710  INFO [service2,2485ec27856c56f4,9aa10ee6fbde75fa,true] 68059 --- [nio-8082-exec-1] i.s.c.sleuth.docs.service2.Application   : Hello from service2. Calling service3 and then service4
service3.log:2016-02-26 11:15:47.895  INFO [service3,2485ec27856c56f4,1210be13194bfe5,true] 68060 --- [nio-8083-exec-1] i.s.c.sleuth.docs.service3.Application   : Hello from service3
service2.log:2016-02-26 11:15:47.924  INFO [service2,2485ec27856c56f4,9aa10ee6fbde75fa,true] 68059 --- [nio-8082-exec-1] i.s.c.sleuth.docs.service2.Application   : Got response from service3 [Hello from service3]
service4.log:2016-02-26 11:15:48.134  INFO [service4,2485ec27856c56f4,1b1845262ffba49d,true] 68061 --- [nio-8084-exec-1] i.s.c.sleuth.docs.service4.Application   : Hello from service4
service2.log:2016-02-26 11:15:48.156  INFO [service2,2485ec27856c56f4,9aa10ee6fbde75fa,true] 68059 --- [nio-8082-exec-1] i.s.c.sleuth.docs.service2.Application   : Got response from service4 [Hello from service4]
service1.log:2016-02-26 11:15:48.182  INFO [service1,2485ec27856c56f4,2485ec27856c56f4,true] 68058 --- [nio-8081-exec-1] i.s.c.sleuth.docs.service1.Application   : Got response from service2 [Hello from service2, response from service3 [Hello from service3] and from service4 [Hello from service4]]
```
1.	Fields to be parsed are: thread, span, trace, service, severity, class
2.	Fluentd configuration should handle multi-line log entries (i.e. exceptions stack trace)
3.	Log Events should be sent to the same elastic search index

A [custom Perl application](https://github.com/SLionB/noise) has been created to generate sample logs of this type.


## How it works
Default OCP Fluentd pipeline reads container logs from journald source using Fluentd INGRESS labeled configuration and forwards them to OUTPUT labeled configuration which is the Elasticsearch backend like that: source->INGRESS->OUTPUT
To support multiline handling and custom field parsing, the main pipeline in fluent.conf has to be modified as follows:
source->INGRESS->MULTILINE->PARSER->OUTPUT.

1. The  new configuration file filter-post-springboot.conf in the INGRESS block  replaces the default filter-post-*.conf and forwards the logs to MULTILINE block instead of the OUTPUT block. Furtnermore it creates a docker_container_id field as the default nested one docker.container_id cannot be used by concat Fluentd filter.
2. The file out-multiline-springboot.conf in the MULTILINE block uses the concat filter to concatenate multiline logs based on a custom regular expression (you can use your own) in the contents of the message field grouped by the calculated docker_container_id field. Concatenated logs are forwarded to PARSER block for field parsing.
3. The file out-parser-springboot.conf in the PARSER block users a custom regular expression (you can use your own) in the parser filter to parse custom application fields and forwards them to OUTPUT block.

## Installing Custom Multiline Parsing
1. On an OpenShift master node login to OpenShift cluster using the CLI:
```
$ oc login <admin> <password>
```
2. Change to the logging project. Replace with your openshift logging project name if necessary:
```
$ oc project openshift-logging
```
3. Find the pods in your OpenShift cluster that run Fluentd agents:
```
$ oc get pods -l component=fluentd
NAME READY STATUS RESTARTS AGE
logging-fluentd-8j6c9 1/1 Running 0 1d
logging-fluentd-mbpk8 1/1 Running 0 1d
logging-fluentd-q5z7f 1/1 Running 0 1d
```
4. Copy existing Fluentd plugins from one of the Fluentd pods to the master node:
```
$ oc exec logging-fluentd-8j6c9 -- bash -c 'mkdir /extract; cp /etc/fluent/plugin/*.rb /extract;'
$ oc cp logging-fluentd-8j6c9:/extract /work/fluentd/plugin/
$ oc exec logging-fluentd-8j6c9 -- bash -c 'rm -r /extract;'
```
5. Copy existing Fluentd configuration files from one of the Fluentd pods to the master node:
```
$ oc exec logging-fluentd-8j6c9 -- bash -c 'mkdir /extract; cp /etc/fluent/configs.d/user/*.conf /extract; cp /etc/fluent/configs.d/user/*.yaml /extract;'
$ oc cp logging-fluentd-8j6c9:/extract /work/fluentd/configs.d/user/
$ oc exec logging-fluentd-8j6c9 -- bash -c 'rm -r /extract;'
```
6. Backup existing Fluentd plugins and configuration files to be available for the uninstallation process:
```
$ cp -R /work/fluentd/ /work/fluentd_backup
```
7. Download [Fluentd Concat Filter plugin](https://github.com/fluent-plugins-nursery/fluent-plugin-concat) to concatenate multiline logs separated in multiple events and copy the ruby file at the plugin folder. Please check your fluentd version to download the correct plugin version:
```
$ wget -O /work/fluentd/plugin/filter_concat.rb https://raw.githubusercontent.com/fluent-plugins-nursery/fluent-plugin-concat/v1.0.0/lib/fluent/plugin/filter_concat.rb
```
8. Download [Fluentd SpringBoot Custom Multiline Parser](https://github.com/SLionB/springboot-multiline-parser) zip file and unzip Fluentd configuration files at the configuration folder:
```
$ wget -O springboot-multiline-parser.zip https://github.com/SLionB/springboot-multiline-parser/archive/master.zip
$ unzip -oj springboot-multiline-parser.zip -d /work/fluentd/configs.d/user/
$ rm /work/fluentd/configs.d/user/README.md
```
9. Because original fluent.conf should have been overwritten, verify that the appropriate changes that modify the Fluentd pipeline with the new Fluentd configuration in/work/fluentd/configs.d/user/fluent.conf have been applied:
```
#This line should have beeen commented out
#@include configs.d/openshift/filter-post-*.conf
#The following 2 new lines should have been added just below the previous one
@include configs.d/openshift/filter-post-genid.conf
@include configs.d/user/filter-post-springboot.conf
```
10. Create a new ConfigMap for the Fluentd plugins:
```
$ oc create configmap logging-fluentd-plugin --from-file= /work/fluentd/plugin/
```
11. Replace the existing ConfigMap for the Fluentd configuration file with the updated one:
```
$ oc delete configmap logging-fluentd
$ oc create configmap logging-fluentd --from-file= /work/fluentd/configs.d/user/
```
12. On logging-fluentd DaemonSet add a new ConfigMap type Volume that mounts the Fluentd plugins folder:
```
$ oc volume ds/logging-fluentd --add --name=plugin --type=configmap --configmap-name=logging-fluentd-plugin --mount-path= /etc/fluent/plugin  --default-mode=0644
```
13. DaemonSet update will trigger a new Fluentd pods deployment with the new configuration. If everything has been deployed successfully the Fluentd pods will start without errors.
14. If you want to test this, in a non production environment, containarize in OpenShift this [sample Perl application (noise)](https://github.com/SLionB/noise) that generates sample Spring Boot custom application logs. 
14. Log in to Kibana and verify the presense of the added custom Spring Boot application's fields and that the multiline logs are handled successfully.
## Temporary enable/disable Custom Multiline Parsing
After installation you can easily go back and forth between the original logging configuration and the new one by just following the following trivial process:
1. Log in to OpenShift Console and change to openshift-logging project.
2. Select the logging-fluentd ConfigMap and click Edit.
3. Go to fluent.conf entry and modify the following part

to disable custom multiline parsing:
```
#This line should be commented in
@include configs.d/openshift/filter-post-*.conf
#The following 2 new lines should be commented out
#@include configs.d/openshift/filter-post-genid.conf
#@include configs.d/user/filter-post-springboot.conf
```
to enable custom multiline parsing:
```
#This line should be commented out
#@include configs.d/openshift/filter-post-*.conf
#The following 2 new lines should be added just below the previous one
@include configs.d/openshift/filter-post-genid.conf
@include configs.d/user/filter-post-springboot.conf
```
4. Delete all Fluentd pods:
```
$ oc label node -l logging-infra-fluentd=true --overwrite logging-infra-fluentd=false
```
5. Wait until all Fluentd pods are deleted:
```
$ oc get pods -l component=fluentd
```
6. Create all Fluentd pods again:
```
$ oc label node -l logging-infra-fluentd=false --overwrite logging-infra-fluentd=true
```
7. Wait until all Fluentd pods are created with the new configuration:
```
$ oc get pods -l component=fluentd
```

## Uninstalling Custom Multiline Parsing
If you have followed the backup process during installation, you can always restore back to the original configuration using the following procedure.
1. Delete Fluentd ConfigMap created for plugins:
```
$ oc delete configmap logging-fluentd-plugin
```
2. Recreate the original Fluentd ConfigMap with the original Fluentd configuration files:
```
$ oc delete configmap logging-fluentd
$ oc create configmap logging-fluentd --from-file= /work/fluentd_backup/configs.d/user/
```
3. Remove ConfigMap type Volume and the associated VolumeMount that mounts the Fluentd plugin folder from logging-fluentd DaemonSet:
```
$ oc volume ds/logging-fluentd --remove --name=plugin --type=configmap
```
4. DaemonSet update will trigger a new Fluentd pods deployment with the new configuration. If everything has been deployed successfully the pods will start without errors.
