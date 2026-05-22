# Lab 2 — Simulación de Ataques con Kali Linux

**Estado:** ✅ Completado  
**Objetivo:** Simular ataques reales desde una máquina atacante y verificar que el SIEM los detecta, correlaciona y alerta correctamente.  
**Total generado: 4.763 eventos en ~1 hora.**

---

## Índice

1. [Creación de la VM Kali Attacker](#1-creación-de-la-vm-kali-attacker)
2. [Fase 1 — Reconocimiento con Nmap](#2-fase-1--reconocimiento-con-nmap)
3. [Fase 2 — Brute Force RDP con Crowbar](#3-fase-2--brute-force-rdp-con-crowbar)
4. [Fase 3 — Brute Force SSH con Hydra](#4-fase-3--brute-force-ssh-con-hydra)
5. [Resumen de detecciones](#5-resumen-de-detecciones)

---

## 1. Creación de la VM Kali Attacker

| Parámetro | Valor |
|---|---|
| VM ID | `203` |
| Name | `kali-attacker` |
| ISO | `kali-linux-2025.1-installer-amd64.iso` |
| Machine | `q35` |
| RAM | `2048 MB (2 GB)` |
| CPU | `2 cores · type: host` |
| Disco | `40 GB · tank-vms (NFS)` |
| Red | `vmbr0 · VirtIO` |

Instalación con entorno gráfico Xfce y herramientas `kali-linux-default`.

Post-instalación:

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install -y crowbar
sudo gunzip /usr/share/wordlists/rockyou.txt.gz
```

Snapshot guardado como `kali-base` para poder restaurar el estado limpio.

---

## 2. Fase 1 — Reconocimiento con Nmap

**MITRE ATT&CK:** T1046 — Network Service Scanning

```bash
nmap -sS -sV 192.168.100.103
```

| Flag | Significado |
|---|---|
| `-sS` | SYN scan (sigiloso) — no completa el handshake TCP, no queda registro en el target |
| `-sV` | Version detection — identifica nombre y versión del servicio |

**Resultado del scan:**

```
PORT     STATE SERVICE         VERSION
3389/tcp open  ms-wbt-server   Microsoft Terminal Services
5357/tcp open  http            Microsoft HTTPAPI httpd 2.0
```

| Puerto | Riesgo | Observación |
|---|---|---|
| 3389/tcp (RDP) | **ALTO** | Vector principal de brute force, expuesto sin restricción |
| 5357/tcp (WS-Discovery) | Bajo | Servicio de descubrimiento normal en LAN |

**Detección en Wazuh: ❌ No detectado.**

El scan SYN no completa el handshake TCP → no queda registro en los logs de Windows → Wazuh no ve nada. Sin un IDS/IPS de red (Suricata, Zeek), el reconocimiento activo pasa desapercibido para el SIEM. Este gap de coverage es documentado intencionalmente.

---

## 3. Fase 2 — Brute Force RDP con Crowbar

**MITRE ATT&CK:** T1110.001 — Brute Force: Password Guessing

### Errores resueltos antes del ataque exitoso

**Intento 1 — Hydra (descartado):**
```bash
hydra -l Administrator -P /usr/share/wordlists/rockyou.txt rdp://192.168.100.103 -t 4
# Resultado: 0 valid passwords found (terminó sin intentar la wordlist)
```
El módulo RDP de Hydra es experimental e inestable. Se cambió a Crowbar.

**Intento 2 — Crowbar con encoding incorrecto:**
```bash
crowbar -b rdp -s 192.168.100.103/32 -u Administrator -C /usr/share/wordlists/rockyou.txt -n 1
# Error: UnicodeDecodeError — caracteres non-UTF8 en rockyou.txt
```

**Solución — wordlist limpia:**
```bash
strings /usr/share/wordlists/rockyou.txt | head -5000 > /tmp/rockyou_clean.txt
```

### Ataque exitoso

```bash
crowbar -b rdp -s 192.168.100.103/32 -u Administrator -C /tmp/rockyou_clean.txt -n 1
```

| Flag | Significado |
|---|---|
| `-b rdp` | Protocolo objetivo |
| `-s 192.168.100.103/32` | IP objetivo (/32 = host único) |
| `-u Administrator` | Usuario a atacar |
| `-n 1` | Hilos paralelos |

**Detección en Wazuh:**

| Timestamp | Rule ID | Descripción | Nivel |
|---|---|---|---|
| 19:54:01 | 60122 | Windows: Logon Failure — Unknown user or bad password | 5 |
| 19:54:01 | 60122 | Windows: Logon Failure — Unknown user or bad password | 5 |
| 19:54:01 | 60122 | Windows: Logon Failure — Unknown user or bad password | 5 |

4 intentos fallidos en el mismo segundo — patrón característico de brute force automatizado. Wazuh correlacionó automáticamente los eventos Windows Security Log (Event ID 4625) con Rule 60122.

**Resultado:** Brute Force RDP detectado. Rule 60122 activada. ✅

---

## 4. Fase 3 — Brute Force SSH con Hydra

**MITRE ATT&CK:** T1110.001 — Brute Force: Password Guessing

```bash
hydra -l root -P /usr/share/wordlists/rockyou.txt ssh://192.168.100.50 -t 4 -V
```

| Flag | Significado |
|---|---|
| `-l root` | Usuario objetivo |
| `-P rockyou.txt` | Wordlist de contraseñas |
| `-t 4` | 4 hilos paralelos |
| `-V` | Verbose — muestra cada intento |

**Verificación de intentos registrados:**

```bash
# En el Wazuh Manager:
sudo grep "192.168.100.114" /var/log/auth.log | grep "Failed" | wc -l
# Resultado: 4763
```

**Detección y correlación en Wazuh:**

| Rule ID | Descripción | Nivel | Tipo |
|---|---|---|---|
| 5760 | `sshd: authentication failed` | 5 | Alerta individual |
| 5758 | Maximum authentication attempts exceeded | 8 | Escalada por conexión |
| 5551 | PAM: Multiple failed logins in small period | **10** | Correlación temporal — BRUTE FORCE |
| 2502 | syslog: User missed password multiple times | **10** | Correlación por usuario — BRUTE FORCE |

**Escalado de niveles:**
```
Nivel 5  → Intentos fallidos individuales (puede ser un typo)
Nivel 8  → Límite de intentos por conexión excedido (anormal)
Nivel 10 → Patrón temporal = brute force confirmado
```

Wazuh no solo registra eventos — correlaciona patrones en el tiempo. Este es el corazón del valor de un SIEM real.

**Resultado:** Brute Force SSH detectado y correlacionado. 4.763 eventos, nivel 10. ✅

---

## 5. Resumen de detecciones

| # | Ataque | Herramienta | Target | Detectado | Nivel máx. | MITRE |
|---|---|---|---|---|---|---|
| 1 | Port scan SYN | Nmap -sS | Win10 | ❌ No | — | T1046 |
| 2 | Brute Force RDP | Crowbar | Win10 | ✅ Sí | 5 | T1110.001 |
| 3 | Brute Force SSH | Hydra | Wazuh Mgr | ✅ Sí | **10** | T1110.001 |

**Total: 4.763 eventos en ~1 hora.**

La no detección del scan Nmap está documentada intencionalmente. En un entorno productivo se mitigaría con Suricata como IDS de red. La ausencia de detección es tan informativa como la detección misma — revela los gaps de coverage del SIEM.
