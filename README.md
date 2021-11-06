# K8s Cluster on Photon OS (Hyper-V Instructions)
If you have Windows 10/11 Hyper-V, and want to experience Kubernetes cluster without using standalone clusters (_Minikube_,  _K3s_,  _MicroK8s_), then follow this guide. If you want to experience standalone clusters, then please refer below URLs.
1. Minikube (https://github.com/kubernetes/minikube/)
2. K3s (https://k3s.io/)
3. MicroK8s (https://microk8s.io/)
  
## Enable Hyper-V using PowerShell on Windows 10/11 Pro/Enterprise
Open a PowerShell console as Administrator. Run the following command:

```
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




