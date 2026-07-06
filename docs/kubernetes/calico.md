Cloud native networking and network security.

Install Calico with Kubernetes API datastore, 50 nodes or less:

1. Download the Calico networking manifest for the Kubernetes API datastore.
```bash
curl https://raw.githubusercontent.com/projectcalico/calico/v3.32.0/manifests/calico.yaml -O
```
2. If you are using pod CIDR 192.168.0.0/16, skip to the next step. If you are using a different pod CIDR with kubeadm, no changes are required — Calico will automatically detect the CIDR based on the running configuration. For other platforms, make sure you uncomment the CALICO_IPV4POOL_CIDR variable in the manifest and set it to the same value as your chosen pod CIDR.

3. Customize the manifest as necessary.

4. Apply the manifest using the following command.
```bash
kubectl apply -f calico.yaml
```

For more info: [Official Calico Webpage](https://docs.tigera.io/calico/latest/getting-started/kubernetes/self-managed-onprem/onpremises#install-calico-with-kubernetes-api-datastore-50-nodes-or-less)