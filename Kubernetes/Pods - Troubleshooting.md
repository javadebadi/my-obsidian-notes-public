
To troubleshoot pods we usually troubleshoot with following steps:
- use `kubectl get pods` to check on current Pod status.
- use `kubectl describe pod <pod-name>` to get more information about Pod status 
- use `kubectl logs <pod-name>` to get access to Pod application output that has been logged.
- use `kubectl exec -it <pod-name> sh`  to open a shell on the Pod and check specific components.
