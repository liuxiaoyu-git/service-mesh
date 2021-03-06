# Secure Service with Mutual TLS



## Setup

```
oc apply -f ocp/frontend-v1-deployment.yml -n $USERID
oc apply -f ocp/frontend-service.yml -n $USERID
oc apply -f ocp/frontend-route.yml -n $USERID
oc apply -f ocp/backend-v1-service.yml -n $USERID
oc apply -f ocp/backend-v2-service.yml -n $USERID
oc apply -f ocp/backend-service.yml -n $USERID
```

Create another pod without sidecar for testing
```
oc create -f ocp/station-deployment.yml -n $USERID
```

Verify that deployment of station

```
...
annotations:
        sidecar.istio.io/inject: "false"
...
```

Check that station pod has no sidecar

```
oc get pod
```

Sample output. station pod is 1/1

```

NAME                          READY   STATUS    RESTARTS   AGE
backend-v1-6ddf9c7dcf-vlcr9   2/2     Running   2          18h
backend-v2-7655885b8c-rt4jz   2/2     Running   2          3h25m
frontend-v1-655f4478c-7b5dw   2/2     Running   2          18h
station-598ffcf464-8tkqg      1/1     Running   0          66s
```

Test by using station pod to connect to backend pod

```

oc exec <station pod> curl http://backend:8080
```

Sample Outout

```

Backend version:v1,Response:200,Host:backend-v1-6ddf9c7dcf-vlcr9, Message: Hello World!!
```

## Enable Mutual TLS for Backend Service
Review the following Istio's authenticaiton rule configuration file [authentication-backend-enable-mtls.yml](../istio-files/authentication-backend-enable-mtls.yml)  to enable authenticaion with following configuration.

```
...
spec:
  targets:
  - name: backend
    ports:
    - number: 8080
  peers:
  - mtls: {}
...
```

Review the following Istio's destination rule configuration file [destination-rule-backend-v1-v2-mtls.yml](../istio-files/destination-rule-backend-v1-v2-mtls.yml)  to enable client side certification (Mutual TLS) with following configuration.

```
...
  trafficPolicy:
      loadBalancer:
        simple: ROUND_ROBIN
      tls:
        mode: ISTIO_MUTUAL
...
```

Apply authentication, destination rule and virtual service to backend service

```
oc apply -f istio-files/destination-rule-backend-v1-v2-mtls.yml -n $USERID
oc apply -f istio-files/virtual-service-backend-v1-v2-50-50.yml -n $USERID
oc apply -f istio-files/authentication-backend-enable-mtls.yml -n $USERID
```

Sample output

```
destinationrule.networking.istio.io/backend created
virtualservice.networking.istio.io/backend-virtual-service created
policy.authentication.istio.io/authentication-backend-mtls created
```

Test with oc exec again from station pod

```
oc exec <station pod> curl http://backend:8080
```

Sample output

```
curl: (56) Recv failure: Connection reset by peer
command terminated with exit code 56
```

Because station pod is not part of Service Mesh then authentication is failed.

Test again with oc exec from frontend pod

```
oc exec <frontend pod> -c frontend curl http://backend:8080
```

Sample output

```
Backend version:v1,Response:200,Host:backend-v1-6ddf9c7dcf-vlcr9, Message: Hello World!!
```
Because frontend pod is part of Service Mesh then authentication is sucessed.

## Enable Mutual TLS for Frontend Service
Same as previous step for enable Mutual TLS to backend service. Enable mTLS with following command

```
oc apply -f istio-files/authentication-frontend-enable-mtls.yml -n $USERID
oc apply -f istio-files/destination-rule-frontend-mtls.yml -n $USERID
oc apply -f istio-files/virtual-service-frontend.yml -n $USERID
```

Sample output
```
policy.authentication.istio.io/authentication-frontend-mtls created
destinationrule.networking.istio.io/frontend created
virtualservice.networking.istio.io/frontend created
```

Test with oc exec again from station pod

```
oc exec <station pod> curl http://frontend:8080
```

Sample output

```
curl: (56) Recv failure: Connection reset by peer
command terminated with exit code 56
```

Because station pod is not part of Service Mesh then authentication is failed.

Test again with oc exec from backend pod

```
oc exec <backend pod> -c backend curl http://frontend:8080
```

Sample output

```
Frontend version: v1 => [Backend: http://backend:8080, Response: 200, Body: Backend version:v2,Response:200,Host:backend-v2-7655885b8c-rt4jz, Message: Hello World!!]
```

Because backend pod is part of Service Mesh then authentication is sucessed.

Test with cURL to OpenShift's Router

```
curl -v $FRONTEND_URL
```

You will get following error because OpenShift's router is not part of Service Mesh then router pod cannot authenticate to frontend

```
curl: (52) Empty reply from server
```

This can be solved by create Istio Ingress Gateway and connect to frontend with Istio Ingress Gateway.

```
oc apply -f istio-files/frontend-gateway.yml -n $USERID
```

Test again with cURL

```
curl $GATEWAY_URL
```

Sample output

```
Frontend version: v1 => [Backend: http://backend:8080, Response: 200, Body: Backend version:v2,Response:200,Host:backend-v2-7655885b8c-rt4jz, Message: Hello World!!]
```