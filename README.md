# Plugin POC

This POC will install Consul Connect, Ambassador, Prometheus, Grafan, and Zipkin, along with some demo services.

## Initial setup

1. If you're on GKE, make sure you have an admin cluster role binding:

```
kubectl create clusterrolebinding my-cluster-admin-binding --clusterrole=cluster-admin --user=$(gcloud info --format="value(config.account)")
```

2. Clone this repository locally:

```
git clone https://github.com/datawire/plugin-poc.git
```

3. Create the `ambassador-pro-registry-credentials` secret in your cluster, if you haven't done so already.

4. Add your license key to the `ambassador-pro.yaml` file.

5. Initialize Helm with the appropriate permissions.

```
kubectl apply -f helm-rbac.yaml
helm init --service-account=tiller
```

## Set up Istio

We will install Istio with mTLS enabled via `kubectl`.

First, install the Istio `CustomResourceDefinitions`:

```
kubectl apply -f istio/crds.yaml
```

Next install istio with mTLS enabled:

```
kubectl apply -f isitio/istio-install.yaml
```

### Verify the Istio Installation

We're going to test our Istio installation with the example bookinfo application.

```shell
kubectl apply -f istio/bookinfo.yaml
```

This creates a couple of services for testing the service mesh functions. See the [bookinfo example documentation](https://istio.io/docs/examples/bookinfo/) for more information.

We will issue a request through the Istio gateway to verify the application and Istio are running properly.

First, get the IP address of the Istio gateway

```shell
export ISTIO_IP=$(kubectl -n istio-system get service istio-ingressgateway -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
```

Then send a request to the bookinfo application.

```shell
curl -v http://$ISTIO_IP/productpage
```

If the request returns a 200 then Istio is running properly. 


## Install Ambassador

1. Install Ambassador with the following commands, along with the demo QOTM service and a route to HTTPbin service.
   
   ```shell
   kubectl apply -f statsd-sink.yaml
   kubectl apply -f ambassador/ambassador-pro-istio.yaml
   kubectl apply -f ambassador-service.yaml
   kubectl apply -f httpbin.yaml
   ```

2. Get the IP address of Ambassador: 

   ```shell
   AMBASSADOR_IP=$(kubectl get service ambassador -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
   ```

3. Verify Ambassador is running by sending a request to the external `httpbin` service
   ```shell
   curl -v http://$AMBASSADOR_IP/httpbin/ip
   {
      "origin": "108.20.119.124, 35.184.242.212, 108.20.119.124"
   }
   ```
4. Now send a request to the bookinfo application running in the service mesh
   ```shell
   curl -v http://$AMBASSADOR_IP/productpage
   ```

   Ambassador is able to proxy requests to the bookinfo application by encrypting those requests with the certificates provided by Istio. Visit the [Istio documentation](https://www.getambassador.io/user-guide/with-istio) for more information on how this is configured.

5. Since we have now verified Ambassador can proxy requests to services running in the Istio service mesh, we can delete the Istio gateway service to save us the cost of a LoadBalancer. 
   ```shell
   kubectl delete svc -n isitio-system istio-ingressgateway
   ```

## Metrics

Next, we'll set up metrics using Prometheus and Grafana.

1. Install the Prometheus Operator.

   ```
   kubectl apply -f monitoring/prometheus.yaml
   ```

2. Wait 30 seconds until the `prometheus-operator` pod is in the `Running` state.

3. Create the rest of the monitoring setup:

   ```
   kubectl apply -f monitoring/prom-cluster.yaml
   kubectl apply -f monitoring/prom-svc.yaml
   kubectl apply -f monitoring/servicemonitor.yaml
   kubectl apply -f monitoring/grafana.yaml
   ```

4. Send some traffic through Ambassador (metrics won't appear until some traffic is sent). You can just run the `curl` command to httpbin above a few times.

5. Get the IP address of Grafana: `kubectl get svc grafana`

6. In your browser, go to the `$GRAFANA_IP` and log in using username `admin`, password `admin`.

7. Configure Prometheus as the Grafana data source. Give it a name, choose type Prometheus, and point the HTTP URL to `http://prometheus.default:9090`. Save & Test the Data Source.

8. Import a dashboard. Click on the + button, and then choose Import. Upload the `ambassador-dashboard.json` file to Grafana. Choose the data source you created in the previous step, and click import.

9. Go to the Ambassador dashboard!

## Distributed Tracing

We will now set up distributed tracing using Zipkin.

1. Create the `TracingService`

    ```
    kubectl apply -f tracing/zipkin.yaml
    kubectl apply -f tracing/tracing-config.yaml
    ```
  
    This will tell Ambassador to generate a tracing header for all requests through it. Ambassador needs to be restarted for this configuration to take effect.

2. Restart Ambassador to reload the Tracing configuration.

    - Get your Ambassador Pod name

       ```
       kubectl get pods

       ambassador-7bbd676d59-7b8w6                                   2/2     Running   0          10m
       ambassador-pro-consul-connect-integration-6d7d489b4b-fwndt    1/1     Running   0          4h
       ambassador-pro-redis-6cbb7dfbb-pzg66                          1/1     Running   0          4h
       consul-76ks4                                                  1/1     Running   0          4h
       consul-connect-injector-webhook-deployment-7846847f9f-r8w8p   1/1     Running   0          4h
       consul-p89q4                                                  1/1     Running   0          4h
       consul-server-0                                               1/1     Running   0          4h
       consul-server-1                                               1/1     Running   0          4h
       consul-server-2                                               1/1     Running   0          4h
       consul-vvl7b                                                  1/1     Running   0          4h
       qotm-794f5c7665-26bf9                                         2/2     Running   0          4h
       zipkin-98f9cbc58-zjksk                                        1/1     Running   0          7m
       ````

    - Deleting the pod will tell Kubernetes to restart another one

       ```
       kubectl delete po ambassador-7bbd676d59-7b8w6 
       pod "ambassador-7bbd676d59-7b8w6" deleted
       ```

    - Wait for the new Pod to come online

3. Apply the micro donuts demo service

    ```
    kubectl apply -f tracing/microdonut.yaml
    ```

4. Test the tracing service

    - From a web-browser, go to http://{AMBASSADOR_IP}/microdonut/
    - Use the UI to select and order a number of donuts
    - After clicking `order`, from a new tab, access http://{AMBASSADOR_IP}/zipkin/
    - In the search parameter box, expand the `Limit` to 1000 so you can see all of the traces
    - Click `Find Traces` and you will see a list of Traces for requests through Ambassador.
    - Find a trace that has > 2 spans and you will see a trace for all the request our donut order made

## Key Takeaways (TODO)

* We're dynamically registering and resolving routes for services, e.g., in `consul-connect/qotm.yaml` and `httpbin.yaml`, new services are dynamically registered with Ambassador. Ambassador then uses Kubernetes DNS to resolve the actual IP address of these services.
* Ambassador is processing inbound (North/South) requests to the mesh, and dynamically processing URL variables to determine where a given request should be routed.
* Ambassador automatically obtains certificates from Consul Connect, and uses these certificate to originate encrypted TLS connections to target services in the mesh.
* Prometheus is collecting high resolution metrics from Ambassador. A sample dashboard of some of these metrics is displayed in Grafana.

