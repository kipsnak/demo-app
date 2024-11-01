# Notes

## Disclaimer

The current projet is a simple challenge task done to only demonstrate one of many way to deploy aplications on kubenetes infrastructure. This challenge aims to show how could a DevOps team provide a complete and ready to use to environment for your application. The project has no mean to be used in any "real world" environment.

## Contexts

### Applications
The following we are about to deploy two diffrents applications. First one is going to be a backend with `Spring Boot >=3.0` builded with `Maven`. The second is the frontend with React.

Since the goal of this demonstration is not the applications themselves, rather the way to handle them. I have picked other project from github with applications that fullfil my requirements above.


### Requirements

> [!note]  
> Please adapt your install for your own os :)


* Qemu
```bash
sudo apt -y install qemu-system
```

* Podman
```bash
sudo apt -y install podman
```

* libvirt
```bash
sudo apt install qemu-kvm libvirt-clients libvirt-daemon-system bridge-utils virtinst libvirt-daemon
```

## Cluster Kubernetes

References and sources :
* https://docs.fedoraproject.org/en-US/fedora-coreos/getting-started/
* https://dev.to/carminezacc/creating-a-kubernetes-cluster-with-fedora-coreos-36-j17
* https://fedoraproject.org/en/coreos/download?tab=metal_virtualized&stream=stable&arch=x86_64#arches
* https://discuss.kubernetes.io/t/facing-challanges-for-installation-of-kubernest-cluster-setup-through-kubeadm-on-rhel8-9-vm-404-for-https-packages-cloud-google-com-yum-repos-kubernetes-el7-x86-64-repodata-repomd-xml-ip-142-250-115-139/27345/2

The whole idea of our infrastructure is to provide 3 VMs with QEMU/KVM, then install and configure kubenetes cluster on fedora's `coreos`.

change directory to kubernetes
```bash
cd <project root dir>/kubernetes
```

Let's start with `coreos` image download, we need the QEMU image with qcow2 disk format.

> [!warning]  
> this is going to take a little while, the image is quite heavy... sorry!

```bash
podman run --rm -v $(pwd)/images/:/data -w /data quay.io/coreos/coreos-installer:release download -s stable -p qemu -f qcow2.xz --decompress
```
```
gpg: Signature made Sat Oct 26 14:50:00 2024 UTC
gpg:                using RSA key 115DF9AEF857853EE8445D0A0727707EA15B79CC
gpg: checking the trustdb
gpg: marginals needed: 3  completes needed: 1  trust model: pgp
gpg: depth: 0  valid:   4  signed:   0  trust: 0-, 0q, 0n, 0m, 0f, 4u
gpg: Good signature from "Fedora (40) <fedora-40-primary@fedoraproject.org>" [ultimate]
./fedora-coreos-40.20241019.3.0-qemu.x86_64.qcow2
```
`fedora-coreos-40.20241019.3.0-qemu.x86_64` is the current version. 

Make sur libevirt daemon is up and running
```bash
sudo systemctl status libvirtd
```

check the default network is active
```bash
sudo virsh net-list
 Name      State    Autostart   Persistent
--------------------------------------------
 default   active   no          yes
```

if not, run
```bash
sudo virsh net-start default
```

Using `butane` file, generate the ignition file
```bash
podman run --interactive --rm quay.io/coreos/butane:release --pretty --strict < fcos.yaml > fcos.ign
```

We have now our ignition file for the cluster, let's create the first node which will play the role of the control plane
```bash
export IMAGE="$(pwd)/image/fedora-coreos-40.20241019.3.0-qemu.x86_64.qcow2" # Put the downloaded version above
export IGN_FILE="$(pwd)/fcos.ign"

sudo ./start_node --node 1 --image ${IMAGE} -f ${IGN_FILE}
```

hit `ctrl + ]` to exit.

check node ip
```bash
sudo virsh domifaddr node-1 | grep -oE "\b([0-9]{1,3}\.){3}[0-9]{1,3}\b"
192.168.122.79
```

connect with ssh to the first node
```bash
ssh -i .ssh/id_rsa core@$(sudo virsh domifaddr node-1 | grep -oE "\b([0-9]{1,3}\.){3}[0-9]{1,3}\b")
```

Install kubenetes tools for the control plane
```bash
sudo -i # switch root
echo "node-1" > /etc/hostname
rpm-ostree install kubelet kubeadm kubectl cri-o
```

reboot
```bash
systemctl reboot
```

Then, create the second node
```bash
sudo ./start_node --node 2 --image ${IMAGE} -f ${IGN_FILE}
# ctrl + ] to exit
sudo -i
echo "node-2" > /etc/hostname
```

and third
```bash
sudo ./start_node --node 3 --image ${IMAGE} -f ${IGN_FILE}
# ctrl + ] to exit
sudo -i
echo "node-3" > /etc/hostname
```

Then for both
```bash
rpm-ostree install kubelet kubeadm cri-o
systemctl reboot
```

On all node enable cri-o and kubelet services
```bash
sudo systemctl enable --now crio kubelet
```

At this point, the node are ready to host the cluster. let's go back to the control plane, the first cluster
```bash
ssh -i .ssh/id_rsa core@$(sudo virsh domifaddr node-1 | grep -oE "\b([0-9]{1,3}\.){3}[0-9]{1,3}\b")
core@node-1:~$ sudo -i
```

check kubeadm version
```bash
kubeadm version
kubeadm version: &version.Info{Major:"1", Minor:"29", GitVersion:"v1.29.10", GitCommit:"f0c1ea863533246b6d3fda3e6addb7c13c8a6359", GitTreeState:"clean", BuildDate:"2024-10-22T20:34:11Z", GoVersion:"go1.22.8", Compiler:"gc", Platform:"linux/amd64"}
```

```bash
cat <<EOF > clusterconfig.yml
apiVersion: kubeadm.k8s.io/v1beta3
kind: ClusterConfiguration
kubernetesVersion: v1.29.0 # adjust with your kubeadm version
controllerManager:
  extraArgs: # specify a R/W directory for FlexVolumes (cluster won't work without this even though we use PVs)
    flex-volume-plugin-dir: "/etc/kubernetes/kubelet-plugins/volume/exec"
networking: # pod subnet definition
  podSubnet: 10.244.0.0/16
---
apiVersion: kubeadm.k8s.io/v1beta3
kind: InitConfiguration
EOF
```

Initiate the cluster
```bash
kubeadm init ––config clusterconfig.yml
```

Be careful, the `kubeadm init` will print the command line with a generated discovery token that allow workers to join the cluster.

Set up the kube-router
```bash
kubectl apply -f https://raw.githubusercontent.com/cloudnativelabs/kube-router/master/daemonset/kubeadm-kuberouter.yaml
```

Then use the token to let node-2 and node-3 join the cluster
```bash
sudo -i
kubeadm join 192.168.122.79:6443 --token jfy1ea.v2qunivd3u2t2jep \
        --discovery-token-ca-cert-hash sha256:a1e7f2fa7e1de3d31a9152864c5621128a0a36d2c7def0c6d2bdbac2aeeb3f48
```

check
```bash
core@node-1:~$ kubectl get nodes
```
```
NAME     STATUS   ROLES           AGE     VERSION
node-1   Ready    control-plane   10m     v1.29.10
node-2   Ready    <none>          4m30s   v1.29.10
node-3   Ready    <none>          4m18s   v1.29.10
```
