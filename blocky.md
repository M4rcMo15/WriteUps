# Blocky — Writeup

**Resumen rápido**  
Escaneé la máquina y encontré servicios FTP, SSH, HTTP y Minecraft. La web en `/` resultó ser un WordPress 4.8; mediante fuzzing descubrí directorios `uploads` y `/plugins` que contenían dos JARs (plugins de Minecraft). Inspeccioné los JARs con `jd-gui`, hallé una contraseña en texto claro para la base de datos y la probé contra `ssh` para el usuario `notch`. La contraseña funcionó, me logueé como `notch` y comprobé que `notch` pertenece al grupo `sudo`, por lo que hice `sudo -i` para obtener ROOT. Máquina completada.

---

## Tabla de contenidos
- [Resumen](#resumen)
- [Reconocimiento](#reconocimiento)
- [Enumeración web y descubrimiento de plugins](#enumeración-web-y-descubrimiento-de-plugins)
- [Análisis de los JARs y credenciales](#análisis-de-los-jars-y-credenciales)
- [Acceso inicial — SSH como `notch`](#acceso-inicial---ssh-como-notch)
- [Escalada a `root`](#escalada-a-root)
- [Cadena de explotación (resumen)](#cadena-de-explotación-resumen)
- [Mitigaciones recomendadas](#mitigaciones-recomendadas)
- [Appendix — comandos y herramientas usadas (ejemplos)](#appendix--comandos-y-herramientas-usadas-ejemplos)

---

## Resumen
En esta caja me centré en la superficie web y los artefactos del servidor de Minecraft. La clave fue encontrar los plugins (.jar) expuestos en la web, inspeccionarlos para recuperar credenciales en claro, y reutilizar esas credenciales para acceder por SSH. El usuario obtenido tenía permisos `sudo`, por lo que la escalada fue directa.

---

## Reconocimiento

- Escaneé la máquina para identificar servicios TCP relevantes. Tras el `scan` encontré:
  - **21/tcp (FTP)**  
  - **22/tcp (SSH)**  
  - **80/tcp (HTTP)**  
  - **25565/tcp (Minecraft)** — puerto típico de servidores Minecraft.

- Intenté acceso FTP sin éxito (no encontré credenciales útiles ni anon login interesante).

- Navegué al servicio HTTP (puerto 80). La web parecía ser un sitio de comunidad/servidor de Minecraft y **resultó ser un WordPress**. Al inspeccionar el HTML pude identificar que la versión es **WordPress 4.8** por referencias en el código fuente.

---

## Enumeración web y descubrimiento de plugins

- Ejecuté un escaneo específico de WordPress (`wpscan`) y no encontré plugins públicos o vectores obvios porque la instalación no expuso `wp-plugins` típicos o las rutas esperadas.
- Hice fuzzing de directorios con `wfuzz` / `gobuster` y encontré:
  - Un directorio `uploads` (esperable en WordPress).
  - Un directorio `/plugins` que **no** parecía ser el típico `wp-content/plugins` de WordPress, sino un repo público de artefactos del servidor.

- En `/plugins` encontré **dos archivos `.jar`** — plugins del servidor de Minecraft que el sitio ponía a disposición para descarga.

---

## Análisis de los JARs y credenciales

- Descargué los JARs y los abrí con `jd-gui` para inspeccionar el código Java (decompilado).  
- Revisando el código fuente de los plugins encontré **una contraseña en texto plano** relacionada con la base de datos o con el propio servidor (cadena clara codificada en el código).
- Probé la contraseña en varios servicios: intenté con `phpMyAdmin` y la base de datos pero no obtuve cosas útiles desde ahí.  
- Finalmente probé la **misma contraseña contra SSH** para el usuario que había visto en la web (`notch`) y **funcionó**.

> Nota: la presencia de credenciales en texto claro dentro de artefactos descargables (plugins, jars, zip, etc.) es un fallo clásico de seguridad — siempre revisar binarios/artefactos que el servidor publiques.

---

## Acceso inicial — SSH como `notch`

- Me conecté por SSH:
```bash
ssh notch@IP_OBJETIVO
# (usé la contraseña encontrada en el JAR)
````

* Tras autenticación, estuve dentro como `notch`. Verifiqué información del usuario y grupos:

```bash
id
groups
```

* En los grupos aparecía `sudo` (o `wheel`/grupo con privilegios sudo), lo que indicaba que `notch` podía ejecutar comandos con `sudo`.

---

## Escalada a `root`

* Dado que `notch` tenía permisos para usar `sudo`, solicité una shell privilegiada:

```bash
sudo -i
# o
sudo su -
```

* Introduje la contraseña de `notch` cuando fue solicitada y obtuve `root`.

* Recuperé la `root.txt`.

---

## Cadena de explotación (resumen)

1. Escaneé la máquina → puertos 21, 22, 80, 25565.
2. Accedí a la web en el puerto 80 → sitio WordPress 4.8.
3. Hice fuzzing de directorios → descubrí `/uploads` y `/plugins`.
4. Descargué JARs desde `/plugins` y los inspeccioné con `jd-gui`.
5. Encontré una contraseña en texto claro en el código del plugin.
6. Probé la contraseña contra SSH y obtuve acceso como `notch`.
7. `notch` pertenecía al grupo sudo → ejecuté `sudo -i` y me hice `root`.

---

## Appendix — comandos y herramientas usadas (ejemplos)

> Herramientas principales utilizadas: `nmap`, `wpscan`, `gobuster`/`wfuzz`, navegador/inspección de HTML, `jd-gui` para JARs, `ssh`, `sudo`.

### Escaneo inicial (ejemplo)

```bash
# nmap scan rápido de puertos comunes
nmap -sC -sV -p- IP_OBJETIVO
```

### WordPress enumeration (ejemplo)

```bash
wpscan --url http://IP_OBJETIVO --enumerate u,vp,ap
```

### Fuzzing (ejemplo)

```bash
gobuster dir -u http://IP_OBJETIVO -w /usr/share/seclists/Discovery/Web-Content/raft-small-directories.txt -t 50
# o wfuzz para fuzz más flexible
wfuzz -c -z file,/path/to/wordlist --hc 404 http://IP_OBJETIVO/FUZZ
```

### Inspección de JARs

* Descargar los JARs desde la web (browser o `wget`) y abrirlos con `jd-gui` para revisar el código Java decompilado.

### SSH (ejemplo)

```bash
ssh notch@IP_OBJETIVO
# introducir contraseña encontrada
```

### Comprobación de grupos / escalada

```bash
id
groups
sudo -l
sudo -i
```

---

## Notas finales

* Blocky es un buen recordatorio de que **los secretos en artefactos descargables** son una fuente rápida de compromiso: revisar y evitar credenciales embebidas es crucial.
* Además muestra la importancia de aplicar el principio de mínimos privilegios sobre cuentas de usuario (no añadir usuarios irrelevantes al grupo `sudo`).
* Si quieres, puedo:

  * Formatear esto como `README.md` listo para subir (archivo descargable).
  * Añadir capturas de pantalla o salidas de comandos concretas si me pasas las evidencias que quieras incluir.

```
::contentReference[oaicite:0]{index=0}
```
