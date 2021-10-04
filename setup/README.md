# prepare nodes

```
ansible-playbook -i hosts.ini setup.yaml
```


# install rancher.

get kubeconfig from rancher-node

```
scp root@rancher.rwcloud.org:/etc/rancher/rke2/rke2.yaml .
```

replace url in config-file 

temporarily enable port 6443 on rancher node

```
ufw allow 6443
```

## install cert-manager

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

## install rancher 

```
helm --kubeconfig rke2.yaml install rancher rancher-latest/rancher \
  --namespace cattle-system \
  --set replicas=1 \
  --set hostname=rancher.rwcloud.org \
  --set bootstrapPassword=admin \
  --set ingress.tls.source=secret \
  --version 2.6.0
```

login to rancher gui, create new user, login with new user and download the kubeconfig for local cluster
edit your local kubeconfig
should have urls to the rancher cluster now.

close port in ufw firewall

```
ufw status numbered
ufw delete <number>
```

# import workload cluster

in rancher, import cluster

login to one node and apply the commands, shown in rancher gui.

```
 /var/lib/rancher/rke2/bin/kubectl --kubeconfig /etc/rancher/rke2/rke2.yaml apply -f https://rancher.rwcloud.org/v3/import/kljljlekwjlkejwlkdjewlkjdewkl.yaml
```


