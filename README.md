# RPI // from Nothing to Something // Kubernetes

(4) Raspberry Pi 4: 4Gb

Ubuntu: 19.10 (GNU/Linux 5.3.0-1022-raspi2 aarch64)

Docker CE: Client & Server Version 19.03.8

Microk8s Kubernetes: Client & Server Version 1.18.0

### Warning: This document has high entropy, last valid test TBD
## This document is a work in progress, it does not build a working RPI cluster YET.

## Prepare Hardware
1. Load [Ubuntu Server] 19.10 onto sd card(s)
2. Insert SD card into RPI, power on, and directly connect via micro hdmi & keyboard
3. Change Password (at prompt)
4. Change [Hostname] by editing these files;
  * /etc/hostname
  * /etc/hosts [edit all occurances of hostname]
5. Reboot host
6. Configure [Ethernet] Interface
  * /etc/netplan/99_config.yaml
  * run `$ sudo netplan apply`
7. **WIP** Configure [SSH]
  * ~~WORK IN PROGRESS~~ ADDITIONAL INFO HERE https://www.cyberciti.biz/faq/ubuntu-18-04-setup-ssh-public-key-authentication/
8. Disconnect from RPI, attach to ethernet interface and SSH in from work terminal
8. Fix the Cgroup [bug]!
  * /boot/firmware/nobtcmd.txt
  * Append `cgroup_enable=memory cgroup_memory=1`
9. Modify ufw to allow cluster communication
```sh
$ sudo ufw allow in on cni0 && sudo ufw allow out on cni0
$ sudo ufw default allow routed
```

## Install [Docker] CE
1. Uninstall old versions
  * `$ sudo apt-get remove docker docker-engine docker.io containerd runc`
2. Update 'apt' package index
  * `$ sudo apt-get update`
3. Install packages to allow apt'to use a repository over HTTPS
```sh
$ sudo apt-get install \
    apt-transport-https \
    ca-certificates \
    curl \
    gnupg-agent \
    software-properties-common
```
4. Add Docker's Official GPG key
  * `curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -`
5. Verify key fingerprint [9DC8 5822 9FC7 DD38 854A E2D8 8D81 803C 0EBF CD88]
  * `$ sudo apt-key fingerprint 0EBFCD88`
6. Perform architecture specific install (arm64 below, others in source document, link in docker header)
```sh
$ sudo add-apt-repository \
   "deb [arch=arm64] https://download.docker.com/linux/ubuntu \
   $(lsb_release -cs) \
   stable"
```
  * Use `uname -m` to determine your architecture
7. Install Docker CE
  * `$ sudo apt-get update`
  * `$ sudo apt-get install docker-ce docker-ce-cli containerd.io`
8. Enable docker on startup
  * `$ sudo systemctl enable docker`

## Install [Microk8s]
1. Update 'apt' package index
  * `$ sudo apt-get update`
2. Install all updates available in 'apt' sources
  * `$ sudo apt-get upgrade`
3. Use a snap to install the latest version
  * `$ sudo snap install microk8s --classic`

#### Build the [Cluster]
###### Repeat above steps for each RPI first
1. From Master node run
  * `$ sudo microk8s.add-node`
2. Copy the `join` connection string and run on worker 1
3. Repeat step 1 to create the connection key for each additional worker
4. Verify attached nodes from master
  * `$ sudo microk8s.kubectl get node`
#### Post Install (Master Only)
1. Create a [new user] account, give it kubectl and docker rights  
  * `$ sudo adduser <username>` and complete all fields
  * `$ sudo usermod -aG microk8s <username>`
  * `$ sudo usermod -aG docker <username>`
  * `$ sudo chown -f -R <username> ~/.kube`
  * switch to user for all following steps
  * `$ su - <username>`
2. Create permanent alias (if Microk8s is only kubernetes implementation running in cluster)
  * From home folder create '.bash_alias'
```sh
#Alias for Microk8s.kubectl, will work on next login
alias kubectl='microk8s.kubectl'
```
  * Save and exit
  * Create temporary alias until next login
    * `$ alias kubectl='microk8s.kubectl'`
3. Enable Microk8s prepackaged Dashboard, DNS, and local storage services
  * `$ microk8s.enable dashboard dns storage`
4. Connect to Grafana Dashboard using default credentials
  * Run `$ kubectl config view` and copy password field
  * Run `$ kubectl cluster-info` and copy Grafana URL
  * Open web browser (I use Firefox) and point to Grafana URL, exchanging the 127.0.0.1 for the eth0 IP

#### Configure and connect to Kubernetes [Dashboard] with dedicated credentials
1. Connect to account authentication page URL
  * https://{Eth0-IP}:{cluster port}/api/v1/namespaces/kube-system/services/https:kubernetes-dashboard:/proxy/
    * NOTE: Use `$ kubectl cluster-info` to get port if needed
2. Create a Service Account (change name if desired)
  * `$ kubectl create serviceaccount dashboard-admin-sa`
3. Bind new account to cluster admin role
```sh
$ create clusterrolebinding dashboard-admin-sa 
--clusterrole=cluster-admin --serviceaccount=default:dashboard-admin-sa
```
4. Pull automatically created token for new account
  * `$ kubectl get secrets`
  * `$ kubectl describe secret <Name from previous command>`
  * Copy token
5. Paste token into entry field of dashboard URL and submit

#### **WIP:** Deploy your first Kubernetes workload from public Docker
1. Log into Docker
  * `$docker login`
  * Provide username/password
2.  Ensure cluster is healthy
  * `$ kubectl get nodes` (all nodes should be in 'ready' status)
  * 
  
## test, source oreilly 'start containers using kubectl'
kubectl run http --image=katacoda/docker-http-server:latest





[Ubuntu Server]: <https://ubuntu.com/download/raspberry-pi/thank-you?version=19.10.1&architecture=arm64+raspi3>

[Hostname]: <https://www.cyberciti.biz/faq/ubuntu-change-hostname-command/>

[Ethernet]: <https://help.ubuntu.com/lts/serverguide/network-configuration.html>

[SSH]: <https://help.ubuntu.com/lts/serverguide/openssh-server.html>

[bug]: <https://microk8s.io/docs/install-alternatives#arm>

[Docker]: <https://docs.docker.com/engine/install/ubuntu/>

[MicroK8s]: <https://ubuntu.com/tutorials/install-a-local-kubernetes-with-microk8s#2-deploying-microk8s>

[Cluster]: <https://discourse.ubuntu.com/t/how-to-build-a-raspberry-pi-kubernetes-cluster-using-microk8s/14792>

[new user]: <https://www.cyberciti.biz/faq/create-a-user-account-on-ubuntu-linux/>

[Dashboard]: <https://www.replex.io/blog/how-to-install-access-and-add-heapster-metrics-to-the-kubernetes-dashboard>
