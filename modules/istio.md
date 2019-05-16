# Istio

## Objectives
* Install Istio
* Configure the Default Namespace
* Deploying a sample app with Istio sidecars
* Create an Istio Ingress Gateway
* Create a Virtual Service
* Generate Traffic
* Monitoring and Tracing
* Request Routing
* Traffic Mirroring
* Traffic Shifting
* Fault Injection
* Circuit Breaking
* Rate Limiting using Memquota Memory Store
* Rate Limiting using Redis Memory Store

# Prerequisites
- Deploy a cluster on GKE with

  ```
  gcloud beta container clusters create gke-workshop --num-nodes 4 --zone us-west2-b --preemptible --enable-stackdriver-kubernetes
  ```

# Install Istio

In this exercises you will deploy Istio onto the GKE cluster.

1. Grant cluster admin permissions to the current user:

    ```shell
    kubectl create clusterrolebinding cluster-admin-binding \
      --clusterrole=cluster-admin \
      --user="$(gcloud config get-value core/account)"
    ```

    > Note: You need these permissions to create the necessary role based access control (RBAC) rules for Istio

1. Download the Istio release.

    ```shell
    cd $HOME
    wget https://github.com/istio/istio/releases/download/1.0.5/istio-1.0.5-linux.tar.gz
    tar xzf istio-1.0.5-linux.tar.gz
    cd istio-1.0.5
    ```

1. Add the `istioctl` client to your PATH, we can do this by adding the following line to the `~/.bashrc` file:

    ```shell
    export PATH=$HOME/istio-1.0.5/bin:$PATH
    ```

1. Install Istio's core components:

    ```shell
    kubectl apply -f install/kubernetes/istio-demo-auth.yaml
    ```

    This does the following:

    * Creates the `istio-system` Namespace along with the required RBAC permissions

    * Deploys the core Istio components:

        * `Istio-Pilot` is responsible for service discovery and for configuring the Envoy sidecar proxies in an Istio service mesh

        * The Mixer components `Istio-Policy` and `Istio-Telemetry` enforce usage policies and gather telemetry data across the service mesh

        * `Istio-Ingressgateway` provides an ingress point for traffic from outside the cluster

        * `Istio-Citadel` automates key and certificate management for Istio

    * Deploys plugins for metrics, logs, and tracing

    * Enables mutual TLS authentication between Envoy sidecars

1. Verify that Istio is correctly installed.

    ```shell
    kubectl get service -n istio-system
    kubectl get pods -n istio-system
    ```

    All pods should be in `Running` or `Completed` state.

Now you are ready to deploy the sample application to the Istio cluster.

# Deploy Microservices - Sidecar Injection

1. Enable automatic sidecar injection
    ```shell
    kubectl label namespace default istio-injection=enabled
    ```
    Injection can also be done using the `istioctl` utility. Here is an example of how you would inject sidecars into our sample app:

    ```shell
    # Inject sidecars and save as another manifest
    istioctl kube-inject -f manifests/sample-app.yaml  > manifests/sample-app-istio.yaml

    # DO NOT RUN THIS - THIS IS AN EXAMPLE
    # Apply the new manifest

    # kubectl apply -f manifests/sample-app-istio.yaml
    ```

1. Enable mTLS for all apps in the namespace. Save this file as `manifests/enable-mtls.yaml`, and apply
    ```yaml
    apiVersion: authentication.istio.io/v1alpha1
    kind: Policy
    metadata:
      name: default
      namespace: default
    spec:
      peers:
      - mtls: {}
    ```

    ```shell
    kubectl apply -f manifests/enable-mtls.yaml
    ```

## Deploy the Sample App

We will deploy a simple microservices architecture app.

Here is a very high-level view of the architecture.
```
-----------      ------------      -----------      -----------
|         |      |          |      |         |      |         |
|  user   | ---> |  gceme   | ---> |  gceme  | ---> |  mysql  |
|(browser)|      |(frontend)|      |(backend)|      |  (db)   |
|         |      |          |      |         |      |         |
-----------      ------------      -----------      -----------
```

1. Deploy the sample app. Save the following as `manifests/sample-app.v1.yaml` and apply

    ```yaml
    ---
    apiVersion: v1
    data:
      password: cm9vdA==
    kind: Secret
    metadata:
      name: mysql
    ---
    apiVersion: v1
    kind: Service
    metadata:
      name: db-v1
      labels:
        app: gcme
        role: db
        version: v1
    spec:
      type: ClusterIP
      ports:
        - port: 3306
      selector:
        app: gceme
        role: db
        version: v1
    ---
    apiVersion: apps/v1
    kind: Deployment
    metadata:
      name: db-v1
      labels:
        app: gcme
        role: db
        version: v1
    spec:
      replicas: 1
      selector:
        matchLabels:
          app: gceme
          role: db
          version: v1
      template:
        metadata:
          name: db
          labels:
            app: gceme
            role: db
            version: v1
        spec:
          containers:
          - image: mysql:5.6
            name: mysql
            env:
            - name: MYSQL_ROOT_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: mysql
                  key: password
            ports:
            - containerPort: 3306
              name: mysql
    ---
    kind: Service
    apiVersion: v1
    metadata:
      name: backend-v1
      labels:
        app: gcme
        role: backend
        version: v1
    spec:
      type: ClusterIP
      ports:
      - name: http
        port: 8080
      selector:
        role: backend
        app: gceme
        version: v1
    ---
    apiVersion: apps/v1
    kind: Deployment
    metadata:
      name: backend-v1
      labels:
        app: gcme
        role: backend
        version: v1
    spec:
      replicas: 1
      selector:
        matchLabels:
          app: gceme
          role: backend
          version: v1
      template:
        metadata:
          name: backend
          labels:
            app: gceme
            role: backend
            version: v1
        spec:
          containers:
          - name: backend
            image: gcr.io/barry-williams/sample-k8s-app:v1
            env:
            - name: MYSQL_ROOT_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: mysql
                  key: password
            imagePullPolicy: Always
            command: ["sh", "-c", "app -run-migrations -mode=backend -run-migrations -port=8080 -db-host=db-v1 -db-password=$MYSQL_ROOT_PASSWORD" ]
            ports:
            - name: backend
              containerPort: 8080
    ---
    kind: Service
    apiVersion: v1
    metadata:
      name: frontend
      labels:
        app: gcme
        role: frontend
    spec:
      type: ClusterIP
      ports:
      - name: http
        port: 80
      selector:
        app: gceme
        role: frontend
    ---
    apiVersion: apps/v1
    kind: Deployment
    metadata:
      name: frontend-v1
      labels:
        app: gcme
        role: frontend
        version: v1
    spec:
      replicas: 1
      selector:
        matchLabels:
          app: gceme
          role: frontend
          version: v1
      template:
        metadata:
          name: frontend
          labels:
            app: gceme
            role: frontend
            version: v1
        spec:
          containers:
          - name: frontend
            image: gcr.io/barry-williams/sample-k8s-app:v1
            imagePullPolicy: Always
            command: ["app", "-mode=frontend", "-backend-service=http://backend-v1:8080", "-port=80"]
            ports:
            - name: frontend
              containerPort: 80
    ```

    ```shell
    kubectl apply -f manifests/sample-app.v1.yaml
    ```

1. Check to see how many containers are in the pod

    ```shell
    kubectl get pods
    ```

# Create an Istio Ingress Gateway

At this moment, there is no path to send traffic to our app

1. Run this command to setup port forwarding with the istio-ingress's admin site:

```shell
kubectl -n istio-system port-forward `kubectl -n istio-system get pods -l app=istio-ingressgateway -o go-template --template '{{range .items}}{{.metadata.name}}{{"\n"}}{{end}}' | head -n 1` 15000
```

1. Now open the GCP console's web preview on port 15000

1. Click `listeners`

    You should see something like

    ```json
    ["https://15090-dot-5113692-dot-devshell.appspot.com"]
    ```

    These are the endpoints that the gateway is currently listening for. This is only one host, and it doesn't give us access to our app. We need to open up port 80 and listen on all hosts.

Lets create a `gateway` to intercept ingress traffic

1. Open another cloud shell window and create an Istio Gateway by saving this file as `manifests/istio-gateway.yaml` and applying with kubectl

    ```yaml
    apiVersion: networking.istio.io/v1alpha3
    kind: Gateway
    metadata:
      name: gceme-gateway
    spec:
      selector:
        istio: ingressgateway # Use Istio default controller
      servers:
      - port:
          number: 80
          name: http
          protocol: HTTP
        hosts:
          - "*"
    ```

    ```shell
    kubectl apply -f manifests/istio-gateway.yaml
    ```

1. Check that the Gateway is created

    ```shell
    kubectl get gateway
    ```

    ```
    NAME            AGE
    gceme-gateway   3m
    ```

1. Go back to the admin site. If needed, click listeners, or refresh the page.

    You should now see something like this:

    ```json
    ["https://15090-dot-5113692-dot-devshell.appspot.com","0.0.0.0:80"]
    ```

    If you do not see anything, you may need to restart the ingressgateway pod with this command:

    ```shell
    # Only if the gateway is not showing the new host
    kubectl delete pod -n istio-system -l app=istio-ingressgateway
    ```

    We now have an endpoint that will accept http traffic. Lets check it out

1. Get the IP of the Ingress (from the load balancer service)

    ```shell
    export INGRESS_HOST=$(kubectl -n istio-system get service istio-ingressgateway -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
    export INGRESS_PORT=$(kubectl -n istio-system get service istio-ingressgateway -o jsonpath='{.spec.ports[?(@.name=="http2")].port}')
    export SECURE_INGRESS_PORT=$(kubectl -n istio-system get service istio-ingressgateway -o jsonpath='{.spec.ports[?(@.name=="https")].port}')
    export GATEWAY_URL=$INGRESS_HOST:$INGRESS_PORT

    echo $GATEWAY_URL
    ```

    > Note: You can add this to your `~/.profile` file to automatically load on startup of new shell sessions.

1. Go to that IP address in your browser

    What do you see?

    You may have noticed that you received a 404

    Even though we can send traffic to the `ingressgateway`, we still get an error because it does not yet know where to send traffic. The gateway is just a listener.

## Create a Virtual Service

1. Create a VirtualService for the frontend by saving this file as `manifests/frontend-vs.yaml` and apply with kubectl

    ```yaml
    apiVersion: networking.istio.io/v1alpha3
    kind: VirtualService
    metadata:
      name: frontend-vs
    spec:
      hosts:
      - "*"
      gateways:
      - gceme-gateway
      http:
      - route:
        - destination:
            host: frontend
            port:
              number: 80
    ```

    ```shell
    kubectl apply -f manifests/frontend-vs.yaml
    ```

1. Once again, open the value of `$GATEWAY_URL` in a browser. The app should now be accessible.

    You can write notes and save them in the database.

    > Note: This app was written to talk to the GCE API, but you don't see a majority of the information about the GCE instance. This is because the app gets this info from the `metadata.google.internal` server, which is not part of the Istio service mesh.

## Generate traffic

This will be referred to several times throughout the workshop

1. Install `hey`, a load testing tool

    ```shell
    go get -u github.com/rakyll/hey
    ```

1. Generate requests (Runs for 60 minutes and generates 10 requests per second)

    ```shell
    hey -z 60m -c 1 -q 10 http://$GATEWAY_URL
    ```

# Monitoring and Tracing

1. Set up a temporary tunnel to Grafana by using port-forwarding.

    ```shell
    kubectl -n istio-system port-forward $(kubectl -n istio-system get pod -l app=grafana -o jsonpath='{.items[0].metadata.name}') 3000:3000
    ```

    > Note: When you CTRL+C from this process, the tunnel will be closed and you will no longer be able to access the UI.

1. Open Web Preview at port `3000`, you should see the Grafana interface.

1. In the left panel click on `Dashboards -> Manage` and then select the `istio` folder. You should see a lot of dashboards, let's check a couple of them.

    * `Istio Mesh Dashboard` - Displays the overall volume of the requests as well as number of failed requests and track requests latency. Refresh the frontend page several times, add a couple of notes and make sure you see a spike in global request volume graph and changes in overall request statistics.
    * `Istio Service Dashboard` - Contains more detail information about request statistics. You can use this dashboard to see the request statistics per service.
    * `Istio Workload Dashboard` - Gives details about metrics for each workload and then inbound workloads (workloads that are sending request to this workload) and outbound services (services to which this workload send requests) for that workload. Note that the workload name is the name of the deployment.
        - look at "Incoming Requests by Source and Response Code" - this shows a detailed view of source hosts. Switch the view to "backend-v1.default.svc.cluster.local", you should see that all requests are coming from frontend, and should mostly be 200.
        - Switch to frontend-v1.default.svc.cluster.local. You should see that most of it's requests are coming from the ingressgateway

1. Press `CTRL-C` to exit

1. Set up a temporary tunnel to ServiceGraph by using port-forwarding.

    ```shell
    kubectl -n istio-system port-forward $(kubectl -n istio-system get pod -l app=servicegraph -o jsonpath='{.items[0].metadata.name}') 8088:8088
    ```

1. Open web preview at port `8088`

1. Re-write the path again to be: `/force/forcegraph.html?time_horizon=15s&filter_empty=true`

    What do you see?

    Click on a service. What details pop up?

    Parameters explained:

    `filter_empty=true` will only show services that are currently receiving traffic within the time horizon.

    `time_horizon=15s` affects the filter above, and also affects the reported traffic information when clicking on a service. The traffic information will be aggregated over the specified time horizon.

1. Re-write the path to be `/dotviz`. You should see the full topology of services in the mesh

1. Press `CTRL-C` to exit

1. Set up a temporary tunnel the Jaeger dashboard by using port-forwarding.

    ```shell
    kubectl port-forward -n istio-system $(kubectl get pod -n istio-system -l app=jaeger -o jsonpath='{.items[0].metadata.name}') 16686:16686
    ```

1. Open web preview at port `16686`, you should see the Jaeger interface.

1. Open the app and add some notes.

1. Look at requests as they came through the ingress gateway - In the Jaeger UI select `Service=istio-ingressgateway`, `Operation=all` and click "Find Traces".

1. In the Jaeger UI you should see all requests that you've just sent. Open one of the requests. You should see all sub-requests that were sent in the context of main request (including request to the backend and request to the Istio internal components).

1. You can also look at requests at the application level. In the Jaegger UI select `Service=gceme`, `Operation=frontend.default.svc.cluster.local:80/add-note` and click "Find Traces".

1. Press `CTRL-C` to exit

# Deploy a v2 Frontend

1. Deploy a newer version of the frontend. Save this file as `manifests/sample-app.v2.yaml`

    This is a similar deployment as before with the following changes

    ```yaml
    ---
    apiVersion: v1
    kind: Service
    metadata:
      name: db-v2
      labels:
        app: gcme
        role: db
        version: v2
    spec:
      type: ClusterIP
      ports:
        - port: 3306
      selector:
        app: gceme
        role: db
        version: v2
    ---
    apiVersion: apps/v1
    kind: Deployment
    metadata:
      name: db-v2
      labels:
        app: gcme
        role: db
        version: v2
    spec:
      replicas: 1
      selector:
        matchLabels:
          app: gceme
          role: db
          version: v2
      template:
        metadata:
          name: db
          labels:
            app: gceme
            role: db
            version: v2
        spec:
          containers:
          - image: mysql:5.6
            name: mysql
            env:
            - name: MYSQL_ROOT_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: mysql
                  key: password
            ports:
            - containerPort: 3306
              name: mysql
    ---
    kind: Service
    apiVersion: v1
    metadata:
      name: backend-v2
      labels:
        app: gcme
        role: backend
        version: v2
    spec:
      type: ClusterIP
      ports:
      - name: http
        port: 8080
      selector:
        role: backend
        app: gceme
        version: v2
    ---
    apiVersion: apps/v1
    kind: Deployment
    metadata:
      name: backend-v2
      labels:
        app: gcme
        role: backend
        version: v2
    spec:
      replicas: 1
      selector:
        matchLabels:
          app: gceme
          role: backend
          version: v2
      template:
        metadata:
          name: backend
          labels:
            app: gceme
            role: backend
            version: v2
        spec:
          containers:
          - name: backend
            image: gcr.io/barry-williams/sample-k8s-app:v2
            env:
            - name: MYSQL_ROOT_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: mysql
                  key: password
            imagePullPolicy: Always
            command: ["sh", "-c", "app -run-migrations -mode=backend -run-migrations -port=8080 -db-host=db-v2 -db-password=$MYSQL_ROOT_PASSWORD" ]
            ports:
            - name: backend
              containerPort: 8080
    ---
    apiVersion: apps/v1
    kind: Deployment
    metadata:
      name: frontend-v2
      labels:
        app: gcme
        role: frontend
        version: v2
    spec:
      replicas: 1
      selector:
        matchLabels:
          app: gceme
          role: frontend
          version: v2
      template:
        metadata:
          name: frontend
          labels:
            app: gceme
            role: frontend
            version: v2
        spec:
          containers:
          - name: frontend
            image: gcr.io/barry-williams/sample-k8s-app:v2
            imagePullPolicy: Always
            command: ["app", "-mode=frontend", "-backend-service=http://backend-v2:8080", "-port=80"]
            ports:
            - name: frontend
              containerPort: 80
    ```

    ```shell
    kubectl apply -f manifests/sample-app.v2.yaml
    ```

## Add a Destination Rule

The destination rule will describe how to get to our new version.

1. Save and apply this Destination Rule as `manifests/frontend-dr.yaml`

    ```yaml
    apiVersion: networking.istio.io/v1alpha3
    kind: DestinationRule
    metadata:
      name: frontend-dr
    spec:
      host: frontend
      trafficPolicy:
        tls:
          mode: ISTIO_MUTUAL
      subsets:
      - name: v1
        labels:
          version: v1
      - name: v2
        labels:
          version: v2
    ```

    ```shell
    kubectl apply -f manifests/frontend-dr.yaml
    ```

    This rule identifies our multiple `frontend` versions.

# Traffic Mirroring

At this point, traffic in our browser should only go to the v1 app because of the virtual service we applied in the last section

1. Add a note to the application: "v1 before"

1. Overwrite the VirtualService for the frontend as `manifests/frontend-vs.yaml` and apply the changes.

    ```yaml
    apiVersion: networking.istio.io/v1alpha3
    kind: VirtualService
    metadata:
      name: frontend-vs
    spec:
      hosts:
      - "*"
      gateways:
      - gceme-gateway
      http:
      - route:
        - destination:
            host: frontend
            subset: v1
        mirror:
          host: frontend
          subset: v2
    ```

    ```shell
    kubectl apply -f manifests/frontend-vs.yaml
    ```

    All traffic will flow to v1, however v2 will receive a copy of each request. We will use our `End-User: barry` header to access our v2 app.

1. Go to the app and add the note "v1 mirror"

    In the app, you should see the notes we created:
    - v1 before
    - v1 mirror

Traffic mirroring does not allow us to access the v2 frontend through a browser. We will exec into the pod and run a local curl command instead.

1. Exec into the frontend-v2 pod and run curl

    ```shell
    kubectl exec -it `kubectl get pod -l app=gceme,role=frontend,version=v2 -o jsonpath='{.items[0].metadata.name}'` sh
    ```

    ```shell
    # pull back all the messages
    curl localhost | grep '<li class="collection-item">'
    ```

    What messages appear?

    You should see all the messages created since traffic mirroring began

# Traffic Shifting

Let's now see how Istio can help us to add new features to our application. Let's imagine that we want to add a new feature to the app and test it on a small percent of our users (also called a 'Canary deployment').

1. Generate some requests with `hey`

    ```shell
    hey -z 60m -c 10 -q 10 http://$GATEWAY_URL
    ```

    This will generate about 100 requests per second on the frontend

1. Overwrite the VirtualService for the frontend as `manifests/frontend-vs.yaml` and apply the changes.

    ```yaml
    apiVersion: networking.istio.io/v1alpha3
    kind: VirtualService
    metadata:
      name: frontend-vs
    spec:
      hosts:
      - "*"
      gateways:
      - gceme-gateway
      http:
      - route:
        - destination:
            host: frontend
            subset: v1
          weight: 75
        - destination:
            host: frontend
            subset: v2
          weight: 25
    ```

    ```shell
    kubectl apply -f manifests/frontend-vs.yaml
    ```

1. Open the app and refresh the page several times. You should see `v1` backend version in about 75% of cases and `v2` in 25%.

1. Goto Grafana to the Istio Workload Dashboard

1. Select the frontend-v1 workload

1. Now switch to the frontend-v2 workload

    What does Grafana tell you about the traffic load on v1 and v2?

# Traffic Steering

We will route a request based on a particular header value (`End-User: barry`)

Routes are managed on a virtual service, using the following elements:

  - `match` - Criteria about the request
  - `route` - Where to send a request when a match has been satisfied

1. Kill `hey` for this section. Use `CTRL-C`

1. Overwrite the VirtualService for the frontend as `manifests/frontend-vs.yaml` and apply the changes.

    This will route traffic according to the value of a header

    ```yaml
    apiVersion: networking.istio.io/v1alpha3
    kind: VirtualService
    metadata:
      name: frontend-vs
    spec:
      hosts:
      - "*"
      gateways:
      - gceme-gateway
      http:
      - match:
        - headers:
            end-user:
              exact: barry
        route:
        - destination:
            host: frontend
            subset: v2
      - route:
        - destination:
            host: frontend
            subset: v1
    ```

    ```shell
    kubectl apply -f manifests/frontend-vs.yaml
    ```

1. Lets hit our application as this user. The browser doesn't let us apply our own header, so we will make the call with curl.

    We will compare with version and color scheme.

    ```shell
    # With header
    curl http://$GATEWAY_URL/ -H "End-User: barry" | grep -E 'v1|v2' -a2
    curl http://$GATEWAY_URL/ -H "End-User: barry" | grep -E 'green|blue'

    # Without header
    curl http://$GATEWAY_URL/ | grep -E 'v1|v2' -a2
    curl http://$GATEWAY_URL/ | grep -E 'green|blue'
    ```

    You should see that the custom-header version shows it's version is v2, and that it is using the "green" color scheme.

1. Lets see if we can find our calls in Jaeger

    1. Set up a temporary tunnel the Jaeger dashboard by using port-forwarding.

    ```shell
    kubectl port-forward -n istio-system $(kubectl get pod -n istio-system -l app=jaeger -o jsonpath='{.items[0].metadata.name}') 16686:16686
    ```

    1. Open web preview at port `16686`, you should see the Jaeger interface.

    Which application versions were our curl commands sent to?

## Clean up v1 app

We will no longer use multiple app versions, so we will delete anything for v1

    ```shell
    kubectl delete destinationRule frontend-dr
    kubectl delete deployment frontend-v1 backend-v1 db-v1
    kubectl delete service backend-v1 db-v1
    ```

1. Overwrite the VirtualService for the frontend as `manifests/frontend-vs.yaml` and apply the changes.

    ```yaml
    apiVersion: networking.istio.io/v1alpha3
    kind: VirtualService
    metadata:
      name: frontend-vs
    spec:
      hosts:
      - "*"
      gateways:
      - gceme-gateway
      http:
      - route:
        - destination:
            host: frontend
            port:
              number: 80
    ```

    ```shell
    kubectl apply -f manifests/frontend-vs.yaml
    ```

# Fault Injection

One of the most difficult aspects of testing microservice applications is verifying that the application is resilient to failures. Each service should not assume that all its dependencies are available 100% of the time, instead it should be ready to handle any unexpected failure.

Usually people manually shut down application instances or block application ports in order to simulate failures, Istio provides us with a much better way: Fault Injection.

We will be injecting faults into the backend service.

1. Create a VirtualService for the backend as `manifests/backend-vs.yaml` and apply the changes.

    ```yaml
    apiVersion: networking.istio.io/v1alpha3
    kind: VirtualService
    metadata:
      name: backend-vs
    spec:
      hosts:
      - backend-v2
      http:
      - route:
        - destination:
            host: backend-v2
            port:
              number: 8080
        fault:
          delay:
            fixedDelay: 3s
            percent: 50
    ```

    ```shell
    kubectl apply -f manifests/backend-vs.yaml
    ```

1. Ensure `hey` is running

    ```shell
    hey -z 60m -c 1 -q 10 http://$GATEWAY_URL
    ```

1. Open the app and verify that in 50% of the times it should take 3 seconds to complete the request.

1. Open Grafana, open the Istio Service Dashboard, switch to service frontend.default.svc.cluster.local

    Can you explain the graph Client Request Duration (very top, 2nd to the right)

    Explain "Server Request Duration" (just below Client Request Duration)

In a similar way you can inject not only delays, but also failures.

Lets inject an http failure

1. Overwrite the VirtualService for the backend as `manifests/backend-vs.yaml` and apply the changes.

    ```yaml
    apiVersion: networking.istio.io/v1alpha3
    kind: VirtualService
    metadata:
      name: backend-vs
    spec:
      hosts:
      - backend-v2
      http:
      - route:
        - destination:
            host: backend-v2
            port:
              number: 8080
        fault:
          abort:
            httpStatus: 500
            percent: 50
    ```

    ```shell
    kubectl apply -f manifests/backend-vs.yaml
    ```

1. Go back to the Grafana Istio Service Dashboard for the backend-v2 service

    What changed?

    Look at Incoming Requests By Source and Response Code. How do the 500 errors compare in frequency as the 200's?

1. Cleanup

    ```shell
    kubectl delete VirtualService backend-vs
    ```

# Circuit Breaker

1. Run `hey`

  ```shell
  hey -z 60m -c 10 -q 10 http://$GATEWAY_URL
  ```

  This will run at 100 requests per second

1. Observe traffic in Grafana

    1. Set up a temporary tunnel to Grafana by using port-forwarding.

        ```shell
        kubectl -n istio-system port-forward $(kubectl -n istio-system get pod -l app=grafana -o jsonpath='{.items[0].metadata.name}') 3000:3000
        ```

        > Note: When you CTRL+C from this process, the tunnel will be closed and you will no longer be able to access the UI.

    1. Open Web Preview at port `3000`, you should see the Grafana interface.

    1. In the left panel click on `Dashboards -> Manage` and then select the `istio` folder.

    1. Select `Istio Service Dashboard`

    1. Observe the graph "Incoming Requests by Source and Response Code"


1. Scale up the backend to 3 pods

    ```shell
    kubectl scale deployment backend-v2 --replicas=3
    ```

1. Apply a circuit breaker to the backend

    ```yaml
    apiVersion: networking.istio.io/v1alpha3
    kind: DestinationRule
    metadata:
      name: backend-dr
    spec:
      host: backend-v2
      trafficPolicy:
        tls:
          mode: ISTIO_MUTUAL
        outlierDetection:
          consecutiveErrors: 1
          interval: 1s
          baseEjectionTime: 30s
          maxEjectionPercent: 100
    ```

Cause a backend pod to start throwing errors on every request

1. Pick a backend pod (Remember your selection)

    ```shell
    kubectl get pods -l role=backend
    ```

1. In a separate command window, follow the logs of that backend pod

    ```shell
    kubectl logs -f <backend pod name>
    ```

    You should see a group of requests every second.

    Keep this command window open watching logs

1. In a separate cloud shell window, run a curl command to tell the backend pod to start throwing errors

    ```shell
    kubectl exec <backend pod name> curl localhost:8080/start-error
    ```

1. Go back to the log output

    What is happening to the throughput?

    Do any requests get through?

    Why do we still occasionally see calls go through?

        The requests that came through are where the circuit breaker is testing our backend.  It will perform a time increasing backoff

1. Run a curl command to tell the backend pod to stop throwing errors

    ```shell
    kubectl exec <backend pod name> curl localhost:8080/end-error
    ```

1. Watch the logs again

    Do the requests resume immediately?

1. What behavior was seen in Grafana?

# Rate Limiting using Memquota Memory Store

1. Send traffic with hey

    This will run `hey` for only 30 seconds

    ```shell
    hey -z 60m -c 5 -q 10 http://$GATEWAY_URL
    ```

    We should expect around 50 requests per second to the frontend service

1. Observe traffic in Grafana

    1. Set up a temporary tunnel to Grafana by using port-forwarding.

        ```shell
        kubectl -n istio-system port-forward $(kubectl -n istio-system get pod -l app=grafana -o jsonpath='{.items[0].metadata.name}') 3000:3000
        ```

        > Note: When you CTRL+C from this process, the tunnel will be closed and you will no longer be able to access the UI.

    1. Open Web Preview at port `3000`, you should see the Grafana interface.

    1. In the left panel click on `Dashboards -> Manage` and then select the `istio` folder.

    1. Select `Istio Service Dashboard`

    1. Observe the graph "Incoming Requests by Source and Response Code"

        It should be at about 50 requests per second

1. Add rate limiting to the frontend. Save this file as `manifests/rate-limit.yaml`

    ```yaml
    apiVersion: "config.istio.io/v1alpha2"
    kind: memquota
    metadata:
      name: handler
      namespace: default
    spec:
      quotas:
      - name: frontend-quota.quota.default
        maxAmount: 10
        validDuration: 1s
        overrides:
        - dimensions:
            destination: frontend
          maxAmount: 1
          validDuration: 1s
    ---
    apiVersion: "config.istio.io/v1alpha2"
    kind: quota
    metadata:
      name: frontend-quota
      namespace: default
    spec:
      dimensions:
        source: source.labels["app"] | source.service | "unknown"
        destination: destination.labels["app"] | destination.service | "unknown"
    ---
    apiVersion: "config.istio.io/v1alpha2"
    kind: rule
    metadata:
      name: quota
      namespace: default
    spec:
      actions:
      - handler: handler.memquota
        instances:
        - frontend-quota.quota
    ---
    apiVersion: config.istio.io/v1alpha2
    kind: QuotaSpec
    metadata:
      creationTimestamp: null
      name: front-end
      namespace: default
    spec:
      rules:
      - quotas:
        - charge: 1
          quota: frontend-quota
    ---
    apiVersion: config.istio.io/v1alpha2
    kind: QuotaSpecBinding
    metadata:
      creationTimestamp: null
      name: front-end
      namespace: default
    spec:
      quotaSpecs:
      - name: front-end
        namespace: default
      services:
      - name: frontend
        namespace: default
    ```

    ```shell
    kubectl apply -f manifests/rate-limit.yaml
    ```

1. Go back to the Grafana graph

    What is happening?

    Can you explain the changes in the latency graph (Server Request Duration)?

# Rate Limiting using Redis Memory Store

1. Start a Redis Cluster. Save this file as `manifests/redis.yaml` and apply.

    ```yaml
    apiVersion: extensions/v1beta1
    kind: Deployment
    metadata:
      name: redis-master
      namespace: default
    spec:
      replicas: 1
      template:
        metadata:
          labels:
            app: redis
            role: master
            tier: backend
        spec:
          containers:
          - name: master
            image: gcr.io/google_containers/redis:e2e
            resources:
              requests:
                cpu: 100m
                memory: 100Mi
            ports:
            - containerPort: 6379
    ---
    apiVersion: extensions/v1beta1
    kind: Deployment
    metadata:
      name: redis-slave
      namespace: default
    spec:
      replicas: 2
      template:
        metadata:
          labels:
            app: redis
            role: slave
            tier: backend
        spec:
          containers:
          - name: slave
            image: gcr.io/google_samples/gb-redisslave:v1
            resources:
              requests:
                cpu: 100m
                memory: 100Mi
            env:
            - name: GET_HOSTS_FROM
              value: dns
            ports:
            - containerPort: 6379
    ---
    apiVersion: v1
    kind: Service
    metadata:
      name: redis-master
      namespace: default
      labels:
        app: redis
        tier: backend
        role: master
    spec:
      ports:
      - port: 6379
        targetPort: 6379
      selector:
        app: redis
        tier: backend
        role: master
    ---
    apiVersion: v1
    kind: Service
    metadata:
      name: redis-slave
      namespace: default
      labels:
        app: redis
        tier: backend
        role: slave
    spec:
      ports:
      - port: 6379
      selector:
        app: redis
        tier: backend
        role: slave
    ```

    ```shell
    kubectl apply -f manifests/redis.yaml
    ```

1. Send traffic with hey

    This will run `hey` for only 30 seconds

    ```shell
    hey -z 60m -c 5 -q 10 http://$GATEWAY_URL
    ```

    We should expect around 50 requests per second to the frontend service

1. Observe traffic in Grafana

    1. Set up a temporary tunnel to Grafana by using port-forwarding.

        ```shell
        kubectl -n istio-system port-forward $(kubectl -n istio-system get pod -l app=grafana -o jsonpath='{.items[0].metadata.name}') 3000:3000
        ```

        > Note: When you CTRL+C from this process, the tunnel will be closed and you will no longer be able to access the UI.

    1. Open Web Preview at port `3000`, you should see the Grafana interface.

    1. In the left panel click on `Dashboards -> Manage` and then select the `istio` folder.

    1. Select `Istio Service Dashboard`

    1. Observe the graph "Incoming Requests by Source and Response Code"

        It should be at about 50 requests per second

1. Add rate limiting to the frontend. Save this file as `manifests/redis-rate-limit.yaml`

    ```yaml
    apiVersion: "config.istio.io/v1alpha2"
    kind: redisquota
    metadata:
      name: handler
      namespace: defalt
    spec:
      redisServerUrl: redis-master.istio-system.svc.cluster.local:6379
      connectionPoolSize: 10
      quotas:
      - name: frontend-quota.quota.default
        maxAmount: 10
        validDuration: 1s
        bucketDuration: 1s
        rateLimitAlgorithm: ROLLING_WINDOW
        overrides:
        - dimensions:
            destination: frontend
          maxAmount: 1
          validDuration: 1s
    ---
    apiVersion: "config.istio.io/v1alpha2"
    kind: quota
    metadata:
      name: frontend-quota
      namespace: default
    spec:
      dimensions:
        source: source.labels["app"] | source.service | "unknown"
        destination: destination.labels["app"] | destination.service | "unknown"
    ---
    apiVersion: "config.istio.io/v1alpha2"
    kind: rule
    metadata:
      name: quota
      namespace: default
    spec:
      actions:
      - handler: handler.redisquota
        instances:
        - frontend-quota.quota
    ---
    apiVersion: config.istio.io/v1alpha2
    kind: QuotaSpec
    metadata:
      creationTimestamp: null
      name: front-end
      namespace: default
    spec:
      rules:
      - quotas:
        - charge: 1
          quota: frontend-quota
    ---
    apiVersion: config.istio.io/v1alpha2
    kind: QuotaSpecBinding
    metadata:
      creationTimestamp: null
      name: front-end
      namespace: default
    spec:
      quotaSpecs:
      - name: front-end
        namespace: default
      services:
      - name: frontend
        namespace: default
    ```

    ```shell
    kubectl apply -f manifests/redis-rate-limit.yaml
    ```

1. Go back to the Grafana graph

    What is happening?

    Can you explain the changes in the latency graph (Server Request Duration)?
