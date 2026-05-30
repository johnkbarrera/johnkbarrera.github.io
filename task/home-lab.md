# Manual Completo — Cluster Dinámico con k3s + Wake-on-LAN + PCs Personales

# Objetivo

Construir desde cero un cluster dinámico donde:

- Solo el master está siempre encendido
- Los workers pueden apagarse
- Los nodos se prenden ON-DEMAND
- Kubernetes coordina la carga
- Ubuntu Ryzen y Windows Ryzen actúan como workers opcionales

---

# Arquitectura Final

```text
                 SIEMPRE ENCENDIDO

┌─────────────────────────────┐
│ N100 MASTER                 │
│ Ubuntu Server               │
│ k3s server                  │
│ Monitoring                  │
│ Wake-on-LAN manager         │
└──────────────┬──────────────┘
               │

        OPCIONAL 24/7

┌──────────────▼──────────────┐
│ N100 Worker #1              │
│ Ubuntu Server               │
│ k3s agent                   │
└─────────────────────────────┘


         NODOS ON-DEMAND

┌─────────────────────────────┐
│ N100 Worker #2              │
│ Ubuntu Server               │
│ k3s agent                   │
│ apagado normalmente         │
└─────────────────────────────┘

┌─────────────────────────────┐
│ Ubuntu Ryzen                │
│ Ubuntu Desktop              │
│ k3s agent                   │
└─────────────────────────────┘

┌─────────────────────────────┐
│ Windows Ryzen               │
│ WSL2 Ubuntu                 │
│ k3s agent                   │
└─────────────────────────────┘
```

---

# PARTE 1 — Hardware Recomendado

# Master

## Recomendado

- Intel N100
- 16GB RAM
- SSD NVMe
- Ethernet

---

# Workers

## Recomendado

- Intel N100/N305
- 16GB RAM
- SSD NVMe

---

# Ubuntu Workstation

## Recomendado

- Ryzen 7 8845HS
- 32GB RAM
- Ubuntu Desktop 24.04

---

# Windows Workstation

## Recomendado

- Ryzen 7 / Intel Core Ultra
- 32GB RAM
- Windows 11

---

# Red

## MUY IMPORTANTE

Usar:
- Ethernet

Evitar:
- WiFi para nodos permanentes

---

# Recomendación ideal

- Switch 2.5GbE

---

# PARTE 2 — Instalar Ubuntu Server en el Master

# 1. Descargar Ubuntu Server

https://ubuntu.com/download/server

---

# 2. Crear USB booteable

## Windows

Usar:
- Rufus

## Linux

```bash
sudo dd if=ubuntu.iso of=/dev/sdX bs=4M status=progress
```

---

# 3. Instalar Ubuntu Server

## Durante instalación

Instalar:
- OpenSSH Server

---

# 4. Configurar IP fija

Editar:

```bash
sudo nano /etc/netplan/00-installer-config.yaml
```

Ejemplo:

```yaml
network:
  version: 2
  ethernets:
    eno1:
      dhcp4: false
      addresses:
        - 192.168.1.10/24
      gateway4: 192.168.1.1
      nameservers:
        addresses:
          - 1.1.1.1
```

Aplicar:

```bash
sudo netplan apply
```

---

# 5. Actualizar sistema

```bash
sudo apt update && sudo apt upgrade -y
```

---

# PARTE 3 — Instalar k3s Master

# 1. Instalar k3s

```bash
curl -sfL https://get.k3s.io | sh -
```

---

# 2. Verificar servicio

```bash
sudo systemctl status k3s
```

---

# 3. Ver nodos

```bash
sudo kubectl get nodes
```

---

# 4. Obtener token del cluster

```bash
sudo cat /var/lib/rancher/k3s/server/node-token
```

Guardar:
- TOKEN
- IP DEL MASTER

---

# PARTE 4 — Configurar Worker N100

# 1. Instalar Ubuntu Server

Mismo proceso.

---

# 2. Configurar IP fija

Ejemplo:

```text
192.168.1.20
```

---

# 3. Instalar k3s Agent

```bash
curl -sfL https://get.k3s.io | K3S_URL=https://192.168.1.10:6443 K3S_TOKEN=TOKEN sh -
```

---

# 4. Verificar desde master

```bash
kubectl get nodes
```

---

# PARTE 5 — Configurar Wake-on-LAN

# IMPORTANTE

Esto permite prender nodos apagados.

---

# 1. Activar WoL en BIOS

Buscar opciones:

- Wake-on-LAN
- Wake on PCI-E
- Power on by LAN

Activar.

---

# 2. Instalar ethtool

```bash
sudo apt install ethtool
```

---

# 3. Verificar interfaz

```bash
ip a
```

Ejemplo:
- eno1

---

# 4. Activar WoL

```bash
sudo ethtool -s eno1 wol g
```

---

# 5. Verificar WoL

```bash
sudo ethtool eno1
```

Debe decir:

```text
Wake-on: g
```

---

# 6. Obtener MAC Address

```bash
ip link
```

Guardar MAC.

---

# 7. Instalar wakeonlan en master

```bash
sudo apt install wakeonlan
```

---

# 8. Encender nodo remoto

```bash
wakeonlan MAC_ADDRESS
```

Ejemplo:

```bash
wakeonlan 00:11:22:33:44:55
```

---

# PARTE 6 — Ubuntu Ryzen como Worker

# IMPORTANTE

Ubuntu Ryzen:
- NO usa Proxmox
- es una workstation normal

---

# 1. Instalar Ubuntu Desktop 24.04

---

# 2. Actualizar sistema

```bash
sudo apt update && sudo apt upgrade -y
```

---

# 3. Instalar k3s agent

```bash
curl -sfL https://get.k3s.io | K3S_URL=https://192.168.1.10:6443 K3S_TOKEN=TOKEN sh -
```

---

# 4. Etiquetar nodo potente

```bash
kubectl label node ubuntu-ryzen tier=highpower
```

---

# PARTE 7 — Windows Ryzen como Worker

# IMPORTANTE

Windows:
- NO usa Proxmox
- usa WSL2

---

# 1. Instalar WSL2

Abrir PowerShell Admin:

```powershell
wsl --install
```

Reiniciar.

---

# 2. Instalar Ubuntu desde Microsoft Store

Recomendado:
- Ubuntu 24.04

---

# 3. Dentro de WSL actualizar

```bash
sudo apt update && sudo apt upgrade -y
```

---

# 4. Instalar k3s agent

```bash
curl -sfL https://get.k3s.io | K3S_URL=https://192.168.1.10:6443 K3S_TOKEN=TOKEN sh -
```

---

# 5. Etiquetar nodo

```bash
kubectl label node windows-ryzen tier=highpower
```

---

# PARTE 8 — Etiquetas (Labels)

# Workers pequeños

```bash
kubectl label node worker-n100 tier=lowpower
```

---

# Workers potentes

```bash
kubectl label node ubuntu-ryzen tier=highpower
```

---

# PARTE 9 — Ejecutar cargas pesadas

# Objetivo

Que IA/render/compilación:
- usen nodos Ryzen

---

# Ejemplo Deployment

```yaml
apiVersion: apps/v1
kind: Deployment

metadata:
  name: heavy-workload

spec:
  replicas: 1

  selector:
    matchLabels:
      app: heavy

  template:
    metadata:
      labels:
        app: heavy

    spec:
      nodeSelector:
        tier: highpower

      containers:
      - name: heavy
        image: ubuntu
```

---

# PARTE 10 — Apagar Workers

# IMPORTANTE

Antes de apagar:

```bash
kubectl drain NOMBRE_NODO --ignore-daemonsets
```

---

# Luego apagar

```bash
shutdown now
```

---

# PARTE 11 — Agregar nuevos nodos

# 1. Instalar Ubuntu Server

---

# 2. Configurar IP fija

---

# 3. Instalar k3s agent

```bash
curl -sfL https://get.k3s.io | K3S_URL=https://MASTER_IP:6443 K3S_TOKEN=TOKEN sh -
```

---

# 4. Verificar

```bash
kubectl get nodes
```

---

# 5. Configurar WoL

Mismo proceso anterior.

---

# PARTE 12 — Monitoring

# Recomendado

Instalar:
- Prometheus
- Grafana

---

# PARTE 13 — Comandos útiles

# Ver nodos

```bash
kubectl get nodes
```

---

# Ver pods

```bash
kubectl get pods -A
```

---

# Ver uso de nodos

```bash
kubectl top nodes
```

---

# Ver uso de pods

```bash
kubectl top pods -A
```

---

# Drenar nodo

```bash
kubectl drain NOMBRE_NODO --ignore-daemonsets
```

---

# Reactivar nodo

```bash
kubectl uncordon NOMBRE_NODO
```

---

# PARTE 14 — Escalado futuro

Puedes agregar:

- más N100
- GPU nodes
- AI workers
- NAS
- Longhorn
- ArgoCD
- Terraform
- Ansible

---

# Filosofía Final

## Cluster dinámico

- pocos nodos siempre activos
- potencia bajo demanda

---

# Kubernetes es el cerebro

Kubernetes:
- coordina workloads
- distribuye carga
- maneja nodos dinámicos

---

# Resultado

Un homelab:
- moderno
- eficiente
- silencioso
- escalable
- flexible
- muy cercano a infraestructura real