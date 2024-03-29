# ACES clusters Terraform

This config creates two clusters in the same on AWS on top EC2 instances using `kubeadm`.

The kubeconfig files for the individual clusters will be saved in the current working directory with a name corresponding to the cluster they belong to.

## Requirements

This need to be installed locally:
- terraform
- awscli => 2.x
- kubectl => 1.27
- helm >= 3.x
- openssh (your local ssh key `~/.ssh/id_rsa` will be use to access the clusters)

Your AWS account must be configure. If it is not the case already, use `aws configure`.

## Usage

Look at the main.tf file to check parameters.

Then apply with:
```sh
terraform init
terraform apply
```
The Kubernetes clusters are set and can be accessed with:
```sh
export KUBCONFIG=./aces-1.yaml
kubectl get nodes
```

## Post install

> NOTE: This might be move to terraform with something like the Helm provider...

Install The Flannel CNI on both clusters
```sh
helm repo add flannel https://flannel-io.github.io/flannel/

export KUBECONFIG=aces-1.conf
kubectl create ns kube-flannel
kubectl label --overwrite ns kube-flannel pod-security.kubernetes.io/enforce=privileged
helm install flannel --set podCidr="10.42.0.0/16" --namespace kube-flannel flannel/flannel

export KUBECONFIG=aces-2.conf
kubectl create ns kube-flannel
kubectl label --overwrite ns kube-flannel pod-security.kubernetes.io/enforce=privileged
helm install flannel --set podCidr="10.0.0.0/16" --namespace kube-flannel flannel/flannel
```

Now you can install submariner CLI `subctl` with:
```sh
curl -Ls https://get.submariner.io | VERSION=0.17.0 bash
export PATH=$PATH:~/.local/bin
```

Now will use aces-1 as a Submariner broker:
```
subctl deploy-broker --kubeconfig aces-1.conf --namespace submariner
```

And finally join the other cluster with:
```sh
subctl join --kubeconfig aces-1.conf broker-info.subm --clusterid aces-1
# Choose master as the gateway
subctl join --kubeconfig aces-2.conf broker-info.subm --clusterid aces-2
# Choose master as the gateway
```

Now that the installation is done you can verify the connectivity with:
```sh
export KUBECONFIG=aces-1.yaml:aces-2.yaml
subctl verify --context aces-1 --tocontext aces-2 --only service-discovery,connectivity --verbose
```

> FIXME: The connectivity between the cluster is not working yet. Might be due
> to some firewall rules. To be continued...

## Features

- [X] Two Kubernetes 1.29 clusters
- [X] Flanel CNI
- [-] Submariner
- [ ] AWS EBS CSI for volumes

## Create Access for users

To create one namespace per partner and give them access to both clusters run the `create-users.sh` script.
Partners listed in the `./partners` file.

All partners config with both cluster credentials in it, named `<partner name>-kubeconfig.yaml` should be available in the current directory.

To give more rights to one partner just change its Role assignment to a more permissive one in both clusters.

