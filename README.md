# Wazuh SIEM, Sysmon & Active Response Lab

Laboratorio práctico de monitoreo de seguridad (Blue Team / SOC) desplegado con **Wazuh SIEM** para la auditoría de integridad de archivos, telemetría avanzada en endpoints Windows y mitigación automatizada de amenazas.

---

## Arquitectura del Entorno

- **SIEM / Manager:** Wazuh Server `v4.14.9 (rc2)` (`192.168.100.12`)
- **Endpoint Monitoreado:** Agente Windows 11 (`magrenfoxx12` / `192.168.100.5`)
- **Telemetría:** Microsoft Sysmon `v15.22` integrado con el agente de Wazuh y mapeo MITRE ATT&CK.

---

## Configuraciones Clave en `ossec.conf`

Para reproducir las detecciones y la mitigación automatizada en el laboratorio, se aplicaron las siguientes configuraciones:

**1. Monitoreo de Integridad de Archivos (`syscheck` en agente Windows):**
```xml
<syscheck>
  <disabled>no</disabled>
  <directories check_all="yes" realtime="yes" report_changes="yes">c:\monitoreo_cv\confidencial.txt</directories>
</syscheck>
```

**2. Recolección de Telemetría Sysmon (`localfile` en agente Windows):**
```xml
<localfile>
  <location>Microsoft-Windows-Sysmon/Operational</location>
  <log_format>eventchannel</log_format>
</localfile>
```

**3. Respuesta Activa Automatizada (`active-response` en Wazuh Manager):**
```xml
<command>
  <name>win_route-null</name>
  <executable>route-null.exe</executable>
  <timeout_allowed>yes</timeout_allowed>
</command>

<active-response>
  <command>win_route-null</command>
  <location>local</location>
  <level>10</level>
  <timeout>600</timeout>
</active-response>
```

---

## Pruebas Realizadas y Evidencia (Investigación de Incidente)

### 1. File Integrity Monitoring (FIM)
- **Qué pasó:** Se modificó el contenido del archivo vigilado `c:\monitoreo_cv\confidencial.txt`, disparando una alerta en tiempo real de nivel 7 por alteración de integridad.
- **Qué hice:** Analicé el evento en el módulo FIM para verificar la marca de tiempo, el usuario responsable y la variación de los hashes (`MD5`, `SHA1`, `SHA256`) antes y después del cambio.
- **Qué concluí:** El monitoreo en tiempo real con `syscheck` permite detectar manipulaciones no autorizadas de archivos críticos de forma inmediata y conservar evidencia forense exacta.
- **Regla disparada:** `550` (*Integrity checksum changed*).

![Evidencia FIM](Captura%20de%20pantalla%202026-11-22%20212946.png)

---

### 2. Threat Hunting y Telemetría con Sysmon
- **Qué pasó:** Tras simular acceso al endpoint, se ejecutaron comandos de reconocimiento interno (`cmd.exe` invocando `net.exe`) para enumerar usuarios y configuración de red, siendo capturados por Sysmon `v15.22`.
- **Qué hice:** Filtré los eventos en *Threat Hunting* por el agente `magrenfoxx12` y las reglas disparadas, inspeccionando el proceso padre y los argumentos de línea de comandos para confirmar la actividad de *Discovery*.
- **Qué concluí:** La integración de Sysmon aporta visibilidad detallada sobre técnicas MITRE ATT&CK de reconocimiento interno que los registros nativos de Windows no logran evidenciar por sí solos.
- **Reglas disparadas:**
  - `92052` – *Windows command prompt started by an abnormal process*.
  - `92032` – *Suspicious Windows cmd shell execution*.
  - `92031` – *Discovery activity executed*.
  - `92036` – *net.exe binary was started by a Windows cmd shell*.

![Alerta Inicial Sysmon](Captura%20de%20pantalla%202026-11-22%20213646.png)

![Cadena de Threat Hunting](Captura%20de%20pantalla%202026-11-22%20213217.png)

---

### 3. Active Response (Respuesta Automatizada)
- **Qué pasó:** Se simularon múltiples intentos repetidos de acceso no autorizado contra el endpoint, superando los umbrales configurados en las reglas de detección de fuerza bruta.
- **Qué hice:** Verifiqué la ejecución automática de `wazuh-execd` y confirmé que se aplicara la regla de bloqueo en el Firewall de Windows mediante `netsh.exe`.
- **Qué concluí:** La respuesta automatizada reduce el tiempo de contención a segundos frente a ataques de fuerza bruta; en producción es fundamental calibrar bien los umbrales y definir listas blancas para evitar bloqueos accidentales a usuarios legítimos.
- **Acción verificada:** Ejecución del comando `add` por `wazuh-execd` en `magrenfoxx12` (*Host Blocked*).

![Evidencia Active Response](Captura%20de%20pantalla%202026-11-22%20213307.png)

---

## Notas Técnicas, Retos y Aprendizajes

- **Troubleshooting de red, acceso y API:** Al inicio enfrenté problemas de conectividad y DNS al intentar administrar el servidor vía SSH desde Windows, además de un bloqueo de credenciales en el acceso web que resolví restableciendo las contraseñas desde la terminal del servidor. También solucioné un error de conexión con la API de Wazuh (*API connection / version mismatch*) actualizando y reiniciando los servicios correspondientes desde consola.
