# BoardLight — Writeup

**Resumen rápido**  
Escaneé la máquina, identifiqué servicios HTTP y SSH, descubrí un subdominio `crm.board.htb` mediante fuzzing, encontré un login por defecto en una instancia de Dolibarr (v17.0.0), utilicé un PoC público asociado a la CVE-2023-30253 para obtener RCE como `www-data`, encontré credenciales en `htdocs/conf/conf.php` que me permitieron SSHear como `larissa` (obtener `user.txt`) y, finalmente, aproveché un binario `enlightenment` con SUID usando un exploit público que me permitió escalar a `root` y obtener `root.txt`.

---

## Tabla de contenidos
- [Reconocimiento](#reconocimiento)
- [Enumeración y fuzzing](#enumeraci%C3%B3n-y-fuzzing)
- [Acceso inicial — Dolibarr RCE](#acceso-inicial---dolibarr-rce)
- [Obtención de `larissa` (user)](#obtenci%C3%B3n-de-larissa-user)
- [Escalada a `root` (enlightenment SUID)](#escalada-a-root-enlightenment-suid)
- [Cadena de explotación (resumen)](#cadena-de-explotaci%C3%B3n-resumen)
- [Mitigaciones recomendadas](#mitigaciones-recomendadas)
- [Appendix — comandos y herramientas usadas (sin cambios)](#appendix--comandos-y-herramientas-usadas-sin-cambios)

---

## Reconocimiento

Escaneé los puertos TCP para identificar servicios disponibles y obtuve como resultado abierto:
- `22/tcp` (SSH)  
- `80/tcp` (HTTP)

Accedí a la web en el puerto 80; para visualizar correctamente algunos recursos añadí el dominio apropiado en `/etc/hosts` apuntando a la IP de la máquina (p. ej. `board.htb` y luego el `crm.board.htb` cuando lo descubrí).

---

## Enumeración y fuzzing

Probé la web manualmente (formularios, inputs) pero no encontré vectores obvios. Pasé a una fase de reconocimiento activo usando fuzzing de directorios y subdominios para buscar superficies adicionales.

El comando que usé para fuzzear subdominios fue (tal cual):
```

wfuzz -c -t 200 -w /usr/share/SecLists/Discovery/DNS/subdomains-top1million-110000.txt -H "Host: FUZZ.board.htb" [http://board.htb/](http://board.htb/)

```

Obtuve muchas respuestas `200` con líneas repetidas de tamaño 15949 caracteres, por lo que oculté esas líneas con la flag `--hl=517` (hide line) para filtrar ruido y poder ver entradas diferentes. Tras ese filtrado apareció `crm.board.htb`. Añadí `crm.board.htb` en `/etc/hosts` con la misma IP y visité la URL.

En `crm.board.htb` encontré una instancia de **Dolibarr 17.0.0** con una página de login.

Probé credenciales triviales y **admin:admin** funcionó, lo que me dio acceso inicial al panel.

---

## Acceso inicial — Dolibarr RCE

Una vez dentro de Dolibarr (v17.0.0) busqué exploits conocidos para esa versión y encontré un PoC público identificado bajo **CVE-2023-30253**. Descargué el PoC y lo ejecuté (PoC público) para obtener RCE, lo que me dio acceso a la máquina como `www-data`.

> Nota: no incluyo aquí el código del exploit; usé un PoC público encontrado en bases de datos de vulnerabilidades para esta CVE.

---

## Obtención de `larissa` (user)

Con acceso `www-data` busqué credenciales en los ficheros de la web. Encontré un archivo con credenciales en texto claro:
```

/htdocs/conf/conf.php

```
Usé las credenciales halladas para intentar acceso SSH y probé con el usuario `larissa` (identificado en `/etc/passwd`), y efectivamente pude iniciar sesión por SSH como `larissa`. Recuperé `user.txt` desde su home.

---

## Escalada a `root` (enlightenment SUID)

Busqué binarios SUID en el sistema para ver posibles vectores de escalada:
```

find / -perm -4000 -user root 2>/dev/null

```
Entre los resultados encontré varios binarios relacionados con `enlightenment`. Consulté `searchsploit` y localicé un exploit de escalada de privilegios para **enlightenment v0.25.3**; confirmé que el exploit público funcionaba en esta máquina y lo utilicé para escalar a `root`. Finalmente obtuve `root.txt` en `/root/root.txt`.

---

## Cadena de explotación (resumen)

1. Escaneé la máquina → puertos 22 y 80 abiertos.  
2. Navegué la web en 80; añadí dominio a `/etc/hosts`.  
3. Fuzzing de subdominios con `wfuzz` → descubrí `crm.board.htb`.  
4. Accedí a `crm.board.htb` → Dolibarr 17.0.0 → login `admin:admin`.  
5. Encontré PoC para **CVE-2023-30253** y ejecuté exploit → RCE como `www-data`.  
6. Busqué credenciales en ficheros web → `htdocs/conf/conf.php` → credenciales encontradas.  
7. SSH con usuario `larissa` usando credenciales halladas → `user.txt`.  
8. Busqué SUIDs → `enlightenment` binarios → exploit público para versión vulnerable → escalada a `root` → `root.txt`.

---

## Appendix — comandos y herramientas usadas (sin cambios)

### Fuzzing subdominios (wfuzz)
```

wfuzz -c -t 200 -w /usr/share/SecLists/Discovery/DNS/subdomains-top1million-110000.txt -H "Host: FUZZ.board.htb" [http://board.htb/](http://board.htb/)

# Usé --hl=517 para ocultar líneas repetidas y filtrar ruido

```

### Buscar SUIDs
```

find / -perm -4000 -user root 2>/dev/null

```

### Archivos relevantes revisados
- `htdocs/conf/conf.php` — credenciales en texto claro (usadas para SSH).

---

## Notas finales

He documentado mis pasos en primera persona para que el flujo sea claro y reproducible en contexto de laboratorio (Hack The Box). Si quieres, puedo convertir esto en un archivo `README.md` descargable listo para tu repositorio, o añadir capturas de pantalla y salidas de comandos como evidencia — pásamelas y las incluyo.  

