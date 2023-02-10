# TP Déployer un cluster Kubernetes
Nathan BOULOGNE

**Préparation infrastructure**

- Récupérer le terraform permettant de faire le déploiement

    git clone git@github.com:Arcahub/kube-install-tuto.git

- API key pour l’utilisateur en sélectionnant le projet M1-M2 Conteneur et Orchestration sur Scaleway. Ajouter ensuite la clé d’accès et la clé secrète : 

`   scw init`

- Modifier les fichiers variables.yaml kube-install-tuto/terraform/modules/k8s/modules:

`type : DEV1-M`

- dossier terraform et déployer le projet

```
cd kube-install-tuto/terraform
terraform init
terraform plan
terraform apply -var="project_name=nathan-kube"
```

**Installation de Kubernetes avec Kubeadm**

ssh -J tp3-kubernetes@208.47.124.88:61000 root@10.73.23.239
ssh -J tp3-kubernetes@208.47.124.88:61000 root@10.70.76.32
ssh -J tp3-kubernetes@208.47.124.88:61000 root@10.72.167.11

- Préparation des nodes

```
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF

sudo modprobe overlay
sudo modprobe br_netfilter

# sysctl params required by setup, params persist across reboots
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF

# Apply sysctl params without reboot
sudo sysctl --system
```

- Installer containerd

```
# Install required packages for https repository
sudo apt-get update && sudo apt-get install -y apt-transport-https ca-certificates curl
# Add Docker’s official GPG key
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -
# Add Docker repository
sudo add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable"
# Update package manager index
sudo apt-get update
```

```
sudo apt-get install -y containerd.io
```
```
sudo mkdir -p /etc/containerd
sudo containerd config default > /etc/containerd/config.toml
```

- Modifier le fichier /etc/containerd/config.toml en mettant la valeur du SystemdCgroup à `true`. Redémarrer ensuite le service containerd

```
nano /etc/containerd/config.toml
```
![](https://i.imgur.com/6z8OwJz.png)

```
sudo systemctl restart containerd
```

- Test du service containerd

```
ctr images pull docker.io/library/hello-world:latest
sudo ctr run --rm docker.io/library/hello-world:latest hello-world
ctr images rm docker.io/library/hello-world:latest
```

- Installe les paquets kubeadm, kubelet et kubectl

```
# Install required packages for https repository
sudo apt-get update && sudo apt-get install -y apt-transport-https curl

# Add Kubernetes’s official GPG key
curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add -

# Add Kubernetes repository
cat <<EOF | sudo tee /etc/apt/sources.list.d/kubernetes.list
deb https://apt.kubernetes.io/ kubernetes-xenial main
EOF

# Update package manager index
sudo apt-get update

# Install kubeadm, kubelet and kubectl with the exact same version or else components could be incompatible
sudo apt-get install -y kubelet=1.25.0-00 kubeadm=1.25.0-00 kubectl=1.25.0-00

# Hold the version of the packages
sudo apt-mark hold kubelet kubeadm kubectl
```

**paramètrer node control plane**

- Se connecter au node control plane et initier le nœud avec kubeadm

![](https://i.imgur.com/LfzizNN.png)

- Vérifier les nodes et les pods

```
kubectl get nodes

NAME                        STATUS     ROLES           AGE     VERSION
nathan-kube-controlplane   NotReady   control-plane   4m54s   v1.25.0
```
```
kubectl get pods --all-namespaces

NAMESPACE     NAME                                                READY   STATUS    RESTARTS   AGE
kube-system   coredns-565d847f94-8g78h                            0/1     Pending   0          5m17s
kube-system   coredns-565d847f94-dxdqc                            0/1     Pending   0          5m17s
kube-system   etcd-nathan-kube-controlplane                       1/1     Running   0          5m20s
kube-system   kube-apiserver-nathan-kube-controlplane             1/1     Running   0          5m20s
kube-system   kube-controller-manager-nathan-kube-controlplane    1/1     Running   0          5m20s
kube-system   kube-proxy-857mc                                    1/1     Running   0          5m17s
kube-system   kube-scheduler-nathan-kube-controlplane             1/1     Running   0          5m22s
```

- Installer le plugin CNI

```
kubectl apply -f https://github.com/weaveworks/weave/releases/download/v2.8.1/weave-daemonset-k8s.yaml
```

- Vérifier que le node control plane est ready

```
kubectl get nodes
NAME                        STATUS   ROLES           AGE   VERSION
nathan-kube-controlplane   Ready    control-plane   10m   v1.25.0
```

```
kubectl get pods --all-namespaces
NAMESPACE     NAME                                                READY   STATUS    RESTARTS        AGE
kube-system   coredns-565d847f94-8g78h                            1/1     Running   0               12m
kube-system   coredns-565d847f94-dxdqc                            1/1     Running   0               12m
kube-system   etcd-nathan-kube-controlplane                       1/1     Running   0               12m
kube-system   kube-apiserver-nathan-kube-controlplane             1/1     Running   0               12m
kube-system   kube-controller-manager-nathan-kube-controlplane    1/1     Running   0               12m
kube-system   kube-proxy-857mc                                    1/1     Running   0               12m
kube-system   kube-scheduler-nathan-kube-controlplane             1/1     Running   0               12m
kube-system   weave-net-s6v44
```   

**Worker node**

- créer un token pour relier les 2 workers au cluster

```
kubeadm token create --print-join-command
kubeadm join 192.168.1.62:6443 --token iuqy4s.sqv2yfibfdfw746p --discovery-token-ca-cert-hash sha256:2908618fb5d9039b23230718031e0bc74dfcef35e7739137eb3980b2ef3eefa6
```

- Se connecter aux 2 nodes puis lancer la commande générée pour ajouter les ajouter au cluster

```
ssh -J tp3-kubernetes@208.47.124.88:61000  root@10.71.96.47
ssh -J tp3-kubernetes@208.47.124.88:61000  root@10.70.152.77
kubeadm join 192.168.1.62:6443 --token iuqy4s.sqv2yfibfdfw746p --discovery-token-ca-cert-hash sha256:2908618fb5d9039b23230718031e0bc74dfcef35e7739137eb3980b2ef3eefa6
```

- Vérifier les workers
```
kubectl label node nathan-kube-node-1 node-role.kubernetes.io/worker=worker
node/nathan-kube-node-1 labeled

kubectl label node nathan-kube-node-2 node-role.kubernetes.io/worker=worker
node/nathan-kube-node-2 labeled
```

**Les pods statiques**

- Verifier les fichiers dans /etc/kubernetes/manifests pour avoircomme namespace kube-system

```
sudo ls /etc/kubernetes/manifests
kubectl get pods --namespace=kube-system
```

- Supprimer le pod kube-apiserver pour voir si le pod se recréer automatiquement.

```
kubectl delete pod kube-apiserver-nathan-kube-controlplane --namespace=kube-system
pod "kube-apiserver-nathan-kube-controlplane" deleted
kubectl get pods --namespace=kube-system
```

- Simuler la suppression du fichier du pod kube-apiserver.
```
sudo mv /etc/kubernetes/manifests/kube-apiserver.yaml ~/kube-apiserver.yaml
kubectl get pods --namespace=kube-system
```

**Les certificats**

- L’expiration des certificats avec `kubeadm`

```
kubeadm certs check-expiration
```

- On peutrenouveler les certificats. Il faut ensuite redémarrer le kubelet

```
kubeadm certs renew all
sudo systemctl daemon-reload
sudo systemctl restart kubelet
```

**Créer un premier déploiement**

- Déployer une première application nginx avec 3 replicas

```
kubectl create deployment nginx --image=nginx --replicas=3
kubectl get deployments
NAME    READY   UP-TO-DATE   AVAILABLE   AGE
nginx   3/3     3            3           9s 
```

**Le Scheduler**

- Déplacer le fichier kube-scheduler.yaml afin de s’assurer qu’aucune modification ne soit effectuée par ce dernier. Vérifier que le pod ne tourne plus.

```
sudo mv /etc/kubernetes/manifests/kube-scheduler.yaml /tmp
kubectl get pods -n kube-system
```

- Créer un pod nginx en créant un fichier nginx.yaml dans manifest pour pouvoir y apporter des modifications.

`kubectl run nginx --image=nginx --dry-run=client -o yaml > ~/nginx.yaml`

- Créer le pod nginx

```
kubectl apply -f ~/nginx.yaml
pod/nginx created
```

- Vérifier que le pod s’est bien créé. Le pod est en état pending.

```
kubectl get pods
NAME    READY   STATUS    RESTARTS   AGE 
nginx   0/1     Pending   0          2m29s
```

**Node selector**

- Utiliser un label pour scheduler un pod. Ajouter un label à un node existant

```
kubectl label node nathan-kube-node-1 node-type=worker
node/nathan-kube-node-1 labeled
```

- Créer un pod avec le node selector

```
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: nginx
spec:
  containers:
  - name: nginx
    image: nginx
  nodeSelector:
    node-type: worker
EOF

pod/nginx created
```

- Vérification du pod

```
kubectl get pods -o wide
```

- Supprimer le pod

```
kubectl delete pod nginx
pod "nginx" deleted
```

**Taints and tolerations**

- Le node control plane a par défaut un taint.

```
kubectl describe node nathan-kube-controlplane | grep Taints
Taints:             node-role.kubernetes.io/control-plane:NoSchedule
```

- Créer un pod qui fonctionne sur le node control plane

```
cat<<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: nginx
spec:
    containers:
    - name: nginx
      image: nginx
    tolerations:
    - key: node-role.kubernetes.io/master
      operator: Equal
      value: ""
    - key: node-role.kubernetes.io/control-plane
      operator: Equal
      value: ""
    nodeSelector:
      node-role.kubernetes.io/control-plane: ""
EOF

pod/nginx created
```

- Vérification du pod

```
kubectl get pods
NAME    READY   STATUS    RESTARTS   AGE
nginx   1/1     Running   0          36s
```

- Supprimer le pod

```
kubectl delete pod nginx
pod "nginx" deleted
```

- Recréer le pod sur le control plane et vérifier son état. Le pod reste en état pending car il n’y a pas de tolérance.

```
cat<<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: nginx
spec:
    containers:
    - name: nginx
      image: nginx
    nodeSelector:
      node-role.kubernetes.io/control-plane: ""
EOF
```

```
pod/nginx created
kubectl get pods
NAME    READY   STATUS    RESTARTS   AGE
nginx   0/1     Pending   0          46s
```

- Supprimer le pod

```
kubectl delete pod nginx
pod "nginx" deleted
```


**Upgrade worker nodes**

J'ai pas trop compris cette partie 

