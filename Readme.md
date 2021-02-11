# Vagrant kubernetes 3 node cluster with ubuntu 20.04

<br/>

### On Host (with linux)

<br/>

**kubectl installation:**

```
$ curl -LO https://storage.googleapis.com/kubernetes-release/release/$(curl -s https://storage.googleapis.com/kubernetes-release/release/stable.txt)/bin/linux/amd64/kubectl && chmod +x kubectl && sudo mv kubectl /usr/local/bin/
```

<br/>

    $ sudo vi /etc/hosts

```
#---------------------------------------------------------------------
# Kubernetes cluster
#---------------------------------------------------------------------

192.168.0.10 master.k8s master
192.168.0.11 node1.k8s node1
192.168.0.12 node2.k8s node2
```

<br/>

    $ vagrant plugin install vagrant-hostmanager

<br/>

    $ mkdir ~/vagrant-kubernetes-ubuntu-20 && cd ~/vagrant-kubernetes-ubuntu-20

    $ git clone https://github.com/webmakaka/vagrant-kubernetes-3-node-cluster-ubuntu-20.04 .

    $ cd latest

    $ vagrant box update

    $ vagrant up

    $ vagrant status
    Current machine states:

    master.k8s                running (virtualbox)
    node1.k8s                 running (virtualbox)
    node2.k8s                 running (virtualbox)

<br/>

### Copy kubernetes config for manage kubernetes cluster remotely

<br/>

    $ mkdir -p ~/.kube

<br/>

    // root password: kubeadmin
    $ scp root@master:/etc/kubernetes/admin.conf ~/.kube/config

<br/>

```
$ kubectl version --short
Client Version: v1.20.2
Server Version: v1.20.2
```

<br/>

```
$ kubectl get nodes
NAME     STATUS   ROLES                  AGE     VERSION
master   Ready    control-plane,master   12m     v1.20.2
node1    Ready    <none>                 7m10s   v1.20.2
node2    Ready    <none>                 101s    v1.20.2
```

<br/>

### Get Additional Inforamtion

<br/>

```
$ kubectl get po -n kube-system
NAME                                       READY   STATUS    RESTARTS   AGE
calico-kube-controllers-57fc9c76cc-z8z69   1/1     Running   0          12m
calico-node-kmjdc                          1/1     Running   0          7m27s
calico-node-skkvh                          1/1     Running   0          12m
calico-node-skrmb                          1/1     Running   0          119s
coredns-74ff55c5b-9sshg                    1/1     Running   0          12m
coredns-74ff55c5b-cmzq5                    1/1     Running   0          12m
etcd-master                                1/1     Running   0          12m
kube-apiserver-master                      1/1     Running   0          12m
kube-controller-manager-master             1/1     Running   0          12m
kube-proxy-2nvst                           1/1     Running   0          12m
kube-proxy-nvl5m                           1/1     Running   0          7m27s
kube-proxy-ssngv                           1/1     Running   0          119s
kube-scheduler-master                      1/1     Running   0          12m
```

<br/>

```
$ kubectl cluster-info
Kubernetes control plane is running at https://192.168.0.10:6443
KubeDNS is running at https://192.168.0.10:6443/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy
```

<br/>

```
$ kubectl get cs
Warning: v1 ComponentStatus is deprecated in v1.19+
NAME                 STATUS      MESSAGE                                                                                       ERROR
controller-manager   Unhealthy   Get "http://127.0.0.1:10252/healthz": dial tcp 127.0.0.1:10252: connect: connection refused
scheduler            Unhealthy   Get "http://127.0.0.1:10251/healthz": dial tcp 127.0.0.1:10251: connect: connection refused
etcd-0               Healthy     {"health":"true"}
```

<br/>

## Possible checks

<br/>

```
$ cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nodejs-cats-app
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nodejs-cats-app
  template:
    metadata:
      labels:
        app: nodejs-cats-app
        env: dev
    spec:
      containers:
      - name: nodejs-cats-app
        image: webmakaka/cats-app
        imagePullPolicy: Always
        ports:
        - containerPort: 8080
EOF
```

<br/>

```
$ cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Service
metadata:
  name: nodejs-cats-app-nodeport
spec:
  type: NodePort
  ports:
  - port: 80
    targetPort: 8080
    nodePort: 30123
  selector:
    app: nodejs-cats-app
EOF
```

<br/>

```
$ kubectl get pods
NAME                              READY   STATUS    RESTARTS   AGE
nodejs-cats-app-56cc7754f-ghvzq   1/1     Running   0          25s
nodejs-cats-app-56cc7754f-r2c95   1/1     Running   0          25s
nodejs-cats-app-56cc7754f-vjnfb   1/1     Running   0          25s
```

<br/>

```
$ kubectl get svc
NAME                       TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)        AGE
kubernetes                 ClusterIP   10.96.0.1        <none>        443/TCP        14m
nodejs-cats-app-nodeport   NodePort    10.105.129.237   <none>        80:30123/TCP   30s
```

<br/>

Should work

```
http://node1.k8s:30123
http://node2.k8s:30123
```

<br/>

    // Delete created resources
    // $ kubectl delete svc nodejs-cats-app-nodeport
    // $ kubectl delete deployment nodejs-cats-app

<br/>

## Metal LB

<br/>

https://www.youtube.com/watch?v=2SmYjj-GFnE

<br/>

https://metallb.universe.tf/installation/

<br/>


```
$ LATEST_VERSION=$(curl --silent "https://api.github.com/repos/metallb/metallb/releases/latest" | grep '"tag_name"' | sed -E 's/.*"([^"]+)".*/\1/')

$ echo ${LATEST_VERSION}
```
<br/>

```
$ kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/${LATEST_VERSION}/manifests/namespace.yaml

$ kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/${LATEST_VERSION}/manifests/metallb.yaml

# On first install only
$ kubectl create secret generic -n metallb-system memberlist --from-literal=secretkey="$(openssl rand -base64 128)"
```

<br/>

```
$ cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: ConfigMap
metadata:
  namespace: metallb-system
  name: config
data:
  config: |
    address-pools:
    - name: default
      protocol: layer2
      addresses:
      - 192.168.0.21-192.168.0.50
EOF
```

<br/>

```
$ kubectl get pods -n metallb-system
NAME                          READY   STATUS    RESTARTS   AGE
controller-65db86ddc6-glpk9   1/1     Running   0          2m7s
speaker-26rbj                 1/1     Running   0          2m7s
speaker-7485w                 1/1     Running   0          2m7s
speaker-mvdnb                 1/1     Running   0          2m7s
```

<br/>

```
$ cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: cats-app-deployment
  labels:
    app: cats-app
spec:
  replicas: 3
  selector:
    matchLabels:
      app: cats-app
  template:
    metadata:
      labels:
        app: cats-app
    spec:
      containers:
      - name: cats-app
        image: webmakaka/cats-app:latest
        ports:
        - containerPort: 8080
EOF
```

<br/>

```
$ kubectl expose deploy cats-app --port 80 --target-port 8080 --type LoadBalancer
```

<br/>

```
$ kubectl get svc
NAME         TYPE           CLUSTER-IP      EXTERNAL-IP    PORT(S)        AGE
cats-app     LoadBalancer   10.98.196.136   192.168.0.21   80:30520/TCP   5s
kubernetes   ClusterIP      10.96.0.1       <none>         443/TCP        86m
```

<br/>

http://192.168.0.21/  
OK!

## Kubernetes ingress

<br/>

https://www.youtube.com/watch?v=UvwtALIb2U8

<br/>

https://kubernetes.github.io/ingress-nginx/deploy/

<br/>

### Helm3 install

    $ curl https://raw.githubusercontent.com/helm/helm/master/scripts/get-helm-3 | bash

    $ helm version --short --client
    v3.4.2+g23dd3af

<br/>

    $ helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx

    $ helm repo update

<br/>

    $ kubectl create ns ingress-nginx

    $ helm show values ingress-nginx/ingress-nginx > /tmp/ingress-nginx.yaml

    $ vi /tmp/ingress-nginx.yaml +54

```
hostNetwork: true

****

  hostPort:
    enabled: true

```

<br/>

    $ vi /tmp/ingress-nginx.yaml +144

```
   kind: Deployment
```

<br/>

```
   kind: DaemonSet
```

<br/>

```
$ helm install ingress-nginx \
  ingress-nginx/ingress-nginx \
  -n ingress-nginx \
  --values /tmp/ingress-nginx.yaml
```

<br/>

```
$ helm list -n ingress-nginx
NAME         	NAMESPACE    	REVISION	UPDATED                                	STATUS  	CHART               	APP VERSION
ingress-nginx	ingress-nginx	1       	2021-01-14 16:39:12.652014803 +0300 MSK	deployed	ingress-nginx-3.20.1	0.43.0
```

<br/>

```
$ kubectl  get all
NAME                            READY   STATUS    RESTARTS   AGE
pod/cats-app-595964467d-bx2l8   1/1     Running   0          16m
pod/cats-app-595964467d-j5qwq   1/1     Running   0          16m
pod/cats-app-595964467d-rgqrr   1/1     Running   0          16m

NAME                 TYPE           CLUSTER-IP      EXTERNAL-IP    PORT(S)        AGE
service/cats-app     LoadBalancer   10.101.35.242   192.168.0.21   80:30570/TCP   15m
service/kubernetes   ClusterIP      10.96.0.1       <none>         443/TCP        43m

NAME                       READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/cats-app   3/3     3            3           16m

NAME                                  DESIRED   CURRENT   READY   AGE
replicaset.apps/cats-app-595964467d   3         3         3       16m
```

<br/>

```
$ cat <<EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: ingress-cats-app
spec:
  rules:
  - host: cats.app
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: cats-app-deployment
            port:
              number: 80
EOF
```

<br/>

```
$ kubectl get ing
NAME               CLASS    HOSTS      ADDRESS   PORTS   AGE
ingress-cats-app   <none>   cats.app             80      52s
```

<br/>

    $ kubectl -n ingress-nginx get all

```
$ kubectl -n ingress-nginx get all
NAME                                 READY   STATUS    RESTARTS   AGE
pod/ingress-nginx-controller-g6b8d   1/1     Running   0          14m
pod/ingress-nginx-controller-kfj5d   1/1     Running   0          14m

NAME                                         TYPE           CLUSTER-IP      EXTERNAL-IP    PORT(S)                      AGE
service/ingress-nginx-controller             LoadBalancer   10.109.73.155   192.168.0.22   80:31143/TCP,443:32332/TCP   14m
service/ingress-nginx-controller-admission   ClusterIP      10.105.83.58    <none>         443/TCP                      14m

NAME                                      DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR            AGE
daemonset.apps/ingress-nginx-controller   2         2         2       2            2           kubernetes.io/os=linux   14m

```

<br/>

Add EXTERNAL-IP to hosts file

<br/>

    $ sudo vi /etc/hosts

```
***
192.168.0.22 cats.app

```

<br/>

chrome browser: cats.app

type: thisisunsafe in the browser window with security warning.

<br/>
<br/>

**Original src:**  
https://github.com/justmeandopensource/kubernetes

**How to run original:**  
https://www.youtube.com/watch?v=AoEWX84h_ig

<br/>

---

<br/>

**Marley**

Any questions in english: <a href="https://jsdev.org/chat/">Telegram Chat</a>  
Любые вопросы на русском: <a href="https://jsdev.ru/chat/">Телеграм чат</a>
