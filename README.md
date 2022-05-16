# vodTestingDocs
vod testing and docs

## deploying vod from scratch with argocd

repo: https://github.com/zivled/vodTestingDocs

### prerequisite:

#### deployer:

install packages: curl, docker, git, ansible, kubectl, rke, helm.

#### cluster:

There is no minimum requirement for number of instances,
you can work with one node.
In this case we have 2 instance cluster (ubuntu 18.04) in wild-raanan:

1. VOD-MASTER. ip: 10.40.134.6 (master)

2. VOD-W1. ip:10.40.134.7 (worker)

it is recommended to enlarge var directory (to prevent future overload of logs) :
```bash
sudo fdisk /dev/sdb 

> m 
> n 
> p 
> Enter 
> Enter 
> Enter 
> w 

sudo mkdir /mnt/newvar 

sudo mount /dev/sdb1 /mnt/newvar 

sudo cp –apx /var/* /mnt/newvar 

mv /var /var.old 

sudo mkdir /var 

```

### instructions:

1. Disable rke’s nginx installation 
<!-- todo: explain how -->

2. run the pre_k8s role (I created for ubuntu):
    
    a. pull it from here: https://github.com/zivlederer/k8s_pre_role
    and put it on your local /etc/ansible/roles
    
    or by: ansible-galaxy install zivlederer.k8s_pre_role
    
    b. create inventory file.
    
    c. run it either by making a playbook or by this command:

    ```bash
    sudo ansible all -m include_role -a name=k8s_pre.yml --become -i inventory.yml
    ```

3. create k8s cluster with rke. example cluster.yaml in repo

4. Deploy argo-cd I did it without helm chart : https://github.com/argoproj/argo-cd/releases/tag/v2.2.2
<!--  (next time with helm ) -->

5. Port-forward to laptop. 

6. Create nfs server: 
    
    a. Install nfs server on the nfs host:
    ```bash
    sudo apt-get install nfs-kernel-server -y 
    ```

    b. Create “publicdata” and all other relevant directories that are needed with ownership nobody and 777. example:
        
    ```bash
    sudo mkdir -r /mnt/publicdata/rms-balmas1
    mkdir /tmp/nfs_mount_tmp
    mkdir /nfs_mount
    sudo chown -R nobody:nogroup /mnt/publicdata/
    sudo chmod 777 /mnt/publicdata
    ```

    c. Edit /etc/exports only on nfs server:
    ```bash
    10.40.134.6:/mnt/publicdata 10.40.134.0/24(rw,sync,no_subtree_check)
    ``` 

    d. apply & test with command :
    ```bash
    sudo exportfs -ra
    ```

    <!-- required?  -->
    e. restart nfs:
    ```bash
    sudo systemctl restart nfs-kernel-server
    ```

    f. Edit /etc/fstab on all worker nodes:
    ```bash
    10.40.134.6:/mnt/publicdata /tmp/nfs_mount_tmp nfs defaults 0 0
    10.40.134.6:/mnt/publicdata/rms-balmas1 /nfs_mount nfs defaults 0 0
    ```
    g. apply & test with command :
    ```bash
    sudo mount -a
    ```

7. Install nfs common
<!--  on master or worker? -->

8. Create nfs dynamic provisioner with helm chart: 

    https://github.com/kubernetes-sigs/nfs-subdir-external-provisioner/releases/tag/nfs-subdir-external-provisioner-4.0.14 

    ```bash
    helm install nfs-subdir-external-provisioner nfs-subdir-external-provisioner/nfs-subdir-external-provisioner     --set nfs.server=10.40.134.6     --set nfs.path=/mnt/publicdata
    ```
    
    *הערה: לשקול לעשות כבר קובץ values וגם שה -sc יהיה default storage class.

    storage class is created automatically based on nfs dynamic provisioner by its chart.

9. Install nginx ingress controller v. 3.11.1 with helm chart: https://github.com/kubernetes/ingress-nginx/releases/tag/ingress-nginx-3.11.1 

    using a values file:
    ```bash
    helm install ingress-nginx ingress-nginx -f projects/vod/nginxValues.yaml
    ```

10. Create all 4 labels for all worker nodes: backend=true, frontend=true, api=true, worker=true on all worker nodes:
    ```bash
    kubectl label <FE NODES> frontend=true
    kubectl label <BE NODES> backend=true
    kubectl label <WORKER NODES> worker=true
    ```

11. Create secret if not already exist – artifactory-dd-vod . 

    There may be a bug in the helm chart that overrides the secret with a blank secret with the same name so you may need to create another one. example.:

    ```bash
    kubectl create secret docker-registry artifactory-dd-vod --docker-server=artifactory.rnd-hub.com:5000 --docker-password=xxxxx --docker-username=zivled@rafael.co.il
    ```

12. Deploy manifest on argo-cd through ui. 

13. next step is to add dns to the ingress

testing:

test if ok by entering the rms-control and from there accesing and testing each component

example: http://10.40.134.7/rms/rms-control/index.html 


### now that vod is deployed you want to check if its performing ok.

### first lets know the..

## functonality of the system

Cluster versions page: show all componenets of the system with links to get to their ui and their versions.

<!-- i dont have it in cluster versions -->
<!-- auth: testing to see if api is ok. -->

rms ui: here you can watch your channels. you can choose a channel in the hamburger menu.

sds: the catalog of channels. here you can create new or edit existing channels.
enter: sensor id, ip, port, teken1.

note- the same functionality also exists in the simulator inside the channels simulator tab and i use it instead.

simulator: there are 2 tables.

the first one is for recordings. you can initiate a new recording or start streaming a new recording (and after that play the stream inside a channel)

the second one is for channels catalog (just like the sds). 

swagger: explains how to work with the relavant api.

## how to test vod?

- make sure all pages are up.
- in the simulator page click on "simulate from recording" (video roll icon) then choose the battlefield recording in the newly open section. leave ip and port as is.

