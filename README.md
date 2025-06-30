# wordpress-mysql-k8s
LAB Deploy wordpress and mysql on kubernetes

## K8s Cluster spec
```
vagrant@controlplane:~/wordpressMysql$ kubectl get nodes -o wide
NAME           STATUS   ROLES           AGE     VERSION   INTERNAL-IP     EXTERNAL-IP   OS-IMAGE             KERNEL-VERSION       CONTAINER-RUNTIME
controlplane   Ready    control-plane   5d1h    v1.33.2   192.168.1.112   <none>        Ubuntu 22.04.5 LTS   5.15.0-142-generic   containerd://1.7.27
node01         Ready    <none>          4d21h   v1.33.2   192.168.1.41    <none>        Ubuntu 22.04.5 LTS   5.15.0-142-generic   containerd://1.7.27
node02         Ready    <none>          4d15h   v1.33.2   192.168.1.139   <none>        Ubuntu 22.04.5 LTS   5.15.0-142-generic   containerd://1.7.27

```
## Deploy configMap sectet pv pvc:
``` 
kubectl apply -f mysql-pv.yaml mysql-pvc.yaml mysql-secret.yaml wordpress-configmap.yaml

```
## Deploy mysql : clusterPort
```
kubectl apply -f mysql-deployment.yaml
kubectl apply -f mysql-service.yaml
```
## Deploy wordpress NodePort 
```
kubectl apply -f 
wordpress-deployment.yaml
wordpress-service.yaml

apiVersion: v1
kind: Service
metadata:
  name: wordpress
spec:
  type: NodePort
  selector:
    app: wordpress
  ports:
    - port: 80
      targetPort: 80
      nodePort: 30080
```
## Access to wordpress HTTP
```
http://ip-node:30080
```

## Ajouter le service ingress
```
Deployer Ingress controller (nginx):
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.9.6/deploy/static/provider/baremetal/deploy.yaml

Ajouter Ingress Ressource:
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: wordpress-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  rules:
  - host: wordpress.local
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: wordpress
            port:
              number: 80

kubectl apply -f wordpress-ingress.yaml
Tester : curl -H "Host: wordpress.local" http://192.168.1.41:30123
```

## Deployer un load balancer (metallb)
```
Pour avoir une IP public et acces en port 80
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.15.2/config/manifests/metallb-native.yaml
kubctl apply -i metallb/pool1.yaml
kubctl apply -i metallb/advertise.yaml
Modifier type de service iginx controller : type: LoadBalancer
kubectl edit svc ingress-nginx-controller -n ingress-nginx

Check:
kubectl get svc -n ingress-nginx
You should see the EXTERNAL-IP field populated with a MetalLB IP like 192.168.1.240

```
###
Déploployer SSL/HTTPS:
kubectl apply -f selfsigned-clusterissuer.yaml
kubectl apply -f wordpress-cert.yaml

Update wordpress Ingress:
spec:
  tls:
    - hosts:
        - wordpress.local
      secretName: wordpress-local-tls

Check:
curl -k https://wordpress.local


[Browser / curl on host]
    |
    v
[Metallb (port 80)]
    |
    v
[Ingress Controller in K8s ]
    |
    v
[Ingress Rule -> WordPress Service tls]
    |
    v
[WordPress Pod]
```

