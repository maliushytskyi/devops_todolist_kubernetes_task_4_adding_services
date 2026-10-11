# ToDo app: pods and services

## 1. Apply the manifests

```bash
kubectl apply -f .infrastructure/namespace.yml
kubectl apply -f .infrastructure/busybox.yml
kubectl apply -f .infrastructure/todoapp-pod.yml
kubectl apply -f .infrastructure/clusterip-service.yml
kubectl apply -f .infrastructure/nodeport-service.yml
```

Check that the pods are `1/1 Running` and that the services have both pods as endpoints:

```bash
kubectl get pods -n todoapp --show-labels
kubectl get svc -n todoapp
kubectl get endpoints -n todoapp
```

## 2. Test the ClusterIP service DNS from the busybox container

The service DNS name is `<service>.<namespace>.svc.cluster.local`:

```bash
kubectl exec busybox -n todoapp -- curl -s http://todoapp-clusterip.todoapp.svc.cluster.local/api/ready/
```

Expected output: `{"status": "ready"}`.

To see load balancing, send several requests and check which pod received them:

```bash
for i in 1 2 3 4 5 6 7 8; do
  kubectl exec busybox -n todoapp -- curl -s -o /dev/null http://todoapp-clusterip/
done
kubectl logs todoapp-1 -n todoapp | grep 'GET / '
kubectl logs todoapp-2 -n todoapp | grep 'GET / '
```

Both pods should show requests.

## 3. Test with port-forward to the service

```bash
kubectl port-forward service/todoapp-clusterip 8080:80 -n todoapp
```

Open http://localhost:8080/ or run `curl http://localhost:8080/api/ready/`.
Note: port-forward to a service connects to one selected pod and does not balance requests.

## 4. Access the app through the NodePort service

Port 30007 is open on every node.

```bash
kubectl get nodes -o wide        # note the INTERNAL-IP of the node
```

- Minikube: `minikube service todoapp-nodeport -n todoapp --url`
- Cluster where the node IP is reachable from your machine: `curl http://<NODE_IP>:30007/api/ready/`
- kind / Docker Desktop (the node is a Docker container, its IP is not reachable from the host):
```bash
  docker exec desktop-control-plane curl -s http://localhost:30007/api/ready/
```
  or from inside the cluster:
```bash
  kubectl exec busybox -n todoapp -- curl -s http://<NODE_IP>:30007/api/ready/
```

## 5. Cleanup

```bash
kubectl delete namespace todoapp
```