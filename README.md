# Plugin POC

## Quick start

1. Create the `ambassador-pro-registry-credentials` secret in your cluster, if you haven't done so already.
2. Add your license key to the `ambassador-pro.yaml` file.
3. Apply all the YAML in the cluster:
   
   ```
   kubectl apply -f httpbin.yaml
   kubectl apply -f ambassador-pro.yaml
   kubectl apply -f ambassador-pro-auth.yaml
   kubectl apply -f ambassador-service.yaml
   kubectl apply -f dynamic-route.yaml
   ```

4. Get the IP address of your LoadBalancer: `kubectl get svc ambassador`

5. `curl` to the load balancer to test different variables:

   ```
   curl $IP/test/?db=1
   curl $IP/test/?db=2
   ```

   Note that if the value is odd, you will go to httpbin.org. If the value is even, you will go to Google.com.

### Behind the scenes

* A Golang plugin is looking at the request ID, and setting an HTTP header called `X-Dc` to Odd or Even
* In the `httpbin.yaml`, we create two mappings. One mapping maps to `X-Dc: Odd` and routes to httpbin.org. The other is a fallback mapping that routes to Google.com.
* The source code to the Golang plugin is in https://github.com/datawire/apro-example-plugin (look at `param-plugin.go`).
* Updating the plug-in involves making changes to the source code, `make DOCKER_REGISTRY=...`, and then updating the `ambassador-pro.yaml` sidecar to point to the new image

## Consul Connect Integration

1. Have Consul Connect installed in your cluster
2. Install the Consul Connect integration and QoTM service for testing mTLS with Connect sidecar

    ```
    kubectl apply -f consul-connect/
    ```

3. Get the IP address of your LoadBalancer: `kubectl get svc ambassador`

4. `curl` to the load balancer to test different variables:

   ```
   $ curl $IP/qotm/
   {"hostname":"qotm-794f5c7665-8fntp","ok":true,"quote":"668: The Neighbor of the Beast.","time":"2019-02-19T17:48:33.922354","version":"1.3"}
   ```

### Behind the scenes

* Ambassador is talking with the Consul API and grabbing the TLS certificate it is using for mTLS
* Ambassador then creates a secret called `ambassador-consul-connect` holding this TLS certificate
* Ambassador is told to use that secret when talking to the QoTM service with a `TLSContext` and the `tls` `Mapping` attribute