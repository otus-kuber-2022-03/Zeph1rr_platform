# Подготовка

развернут 1 `master` и 3 `worker` c помощью `terraform` - [infra](infra/)

```
➜  kubernetes-production-clusters git:(kubernetes-production-clusters) ✗ gcloud compute instances list                                         
NAME      ZONE            MACHINE_TYPE   PREEMPTIBLE  INTERNAL_IP  EXTERNAL_IP     STATUS
master-0  europe-west1-b  n1-standard-2               10.132.0.43  34.78.182.17    RUNNING
worker-0  europe-west1-b  n1-standard-1               10.132.0.41  34.78.86.155    RUNNING
worker-1  europe-west1-b  n1-standard-1               10.132.0.42  35.241.144.104  RUNNING
worker-2  europe-west1-b  n1-standard-1               10.132.0.44  34.76.136.216   RUNNING
```

на всех нодах выполнен скрипт - [run.sh](run.sh).

---

# Создание кластера

на мастере
```
kubeadm init --pod-network-cidr=192.168.0.0/24
```

```
  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config
```
```
root@master-0:~# kubectl get nodes
NAME       STATUS     ROLES    AGE     VERSION
master-0   NotReady   master   4m10s   v1.17.4
```
устанавливаем calico
```
kubectl apply -f https://docs.projectcalico.org/manifests/calico.yaml
```

добавляем ноды
```
kubeadm join 10.132.0.43:6443 --token 3yfonj.jke7kufzxgzogefa --discovery-token-ca-cert-hash sha256:fc63458bc01476316a192e965be4f748306bc66dd97a5585accca82df9ab7c36
```

```
root@master-0:~# kubectl get nodes
NAME       STATUS   ROLES    AGE     VERSION
master-0   Ready    master   8m37s   v1.17.4
worker-0   Ready    <none>   42s     v1.17.4
worker-1   Ready    <none>   41s     v1.17.4
worker-2   Ready    <none>   42s     v1.17.4
```
```
kubectl apply -f deployment.yaml
```
```
kubectl get po -o wide
NAME                               READY   STATUS    RESTARTS   AGE   IP              NODE       NOMINATED NODE   READINESS GATES
nginx-deployment-c8fd555cc-6lzjg   1/1     Running   0          37s   192.168.0.193   worker-2   <none>           <none>
nginx-deployment-c8fd555cc-wqcg8   1/1     Running   0          37s   192.168.0.130   worker-0   <none>           <none>
nginx-deployment-c8fd555cc-x7m9c   1/1     Running   0          37s   192.168.0.129   worker-0   <none>           <none>
nginx-deployment-c8fd555cc-zw4s8   1/1     Running   0          37s   192.168.0.1     worker-1   <none>           <none>
```

---

# Обновление

на мастере
```
apt-get update && apt-get install -y kubeadm=1.18.0-00 \
kubelet=1.18.0-00 kubectl=1.18.0-00
```
```
root@master-0:~# kubectl get nodes
NAME       STATUS   ROLES    AGE   VERSION
master-0   Ready    master   19m   v1.18.0
worker-0   Ready    <none>   11m   v1.17.4
worker-1   Ready    <none>   11m   v1.17.4
worker-2   Ready    <none>   11m   v1.17.4
```

обновление
```
kubeadm upgrade plan
kubeadm upgrade apply v1.18.0
```


выводим ноды из кластера
```
kubectl drain worker-0 --ignore-daemonsets
kubectl drain worker-1 --ignore-daemonsets
kubectl drain worker-2 --ignore-daemonsets
```

на worker нодах выполняем
```
apt-get install -y kubelet=1.18.0-00 kubeadm=1.18.0-00
systemctl restart kubelet
kubectl uncordon worker-0
kubectl uncordon worker-1
kubectl uncordon worker-2
```

```
root@master-0:~# kubectl get nodes
NAME       STATUS   ROLES    AGE   VERSION
master-0   Ready    master   42m   v1.18.0
worker-0   Ready    <none>   34m   v1.18.0
worker-1   Ready    <none>   34m   v1.18.0
worker-2   Ready    <none>   34m   v1.18.0
```

---

# Kubespray

```
git clone https://github.com/kubernetes-sigs/kubespray.git
cd kubespray/
sudo pip install -r requirements.txt
cp -rfp inventory/sample inventory/mycluster
```
отредактирован [inventory.ini](inventory.ini)

установка кластера
```
ansible-playbook -i inventory/mycluster/inventory.ini --become --become-user=root \
--user=${SSH_USERNAME} --key-file=${SSH_PRIVATE_KEY} cluster.yml
```

```
root@node1:~# kubectl get nodes
NAME    STATUS   ROLES    AGE     VERSION
node1   Ready    master   6m10s   v1.19.2
node2   Ready    <none>   4m56s   v1.19.2
node3   Ready    <none>   4m56s   v1.19.2
node4   Ready    <none>   4m56s   v1.19.2
```
---