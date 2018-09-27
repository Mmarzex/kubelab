# kubelab

Easy instructions for a self-hosted Kubernetes installation on Digital Ocean.

Built on top of
[kubespray](https://github.com/kubernetes-incubator/kubespray/) (sic).
Check out the [kubespray
docs](https://github.com/kubernetes-incubator/kubespray#documents) if
you want to customize this further.

Note: Digital Ocean is preparing to release their own Kubernetes
platform soon, you may want to check that out. This uses many of the
same peices, as Digital Ocean open sources most of it. Having a DIY
platform that you fully control, may still be desirable, so here you
go.

## Requirements

 * A Digital Ocean account
 * An SSH client
 * A domain name, hosted on Digital Ocean DNS

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

 - Login to your Digital Ocean account.
 - Create a droplet using **Fedora Atomic** (on the Container distributions tab).
 - The small $5 size is ideal.
 - Use private networking.
 - Fill in the User Data field by copy/pasting from
   [kubelab/k8s-atomic-cloud-init.yml](https://raw.githubusercontent.com/EnigmaCurry/kubelab/kubelab/kubelab/k8s-atomic-cloud-init.yml).
 - Wait about two minutes, allow for the droplet to reboot one time, then ssh into the droplet as root.
 - Watch the log for the rest of the installation process:

```
journalctl -f --unit post-install
```

 - Wait for the installation to complete, you should see a message at
  the end: "Post Installation tasks Complete."
 - Generate an ssh key to manage the cluster

```
kubelab-ssh-keygen.sh
```
 - Copy the ssh public key this outptus, and add it to your Digital Ocean account (Security tab)

## Launch cluster nodes

The cluster nodes are where kubernetes runs, and are seperate from the kubelab controller.

 - Login to your Digital Ocean account.
 - Create 3 or however many droplets using **Ubuntu 18.04**.
 - 2GB of ram is recommended mimimum.
 - Use private networking.
 - Make sure to choose the same region as the kubelab controller.
 - Fill in the User Data field by copy/pasting from
   [kubelab/ubuntu-cloud-init.yml](https://raw.githubusercontent.com/EnigmaCurry/kubelab/kubelab/kubelab/ubuntu-cloud-init.yml).
 - Use the ssh key you generated for the kubelab controller (above), and at least one other backup key.

## Deploy kubernetes

 - From the kubelab controller, run the setup command, using the *private* IP
   addresses for the droplets just created:

```
# Replace with your cluster nodes private ip addresses:
DROPLET_IPS="10.93.109.42 10.93.109.70 10.93.111.109" kubelab-setup.sh
```

 - Setup installs kubelab code on the kubelab controller to
   `/var/lib/kubelab` and sets up the Ansible inventory files
   according to the `DROPLET_IPS`.
 - Once setup is complete, deploy the cluster:

```
kubelab-deploy.sh
```

Grab a bite to eat, come back in 15 minutes, and ansible should be done
creating the cluster, showing a PLAY RECAP indicating no failures.

## Access kubernetes

The kubernetes config has been copied to the controller node in
`/root/.kube/config`. You can run kubectl directly from the controller:

```
kubectl get nodes
```

`/root/.ssh/config` has been setup on the controller node with aliases
for all of the cluster nodes (node1, node2, node3, etc.) You can use
ssh to login to any of the nodes:

```
ssh node1
```

# Configure Traefik Ingress Controller

See [Helm chart](https://github.com/EnigmaCurry/charts/tree/master/stable/traefik)
and [Traefik docs](https://docs.traefik.io/configuration/backends/kubernetes/)

Traefik needs an API token to manage DNS for your domain name on
Digital Ocean. It uses this for ACME domain verification and issuing a
wildcard SSL/TLS certificate.

Store this as a secret in the format that traefik helm chart expects:

 * Login to your Digital Ocean account
 * Click on API tab and generate a new API token.
 * Name the token something like `kubelab-traefik`
 * Create the secret for traefik, replacing
   `PUT-YOUR-TOKEN-HERE` with your real API token.

```
SECRET=DO_AUTH_TOKEN=PUT-YOUR-TOKEN-HERE \
SECRET_NAME=traefik-dnsprovider-config \
NAMESPACE=kube-system \
ENCRYPTED_OUTPUT=traefik-dnsprovider-secret.yml

kubectl create secret generic $SECRET_NAME \
    -o json \
    --from-literal="$SECRET" \
    --dry-run \
    | kubeseal -n $NAMESPACE --format=yaml \
    > $ENCRYPTED_OUTPUT
```

Install the secret:

```
kubectl apply -f traefik-dnsprovider-secret.yml
```

 * Edit the traefik helm values in
   [kubelab/helm/traefik.yml](kubelab/helm/traefik.yml)
 * Modify the `example.com` domain names to match your own.
 * Modify the email address, this is the email contact Let's Encrypt
   will use to notify you (certifcate expirations, security notices etc.)

Normally the traefik helm chart creates its own
secret. `acme.dnsProvider.createSecret: false` skips this and instead
uses the existing secret you defined above.

Install traefik:

```
helm upgrade --install \
     traefik /var/lib/charts/stable/traefik \
     --namespace kube-system \
     --values /var/lib/kubelab/kubelab/helm/traefik.yml
```
