# Notes

![example workflow](https://github.com/kipsnak/demo-app/actions/workflows/docker-image.yml/badge.svg)

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
Create a network
```bash
cat <<EOF > default.xml
<network>
  <name>default</name>
  <bridge name='virbr0'/>
  <forward/>
  <ip address='192.168.122.1' netmask='255.255.255.0'>
    <dhcp>
      <range start='192.168.122.2' end='192.168.122.254'/>
    </dhcp>
  </ip>
</network>
EOF
```
```bash
sudo virsh net-define Kubernetes/network/default.xml
```
```bash
sudo virsh net-start default
```

Somehow, after a day and pc hebernation, could not associate the bridge on the host with my libvirt network
```
error: Failed to start network default
error: internal error: Network is already in use by interface virbr0
```

To fix this
```bash
# needs bridge-utils
# sudo apt install bridge-utils
sudo ip link set virbr0 down
sudo brctl delbr virbr0
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
sudo virsh net-autostart default
```

Using `butane` file, generate the ignition file
```bash
podman run --interactive --rm quay.io/coreos/butane:release --pretty --strict < fcos.yaml > fcos.ign
```

We have now our ignition file for the cluster, let's create the first node which will play the role of the control plane
```bash
export IMAGE="$(pwd)/images/fedora-coreos-40.20241019.3.0-qemu.x86_64.qcow2" # Put the downloaded version above
export IGN_FILE="$(pwd)/fcos.ign"

sudo ./start_node --node 1 --image ${IMAGE} -f ${IGN_FILE}
```

hit `ctrl + ]` to exit.

check node ip
```bash
sudo virsh domifaddr node-1 | grep -oE "\b([0-9]{1,3}\.){3}[0-9]{1,3}\b"
192.168.122.222
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
export IMAGE="$(pwd)/images/fedora-coreos-40.20241019.3.0-qemu.x86_64.qcow2" # Put the downloaded version above
export IGN_FILE="$(pwd)/fcos.ign"
sudo ./start_node --node 2 --image ${IMAGE} -f ${IGN_FILE}
# ctrl + ] to exit
sudo virsh domifaddr node-2 | grep -oE "\b([0-9]{1,3}\.){3}[0-9]{1,3}\b"
# 192.168.122.248
ssh -i .ssh/id_rsa core@$(sudo virsh domifaddr node-2 | grep -oE "\b([0-9]{1,3}\.){3}[0-9]{1,3}\b")
sudo -i
echo "node-2" > /etc/hostname
rpm-ostree install kubelet kubeadm cri-o
systemctl reboot
```

and finally the third node
```bash
export IMAGE="$(pwd)/images/fedora-coreos-40.20241019.3.0-qemu.x86_64.qcow2" # Put the downloaded version above
export IGN_FILE="$(pwd)/fcos.ign"
sudo ./start_node --node 3 --image ${IMAGE} -f ${IGN_FILE}
# ctrl + ] to exit
sudo virsh domifaddr node-3 | grep -oE "\b([0-9]{1,3}\.){3}[0-9]{1,3}\b"
# 192.168.122.27
ssh -i .ssh/id_rsa core@$(sudo virsh domifaddr node-3 | grep -oE "\b([0-9]{1,3}\.){3}[0-9]{1,3}\b")
sudo -i
echo "node-3" > /etc/hostname
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
kubeadm init --config clusterconfig.yml
```

Be careful, the `kubeadm init` will print the command line with a generated discovery token that allow workers to join the cluster.
```
Your Kubernetes control-plane has initialized successfully!

To start using your cluster, you need to run the following as a regular user:

  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config

Alternatively, if you are the root user, you can run:

  export KUBECONFIG=/etc/kubernetes/admin.conf

You should now deploy a pod network to the cluster.
Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
  https://kubernetes.io/docs/concepts/cluster-administration/addons/

Then you can join any number of worker nodes by running the following on each as root:

kubeadm join 192.168.122.222:6443 --token q7lnnh.4l3otpb53usf6urx \
        --discovery-token-ca-cert-hash sha256:32b04078d12e4accd27b7a96ae966dc87312b51e8a2d4eac174ee0e805f98e0a
```

configure kubectl kbe config
```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

Set up the kube-router
```bash
kubectl apply -f https://raw.githubusercontent.com/cloudnativelabs/kube-router/master/daemonset/kubeadm-kuberouter.yaml
```
```
configmap/kube-router-cfg created
daemonset.apps/kube-router created
serviceaccount/kube-router created
clusterrole.rbac.authorization.k8s.io/kube-router created
clusterrolebinding.rbac.authorization.k8s.io/kube-router created
```

Then use the token to let node-2 and node-3 join the cluster
```bash
sudo kubeadm join 192.168.122.222:6443 --token q7lnnh.4l3otpb53usf6urx \
        --discovery-token-ca-cert-hash sha256:32b04078d12e4accd27b7a96ae966dc87312b51e8a2d4eac174ee0e805f98e0a
```
```
[preflight] Running pre-flight checks
[preflight] Reading configuration from the cluster...
[preflight] FYI: You can look at this config file with 'kubectl -n kube-system get cm kubeadm-config -o yaml'
[kubelet-start] Writing kubelet configuration to file "/var/lib/kubelet/config.yaml"
[kubelet-start] Writing kubelet environment file with flags to file "/var/lib/kubelet/kubeadm-flags.env"
[kubelet-start] Starting the kubelet
[kubelet-start] Waiting for the kubelet to perform the TLS Bootstrap...

This node has joined the cluster:
* Certificate signing request was sent to apiserver and a response was received.
* The Kubelet was informed of the new secure connection details.

Run 'kubectl get nodes' on the control-plane to see this node join the cluster.
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

```bash
core@node-1:~$ kubectl get namespaces -A
```
```
NAME              STATUS   AGE
default           Active   11m
kube-node-lease   Active   11m
kube-public       Active   11m
kube-system       Active   11m
```

To use helm on the local host, we need to get the kubeconfig from our control plane

```bash
# if you have an existing kubeconfig back it up
# cp -a ~/.kube/config{,#back}
ssh -i .ssh/id_rsa core@$(sudo virsh domifaddr node-1 | grep -oE "\b([0-9]{1,3}\.){3}[0-9]{1,3}\b") "cat .kube/config" > ~/.kube/config
```

## Database

to emulate an external database, it would not be deployed on the k8s cluster. Rather, on a podman kube.

start with changing directory, and prepare where to put datas
```bash
cd <project root dir>
mkdir -p Database/data
```

let's create a pod which will host the database as well as the web client
```bash
podman network create demo-network
podman pod create --network=demo-network --name postgres-101 -p 9876:80 -p 5432:5432
```

check
```bash
podman pod ls
POD ID        NAME          STATUS      CREATED        INFRA ID      # OF CONTAINERS
956602f69c6b  postgres-101  Created     7 seconds ago  7b849c558966  1
```

add the pgadmin
```bash
podman run --pod=postgres-101 -e 'PGADMIN_DEFAULT_EMAIL=me@mine.me' -e 'PGADMIN_DEFAULT_PASSWORD=mysecret' --name pg-admin -d dpage/pgadmin4
```

then the database
```bash
podman run --pod=postgres-101 -v $(pwd)/data:/var/lib/postgresql/data:Z -e 'POSTGRES_PASSWORD=M3P@ssw0rd!' -e POSTGRES_USER=myapplication --name pg-db -d postgres
```

check
```bash
podman ps
CONTAINER ID  IMAGE                              COMMAND     CREATED         STATUS             PORTS                                         NAMES
7b849c558966  localhost/podman-pause:4.3.1-0                 5 minutes ago   Up 3 minutes ago   0.0.0.0:5432->5432/tcp, 0.0.0.0:9876->80/tcp  956602f69c6b-infra
5361d1bf8043  docker.io/dpage/pgadmin4:latest                4 minutes ago   Up 3 minutes ago   0.0.0.0:5432->5432/tcp, 0.0.0.0:9876->80/tcp  pg-admin
27495037d272  docker.io/library/postgres:latest  postgres    26 seconds ago  Up 22 seconds ago  0.0.0.0:5432->5432/tcp, 0.0.0.0:9876->80/tcp  pg-db
```

From pgadmin, create database named `employees`

## Backend

### Build image

Using multi-stage to reduce built image size.
```bash
cd <project root dir>/Backend
podman build -t demo-backend:latest .
```

### Connecting backend with database locally

check database network config
```bash
podman inspect pg-db | jq ".[0].NetworkSettings.Networks"
{
  "demo-network": {
    "EndpointID": "",
    "Gateway": "10.89.0.1",
    "IPAddress": "10.89.0.2",
    "IPPrefixLen": 24,
    "IPv6Gateway": "",
    "GlobalIPv6Address": "",
    "GlobalIPv6PrefixLen": 0,
    "MacAddress": "36:fe:99:9c:db:87",
    "NetworkID": "demo-network",
    "DriverOpts": null,
    "IPAMConfig": null,
    "Links": null,
    "Aliases": [
      "f4a6c332545e"
    ]
  }
}
```

Use its IP for `SPRING_DATASOURCE_URL` env variable
```bash
podman run --network=demo-network -it --rm \
    -e "SPRING_DATASOURCE_URL=jdbc:postgresql://10.89.0.2:5432/employees" \
    -e "SPRING_DATASOURCE_USER=myapplication" \
    -e "SPRING_DATASOURCE_PASSWORD=M3P@ssw0rd\!" -p 8080:8080 demo-backend:latest /bin/sh
```

check
```
  .   ____          _            __ _ _
 /\\ / ___'_ __ _ _(_)_ __  __ _ \ \ \ \
( ( )\___ | '_ | '_| | '_ \/ _` | \ \ \ \
 \\/  ___)| |_)| | | | | || (_| |  ) ) ) )
  '  |____| .__|_| |_|_| |_\__, | / / / /
 =========|_|==============|___/=/_/_/_/
 :: Spring Boot ::                (v3.0.4)

2024-11-01T05:01:11.203Z  INFO 1 --- [           main] n.j.s.SpringbootBackendApplication       : Starting SpringbootBackendApplication v0.0.1-SNAPSHOT using Java 17-ea with PID 1 (/app/runme.jar started by root in /)
2024-11-01T05:01:11.206Z  INFO 1 --- [           main] n.j.s.SpringbootBackendApplication       : No active profile set, falling back to 1 default profile: "default"
2024-11-01T05:01:11.770Z  INFO 1 --- [           main] .s.d.r.c.RepositoryConfigurationDelegate : Bootstrapping Spring Data JPA repositories in DEFAULT mode.
2024-11-01T05:01:11.822Z  INFO 1 --- [           main] .s.d.r.c.RepositoryConfigurationDelegate : Finished Spring Data repository scanning in 43 ms. Found 1 JPA repository interfaces.
2024-11-01T05:01:12.278Z  INFO 1 --- [           main] o.s.b.w.embedded.tomcat.TomcatWebServer  : Tomcat initialized with port(s): 8080 (http)
2024-11-01T05:01:12.285Z  INFO 1 --- [           main] o.apache.catalina.core.StandardService   : Starting service [Tomcat]
2024-11-01T05:01:12.285Z  INFO 1 --- [           main] o.apache.catalina.core.StandardEngine    : Starting Servlet engine: [Apache Tomcat/10.1.5]
2024-11-01T05:01:12.341Z  INFO 1 --- [           main] o.a.c.c.C.[Tomcat].[localhost].[/]       : Initializing Spring embedded WebApplicationContext
2024-11-01T05:01:12.343Z  INFO 1 --- [           main] w.s.c.ServletWebServerApplicationContext : Root WebApplicationContext: initialization completed in 1087 ms
2024-11-01T05:01:12.480Z  INFO 1 --- [           main] o.hibernate.jpa.internal.util.LogHelper  : HHH000204: Processing PersistenceUnitInfo [name: default]
2024-11-01T05:01:12.525Z  INFO 1 --- [           main] org.hibernate.Version                    : HHH000412: Hibernate ORM core version 6.1.7.Final
2024-11-01T05:01:12.728Z  INFO 1 --- [           main] com.zaxxer.hikari.HikariDataSource       : HikariPool-1 - Starting...
2024-11-01T05:01:12.886Z  INFO 1 --- [           main] com.zaxxer.hikari.pool.HikariPool        : HikariPool-1 - Added connection org.postgresql.jdbc.PgConnection@aec50a1
2024-11-01T05:01:12.888Z  INFO 1 --- [           main] com.zaxxer.hikari.HikariDataSource       : HikariPool-1 - Start completed.
2024-11-01T05:01:12.922Z  INFO 1 --- [           main] SQL dialect                              : HHH000400: Using dialect: org.hibernate.dialect.PostgreSQLDialect
2024-11-01T05:01:13.524Z  INFO 1 --- [           main] o.h.e.t.j.p.i.JtaPlatformInitiator       : HHH000490: Using JtaPlatform implementation: [org.hibernate.engine.transaction.jta.platform.internal.NoJtaPlatform]
2024-11-01T05:01:13.532Z  INFO 1 --- [           main] j.LocalContainerEntityManagerFactoryBean : Initialized JPA EntityManagerFactory for persistence unit 'default'
2024-11-01T05:01:13.714Z  WARN 1 --- [           main] JpaBaseConfiguration$JpaWebConfiguration : spring.jpa.open-in-view is enabled by default. Therefore, database queries may be performed during view rendering. Explicitly configure spring.jpa.open-in-view to disable this warning
2024-11-01T05:01:13.991Z  INFO 1 --- [           main] o.s.b.w.embedded.tomcat.TomcatWebServer  : Tomcat started on port(s): 8080 (http) with context path ''
2024-11-01T05:01:13.999Z  INFO 1 --- [           main] n.j.s.SpringbootBackendApplication       : Started SpringbootBackendApplication in 3.108 seconds (process running for 3.465)
```

## Frontend

For the frontend, i'll be using an autogenerated React app

see more info: https://github.com/facebook/create-react-app?tab=readme-ov-file#creating-an-app

_more details will come when i'm done debuging... sadge_

### update

Frot ends and client side languages are really not my cup of tea.

I tried to start from scrach while inspiring from an already fully developped app, but customizing something I don't master in a short span of time is quite hard actually.

So I took the already ready app and put it in as it is.

Just added a `Dockerfile` to build it.

Using multi-stage to reduce built image size.
```bash
cd <project root dir>/Frontend
podman build -t demo-frontend:latest .
```

```bash
podman run --rm -p 8080:80 testfront:latest
```
```
/docker-entrypoint.sh: /docker-entrypoint.d/ is not empty, will attempt to perform configuration
/docker-entrypoint.sh: Looking for shell scripts in /docker-entrypoint.d/
/docker-entrypoint.sh: Launching /docker-entrypoint.d/10-listen-on-ipv6-by-default.sh
10-listen-on-ipv6-by-default.sh: info: Getting the checksum of /etc/nginx/conf.d/default.conf
10-listen-on-ipv6-by-default.sh: info: Enabled listen on IPv6 in /etc/nginx/conf.d/default.conf
/docker-entrypoint.sh: Sourcing /docker-entrypoint.d/15-local-resolvers.envsh
/docker-entrypoint.sh: Launching /docker-entrypoint.d/20-envsubst-on-templates.sh
/docker-entrypoint.sh: Launching /docker-entrypoint.d/30-tune-worker-processes.sh
/docker-entrypoint.sh: Configuration complete; ready for start up
[ ... ]
10.0.2.100 - - [04/Nov/2024:19:12:56 +0000] "GET / HTTP/1.1" 304 0 "-" "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/130.0.0.0 Safari/537.36" "-"
10.0.2.100 - - [04/Nov/2024:19:12:56 +0000] "GET /static/css/2.af3c1da9.chunk.css HTTP/1.1" 304 0 "http://localhost:8080/" "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/130.0.0.0 Safari/537.36" "-"
10.0.2.100 - - [04/Nov/2024:19:12:56 +0000] "GET /static/css/main.941a7d5c.chunk.css HTTP/1.1" 304 0 "http://localhost:8080/" "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/130.0.0.0 Safari/537.36" "-"
10.0.2.100 - - [04/Nov/2024:19:12:56 +0000] "GET /static/js/2.adf17f9f.chunk.js HTTP/1.1" 304 0 "http://localhost:8080/" "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/130.0.0.0 Safari/537.36" "-"
10.0.2.100 - - [04/Nov/2024:19:12:56 +0000] "GET /static/js/main.57c28490.chunk.js HTTP/1.1" 304 0 "http://localhost:8080/" "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/130.0.0.0 Safari/537.36" "-"
2024/11/04 19:12:56 [error] 28#28: *4 open() "/usr/share/nginx/html/api/v1/employees" failed (2: No such file or directory), client: 10.0.2.100, server: localhost, request: "GET /api/v1/employees HTTP/1.1", host: "localhost:8080", referrer: "http://localhost:8080/"
10.0.2.100 - - [04/Nov/2024:19:12:56 +0000] "GET /api/v1/employees HTTP/1.1" 404 555 "http://localhost:8080/" "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/130.0.0.0 Safari/537.36" "-"
2024/11/04 19:12:56 [error] 29#29: *5 open() "/usr/share/nginx/html/manifest.json" failed (2: No such file or directory), client: 10.0.2.100, server: localhost, request: "GET /manifest.json HTTP/1.1", host: "localhost:8080", referrer: "http://localhost:8080/"
10.0.2.100 - - [04/Nov/2024:19:12:56 +0000] "GET /manifest.json HTTP/1.1" 404 555 "http://localhost:8080/" "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/130.0.0.0 Safari/537.36" "-"
```

## Helm

Packaging these apps requires first pushing images to a registry. So let's push the backend to docker hub.

start by generating a new token or use an old valide one.

### Manually push image to docker hub

```bash
# generate an access token on docker hub setting
podman login -u kipsnak docker.io
Password:
Login Succeeded!
```

retag local built image
```bash
podman tag localhost/demo-backend:latest docker.io/kipsnak/demo-backend:latest
podman tag localhost/demo-frontend:latest docker.io/kipsnak/demo-frontend:latest
```

push image to docker.io
```bash
podman push docker.io/kipsnak/demo-backend:latest
podman push docker.io/kipsnak/demo-frontend:latest
```

### Automatically build and push image to docker hub

TODO

### Helm templating

Let's check if the manifests are proprely generated
```bash
helm template . >  manifests.yaml
```

see `manifests.yaml` in the current repo

### Deploy app with helm

make sur we are connected to the cluster
```bash
 kubectl get nodes
NAME     STATUS   ROLES           AGE     VERSION
node-1   Ready    control-plane   6h3m    v1.29.10
node-2   Ready    <none>          5h58m   v1.29.10
node-3   Ready    <none>          5h58m   v1.29.10
```

Change directory to Helm package files
```bash
cd <project root dir>/Helm
```

install backend package
```bash
helm install --name-template demo-app .
```
```
NAME: demo-app
LAST DEPLOYED: Mon Nov  4 20:20:42 2024
NAMESPACE: default
STATUS: deployed
REVISION: 1
TEST SUITE: None
```

check
```bash
kubectl -n demo-app-ns get all
```
```
NAME                                       READY   STATUS    RESTARTS   AGE
pod/app-backend-deploy-75db86bc69-tptc9    1/1     Running   0          43s
pod/app-frontend-deploy-7ccd9849df-p87ld   1/1     Running   0          43s

NAME                       TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)    AGE
service/app-backend-svc    ClusterIP   10.100.33.156   <none>        8080/TCP   43s
service/app-frontend-svc   ClusterIP   10.102.97.78    <none>        80/TCP     43s

NAME                                  READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/app-backend-deploy    1/1     1            1           43s
deployment.apps/app-frontend-deploy   1/1     1            1           43s

NAME                                             DESIRED   CURRENT   READY   AGE
replicaset.apps/app-backend-deploy-75db86bc69    1         1         1       43s
replicaset.apps/app-frontend-deploy-7ccd9849df   1         1         1       43s
```

check
```bash
kubectl logs -n demo-app-ns pods/app-backend-deploy-75db86bc69-tptc9
```

```
  .   ____          _            __ _ _
 /\\ / ___'_ __ _ _(_)_ __  __ _ \ \ \ \
( ( )\___ | '_ | '_| | '_ \/ _` | \ \ \ \
 \\/  ___)| |_)| | | | | || (_| |  ) ) ) )
  '  |____| .__|_| |_|_| |_\__, | / / / /
 =========|_|==============|___/=/_/_/_/
 :: Spring Boot ::                (v3.0.4)
2024-11-04T21:44:55.973Z  INFO 1 --- [           main] n.j.s.SpringbootBackendApplication       : Starting SpringbootBackendApplication v0.0.1-SNAPSHOT using Java 17-ea with PID 1 (/app/runme.jar started by root in /)
2024-11-04T21:44:55.977Z  INFO 1 --- [           main] n.j.s.SpringbootBackendApplication       : No active profile set, falling back to 1 default profile: "default"
2024-11-04T21:44:56.605Z  INFO 1 --- [           main] .s.d.r.c.RepositoryConfigurationDelegate : Bootstrapping Spring Data JPA repositories in DEFAULT mode.
2024-11-04T21:44:56.655Z  INFO 1 --- [           main] .s.d.r.c.RepositoryConfigurationDelegate : Finished Spring Data repository scanning in 42 ms. Found 1 JPA repository interfaces.
2024-11-04T21:44:57.169Z  INFO 1 --- [           main] o.s.b.w.embedded.tomcat.TomcatWebServer  : Tomcat initialized with port(s): 8080 (http)
2024-11-04T21:44:57.179Z  INFO 1 --- [           main] o.apache.catalina.core.StandardService   : Starting service [Tomcat]
2024-11-04T21:44:57.180Z  INFO 1 --- [           main] o.apache.catalina.core.StandardEngine    : Starting Servlet engine: [Apache Tomcat/10.1.5]
2024-11-04T21:44:57.264Z  INFO 1 --- [           main] o.a.c.c.C.[Tomcat].[localhost].[/]       : Initializing Spring embedded WebApplicationContext
2024-11-04T21:44:57.266Z  INFO 1 --- [           main] w.s.c.ServletWebServerApplicationContext : Root WebApplicationContext: initialization completed in 1227 ms
2024-11-04T21:44:57.597Z  INFO 1 --- [           main] o.hibernate.jpa.internal.util.LogHelper  : HHH000204: Processing PersistenceUnitInfo [name: default]
2024-11-04T21:44:57.644Z  INFO 1 --- [           main] org.hibernate.Version                    : HHH000412: Hibernate ORM core version 6.1.7.Final
2024-11-04T21:44:57.928Z  INFO 1 --- [           main] com.zaxxer.hikari.HikariDataSource       : HikariPool-1 - Starting...
2024-11-04T21:44:58.111Z  INFO 1 --- [           main] com.zaxxer.hikari.pool.HikariPool        : HikariPool-1 - Added connection org.postgresql.jdbc.PgConnection@415e0bcb
2024-11-04T21:44:58.113Z  INFO 1 --- [           main] com.zaxxer.hikari.HikariDataSource       : HikariPool-1 - Start completed.
2024-11-04T21:44:58.162Z  INFO 1 --- [           main] SQL dialect                              : HHH000400: Using dialect: org.hibernate.dialect.PostgreSQLDialect
2024-11-04T21:44:58.993Z  INFO 1 --- [           main] o.h.e.t.j.p.i.JtaPlatformInitiator       : HHH000490: Using JtaPlatform implementation: [org.hibernate.engine.transaction.jta.platform.internal.NoJtaPlatform]
2024-11-04T21:44:59.001Z  INFO 1 --- [           main] j.LocalContainerEntityManagerFactoryBean : Initialized JPA EntityManagerFactory for persistence unit 'default'
2024-11-04T21:44:59.253Z  WARN 1 --- [           main] JpaBaseConfiguration$JpaWebConfiguration : spring.jpa.open-in-view is enabled by default. Therefore, database queries may be performed during view rendering. Explicitly configure spring.jpa.open-in-view to disable this warning
2024-11-04T21:44:59.573Z  INFO 1 --- [           main] o.s.b.w.embedded.tomcat.TomcatWebServer  : Tomcat started on port(s): 8080 (http) with context path ''
2024-11-04T21:44:59.582Z  INFO 1 --- [           main] n.j.s.SpringbootBackendApplication       : Started SpringbootBackendApplication in 4.1 seconds (process running for 4.597)
```




# TODO:

* add ingress nginx
* finish writing the readme
* automate cluster install completely
* write CI/CD ?
