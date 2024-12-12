## In-mesh L4 traffic management in the Istio ambient mode

There is a known bug in the Istio ambient mode, where L4 traffic management fails to work inside a mesh, i.e. in the so-called the east/west direction ([Issue 54119](https://github.com/istio/istio/issues/54119)). More specifically, after an application developer creates such a TCP route, it refuses to take effect on the waypoint concerned, though the latter is designed to manage both the L4 and L7 communucation flows. 

Let's walk through an example of reproducing the bug in an attempt to shift the L4 traffic between two TCP echo service versions. After that, we will introduce a handly but dirty hack to force the waypoint to specific TCP route by patching one of the listener filters through the EnvoyFilter API.

![L4 Traffic Management in Istio Ambient Mesh](Istio_ambient_east-west_L4.png)
