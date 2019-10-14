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
4. Once SSHed in, execute `sudo raspi-config` and setup the user password and the network name of the device in the network options (f.e. k8s-master, k8s-node-1). Also change the memory split from 64 MB to 16 MB.
5. Reboot the pi and SSH back in.
6. Execute `sudo nano /etc/dhcpcd.conf` and add the following lines at the bottom to give it a static IP. The router ip usually is the same as the device ip but ending with a ".1"

```
interface eth0
static ip_address=<ip>/24
static routers=<router-ip>
static domain_name_servers=8.8.8.8
```

7. Reboot device and SSH back in. 

## Master Node Setup

1. Get k3s via `curl -sfL https://get.k3s.io | sh -` and after installation check if the systemd service started correctly with `sudo systemctl status k3s`
2. Get the token for the k3s server with `sudo cat /var/lib/rancher/k3s/server/node-token`, we will need that to join the worker to the cluster.

## Worker Node Setup

1. Execute `export K3S_URL="https://<k3s_server_host_ip>:6443"` and `export K3S_TOKEN="<k3s_token>"` to include the node in the existing cluster
2. Run `curl -sfL https://get.k3s.io | sh -` and wait for it to finish
3. Check with `sudo kubectl get nodes` on the server host machine to see if the node was added correctly.

## Deploy a container with a specific image

1. Copy your kubernetes service configuration yaml and your deployment configuration yaml onto the k3s server.
2. Execute `sudo kubectl apply -f <deployment_config>.yaml,<service_config>.yaml`
3. If needed scale the replicaset with `sudo kubectl scale deploy/<deplyoment_name> --replicas=4` 