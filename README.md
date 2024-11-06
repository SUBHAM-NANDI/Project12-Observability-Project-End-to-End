# Project12-Observability-Project-End-to-End


## What is the Demo Application?

The demo application we’ll be using was developed by top observability companies like Datadog, Dynatrace, Microsoft, Alibaba, and Grafana Labs. This open-source project contains various microservices written in different languages, each one using OpenTelemetry to emit metrics, logs, and traces. This setup is perfect for learning observability because:
- It simulates a realistic, multi-service architecture typical in production applications.
- Each microservice supports OpenTelemetry, allowing you to learn how telemetry data is instrumented and collected.

### Key Features of the Application
- **Multi-microservice Architecture**: Similar to an e-commerce system, with services like Cart, Currency, Payment, Notification, Email, Recommendation, and Shipping.
- **Polyglot Setup**: Each service is written in a different language, such as Python, Go, and Java, enabling you to learn how OpenTelemetry works across multiple languages.
- **Documentation and Flexibility**: The application comes with instructions for deploying on Docker and Kubernetes, using Helm charts, and setting up on Kind or EKS clusters.

---

## Understanding OpenTelemetry and Its Role

Observability involves collecting, processing, and analyzing telemetry data (logs, metrics, and traces) from applications. OpenTelemetry, a CNCF project, standardizes this data collection process and is designed to work with any observability tool (like Datadog, Grafana, or Jaeger). It provides SDKs and APIs for developers to instrument telemetry data in a tool-agnostic manner.

### Why Use OpenTelemetry?
In modern DevOps, switching from one observability tool to another is common. Without OpenTelemetry, applications hardcode SDKs for specific tools (e.g., Prometheus or Nagios), making switching tools cumbersome and expensive. With OpenTelemetry:
- Developers instrument their code using a standard API.
- A configuration file determines which observability tool the data will be exported to, making it easy to change backends without modifying application code.

### Components of OpenTelemetry
1. **Receiver**: Collects metrics, logs, and traces from applications.
2. **Processor**: Processes telemetry data as needed.
3. **Exporter**: Exports telemetry data to the specified backend (e.g., Prometheus, Jaeger).

---

## Reviewing the Application’s Telemetry Instrumentation

In this application:
- Each microservice emits metrics, logs, and traces using OpenTelemetry SDKs.
- For instance, the **Recommendation** service, written in Python, uses OpenTelemetry SDK to define and export traces and metrics.

You can explore the codebase to see how different OpenTelemetry SDKs are used for each microservice. This will help you understand how telemetry data is implemented and structured within an application.

---

the observability backend (e.g., Prometheus for metrics, Jaeger for traces).

In this application, these configurations are defined in an **exporter configuration file**, where you specify your telemetry backend (Prometheus, Jaeger, etc.).

---


### Step 1: Set Up an EKS Cluster

1. **Prerequisites**: 
   - Ensure you have `eksctl` installed. You can check by running `eksctl version`.
   - You’ll also need AWS CLI and `kubectl` installed and configured.

2. **Create an EKS Cluster**:
   - Open a terminal and use the following commands to set up your EKS cluster:
     ```bash
     eksctl create cluster --name observability-demo --region <your-region> --nodegroup-name standard-workers --node-type t3.medium --nodes 3 --nodes-min 1 --nodes-max 4 --managed
     ```

3. **Configure IAM for EKS**:
   - Attach the IAM OIDC provider to your EKS cluster, which is required for OpenTelemetry setup:
     ```bash
     eksctl utils associate-iam-oidc-provider --region <your-region> --cluster observability-demo --approve
     ```

### Step 2: Deploy the Demo Application with OpenTelemetry on Kubernetes

1. **Clone the Demo Application**:
   - Clone the GitHub repository that contains the OpenTelemetry demo application. This project will allow us to learn observability through a microservice-based setup.
     ```bash
     git clone https://github.com/open-telemetry/opentelemetry-demo.git
     cd opentelemetry-demo
     ```

2. **Install Helm** (if not installed):
   - Install Helm, which is used for managing Kubernetes applications.
     ```bash
     curl https://raw.githubusercontent.com/helm/helm/master/scripts/get-helm-3 | bash
     ```

3. **Add Helm Chart for the Demo Application**:
   - The demo application has a Helm chart for easy deployment. Add the Helm repo:
     ```bash
     helm repo add open-telemetry https://open-telemetry.github.io/opentelemetry-helm-charts
     helm repo update
     ```

4. **Deploy the Demo Application**:
   - Deploy the application to your EKS cluster:
     ```bash
     helm install otel-demo open-telemetry/opentelemetry-demo
     ```

5. **Verify Deployment**:
   - After deployment, check if all pods are running. This demo has multiple microservices, so expect to see several pods.
     ```bash
     kubectl get pods
     ```

### Step 3: Expose the Demo Application

1. **Port Forwarding for Accessing the Application**:
   - For local testing, port forward the front-end service of the demo application:
     ```bash
     kubectl port-forward svc/frontend 8080:8080
     ```
   - Access the application at `http://localhost:8080`. Add some products to the cart to generate traffic.

2. **Alternative for Cloud-Based Access**:
   - If you are using a cloud instance, you can port forward like this:
     ```bash
     kubectl port-forward svc/frontend 8080:8080 --address 0.0.0.0
     ```
   - Use your cloud instance’s public IP to access the application.

### Step 4: Setting Up OpenTelemetry Components

OpenTelemetry uses a receiver, processor, and exporter to handle telemetry data. Let’s configure these to collect metrics, traces, and logs.

1. **Install OpenTelemetry Collector**:
   - Deploy the OpenTelemetry collector, which will process and export telemetry data.
     ```bash
     kubectl apply -f https://raw.githubusercontent.com/open-telemetry/opentelemetry-operator/main/config/crd/bases/opentelemetry.io_opentelemetrycollectors.yaml
     ```

2. **Configure the OpenTelemetry Collector**:
   - Define a configuration file that will export traces and metrics to Jaeger and Prometheus.
   - Create an `otel-collector.yaml` file:
     ```yaml
     apiVersion: opentelemetry.io/v1alpha1
     kind: OpenTelemetryCollector
     metadata:
       name: otel-collector
     spec:
       config: |
         receivers:
           otlp:
             protocols:
               grpc:
               http:
         exporters:
           prometheus:
           jaeger:
             endpoint: "http://jaeger:14250"
         service:
           pipelines:
             metrics:
               receivers: [otlp]
               exporters: [prometheus]
             traces:
               receivers: [otlp]
               exporters: [jaeger]
     ```
   - Deploy this configuration:
     ```bash
     kubectl apply -f otel-collector.yaml
     ```

3. **Install Jaeger and Prometheus**:
   - Use Helm to install Jaeger and Prometheus.
     ```bash
     helm install jaeger open-telemetry/jaeger --namespace observability
     helm install prometheus prometheus-community/prometheus --namespace observability
     ```

4. **Set Up Grafana for Visualization**:
   - Install Grafana, which will visualize the metrics data.
     ```bash
     helm install grafana grafana/grafana --namespace observability
     ```

### Step 5: Access Observability Tools

1. **Access Jaeger**:
   - Forward Jaeger’s UI port and open it in your browser to view traces.
     ```bash
     kubectl port-forward svc/jaeger-query 16686:16686 -n observability
     ```
   - Visit `http://localhost:16686` and search for traces generated by the demo application.

2. **Access Prometheus**:
   - Forward Prometheus’s UI port to view metrics.
     ```bash
     kubectl port-forward svc/prometheus-server 9090:80 -n observability
     ```
   - Access Prometheus at `http://localhost:9090`.

3. **Access Grafana**:
   - Forward Grafana’s UI port and set up dashboards for visualizing metrics and traces.
     ```bash
     kubectl port-forward svc/grafana 3000:3000 -n observability
     ```
   - Access Grafana at `http://localhost:3000`. The default login is `admin/admin`.
   - In Grafana, add Prometheus as a data source and create dashboards for visualizing application metrics.

### Step 6: Observing Telemetry Data

1. **Simulate Traffic**:
   - To generate more telemetry data, interact with the demo application by adding products to the cart, removing items, and checking out.

2. **Monitor Metrics in Grafana**:
   - Use Grafana’s Prometheus dashboards to view metrics like CPU usage, memory, and request rates.

3. **Trace Requests with Jaeger**:
   - Use Jaeger to trace the requests across microservices. Look for spans related to services like `checkout` or `recommendation`, and check the time spent in each service.

### Step 7: Customize and Extend Observability

1. **Add Additional Metrics or Traces**:
   - Edit the source code of a microservice (e.g., the recommendation service in Python) to add custom metrics or trace points using OpenTelemetry SDKs.

2. **Deploy the Updated Code**:
   - Rebuild and redeploy the application to observe your new metrics or trace points in action.

3. **Configure Alerts**:
   - Set up alerts in Prometheus or Grafana to notify you if certain metrics exceed thresholds (e.g., high latency or CPU usage).

---
