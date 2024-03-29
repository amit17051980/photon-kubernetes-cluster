# K8s Cluster on PhotonOS (Hyper-V Instructions)
If you have Windows 10/11 Hyper-V, and want to experience Kubernetes cluster without using standalone clusters (_Minikube_,  _K3s_,  _MicroK8s_), then follow this guide. If you want to experience standalone clusters, then please refer below URLs.
1. Minikube (https://github.com/kubernetes/minikube/)
2. K3s (https://k3s.io/)
3. MicroK8s (https://microk8s.io/)
  
## Enable Hyper-V using PowerShell on Windows 10/11 Pro/Enterprise
Open a PowerShell console as Administrator. Run the following command:

```
Set-ExecutionPolicy -ExecutionPolicy Unrestricted -Force
Enable-WindowsOptionalFeature -Online -FeatureName Microsoft-Hyper-V -All
```

If the command couldn't be found, make sure you're running PowerShell as Administrator.
When the installation has completed, reboot.
 
## Setup Hyper-V Internal Switch and assign IP
On the Windows desktop, click the Start button and type any part of the name Windows PowerShell.
Right-click Windows PowerShell and select Run as Administrator.

To create an internal switch, run the following command.

```
New-VMSwitch -name K8sInternalSwitch -SwitchType Internal
```

To assign an IP, run the following command.

```
New-NetIPAddress -InterfaceAlias 'vEthernet (K8sInternalSwitch)' -IPAddress 10.0.0.1 -PrefixLength 16
```

## Create PhotonOS VMs (kube-master and kube-node)
To download PhotonOS, use the following link.
https://packages.vmware.com/photon/4.0/Rev1/iso/photon-minimal-4.0-ca7c9e933.iso

Move the `iso` file to a known location. The path `C:\Media-Files\photon-minimal-4.0-ca7c9e933.iso` has been assumed for the commands below.

To create Kube-Master and Kube-Node VMs, run following command.

```
Set-ExecutionPolicy -ExecutionPolicy Unrestricted -Force
# Master-1 Creation
New-VM -Name Kube-Master-1 -MemoryStartupBytes 2GB -BootDevice VHD -NewVHDPath 'C:\Users\Public\Documents\Hyper-V\Virtual hard disks\Kube-Master-1.vhdx' -NewVHDSizeBytes 40GB -Generation 1 -Switch 'Default Switch'
Set-VMProcessor -VMName Kube-Master-1 -Count 2
Get-VM -Name Kube-Master-1 | Add-VMNetworkAdapter -SwitchName 'K8sInternalSwitch'
Add-VMDvdDrive -VMName Kube-Master-1 -Path 'C:\Media-Files\photon-minimal-4.0-ca7c9e933.iso'

# Node-1 Creation
New-VM -Name Kube-Node-1 -MemoryStartupBytes 4GB -BootDevice VHD -NewVHDPath 'C:\Users\Public\Documents\Hyper-V\Virtual hard disks\Kube-Node-1.vhdx' -NewVHDSizeBytes 80GB -Generation 1 -Switch 'Default Switch'
Set-VMProcessor -VMName Kube-Node-1 -Count 2
Get-VM -Name Kube-Node-1 | Add-VMNetworkAdapter -SwitchName 'K8sInternalSwitch'
Add-VMDvdDrive -VMName Kube-Node-1 -Path 'C:\Media-Files\photon-minimal-4.0-ca7c9e933.iso'

# Start and Conect to VMs
Start-VM -Name Kube-Master-1
Start-VM -Name Kube-Node-1
vmconnect localhost Kube-Master-1
vmconnect localhost Kube-Node-1
```

To install PhotonOS on these VMs follow below step-by-step instructions. The instructions shown below were applied to both the VMs in parallel.
If PhotonOS team release Hyper-V compatible VHD, the instructions below can be reduced.


**1. Install**

![](Install.png)

**2. Accept the Agreement**


**3. Select Auto configuration**


**4. Click Next**


**5. Enter Host Name in lower case (Same as VM Name)**


**6. Enter Password and Confirm Password**


**7. Select Yes to start the Installation**


**8. Wait for 40 seconds, and press any key**

![](Finish.png) 

**9. Wait 2-5 minutes. Login as 'root' User, and update software.**

```
tdnf -y update
```

**10. Enable Remote Login**

```
sed -i 's/PermitRootLogin no/PermitRootLogin yes/g' /etc/ssh/sshd_config
systemctl restart sshd
```

**11. Get the IP (eth0 IPv4) address to follow next steps**

```
ip a |grep 'dynamic eth0'
```

## Assign static IP using K8sInternalSwitch Gateway IP on VMs

### Kube-Master VM
  1. Connect to Kube Master IP (found in previous step) using PowerShell ssh client.
  ```
  ssh root@{IP OF Kube Master}
  ```
  2. Assign Static IP
  ```
  networkctl

  cat > /etc/systemd/network/10-eth1-static-en.network << "EOF"
  [Match]
  Name=eth1

  [Network]
  Address=10.0.0.10

  EOF

  chmod 644 /etc/systemd/network/10-eth1-static-en.network

  systemctl restart systemd-networkd
  ```
  3. Set Local Time Zone
  ```
  ln -sf /usr/share/zoneinfo/Europe/London /etc/localtime && echo "Europe/London" > /etc/timezone
  ```
  4. Add Kube-Node IP into hosts file
  ```
  echo "10.0.0.11    kube-node-1" >> /etc/hosts
  ```
### Kube-Node VM
  1. Connect to Kube Node IP (found in previous step) using PowerShell ssh client.
  ```
  ssh root@{IP OF Kube Node}
  ```
  2. Assign Static IP
  ```
  networkctl

  cat > /etc/systemd/network/10-eth1-static-en.network << "EOF"
  [Match]
  Name=eth1

  [Network]
  Address=10.0.0.11

  EOF

  chmod 644 /etc/systemd/network/10-eth1-static-en.network

  systemctl restart systemd-networkd
  ```
  3. Set Local Time Zone
  ```
  ln -sf /usr/share/zoneinfo/Europe/London /etc/localtime && echo "Europe/London" > /etc/timezone
  ```
  4. Add Kube-Master IP into hosts file
  ```
  echo "10.0.0.10    kube-master-1" >> /etc/hosts
  ```

## Add firewall rules for some important ports

### Kube-Master VM
```
iptables -A INPUT -i eth1 -p tcp --dport 6443 -j ACCEPT
iptables -A INPUT -i eth1 -p tcp --dport 2379:2380 -j ACCEPT
iptables -A INPUT -i eth1 -p tcp --dport 10248 -j ACCEPT
iptables -A INPUT -i eth1 -p tcp --dport 10250 -j ACCEPT
iptables -A INPUT -i eth1 -p tcp --dport 10259 -j ACCEPT
iptables -A INPUT -i eth1 -p tcp --dport 10257 -j ACCEPT
```

### Kube-Node VM
```
iptables -A INPUT -i eth1 -p tcp --dport 30000:32767 -j ACCEPT
iptables -A INPUT -i eth1 -p tcp --dport 10248 -j ACCEPT
iptables -A INPUT -i eth1 -p tcp --dport 10250 -j ACCEPT
```

## Kernel Configuration

### Kube-Master & Kube-Node VMs
```
cat <<EOF | sudo tee /etc/sysctl.d/kubernetes.conf
net.ipv4.ip_forward = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
EOF

sysctl --system
```

## Configure docker daemon to use systemd as the cgroup driver

### Kube-Master & Kube-Node VMs
```
systemctl enable docker
systemctl start docker
systemctl stop docker

cat <<EOF | sudo tee /etc/docker/daemon.json
{
  "exec-opts": ["native.cgroupdriver=systemd"],
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "100m"
  },
  "storage-driver": "overlay2"
}
EOF

systemctl daemon-reload
systemctl start docker
```

## Install Kubeadm, Kubelet and Kubectl

### Kube-Master & Kube-Node VMs
```
cat <<EOF | sudo tee /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://packages.cloud.google.com/yum/repos/kubernetes-el7-\$basearch
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://packages.cloud.google.com/yum/doc/yum-key.gpg https://packages.cloud.google.com/yum/doc/rpm-package-key.gpg
exclude=kubelet kubeadm kubectl
EOF

tdnf install -y kubelet kubeadm kubectl

systemctl enable --now kubelet
```

## Configure Kubernetes Cluster in Kube-Master
```
sudo kubeadm init --apiserver-advertise-address=10.0.0.10  --apiserver-cert-extra-sans=10.0.0.10  --node-name kube-master-1
```

Save the output to a secure file. It consist the cluster joining information for Kube-Node.
E.g.,
```
You should now deploy a pod network to the cluster.
Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
  https://kubernetes.io/docs/concepts/cluster-administration/addons/

Then you can join any number of worker nodes by running the following on each as root:

kubeadm join 10.0.0.10:6443 --token srs3x0.xc9xo9w5izfwwt4l \
        --discovery-token-ca-cert-hash sha256:6a53b891dfdfe10e3997e83fc0c5f0d35fd2ac4451f4e71317e98bf997157abf
```

If you missed to record the commands as shown above, execute the following command in the master node to recreate the token with the join command.
```
kubeadm token create --print-join-command
```

To start using your cluster, you need to run the following as a regular user:
  ```
  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config
  ```
Alternatively, if you are the root user, you can run:
  ```
  export KUBECONFIG=/etc/kubernetes/admin.conf
  ```

## Install calico network plugin on the cluster
```
kubectl apply -f https://docs.projectcalico.org/manifests/calico.yaml
```
Wait for 1-2 minutes, and then run the command below to see everything is running.
```
kubectl get po -n kube-system
```

## Join Kube-Node to the Cluster
Run the command (_noted from the output during Cluster Creation_) in Kube-Node to join Node into the cluster. E.g.,

```
kubeadm join 10.0.0.10:6443 --token srs3x0.xc9xo9w5izfwwt4l \
        --discovery-token-ca-cert-hash sha256:6a53b891dfdfe10e3997e83fc0c5f0d35fd2ac4451f4e71317e98bf997157abf
```

To verify the successful operation, run `kubectl get nodes` on the Kube-Master node.

E.g (Output).,
```
root@kube-master-1 [ ~ ]# kubectl get nodes
NAME            STATUS   ROLES                  AGE     VERSION
kube-master-1   Ready    control-plane,master   68m     v1.22.3
kube-node-1     Ready    <none>                 2m12s   v1.22.3
```

## Deploy A Sample Nginx Application
```
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  selector:
    matchLabels:
      app: nginx
  replicas: 2 
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:latest
        ports:
        - containerPort: 80      
EOF
```

```
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
spec:
  selector: 
    app: nginx
  type: NodePort  
  ports:
    - port: 80
      targetPort: 80
      nodePort: 32000
EOF
```
Once the deployment is up, you should be able to access Nginx home page on the allocated NodePort.

e.g.,
If you have assigned an external switch to the Node VM so that you can access from another computer in your home network.
Get the IP like below.
```
ip a|grep 192.168.0
---
inet 192.168.0.37/24 brd 192.168.0.255 scope global dynamic eth2
---
```
Access the service like url : http://192.168.0.37:3200

If you are on the Windows PC where Hyper-V is running.
Access the service like url : http://10.0.0.11:3200


## Install Kubernetes Dashboard (Optional)
```
kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/v2.4.0/aio/deploy/recommended.yaml

cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: ServiceAccount
metadata:
  name: admin-user
  namespace: kubernetes-dashboard
EOF

cat <<EOF | kubectl apply -f -
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: admin-user
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
- kind: ServiceAccount
  name: admin-user
  namespace: kubernetes-dashboard
EOF
```

Get the token of the user (created above)
```
kubectl -n kubernetes-dashboard get secret $(kubectl -n kubernetes-dashboard get sa/admin-user -o jsonpath="{.secrets[0].name}") -o go-template="{{.data.token | base64decode}}"
```

To access Kubernetes Dashboard from remote machine using above token, follow the instructions below.

```
kubectl patch svc kubernetes-dashboard -n kubernetes-dashboard -p '{"spec": {"type": "LoadBalancer", "externalIPs":["10.0.0.10"]}}'
```

Use the url like : https://10.0.0.10/

## Next Steps
1. Follow the Apendix to create Kubernetes Client if you prefer using another VM as a Kube Client to interact with Cluster
2. Follow the Apendix to create Private Docker Registry Secret to be used by Cluster
3. Review the References to achieve more.

## Appendix (Kubernetes Client VM)

1. Create a VM
```
# Client-1 Creation
New-VM -Name Kube-Client-1 -MemoryStartupBytes 4GB -BootDevice VHD -NewVHDPath 'C:\Users\Public\Documents\Hyper-V\Virtual hard disks\Kube-Client-1.vhdx' -NewVHDSizeBytes 80GB -Generation 1 -Switch 'Default Switch'
Set-VMProcessor -VMName Kube-Client-1 -Count 2
Get-VM -Name Kube-Client-1 | Add-VMNetworkAdapter -SwitchName 'K8sInternalSwitch'
Add-VMDvdDrive -VMName Kube-Client-1 -Path 'C:\Media-Files\photon-minimal-4.0-ca7c9e933.iso'

# Start and Conect to VMs
Start-VM -Name Kube-Client-1
vmconnect localhost Kube-Client-1
```
2. Enable Remote Login
```
sed -i 's/PermitRootLogin no/PermitRootLogin yes/g' /etc/ssh/sshd_config
systemctl restart sshd
```
3. Get the IP (eth0 IPv4) address to follow next steps
```
ip a |grep 'dynamic eth0'
```
4. Assign static IP using K8sInternalSwitch Gateway IP on VMs
```
networkctl

cat > /etc/systemd/network/10-eth1-static-en.network << "EOF"
[Match]
Name=eth1

[Network]
Address=10.0.0.12

EOF

chmod 644 /etc/systemd/network/10-eth1-static-en.network

systemctl restart systemd-networkd
```
5. Set Local Time Zone
```
ln -sf /usr/share/zoneinfo/Europe/London /etc/localtime && echo "Europe/London" > /etc/timezone
```
6. Add Kube-Master IP into hosts file
```
echo "10.0.0.10    kube-master-1" >> /etc/hosts
```
7. Install Kubectl
```
cat <<EOF | sudo tee /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://packages.cloud.google.com/yum/repos/kubernetes-el7-x86_64
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://packages.cloud.google.com/yum/doc/yum-key.gpg https://packages.cloud.google.com/yum/doc/rpm-package-key.gpg
EOF
sudo yum install -y kubectl
```
8. Enable kubectl autocompletion
```
echo 'source <(kubectl completion bash)' >>~/.bashrc
```
9. Next step is to get kube configuration ready on this server so that interaction to the cluster can be establised. There are 2 options.
    >> 1. Create an user with right cluster and namespace privileges and use the kube config of the user in `~/.kube/config`. E.g.,
    <pre>
    # Namespace Creation
    ---
    apiVersion: v1
    kind: Namespace
    metadata:
      name: documentum
    
    # ServiceAccount Creation
    ---
    apiVersion: v1
    kind: ServiceAccount
    metadata:
      name: dmadmin
      namespace: documentum
      
    # Role Creation
    ---
    apiVersion: rbac.authorization.k8s.io/v1
    kind: Role
    metadata:
      name: documentum-role
      namespace: documentum
    rules:
    - apiGroups: ["*"]
      resources: ["*"]
      verbs: ["*"]
    
    # RoleBinding Creation
    ---
    apiVersion: rbac.authorization.k8s.io/v1
    kind: RoleBinding
    metadata:
      name: documentum-rolebinding
      namespace: documentum
    roleRef:
      apiGroup: rbac.authorization.k8s.io
      kind: Role
      name: documentum-role
    subjects:
    - namespace: documentum
      kind: ServiceAccount
      name: documentum
    
    # ClusterRoleBinding Creation
    ---
    apiVersion: rbac.authorization.k8s.io/v1
    kind: ClusterRoleBinding
    metadata:
      name: documentum-clusterrolebinding
    roleRef:
      apiGroup: rbac.authorization.k8s.io
      kind: ClusterRole
      name: cluster-admin
    subjects:
    - kind: ServiceAccount
      name: dmadmin
      namespace: documentum
    </pre>
    
    >> 2. Copy `~/.kube/config` file from Control Plane (Kube Master) to the same location in Kube Client.

10. Verify the connectivity to cluster
```
kubectl cluster-info
```

## Appendix (Configure Cluster to use Private Docker Registry)
The steps below are solely for demonstration purpose, but can be used for other private registry. There are ways to load external registry images to enterprise specific private registry and use from there. This is the case when Client PCs and Cluster are blocked to talk to external registry.

1. Perform Docker Login on Node/Master/Client where docker is installed and can talk to Kubernetes Cluster

```
docker login registry.opentext.com
```

Provide User and PAssword. You should see success message like below.
```
Username: ********
Password: ********
WARNING! Your password will be stored unencrypted in /root/.docker/config.json.
Configure a credential helper to remove this warning. See
https://docs.docker.com/engine/reference/commandline/login/#credentials-store

Login Succeeded
```

2. Create Secret from the config.json (created above).
```
kubectl create secret generic regcred \
    --from-file=.dockerconfigjson=/root/.docker/config.json \
    --type=kubernetes.io/dockerconfigjson
```

3. Delete /root/.docker/config.json


## References
1. https://devopscube.com/setup-kubernetes-cluster-kubeadm/
2. https://pirivan.gitlab.io/post/installing-kubernetes-on-the-photon-os/
3. https://docs.armory.io/docs/armory-admin/manual-service-account/
4. https://docs.bitnami.com/tutorials/configure-rbac-in-your-kubernetes-cluster/
5. https://kubernetes.io/docs/tasks/access-application-cluster/web-ui-dashboard/
6. https://cloud.google.com/kubernetes-engine/docs/how-to/exposing-apps#kubectl-apply
7. https://metallb.universe.tf/installation/
8. https://helm.sh/
9. https://www.thegeekdiary.com/how-to-access-kubernetes-dashboard-externally/
10. https://kubernetes.io/docs/tasks/configure-pod-container/pull-image-private-registry/
11. https://helm.sh/docs/topics/rbac/

