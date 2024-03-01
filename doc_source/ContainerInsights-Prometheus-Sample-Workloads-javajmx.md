# Set Up Java/JMX sample workload on Amazon EKS and Kubernetes<a name="ContainerInsights-Prometheus-Sample-Workloads-javajmx"></a>

JMX Exporter is an official Prometheus exporter that can scrape and expose JMX mBeans as Prometheus metrics\. For more information, see [prometheus/jmx\_exporter](https://github.com/prometheus/jmx_exporter)\.

Container Insights can collect predefined Prometheus metrics from Java Virtual Machine \(JVM\), Java, and Tomcat \(Catalina\) using the JMX Exporter\.

## Default Prometheus Scrape Configuration<a name="ContainerInsights-Prometheus-Sample-Workloads-javajmx-default"></a>

By default, the CloudWatch agent with Prometheus support scrapes the Java/JMX Prometheus metrics from `http://CLUSTER_IP:9404/metrics` on each pod in an Amazon EKS or Kubernetes cluster\. This is done by `role: pod` discovery of Prometheus `kubernetes_sd_config`\. 9404 is the default port allocated for JMX Exporter by Prometheus\. For more information about `role: pod` discovery, see [ pod](https://prometheus.io/docs/prometheus/latest/configuration/configuration/#pod)\. You can configure the JMX Exporter to expose the metrics on a different port or metrics\_path\. If you do change the port or path, update the default jmx scrape\_config in the CloudWatch agent config map\. Run the following command to get the current CloudWatch agent Prometheus configuration:

```
kubectl describe cm prometheus-config -n amazon-cloudwatch
```

The fields to change are the `/metrics` and `regex: '.*:9404$'` fields, as highlighted in the following example\.

```
job_name: 'kubernetes-jmx-pod'
sample_limit: 10000
metrics_path: /metrics
kubernetes_sd_configs:
- role: pod
relabel_configs:
- source_labels: [__address__]
  action: keep
  regex: '.*:9404$'
- action: replace
  regex: (.+)
  source_labels:
```

## Other Prometheus Scrape Configuration<a name="ContainerInsights-Prometheus-Sample-Workloads-javajmx-other"></a>

If you expose your application running on a set of pods with Java/JMX Prometheus exporters by a Kubernetes Service, you can also switch to use `role: service` discovery or `role: endpoint` discovery of Prometheus `kubernetes_sd_config`\. For more information about these discovery methods, see [ service](https://prometheus.io/docs/prometheus/latest/configuration/configuration/#service), [ endpoints](https://prometheus.io/docs/prometheus/latest/configuration/configuration/#endpoints), and [ <kubernetes\_sd\_config>\.](https://prometheus.io/docs/prometheus/latest/configuration/configuration/#kubernetes_sd_config)\. 

More meta labels are provided by these two service discovery modes which could be useful for you to build the CloudWatch metrics dimensions\. For example, you can relabel `__meta_kubernetes_service_name` to `Service` and include it into your metrics’ dimension\. For more informatio about customizing your CloudWatch metrics and their dimensions, see [ CloudWatch Agent Configuration for Prometheus](ContainerInsights-Prometheus-Setup-configure-ECS.md#ContainerInsights-Prometheus-Setup-cw-agent-config)\.

## Docker Image with JMX Exporter<a name="ContainerInsights-Prometheus-Sample-Workloads-javajmx-docker"></a>

Next, build a docker image\. The following sections provide two example dockerfiles\.

When you have built the image, load it into Amazon EKS or Kubernetes, and then run the following command to verify that Prometheus metrics are exposed by JMX\_EXPORTER on port 9404\. Replace *$JAR\_SAMPLE\_TRAFFIC\_POD* with the running pod name and replace *$JAR\_SAMPLE\_TRAFFIC\_NAMESPACE* with your application namespace\. 

If you are running JMX Exporter on a cluster with the Fargate launch type, you also need to set up a Fargate profile before doing the steps in this procedure\. To set up the profile, enter the following command\. Replace *MyCluster* with the name of your cluster\.

```
eksctl create fargateprofile --cluster MyCluster \
--namespace $JAR_SAMPLE_TRAFFIC_NAMESPACE\
 --name $JAR_SAMPLE_TRAFFIC_NAMESPACE
```

```
kubectl exec $JAR_SAMPLE_TRAFFIC_POD -n $JARCAT_SAMPLE_TRAFFIC_NAMESPACE -- curl http://localhost:9404
```

## Example: Apache Tomcat Docker Image with Prometheus Metrics<a name="ContainerInsights-Prometheus-Sample-Workloads-javajmx-tomcat"></a>

Apache Tomcat server exposes JMX mBeans by default\. You can integrate JMX Exporter with Tomcat to expose JMX mBeans as Prometheus metrics\. The following example dockerfile shows the steps to build a testing image: 

```
# From Tomcat 9.0 JDK8 OpenJDK 
FROM tomcat:9.0-jdk8-openjdk 

RUN mkdir -p /opt/jmx_exporter

COPY ./jmx_prometheus_javaagent-0.12.0.jar /opt/jmx_exporter
COPY ./config.yaml /opt/jmx_exporter
COPY ./setenv.sh /usr/local/tomcat/bin 
COPY your web application.war /usr/local/tomcat/webapps/

RUN chmod  o+x /usr/local/tomcat/bin/setenv.sh

ENTRYPOINT ["catalina.sh", "run"]
```

The following list explains the four `COPY` lines in this dockerfile\.
+ Download the latest JMX Exporter jar file from [https://github\.com/prometheus/jmx\_exporter](https://github.com/prometheus/jmx_exporter)\.
+ `config.yaml` is the JMX Exporter configuration file\. For more information, see [https://github\.com/prometheus/jmx\_exporter\#Configuration](https://github.com/prometheus/jmx_exporter#Configuration )\.

  Here is a sample configuration file for Java and Tomcat:

  ```
  lowercaseOutputName: true
  lowercaseOutputLabelNames: true
  
  rules:
  - pattern: 'java.lang<type=OperatingSystem><>(FreePhysicalMemorySize|TotalPhysicalMemorySize|FreeSwapSpaceSize|TotalSwapSpaceSize|SystemCpuLoad|ProcessCpuLoad|OpenFileDescriptorCount|AvailableProcessors)'
    name: java_lang_OperatingSystem_$1
    type: GAUGE
  
  - pattern: 'java.lang<type=Threading><>(TotalStartedThreadCount|ThreadCount)'
    name: java_lang_threading_$1
    type: GAUGE
  
  - pattern: 'Catalina<type=GlobalRequestProcessor, name=\"(\w+-\w+)-(\d+)\"><>(\w+)'
    name: catalina_globalrequestprocessor_$3_total
    labels:
      port: "$2"
      protocol: "$1"
    help: Catalina global $3
    type: COUNTER
  
  - pattern: 'Catalina<j2eeType=Servlet, WebModule=//([-a-zA-Z0-9+&@#/%?=~_|!:.,;]*[-a-zA-Z0-9+&@#/%=~_|]), name=([-a-zA-Z0-9+/$%~_-|!.]*), J2EEApplication=none, J2EEServer=none><>(requestCount|maxTime|processingTime|errorCount)'
    name: catalina_servlet_$3_total
    labels:
      module: "$1"
      servlet: "$2"
    help: Catalina servlet $3 total
    type: COUNTER
  
  - pattern: 'Catalina<type=ThreadPool, name="(\w+-\w+)-(\d+)"><>(currentThreadCount|currentThreadsBusy|keepAliveCount|pollerThreadCount|connectionCount)'
    name: catalina_threadpool_$3
    labels:
      port: "$2"
      protocol: "$1"
    help: Catalina threadpool $3
    type: GAUGE
  
  - pattern: 'Catalina<type=Manager, host=([-a-zA-Z0-9+&@#/%?=~_|!:.,;]*[-a-zA-Z0-9+&@#/%=~_|]), context=([-a-zA-Z0-9+/$%~_-|!.]*)><>(processingTime|sessionCounter|rejectedSessions|expiredSessions)'
    name: catalina_session_$3_total
    labels:
      context: "$2"
      host: "$1"
    help: Catalina session $3 total
    type: COUNTER
  
  - pattern: ".*"
  ```
+ `setenv.sh` is a Tomcat startup script to start the JMX exporter along with Tomcat and expose Prometheus metrics on port 9404 of the localhost\. It also provides the JMX Exporter with the `config.yaml` file path\.

  ```
  $ cat setenv.sh 
  export JAVA_OPTS="-javaagent:/opt/jmx_exporter/jmx_prometheus_javaagent-0.12.0.jar=9404:/opt/jmx_exporter/config.yaml $JAVA_OPTS"
  ```
+ your web application\.war is your web application `war` file to be loaded by Tomcat\.

Build a docker image with this configuration and upload it to an image repository\.

## Example: Java Jar Application Docker Image with Prometheus Metrics<a name="ContainerInsights-Prometheus-Sample-Workloads-javajmx-jar"></a>

The following example dockerfile shows the steps to build a testing image: 

```
# Alpine Linux with OpenJDK JRE
FROM openjdk:8-jre-alpine

RUN mkdir -p /opt/jmx_exporter

COPY ./jmx_prometheus_javaagent-0.12.0.jar /opt/jmx_exporter
COPY ./SampleJavaApplication-1.0-SNAPSHOT.jar /opt/jmx_exporter
COPY ./start_exporter_example.sh /opt/jmx_exporter
COPY ./config.yaml /opt/jmx_exporter

RUN chmod -R o+x /opt/jmx_exporter
RUN apk add curl

ENTRYPOINT exec /opt/jmx_exporter/start_exporter_example.sh
```

The following list explains the four `COPY` lines in this dockerfile\.
+ Download the latest JMX Exporter jar file from [https://github\.com/prometheus/jmx\_exporter](https://github.com/prometheus/jmx_exporter)\.
+ `config.yaml` is the JMX Exporter configuration file\. For more information, see [https://github\.com/prometheus/jmx\_exporter\#Configuration](https://github.com/prometheus/jmx_exporter#Configuration )\.

  Here is a sample configuration file for Java and Tomcat:

  ```
  lowercaseOutputName: true
  lowercaseOutputLabelNames: true
  
  rules:
  - pattern: 'java.lang<type=OperatingSystem><>(FreePhysicalMemorySize|TotalPhysicalMemorySize|FreeSwapSpaceSize|TotalSwapSpaceSize|SystemCpuLoad|ProcessCpuLoad|OpenFileDescriptorCount|AvailableProcessors)'
    name: java_lang_OperatingSystem_$1
    type: GAUGE
  
  - pattern: 'java.lang<type=Threading><>(TotalStartedThreadCount|ThreadCount)'
    name: java_lang_threading_$1
    type: GAUGE
  
  - pattern: 'Catalina<type=GlobalRequestProcessor, name=\"(\w+-\w+)-(\d+)\"><>(\w+)'
    name: catalina_globalrequestprocessor_$3_total
    labels:
      port: "$2"
      protocol: "$1"
    help: Catalina global $3
    type: COUNTER
  
  - pattern: 'Catalina<j2eeType=Servlet, WebModule=//([-a-zA-Z0-9+&@#/%?=~_|!:.,;]*[-a-zA-Z0-9+&@#/%=~_|]), name=([-a-zA-Z0-9+/$%~_-|!.]*), J2EEApplication=none, J2EEServer=none><>(requestCount|maxTime|processingTime|errorCount)'
    name: catalina_servlet_$3_total
    labels:
      module: "$1"
      servlet: "$2"
    help: Catalina servlet $3 total
    type: COUNTER
  
  - pattern: 'Catalina<type=ThreadPool, name="(\w+-\w+)-(\d+)"><>(currentThreadCount|currentThreadsBusy|keepAliveCount|pollerThreadCount|connectionCount)'
    name: catalina_threadpool_$3
    labels:
      port: "$2"
      protocol: "$1"
    help: Catalina threadpool $3
    type: GAUGE
  
  - pattern: 'Catalina<type=Manager, host=([-a-zA-Z0-9+&@#/%?=~_|!:.,;]*[-a-zA-Z0-9+&@#/%=~_|]), context=([-a-zA-Z0-9+/$%~_-|!.]*)><>(processingTime|sessionCounter|rejectedSessions|expiredSessions)'
    name: catalina_session_$3_total
    labels:
      context: "$2"
      host: "$1"
    help: Catalina session $3 total
    type: COUNTER
  
  - pattern: ".*"
  ```
+ `start_exporter_example.sh` is the script to start the JAR application with the Prometheus metrics exported\. It also provides the JMX Exporter with the `config.yaml` file path\.

  ```
  $ cat start_exporter_example.sh 
  java -javaagent:/opt/jmx_exporter/jmx_prometheus_javaagent-0.12.0.jar=9404:/opt/jmx_exporter/config.yaml -cp  /opt/jmx_exporter/SampleJavaApplication-1.0-SNAPSHOT.jar com.gubupt.sample.app.App
  ```
+ SampleJavaApplication\-1\.0\-SNAPSHOT\.jar is the sample Java application jar file\. Replace it with the Java application that you want to monitor\.

Build a docker image with this configuration and upload it to an image repository\.