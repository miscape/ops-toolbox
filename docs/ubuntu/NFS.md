# Configure NFS Server

Questo laboratorio usa:

- un cluster Kubernetes creato con `kind`;
- una VM Multipass come server NFS;
- i nodi kind come client NFS.

```text
+----------------------+                         +----------------------+
| [    NFS Server    ] |                         | [   NFS Clients    ] |
|    VM Multipass      |                         |   nodi kind/Docker   |
|                      |                         |                      |
|  nfs-server          |                         |  lab-control-plane   |
|  10.80.32.209        |                         |  172.18.0.4          |
|                      |                         |                      |
|  export NFS:         |                         |  lab-control-plane2  |
|  /srv/nfs/kind       |                         |  172.18.0.2          |
|                      |                         |                      |
|                      |                         |  lab-control-plane3  |
|                      |                         |  172.18.0.3          |
+----------+-----------+                         +----------+-----------+
           |                                                |
           |              traffico NFS                      |
           +------------------------------------------------+
```

Obiettivo: permettere ai pod del cluster kind di usare una directory esportata dalla VM Multipass tramite NFS.

## Situazione iniziale

VM Multipass:

```bash
multipass list
```

Esempio:

```text
Name        State    IPv4           Image
nfs-server  Running  10.80.32.209   Ubuntu 26.04 LTS
```

Cluster kind:

```bash
kubectl get nodes -o wide
```

Esempio:

```text
NAME                  STATUS   ROLES           INTERNAL-IP
lab-control-plane    Ready    control-plane   172.18.0.4
lab-control-plane2   Ready    control-plane   172.18.0.2
lab-control-plane3   Ready    control-plane   172.18.0.3
```

## Idea di rete

I nodi kind sono container Docker. Nel caso dell'esempio hanno IP `172.18.0.x`.

La VM Multipass ha IP `10.80.32.209`.

Il collegamento funziona se i container Docker di kind riescono a raggiungere la VM Multipass e se dentro i nodi kind sono presenti i pacchetti client NFS.

## 1. Verificare la raggiungibilita' della VM

Dal computer host:

```bash
docker exec -it lab-control-plane ping -c 3 10.80.32.209
```

Se compare questo errore:

```text
exec: "ping": executable file not found in $PATH
```

non significa che la rete non funziona. Significa solo che nel container del nodo kind manca il comando `ping`.

Installare `ping` nel nodo:

```bash
docker exec -it lab-control-plane bash -lc "apt update && apt install -y iputils-ping"
```

Poi riprovare:

```bash
docker exec -it lab-control-plane ping -c 3 10.80.32.209
```

Ripetere, se serve, anche sugli altri nodi:

```bash
docker exec -it lab-control-plane2 bash -lc "apt update && apt install -y iputils-ping"
docker exec -it lab-control-plane3 bash -lc "apt update && apt install -y iputils-ping"
```

## 2. Preparare il server NFS nella VM Multipass

Entrare nella VM:

```bash
multipass shell nfs-server
```

Installare il server NFS:

```bash
sudo apt update
sudo apt install -y nfs-kernel-server
```

Creare la directory da esportare:

```bash
sudo mkdir -p /srv/nfs/kind
sudo chown nobody:nogroup /srv/nfs/kind
sudo chmod 777 /srv/nfs/kind
```

Aprire il file exports:

```bash
sudo nano /etc/exports
```

Aggiungere:

```text
/srv/nfs/kind 10.209.71.0/24(rw,sync,no_subtree_check,no_root_squash,insecure) 172.18.0.0/16(rw,sync,no_subtree_check,no_root_squash,insecure)
```

Applicare la configurazione:

```bash
sudo exportfs -ra
sudo exportfs -v
sudo systemctl restart nfs-server
sudo systemctl status nfs-server
```

Uscire dalla VM:

```bash
exit
```

## 3. Installare il client NFS nei nodi kind

Il mount NFS viene fatto dal nodo Kubernetes. Nel caso di kind, il nodo e' un container Docker.

Installare `nfs-common` su ogni nodo kind:

```bash
docker exec -it lab-control-plane bash -lc "apt update && apt install -y nfs-common"
docker exec -it lab-control-plane2 bash -lc "apt update && apt install -y nfs-common"
docker exec -it lab-control-plane3 bash -lc "apt update && apt install -y nfs-common"
```

Nota: questa modifica e' temporanea. Se il cluster kind viene distrutto e ricreato, bisogna rifarla.

## 4. Testare il mount NFS da un nodo kind

Entrare nel primo nodo:

```bash
docker exec -it lab-control-plane bash
```

Creare un mount point:

```bash
mkdir -p /mnt/test-nfs
```

Montare la directory NFS:

```bash
mount -t nfs 10.80.32.209:/srv/nfs/kind /mnt/test-nfs
```

Creare un file di prova:

```bash
echo "test da kind" > /mnt/test-nfs/prova.txt
ls -l /mnt/test-nfs
```

Uscire dal nodo:

```bash
exit
```

Verificare dalla VM Multipass:

```bash
multipass shell nfs-server
ls -l /srv/nfs/kind
cat /srv/nfs/kind/prova.txt
```

Se il file e' visibile dalla VM, il collegamento NFS funziona.