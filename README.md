# Blog Link
**https://blog.kubesimplify.com/optimizing-scalability-a-deep-dive-into-load-testing-with-locust-on-eks**


# Introduction
This deep dive explores strategies for optimizing scalability using Locust for load testing on Amazon EKS. We'll delve into scaling a Node.js app using Kubernetes' HPA and Cluster Autoscaler based on load generated by Locust workers. The aim is to provide practical insights into ensuring applications can efficiently handle increasing user loads.



# Prerequisites
- Ensure you have AWS CLI configured with appropriate permissions and Terraform installed locally.
- Additionally, a basic understanding of AWS services, Terraform, Kubernetes concepts, Horizontal Pod Autoscaling (HPA) principles, and familiarity with the Kubernetes Cluster Autoscaler is required.



# Understanding Horizontal Pod Autoscaler
HPA automatically adjusts the number of replica pods in a deployment or replication controller based on observed CPU utilization or other custom metrics. This ensures that the application has sufficient resources to handle varying loads, thus improving performance and scalability.



# Introduction to Locust
Locust is an open-source load testing tool that allows you to define user behavior with Python code. It simulates thousands of concurrent users hitting your application, making it an excellent choice for load testing in Kubernetes environments.



# [Create VPC and EKS using Terraform module](https://github.com/anshu049/Load-Test-using-Locust-and-Monitoring/tree/master/eks_using_terraform_modules)
- Apply the terraform command..
- Add [**policy**](https://github.com/kubernetes/autoscaler/blob/master/cluster-autoscaler/cloudprovider/aws/README.md) to the worker node.
- Apply [**apply cluster-autoscaler-autodiscover.yaml**](https://github.com/kubernetes/autoscaler/blob/master/cluster-autoscaler/cloudprovider/aws/examples/cluster-autoscaler-autodiscover.yaml) after updating the `cluster-name` and `image version` in the the deployment section to match your Kubernetes version.
- Add [**metric server**](https://github.com/kubernetes-sigs/metrics-server) for HPA to gather data.
- We need to create HPA for both locust and app after deploying them.
- `kubectl autoscale deployment <deploy-name> --cpu-percent=50 --min=1 --max=10`



# Install monitoring components on cluster
- **Run the following commands:**
- `helm repo add prometheus-communityhttps://prometheus-community.github.io/helm-charts`
- `helm repo update`
- `kubectl create ns monitoring`
- `helm install monitoring prometheus-community/kube-prometheus-stack -n monitoring`
- Edit the service of Prometheus and Grafana from ClusterIP to `LoadBalancer` to access the UI.
- Prometheus don't require initial password, for Grafana we can run
- `kubectl get secret --namespace monitoring monitoring-grafana -o jsonpath="{.data.admin-password}" | base64 --decode ; echo`



# [Deploy Loust](https://github.com/anshu049/Load-Test-using-Locust-and-Monitoring/tree/master/locust)
- Update the Configmap provided in the GitHub repo before applying it. Add the actual URL of the app in the `task` section so that Locust can send traffic to the right place. Locust's master-service is configured as LoadBalancer type to access the UI.


  
# [Build Docker image using Github Action(CI)](https://github.com/anshu049/Load-Test-using-Locust-and-Monitoring/blob/master/.github/workflows/CI.yaml)
- Github Action automatically creates new Image from the latest changes made in app.



# [Deployment of App using ArgoCD(CD)](https://github.com/anshu049/Node-App-Manifests)
- After applying the manifests, access the app using the LoadBalancer IP.
![argo](https://github.com/anshu049/Load-Test-using-Locust-and-Monitoring/assets/95365748/7f3d931b-a8ec-4543-947d-291708a064d4)

-- **Configuration file of our application for monitoring**
```  apiVersion: monitoring.coreos.com/v1
  kind: ServiceMonitor
  metadata:
    name: monitoring-node-app
    labels:
      release: monitoring
      app: nodeapp
  spec:
    endpoints:
    - path: /metrics
      port: service
      targetPort: 3000
    namespaceSelector:
      matchNames:
      - nodeapp # namespace in which app is deployed
    selector:
      matchLabels:
        app: nodeapp
```
- After applying the above YAML, add metrics in Grafana to view the dashboard.
- In Prometheus UI search for http_request_operations_total (for total request generated) and sum(rate(http_request_operations_total[15m])) (for request/second).
- In Grafana UI create new dashboard by adding above two metrics.



# Demo
- Access UI of locust and Run Test
Access the Locust UI to configure and initiate the load test. Define the behavior of simulated users, such as the number of users, the rate of requests, and specific endpoints to target.
![adding-url-to-locust](https://github.com/anshu049/Load-Test-using-Locust-and-Monitoring/assets/95365748/7434ca15-435d-4c13-b3d2-26ce97335dbc)

## Grafana Dashboard
This metric indicates the rate at which Locust is generating requests to your Node.js app, showcasing the simulated load.
![request:second-final](https://github.com/anshu049/Load-Test-using-Locust-and-Monitoring/assets/95365748/92c36e31-8c81-4160-ade8-41aaa1af8f2a)


This metric presents the total number of requests sent by Locust to your application, offering a comprehensive view of the applied load..
![total-request-generated-final](https://github.com/anshu049/Load-Test-using-Locust-and-Monitoring/assets/95365748/6e081252-9d71-4863-bbe6-ab246e94dc2a)


Observe the cluster's resource utilization metrics, including CPU and memory usage, and witness the dynamic scaling of the cluster in response to increased load.
![cluster-final](https://github.com/anshu049/Load-Test-using-Locust-and-Monitoring/assets/95365748/e006ad5d-3019-475d-a89e-4ea706af0d27)


Allow it to run for a few minutes, during which time you can switch between the "Statistics" tab and the "Charts" tab to observe the progress of the test.
![locust-overview](https://github.com/anshu049/Load-Test-using-Locust-and-Monitoring/assets/95365748/089e919e-9f5f-48bc-9306-2b7cc078ac4b)
We can observe that 69 Locust workers have been created using HPA, to generate load on our app.



# Now, let's observe how our cluster has scaled

# HPA
When the workload on your Node.js application surpasses the defined threshold, which is determined by CPU utilization, the Horizontal Pod Autoscaler (HPA) takes action. It dynamically adjusts the number of pods running your application to match the current demand. This means that if your application experiences higher traffic or processes more requests, HPA will trigger the creation of a new pod to handle the additional load.
![hpa](https://github.com/anshu049/Load-Test-using-Locust-and-Monitoring/assets/95365748/45abf727-5d98-43de-b863-42b2092c5465)


# Pending State
After HPA initiates the creation of a new pod, the Kubernetes scheduler tries to assign the pod to an available node within your cluster. However, if there aren't enough resources (such as CPU, memory, or disk space) on the existing nodes to accommodate the new pod, it enters a "pending" state. This indicates that the pod is waiting for resources to become available before it can start running.
![pods-pending](https://github.com/anshu049/Load-Test-using-Locust-and-Monitoring/assets/95365748/4cda45dd-5697-4880-8c88-7a14510b5dae)


# Automatic Node Creation by Cluster Autoscaler
Recognizing that the pending pod requires additional resources that cannot be met by the existing nodes, the Cluster Autoscaler takes action. It monitors the resource utilization across your cluster and identifies the need for more compute capacity. In response to this demand, the Cluster Autoscaler automatically provisions a new worker node (virtual machine) within your Kubernetes cluster.
![new-node-created](https://github.com/anshu049/Load-Test-using-Locust-and-Monitoring/assets/95365748/88a173c8-2e73-4ad2-9248-482279cbb5e6)


# Transition to Running State for Pods
Once the new worker node is provisioned and ready to accept pods, the pending pod transitions from the "pending" state to the "running" state. This signifies that the pod is now actively serving requests and contributing to handling the increased load on your application. With the new pod running and workload distributed across multiple pods, your application can effectively manage the surge in traffic without compromising performance or availability.
![new-pod-started](https://github.com/anshu049/Load-Test-using-Locust-and-Monitoring/assets/95365748/3b21cd0d-70ea-45c2-a6a2-422a6efe79c5)


# Conclusion 
In summary, this detailed exploration of load testing with Locust on Amazon EKS focused on optimizing scalability for a Node.js application using Kubernetes' HPA and Cluster Autoscaler. Key steps included setting up EKS clusters, implementing worker node autoscaling policies, enabling HPA, integrating the Cluster Autoscaler, and deploying monitoring components like Prometheus and Grafana. The process showcased how the system dynamically scaled resources in response to increased load, ensuring efficient handling of user traffic. Overall, the guide offers practical insights for developers and DevOps teams to improve scalability and performance in Kubernetes environments.

**Remember to delete all resources after the demo.**

**Note: Yellow-colored text signifies hyperlinks to GitHub repositories used.**
