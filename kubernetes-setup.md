# Kubeflow 1.6 on DGX enabled instances
This document contains steps to deploy charmed kubeflow on DGX enabled GPU instances with kuberntes 1.22 (currently highest version charmed kubeflow supports).

## 1. Ubuntu setup
This document was tested on Ubuntu 20.04 vanilla

**Hint:** please make sure you don’t have any nvidia drivers preinstalled. You can do that with following steps:

**Check for apt packages (if empty you OK)** :
```{bash}
$ sudo apt list --installed | grep nvidia
```

If there are some packages presented you can try to remove them with

```{bash}
$ sudo apt remove <package-name>
$ sudo apt autoremove
```

**Check for kernel modules (if empty you OK):**
```{bash}
$ lsmod | grep nvidia
```

If there are some presented you can try to remove them with

```{bash}
sudo modprobe -r <module-name>
```

**Reboot**
```{bash}
$ sudo reboot
```

## 2. Grub setup
Edit `/etc/default/grub` and add the following options to

```
GRUB_CMDLINE_LINUX_DEFAULT: modprobe.blacklist=nouveau nouveau.modeset=
```

```{bash}
$ sudo reboot
```

## 3. Kubernetes installation (Microk8s)
Install microk8s and enable required addons
```{bash}
$ sudo snap install microk8s --classic --channel 1.22
 
$ sudo microk8s enable dns:10.229.32.21 storage ingress registry rbac helm3 metallb:10.64.140.43-10.64.140.49,192.168.0.105-192.168.0.111
 
$ sudo usermod -a -G microk8s ubuntu
$ sudo chown -f -R ubuntu ~/.kube
$ newgrp microk8s
```
Edit `/var/snap/microk8s/current/args/containerd-template.toml`. Add:
```
[plugins."io.containerd.grpc.v1.cri".registry.configs]

[plugins."io.containerd.grpc.v1.cri".registry.configs."registry-1.docker.io".auth]
username = "afrikha"
password = "<>"
```
```
$ microk8s.stop; microk8s.start
```

## 4. Enable GPU addon and configure MIG

Install GPU operator

```{bash}
$ sudo microk8s.enable gpu
$ mkdir .kube
$ microk8s config > ~/.kube/config
```

Check gpu count for k8s

```{bash}
$ kubectl get nodes --show-labels | grep gpu.count
```

Configure MIG devices

```{bash}
$ kubectl label nodes blanka nvidia.com/mig.config=all-1g.5gb --overwrite
```
Recheck gpu count (should be increased)

```{bash}
kubectl get nodes --show-labels | grep gpu.count
```

**Troubleshooting**: If none of the nodes appear in `get nodes` command please makes ure to uninstall all GPU drivers form kubernetes nodes and reinstall the microk8s.

## 5. Deploy Kubeflow

Bootstrap juju

```{bash}
$ sudo snap install juju --classic
```

```{bash}
$ mkdir .kube
$ microk8s config > ~/.kube/config

$ juju bootstrap microk8s micro
$ juju add-model kubeflow
```

Deploy kubeflow bundle

```{bash}
$ juju deploy kubeflow --channel 1.6/beta --trust
```

Work around the issue: https://github.com/canonical/bundle-kubeflow/issues/

```{bash}
$ sudo sysctl fs.inotify.max_user_instances=
$ sudo sysctl fs.inotify.max_user_watches=
```

Setup authentication
```{bash}
$ juju config dex-auth public-url=http://10.64.140.43.nip.io
$ juju config oidc-gatekeeper public-url=http://10.64.140.43.nip.io

$ juju config dex-auth static-username=admin
$ juju config dex-auth static-password=admin
```

Wait before all components are available. Run this command 

```{bash}
nice -n 16 watch -n 1 -c juju status --relations --color
```

Expected state: 
```
App                        Version                    Status   Scale  Charm                    Channel         Rev  Address         Exposed  Message
admission-webhook          res:oci-image@84a4d7d      active       1  admission-webhook        1.6/stable       50  10.152.183.98   no       
argo-controller            res:oci-image@669ebd5      active       1  argo-controller          3.3/stable       99                  no       
argo-server                res:oci-image@576d038      active       1  argo-server              3.3/stable       45                  no       
dex-auth                                              active       1  dex-auth                 2.31/stable     129  10.152.183.147  no       
istio-ingressgateway                                  active       1  istio-gateway            1.11/stable     114  10.152.183.29   no       
istio-pilot                                           active       1  istio-pilot              1.11/stable     131  10.152.183.104  no       
jupyter-controller         res:oci-image@8f4ec33      active       1  jupyter-controller       1.6/stable      138                  no       
jupyter-ui                 res:oci-image@cde6632      active       1  jupyter-ui               1.6/stable       99  10.152.183.222  no       
katib-controller           res:oci-image@03d47fb      active       1  katib-controller         0.14/stable      92  10.152.183.220  no       
katib-db                   mariadb/server:10.3        active       1  charmed-osm-mariadb-k8s  latest/stable    35  10.152.183.23   no       ready
katib-db-manager           res:oci-image@16b33a5      active       1  katib-db-manager         0.14/stable      66  10.152.183.25   no       
katib-ui                   res:oci-image@c7dc04a      active       1  katib-ui                 0.14/stable      90  10.152.183.197  no       
kfp-api                    res:oci-image@1b44753      active       1  kfp-api                  2.0/stable       81  10.152.183.8    no       
kfp-db                     mariadb/server:10.3        active       1  charmed-osm-mariadb-k8s  latest/stable    35  10.152.183.148  no       ready
kfp-persistence            res:oci-image@31f08ad      active       1  kfp-persistence          2.0/stable       76                  no       
kfp-profile-controller     res:oci-image@d86ecff      active       1  kfp-profile-controller   2.0/stable       61  10.152.183.173  no       
kfp-schedwf                res:oci-image@51ffc60      active       1  kfp-schedwf              2.0/stable       80                  no       
kfp-ui                     res:oci-image@55148fd      active       1  kfp-ui                   2.0/stable       80  10.152.183.146  no       
kfp-viewer                 res:oci-image@7190aa3      active       1  kfp-viewer               2.0/stable       79                  no       
kfp-viz                    res:oci-image@67e8b09      active       1  kfp-viz                  2.0/stable       74  10.152.183.16   no       
kubeflow-dashboard         res:oci-image@6fe6eec      active       1  kubeflow-dashboard       1.6/stable      154  10.152.183.61   no       
kubeflow-profiles          res:profile-image@0a46ffc  active       1  kubeflow-profiles        1.6/stable       82  10.152.183.204  no       
kubeflow-roles                                        active       1  kubeflow-roles           1.6/stable       31  10.152.183.141  no       
kubeflow-volumes           res:oci-image@cc5177a      active       1  kubeflow-volumes         1.6/stable       64  10.152.183.100  no       
metacontroller-operator                               active       1  metacontroller-operator  2.0/stable       48  10.152.183.169  no       
minio                      res:oci-image@1755999      active       1  minio                    ckf-1.6/stable   99  10.152.183.193  no              
oidc-gatekeeper            res:oci-image@32de216      active       1  oidc-gatekeeper          ckf-1.6/stable   76  10.152.183.194  no       
seldon-controller-manager  res:oci-image@eb811b6      active       1  seldon-core              1.14/stable      92  10.152.183.221  no       
tensorboard-controller     res:oci-image@667e455      active       1  tensorboard-controller   1.6/stable       56  10.152.183.232  no       
tensorboards-web-app       res:oci-image@914a8ab      active       1  tensorboards-web-app     1.6/stable       57  10.152.183.174  no       
training-operator                                     active       1  training-operator        1.5/stable       65  10.152.183.14   no 

Unit                          Workload  Agent  Address      Ports              Message
admission-webhook/0*          active    idle   10.1.19.32   4443/TCP           
argo-controller/0*            active    idle   10.1.19.82                      
argo-server/0*                active    idle   10.1.19.53   2746/TCP           
dex-auth/0*                   active    idle   10.1.19.56                      
istio-ingressgateway/0*       active    idle   10.1.19.28                      
istio-pilot/0*                active    idle   10.1.19.62                      
jupyter-controller/0*         active    idle   10.1.19.60                      
jupyter-ui/0*                 active    idle   10.1.19.55   5000/TCP           
katib-controller/0*           active    idle   10.1.19.46   443/TCP,8080/TCP   
katib-db-manager/0*           active    idle   10.1.19.2    6789/TCP           
katib-db/0*                   active    idle   10.1.19.57   3306/TCP           ready
katib-ui/0*                   active    idle   10.1.19.36   8080/TCP           
kfp-api/0*                    active    idle   10.1.19.85   8888/TCP,8887/TCP  
kfp-db/0*                     active    idle   10.1.19.23   3306/TCP           ready
kfp-persistence/0*            active    idle   10.1.19.80                      
kfp-profile-controller/0*     active    idle   10.1.19.83   80/TCP             
kfp-schedwf/0*                active    idle   10.1.19.22                      
kfp-ui/0*                     active    idle   10.1.19.87   3000/TCP           
kfp-viewer/0*                 active    idle   10.1.19.51                      
kfp-viz/0*                    active    idle   10.1.19.78   8888/TCP           
kubeflow-dashboard/0*         active    idle   10.1.19.86   8082/TCP           
kubeflow-profiles/0*          active    idle   10.1.19.49   8080/TCP,8081/TCP  
kubeflow-roles/0*             active    idle   10.1.19.31                      
kubeflow-volumes/0*           active    idle   10.1.19.63   5000/TCP           
metacontroller-operator/0*    active    idle   10.1.19.1                       
minio/0*                      active    idle   10.1.19.47   9000/TCP,9001/TCP  
oidc-gatekeeper/0*            active    idle   10.1.19.89   8080/TCP           
seldon-controller-manager/0*  active    idle   10.1.19.24   8080/TCP,4443/TCP  
tensorboard-controller/0*     active    idle   10.1.19.90   9443/TCP           
tensorboards-web-app/0*       active    idle   10.1.19.88   5000/TCP           
training-operator/0*          active    idle   10.1.19.38                      
```

Access the dashboard at http://10.64.140.43.nip.io. If you are running it on cloud and want to access it from local machine you can create socks proxy as follows.

Ssh to instance (the from which you ran commands above):
```
ssh -D9999 <user>@<IP>
```

On your computer's browser, go to `Settings > Network > Network Proxy`, and enable SOCKS proxy pointing to: `127.0.0.1:9999`

On a new browser window, access the link given in the previous step, appended by .nip.io, for example: http://10.64.140.43.nip.io