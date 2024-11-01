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
    "IPAddress": "10.89.0.4",
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
    -e "SPRING_DATASOURCE_URL=jdbc:postgresql://10.89.0.4:5432/employee_management_system" \
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

## Helm

Packaging these apps requires first pushing images to a registry. So let's push the backend to docker hub.

start by generating a new token or use an old valide one.

```bash
podman login -u kipsnak docker.io
Password:
Login Succeeded!
```

retag local built image
```bash
podman tag localhost/demo-backend:latest docker.io/kipsnak/demo-backend:latest
```

push image to docker.io
```bash
podman push docker.io/kipsnak/demo-backend:latest
```

make sur we are connected to the cluster
```bash
 kubectl get nodes
NAME     STATUS   ROLES           AGE     VERSION
node-1   Ready    control-plane   6h3m    v1.29.10
node-2   Ready    <none>          5h58m   v1.29.10
node-3   Ready    <none>          5h58m   v1.29.10
```

install backend package
```bash
helm install --name-template demo-app .
```

check
```bash
kubectl -n demo-app-ns get all
NAME                                      READY   STATUS   RESTARTS     AGE
pod/app-backend-deploy-79f76b6db5-tvwnr   0/1     Error    1 (8s ago)   13s

NAME                      TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)    AGE
service/app-backend-svc   ClusterIP   10.100.205.250   <none>        8080/TCP   13s

NAME                                 READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/app-backend-deploy   0/1     1            0           13s

NAME                                            DESIRED   CURRENT   READY   AGE
replicaset.apps/app-backend-deploy-79f76b6db5   1         1         0       13s
```

> [!warning]  
> App shutdown with error; I need to fix network access to database from kubernetes cluster


# TODO:

* Fix frontend and make it compile
* push frontend image to registry
* fix network access to database (from k8s cluster -> local podman network)
* activate frontend Helm values
* finish writing the readme
* automate cluster install completely
* write CI/CD ?