# Vagrant kubernetes 3 node cluster with ubuntu 20.04

<br/>

Download images from AWS can be extremely slow!

<br/>

```
Download redirected to host: vagrantcloud-files-production.s3.amazonaws.com
Progress: 7% (Rate: 324k/s, Estimated time remaining: 1:26:17)
```

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

## Possible check

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
http://node1.k8s:30123
```

<br/>

    // Delete created resources
    // $ kubectl delete svc nodejs-cats-app-nodeport
    // $ kubectl delete deployment nodejs-cats-app

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
