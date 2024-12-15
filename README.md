## In-mesh L4 traffic management in the Istio ambient mode

There is a known bug in the Istio ambient mode, where L4 traffic management fails to work inside a mesh, i.e. in the so-called the east/west direction ([Issue 54119](https://github.com/istio/istio/issues/54119)). More specifically, after an application developer creates such a TCP route, the waypoint concerned refuses to honour the traffic rule, though it is obliged by design to manage both the L4 and L7 communucation flows. 

Let's walk through an example of reproducing the bug in an attempt to shift the L4 traffic between two TCP echo service versions. After that, we will introduce a handly but dirty hack to force the waypoint to honour a specific TCP route by patching one of the listener filters through the EnvoyFilter API.

![L4 Traffic Management in Istio Ambient Mesh](Istio_ambient_east-west_L4.png)

### Limitation

In this example, we use the Kubernetes Gateway API to test out TCP traffic shifting in an Istio ambient mesh. It can be shown elsewhere that the same bug will also happen to the Istio APIs of VirtualService, Gateway and the likes. The crux of the problem is that the L4 traffic management logic simply fails to register with waypoint, whether configued via the Kubernetes Gateway API or Istio APIs.

### Bug reproduction

The use case refers to the task outlined in the Istio doc, [TCP Traffic Shifting](https://istio.io/latest/docs/tasks/traffic-management/tcp-traffic-shifting/). 

1. Assume the Isto version is 1.24.0. First and foremost, ensure the Istio ambient mode is installed with the flag, --set values.pilot.env.PILOT_ENABLE_ALPHA_GATEWAY_API=true.
```
wget https://github.com/istio/istio/releases/download/1.24.0/istio-1.24.0-linux-amd64.tar.gz
curl -L https://istio.io/downloadIstio | sh -
cd istio-1.24.0
export PATH=$PWD/bin:$PATH
istioctl install --set values.pilot.env.PILOT_ENABLE_ALPHA_GATEWAY_API=true --set profile=ambient --skip-confirmation
```

2. Create and label the name space under test.
```
kubectl create namespace istio-io-tcp-traffic-shifting
kubectl label namespace istio-io-tcp-traffic-shifting istio.io/dataplane-mode=ambient
```

3. Create and label the name space under test.
```
kubectl create namespace istio-io-tcp-traffic-shifting
kubectl label namespace istio-io-tcp-traffic-shifting istio.io/dataplane-mode=ambient
```

4. Deploy the K8s workload and service objects for the TCP echo server and client from the bundled sample.
```
kubectl -n istio-io-tcp-traffic-shifting apply -f samples/curl/curl.yaml
kubectl -n istio-io-tcp-traffic-shifting apply -f samples/tcp-echo/tcp-echo-services.yaml
```

5. Observe that the client requests for the plain, TCP route-free K8s service, tcp-echo, are randomly distribured beteen the v1 and v2 pods.
```
keyuser@ubunclone:~/istio-1.24.0$ kubectl -n istio-io-tcp-traffic-shifting exec deploy/curl -- sh -c "while true
do
        date | nc tcp-echo 9000
        sleep 2
done"
two Wed Dec  4 00:14:09 UTC 2024
one Wed Dec  4 00:14:11 UTC 2024
one Wed Dec  4 00:14:13 UTC 2024
two Wed Dec  4 00:14:15 UTC 2024
one Wed Dec  4 00:14:17 UTC 2024
two Wed Dec  4 00:14:20 UTC 2024
```

6. Deploy the K8s services tcp-echo-v1 & tcp-echo-v2, Istio ingress (north/west) gateway tcp-echo-gateway and tcp route tcp-echo. The tcp route is intended to shift all the tcp echo traffic to v1. Its frontend field, namely parentRefs, is set to tcp-echo-gateway.
```
kubectl -n istio-io-tcp-traffic-shifting apply -f samples/tcp-echo/gateway-api/tcp-echo-all-v1.yaml
```
```
keyuser@ubunclone:~/istio-1.24.0$ kubectl -n istio-io-tcp-traffic-shifting get gtw
NAME               CLASS   ADDRESS       PROGRAMMED   AGE
tcp-echo-gateway   istio   172.18.0.11   True         20s
keyuser@ubunclone:~/istio-1.24.0$
keyuser@ubunclone:~/istio-1.24.0$ kubectl -n istio-io-tcp-traffic-shifting get tcproute
NAME       AGE
tcp-echo   24s
keyuser@ubunclone:~/istio-1.24.0$
keyuser@ubunclone:~/istio-1.24.0$ kubectl -n istio-io-tcp-traffic-shifting get svc
NAME                     TYPE           CLUSTER-IP      EXTERNAL-IP   PORT(S)                           AGE
curl                     ClusterIP      10.96.255.92    <none>        80/TCP                            76m
tcp-echo                 ClusterIP      10.96.163.188   <none>        9000/TCP,9001/TCP                 76m
tcp-echo-gateway-istio   LoadBalancer   10.96.59.54     172.18.0.11   15021:30471/TCP,31400:32241/TCP   67m
tcp-echo-v1              ClusterIP      10.96.129.103   <none>        9000/TCP                          67m
tcp-echo-v2              ClusterIP      10.96.74.70     <none>        9000/TCP                          67m
keyuser@ubunclone:~/istio-1.24.0$
keyuser@ubunclone:~/istio-1.24.0$ kubectl -n istio-io-tcp-traffic-shifting get pod -o wide
NAME                                      READY   STATUS    RESTARTS   AGE     IP            NODE              NOMINATED NODE   READINESS GATES
curl-6f9bc47956-2l95k                     1/1     Running   0          76m   10.244.2.8    ambient-worker    <none>           <none>
tcp-echo-gateway-istio-57844964f7-sl2j2   1/1     Running   0          67m   10.244.1.5    ambient-worker2   <none>           <none>
tcp-echo-v1-5f8dd78684-bwvqk              1/1     Running   0          76m   10.244.2.9    ambient-worker    <none>           <none>
tcp-echo-v2-794b5ff9c7-mwpbx              1/1     Running   0          76m   10.244.2.10   ambient-worker    <none>           <none>
```

7. For the purpose of comparing with the east-west traffic test later, observe that tcp-echo-gateway applies the tcp-echo routing rule to distributes all the tcp-echo client requests to v1.
```
keyuser@ubunclone:~$ kubectl -n istio-io-tcp-traffic-shifting exec deploy/curl -- sh -c "while true
do
        date | nc tcp-echo-gateway-istio 
        sleep 2
done"
one Wed Dec  4 23:21:19 UTC 2024
one Wed Dec  4 23:21:21 UTC 2024
one Wed Dec  4 23:21:23 UTC 2024
one Wed Dec  4 23:21:25 UTC 2024
one Wed Dec  4 23:21:27 UTC 2024
one Wed Dec  4 23:21:29 UTC 2024
one Wed Dec  4 23:21:31 UTC 2024
one Wed Dec  4 23:21:33 UTC 2024
one Wed Dec  4 23:21:35 UTC 2024
one Wed Dec  4 23:21:37 UTC 2024
```

8. Deploy a waypoint proxy in the namespace concerned. It will function as a transparent in-mesh gateway to manage the east-west traffic on both L4 and L7.
```
istioctl waypoint apply -n istio-io-tcp-traffic-shifting --enroll-namespace --overwrite --wait
```
```
keyuser@ubunclone:~/istio-1.24.0$ kubectl -n istio-io-tcp-traffic-shifting get gtw                            NAME               CLASS            ADDRESS        PROGRAMMED   AGE
tcp-echo-gateway   istio            172.18.0.11    True         67m
waypoint           istio-waypoint   10.96.116.49   True         51m
keyuser@ubunclone:~/istio-1.24.0$
keyuser@ubunclone:~/istio-1.24.0$ kubectl -n istio-io-tcp-traffic-shifting get tcproute
NAME          AGE
tcp-echo      67m
keyuser@ubunclone:~/istio-1.24.0$
keyuser@ubunclone:~/istio-1.24.0$ kubectl -n istio-io-tcp-traffic-shifting get svc
NAME                     TYPE           CLUSTER-IP      EXTERNAL-IP   PORT(S)                           AGE
curl                     ClusterIP      10.96.255.92    <none>        80/TCP                            76m
tcp-echo                 ClusterIP      10.96.163.188   <none>        9000/TCP,9001/TCP                 76m
tcp-echo-gateway-istio   LoadBalancer   10.96.59.54     172.18.0.11   15021:30471/TCP,31400:32241/TCP   67m
tcp-echo-v1              ClusterIP      10.96.129.103   <none>        9000/TCP                          67m
tcp-echo-v2              ClusterIP      10.96.74.70     <none>        9000/TCP                          67m
waypoint                 ClusterIP      10.96.116.49    <none>        15021/TCP,15008/TCP               51m
keyuser@ubunclone:~/istio-1.24.0$
keyuser@ubunclone:~/istio-1.24.0$ kubectl -n istio-io-tcp-traffic-shifting get pods -o wide
NAME                                      READY   STATUS    RESTARTS   AGE   IP            NODE              NOMINATED NODE   READINESS GATES
curl-6f9bc47956-2l95k                     1/1     Running   0          76m   10.244.2.8    ambient-worker    <none>           <none>
tcp-echo-gateway-istio-57844964f7-sl2j2   1/1     Running   0          67m   10.244.1.5    ambient-worker2   <none>           <none>
tcp-echo-v1-5f8dd78684-bwvqk              1/1     Running   0          76m   10.244.2.9    ambient-worker    <none>           <none>
tcp-echo-v2-794b5ff9c7-mwpbx              1/1     Running   0          76m   10.244.2.10   ambient-worker    <none>           <none>
waypoint-5d4bd47989-ww8hb                 1/1     Running   0          51m   10.244.2.11   ambient-worker    <none>           <none>
```

9. Set up an east-west bound TCP route tcp-echo-ew that is designed to send 90% of ciient requests to v1 and 10% to v2. The frontend field parentRefs points to the existing K8s service tcp-echo.

```
kubectl -n istio-io-tcp-traffic-shifting apply -f - <<EOF
apiVersion: gateway.networking.k8s.io/v1alpha2
kind: TCPRoute
metadata:
  name: tcp-echo-ew
spec:
  parentRefs:
  - group: ""
    kind: Service
    name: tcp-echo
  rules:
  - backendRefs:
    - name: tcp-echo-v1
      port: 9000
      weight: 90
    - name: tcp-echo-v2
      port: 9000
      weight: 10
EOF
```
```
keyuser@ubunclone:~/istio-1.24.0$ kubectl -n istio-io-tcp-traffic-shifting get tcproute
NAME          AGE
tcp-echo      11m
tcp-echo-ew   5m22s
```

10. Finally come to the scene of the bug by trying to connect to tcp-echo as the frontend of tcp-echo-ew. Note that the tcp-echo traffic is still distributed randomly between v1 and v2.
```
keyuser@ubunclone:~/istio-1.24.0$ kubectl -n istio-io-tcp-traffic-shifting exec deploy/curl -- sh -c "while true
do
        date | nc tcp-echo 9000
        sleep 2
done"
two Sat Dec  7 01:47:37 UTC 2024
one Sat Dec  7 01:47:39 UTC 2024
two Sat Dec  7 01:47:41 UTC 2024
one Sat Dec  7 01:47:43 UTC 2024
two Sat Dec  7 01:47:45 UTC 2024
one Sat Dec  7 01:47:47 UTC 2024
one Sat Dec  7 01:47:49 UTC 2024
one Sat Dec  7 01:47:51 UTC 2024
one Sat Dec  7 01:47:53 UTC 2024
one Sat Dec  7 01:47:55 UTC 2024
two Sat Dec  7 01:47:57 UTC 2024
two Sat Dec  7 01:47:59 UTC 2024
one Sat Dec  7 01:48:01 UTC 2024
^C
```

### Workaround hack

There is a handy but dirty hack to force the waypoint proxy to implement the logic of the given TCP route, tcp-echo-ew. This is done by using the Istio EnvoyFilter API to patch the relevant listener filter of the waypoint proxy.

Define an EnvoyFilter object called tcp-echo-filter, which is applied to these waypoint components.
* listener: main_internal
* filter chain: inbound-vip|9000|tcp|tcp-echo.istio-io-tcp-traffic-shifting.svc.cluster.local
* filter: envoy.filters.network.tcp_proxy

The target of the filter is specified as a weighted cluster between tcp-echo-v1 (90%) and tcp-echo-v2 (10%).
```
kubectl -n istio-io-tcp-traffic-shifting apply -f - <<EOF
apiVersion: networking.istio.io/v1alpha3
kind: EnvoyFilter
metadata:
  name: tcp-echo-filter
  namespace: istio-io-tcp-traffic-shifting
spec:
  workloadSelector:
    labels:
      gateway.networking.k8s.io/gateway-name: waypoint
  configPatches:
  - applyTo: NETWORK_FILTER
    match:
      context: SIDECAR_INBOUND # will match inbound listeners in all sidecars
      listener:
        name: "main_internal"
        filterChain:
          name: "inbound-vip|9000|tcp|tcp-echo.istio-io-tcp-traffic-shifting.svc.cluster.local"
          filter:
            name: "envoy.filters.network.tcp_proxy"
    patch:
      operation: REPLACE
      value:
        name: "envoy.filters.network.tcp_proxy"
        typed_config:
          "@type": "type.googleapis.com/envoy.extensions.filters.network.tcp_proxy.v3.TcpProxy"
          statPrefix: "inbound-vip|9000|tcp|tcp-echo.istio-io-tcp-traffic-shifting.svc.cluster.local"
          weightedClusters:
            clusters:
            - name: "inbound-vip|9000|tcp|tcp-echo-v1.istio-io-tcp-traffic-shifting.svc.cluster.local"
              weight: 90
            - name: "inbound-vip|9000|tcp|tcp-echo-v2.istio-io-tcp-traffic-shifting.svc.cluster.local"
              weight: 10
EOF
```
```
keyuser@ubunclone:~/istio-1.24.0$ kubectl -n istio-io-tcp-traffic-shifting apply -f - <<EOF
apiVersion: networking.istio.io/v1alpha3
kind: EnvoyFilter
metadata:
  name: tcp-echo-filter
  namespace: istio-io-tcp-traffic-shifting
spec:
  workloadSelector:
    labels:
      gateway.networking.k8s.io/gateway-name: waypoint
  configPatches:
  - applyTo: NETWORK_FILTER
    match:
      context: SIDECAR_INBOUND
      listener:
        name: "main_internal"
        filterChain:
          name: "inbound-vip|9000|tcp|tcp-echo.istio-io-tcp-traffic-shifting.svc.cluster.local"
          filter:
            name: "envoy.filters.network.tcp_proxy"
    patch:
      operation: REPLACE
      value:
        name: "envoy.filters.network.tcp_proxy"
        typed_config:
          "@type": "type.googleapis.com/envoy.extensions.filters.network.tcp_proxy.v3.TcpProxy"
          statPrefix: "inbound-vip|9000|tcp|tcp-echo.istio-io-tcp-traffic-shifting.svc.cluster.local"
          weightedClusters:
            clusters:
            - name: "inbound-vip|9000|tcp|tcp-echo-v1.istio-io-tcp-traffic-shifting.svc.cluster.local"
              weight: 90
            - name: "inbound-vip|9000|tcp|tcp-echo-v2.istio-io-tcp-traffic-shifting.svc.cluster.local"
              weight: 10
EOF
EnvoyFilter exposes internal implementation details that may change at any time. Prefer other APIs if possible, and exercise extreme caution, especially around upgrades.
envoyfilter.networking.istio.io/tcp-echo-filter configured
keyuser@ubunclone:~/istio-1.24.0$ kubectl -n istio-io-tcp-traffic-shifting get envoyfilter
NAME              AGE
tcp-echo-filter   19s
```

Now we can shift 90% of the tcp-echo traffic to v1 and 10% to v2 simply by accessing the tcp-echo service itself.
```
keyuser@ubunclone:~/istio-1.24.0$ kubectl -n istio-io-tcp-traffic-shifting exec deploy/curl -- sh -c "while true
do
        date | nc tcp-echo 9000
        sleep 2
done"
one Mon Dec  9 20:18:19 UTC 2024
one Mon Dec  9 20:18:21 UTC 2024
one Mon Dec  9 20:18:23 UTC 2024
one Mon Dec  9 20:18:25 UTC 2024
one Mon Dec  9 20:18:27 UTC 2024
one Mon Dec  9 20:18:29 UTC 2024
one Mon Dec  9 20:18:31 UTC 2024
one Mon Dec  9 20:18:33 UTC 2024
one Mon Dec  9 20:18:35 UTC 2024
one Mon Dec  9 20:18:37 UTC 2024
two Mon Dec  9 20:18:39 UTC 2024
one Mon Dec  9 20:18:41 UTC 2024
one Mon Dec  9 20:18:43 UTC 2024
one Mon Dec  9 20:18:45 UTC 2024
two Mon Dec  9 20:18:47 UTC 2024
one Mon Dec  9 20:18:49 UTC 2024
one Mon Dec  9 20:18:51 UTC 2024
one Mon Dec  9 20:18:53 UTC 2024
one Mon Dec  9 20:18:55 UTC 2024
^C
```

