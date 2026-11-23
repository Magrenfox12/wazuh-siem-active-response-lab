# Wazuh SIEM, Sysmon & Active Response Lab

Laboratorio práctico de monitoreo de seguridad (Blue Team / SOC) desplegado con **Wazuh SIEM** para la auditoría de integridad de archivos, telemetría avanzada en endpoints Windows y mitigación automatizada de amenazas.

---

## Arquitectura del Entorno

* **SIEM / Manager:** Wazuh Server (`192.168.100.12`)
* **Endpoint Monitoreado:** Agente Windows (`magrenfoxx12` / `192.168.100.5`)
* **Telemetría:** Microsoft Sysmon integrado con el agente de Wazuh y mapeo MITRE ATT&CK.

---

## Pruebas Realizadas y Evidencia

### 1. File Integrity Monitoring (FIM)
Configuración del módulo `syscheck` en `ossec.conf` para vigilar cambios en el directorio crítico `c:\monitoreo_cv\confidencial.txt`, generando alertas de nivel 7 ante modificaciones de contenido o atributos.
* **Regla disparada:** `550` (*Integrity checksum changed*).

![Evidencia FIM](Captura%20de%20pantalla%202026-11-22%20212946.png)

---

### 2. Threat Hunting y Telemetría con Sysmon
Integración de logs de Sysmon para detectar ejecución anómala de intérpretes de comandos y actividad de reconocimiento interno (*Discovery*).
* **Reglas disparadas:**
  * `92052` – *Windows command prompt started by an abnormal process*.
  * `92032` – *Suspicious Windows cmd shell execution*.
  * `92031` – *Discovery activity executed*.
  * `92036` – *net.exe binary was started by a Windows cmd shell*.

![Alerta Inicial Sysmon](Captura%20de%20pantalla%202026-11-22%20213646.png)

![Cadena de Threat Hunting](Captura%20de%20pantalla%202026-11-22%20213217.png)

---

### 3. Active Response (Respuesta Automatizada)
Configuración de `wazuh-execd` para bloquear automáticamente direcciones IP mediante reglas en el Firewall de Windows (`netsh.exe`) tras detectar intentos repetidos de acceso no autorizado.
* **Acción verificada:** Ejecución del comando `add` por `wazuh-execd` en `magrenfoxx12` (*Host Blocked*).

![Evidencia Active Response](Captura%20de%20pantalla%202026-11-22%20213307.png)

---

## Notas Técnicas, Retos y Aprendizajes

* **Troubleshooting de red, acceso y API:** Al inicio enfrenté problemas de conectividad y DNS al intentar administrar el servidor vía SSH desde Windows, además de un bloqueo de credenciales en el acceso web que resolví restableciendo las contraseñas desde la terminal del servidor. También solucioné un error de conexión con la API de Wazuh (*API connection / version mismatch*) actualizando y reiniciando los servicios correspondientes desde consola.
* **Comportamiento del Active Response en pruebas:** Durante la simulación de fuerza bruta pude comprobar la sensibilidad de las reglas de respuesta activa: cualquier error de tecleo repetido disparaba de inmediato el bloqueo automático de la IP, demostrando la importancia de ajustar umbalas y listas blancas en entornos reales.
* **Visibilidad y triaje de alertas (SOC):** Comprendí que Wazuh va mucho más allá del monitoreo básico, integrando capacidades de Threat Hunting y detección de malware. Aprendí a interpretar y priorizar eventos según su nivel de severidad (baja, media y crítica) para distinguir actividad rutinaria o sospechosa de incidentes críticos que requieren contención inmediata.
