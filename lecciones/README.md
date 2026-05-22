# Lecciones Aprendidas

---

## Técnicas

| # | Lección | Contexto | Impacto |
|---|---|---|---|
| 1 | La contraseña `admin` de Wazuh está sincronizada en 3 componentes (Indexer, Filebeat, Dashboard). Cambiarla desde la UI solo actualiza uno, rompiendo el sistema con error 500. El procedimiento correcto requiere `wazuh-passwords-tool.sh` + `filebeat keystore add password --force` en la misma operación. | Lab 1 | Pérdida temporal de acceso, recuperado en ~15 min |
| 2 | Los scans SYN de Nmap (`-sS`) son sigilosos por diseño. Sin un IDS/IPS de red (Suricata, Zeek), el reconocimiento activo pasa completamente desapercibido para el SIEM. Este es el gap de coverage más común en entornos SOC reales. | Lab 2 | Arquitectura de detección incompleta documentada |
| 3 | Wazuh implementa correlación temporal de eventos. Intentos SSH fallidos individuales (nivel 5) se correlan y escalan progresivamente a nivel 10 cuando el patrón temporal confirma brute force. Este comportamiento es el corazón del valor de un SIEM real. | Lab 2 | Demostración de detección SIEM real |
| 4 | Hydra es inestable para ataques RDP (módulo experimental). Crowbar es la herramienta correcta para este protocolo. La wordlist `rockyou.txt` contiene caracteres non-UTF8 que requieren preprocesamiento con `strings` antes de usarse. | Lab 2 | Resolución de problemas de tooling |
| 5 | Los discos SMR (Shingled Magnetic Recording) como el Seagate SkyHawk no son aptos para VMs con I/O aleatorio intenso. La latencia de escritura es prohibitiva para bases de datos, indexers y sistemas de logs. Para VMs de laboratorio, SSD o discos CMR son obligatorios. | Cierre | Limitación de hardware que forzó el cierre del Lab 3 |
| 6 | Un CVE con CVSS 9.8 no implica acción inmediata automática. El análisis contextual (arquitectura del sistema, vector de explotación, exposición real, disponibilidad de parche) es lo que determina la respuesta correcta. En CVE-2026-5121: Critical en papel, riesgo bajo real (sistema 64-bit, red local, sin parche upstream). | Lab 1 | Pensamiento analítico SOC documentado |
| 7 | La variable de entorno de Shuffle cambió de nombre entre versiones (`SHUFFLE_APP_HOTLOAD_FOLDER` → `SHUFFLE_APP_HOTLOAD_LOCATION`). Siempre verificar el `docker-compose.yml` descargado contra el `.env` antes de levantar el stack. | Lab 3 | Resolución de problemas de despliegue Docker |

---

## Cierre del laboratorio

### Limitación de hardware

| Servicio | RAM asignada |
|---|---|
| Proxmox OS + overhead | ~1.5 GB |
| Wazuh Manager (VM 101) | 6 GB |
| Windows 10 Endpoint (VM 202) | 3 GB |
| Kali Attacker (VM 203) | 2 GB |
| SOAR VM (VM 204) | 6 GB |
| Servicios del HomeLab (LXCs) | ~1.5 GB |
| **Total requerido** | **~20 GB** |
| **Total disponible** | **16 GB** |

El stack completo supera la RAM disponible. Se llegó a este límite al intentar ejecutar Lab 3 con todos los servicios activos.

### Limitación de almacenamiento

El disco Seagate SkyHawk ST6000VX0003 es un disco SMR. Las VMs en el pool NFS sobre este disco experimentaron degradación severa de rendimiento — los discos SMR tienen latencia de escritura aleatoria hasta 10x mayor que CMR, haciéndolos inadecuados para VMs con I/O intensivo.

---

## Notas de portafolio

- Documentar la **no detección** (ej. el scan Nmap silencioso) es tan valioso como documentar las detecciones. Muestra comprensión real de los límites del SIEM.
- El número concreto **4.763 eventos en una hora** es verificable, específico y resume el volumen real del lab.
- Aplicar el workflow SOC sobre CVE-2026-5121 demuestra razonamiento analítico de nivel Tier 1, no solo ejecución de comandos.
- Documentar errores resueltos demuestra capacidad de troubleshooting real — más valioso en un CV que una guía donde todo funciona a la primera.
