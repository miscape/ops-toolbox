# Dynamic Volume Provisioning (NFS)
Utilizzo la stessa architettura usata per [NFS Server](https://miscape.github.io/ops-toolbox/ubuntu/nfs-server/).

Il dynamic NFS provisioning serve a far creare automaticamente a Kubernetes i volumi NFS quando un’applicazione richiede una PVC.

In pratica lato Admin prepari una volta sola `NFS server + nfs-subdir-external-provisioner + StorageClass`.

I developers quando devono deployare aggiungono una PVC con la `storageClassName: nfs-retain` creata dall'Admin.

Kubernetes crea automaticamente `PVC -> PV -> sottodirectory sul server NFS`.

## 1. Installare Helm repo del provisioner

Il provisioner che useremo è `nfs-subdir-external-provisioner`.

Questo componente non crea un server NFS: usa un server NFS già esistente e crea automaticamente una sottodirectory per ogni PVC.

Nel nostro laboratorio il server NFS è:

```text
10.80.32.209:/srv/nfs/kind
```

Aggiungere il repository Helm:

```bash
helm repo add nfs-subdir-external-provisioner https://kubernetes-sigs.github.io/nfs-subdir-external-provisioner/
helm repo update
```

Creare il namespace dedicato:

```bash
kubectl create namespace nfs-provisioner
```

Se il namespace esiste già, il comando può restituire errore. In quel caso si può proseguire.

## 2. Installare due provisioner NFS

Installeremo due release Helm separate:

| Release Helm | StorageClass | Reclaim policy | Uso consigliato |
|---|---|---|---|
| `nfs-delete` | `nfs-delete` | `Delete` | test, dati temporanei |
| `nfs-retain` | `nfs-retain` | `Retain` | dati da conservare |

Lo stesso export NFS viene usato da entrambi:

```text
10.80.32.209:/srv/nfs/kind
```

La differenza è nella policy della StorageClass.

### Installare provisioner con policy Delete

```bash
helm install nfs-delete \
  nfs-subdir-external-provisioner/nfs-subdir-external-provisioner \
  --namespace nfs-provisioner \
  --set nfs.server=10.80.32.209 \
  --set nfs.path=/srv/nfs/kind \
  --set storageClass.name=nfs-delete \
  --set storageClass.provisionerName=k8s-sigs.io/nfs-delete \
  --set storageClass.reclaimPolicy=Delete \
  --set storageClass.onDelete=delete \
  --set storageClass.defaultClass=false
```

### Installare provisioner con policy Retain

```bash
helm install nfs-retain \
  nfs-subdir-external-provisioner/nfs-subdir-external-provisioner \
  --namespace nfs-provisioner \
  --set nfs.server=10.80.32.209 \
  --set nfs.path=/srv/nfs/kind \
  --set storageClass.name=nfs-retain \
  --set storageClass.provisionerName=k8s-sigs.io/nfs-retain \
  --set storageClass.reclaimPolicy=Retain \
  --set storageClass.onDelete=retain \
  --set storageClass.defaultClass=false
```

## 3. Verificare installazione

Controllare le release Helm:

```bash
helm list -n nfs-provisioner
```

Output atteso:

```text
NAME        NAMESPACE        STATUS    CHART
nfs-delete  nfs-provisioner  deployed  nfs-subdir-external-provisioner-4.0.18
nfs-retain  nfs-provisioner  deployed  nfs-subdir-external-provisioner-4.0.18
```

Controllare i pod:

```bash
kubectl get pods -n nfs-provisioner -o wide
```

Output atteso:

```text
NAME                                      READY   STATUS
nfs-delete-nfs-subdir-external-...        1/1     Running
nfs-retain-nfs-subdir-external-...        1/1     Running
```

Controllare le StorageClass:

```bash
kubectl get storageclass
```

Output atteso:

```text
NAME                 PROVISIONER              RECLAIMPOLICY
nfs-delete           k8s-sigs.io/nfs-delete   Delete
nfs-retain           k8s-sigs.io/nfs-retain   Retain
standard (default)   rancher.io/local-path    Delete
```

In questo laboratorio `standard` resta la StorageClass di default. Le due StorageClass NFS si usano esplicitamente nei manifest tramite `storageClassName`.

## 4. Testare StorageClass nfs-delete

Creare il file `test-nfs-delete.yaml`:

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pvc-nfs-delete
spec:
  storageClassName: nfs-delete
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 100Mi
---
apiVersion: v1
kind: Pod
metadata:
  name: pod-nfs-delete
spec:
  containers:
    - name: test
      image: busybox:1.36
      command:
        - sh
        - -c
        - |
          echo "file creato usando StorageClass nfs-delete" > /data/delete-test.txt
          sleep 3600
      volumeMounts:
        - name: data
          mountPath: /data
  volumes:
    - name: data
      persistentVolumeClaim:
        claimName: pvc-nfs-delete
```

Applicare:

```bash
kubectl apply -f test-nfs-delete.yaml
```

Verificare:

```bash
kubectl get pvc,pv,pod
kubectl exec -it pod-nfs-delete -- cat /data/delete-test.txt
```

## 5. Testare StorageClass nfs-retain

Creare il file `test-nfs-retain.yaml`:

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pvc-nfs-retain
spec:
  storageClassName: nfs-retain
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 100Mi
---
apiVersion: v1
kind: Pod
metadata:
  name: pod-nfs-retain
spec:
  containers:
    - name: test
      image: busybox:1.36
      command:
        - sh
        - -c
        - |
          echo "file creato usando StorageClass nfs-retain" > /data/retain-test.txt
          sleep 3600
      volumeMounts:
        - name: data
          mountPath: /data
  volumes:
    - name: data
      persistentVolumeClaim:
        claimName: pvc-nfs-retain
```

Applicare:

```bash
kubectl apply -f test-nfs-retain.yaml
```

Verificare:

```bash
kubectl get pvc,pv,pod
kubectl exec -it pod-nfs-retain -- cat /data/retain-test.txt
```

## 6. Verificare le directory sul server NFS

Entrare nella VM Multipass:

```bash
multipass shell nfs-server
```

Controllare cosa è stato creato dal provisioner:

```bash
find /srv/nfs/kind -maxdepth 2 -print
```

Esempio di directory generata automaticamente:

```text
/srv/nfs/kind/default-pvc-nfs-retain-pvc-597ca00d-57ce-4c6a-a0e0-bb4da9f2773a
```

Il nome della directory segue questo schema:

```text
namespace-nomePVC-nomePV
```

## 7. Provare la differenza tra Delete e Retain

Prima cancellare i pod:

```bash
kubectl delete pod pod-nfs-delete pod-nfs-retain
```

Poi cancellare le PVC:

```bash
kubectl delete pvc pvc-nfs-delete pvc-nfs-retain
```

Controllare i PV:

```bash
kubectl get pv
```

Risultato atteso:

```text
nfs-delete -> il PV sparisce
nfs-retain -> il PV resta in stato Released
```

Controllare anche il server NFS:

```bash
multipass shell nfs-server
find /srv/nfs/kind -maxdepth 2 -print
```

Risultato atteso:

```text
nfs-delete -> la directory viene eliminata
nfs-retain -> la directory resta sul server NFS
```

## 8. Pulizia finale del laboratorio

Se sono rimasti oggetti di test:

```bash
kubectl delete -f test-nfs-delete.yaml --ignore-not-found
kubectl delete -f test-nfs-retain.yaml --ignore-not-found
```

Se un PV `Retain` resta in stato `Released`, e si vuole rimuoverlo dal cluster:

```bash
kubectl delete pv <nome-pv>
```

Attenzione: cancellare il PV `Retain` rimuove l'oggetto Kubernetes, ma non cancella automaticamente la directory sul server NFS.

Per cancellare manualmente i dati rimasti sul server NFS:

```bash
multipass shell nfs-server
sudo rm -rf /srv/nfs/kind/<directory-da-rimuovere>
```

## Troubleshooting veloce

### Pod del provisioner in Pending

Se i pod restano in `Pending`:

```bash
kubectl get pods -n nfs-provisioner -o wide
```

e nella colonna `NODE` compare `<none>`, controllare gli eventi:

```bash
kubectl get events -n nfs-provisioner --sort-by='.lastTimestamp'
```

Se il problema è una taint sui nodi control-plane, per un laboratorio kind si può rimuovere la taint:

```bash
kubectl taint nodes --all node-role.kubernetes.io/control-plane:NoSchedule-
```

In alternativa si possono installare i provisioner con una `toleration`, ma per questo laboratorio rimuovere la taint è la strada piu' semplice.

### Errore: ping non trovato

Problema:

```text
exec: "ping": executable file not found in $PATH
```

Soluzione:

```bash
docker exec -it lab-control-plane bash -lc "apt update && apt install -y iputils-ping"
```

### Errore: mount failed exit status 32

Di solito significa che il nodo Kubernetes non riesce a montare il volume NFS.

Controllare:

```bash
docker exec -it lab-control-plane bash -lc "dpkg -l | grep nfs-common"
docker exec -it lab-control-plane bash -lc "showmount -e 10.80.32.209"
```

Se `nfs-common` manca:

```bash
docker exec -it lab-control-plane bash -lc "apt update && apt install -y nfs-common"
```

Se `showmount` non mostra l'export, controllare il server NFS nella VM:

```bash
multipass shell nfs-server
sudo exportfs -v
sudo systemctl status nfs-server
```

### Verificare se la porta NFS risponde

NFS usa normalmente la porta TCP `2049`.

Dal nodo kind:

```bash
docker exec -it lab-control-plane bash -lc "timeout 3 bash -c '</dev/tcp/10.80.32.209/2049' && echo OK || echo KO"
```

Se risponde `OK`, la porta NFS è raggiungibile dal nodo.