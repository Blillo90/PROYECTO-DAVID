# PROYECTO DAVID (ASIR) – Guía desde cero (2 opciones)

> **Opción A:** 1 nodo (simple, estable, sin balanceo real)
>
> **Opción B:** 3 nodos (balanceo real con Traefik + WordPress escalable con RWX vía NFS)

Esta guía está pensada para **VirtualBox + Ubuntu Server + K3s** y está escrita para ejecutarse **paso a paso**.

---

## Índice

1. Requisitos y conceptos
2. Preparación en VirtualBox (red y VMs)
3. Instalación Ubuntu Server y ajustes base
4. Red (Netplan) y nombres
5. SSH y sudo para Ansible
6. Kubernetes (K3s) – instalación
7. Kubectl sin sudo (kubeconfig)
8. **Opción A (1 nodo): WordPress completo (sin balanceo real)**
9. **Opción B (3 nodos): WordPress completo y balanceado (con NFS RWX)**
10. Verificaciones y troubleshooting

---

## 1) Requisitos y conceptos

### 1.1 ¿Por qué NO LAMP en las VMs?

En Kubernetes **NO instalas Apache/MySQL/PHP en la VM**.

* WordPress corre en un **contenedor** (incluye Apache + PHP).
* MySQL corre en otro **contenedor**.
* Traefik (incluido en K3s) actúa como **reverse proxy/ingress**.

### 1.2 Balanceo real y persistencia

* Para balancear WordPress necesitas **más de 1 réplica**.
* Para que varias réplicas compartan `wp-content`, hace falta almacenamiento **RWX** (ReadWriteMany), por ejemplo **NFS**.
* Con `local-path` (K3s por defecto) normalmente solo tienes **RWO** (ReadWriteOnce), que no vale para múltiples réplicas de WordPress.

---

## 2) Preparación en VirtualBox (red y VMs)

### 2.1 Red Host-Only (LAN del clúster)

VirtualBox → **File > Tools > Network Manager**

* Crear **Host-only Network**
* IPv4: `192.168.56.1`
* DHCP: **OFF**

### 2.2 Internet obligatorio

Cada VM debe tener salida a Internet para instalar paquetes y K3s:

* Adaptador 1: **NAT** (Internet)
* Adaptador 2: **Host-Only** (LAN del clúster)

### 2.3 VMs

#### Opción A (1 nodo)

* `proyecto-master` (Ubuntu Server)

#### Opción B (3 nodos)

* `proyecto-master`  → `192.168.56.10`
* `proyecto-worker1` → `192.168.56.11`
* `proyecto-worker2` → `192.168.56.12`

> Consejo: crea una VM plantilla y clónala 1 o 3 veces. Marca **Reinitialise MAC address** en el clon.

---

## 3) Instalación Ubuntu Server y ajustes base (en TODAS las VMs)

### 3.1 Durante el instalador

* Crear usuario: `teo` (o el tuyo)
* ✅ Marcar **Install OpenSSH Server**
* Sin entorno gráfico

### 3.2 Actualizar sistema

```bash
sudo apt update && sudo apt -y upgrade
sudo reboot
```

---

## 4) Red (Netplan) y nombres

> En VirtualBox normalmente:
>
> * `enp0s3` = NAT (DHCP)
> * `enp0s8` = Host-Only (IP fija)

### 4.1 Ver interfaces

```bash
ip a
```

### 4.2 Netplan

Editar:

```bash
sudo nano /etc/netplan/00-installer-config.yaml
```

#### Opción A (1 nodo)

```yaml
network:
  version: 2
  ethernets:
    enp0s3:
      dhcp4: true
    enp0s8:
      dhcp4: false
      addresses:
        - 192.168.56.10/24
```

#### Opción B (3 nodos)

* Master: `192.168.56.10/24`
* Worker1: `192.168.56.11/24`
* Worker2: `192.168.56.12/24`

Aplicar:

```bash
sudo netplan apply
```

Comprobar:

```bash
ip a
ping -c 3 8.8.8.8
```

### 4.3 Hostname

En cada VM:

```bash
sudo hostnamectl set-hostname proyecto-master
# o
sudo hostnamectl set-hostname proyecto-worker1
sudo hostnamectl set-hostname proyecto-worker2
```

### 4.4 /etc/hosts (recomendado en todos)

```bash
sudo nano /etc/hosts
```

Añadir:

```
192.168.56.10 proyecto-master
192.168.56.11 proyecto-worker1
192.168.56.12 proyecto-worker2
```

---

## 5) SSH y sudo para Ansible (recomendado)

### 5.1 SSH keys (desde tu nodo de control)

```bash
ssh-keygen -t ed25519 -C "ansible"
ssh-copy-id teo@192.168.56.10
ssh-copy-id teo@192.168.56.11
ssh-copy-id teo@192.168.56.12
```

### 5.2 Sudo sin contraseña (en cada VM)

En cada VM:

```bash
ssh teo@192.168.56.10
sudo visudo
```

Añadir al final:

```
teo ALL=(ALL) NOPASSWD:ALL
```

Comprobar:

```bash
sudo ls /
```

---

## 6) Kubernetes (K3s)

> K3s se instala **en el master** como server y en los workers como agent.

### 6.1 Instalar K3s en el master

En `proyecto-master`:

```bash
curl -sfL https://get.k3s.io | sh -
```

Verificar:

```bash
sudo kubectl get nodes
sudo kubectl get pods -A
```

### 6.2 (Solo Opción B) Unir workers al clúster

En el master, obtener token:

```bash
sudo cat /var/lib/rancher/k3s/server/node-token
```

En cada worker (cambiando el token):

```bash
curl -sfL https://get.k3s.io | \
  K3S_URL=https://192.168.56.10:6443 \
  K3S_TOKEN=PEGA_AQUI_EL_TOKEN \
  sh -
```

Verificación en el master:

```bash
sudo kubectl get nodes -o wide
```

---

## 7) Kubectl sin sudo (evitar el warning de permisos)

En el master (para tu usuario):

```bash
mkdir -p ~/.kube
sudo cp /etc/rancher/k3s/k3s.yaml ~/.kube/config
sudo chown -R $USER:$USER ~/.kube
chmod 600 ~/.kube/config
# Cambiar 127.0.0.1 por la IP del master
sed -i 's/127.0.0.1/192.168.56.10/g' ~/.kube/config

kubectl get nodes -o wide
```

---

# 8) OPCIÓN A – 1 nodo (WordPress completo, estable)

✅ Fácil y estable para ASIR.

## 8.1 ¿Qué obtienes?

* 1 master (sin workers)
* WordPress + MySQL con persistencia local (`local-path`)
* Ingress Traefik

⚠️ No hay balanceo real porque solo habrá 1 réplica.

## 8.2 Guardar manifiesto

Ruta recomendada:

```
manifests/wordpress-1nodo.yaml
```

Crear archivo:

```bash
cd ~/PROYECTO-DAVID
mkdir -p manifests
nano manifests/wordpress-1nodo.yaml
```

Pegar:

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: wordpress
---
apiVersion: v1
kind: Secret
metadata:
  name: mysql-secret
  namespace: wordpress
type: Opaque
stringData:
  MYSQL_ROOT_PASSWORD: "RootPass-ChangeMe!"
  MYSQL_DATABASE: "wordpress"
  MYSQL_USER: "wpuser"
  MYSQL_PASSWORD: "WpPass-ChangeMe!"
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mysql-pvc
  namespace: wordpress
spec:
  accessModes: ["ReadWriteOnce"]
  resources:
    requests:
      storage: 8Gi
  storageClassName: local-path
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: wp-pvc
  namespace: wordpress
spec:
  accessModes: ["ReadWriteOnce"]
  resources:
    requests:
      storage: 8Gi
  storageClassName: local-path
---
apiVersion: v1
kind: Service
metadata:
  name: mysql
  namespace: wordpress
spec:
  ports:
    - port: 3306
      targetPort: 3306
  selector:
    app: mysql
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mysql
  namespace: wordpress
spec:
  replicas: 1
  selector:
    matchLabels:
      app: mysql
  strategy:
    type: Recreate
  template:
    metadata:
      labels:
        app: mysql
    spec:
      containers:
        - name: mysql
          image: mysql:8.0
          ports:
            - containerPort: 3306
          env:
            - name: MYSQL_ROOT_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: mysql-secret
                  key: MYSQL_ROOT_PASSWORD
            - name: MYSQL_DATABASE
              valueFrom:
                secretKeyRef:
                  name: mysql-secret
                  key: MYSQL_DATABASE
            - name: MYSQL_USER
              valueFrom:
                secretKeyRef:
                  name: mysql-secret
                  key: MYSQL_USER
            - name: MYSQL_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: mysql-secret
                  key: MYSQL_PASSWORD
          volumeMounts:
            - name: mysql-data
              mountPath: /var/lib/mysql
      volumes:
        - name: mysql-data
          persistentVolumeClaim:
            claimName: mysql-pvc
---
apiVersion: v1
kind: Service
metadata:
  name: wordpress
  namespace: wordpress
spec:
  selector:
    app: wordpress
  ports:
    - port: 80
      targetPort: 80
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: wordpress
  namespace: wordpress
spec:
  replicas: 1
  selector:
    matchLabels:
      app: wordpress
  strategy:
    type: Recreate
  template:
    metadata:
      labels:
        app: wordpress
    spec:
      containers:
        - name: wordpress
          image: wordpress:php8.2-apache
          ports:
            - containerPort: 80
          env:
            - name: WORDPRESS_DB_HOST
              value: "mysql:3306"
            - name: WORDPRESS_DB_NAME
              valueFrom:
                secretKeyRef:
                  name: mysql-secret
                  key: MYSQL_DATABASE
            - name: WORDPRESS_DB_USER
              valueFrom:
                secretKeyRef:
                  name: mysql-secret
                  key: MYSQL_USER
            - name: WORDPRESS_DB_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: mysql-secret
                  key: MYSQL_PASSWORD
          volumeMounts:
            - name: wp-data
              mountPath: /var/www/html
      volumes:
        - name: wp-data
          persistentVolumeClaim:
            claimName: wp-pvc
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: wordpress-ingress
  namespace: wordpress
  annotations:
    kubernetes.io/ingress.class: traefik
spec:
  rules:
    - host: wordpress.local
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: wordpress
                port:
                  number: 80
```

## 8.3 Desplegar

```bash
kubectl apply -f manifests/wordpress-1nodo.yaml
kubectl get pods -n wordpress -o wide
kubectl get ingress -n wordpress
```

## 8.4 Acceso desde Windows

Editar `C:\Windows\System32\drivers\etc\hosts` (admin) y añadir:

```
192.168.56.10 wordpress.local
```

Abrir:

* `http://wordpress.local`

---

# 9) OPCIÓN B – 3 nodos (balanceo real)

✅ WordPress con **2+ réplicas**
✅ Traefik balancea entre pods
✅ Pods repartidos entre nodos (anti-affinity)
✅ Persistencia compartida **RWX** con **NFS**

> Nota: MySQL se mantiene con 1 réplica (estado) y persistencia local (RWO) por simplicidad.

## 9.1 Preparar NFS RWX (en el master)

En `proyecto-master`:

```bash
sudo apt update
sudo apt install -y nfs-kernel-server
sudo mkdir -p /srv/nfs/wp
sudo chown -R nobody:nogroup /srv/nfs/wp
sudo chmod 777 /srv/nfs/wp

echo "/srv/nfs/wp *(rw,sync,no_subtree_check,no_root_squash)" | sudo tee /etc/exports
sudo exportfs -ra
sudo systemctl enable --now nfs-server
sudo exportfs -v
```

## 9.2 Instalar cliente NFS en TODOS los nodos

En master y workers:

```bash
sudo apt install -y nfs-common
```

## 9.3 Manifiesto completo (WordPress balanceado)

Ruta:

```
manifests/wordpress-3nodos-balanceado.yaml
```

Crear:

```bash
cd ~/PROYECTO-DAVID
mkdir -p manifests
nano manifests/wordpress-3nodos-balanceado.yaml
```

Pegar:

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: wordpress
---
apiVersion: v1
kind: Secret
metadata:
  name: mysql-secret
  namespace: wordpress
type: Opaque
stringData:
  MYSQL_ROOT_PASSWORD: "RootPass-ChangeMe!"
  MYSQL_DATABASE: "wordpress"
  MYSQL_USER: "wpuser"
  MYSQL_PASSWORD: "WpPass-ChangeMe!"
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mysql-pvc
  namespace: wordpress
spec:
  accessModes: ["ReadWriteOnce"]
  resources:
    requests:
      storage: 8Gi
  storageClassName: local-path
---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: wp-pv-nfs
spec:
  capacity:
    storage: 10Gi
  accessModes:
    - ReadWriteMany
  persistentVolumeReclaimPolicy: Retain
  nfs:
    server: 192.168.56.10
    path: /srv/nfs/wp
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: wp-pvc-rwx
  namespace: wordpress
spec:
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 10Gi
  volumeName: wp-pv-nfs
---
apiVersion: v1
kind: Service
metadata:
  name: mysql
  namespace: wordpress
spec:
  ports:
    - port: 3306
      targetPort: 3306
  selector:
    app: mysql
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mysql
  namespace: wordpress
spec:
  replicas: 1
  selector:
    matchLabels:
      app: mysql
  strategy:
    type: Recreate
  template:
    metadata:
      labels:
        app: mysql
    spec:
      containers:
        - name: mysql
          image: mysql:8.0
          ports:
            - containerPort: 3306
          env:
            - name: MYSQL_ROOT_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: mysql-secret
                  key: MYSQL_ROOT_PASSWORD
            - name: MYSQL_DATABASE
              valueFrom:
                secretKeyRef:
                  name: mysql-secret
                  key: MYSQL_DATABASE
            - name: MYSQL_USER
              valueFrom:
                secretKeyRef:
                  name: mysql-secret
                  key: MYSQL_USER
            - name: MYSQL_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: mysql-secret
                  key: MYSQL_PASSWORD
          volumeMounts:
            - name: mysql-data
              mountPath: /var/lib/mysql
      volumes:
        - name: mysql-data
          persistentVolumeClaim:
            claimName: mysql-pvc
---
apiVersion: v1
kind: Service
metadata:
  name: wordpress
  namespace: wordpress
spec:
  selector:
    app: wordpress
  ports:
    - port: 80
      targetPort: 80
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: wordpress
  namespace: wordpress
spec:
  replicas: 2
  selector:
    matchLabels:
      app: wordpress
  template:
    metadata:
      labels:
        app: wordpress
    spec:
      affinity:
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
            - weight: 100
              podAffinityTerm:
                labelSelector:
                  matchLabels:
                    app: wordpress
                topologyKey: kubernetes.io/hostname
      containers:
        - name: wordpress
          image: wordpress:php8.2-apache
          ports:
            - containerPort: 80
          env:
            - name: WORDPRESS_DB_HOST
              value: "mysql:3306"
            - name: WORDPRESS_DB_NAME
              valueFrom:
                secretKeyRef:
                  name: mysql-secret
                  key: MYSQL_DATABASE
            - name: WORDPRESS_DB_USER
              valueFrom:
                secretKeyRef:
                  name: mysql-secret
                  key: MYSQL_USER
            - name: WORDPRESS_DB_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: mysql-secret
                  key: MYSQL_PASSWORD
          volumeMounts:
            - name: wp-data
              mountPath: /var/www/html
      volumes:
        - name: wp-data
          persistentVolumeClaim:
            claimName: wp-pvc-rwx
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: wordpress-ingress
  namespace: wordpress
  annotations:
    kubernetes.io/ingress.class: traefik
spec:
  rules:
    - host: wordpress.local
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: wordpress
                port:
                  number: 80
```

## 9.4 Desplegar

```bash
kubectl apply -f manifests/wordpress-3nodos-balanceado.yaml
kubectl get pods -n wordpress -o wide
kubectl get pvc -n wordpress
kubectl get ingress -n wordpress
```

### 9.5 Comprobar balanceo

Ver en qué nodos están los pods:

```bash
kubectl get pods -n wordpress -o wide
```

Deberías ver 2 pods en nodos distintos (si hay recursos).

---

## 10) Verificaciones y troubleshooting

### 10.1 Ver recursos

```bash
kubectl get nodes -o wide
kubectl get pods -A
kubectl get all -n wordpress
```

### 10.2 Ver logs WordPress/MySQL

```bash
kubectl logs -n wordpress deploy/wordpress --tail=100
kubectl logs -n wordpress deploy/mysql --tail=100
```

### 10.3 Si un pod se queda Pending

```bash
kubectl describe pod -n wordpress <NOMBRE_POD>
```

Suele ser:

* falta de almacenamiento
* PVC sin bind
* falta de recursos CPU/RAM

### 10.4 Reiniciar despliegue

```bash
kubectl rollout restart deploy/wordpress -n wordpress
kubectl rollout restart deploy/mysql -n wordpress
```

---

## Notas para la memoria (ASIR)

* **No se instala LAMP** en las VMs: se usa arquitectura **contenedorizada**.
* Traefik actúa como **reverse proxy/ingress**.
* Balanceo real requiere:

  * **replicas > 1**
  * almacenamiento compartido **RWX** (NFS)
* Se recomienda automatizar con **Ansible** (red, SSH, seguridad, K3s, despliegue manifiestos).
