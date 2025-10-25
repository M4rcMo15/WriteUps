# Bashed — Writeup

**Resumen rápido**  
Máquina con un endpoint web que incluye `phpbash.php` (una shell en el navegador). Usando esa webshell conseguimos una reverse shell como `www-data`. Luego encontramos un mecanismo que ejecuta periódicamente `test.py` como `root`, lo que nos permitió modificar ese script para añadir el bit SUID a `/bin/bash`. Al ejecutarlo, se obtuvo una shell privilegiada y se recuperó `root.txt`.

---

## Tabla de contenidos
- [Resumen](#resumen)
- [Reconocimiento](#reconocimiento)
- [Enumeración web](#enumeraci%C3%B3n-web)
- [Explotación inicial — phpBash](#explotaci%C3%B3n-inicial---phpbash)
- [Escalada de privilegios](#escalada-de-privilegios)
- [Post-explotación / obtención de flags](#post-explotaci%C3%B3n--obtenci%C3%B3n-de-flags)
- [Cadena de explotación (resumen)](#cadena-de-explotaci%C3%B3n-resumen)
- [Mitigaciones recomendadas](#mitigaciones-recomendadas)
- [Appendix — comandos y scripts usados (sin cambios)](#appendix--comandos-y-scripts-usados-sin-cambios)

---

## Resumen
Accedí a la web y encontré una interfaz `phpbash.php` que permite ejecución de comandos desde el navegador. Tras obtener una reverse shell como `www-data` y enumerar el sistema, descubrí que `www-data` puede ejecutar comandos como `scriptmanager` usando `sudo` sin contraseña. Cambiando al usuario `scriptmanager` encontré un `test.py` que es ejecutado periódicamente por `root` y sobrescribía `test.txt`. Reemplazando `test.py` por un script malicioso que aplica el bit SUID a `/bin/bash`, cuando `root` ejecutó el script se otorgó SUID a `/bin/bash`, lo que permitió obtener una shell privilegiada (`root`). Recuperé `root.txt` en `/root/root.txt`.

---

## Reconocimiento

- Escaneo básico de puertos: puerto **80 (HTTP)** estaba abierto.
- Accediendo al sitio se encontró `phpbash.php`, una webshell en el navegador.

---

## Enumeración web

- Hicimos fuzzing con `gobuster` para descubrir posibles directorios.
- Se encontró un directorio inusual: `/dev` (no típico en una web en producción).
- Accedimos a `http://IP_MAQUINA/dev/phpbash.php` y confirmamos que la shell estaba disponible allí.

---

## Explotación inicial — phpBash

- Accedimos a la shell web `phpbash.php` y, desde ahí, obtuvimos una reverse shell hacia nuestra máquina atacante.  
- La obtención de la reverse shell se usó la webshell para ejecutar un reverse shell y establecer un listener en el atacante.

---

## Escalada de privilegios

- Ya en la máquina como `www-data`, listamos `/etc/passwd` y vimos usuarios como `scriptmanager` y `arrexal`.
- Ejecutando `sudo -l` descubrimos que `www-data` puede llamar comandos como `scriptmanager` y ejecutar acciones de `scriptmanager` sin necesidad de contraseña.

Comando para cambiarse de usuario:
```

sudo -u scriptmanager bash -i

```

- En la raíz observamos una carpeta `scripts` accesible únicamente por `scriptmanager`.  
- Dentro de `scripts` había `test.py` y `test.txt`. `test.py` pertenecía a `scriptmanager`, pero `test.txt` estaba creado por `root`.

Contenido original de `test.py` (tal cual):
```

f = open("test.txt", "w")
f.write("testing 123!")
f.close

````

- Hipótesis: `root` ejecuta periódicamente `test.py` (por ejemplo por cron). Confirmamos que `test.txt` se recrea periódicamente, lo que indica la existencia de una tarea programada que ejecuta `test.py` como `root`.

- Aprovechando esto, sustituimos `test.py` por un script que, cuando `root` lo ejecute, otorgue el bit SUID a `/bin/bash`. El contenido que se puso en `test.py` fue (tal cual):

```python
import os

os.system("chmod u+s /bin/bash")
````

* Esperamos a que el job/cron ejecute `test.py` como `root`. Cuando esto ocurrió, `/bin/bash` quedó con el bit SUID. Entonces, desde `scriptmanager` o el contexto apropiado, ejecutando:

```
bash -p
```

o simplemente invocando `/bin/bash` con el bit SUID permitido, se obtiene una shell con privilegios `root`.

---

## Post-explotación / obtención de flags

* `user.txt` y `root.txt` fueron obtenidos tras la explotación completa.
* **Ruta de `root.txt`:** `/root/root.txt`

---

## Appendix — comandos y scripts usados (sin cambios)

### Cambio de usuario a `scriptmanager`

```
sudo -u scriptmanager bash -i
```

### Contenido original de `test.py`

```
f = open("test.txt", "w")
f.write("testing 123!")
f.close
```

### Contenido modificado de `test.py` (usado para escalar)

```python
import os

os.system("chmod u+s /bin/bash")
```

---

## Notas finales

* Esta máquina ilustra lo peligroso que es ejecutar tareas periódicas (cronjobs) sobre scripts que no están protegidos correctamente: si un script ejecutado por `root` puede ser editado por un usuario menos privilegiado, la escalada a `root` es trivial.
* Revisa cuidadosamente permisos y políticas de `sudo` en servidores productivos.
* Si quieres, lo puedo transformar en un `README.md` listo para subir (archivo descargable), o añadir capturas/evidencias (salidas de comandos, logs). ¿Lo dejo en un archivo para que lo descargues?

```
::contentReference[oaicite:0]{index=0}
```
