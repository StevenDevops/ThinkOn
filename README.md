# ThinkOn
This is the first step of the Hiring process for the DevOps Engineer position (https://thinkon.applytojobs.ca/software+development/18433)

Preamble
Assume you’re running on your favorite cloud (Azure, AWS, GCP) - you don’t have to make this work specifically for GCP GKE.
Create a README.md that outlines your line of thinking for the solution.
Create plain Kubernetes resources (yaml or json). Please return this file in your response with any other materials you want to share with us.
You can make the following assumptions:
Each system you’re deploying has its own isolated database. You don’t have to worry about the type. You can assume the database is in the same region as your k8s.
You can use any docker image you’d like for your containers. It’s just an example and does not have to work. Any, say, default php docker you can deploy on a pod. What the container it is, does not matter - but we’ll be talking about two different containers in the exercise, one for users, and one for shifts.
Assume daily bell-curve scaling. High traffic during the day, low traffic during the night.

Exercise
1.) We want to deploy two containers that scale independently from one another.
Container 1: This container runs code that runs a small API that returns users from a database.
Container 2: This container runs code that runs a small API that returns shifts from a database.
2.) For the best user experience auto scale this service when the average CPU reaches 70%.
3.) Ensure the deployment can handle rolling deployments and rollbacks.
4.) Your development team should not be able to run certain commands on your k8s cluster, but you want them to be able to deploy and roll back. What types of IAM controls do you put in place?

Bonus
·    How would you apply the configs to multiple environments (staging vs production)?
·    How would you auto-scale the deployment based on network latency instead of CPU?

# Answer
1.) I deployed the [deployment.yaml](https://github.com/StevenDevops/ThinkOn/blob/main/deployment.yaml) file. 
Basically, we had 2 deployments for each API. It depends on the specific requirements and constraints of your use case. In general, having two separate deployments allows for more fine-grained control over the scaling and deployment of each container. For example, if the traffic patterns for the two APIs are different, it may make sense to have separate deployments with different scaling policies.

However, if the two containers are tightly coupled and need to be deployed and scaled together, it may be more convenient to have them in a single deployment. Having a single deployment can also simplify management and reduce resource consumption. That's why I choose this way.

They will get the value from k8s secret [db-credentials.yaml](https://github.com/StevenDevops/ThinkOn/blob/main/db-credentials.yaml)

You can create this secret via command:
```
export ENV=staging $$ kubectl create secret generic db-credentials-$ENV \
--from-literal=DB_HOST=<database-hostname> \
--from-literal=DB_NAME=<database-name> \
--from-literal=DB_USER=<database-username> \
--from-literal=DB_PASSWORD=<database-password>
```
2.) We can use [HPA](https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale/)
This [configuration](https://github.com/StevenDevops/ThinkOn/blob/main/hpa.yaml) specifies that the HPA will monitor CPU utilization and maintain an average utilization of 70%. If the utilization exceeds this threshold, the HPA will automatically increase the number of replicas up to a maximum of 5.

3.) To enable rolling deployments and rollbacks, we need to define a rollout strategy in the deployment configuration. I already added them here: [deployment.yaml](https://github.com/StevenDevops/ThinkOn/blob/main/deployment.yaml#L13-L16)
This configuration specifies that during a rolling update, only one replica can be unavailable at a time (maxUnavailable: 1) and only one new replica can be created at a time (maxSurge: 1).
To roll back to a previous version, we can use the kubectl rollout undo command, which reverts the deployment to the previous revision.

4.) Make sure you already had group `deployer` with correct members. I created the [rbac.yaml](https://github.com/StevenDevops/ThinkOn/blob/main/rbac.yaml)
Once you have created the role and role binding, members of the `deployer` group will be able to deploy and roll back deployments, view logs of pods, but they will not be able to perform other actions, such as creating or modifying secrets or services.


5.) Depends on the situation, we can use [kustomize](https://kustomize.io/) or just simple command via [envsubst](https://man7.org/linux/man-pages/man1/envsubst.1.html):
```
export ENV=staging
envsubst < deployment.yaml | kubectl apply -f -
```

6.) We can use [custom-metrics](https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale/#scaling-on-custom-metrics) from HPA.
The procedure will be:

```
Install and configure Prometheus to scrape network latency metrics from the pods of the deployment.

Install and configure Prometheus Adapter to expose the Prometheus metrics to Kubernetes as custom metrics.

Create a Horizontal Pod Autoscaler (HPA) that uses the custom metric for network latency to scale the deployment.
```
* First, we need to enable  [metrics](https://github.com/kubernetes-sigs/metrics-server)
  `kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml`
  By default, Kubernetes Metrics Server only collects CPU and memory metrics for pods, nodes, and namespaces. To collect other metrics such as network latency, custom metrics adapters or external monitoring solutions can be used. In this case, I think we can use prometheus exporter.
* Install [node-exporter](https://github.com/StevenDevops/ThinkOn/blob/main/deployment.yaml#L74-L76) for PODs via a sidecar
* Install [Prometheus Adapter](https://github.com/kubernetes-sigs/prometheus-adapter) 
  `kubectl apply -f https://github.com/kubernetes-sigs/prometheus-adapter/releases/download/v0.10.0/release-bundle.yaml`
* Create network-latency metric via [service-monitor](https://github.com/StevenDevops/ThinkOn/blob/main/monitoring.yaml)
* Create [HPA](https://github.com/StevenDevops/ThinkOn/blob/main/hpa.yaml)


