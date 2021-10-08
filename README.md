# rhossm-grpc-example
Greeter server and client examples using RHOSSM in OCP. In this example, the greeter server is deployed outside the mesh and the greeter client inside the mesh, so the client will use the egress gateway to reach the greeter server.


## Prerequisites
 - OCP 4.6 or higher.
 - Openshift Service Mesh up and running.
 - OC cli installed.
 - OCP configured for supporting HTTP/2.


## Generate certificates
In this use case, the greeter server and client will be deployed using mTLS mode. Update the file [Generate certs](./certs/generate_certs.sh) and run the script:

```
./certs/generate_certs.sh
```

## Deploy greeter server
Deploy the greeter server in OCP.

The container image must exist in the OCP registry. Here, you can do it as you prefer. Keep in mind that you must update the deployment yaml file with the image name.

Create _grpc-example_ project
```
oc new-project grpc-example
```

Create the server certificates
```
oc create secret generic greeter-server-certs -n grpc-example --from-file=./certs/server.pem --from-file=./certs/server.key --from-file=./certs/ca.pem
```

Deploy the greeter server
```
oc apply -n grpc-example -f greeter-server/greeter-server-mtls.yaml
```

## Deploy greeter client
Deploy the greeter client in OCP.

Repeat the steps from the previous point in order to create and upload the greeter client container image.

### Greeter client in the same namespace as greeter server
You can run this step in order to test the greeter client and server, and the certificates created at the beginning.

Create the client certificates
```
oc create secret generic greeter-client-certs -n grpc-example --from-file=./certs/client.pem --from-file=./certs/client.key --from-file=./certs/ca.pem
```

Deploy the greeter client
```
oc apply -n grpc-example -f greeter-client/greeter-client-mtls.yaml
```

In the pod _greeter-client-mtls-*_, you can check that the greeter-client is authenticated an the server responses "hello":
```
Greeting: Hello you (authenticated as "greeter-client.grpc-example.svc.cluster.local" via mTLS)
Greeting: Hello you (authenticated as "greeter-client.grpc-example.svc.cluster.local" via mTLS)
Greeting: Hello you (authenticated as "greeter-client.grpc-example.svc.cluster.local" via mTLS)
Greeting: Hello you (authenticated as "greeter-client.grpc-example.svc.cluster.local" via mTLS)
Greeting: Hello you (authenticated as "greeter-client.grpc-example.svc.cluster.local" via mTLS)
```

Delete the greeter client
```
oc delete -n grpc-example -f greeter-client/greeter-client-mtls.yaml
```

Delete the client certificates
```
oc delete secret greeter-client-certs -n grpc-example
```

### Greeter client using Openshift Service Mesh
Now, the greeter client will be deployed in the Service Mesh. To connect to the greeter server via mTLS, we will use an egress gateway.

Create _grpc-example-mesh_ project
```
oc new-project grpc-example-mesh
```

Add the _grpc-example-mesh_ namespace to the Service Mesh
```
oc apply -f ossm/grpc-example-mesh/smm.yaml
```

Deploy the greeter client
```
oc apply -n grpc-example-mesh -f greeter-client/greeter-client-mtls_ossm.yaml
```

The greeter client is not able to connect to the server if you check the pod's log. It's time to create the Istio objects.

Create the client certificates, but in this case the egress gateway will use them
```
oc create secret generic greeter-client-certs -n istio-system --from-file=./certs/client.pem --from-file=./certs/client.key --from-file=./certs/ca.pem
```

Configure the egress gateway to use the certificates. The SMCP should be configured like this:
```
        volumes:
        - volume:
            secret:
              secretName: greeter-client-certs
          volumeMount:
            mountPath: /etc/istio/greeter-client-certs
            name: greeter-client-certs
```

You can check Kiali to see how the traffic flow is changing when creating the Istio objects. At this moment, the greeter-client is redirect to BlackHoleCluster.

<img src="./ossm/images/1-greeter-client.png" alt="Kiali-1" width=40%>

Create the Virtual Service to route the traffic from the greeter-client app to the egress gateway K8S Service.
```
oc apply -f ossm/grpc-example-mesh/vs-greeter-server.yaml
```

Now, the greeter-client is redirect to the egress gateway K8S Service but it is not able to connect.

<img src="./ossm/images/2-greeter-client.png" alt="Kiali-2" width=40%>

Apply the Destination Rule to use ISTIO_MUTUAL tls negotiation between the greeter client app and the egress gateways K8S Service.
```
oc apply -f ossm/grpc-example-mesh/dr-greeter-server.yaml
```

The greeter-client is redirect to the egress gateway K8S Service and connecting using ISTIO_MUTUAL.

<img src="./ossm/images/3-greeter-client.png" alt="Kiali-3" width=40%>

Create the Virtual Service, Service Entry and Gateway in _istio-system_ namespace.
```
oc apply -f ossm/istio-system/gw-greeter-server.yaml
oc apply -f ossm/istio-system/se-greeter-server.yaml
oc apply -f ossm/istio-system/vs-greeter-server.yaml
```

At this point, the traffic is being redirected from the greeter-client app to the egress gateway, and then to the greeter-server. But, as you can see in Kiali, the greeter-client is still unable to connect.

<img src="./ossm/images/4-greeter-client.png" alt="Kiali-4" width=30%>

In Kiali we can see that the GRPC code is 14 and the message "Check DestinationRule or VirtualService)

<img src="./ossm/images/5-greeter-client.png" alt="Kiali-5" width=20%>

Apply the Destination Rule to use the greeter-client certificates in the egress gateway.
```
oc apply -f ossm/istio-system/dr-greeter-server.yaml
```

Finally, the greeter-client is able to connect to the greeter-server using the egress gateway.

<img src="./ossm/images/6-greeter-client.png" alt="Kiali-5" width=40%>

In the pod _greeter-client-mtls-*_, you can check that the greeter-client is authenticated an the server responses "hello" again:
```
Greeting: Hello you (authenticated as "greeter-client.grpc-example.svc.cluster.local" via mTLS)
Greeting: Hello you (authenticated as "greeter-client.grpc-example.svc.cluster.local" via mTLS)
Greeting: Hello you (authenticated as "greeter-client.grpc-example.svc.cluster.local" via mTLS)
```