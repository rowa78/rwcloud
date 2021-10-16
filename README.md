# k8s-home

## prepare rasperry-pis

install ubuntu 20.04 lts on your pi's
login and change the default password 'ubuntu' to something you like
copy your ssh-key to the machines:

```
ssh-copy-id root@vps-1
ssh-copy-id root@vps-2
ssh-copy-id root@vps-3
ssh-copy-id root@vps-4
ssh-copy-id root@vps-5
```

## setup cluster

## Setting up the cluster

edit the host-config in setup/hosts.ini

```
cd setup
ansible-playbook -i hosts.ini setup.yaml
```

get kubeconfig from the cluster. It is located in file /etc/rancher/rke2/rke2.yaml. Place it in your homefolder: $HOME/.kube/config and edit the url
```
server: https://domain.com:6443
```


check the new cluster

```
kubectl cluster-info
Kubernetes control plane is running at .....
```

## Setting up direnv

Install direnv and setup some environment-variables in ./.envrc

add to .bashrc if using bash:
```
eval "$(direnv hook bash)"
```
add to .zshrc if using zsh:
```
eval "$(direnv hook zsh)"
```

Create a .envrc in project folder (and never add this file to your repo!)

```
export GITHUB_USER=
# token, that can create repositories (check all permissions under repo)
export GITHUB_TOKEN=

export BOOTSTRAP_GITHUB_REPOSITORY=https://github.com/rowa78/k8s-home

# 1Password-Token
export OP_TOKEN=

# Sops Keys
export KEY_NAME="rwcloud.org"
export KEY_COMMENT="flux secrets for rwcloud-cluster"
```

allow this file

```
direnv allow .envrc
```

Now your environment-variables are set.

create gpg keys for sops

https://fluxcd.io/docs/guides/mozilla-sops/


create a namespace for flux and add the key to it:

```
k create namespace flux-system

gpg --list-secret-keys "${KEY_NAME}"

# store Fingerprint
export KEY_FP=

# add the secret
gpg --export-secret-keys --armor "${KEY_FP}" |
kubectl create secret generic sops-gpg \
--namespace=flux-system \
--from-file=sops.asc=/dev/stdin
```




### create initial config

We use the 1Password-Operator to deliver secrets to out cluster. It need's an secret ' with the 1password-credentials.json in it. create an integration in 1password and save 1password-credentials.json and the token

``` 
kubectl create namespace 1password
kubectl -n 1password create secret generic onepassword-token --from-literal=token=$OP_TOKEN
```

Install the 1password operator

```
helm repo add 1password https://1password.github.io/connect-helm-charts
helm -n 1password upgrade -i connect 1password/connect --version 1.5.0 --set-file connect.credentials=/mnt/c/tmp/1password-credentials.json --values 1password-operator/values.yaml
#kubectl apply -f 1password-operator/clusterrolebinding.yaml
```

### Install cert-manager

```
# Add the Jetstack Helm repository
helm repo add jetstack https://charts.jetstack.io

# Update your local Helm chart repository cache
helm repo update

# Install the cert-manager Helm chart
helm --kubeconfig rke2.yaml install cert-manager jetstack/cert-manager \
  --namespace cert-manager \
  --create-namespace \
  --set installCRDs=true \
  --version v1.5.1
```

setup secrets and certs

k --kubeconfig rke2.yaml -n cert-manager apply -f ~/secret_rwcloud.yaml
k --kubeconfig rke2.yaml -n cert-manager apply -f ~/clusterissuer_rwcloud.yaml
k --kubeconfig rke2.yaml create namespace cattle-system
k --kubeconfig rke2.yaml apply -f ~/certificate_rwcloud.yaml

wait for valid certificate

k --kubeconfig rke2.yaml -n cattle-system get certificate


### Install wildcard-cert

### Install rancher




### Install ingress-nginx



### install flux to cluster

install flux

``` 
kubectl create namespace flux-system
flux bootstrap github --owner=rowa78 --repository=rwcloud --path=./clusters/rwcloud
```

### manual changed

i need to resolve dns-entries in my lan. So i changed the dns-server in configmap for CoreDNS:

```
kc -n kube-system edit configmap coredns
# change forward-line
# forward to my pi-hole
forward . 192.168.0.7
```

perhaps there is a better solution. Will look for that later.

