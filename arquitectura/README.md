# SOC HomeLab

Entorno Blue Team funcional construido sobre hardware reciclado, usando herramientas de nivel empresarial en red local segmentada.

**Hardware:** HP Compaq Pro 4300 SFF · Intel Core i7-3770S · 16 GB DDR3 · SSD 240 GB  
**Período:** Abril – Mayo 2026

---

## Stack tecnológico

| Capa | Tecnología | Versión |
|---|---|---|
| Hipervisor | Proxmox VE | 9.1.1 |
| SIEM / XDR | Wazuh | 4.11 |
| Endpoint telemetría | Sysmon (SwiftOnSecurity) | 15.x |
| Plataforma atacante | Kali Linux | 2025 |
| Brute Force RDP | Crowbar | 0.4.2 |
| Brute Force SSH | Hydra | 9.x |
| Reconocimiento | Nmap | 7.x |
| SOAR | Shuffle | latest |
| Contenedores | Docker CE | 29.4.3 |

---

## Arquitectura

```
Red local: 192.168.100.0/24

┌─────────────────────────────────────────────────────────────┐
│  Proxmox VE 9.1.1                                            │
│                                                              │
│  ┌──────────────────────┐   ┌──────────────────────────┐    │
│  │  VM 101              │   │  VM 202                  │    │
│  │  wazuh-manager       │◄──│  win10-endpoint          │    │
│  │  Ubuntu 22.04        │   │  Windows 10 Pro          │    │
│  │  Wazuh 4.11          │   │  Agente 002 + Sysmon     │    │
│  └──────────┬───────────┘   └──────────────────────────┘    │
│             │ agente 001                                      │
│  ┌──────────▼───────────┐   ┌──────────────────────────┐    │
│  │  Proxmox host        │   │  VM 203                  │    │
│  │  Debian GNU/Linux 13 │   │  kali-attacker           │    │
│  │  Agente 001          │   │  Kali Linux 2025         │    │
│  └──────────────────────┘   └──────────────────────────┘    │
│                                                              │
│  ┌──────────────────────┐                                    │
│  │  VM 204              │                                    │
│  │  soar-thehive        │                                    │
│  │  Ubuntu 22.04        │                                    │
│  │  Shuffle SOAR        │                                    │
│  └──────────────────────┘                                    │
└─────────────────────────────────────────────────────────────┘
```

| VM | OS | Rol |
|---|---|---|
| wazuh-manager | Ubuntu 22.04 LTS | SIEM central |
| win10-endpoint | Windows 10 Pro | Endpoint víctima |
| kali-attacker | Kali Linux 2025 | Máquina atacante |
| soar-thehive | Ubuntu 22.04 LTS | Plataforma SOAR |

---

## Resultados clave

- **507 eventos MITRE ATT&CK** generados automáticamente por Sysmon en los primeros minutos de operación
- **4.763 eventos** de brute force SSH detectados y correlacionados — escalado de alerta nivel 5 → 10
- **CVE-2026-5121 (CVSS 9.8)** analizado aplicando workflow SOC completo: Detectar → Analizar → Decidir → Monitorear → Remediar
- Reconocimiento Nmap SYN **no detectado** por el SIEM — gap de coverage documentado (requiere IDS de red como Suricata)

---

## Laboratorios

| Lab | Descripción | Estado |
|---|---|---|
| [Lab 1 — Wazuh SIEM](./lab1-wazuh/README.md) | Despliegue, agentes Linux/Windows, Sysmon, análisis de CVE | ✅ Completado |
| [Lab 2 — Simulación de Ataques](./lab2-ataques/README.md) | Reconocimiento Nmap, Brute Force RDP/SSH, detección SIEM | ✅ Completado |
| [Lab 3 — Shuffle SOAR](./lab3-soar/README.md) | Despliegue de plataforma SOAR en Docker | ⚠️ Parcial |
| [Lecciones aprendidas](./lecciones/README.md) | Troubleshooting documentado, gaps de seguridad identificados | — |
