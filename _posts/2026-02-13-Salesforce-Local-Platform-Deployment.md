---
layout: single
title: 📦 Salesforce: Local Platform Deployment
excerpt: "El objetivo de esta Prueba de Concepto es establecer un modelo de despliegue local estandarizado utilizando exclusivamente herramientas de Salesforce DX (Developer Experience) y Git local.

En este modelo, la Org de Salesforce se mantiene como la "Fuente de la Verdad", pero el desarrollador utiliza su entorno local como "puente de control". Este enfoque permit:"
date: 2026-02-13
classes: wide
header:
  teaser:
  teaser_home_page: true
  icon:
categories:
  - Salesforce
  - DevOps
tags:  
  - DevOps
  - SF
  - SFDX
  - Salesforce
  - Visual Studio Code 
---

# 📦 Salesforce: Local Platform Deployment

## Introducción
Actualmente, un proyecto puede carecer de una infraestructura de **DevOps** automatizada (**CI/CD**) y repositorios remotos. Esto genera riesgos de sobreescritura de código, pérdida de cambios y una falta de visibilidad sobre lo que se despliega entre los ambientes de **Desarrollo**, **Integración**, **UAT**, etc.

## Propósito
El objetivo de esta **Prueba de Concepto** es establecer un **modelo de despliegue local estandarizado** utilizando exclusivamente herramientas de **Salesforce DX** (**Developer Experience**) y **Git** local.

En este modelo, la **Org de Salesforce se mantiene como la "Fuente de la Verdad"**, pero el desarrollador utiliza su entorno local como "puente de control". Este enfoque permite:

- **Sincronizar metadatos** de manera precisa entre ambientes.
- **Gestionar versiones localmente** para comparar cambios y revertir errores antes de afectar la **Org** de destino.
- **Resolver conflictos de código** de forma visual y amigable antes de cada despliegue.

## Alcance
La **PoC** documentará los pasos técnicos para **extraer metadatos** desde una **Org origen**, realizar un **control de cambios** mediante commits de **Git** locales y **ejecutar el despliegue** hacia una **Org destino** de forma **segura y validada**, todo desde la **estación de trabajo del desarrollador**.

## Contenido
- [Pre-Work](pre-work)
- [Manifest](#manifest)
- [Commit & Recursive Merge & Deployment](#commit--recursive-merge--deployment)
- [Puntos Claves](#puntos-claves)

## ⚙️ Pre-Work

### Salesforce Developer Experience (DX)
Para ejecutar el flujo de la PoC, cada estación de trabajo debe contar con el siguiente stack de herramientas configurado.

#### Salesforce CLI (sf v2)
Es el motor que permite la comunicación entre tu computadora y las Orgs de Salesforce.

- **Instalar Salesforce CLI**
- **Verificación:** Ejecuta `sf --version` en la terminal. Debe mostrar la versión 2.x o superior.

#### Visual Studio Code + Salesforce Extension Pack
El IDE oficial para el desarrollo en la plataforma.

- [Instalar Visual studio code](https://code.visualstudio.com/) 
  - Instalar Extensiones de visual studio code
    - [Salesforce Extension Pack (Expanded)](https://marketplace.visualstudio.com/items?itemName=salesforce.salesforcedx-vscode-expanded)
    - [Salesforce Package.xml Generator Extension for VS Code](https://marketplace.visualstudio.com/items?itemName=modicatech.apex-code-coverage-visualizer)
    - [Apex Code Coverage Visualizer](https://marketplace.visualstudio.com/items?itemName=modicatech.apex-code-coverage-visualizer)
    - [Apex PMD](https://marketplace.visualstudio.com/items?itemName=chuckjonas.apex-pmd)
    - [ESLint](https://marketplace.visualstudio.com/items?itemName=dbaeumer.vscode-eslint)
    - [Git History](https://marketplace.visualstudio.com/items?itemName=donjayamanne.githistory)

#### Salesforce Code Analyzer (Opcional)
Para asegurar que el código extraído de la Org cumple con las mejores prácticas antes de ser movido a Integración:

- **Instalar Salesforce Code Analyzer**
    ```
    sf update
    sf plugins --core
    sf plugins install code-analyzer
    sf plugins install @salesforce/sfdx-scanner
    ```
  
- **Verificación:**
    ```
    sf code-analyzer rules
    sf scanner rule list
    ```

### Source Control System
Aunque no tengamos un servidor remoto (como GitHub), usaremos **Git** localmente para crear "puntos de control" y comparar archivos.

- [Instalar GIT SCM](https://git-scm.com/install/)
- **Configuración mínima:**
    ```
    git config --global user.name "Tu Nombre"
    git config --global user.email "tu@email.com"
    git config list
    ```

## Crear Playgrounds
Como estamos en un enfoque de **Org-Centric**, utilizaremos las **Trailhead Playgrounds** o **Developer Editions** gratuitas, que funcionan exactamente igual que un entorno de producción o sandbox para fines de despliegue.

Tienes dos opciones rápidas:

1. **Recomendada:** ir a [trailhead](https://trailhead.salesforce.com/) > Hands-On Orgs > Click en Create Playground
   - Crear dos playground llamadas **Developer** e **Integration**.
   
2. Ve a [developer.salesforce.com](https://developer.salesforce.com) y crea dos cuentas con correos diferentes (ej: tu_nombre+dev@gmail.com y tu_nombre+int@gmail.com).

### Identificación de Credenciales
Asegúrate de tener el **Username** y **Password** de ambas. En Playgrounds de Trailhead, puedes obtener la contraseña usando el botón **"Get Your Login Credentials**" dentro de la aplicación **"Playground Starter".**

### Vinculación al Entorno Local (CLI Authentication)

#### **Para el Ambiente de Origen (Developer):**

`sf org login web --alias dev_org --instance-url https://MyDomainDeveloperName.my.salesforce.com --set-default`

*Se abrirá el navegador. Ingresa las credenciales de tu primera Org.*

#### **Para el Ambiente de Destino (Integration):**

`sf org login web --alias int_org --instance-url https://MyDomainIntegrationName.my.salesforce.com`

*Ingresa las credenciales de tu segunda Org.*

**Para listar las organizaciones autenticadas:** ``sf org list``


En un entorno corporativo real, la **dev\_org** sería tu **Developer Sandbox** y la **int\_org** sería la **Partial** o **Full Copy Sandbox**. Para esta PoC, las Playgrounds simularán este comportamiento perfectamente, permitiéndote mover metadatos de una a otra sin restricciones.

***En caso de tener errores visitar => [` `***Resolve Common Authorization Errors***](https://developer.salesforce.com/docs/atlas.en-us.sfdx_dev.meta/sfdx_dev/sfdx_dev_troubleshoot_auth_errors.htm)***


## Preparación del Proyecto Local
Antes de iniciar el flujo, el desarrollador debe preparar su área de trabajo una sola vez.

Utilizando el formato de fuente estándar de Salesforce DX, pero basado en un archivo package.xml (manifiesto) para tener control total sobre qué metadatos se mueven.

### Crear el proyecto DX (Estructura de manifiesto)
Abra una terminal en su carpeta de trabajo y ejecute:

```
sf project generate --name LocalPlatformDeployment --template empty --manifest
code LocalPlatformDeployment
```
*Se abrirá la carpeta del proyecto en el **VSC** con los directorios por defecto, Esto creará la carpeta **manifest/package.xm**l donde listamos los componentes a trabajar.*

#### **Configurar el .forceignore:**
Asegúrese de que el archivo .forceignore incluya archivos que no deben viajar a la Org (como configuraciones de VS Code o archivos del sistema). [Ejemplo](https://github.com/trailheadapps/apex-recipes/blob/main/.forceignore)

#### **Configurar el .gitignore**
En caso de que no haya sido creado automáticamente, entonces crear el archivo “.gitignore” en la raíz del directorio. [Ejemplo](https://github.com/trailheadapps/apex-recipes/blob/main/.gitignore)

#### **Crear el árbol de ramas (Pipeline Local)**
Dentro de la carpeta del proyecto:

```
git init
git add .
git commit -m "Estructura inicial del proyecto DX"
git branch integration
git branch developer
```

### Vinculación Inteligente (Git Hooks)
Para evitar hacer **retrieve,** **deploy, etc** en un ambiente incorrecto, implementaremos un "**conmutador**" automático. Para que al cambiar de branch el ***target-org*** del archivo **.sf/config.json** apunte automáticamente a la ORG correspondiente.

**Crear el automatizador, en la terminar del projecto ejecuta:**

`code .git/hooks/post-checkout`

Se abrirá el archivo post-checkout

Paga este código que vincula la branch con la Org:
```
#!/bin/bash

BRANCH=$(git rev-parse --abbrev-ref HEAD)

if [ "$BRANCH" == "developer" ]; then

`    `sf config set target-org dev_org

`    `echo "🔗 Contexto: DEV_ORG activo"

elif [ "$BRANCH" == "integration" ]; then

`    `sf config set target-org int_org

`    `echo "🔗 Contexto: INT_ORG activo"

else

echo -e "==== ERROR ASIGNANDO ORG ===="

fi
```

*A partir de este momento el desarrollador solo debe preocuparse de navegar entre las Branchs.*



## 📝 Manifest
No es el [Manifiesto Hacker](https://phrack.org/issues/7/3). El package.xml no es solo un archivo de descarga, es el alcance del despliegue partiendo del ambiente de origen(developer)

### Generar el package.xml (El alcance)
En un flujo profesional, no movemos "todo" sino sólo lo que hemos construido. El **package.xml** define este alcance.

#### **Identificar qué metadatos existen**
Antes de crear el archivo, el desarrollador debe saber qué **[**tipos de metadatos**](https://developer.salesforce.com/docs/atlas.en-us.api_meta.meta/api_meta/meta_types_list.htm)** están disponibles en la Org de origen.

`sf org list metadata-types`

Esta instruccion retorna una lista de tipos (ej. **ApexClass**, **CustomField**, **Layout**). Esto te confirma cómo escribir el nombre del tipo correctamente en el XML

El desarrollador al saber que componentes creo o modifico, genera el manifiesto directamente haciendo uso del comando [**project generate manifest**](https://developer.salesforce.com/docs/atlas.en-us.sfdx_cli_reference.meta/sfdx_cli_reference/cli_reference_project_commands_unified.htm#cli_reference_project_generate_manifest_unified)

`sf project generate manifest --metadata MetadataType:Components --name package --output-dir ./manifest`

**En nuestro ejemplo:** 

`sf project generate manifest --metadata PermissionSet:Contact_Management_System_End_User --name package --output-dir ./manifest`


*O por lo **contrario** el desarrollador puede usar el plugin llamado [*Salesforce Package.xml Generator Extension for VS Code*](https://marketplace.visualstudio.com/items?itemName=modicatech.apex-code-coverage-visualizer). para generarlo gráficamente.*

## 🌳 Commit & Recursive Merge & Deployment
Este es el proceso **cíclico** para mover cambios de un ambiente a otro, garantizando que el desarrollador siempre tenga un respaldo antes de aplicar cambios.

## Extracción (Origen vs Destino)

**Cambiar** al Branch **developer** ORG y **hacer** retrieve.
```
git checkout developer
sf project retrieve start -x manifest\package.xml
```
**Crear** commit de desarrollo
```
git add .
git commit -m "cambios listos para integración"
```
*Antes de **cambiar** de branch. **Copiar** el contenido del `Package.xml`*

**Cambiar** al Branch **integration** ORG

```
git checkout integration
```
**Crucial**: Antes de mover algo, **descarga** el estado actual de la **Org de destino** para generar un **backup**

***Pegar** el contenido del **mismo** `Package.xml`*
```
sf project retrieve start -x manifest\package.xml
git add .
git commit -m "backup: estado previo de Integración antes del merge"
```

**Ejecutar** el merge y resolver los conflictos. Aquí es donde este **proceso local** brilla sobre los **Change Sets**. Al ejecutar:
```
git merge --no-commit --no-ff developer
```
**Resolución Visual:** VS Code marcará en rojo/azul las colisiones (ej. si otro desarrollador modificó el mismo Layout en Integración). Puedes elegir "Accept Both" para fusionar cambios de ambos sin borrar el trabajo ajeno.

## Validación y Despliegue (Destino)
Una vez resuelto los conflictos en local, la carpeta `force-app` contiene el "**Código Final Fusionado".**  la **"Versión Única de la Verdad**". El objetivo ahora es asegurar que esta versión sea compatible con la Org de destino sin romper funcionalidades existentes.

### Validación Técnica (Dry-Run)
Antes de afectar el **ambiente destino(integration)**, el desarrollador debera ejecutar una **validación**. Usaremos el flag `--dry-run` para **simular el despliegue** y `-l RunSpecifiedTests` para garantizar que la cobertura de código es correcta.

**Validar** en la ORG de destino:

Por buenas prácticas siempre es recomendado correr los test específicos de las clases o de lo contrario los test bases de la organizacion.

```
sf project deploy start -x manifest\package.xml -l RunSpecifiedTests -t "UtilTest" --dry-run
```

*Al usar validate, Salesforce genera un **Job ID**. Si la validación es exitosa, ese ID permite realizar un Quick Deploy (despliegue rápido) sin volver a ejecutar los tests.*

### Monitoreo y Quick Deploy
**Monitorear** el progreso de la **validación** desde la **Org** destino: 

- Setup > **Deployment Status >** Dar click sobre **View Details** y luego sobre “**Quick Deploy”**

**Hacer commit del merge**
Una vez que el **despliegue es confirmado como exitoso** en la Org de destino, el desarrollador procede a cerrar el **ciclo en el control de versiones loca**l. Esto asegura que lo que está en la org de destino coincida con la estación local del desarrollador.

```
git commit -m " Instalado en integración"
```
