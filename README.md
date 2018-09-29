# kubelab

Easy instructions for a self-hosted Kubernetes installation on DigitalOcean.

Built on top of
[kubespray](https://github.com/kubernetes-incubator/kubespray/) (sic).
Check out the [kubespray
docs](https://github.com/kubernetes-incubator/kubespray#documents) if
you want to customize this further.

Note: DigitalOcean is preparing to release their own Kubernetes
platform soon, you may want to check that out. This uses many of the
same peices, as DigitalOcean open sources most of it. Having a DIY
platform that you fully control, may still be desirable, so here you
go.

## Requirements

 * A DigitalOcean account
 * An SSH client
 * A domain name, hosted on DigitalOcean DNS

This guide starts by creating a kubelab controller, which is a
temporary environment for creating a kubernetes cluster. Using the
controller node standardizes the installation steps, as well as
keeping all the configuration off of your personal workstation. The
kubelab controller may be shutdown once the cluster is fully deployed,
or you can leave it online and continue to use it as an admin console
for your cluster.

This guide outlines using Traefik as a Kubernetes Ingress
Controller. Traefik uses Let's Encrypt to create a SSL/TLS certificate
for your domain name, and will provide secure URLs for your services.

## Install kubelab controller environment

 - Login to your DigitalOcean account.
 - Create a droplet using **Fedora Atomic** (on the Container distributions tab).
 - The small $5 size is ideal.
 - Use private networking.
 - Fill in the User Data field by copy/pasting from [kubelab/k8s-atomic-cloud-init.yml](https://raw.githubusercontent.com/EnigmaCurry/kubelab/kubelab/kubelab/k8s-atomic-cloud-init.yml).
 - Wait about three minutes, the droplet will reboot itself, then SSH
   into the droplet as root.

### If you want to watch the install log of the controller:

 - Within about the first three minutes, you can follow the cloud init log:

```
tail -f /var/log/cloud-init-output.log
```

 - After the cloud-init finishes, the droplet will reboot, and then run the
   post-install script.
 - Watch the post install log:

```
journalctl -f --unit post-install
```

 - Upon completion, you should see a message at the end: "Post
  Installation tasks Complete."
 - Press Ctrl-C to exit the log tail.
 
## Launch cluster nodes

The cluster nodes are where kubernetes runs, and are seperate from the kubelab controller.

 - Generate an ssh key to manage the cluster. On the controller, run:

```
kubelab-ssh-keygen.sh
```

Copy the text starting with with `ssh-rsa ....` and use it when
creating cluster droplets.

 - Login to your DigitalOcean account.
 - Create 3 or however many droplets using **Ubuntu 16.04**.
 - 2GB of ram is recommended mimimum.
 - Use private networking.
 - Make sure to choose the same region as the kubelab controller.
 - Fill in the User Data field by copy/pasting from
   [kubelab/ubuntu-cloud-init.yml](https://raw.githubusercontent.com/EnigmaCurry/kubelab/kubelab/kubelab/ubuntu-cloud-init.yml).
 - Create and assign a new SSH key using the one generated above.

## Deploy kubernetes

 - Login to your DigitalOcean account.
 - Click on API tab and generate a new API token.
 - Name the token something like `kubelab`.
 - Copy the given token.
 - From the kubelab controller, run the setup command setting your own
   specific values:
   - DOMAIN - The (sub-)domain you want to use with traefik.
   - EMAIL - Your email address to register with Let's Encrypt.
   - DIGITALOCEAN_API_TOKEN - the token generated above
   - DROPLET_IPS - The **private** IP addresses of your droplets (seperate with spaces)
   
```
# These vars are only used during initial setup:
export DOMAIN=k8s.example.com
export EMAIL=letsencrypt@example.com
export DIGITALOCEAN_API_TOKEN=xxxx
export DROPLET_IPS="10.93.109.42 10.93.109.70 10.93.111.109"
kubelab-setup.sh
```

 - Setup does the following on the kubelab controller:
   - Installs kubelab code to `/var/lib/kubelab` (or `KUBELAB_HOME` if set)
   - Builds the kubelab docker image.
   - Creates ansbile inventory according to DROPLET_IPS and other vars.
   
 - Once setup is complete, deploy the cluster by running:

```
kubelab-deploy.sh
```
 - Deploy does the following:
   - Runs [cluster.yml](cluster.yml) - The main kubespray playbook.
   - Runs [kubelab.yml](kubelab.yml) - Additional kubelab playbook.

Grab a bite to eat, come back in 15 minutes, and ansible should be done
creating the cluster, showing a `PLAY RECAP` indicating no failures.

## Access kubernetes

After deployment, the kubernetes config has been copied to the
controller node in `/root/.kube/config`. You can run kubectl directly
from the controller:

```
kubectl get nodes
```

`/root/.ssh/config` has been setup on the controller node with aliases
for all of the cluster nodes (node1, node2, node3, etc.) You can use
ssh to login to any of the nodes:

```
ssh node1
```

## Running Playbooks

Playbooks can be run with the helper script:

```
kubelab-playbook.sh NAME_OF_PLAYBOOK
```

The playbook argument is relative to KUBELAB_HOME (`/var/lib/kubelab`)

For example:

```
kubelab-playbook.sh kubelab.yml
```

This reapplies the [/var/lib/kubelab/kubelab.yml](kubelab.yml) playbook.
