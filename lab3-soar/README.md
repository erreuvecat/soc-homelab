# Lab 3 — Shuffle SOAR

**Estado:** ⚠️ Parcialmente completado — Shuffle operativo, integraciones pendientes  
**Objetivo:** Desplegar la plataforma SOAR Shuffle y sentar las bases para la automatización del pipeline Wazuh → Shuffle → TheHive.

---

## Índice

1. [Creación de la VM soar-thehive](#1-creación-de-la-vm-soar-thehive)
2. [Instalación de Docker CE](#2-instalación-de-docker-ce)
3. [Despliegue de Shuffle con Docker Compose](#3-despliegue-de-shuffle-con-docker-compose)
4. [Resolución de errores de despliegue](#4-resolución-de-errores-de-despliegue)
5. [Pendientes](#5-pendientes)

---

## 1. Creación de la VM soar-thehive

| Parámetro | Valor |
|---|---|
| VM ID | `204` |
| Name | `soar-thehive` |
| ISO | `ubuntu-22.04.5-live-server-amd64.iso` |
| Machine | `q35` |
| RAM | `6144 MB (6 GB)` |
| CPU | `2 cores · type: host` |
| Disco | `60 GB · local-lvm (SSD)` |
| Red | `vmbr0 · VirtIO` |

Configuración de red estática (`/etc/netplan/00-installer-config.yaml`):

```yaml
network:
  version: 2
  ethernets:
    enp6s18:
      dhcp4: false
      addresses:
        - 192.168.100.108/24
      routes:
        - to: default
          via: 192.168.100.1
      nameservers:
        addresses: [8.8.8.8, 1.1.1.1]
```

```bash
sudo netplan apply && sudo reboot
```

**Resultado:** VM `soar-thehive` operativa con IP estática `192.168.100.108`. ✅

---

## 2. Instalación de Docker CE

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install -y ca-certificates curl gnupg

# Agregar clave GPG oficial de Docker
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | \
  sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg

# Agregar repositorio
echo "deb [arch=amd64 signed-by=/etc/apt/keyrings/docker.gpg] \
  https://download.docker.com/linux/ubuntu jammy stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list

# Instalar
sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-compose-plugin

# Agregar usuario al grupo docker
sudo usermod -aG docker paul
newgrp docker

docker --version
# Docker version 29.4.3, build 055a478 ✅
```

**Resultado:** Docker CE 29.4.3 instalado y operativo. ✅

---

## 3. Despliegue de Shuffle con Docker Compose

**¿Qué es Shuffle?** Plataforma SOAR (Security Orchestration, Automation and Response) open-source. Permite crear workflows visuales (playbooks) que automatizan la respuesta ante incidentes: enriquecer IOCs, abrir tickets en TheHive, notificar al analista.

```bash
mkdir ~/lab3 && cd ~/lab3
curl -fsSL https://raw.githubusercontent.com/Shuffle/Shuffle/main/docker-compose.yml -o docker-compose.yml

# Crear estructura de directorios
mkdir -p ~/lab3/shuffle-database ~/lab3/shuffle-apps ~/lab3/shuffle-files

# Crear archivo de variables de entorno
tee ~/lab3/.env << 'EOF'
FRONTEND_PORT=3001
FRONTEND_PORT_HTTPS=3443
BACKEND_HOSTNAME=shuffle-backend
BACKEND_PORT=5001
SHUFFLE_APP_HOTLOAD_LOCATION=/home/paul/lab3/shuffle-apps
SHUFFLE_FILE_LOCATION=/home/paul/lab3/shuffle-files
OUTER_HOSTNAME=192.168.100.108
SHUFFLE_OPENSEARCH_URL=http://shuffle-opensearch:9200
SHUFFLE_OPENSEARCH_USERNAME=admin
SHUFFLE_OPENSEARCH_PASSWORD=[REDACTED]
SHUFFLE_DEFAULT_USERNAME=admin
SHUFFLE_DEFAULT_PASSWORD=[REDACTED]
EOF
```

```bash
cd ~/lab3 && docker compose up -d
```

Output exitoso:
```
[+] Running 47/47
 ✔ shuffle-backend      Started
 ✔ shuffle-orborus      Started
 ✔ shuffle-opensearch   Started
 ✔ shuffle-frontend     Started
```

**Resultado:** Shuffle SOAR operativo en `https://192.168.100.108:3443`. ✅

---

## 4. Resolución de errores de despliegue

### Error 1 — Variable SHUFFLE_APP_HOTLOAD_FOLDER vs SHUFFLE_APP_HOTLOAD_LOCATION

```
WARN: "SHUFFLE_APP_HOTLOAD_LOCATION" variable is not set
invalid spec: :/shuffle-apps:z: empty section between colons
```

**Causa:** El `docker-compose.yml` espera `SHUFFLE_APP_HOTLOAD_LOCATION`, pero la documentación antigua usa `SHUFFLE_APP_HOTLOAD_FOLDER`.

**Solución:**
```bash
sed -i 's/SHUFFLE_APP_HOTLOAD_FOLDER/SHUFFLE_APP_HOTLOAD_LOCATION/' ~/lab3/.env
```

> **Lección:** Siempre verificar el `docker-compose.yml` descargado contra el `.env` antes de levantar el stack. Los nombres de variables cambian entre versiones.

### Error 2 — OpenSearch pantalla de carga infinita

Todos los contenedores mostraban `Up` pero la UI cargaba indefinidamente.

**Diagnóstico:**
```bash
docker logs shuffle-opensearch 2>&1 | tail -5
# max virtual memory areas vm.max_map_count [65530] likely too low
```

**Solución:**
```bash
sudo chown -R 1000:1000 ~/lab3/shuffle-database
sudo swapoff -a
docker restart shuffle-opensearch
# Esperar 2-3 minutos para inicialización completa (~1.5 GB RAM en arranque)
```

---

## 5. Pendientes

| Tarea | Descripción |
|---|---|
| Instalación de TheHive | Sistema de gestión de casos de incidentes |
| Integración Wazuh → Shuffle | Webhook en Wazuh para enviar alertas nivel ≥5 a Shuffle |
| Integración Shuffle → TheHive | Workflow para apertura automática de casos |
| Enrichment de IOCs | Conectores con VirusTotal y AbuseIPDB |
| Playbook end-to-end | Alerta RDP Brute Force → Enriquecimiento → Caso en TheHive → Notificación |

**Motivo del cierre:** El stack completo del lab (Wazuh + Windows + Kali + SOAR + servicios del HomeLab) requería ~24 GB de RAM. El hardware disponible tiene 16 GB. Ver [lecciones aprendidas](../lecciones/README.md) para el análisis completo.
