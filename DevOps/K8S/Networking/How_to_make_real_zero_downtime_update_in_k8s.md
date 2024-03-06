# How to make real zero downtime update in k8s

#### 1. Graceful app shutdown
1. Stop handling new requests
2. Handle requests that already got
3. Stop the service and exit

#### 2. Update your k8s config
Add `terminationGracePeriodSeconds` and `preStop` lifecycle hook
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ..
spec:
  ...
    spec:
      # This set timeout between SIGTERM and SIGKILL
      terminationGracePeriodSeconds: 5
      containers:
        - name: ...
          ...
          # This stop routing to pod and call the command
          # Which set timeout on sending SIGTERM to 5 sec
          # This gives time to handle in-flight requests 
          # to the pod
          lifecycle:
            preStop:
              exec:
                command: ["sh", "-c", "sleep 5"]
```
