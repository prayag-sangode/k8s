# MicroK8s Installation

This lab demonstrates installation of microk8s:


# Pre-Requisites

Minimum VM:

| Resource | Recommended |
| -------- | ----------- |
| CPU      | 2 vCPU      |
| RAM      | 4 GB        |
| Disk     | 20 GB       |

OS:

* Ubuntu 22.04 / 24.04



# Step 1: Install MicroK8s

Install MicroK8s:

```bash id="dx4azt"
sudo snap install microk8s --classic
```

Add current user:

```bash id="n84uyv"
sudo usermod -a -G microk8s $USER
newgrp microk8s
```

Verify:

```bash id="4l3xb7"
microk8s status --wait-ready
```



# Step 2: Enable Required Addons

Enable DNS:

```bash id="9vc0cw"
microk8s enable dns
```

Enable Helm:

```bash id="5l1mbx"
microk8s enable helm3
```

Enable MetalLB:

```bash id="s0r4v6"
microk8s enable metallb
```

You will be asked for IP range.

Example:

```text id="n14exm"
192.168.1.240-192.168.1.250
```

IMPORTANT:

Choose free IPs from your LAN subnet.



# Step 3: Configure kubectl Alias

```bash id="2a0f0k"
alias kubectl='microk8s kubectl'
```

Persist:

```bash id="h2axjm"
echo "alias kubectl='microk8s kubectl'" >> ~/.bashrc
source ~/.bashrc
```

Verify:

```bash id="7b6xhy"
kubectl get nodes
```




