# Setup a Kubernetes cluster with raspis

## Requirements

- Raspi 3B or better 
- Ethernet Switch + Cables
- Etcher
- Raspbian Lite

## Basic setup required by master and worker

1. [Download Raspbian](https://www.raspberrypi.org/downloads/raspbian/) and flash it with [Etcher](https://www.balena.io/etcher/)
2. After flashing put an empty file with the name "ssh" into the boot partition on the sd card. This will activate ssh which we will use to login onto the raspis, saving us the hassle to connect peripherals.
3. Start the Raspi with the flashed system and get the IP of it either via an IPscanner or directly from the router. SSH into it via `ssh pi@<ip>`. "pi" is the standard user, "raspberry" is the standard password.
4. Once SSHed in, execute `sudo raspi-config` and setup the user password and the network name of the device in the network options (f.e. k8s-master, k8s-node-1). You might also want to setup the Localisation Options, which is optional but required if you want to use Wifi.
5. Reboot the pi and SSH back in.
6. Execute `sudo nano /etc/dhcpcd.conf` and add the following lines at the bottom to give it a static IP. The router ip usually is the same as the device ip but ending with a ".1"

```
interface eth0
static ip_address=<ip>/24
static routers=<router-ip>
static domain_name_servers=8.8.8.8
```

7. Reboot device and SSH back in.
8. Install docker by executing `curl -sSL get.docker.com | sh && sudo usermod pi -aG docker && newgrp docker` to use the install script from docker.com, add the existing user to the docker group and change the current session to the group.
9. Kubernetes requires swap to be disabled by executing `sudo dphys-swapfile swapoff && sudo dphys-swapfile uninstall && sudo update-rc.d dphys-swapfile remove`. Please note that this change needs to be redone after a restart of the Raspi. Check the "Useful information" part at the end of this document on a guide how to disable swap completely. You can check if the disable was a success when the `sudo swapon --summary` returns empty. 
10. Execute `sudo nano /boot/cmdline.txt` and add `cgroup_enable=cpuset cgroup_memory=1 cgroup_enable=memory` to the end of the line. Do not use a new line!
11. Reboot and SSH back in.
12. Execute `sudo touch /etc/apt/sources.list.d/kubernetes.list && sudo nano /etc/apt/sources.list.d/kubernetes.list`. Add `deb http://apt.kubernetes.io/ kubernetes-xenial main` to the file.
13. For this step you need SU prviliges. Execute `sudo su` and add the key via `curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | apt-key add -`. The output "OK" is expected. Use exit to logout of the root user.
14. Update repos and install kubeadm via `sudo apt-get update && sudo apt-get install -qy kubeadm`.

## Master Node Setup

1. Pre-pull images via `sudo kubeadm config images pull -v3`. This may take a while do not abort.
2. Weave Net is used as network overlay. Execute `sudo kubeadm init --token-ttl=0`. The `- -token-ttl = 0` makes sure our token doesn’t expire. This is not a good practice and should not be done in production. This may take a while.
3. We now set up kubeconfig. Start with `mkdir -p $HOME/.kube` to create the directory, followed by `sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config` to copy the config and `sudo chown $(id -u):$(id -g) $HOME/.kube/config` to make it usable.
4. Copy and save the join token. We will need it later to add worker nodes to the cluster. It looks something like this `kubeadm join --token 9e700f.7dc97f5e3a45c9e5 192.168.0.27:6443 --discovery-token-ca-cert-hash sha256:95cbb9ee5536aa61ec0239d6edd8598af68758308d0a0425848ae1af28859bea`
... Incase you loose this line you can find the token with `kubeadm token list`. The `sha hash` bit is trickier to get. Execute `openssl x509 -pubkey -in /etc/kubernetes/pki/ca.crt | openssl rsa -pubin -outform der 2>/dev/null | openssl dgst -sha256 -hex | sed 's/^.* //'` for that.
5. Install the Weave Net network driver with `kubectl apply -f "https://cloud.weave.works/k8s/net?k8s-version=$(kubectl version | base64 | tr -d '\n')"` This applies the yml file found by the link.
6. Execute `kubectl get nodes --all-namespaces` and check the status, which should display running. Some may take some time to get to that state.
7. Execute `sudo sysctl net.bridge.bridge-nf-call-iptables=1` as discussed [here](https://github.com/kubernetes/kubeadm/issues/312).

## Worker Node Setup

1. Execute `sudo kubeadm join --token <token> <master-node-ip>:6443 --discovery-token-ca-cert-hash sha256:<sha256>` to include the node in the existing cluster
2. Wait a bit and run `kubectl get nodes` to check if it was added correctly.

## Deploy a container with a specific image

## Useful information

- you cannot poweroff the cluster at the moment
- If you want to turn off swap completely execute `sudo nano /etc/fstab` and delete everything inside that file. Check if swap is turned off after a reboot with `free -m`