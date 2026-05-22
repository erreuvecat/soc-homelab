# Lab 1 — Wazuh SIEM: Despliegue, Agentes y Detección

**Estado:** ✅ Completado  
**Duración:** ~4 sesiones de trabajo (Abril 2026)  
**Objetivo:** Desplegar Wazuh 4.11 en modo all-in-one, conectar agentes heterogéneos (Linux + Windows), instrumentar el endpoint Windows con Sysmon y demostrar detección activa de amenazas.

---

## Índice

1. [Creación de la VM Wazuh Manager](#1-creación-de-la-vm-wazuh-manager)
2. [Configuración de red estática](#2-configuración-de-red-estática)
3. [Instalación de Wazuh 4.11](#3-instalación-de-wazuh-411-all-in-one)
4. [Agente 001 — Host Proxmox](#4-agente-001--host-proxmox)
5. [Incidente: Rotura del dashboard](#5-incidente-rotura-del-dashboard-por-cambio-de-contraseña)
6. [Creación de VM Windows 10 Endpoint](#6-creación-de-la-vm-windows-10-endpoint)
7. [Agente 002 — Windows 10 Pro](#7-agente-002--windows-10-pro)
8. [Sysmon con SwiftOnSecurity](#8-sysmon-con-swiftonsecurity)
9. [Detección de CVE — Workflow SOC completo](#9-detección-de-cve-2026-5121--workflow-soc-completo)

---

## 1. Creación de la VM Wazuh Manager

**Decisión de diseño:** VMs del lab en SSD local (`local-lvm`) y no en el pool NFS del disco SMR. Wazuh realiza I/O intensivo de logs; el SMR degradaría severamente el rendimiento.

Desde la WebUI de Proxmox → Create VM:

| Parámetro | Valor |
|---|---|
| VM ID | `101` |
| Name | `wazuh-manager` |
| ISO | `ubuntu-22.04.5-live-server-amd64.iso` |
| Machine | `q35` |
| SCSI Controller | `VirtIO SCSI single` |
| Disco | `64 GB · local-lvm · Write back` |
| CPU | `4 cores · type: host` |
| RAM | `6144 MB (6 GB)` |
| Red | `vmbr0 · VirtIO` |

Instalación Ubuntu 22.04 Server con OpenSSH habilitado.

---

## 2. Configuración de red estática

```bash
sudo nano /etc/netplan/00-installer-config.yaml
```

```yaml
network:
  version: 2
  ethernets:
    enp6s18:
      dhcp4: false
      addresses:
        - 192.168.100.50/24
      routes:
        - to: default
          via: 192.168.100.1
      nameservers:
        addresses: [8.8.8.8, 1.1.1.1]
```

```bash
sudo netplan apply
sudo chmod 600 /etc/netplan/00-installer-config.yaml
sudo reboot
```

> **Nota:** En VMs Proxmox con chipset q35, la interfaz de red se llama `enp6s18` en lugar del `ens18` habitual. Siempre verificar con `ip a` antes de editar netplan.

**Resultado:** IP estática `192.168.100.50`, SSH funcional. ✅

---

## 3. Instalación de Wazuh 4.11 (all-in-one)

```bash
sudo apt update && sudo apt upgrade -y
curl -sO https://packages.wazuh.com/4.11/wazuh-install.sh && sudo bash wazuh-install.sh -a
```

El flag `-a` instala los tres componentes en una sola operación:
- **Wazuh Manager** — motor de análisis y correlación de eventos
- **Wazuh Indexer** — base de datos OpenSearch para almacenamiento de eventos
- **Wazuh Dashboard** — interfaz web de análisis y visualización

Tiempo de instalación: ~15-20 minutos.

Al finalizar, el instalador muestra las credenciales generadas automáticamente. **Guardarlas inmediatamente** — son necesarias para todos los pasos siguientes.

Verificar servicios activos:

```bash
sudo systemctl status wazuh-manager wazuh-indexer wazuh-dashboard
# Expected: Active: active (running) para los tres
```

**Resultado:** Wazuh 4.11 operativo. Dashboard mostrando 216 alertas de automonitoreo. ✅

---

## 4. Agente 001 — Host Proxmox (Debian GNU/Linux 13)

Desde la terminal del host Proxmox:

```bash
wget https://packages.wazuh.com/4.x/apt/pool/main/w/wazuh-agent/wazuh-agent_4.11.0-1_amd64.deb
sudo WAZUH_MANAGER='192.168.100.50' WAZUH_AGENT_NAME='proxmox' dpkg -i ./wazuh-agent_4.11.0-1_amd64.deb
```

**Error encontrado:** El instalador no aplicó correctamente la variable `WAZUH_MANAGER`, fallando con:
```
ERROR: (4112): Invalid server address found: 'MANAGER_IP'
```

**Solución — editar `ossec.conf` manualmente:**

```bash
nano /var/ossec/etc/ossec.conf
# Cambiar: <address>MANAGER_IP</address>
# Por:     <address>192.168.100.50</address>
```

```bash
sudo systemctl daemon-reload
sudo systemctl enable wazuh-agent
sudo systemctl start wazuh-agent
```

**Resultado:** Agente `proxmox` (001) activo, reportando logs del hipervisor en tiempo real. ✅

---

## 5. Incidente: Rotura del dashboard por cambio de contraseña

**Descripción:** Se intentó cambiar la contraseña del usuario `admin` desde la UI de Wazuh. Resultado: error HTTP 500, acceso completamente bloqueado.

**Causa raíz:** La contraseña de `admin` está sincronizada entre tres componentes independientes:

```
1. Wazuh Indexer (OpenSearch) — almacena el hash
2. Filebeat keystore           — secreto para enviar logs al indexer
3. Wazuh Dashboard             — conexión al indexer
```

Cambiarla desde la UI solo actualiza el indexer, dejando Filebeat con la contraseña vieja.

**Procedimiento correcto para cambiar contraseña en Wazuh:**

```bash
# Paso 1: Sincronizar en el indexer con la herramienta oficial
sudo /usr/share/wazuh-indexer/plugins/opensearch-security/tools/wazuh-passwords-tool.sh \
  -u admin -p 'NUEVA_CONTRASEÑA'

# Paso 2: Actualizar el keystore de Filebeat con la misma contraseña
sudo filebeat keystore add password --force
# Ingresar la misma contraseña cuando se solicite

# Paso 3: Reiniciar todos los servicios en orden
sudo systemctl restart filebeat wazuh-indexer wazuh-manager wazuh-dashboard
```

> **Regla de oro:** En Wazuh, la contraseña de `admin` SIEMPRE debe cambiarse con `wazuh-passwords-tool.sh` y SIEMPRE debe actualizarse simultáneamente en el keystore de Filebeat. Son operaciones inseparables.

**Resultado:** Dashboard restaurado en ~15 min. ✅

---

## 6. Creación de la VM Windows 10 Endpoint

| Parámetro | Valor |
|---|---|
| VM ID | `202` |
| Name | `win10-endpoint` |
| ISO principal | `Win10_22H2_Spanish_x64.iso` |
| ISO secundario | `virtio-win.iso` (drivers VirtIO) |
| Machine | `q35` |
| Disco | `60 GB · local-lvm` |
| CPU | `2 cores · type: host` |
| RAM | `3072 MB (3 GB)` |
| Red | `vmbr0 · VirtIO` |

**Nota crítica sobre VirtIO:** Sin los drivers VirtIO, Windows no detecta el disco durante la instalación. Al aparecer la pantalla de selección de disco vacía: Cargar controlador → CD VirtIO → `viostor\w10\amd64`.

Post-instalación: instalar driver de red desde Administrador de dispositivos → CD VirtIO → `NetKVM\w10\amd64`.

Habilitar RDP: Configuración → Sistema → Escritorio remoto → Activar.

**Resultado:** Windows 10 Pro instalado, IP `192.168.100.103`, RDP activo. ✅

---

## 7. Agente 002 — Windows 10 Pro

Desde el Wazuh Dashboard → Endpoints Summary → Deploy new agent → Windows.

Ejecutar en **PowerShell como Administrador** en la VM Windows:

```powershell
Invoke-WebRequest `
  -Uri https://packages.wazuh.com/4.x/windows/wazuh-agent-4.11.2-1.msi `
  -OutFile $env:tmp\wazuh-agent.msi

msiexec.exe /i $env:tmp\wazuh-agent.msi /q `
  WAZUH_MANAGER='192.168.100.50' `
  WAZUH_AGENT_NAME='win10-endpoint' `
  WAZUH_REGISTRATION_SERVER='192.168.100.50'

NET START WazuhSvc
```

**Resultado:** Dos agentes activos — `proxmox` (001) y `win10-endpoint` (002). ✅

---

## 8. Sysmon con SwiftOnSecurity

**¿Por qué Sysmon?** El agente Wazuh recolecta el Event Log estándar de Windows. Sysmon agrega telemetría profunda: creación de procesos con hash, conexiones de red por proceso, cambios de registro, carga de DLLs.

La configuración **SwiftOnSecurity** es un proyecto open-source ampliamente adoptado que captura amenazas relevantes sin generar ruido excesivo.

```powershell
# Descargar Sysmon
Invoke-WebRequest -Uri https://download.sysinternals.com/files/Sysmon.zip -OutFile $env:temp\Sysmon.zip
Expand-Archive $env:temp\Sysmon.zip -DestinationPath $env:temp\Sysmon

# Descargar config SwiftOnSecurity
Invoke-WebRequest `
  -Uri https://raw.githubusercontent.com/SwiftOnSecurity/sysmon-config/master/sysmonconfig-export.xml `
  -OutFile $env:temp\sysmonconfig.xml

# Instalar
& "$env:temp\Sysmon\Sysmon64.exe" -accepteula -i $env:temp\sysmonconfig.xml
```

Configurar el agente Wazuh para recolectar el canal de Sysmon — agregar en `C:\Program Files (x86)\ossec-agent\ossec.conf` antes de `</ossec_config>`:

```xml
<localfile>
  <location>Microsoft-Windows-Sysmon/Operational</location>
  <log_format>eventchannel</log_format>
</localfile>
```

```powershell
NET STOP WazuhSvc && NET START WazuhSvc
```

**Eventos detectados en los primeros minutos:**

| Rule ID | Descripción | Técnica MITRE ATT&CK |
|---|---|---|
| 92201 | `powershell.exe` detectado en ubicación crítica | T1059.001 — PowerShell |
| 92066 | `SecEdit.exe` en ubicación sospechosa | T1569 — System Services |
| 92031 | Discovery activity executed | T1082 — System Information Discovery |
| 92039 | `net.exe` account discovery | T1087 — Account Discovery |

**Total: 507 eventos MITRE ATT&CK en los primeros minutos de operación.** ✅

---

## 9. Detección de CVE-2026-5121 — Workflow SOC completo

El módulo de Vulnerability Detection de Wazuh detectó automáticamente `CVE-2026-5121` en el agente `proxmox` durante un scan de inventario.

| Campo | Valor |
|---|---|
| CVE ID | CVE-2026-5121 |
| Paquete | `libarchive13t64:amd64` v3.7.4-4 |
| CVSS Score | **9.8 (Critical)** |
| Vector | ISO9660 maliciosa → heap buffer overflow → RCE |

**Workflow SOC aplicado:**

```
DETECTAR
  Wazuh alertó CVE-2026-5121 en libarchive (CVSS 9.8)
      ↓
ANALIZAR
  ✓ Vector requiere ISO9660 maliciosa procesada por la librería
  ✓ Vulnerabilidad afecta principalmente arquitecturas 32-bit
  ✓ Package.architecture = "amd64" (64-bit) — riesgo reducido
  ✓ Sin procesos activos procesando ISOs de terceros
  ✓ Sistema en red local, sin exposición a internet
  ✓ Sin parche disponible en Debian trixie upstream
      ↓
DECIDIR
  Riesgo aceptado con justificación documentada
      ↓
MONITOREAR
  sudo apt update && apt-cache policy libarchive13t64  (verificación semanal)
      ↓
REMEDIAR (pendiente)
  sudo apt upgrade libarchive13t64  — cuando el parche esté disponible
```

**Conclusión:** CVSS 9.8 en papel → Riesgo bajo real con contexto. Un CVE crítico no implica acción inmediata automática — el análisis contextual es lo que determina la respuesta correcta. ✅
