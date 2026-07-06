Cloud native networking and network security.

## Installazione con Manifest
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


## Installazione con Operator
Con Operator, installi un controller che gestisce il ciclo di vita di Calico. 
La [documentazione](https://docs.tigera.io/calico/latest/getting-started/kubernetes/self-managed-onprem/onpremises) Calico raccomanda l’Operator per installazioni nuove/self-managed.

1. Installa i CRD dell'oprator:
```bash
kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.32.1/manifests/v1_crd_projectcalico_org.yaml
```

2. Installa Tigera Operator
```bash
kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.32.1/manifests/tigera-operator.yaml
```

3. Scarica il file di configurazione
```bash
curl -O https://raw.githubusercontent.com/projectcalico/calico/v3.32.1/manifests/custom-resources.yaml
```

4. Modifica il CIDR nel file
```bash
vim custom-resources.yaml

# deve combaciare col CIDR del kubeadm init

cidr: 10.244.0.0/16
```

Poi installa Calico
```bash
kubectl create -f custom-resources.yaml
```

5. Verifiche:
```bash
kubectl get pods -n tigera-operator
kubectl get pods -n calico-system
kubectl get tigerastatus
kubectl get nodes -o wide
```

Tutto è OK quando vedi:
```bash
ubuntu@master1:~$ kubectl get tigerastatus
NAME        AVAILABLE   PROGRESSING   DEGRADED   SINCE   MESSAGE
apiserver   True        False         False      45m     All objects available
calico      True        False         False      45m     All objects available
goldmane    True        False         False      40m     All objects available
ippools     True        False         False      40m     All objects available
tiers       True        False         False      40m     All objects available
whisker     True        False         False      40m     All objects available
```