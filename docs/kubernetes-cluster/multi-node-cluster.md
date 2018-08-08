# Intro

This document provide instructions on how to setup a multi node Kubernetes cluster.



### K8s Master

References:

* [Kubernetes on bare-metal in 10 minutes](https://blog.alexellis.io/kubernetes-in-10-minutes/)
* [How to install Kubernetes on Ubuntu 18.04 Bionic Beaver Linux](https://linuxconfig.org/how-to-install-kubernetes-on-ubuntu-18-04-bionic-beaver-linux)

Launch a Ubuntu 16.04 machine with 4G of memory and call it `k8s-master`.

Make sure it is up to date.

```
sudo apt-get update

sudo apt-get dist-upgrade

sudo reboot
```

Install Docker

```
sudo apt-get update \
  && sudo apt-get install -qy docker.io
```

Install Kubernetes apt repo

```
sudo apt-get update \
  && sudo apt-get install -y apt-transport-https \
  && curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add -


echo "deb http://apt.kubernetes.io/ kubernetes-xenial main" \
  | sudo tee -a /etc/apt/sources.list.d/kubernetes.list \
  && sudo apt-get update
```

Install `kubelet`, `kubeadm` and `kubernetes-cni`

```
sudo apt-get update \
  && sudo apt-get install -y \
  kubelet \
  kubeadm \
  kubernetes-cni
```

Disable swap

Edit `/etc/fstab`  

```
sudo vim /etc/fstab
```

Comment out swap.

```
LABEL=cloudimg-rootfs   /        ext4   defaults        0 0
/dev/vdb        /mnt    auto    defaults,nofail,x-systemd.requires=cloud-init.service,comment=cloudconfig       0       2
#/dev/vdc       none    swap    sw,comment=cloudconfig  0       0
```

Make sure to reboot to make it permanent.

```
sudo reboot
```

Determine the `k8s-master`s IP address using `ifconfig`.

```
ens3      Link encap:Ethernet  HWaddr fa:16:3e:fc:ec:eb
          inet addr:192.168.23.74  Bcast:192.168.23.255  Mask:255.255.255.0
          inet6 addr: fe80::f816:3eff:fefc:eceb/64 Scope:Link
          UP BROADCAST RUNNING MULTICAST  MTU:1500  Metric:1
          RX packets:41127 errors:0 dropped:0 overruns:0 frame:0
          TX packets:35019 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:1000
          RX bytes:159902830 (159.9 MB)  TX bytes:2417115 (2.4 MB)

```

Initialize the `k8s-master`


Make sure you change the `--apiserver-advertise-address` to the IP of the `k8s-master`.

```
sudo kubeadm init --pod-network-cidr=10.244.0.0/16 --apiserver-advertise-address=192.168.23.74 --kubernetes-version stable-1.11
```

When it's done, you should see the following texts.

```
Your Kubernetes master has initialized successfully!

To start using your cluster, you need to run the following as a regular user:

  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config

You should now deploy a pod network to the cluster.
Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
  https://kubernetes.io/docs/concepts/cluster-administration/addons/

You can now join any number of machines by running the following on each node
as root:

  kubeadm join 192.168.23.74:6443 --token uovgw2.23vhhu4w0qq2g9vt --discovery-token-ca-cert-hash sha256:29d85f42aeb20b43a0687251cb8010600b12ca1d57326e9cfab22de93706b85e

```

Let us follow the instructions above to start using the cluster.

```
  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config

```

![#f03c15](https://placehold.it/15/f03c15/000000?text=+) `!!!! IMPORTANT !!!!`

```diff
- Make note of the for nodes to join the cluster. Save this and put it somewhere.

 kubeadm join 192.168.23.74:6443 --token uovgw2.23vhhu4w0qq2g9vt --discovery-token-ca-cert-hash sha256:29d85f42aeb20b43a0687251cb8010600b12ca1d57326e9cfab22de93706b85e
```

Apply the pod network (flannel)

Configure networking for pods.

```
kubectl apply -f kube-flannel.yml
```

You should see

```
ubuntu@k8s-master:~$ kubectl apply -f kube-flannel.yml
clusterrole.rbac.authorization.k8s.io/flannel created
clusterrolebinding.rbac.authorization.k8s.io/flannel created
serviceaccount/flannel created
configmap/kube-flannel-cfg created
daemonset.extensions/kube-flannel-ds-amd64 created
daemonset.extensions/kube-flannel-ds-arm64 created
daemonset.extensions/kube-flannel-ds-arm created
daemonset.extensions/kube-flannel-ds-ppc64le created
daemonset.extensions/kube-flannel-ds-s390x created

```


```
kubectl apply -f kube-flannel-rbac.yml
```

You should see

```
ubuntu@k8s-master:~$ kubectl apply -f kube-flannel-rbac.yml
clusterrole.rbac.authorization.k8s.io/flannel configured
clusterrolebinding.rbac.authorization.k8s.io/flannel configured
```

Check if everything is working

```
kubectl get all --namespace=kube-system
```

The output should be

```
ubuntu@k8s-master:~$ kubectl get all --namespace=kube-system
NAME                                     READY     STATUS    RESTARTS   AGE
pod/coredns-78fcdf6894-6nfgb             1/1       Running   0          10m
pod/coredns-78fcdf6894-crmj9             1/1       Running   0          10m
pod/etcd-k8s-master                      1/1       Running   0          1m
pod/kube-apiserver-k8s-master            1/1       Running   0          1m
pod/kube-controller-manager-k8s-master   1/1       Running   0          1m
pod/kube-flannel-ds-amd64-dkl5p          1/1       Running   0          1m
pod/kube-proxy-4p8g7                     1/1       Running   0          10m
pod/kube-scheduler-k8s-master            1/1       Running   0          1m

NAME               TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)         AGE
service/kube-dns   ClusterIP   10.96.0.10   <none>        53/UDP,53/TCP   10m

NAME                                     DESIRED   CURRENT   READY     UP-TO-DATE   AVAILABLE   NODE SELECTOR                     AGE
daemonset.apps/kube-flannel-ds-amd64     1         1         1         1            1           beta.kubernetes.io/arch=amd64     1m
daemonset.apps/kube-flannel-ds-arm       0         0         0         0            0           beta.kubernetes.io/arch=arm       1m
daemonset.apps/kube-flannel-ds-arm64     0         0         0         0            0           beta.kubernetes.io/arch=arm64     1m
daemonset.apps/kube-flannel-ds-ppc64le   0         0         0         0            0           beta.kubernetes.io/arch=ppc64le   1m
daemonset.apps/kube-flannel-ds-s390x     0         0         0         0            0           beta.kubernetes.io/arch=s390x     1m
daemonset.apps/kube-proxy                1         1         1         1            1           beta.kubernetes.io/arch=amd64     10m

NAME                      DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/coredns   2         2         2            2           10m

NAME                                 DESIRED   CURRENT   READY     AGE
replicaset.apps/coredns-78fcdf6894   2         2         2         10m

```


### K8s Slave node

Launch another Ubuntu 16.04 with around 8G of memory and 50G of disk space. Call it `k8s-n1`.

Make sure it is up to date.

```
sudo apt-get update

sudo apt-get dist-upgrade

sudo reboot
```

Install Docker

```
sudo apt-get update \
  && sudo apt-get install -qy docker.io
```

Install Kubernetes apt repo

```
sudo apt-get update \
  && sudo apt-get install -y apt-transport-https \
  && curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add -


echo "deb http://apt.kubernetes.io/ kubernetes-xenial main" \
  | sudo tee -a /etc/apt/sources.list.d/kubernetes.list \
  && sudo apt-get update
```

Install `kubeadm`

```
sudo apt-get update \
  && sudo apt-get install -y \
  kubeadm
```

Disable swap

Edit `/etc/fstab`  

```
sudo vim /etc/fstab
```

Comment out swap.

```
LABEL=cloudimg-rootfs   /        ext4   defaults        0 0
/dev/vdb        /mnt    auto    defaults,nofail,x-systemd.requires=cloud-init.service,comment=cloudconfig       0       2
#/dev/vdc       none    swap    sw,comment=cloudconfig  0       0
```

Make sure to reboot to make it permanent.

```
sudo reboot
```

Join the cluster

Use the command you saved earlier when setting up `k8s-master` on how a node joins a cluster.

```
sudo kubeadm join 192.168.23.74:6443 --token uovgw2.23vhhu4w0qq2g9vt --discovery-token-ca-cert-hash sha256:29d85f42aeb20b43a0687251cb8010600b12ca1d57326e9cfab22de93706b85e
```

You should see the output

```
This node has joined the cluster:
* Certificate signing request was sent to master and a response
  was received.
* The Kubelet was informed of the new secure connection details.

Run 'kubectl get nodes' on the master to see this node join the cluster.
```

In the `k8s-master`, type

```
kubectl get nodes
```

The output should be

```
ubuntu@k8s-master:~$ kubectl get nodes
NAME         STATUS    ROLES     AGE       VERSION
k8s-master   Ready     master    1h        v1.11.1
k8s-n1       Ready     <none>    1m        v1.11.1
```


### Managing cluster remotely

#### References

* [Set the KUBECONFIG environment variable](https://kubernetes.io/docs/tasks/access-application-cluster/configure-access-multiple-clusters/#set-the-kubeconfig-environment-variable)
* [Configure Access to Multiple Clusters](https://kubernetes.io/docs/tasks/access-application-cluster/configure-access-multiple-clusters/)
* [Organizing Cluster Access Using kubeconfig Files](https://kubernetes.io/docs/concepts/configuration/organize-cluster-access-kubeconfig/)
* [How can I set my local kubectl to point to 2+ clusters created using kubeadm?](https://stackoverflow.com/questions/47985934/how-can-i-set-my-local-kubectl-to-point-to-2-clusters-created-using-kubeadm)

We don't want to ssh to out `k8s-master` to issue commands. We want to be able to type commands remotely.

Create another VM where you can issue commands to the cluster. This is where we can manage multiple clusters remotely by using kubeconfig to indicate which cluster to work on.

Install `kubeadm`

```
sudo apt-get update \
  && sudo apt-get install -y \
  kubeadm
```

We need to inform it of our cluster.

Create a file called `~/.kube/k8s-config` in your VM.

In your `k8s-master` node, copy the contents of `~/.kube/config` into `~/.kube/k8s-config`.

Open `~/.bash_aliases` to easily switch `kubeconfig`s.

```
ubuntu@ritz-k8s:~$ cat ~/.bash_aliases
alias k8s='export KUBECONFIG=$HOME/.kube/k8s-config'
```

Activate aliases

```
source ~/.bash_aliases
```

Work on `k8s-master` by switching `KUBECONFIG` to point to `$HOME/.kube/k8s-config`

```
k8s
```

Check if you are able to connect to the cluster

```
ubuntu@ritz-k8s:~$ kubectl get nodes
NAME         STATUS    ROLES     AGE       VERSION
k8s-master   Ready     master    3h        v1.11.1
k8s-n1       Ready     <none>    2h        v1.11.1


ubuntu@ritz-k8s:~$ kubectl cluster-info
Kubernetes master is running at https://192.168.23.74:6443
KubeDNS is running at https://192.168.23.74:6443/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy

To further debug and diagnose cluster problems, use 'kubectl cluster-info dump'.

```

Congratulations! Now you have a 2 node (1 master, 1 slave) Kubernetes cluster that you can manage remotely.



