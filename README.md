Let's assume you have a deployment with multiple replicas (for example, 3 replicas). One of them is running at 100% CPU load. 
The other two are idle. You are using classic Kubernetes service and ingress. This means that requests are distributed to 
pods using round-robin. Some users may complain about slowness because Kubernetes is still directing requests to the pod 
that is experiencing a bottleneck, as it is still in the READY state. There are dozens of ways to prevent or solve this 
on the app side and the DevOps side. Here, I will give an example of how we can solve this in the simplest way using 
probes in Kubernetes. This example may not fit the working structure of your app or your load balancing structure. 
In that case, you may need to adapt it to suit your needs.

### Install
```
$ kubectl create ns <NS>
$ kubectl apply -n <NS> configmap.yaml
$ kubectl apply -n <NS> deployment.yaml

$ kubctl get pods -n <NS>
```

In our scenario, when the CPU load exceeds 90%, we put the pod in an unready state and stop sending requests from 
the service to the pod. When the CPU load drops, it returns to the ready state. With livenessProbe, 
we automatically restart the pod when it reaches 100%. 
The startupProbe here is used to delay the start of other probes until after nginx has started, 
because we are performing some setup beforehand.

If the pod itself does not enter a CPU bottleneck, you can trigger it using the stress command via the shell.

kubectl exec -it <POD> -n <NS> --stress --cpu 2 --timeout 60

In an example scenario, you can see it as follows when you describe it.

```
Events:
  Type     Reason     Age                    From               Message
  ----     ------     ----                   ----               -------
  Normal   Scheduled  5m33s                  default-scheduler  Successfully assigned batur/cpu-sensitive-app-595c4fcc4b-8mfg9 to cluster1-control-plane
  Normal   Pulled     5m31s                  kubelet            Successfully pulled image "nginx:1.25" in 1.264s (1.264s including waiting). Image size: 67665983 bytes.
  Normal   Pulling    2m42s (x2 over 5m32s)  kubelet            Pulling image "nginx:1.25"
  Normal   Created    2m40s (x2 over 5m31s)  kubelet            Created container: myapp
  Normal   Started    2m40s (x2 over 5m31s)  kubelet            Started container myapp
  Normal   Pulled     2m40s                  kubelet            Successfully pulled image "nginx:1.25" in 2.119s (2.119s including waiting). Image size: 67665983 bytes.
  Warning  Unhealthy  37s (x12 over 4m47s)   kubelet            Readiness probe failed: CPU Usage: 100%
CPU > 90% → NOT Ready
  Warning  Unhealthy  32s (x13 over 4m52s)  kubelet  Liveness probe failed: CPU Usage: 100%
Established TCP connections: 0
CPU = 100% → Liveness FAILED
  Normal  Killing  22s (x2 over 3m12s)  kubelet  Container myapp failed liveness probe, will be restarted
```

