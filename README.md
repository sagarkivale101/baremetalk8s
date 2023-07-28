
steps to create baremetal k8s
--------------------------------------------------


os => VERSION="18.04.6 LTS (Bionic Beaver server)


-------------------------------------------------------------------------------------------------------------------------------------------
step 0

turn off swap =>

vim /etc/fstab 

sed -e '/swap/ s/^#*/#/' -i /etc/fstab

#/swap.img      none    swap    sw      0       0


$ swapoff -a

$ free -h



-------------------------------------------------------------------------------------------------------------------------------------------
step 1

docker install =>

sudo apt-get update
sudo apt-get install \
    ca-certificates \
    curl \
    gnupg \
    lsb-release
sudo mkdir -p /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg

echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
  $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

sudo apt-get update


VERSION_STRING=5:19.03.10~3-0~ubuntu-focal

sudo apt-get install docker-ce=$VERSION_STRING docker-ce-cli=$VERSION_STRING containerd.io docker-compose-plugin -y

sudo docker run hello-world


-------------------------------------------------------------------------------------------------------------------------------------------
step 2 
install kubelet kubeadm kubectl => 



sudo apt-get update
sudo apt-get install -y apt-transport-https ca-certificates curl
sudo curl -fsSLo /usr/share/keyrings/kubernetes-archive-keyring.gpg https://packages.cloud.google.com/apt/doc/apt-key.gpg
echo "deb [signed-by=/usr/share/keyrings/kubernetes-archive-keyring.gpg] https://apt.kubernetes.io/ kubernetes-xenial main" | sudo tee /etc/apt/sources.list.d/kubernetes.list
sudo apt-get update
sudo apt-get install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl





-------------------------------------------------------------------------------------------------------------------------------------------
step 3

initiate kubeadm =>
 
kubeadm init --pod-network-cidr 192.168.0.0/16   => for calico

kubeadm init --pod-network-cidr=10.244.0.0/16


-----------------------------------------------------------------
ERROR =>
container runtime is not running =>

vim /etc/containerd/config.toml

sed -e '/disabled_plugins/ s/^#*/#/' -i /etc/containerd/config.toml

#disabled_plugins = ["cri"]
 
          OR

rm /etc/containerd/config.toml
systemctl restart containerd

https://github.com/containerd/containerd/issues/4581
-------------------------------------------------------------------

kubectl taint nodes --all node-role.kubernetes.io/control-plane:NoSchedule-
kubectl taint nodes --all node.kubernetes.io/not-ready:NoSchedule-


kubectl apply -f https://docs.projectcalico.org/manifests/calico.yaml



mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config

                or

export KUBECONFIG=/etc/kubernetes/admin.conf

-------------------------------------------------------------------------------------------------------------------------------------------


to join slaves => 

kubeadm join 192.168.10.16:6443 --token t0dh6d.xky6gkp3j6apxof1 \
	--discovery-token-ca-cert-hash sha256:acf46f7ed0b24a93be5e15bf54b606f5a811f1690175caea27f7e323480a1743

-----------------------------------------------

kubeadm token create --print-join-command

kubeadm token create --print-join-command


kubeadm token list => token

discovery-token-ca-cert-hash => sha256 + => sudo openssl x509 -in /etc/kubernetes/pki/ca.crt -noout -pubkey | openssl rsa -pubin -outform DER 2>/dev/null | sha256sum | cut -d' ' -f1













-------------------------------------------------------------------------------------------------------------------------------------------
NFS config

nfs install => https://www.linuxtechi.com/install-nfs-server-on-ubuntu/
nfs-external-provisioner => https://github.com/kubernetes-sigs/nfs-subdir-external-provisioner/tree/master/deploy

guide => https://youtu.be/DF3v2P8ENEg


----------

on master

sudo apt update
sudo apt install nfs-kernel-server
sudo systemctl status nfs-server
sudo mkdir /mnt/k8sdata
sudo chown nobody:nogroup /mnt/k8sdata
sudo chmod -R 777 /mnt/k8sdata


sudo vi /etc/exports => /mnt/k8sdata *(rw,sync,no_subtree_check)

sudo exportfs -a



------------
on slaves

sudo apt update
sudo apt install nfs-common
sudo mkdir -p /mnt/slavek8sdata
sudo mount 192.168.2.103:/mnt/k8sdata /mnt/slavek8sdata

--------------

apiVersion: v1
kind: ServiceAccount
metadata:
  name: nfs-client-provisioner
  # replace with namespace where provisioner is deployed
  namespace: default
---
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: nfs-client-provisioner-runner
rules:
  - apiGroups: [""]
    resources: ["nodes"]
    verbs: ["get", "list", "watch"]
  - apiGroups: [""]
    resources: ["persistentvolumes"]
    verbs: ["get", "list", "watch", "create", "delete"]
  - apiGroups: [""]
    resources: ["persistentvolumeclaims"]
    verbs: ["get", "list", "watch", "update"]
  - apiGroups: ["storage.k8s.io"]
    resources: ["storageclasses"]
    verbs: ["get", "list", "watch"]
  - apiGroups: [""]
    resources: ["events"]
    verbs: ["create", "update", "patch"]
---
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: run-nfs-client-provisioner
subjects:
  - kind: ServiceAccount
    name: nfs-client-provisioner
    # replace with namespace where provisioner is deployed
    namespace: default
roleRef:
  kind: ClusterRole
  name: nfs-client-provisioner-runner
  apiGroup: rbac.authorization.k8s.io
---
kind: Role
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: leader-locking-nfs-client-provisioner
  # replace with namespace where provisioner is deployed
  namespace: default
rules:
  - apiGroups: [""]
    resources: ["endpoints"]
    verbs: ["get", "list", "watch", "create", "update", "patch"]
---
kind: RoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: leader-locking-nfs-client-provisioner
  # replace with namespace where provisioner is deployed
  namespace: default
subjects:
  - kind: ServiceAccount
    name: nfs-client-provisioner
    # replace with namespace where provisioner is deployed
    namespace: default
roleRef:
  kind: Role
  name: leader-locking-nfs-client-provisioner
  apiGroup: rbac.authorization.k8s.io

---

apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: nfs-client
  annotations:
          storageclass.kubernetes.io/is-default-class: "true"
provisioner: k8s-sigs.io/nfs-subdir-external-provisioner # or choose another name, must match deployment's env PROVISIONER_NAME'
parameters:
  archiveOnDelete: "false"

---

apiVersion: apps/v1
kind: Deployment
metadata:
  name: nfs-client-provisioner
  labels:
    app: nfs-client-provisioner
  # replace with namespace where provisioner is deployed
  namespace: default
spec:
  replicas: 1
  strategy:
    type: Recreate
  selector:
    matchLabels:
      app: nfs-client-provisioner
  template:
    metadata:
      labels:
        app: nfs-client-provisioner
    spec:
      serviceAccountName: nfs-client-provisioner
      containers:
        - name: nfs-client-provisioner
          image: k8s.gcr.io/sig-storage/nfs-subdir-external-provisioner:v4.0.2
          volumeMounts:
            - name: nfs-client-root
              mountPath: /persistentvolumes
          env:
            - name: PROVISIONER_NAME
              value: k8s-sigs.io/nfs-subdir-external-provisioner
            - name: NFS_SERVER
              value: 192.168.1.203
            - name: NFS_PATH
              value: /mnt/k8sdata
      volumes:
        - name: nfs-client-root
          nfs:
            server: 192.168.1.203
            path: /mnt/k8sdata


---

## test deployment

---

kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: test-claim
spec:
  storageClassName: nfs-client      # same as storageclass name
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 100Mi

---

kind: Pod
apiVersion: v1
metadata:
  name: test-pod
spec:
  containers:
  - name: test-pod
    image: busybox:stable
    command:
      - "/bin/sh"
    args:
      - "-c"
      - "sleep 360000"
    volumeMounts:
      - name: nfs-pvc
        mountPath: "/mnt"
  restartPolicy: "Never"
  volumes:
    - name: nfs-pvc
      persistentVolumeClaim:
        claimName: test-claim



--------------------------------------------------------------------------------------------------------------------------------------------------

metallb =>

site => https://metallb.universe.tf/configuration/
guide => https://youtu.be/LMOYOtzpoXg


steps =>



kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.13.7/config/manifests/metallb-native.yaml


apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: first-pool
  namespace: metallb-system
spec:
  addresses:
  - 192.168.10.60-192.168.10.70       # 192.168.10.60 is in range  of node's ip in k8s 

---

apiVersion: metallb.io/v1beta1
kind: L2Advertisement
metadata:
  name: example
  namespace: metallb-system
spec:
  ipAddressPools:
  - first-pool



---


2nd loadbalancer

https://github.com/openelb/openelb

---
apiVersion: apps/v1 # for versions before 1.9.0 use apps/v1beta2
kind: Deployment
metadata:
  name: nginx
spec:
  strategy:
    type: Recreate
  selector:
    matchLabels:
      app: nginx
  replicas: 3 # tells deployment to run 1 pods matching the template
  template: # create pods using pod definition in this template
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx
        ports:
        - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: nginx
  namespace: default
  labels:
    app: nginx
  annotations:
    service.beta.kubernetes.io/aws-load-balancer-type: "nlb"
spec:
  externalTrafficPolicy: Local
  ports:
  - name: http
    port: 80
    protocol: TCP
    targetPort: 80
  selector:
    app: nginx
  type: LoadBalancer
