# Linux-SOC-Labs - Portafolio SOC L1 (Blue Team)

Este repositorio reúne mis laboratorios prácticos orientados al rol de **SOC Analyst L1 / Blue Team**. Documenta automatización con Bash, análisis de logs en Linux y captura e inspección de tráfico de red en laboratorio controlado.

**Entorno:** Kali Linux en VirtualBox | **Lenguaje:** Shell (Bash)

---

## Módulo 1 y 2: Fundamentos de Terminal y Archivos

### 1. Entorno de Trabajo y Herramientas
- **Sistema Operativo:** Kali Linux (VirtualBox)
- **Lenguaje:** Bash Shell Scripting

### 2. Comandos Practicados

**2.1 Ubicación y Navegación**
- `pwd`: Ruta del directorio actual
- `ls / ls -la`: Listado incluyendo ocultos y permisos
- `cd`: Navegación entre carpetas

**2.2 Inspección de Archivos y Eventos**
- `cat`: Visualización de archivos en terminal
- `ssh`: Generación de intentos de conexión local para pruebas

**2.3 Aplicación SOC**
- Explorar estructura de archivos en sistema comprometido
- Identificar `Failed password` y conexiones sospechosas
- Base para investigación de fuerza bruta

## Módulo 3: Monitoreo de SSH, Gestión de Servicios y Puertos

### Descripción General
Inspección de puertos abiertos, análisis de conexiones activas y supervisión en tiempo real del servicio SSH. Incluye automatización con `monitor_ssh.sh`.

### Comandos y Herramientas Utilizadas (evidencia real de la imagen)
```bash
ss -tulanp                          # Verifica puertos en escucha, confirma 22/tcp para sshd
ssh hacker_falso@127.0.0.1           # Generación de intentos fallidos (lo que provoca pam_unix: check pass; user unknown)
sudo journalctl -u ssh -f           # Monitoreo en vivo del servicio SSH
sudo journalctl -u ssh -n 15        # Lectura de las últimas 15 líneas - comando usado en la captura
chmod 600 ~/.ssh/id_rsa             # Asignación de permisos estrictos para llave privada
```

### Evidencia de Análisis
![Captura de Logs SSH](./log.ssh.jpeg)
*Captura con `journalctl -u ssh -n 15` tras 3 intentos con usuario `hacker_falso`. Se observan los fallos `Invalid user`, `pam_unix(sshd:auth): check pass; user unknown` y `authentication failure`, confirmando detección de fuerza bruta.*

## Módulo 4: Captura y Análisis de Tráfico de Red (`captura_red.sh`)

### Objetivo
Capturar tráfico en vivo, exportar a `.pcap` e inspeccionar protocolos y capas OSI para análisis SOC.

### Herramientas
- **tcpdump**: Captura CLI en tiempo real
- **Wireshark**: Inspección profunda

### Procedimiento
```bash
ip a                                              # Identifica interfaz eth0 e IP (10.0.2.15)
sudo tcpdump -i eth0 -c 20 -w red_laboratorio.pcap   # Captura solo 20 paquetes - como se ve en la imagen
```
1. Identificación de interfaz `eth0` con `ip a`
2. Captura en vivo de 20 paquetes
3. Exportación a `red_laboratorio.pcap`
4. Inspección OSI (L3 IP, L4 TCP/UDP) y filtros `icmp`, `dns`
5. Automatización con script `captura_red.sh`

![Evidencia Wireshark](./captura_modulo4.jpeg)
*Captura de 20 paquetes con tcpdump y análisis posterior en Wireshark*

## Módulo 5: SIEM (Splunk), Análisis de Logs y Triaje

### Objetivo
Investigación de eventos con filtros en Linux y consultas SPL en Splunk para triaje y contención inicial.

### Consultas Integradas

**Linux (Procesamiento de Logs)**
```bash
# Top IPs con intentos de acceso
cat access.log | awk '{print $1}' | sort | uniq -c | sort -nr | head -n 10

# Login exitoso de IP sospechosa
grep "10.0.0.55" /var/log/auth.log | grep -i "Accepted"
```

**Splunk (SPL)**
```spl
index=main sourcetype=syslog "Failed password" | stats count by src_ip | sort -count
index=main src_ip="10.0.0.55" "Accepted password"
```

### Playbook de Contención
```bash
sudo iptables -A INPUT -s IP_ATACANTE -j DROP  # Bloqueo firewall
sudo ip link set dev eth0 down                 # Aislamiento de red
sudo passwd -l usuario                         # Congelar cuenta
sudo pkill -u usuario                          # Terminar sesiones
```

## Módulo 6: Inspección de Tráfico con tcpdump y Wireshark

### Objetivo
Analizar `.pcap` para detectar tráfico no cifrado y posible exfiltración de credenciales.

### Captura en Consola
```bash
sudo tcpdump -i eth0 -w captura.pcap   # Captura tráfico de eth0 y guarda evidencia en .pcap
sudo tcpdump -r captura.pcap           # Lee e inspecciona el .pcap en terminal sin Wireshark
```

### Filtros Clave Wireshark
```bash
ip.addr == 192.168.1.105               # Tráfico completo de IP sospechosa (origen/destino)
http                                   # Peticiones HTTP en texto plano (puerto 80)
http.request.method == "POST"          # Busca envíos de formularios / credenciales
dns                                    # Consultas de nombres de dominio
```

**Conclusión:** Tráfico HTTP en puerto 80 expone usuario/contraseña en texto plano. Refuerza necesidad de TLS/HTTPS (443).

## Módulo 7: Análisis de Malware e IoCs

### Descripción General
Clasificación de malware y procedimiento SOC para identificación y extracción de evidencias.

### 1. Clasificación de Amenazas
- **Ransomware:** Cifrado con fines de extorsión
- **InfoStealer / Spyware:** Robo de credenciales y tokens
- **Trojan:** Código malicioso en ejecutable legítimo aparente
- **Worm:** Propagación autónoma por red
- **Rootkit:** Ocultamiento a nivel kernel para evadir AV/EDR

### 2. Indicadores de Compromiso (IoCs)
| Tipo de IoC | Ejemplo / Formato | Uso en el SOC |
| :--- | :--- | :--- |
| **Archivo** | SHA-256, MD5 | Identificación en VirusTotal / EDR |
| **Red (C2)** | IP, Dominio, URL | Bloqueo en Firewall, Proxy, DNS Sinkhole |
| **Host** | %APPDATA%, Clave Run | Aislamiento y reglas de detección |

### 3. Flujo Operativo de Triaje
```bash
# Extracción de hash
Get-FileHash -Algorithm SHA256 .\archivo.exe   # PowerShell
sha256sum archivo.exe                           # Linux
```
1. Extracción de hash
2. Análisis estático en VirusTotal
3. Análisis dinámico en sandbox (ANY.RUN, Hybrid Analysis)
4. Contención y bloqueo en EDR

**Estado**
- [x] Conceptos de Malware e IoCs
- [x] Flujo de Triaje y Análisis
- [x] Casos de Práctica y Extracción de IoCs

## Módulo 8: Análisis de Phishing e Investigación de Headers

### 1. Capas de Inspección
- **Cabecera:** Metadatos y enrutamiento, valida IP origen y autenticación
- **Cuerpo y URLs:** Ingeniería social y credential harvesting
- **Adjuntos:** Payloads maliciosos (exe, macro, zip)

### 2. Validación de Autenticación
| Mecanismo | Función | Criterio de Verificación |
| :--- | :--- | :--- |
| **SPF** | IPs autorizadas para enviar por dominio | FAIL = IP no autorizada |
| **DKIM** | Firma criptográfica de integridad | FAIL = mensaje alterado |
| **DMARC** | Política para fallos SPF/DKIM | reject / quarantine / none |

### 3. Flujo Operativo
1. Recepción de ticket (.eml/.msg)
2. Análisis de headers (Google Toolbox, MXToolbox)
3. Inspección segura de URLs (URLScan.io, VirusTotal)
4. Hash de adjuntos (SHA-256) y reputación

### 4. Contención (Verdadero Positivo)
- Bloqueo perimetral de dominio, IP y URLs
- Purga de buzones en M365
- Notificación y cierre de ticket

**Estado**
- [x] Conceptos de Phishing e Ingeniería Social
- [x] Análisis de Cabeceras (SPF / DKIM / DMARC)
- [x] Inspección de URLs e IoCs en Sandbox
- [x] Playbook de Contención y Respuesta

## Módulo 9: Auditoría y Detección de Fuerza Bruta SSH

### Descripción
Análisis de `~/analisis_ataque.log` (simulación de `/var/log/auth.log`) para detectar fuerza bruta SSH, aislar IoCs e IPs sospechosas.

### Metodología
```bash
cat ~/analisis_ataque.log                                      # Visualiza log con intentos Failed password
grep "Failed password" ~/analisis_ataque.log | awk '{print $11}' | sort | uniq -c | sort -nr   # Extrae IP origen (campo $11), agrupa y rankea por frecuencia
```

1. **Filtrado:** Identificar `Failed password`
2. **Aislamiento:** Extraer IP origen con `awk '{print $11}'` 
3. **Consolidación:** Agrupar, contar y rankear IPs con `sort | uniq -c | sort -nr`

### Resultado
Identificación de IP `192.168.1.105` con 3 intentos en intervalo corto, patrón de fuerza bruta confirmado. En entorno real sería `/var/log/auth.log` con `$(NF-3)`.

### Evidencia
![Evidencia del Análisis](./evidencia.jpeg)
*Procesamiento de `analisis_ataque.log` - Top IPs atacantes: 192.168.1.105 (3) y 10.0.0.50 (1)*

### Aplicación SOC L1
- Generación de IoCs
- Bloqueo preventivo en firewall
- Creación de reglas de correlación para SIEM

## Módulo 10: Automatización Defensiva, Triaje y Cierre de Fundamentos

### 1. Introducción
Cierre de fundamentos Linux, Redes y Análisis de Tráfico (Módulos 1-9). Puente hacia operación defensiva avanzada.

### 2. Síntesis del Aprendizaje

**Navegación y Administración (Módulos 1-3)**
Terminal Linux, sistema de archivos, permisos y procesos.

**Análisis de Registros (Módulos 4-6)**
Inspección de `/var/log/syslog`, `/var/log/auth.log` con `cat`, `grep`, `awk`, `sed`.

**Redes e Inspección (Módulos 7-9)**
TCP/IP, análisis `.pcap` con Wireshark y Tshark para IoCs.

### 3. Flujo de Triaje L1
1. **Ingreso:** Alerta de colector de logs / IDS
2. **Parsing y Enriquecimiento:** IP origen/destino, puerto, hash, usuario
3. **Análisis de Contexto:** Validación rápida
4. **Dictamen:**
   - **Verdadero Positivo (VP):** Malicioso confirmado
   - **Falso Positivo (FP):** Legítimo, documentar y tunear regla

### 4. Matriz de Severidad
| Nivel | Descripción | Acción Recomendada |
| :--- | :--- | :--- |
| **Bajo (Low)** | Escaneos bloqueados, eventos informativos | Monitoreo pasivo |
| **Medio (Medium)** | Múltiples login fallidos, tráfico en puertos raros | Revisar /var/log/ y correlacionar IP |
| **Alto (High)** | Conexión a IP C2, comandos sospechosos | Triaje prioritario, extraer .pcap y contener |
| **Crítico (Critical)** | Movimiento lateral, exfiltración, root no autorizado | Escalar a L2 / IR inmediato |

### 5. Transición a Fase Operativa
Base Linux/Redes documentada. Siguiente fase en suite especializada **Kali Purple**.

**Repositorio Operativo:** https://github.com/gutierrezsebasg/Kali-Purple-SOC-Ops
