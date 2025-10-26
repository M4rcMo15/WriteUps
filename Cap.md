# Cap — Writeup

**Resumen rápido**  
Escaneé la máquina y encontré servicios FTP, SSH y HTTP. En la web descubrí una funcionalidad para generar y descargar capturas de tráfico (pcaps). Aproveché un problema de control de acceso (IDOR) en ese servicio para descargar una pcap con tráfico de otro usuario; analizando la pcap encontré credenciales FTP en texto claro, con las que accedí a la máquina como `nathan`. Después de la fase de userland, identifiqué una capability inusual asignada a `python3.8` (`cap_setuid`), la cual me permitió, mediante un script de python, obtener privilegios más altos y leer `root.txt`.

---

## Tabla de contenidos
- [Reconocimiento](#reconocimiento)  
- [Enumeración y descubrimiento de IDOR / PCAPs](#enumeraci%C3%B3n-y-descubrimiento-de-idor--pcaps)  
- [Análisis de la pcap y acceso como `nathan`](#an%C3%A1lisis-de-la-pcap-y-acceso-como-nathan)  
- [Escalada de privilegios (resumen de alto nivel)](#escalada-de-privilegios-resumen-de-alto-nivel)  
- [Cadena de explotación (resumen)](#cadena-de-explotaci%C3%B3n-resumen)  
- [Mitigaciones y detección](#mitigaciones-y-detecci%C3%B3n)  
- [Appendix — comandos de enumeración (sin cambios peligrosos)](#appendix--comandos-de-enumeraci%C3%B3n-sin-cambios-peligrosos)

---

## Reconocimiento

Escaneé la máquina para descubrir servicios TCP abiertos:

- Ejecuté un scan rápido y detecté los puertos **21 (FTP)**, **22 (SSH)** y **80 (HTTP)** abiertos.  
- Accedí al servicio web en puerto 80 y encontré una interfaz administrativa orientada a seguridad/monitorización que permitía generar descargas de capturas de tráfico (pcaps).

---

## Enumeración y descubrimiento de IDOR / PCAPs

- Desde la interfaz web usé la funcionalidad de generación de capturas y observé que las capturas se indexaban en rutas del tipo `/data/<id>` (por ejemplo `/data/1`, `/data/2`).  
- Por pruebas de IDOR validé que era posible solicitar IDs arbitrarios; al comprobar `/data/0` obtuve una pcap que no correspondía a mi tráfico y que contenía información valiosa.  
- Descargué la pcap para su análisis local con Wireshark/tshark.

**Comandos útiles (uso documental):**
```bash
# Ejemplo de scan
nmap -sC -sV -p 21,22,80 <IP_OBJETIVO>

# Descargar pcap via HTTP (ejemplo)
wget http://<IP_OBJETIVO>/data/0 -O capture_0.pcap
````

---

## Análisis de la pcap y acceso como `nathan`

* Abrí la pcap en Wireshark y filtré por `ftp` / `ftp.request.command == "USER"` / `ftp.request.command == "PASS"` para localizar credenciales en texto claro.
* Encontré credenciales FTP para el usuario `nathan`. Con dichas credenciales pude:

  * Conectarme al servicio FTP y descargar `user.txt`.
  * Usar las mismas credenciales para autenticarme por SSH (si el servicio lo permitía), obteniendo acceso interactivo como `nathan`.

**Comandos útiles (documentales):**

```bash
# FTP (ejemplo interactivo)
ftp <IP_OBJETIVO>
# luego: USER nathan
#       PASS <contraseña encontrada>

# SSH (ejemplo)
ssh nathan@<IP_OBJETIVO>
```

---

## Escalada de privilegios (resumen de alto nivel)

* Para la fase de post-explotación busqué vectores típicos:

  * Listé binarios SUID (`find / -perm -4000 -user root 2>/dev/null`) para ver si había utilidades explotables.
  * Listé capabilities en el sistema (`getcap -r / 2>/dev/null`) para detectar binarios con capacidades inusuales.
* Observé que `/usr/bin/python3.8` (el intérprete Python) tenía una capability inusual asignada (`cap_setuid`). Esto representa un riesgo crítico si un intérprete general tiene capacidades que permiten cambiar UID, porque un intérprete permite ejecutar código arbitrario.
* Mediante una técnica avanzada de post-explotación aproveché la presencia de esa capability para obtener privilegios más altos y finalmente leer `root.txt`.

* script de python utilizado para cambiar la /bin/bash y añadirle el bit suid

```python
import os
os.setuid(0)
os.system(chmod u+s /bin/bash)
```
Tras esto sólo queda lanzar bash -p y obtenemos una shell con privilegios máximos

---

## Cadena de explotación (resumen)

1. Reconocimiento (nmap) → 21, 22, 80.
2. Identificación de funcionalidad de generación de pcaps en la web.
3. Abuso de un control de acceso (IDOR) en `/data/<id>` → descarga de una pcap ajena.
4. Análisis de la pcap → credenciales FTP en texto claro para `nathan`.
5. Acceso FTP/SSH con `nathan` → obtención de `user.txt`.
6. Enumeración local con `find` / `getcap` → detección de `cap_setuid` en `python3.8`.
7. Explotación para escalada a `root`.
8. Lectura de `root.txt`.

---

## Appendix — comandos de enumeración

> Los siguientes comandos se usan para enumeración/diagnóstico y documentación.

```bash
# Escaneo inicial
nmap -sC -sV -p 21,22,80 <IP_OBJETIVO>

# Descargar pcap demonstrativo desde la web
wget http://<IP_OBJETIVO>/data/0 -O capture_0.pcap

# Abrir la pcap en local (Wireshark/tshark)
wireshark capture_0.pcap
# o (filtrado con tshark)
tshark -r capture_0.pcap -Y ftp -V

# Buscar SUIDs (enumeración)
sudo find / -perm -4000 -user root 2>/dev/null

# Listar capabilities en el sistema (enumeración)
sudo getcap -r / 2>/dev/null

# Acceder por FTP/SSH (uso legítimo con credenciales encontradas)
# ftp <IP_OBJETIVO>
# ssh nathan@<IP_OBJETIVO>
```
```python
# Script para escalar privilegios explotando el set_uid en python3.8
import os
os.setuid(0)
os.system(chmod u+s /bin/bash)
```
