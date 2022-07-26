# prometheus-grafana-with-micrometer

This is simplified guide to create a monitoring dashboard using Prometheus, Grafana and Micrometer.  

Most of yaml files where copied from the Bibin Wilson's article **How to Setup Prometheus Monitoring On Kubernetes Cluster** (https://devopscube.com/setup-prometheus-monitoring-on-kubernetes). If possible, I recommend you to read the article.

Summary:
1. Kubernates setup: prepare the Kubernetes cluster using Minikube;
2. Prometheus setup: deploy Prometheus on Kubernetes;
3. Grafana setup: deploy Grafana on Kubernetes;
4. SpringBoot application setup: configuration and deploy of the test application;
5. Creating a dashboard in Grafana.

## 1. Kubernetes setup

### 1.1 Install minikube

    > curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube_latest_amd64.deb
    > sudo dpkg -i minikube_latest_amd64.deb

PS.: check reference (1.6) to choose the correct installation for your operating system. 

### 1.2 User setup

    > sudo usermod -aG docker $USER && newgrp docker

Alias configuration (file ~/.bashrc):

    > alias kubectl='minikube kubectl --'
    > source ~/.bashrc

### 1.3 Start minikube

    > minikube start
    > kubectl get namespaces

Minikube dashboard:

    > minikube dashboard

### 1.4 Create a namespace and the ClusterRole

    > kubectl create namespace monitoring
    > kubectl create -f clusterRole.yaml

### 1.5 Create a Config Map for prometheus configurations

    > kubectl create -f config-map.yaml

### 1.6 Reference

https://minikube.sigs.k8s.io/docs/start

## 2. Prometheus setup

### 2.1 Deploy prometheus

    > kubectl create -f prometheus-deployment.yaml

### 2.2 Expose prometheus as a service

    > kubectl create -f prometheus-service.yaml --namespace=monitoring

### 2.3 Access prometheus dashboard

Prometheus dashboard can be accessed through the node IP and the service port.

    > kubectl get pods --namespace monitoring
    NAME                                     READY   STATUS    RESTARTS   AGE
    prometheus-deployment-84f65c89c5-tnwkn   1/1     Running   0          21m
    > kubectl describe pod prometheus-deployment-84f65c89c5-tnwkn --namespace monitoring
    ...
    Node:         minikube/192.168.49.2
    ...
    > kubectl get services --namespace monitoring
    NAME                 TYPE       CLUSTER-IP      EXTERNAL-IP   PORT(S)          AGE
    prometheus-service   NodePort   10.101.68.250   <none>        8080:30000/TCP   30m

From the result above, prometheus dashboard is available at http://192.168.49.2:30000.

## 3. Grafana setup

### 3.1 Deploy on Kubernetes

    > kubectl apply -f grafana.yaml

### 3.2 Accessing Grafana console

Grafana can be accessed at the node IP and service port. So, you can use the same IP address used to access Prometheus dashboard and change the port. Again, to find out what is the service port:

    > kubectl get services --namespace monitoring
    NAME                 TYPE       CLUSTER-IP       EXTERNAL-IP   PORT(S)          AGE
    grafana              NodePort   10.98.119.176    <none>        3000:30001/TCP   56m
    
From the result above and considering the node IP 192.168.49.2, Grafana interface must be available at http://192.168.49.2:30001. Use the user and password `admin` to login in Grafana.

### 3.3 Data source configuration

Add a new data source for Prometheus in the menu **Configuration > Data Sources**. You must inform Prometheus URL into the field "URL" (see section 2.3).

### 3.4 Reference

https://grafana.com/docs/grafana/latest/installation/kubernetes/

## 4. SpringBoot application setup

### 4.1 Micrometer dependence

Dependencies (build.gradle):

    implementation 'org.springframework.boot:spring-boot-starter-actuator'
    implementation 'io.micrometer:micrometer-registry-prometheus'

### 4.2 Collecting metrics

Annotate the method with `@Timed`:

    @Timed("beer.create")
    @PostMapping(V1 + "/beer")
    public CreateBeerResponse createBeer(@RequestBody @Valid CreateBeerRequest request) {
        ...
    }

Micrometer will collect metrics for this method and use the prefix `beer_create_` to identify them.

### 4.3 Microservice deploy

    > kubectl create deployment hmo-crud --image=heliomolive/hmo-crud:38 --namespace monitoring
    > kubectl expose deployment hmo-crud --type=NodePort --port 8080 --namespace monitoring

### 4.4 Checking collected metrics

First, find the service port where the microservice was exposed:

    kubectl get services --namespace monitoring
    NAME                 TYPE       CLUSTER-IP       EXTERNAL-IP   PORT(S)          AGE
    hmo-crud             NodePort   10.107.147.180   <none>        8080:30469/TCP   23h

Then, you can use the following endpoints to verify the current metrics:

- http://192.168.49.2:30469/actuator/prometheus - application's endpoint that exhibits all metrics that will be collected by Prometheus.
- http://192.168.49.2:30000/api/v1/query?query=beer_create_seconds_count - Prometheus' endpoint, the return exhibits a vector with the values collected for this metric.

## 5. Tips for Micrometer and Grafana

### 5.1 Use a timer instead of a counter

A counter is a meter type in Grafana that reports only a count. On the other hand, a timer includes a count and other metrics.

### 5.2 Growth rate is more important than the current value

As we are monitoring real time metrics, most often we want to know how much the value increases or decreases over time.

Prometheus' increase function can be very helpful for this. The following example will inform how many beers were created in the last 5 minutes:

    increase(beer_create_seconds_count[5m])

### 5.3 Approximate values

Prometheus will not give you exact values, this can vary depending on the moment when Prometheus scraped the metrics. But it give us an overview of what is happening.

### 5.4 Prefer bigger instead of smaller ranges

The unsteadiness of counter increases is more explicit when you set up a small range length for a counter. This happens with the function `rate` -- so, for instance, you should prefer a range lenght of 1 hour instead of 5 minutes. Remember that you are not working with exact values, so the behaviour of your metrics will be clear with a wider data sample.

### 5.5 References

https://micrometer.io/docs/concepts

https://www.innoq.com/en/blog/prometheus-counters/
