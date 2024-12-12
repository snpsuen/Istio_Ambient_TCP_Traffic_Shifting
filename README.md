## In-mesh L4 traffic management in the Istio ambient mode

There is a known bug in the Istio ambient mode, where L4 traffic management fails to work inside a mesh, i.e. in the so-called the east/west direction ([Issue 54119](https://github.com/istio/istio/issues/54119)). More specifically, after an application developer creates such a TCP route, the waypoint concerned refuses to honour the traffic rule, though it is obliged by design to manage both the L4 and L7 communucation flows. 

Let's walk through an example of reproducing the bug in an attempt to shift the L4 traffic between two TCP echo service versions. After that, we will introduce a handly but dirty hack to force the waypoint to honour a specific TCP route by patching one of the listener filters through the EnvoyFilter API.

![L4 Traffic Management in Istio Ambient Mesh](Istio_ambient_east-west_L4.png)

### Limitation

In this example, we use the Kubernetes Gateway API to test out TCP traffic shifting in an Istio ambient mesh. It can be shown elsewhere that the same bug will also happen to the Istio APIs of VirtualService, Gateway and the likes. The crux of the problem is that the L4 traffic management logic simply fails to register with the waypoint, whether configued via the Kubernetes Gateway API or Istio APIs.

### Bug reproduction

The use case refers to the task outlined in the Istio doc, [TCP Traffic Shifting](https://istio.io/latest/docs/tasks/traffic-management/tcp-traffic-shifting/). 

1. Assume the Isto version is 1.24.0.. First and foremost, ensure the Istio ambient mode is installed with the flag, --set values.pilot.env.PILOT_ENABLE_ALPHA_GATEWAY_API=true.
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

5. Observe that the client requests for the plain, TCP route-free K8s service, tcp-echo, are randomly distribured beteen v1 and v2.
```
keyuser@ubunclone:~/istio-1.24.0$ kubectl -n istio-io-tcp-traffic-shifting exec deploy/curl -- sh -c "while true
do
        date | nc tcp-echo 9000
        sleep 2
done"
two Thu Dec  5 00:14:09 UTC 2024
one Thu Dec  5 00:14:11 UTC 2024
one Thu Dec  5 00:14:13 UTC 2024
two Thu Dec  5 00:14:15 UTC 2024
one Thu Dec  5 00:14:17 UTC 2024
two Thu Dec  5 00:14:20 UTC 2024
```


