![L4 Traffic Management in Istio Ambient Mesh](Istio_ambient_east-west_L4.png)

There is a known bug in the Istio ambient mode, where L4 traffic management fails to work inside a mesh, i.e. in the so-called the east/west direction ([Issue 54119](https://github.com/istio/istio/issues/54119)). More specifically, after an application developer creates such a TCP route, it refuse to take effect on the waypoint concerned, though the latter is designed to manage both L4 and L7 traffic flows. 

Let's walk through an example of reproducing a known bug in Istio ambient mesh.
