---
layout: single
title: Local Platform Deployment - Salesforce
excerpt: "Actualmente, un proyecto puede carecer de una infraestructura de DevOps automatizada (CI/CD) y repositorios remotos. Esto genera riesgos de sobreescritura de código, pérdida de cambios y una falta de visibilidad sobre lo que se despliega entre los ambientes de Desarrollo. integración, UAT, etc."
date: 2026-02-13
classes: wide
header:
  teaser: /assets/images/sfdc-local-platform-deployment/salesforce-dx.webp
  teaser_home_page: true
  icon: /assets/images/salesforce.png
categories:
  - salesforce
  - devops
tags:
  - sf
  - visual studio code 
  - git
  - salesforce extension pack
  - salesforce platform
  - salesforce dx
  - salesforce code analyze
  - apex pmd
  - eslint
---

Actualmente, un proyecto puede carecer de una infraestructura de **DevOps** automatizada (**CI/CD**) y repositorios remotos. Esto genera riesgos de sobreescritura de código, pérdida de cambios y una falta de visibilidad sobre lo que se despliega entre los ambientes de **Desarrollo**, **Integración**, **UAT**, etc.

El objetivo de esta **PoC** es establecer un **modelo de despliegue local estandarizado** utilizando exclusivamente herramientas de **Salesforce DX** (**Developer Experience**) y **Git** local.

En este modelo, la **Org de Salesforce se mantiene como la "Fuente de la Verdad"**, pero el desarrollador utiliza su entorno local como "puente de control". Este enfoque permite:

- **Sincronizar metadatos** de manera precisa entre ambientes.
- **Gestionar versiones localmente** para comparar cambios y revertir errores antes de afectar la **Org** de destino.
- **Resolver conflictos de código** de forma visual y amigable antes de cada despliegue.

La **PoC** documentará los pasos técnicos para **extraer metadatos** desde una **Org origen**, realizar un **control de cambios** mediante commits de **Git** locales y **ejecutar el despliegue** hacia una **Org destino** de forma **segura y validada**, todo desde la **estación de trabajo del desarrollador**.

# Contenido
- [1. Pre-Work](#1-pre-work)
    - [1.2 Salesforce Developer Experience (DX)](#12-salesforce-developer-experience-dx)
        - [1.2.1 Salesforce CLI (sf v2)](#121-salesforce-cli-sf-v2)
        - [1.2.2 Visual Studio Code + Salesforce Extension Pack](#122-visual-studio-code--salesforce-extension-pack)
        - [1.2.3 Salesforce Code Analyzer (Opcional](#123-salesforce-code-analyzer-opcional)
    - [1.3 Source Control System](#13-source-control-system)
    - [1.4 Crear Playgrounds](#14-crear-playgrounds)
    - [1.5 Preparación del Proyecto Local](#15-preparación-del-proyecto-local)
        - [1.5.1 Crear el proyecto DX](#151-crear-el-proyecto-dx)
        - [1.5.2 Git Hooks](#152-git-hooks)
- [2. Manifest](#2-manifest)
    - [2.1 Metadata Types](#21-metadata-types)
    - [2,2 Generar package.xml](#22-generar-packagexml)
- [3. Commit & Recursive Merge & Deployment](#3-commit--recursive-merge--deployment)
    - [3.1 Extracción (Origen vs Destino)](#31-extracción-origen-vs-destino)
    - [3.2 Validación y Despliegue](#32-validación-y-despliegue)
        - [3.2.1 Validación Técnica (Dry-Run)](#321-validación-técnica-dry-run)
        - [3.2.2 Monitoreo y Quick Deploy](#322-monitoreo-y-quick-deploy)
- [4. Puntos claves](#4-puntos-claves)
    - [4.1 Beneficios](#41-beneficios)
    - [4.2 Recomendación](#42-recomendación)

# 1. Pre-Work

## 1.2 Salesforce Developer Experience (DX)
Para ejecutar el flujo de la PoC, cada estación de trabajo debe contar con el siguiente stack de herramientas configurado.

### 1.2.1 Salesforce CLI (sf v2)
Es el motor que permite la comunicación entre tu computadora y las Orgs de Salesforce.

- **Instalar Salesforce CLI**
- **Verificación:** Ejecuta `sf --version` en la terminal. Debe mostrar la versión 2.x o superior.

### 1.2.2 Visual Studio Code + Salesforce Extension Pack
El IDE oficial para el desarrollo en la plataforma.

- [Instalar Visual studio code](https://code.visualstudio.com/) 
- Instalar Extensiones de visual studio code
    - [Salesforce Extension Pack (Expanded)](https://marketplace.visualstudio.com/items?itemName=salesforce.salesforcedx-vscode-expanded)
    - [Salesforce Package.xml Generator Extension for VS Code](https://marketplace.visualstudio.com/items?itemName=modicatech.apex-code-coverage-visualizer)
    - [Apex Code Coverage Visualizer](https://marketplace.visualstudio.com/items?itemName=modicatech.apex-code-coverage-visualizer)
    - [Apex PMD](https://marketplace.visualstudio.com/items?itemName=chuckjonas.apex-pmd)
    - [ESLint](https://marketplace.visualstudio.com/items?itemName=dbaeumer.vscode-eslint)
    - [Git History](https://marketplace.visualstudio.com/items?itemName=donjayamanne.githistory)

### 1.2.3 Salesforce Code Analyzer (Opcional)
Para asegurar que el código extraído de la Org cumple con las mejores prácticas antes de ser movido a Integración:

- **Instalar Salesforce Code Analyzer**
    ```
    sf update
    sf plugins --core
    sf plugins install code-analyzer
    ```
  
- **Verificación:**
    ```
    sf code-analyzer rules
    ```

## 1.3 Source Control System
Aunque no tengamos un servidor remoto (como GitHub), usaremos **Git** localmente para crear "puntos de control" y comparar archivos.

- [Instalar GIT SCM](https://git-scm.com/install/)
- **Configuración mínima:**
    ```
    git config --global user.name "Tu Nombre"
    git config --global user.email "tu@email.com"
    git config list
    ```

## 1.4 Crear Playgrounds
Como estamos en un enfoque de **Org-Centric**, utilizamos las **Trailhead Playgrounds** o **Developer Editions** gratuitas, que funcionan exactamente igual que un entorno de producción o sandbox para fines de despliegue.

Tenemos dos opciones rápidas:

1. **Recomendada:** ir a [trailhead](https://trailhead.salesforce.com/) > Hands-On Orgs > Click en Create Playground
   - Crear dos playground llamadas **Developer** e **Integration**.
   
2. En [developer.salesforce.com](https://developer.salesforce.com) y creamos dos cuentas con correos diferentes.

*Asegúrate de tener el **Username** y **Password** de ambas. En Playgrounds de Trailhead, puedes obtener la contraseña usando el botón **"Get Your Login Credentials**" dentro de la aplicación **"Playground Starter".***

**Vinculación al Entorno Local (CLI Authentication)**

Para el Ambiente de **Origen (Developer)**:

```
sf org login web --alias dev_org --instance-url https://MyDomainDeveloperName.my.salesforce.com --set-default
```

*Se abrirá el navegador. Ingresa las credenciales.*

Para el Ambiente de **Destino (Integration)**:

```
sf org login web --alias int_org --instance-url https://MyDomainIntegrationName.my.salesforce.com
```

*Ingresa las credenciales.*

Para **listar** las organizaciones autenticadas:

 ```
 sf org list
 ```

En un entorno corporativo real, la **dev_org** sería nuestra **Developer Sandbox** y la **int_org** sería la **Partial** o **Full Copy Sandbox**. Para esta PoC, las Playgrounds simularán este comportamiento perfectamente, permitiéndote mover metadatos de una a otra sin restricciones.

***En caso de tener errores visitar => [Resolve Common Authorization Errors](https://developer.salesforce.com/docs/atlas.en-us.sfdx_dev.meta/sfdx_dev/sfdx_dev_troubleshoot_auth_errors.htm)***


## 1.5 Preparación del Proyecto Local
Antes de iniciar el flujo, el desarrollador debe preparar su área de trabajo una sola vez.

Utilizando el formato de fuente estándar de Salesforce DX, pero basado en un archivo `package.xml` (manifiesto) para tener control total sobre qué metadatos se mueven.

### 1.5.1 Crear el proyecto DX
Abra una terminal en su carpeta de trabajo y ejecute:

```
sf project generate --name LocalPlatformDeployment --template empty --manifest
code LocalPlatformDeployment
```
*Se abrirá la carpeta del proyecto en el **VSC** con los directorios por defecto, Esto creará la carpeta `manifest/package.xml` donde se listan los componentes a trabajar.*

El desarrollador debe asegúrese de que el  `.forceignore` incluya archivos que no deben viajar a la Org (como configuraciones de VS Code o archivos del sistema). [Ejemplo](https://github.com/trailheadapps/apex-recipes/blob/main/.forceignore).

En caso de que el `.gitignore` *no haya sido creado automáticamente*, entonces crear en la raíz del directorio. [Ejemplo](https://github.com/trailheadapps/apex-recipes/blob/main/.gitignore).

**Crear** el árbol de ramas (Pipeline Local) del proyecto:

```
git init
git add .
git commit -m "Estructura inicial del proyecto DX"
git branch integration
git branch developer
```

### 1.5.2 Git Hooks

Para evitar hacer **retrieve,** o **deploy** en un ambiente incorrecto, implementaremos un "**conmutador**" automático. Para que al cambiar de branch el ***target-org*** del archivo **.sf/config.json** apunte automáticamente a la ORG correspondiente.

**Crea** el automatizador, en la terminal del proyecto ejecuta:

```
code .git/hooks/post-checkout
```

Se abrirá el archivo `post-checkout`

Paga este código que vincula la branch con la Org:

```bash
#!/bin/bash
BRANCH=$(git rev-parse --abbrev-ref HEAD)

if [ "$BRANCH" == "developer" ]; then
    sf config set target-org dev_org
    echo "🔗 Contexto: DEV_ORG activo"
elif [ "$BRANCH" == "integration" ]; then
    sf config set target-org int_org
    echo "🔗 Contexto: INT_ORG activo"
else
echo -e "==== ERROR ASIGNANDO ORG ===="
fi

```

*A partir de este momento el desarrollador solo debe preocuparse de navegar entre las branches.*

```
PS C:\Users\jull.quintero\Documents\LocalPlatformDeployment> git checkout integration
Switched to branch 'integration'
Set Config
┌────────────┬─────────┬─────────┐
│ Name       │ Value   │ Success │
├────────────┼─────────┼─────────┤
│ target-org │ int_org │ true    │
└────────────┴─────────┴─────────┘

🔗 Contexto: INT_ORG activo
```

# 2. Manifest

No es el [Manifiesto Hacker](https://phrack.org/issues/7/3). Una vez que se han probado los cambios, el desarrollador debe crear un archivo de manifiesto llamado `package.xml`. Es un artefacto de entrega (release artifact) que enumera todos los componentes que deben implementarse en la organización de destino.

## 2.1 Metadata Types

Antes de crear el archivo, el desarrollador debe saber qué **[tipos de metadatos](https://developer.salesforce.com/docs/atlas.en-us.api_meta.meta/api_meta/meta_types_list.htm)** están disponibles en la Org de origen (ej. **ApexClass**, **CustomField**, **Layout**). Esto confirma cómo escribir el nombre del tipo correctamente en el XML

```
sf org list metadata-types
```

## 2.2 Generar package.xml

El desarrollador al saber que componentes creo o modifico, genera el manifiesto directamente haciendo uso del comando [**project generate manifest.**](https://developer.salesforce.com/docs/atlas.en-us.sfdx_cli_reference.meta/sfdx_cli_reference/cli_reference_project_commands_unified.htm#cli_reference_project_generate_manifest_unified)

```
sf project generate manifest --metadata MetadataType:Component, ..[]  --name package --output-dir ./manifest
```

**En nuestro ejemplo:** Llevaremos a cabo el levantamiento del *Lightning Web Components (LWC)* llamado `LdsDeleteRecord` que hace parte del [lwc-recipes](https://github.com/trailheadapps/lwc-recipes/tree/main/force-app/main/default/lwc/contactList), en este caso no se levantara la org mediante los metadatos(Scratch ORG). El desarrollador deberá llevarlo a su entorno y preparar el manifiesto. 

- Metadatype Components
    - ApexClass
        - `AccountController.cls`
        - `TestAccountController.cls`
    - LightningComponentBundle(LWC)
        - `LdsDeleteRecord`
        - `ErrorPanel`
        - `ViewSource`
        - `ldsUtils`
    - FlexiPage
        - `Home_Page_Default2`


```
sf project generate manifest --metadata ApexClass:AccountController, ApexClass:TestAccountController, FlexiPage:Home_Page_Default2, LightningComponentBundle:errorPanel, LightningComponentBundle:ldsDeleteRecord, LightningComponentBundle:ldsUtils, LightningComponentBundle:viewSource --name package --output-dir ./manifest
```

**Manifiesto generado**

```xml
<?xml version="1.0" encoding="UTF-8"?>
<Package xmlns="http://soap.sforce.com/2006/04/metadata">
    <types>
        <members>AccountController</members>
        <members>TestAccountController</members>
        <name>ApexClass</name>
    </types>
    <types>
        <members>Home_Page_Default2</members>
        <name>FlexiPage</name>
    </types>
    <types>
        <members>errorPanel</members>
        <members>ldsDeleteRecord</members>
        <members>ldsUtils</members>
        <members>viewSource</members>
        <name>LightningComponentBundle</name>
    </types>
    <version>65.0</version>
</Package>
```

*O por lo **contrario** el desarrollador puede usar el plugin llamado [*Salesforce Package.xml Generator Extension for VS Code*](https://marketplace.visualstudio.com/items?itemName=modicatech.apex-code-coverage-visualizer). para generarlo gráficamente.*

# 3. Commit & Recursive Merge & Deployment

Este es el proceso **cíclico** para mover cambios de un ambiente a otro, garantizando que el desarrollador siempre tenga un respaldo antes de aplicar cambios.

## 3.1 Extracción (Origen vs Destino)

**Cambiar** al Branch **developer** ORG y **hacer** retrieve:

```
git checkout developer
sf project retrieve start -x manifest\package.xml
```

**Crear** commit del desarrollo:

```
git add .
git commit -m "cambios listos para integración"
```

*Antes de **cambiar** de branch. **Copiar** el contenido del `Package.xml`*

**Cambiar** al Branch **integration** ORG:

```
git checkout integration
```

**CRUCIAL**: Antes de mover algo, **descarga** el estado actual de la **Org de destino** para generar un **backup**

***Pegar** el contenido del **mismo** `Package.xml`*:

```
sf project retrieve start -x manifest\package.xml
git add .
git commit -m "backup: estado previo de Integración antes del merge"
```

**Ejecutar** el merge y resolver los conflictos. Aquí es donde este **proceso local** brilla sobre los **Change Sets**. Al ejecutar:

```
git merge --no-commit --no-ff developer
```

**Resolución de conflictos:** VS Code marcará en rojo/azul las colisiones (ej. si otro desarrollador modificó el mismo Layout en Integración). Puedes elegir "Accept Both" para fusionar cambios de ambos sin borrar el trabajo ajeno.

## 3.2 Validación y Despliegue

Una vez resuelto los conflictos en local, la carpeta `force-app` contendrá el "**Código Final Fusionado".**  la **"Versión Única de la Verdad**". El objetivo ahora es asegurar que esta versión sea compatible con la **Org de destino** sin romper funcionalidades existentes.

### 3.2.1 Validación Técnica (Dry-Run)

Antes de afectar el **ambiente destino(integration)**, el desarrollador deberá ejecutar una **validación** en la misma. Usando la flag `--dry-run` para **simular el despliegue** y `-l RunSpecifiedTests` para garantizar que la cobertura de código es correcta.

Por buenas prácticas se recomendado ejecutar *siempre* los test específicos de las clases o de lo contrario los test bases de la organizacion:

```
sf project deploy start -x manifest\package.xml -l RunSpecifiedTests -t "TestAccountController" --dry-run
```

*Al usar validate, Salesforce genera un **Job ID**. Si la validación es exitosa, ese ID permite realizar un Quick Deploy (despliegue rápido) sin volver a ejecutar los tests.*

```
 ────────── Deploying Metadata (dry-run) ──────────

 Deploying (dry-run) v65.0 metadata to jull.quintero@mindful-bear-8gul7q.com using the v65.0 SOAP API.

 ✔ Preparing 375ms
 ◯ Waiting for the org to respond - Skipped
 ✔ Deploying Metadata 1.19s
   ▸ Components: 7/7 (100%)
 ✔ Running Tests 874ms
   ▸ Successful: 5/5 (100%)
   ▸ Failed: -
 ◯ Updating Source Tracking - Skipped
 ✔ Done 0ms

 Status: Succeeded
 Deploy ID: 0Affj00000AiVrxCAF
 Target Org: jull.quintero@mindful-bear-8gul7q.com
 Elapsed Time: 2.45s

Test Results Summary
Passing: 5
Failing: 0
Total: 5
Time: 608
Dry-run complete.
````

### 3.2.2 Monitoreo y Quick Deploy

**Monitorear** el progreso de la **validación** desde la **Org** destino: 

- Setup > **Deployment Status >** Dar click sobre **View Details** y luego sobre **Quick Deploy”**

Una vez que el **despliegue es confirmado como exitoso** en la Org de destino, el desarrollador procede a cerrar el **ciclo en el control de versiones local**. Esto asegura que lo que está en la org de destino coincida con la estación local del desarrollador:

```
git commit -m "Instalado en integración"
```

![](/assets/images/sfdc-local-platform-deployment/componente.png)

# 4. Puntos claves

Esta estrategia técnica de **"puente local"** es la solución más robusta para equipos sin infraestructura de servidores. Al usar Git como un **middleware** de comparación, el desarrollador transforma una tarea manual y arriesgada en un proceso auditable.

## 4.1 Beneficios

- **Independencia de Infraestructura**: No requiere servidores de Jenkins ni licencias de herramientas pagas.

- **Trazabilidad**: Cada cambio en la Org de Integración tiene un commit de Git asociado.

- **Curva de Aprendizaje**: Prepara al equipo para cuando se implemente **una herramienta formal** como Salesforce DevOps Center, GitHub Actions, Copado, Jenkins, etc.

- **Git como Árbitro:** Aunque no haya repositorio remoto (GitHub/GitLab), Git local nos permite hacer Rollbacks rápidos. Si el despliegue a Integración falla, puedes volver a tu estado anterior.

## 4.2 Recomendaciones

- **Resolución de Conflictos:**(especialmente para archivos cómo .profile o .permissionset que suelen ser problemáticos). Documentar esto dependiendo del proyecto. 

- **Destructive Changes**: Al ser la **Org la fuente de la verdad,** si borras algo localmente y despliegas, no se borrará en la Org automáticamente. Documentar el uso del **archivo** destructiveChanges.xml para limpiezas.

- **Automatización**: Este proceso puede ser automatizado localmente con un script, o corriendo jenkins en el localhost.

- **Branching model**: Implementar un modelo de ramas que incluya feature branches, similar al modelo [Gitflow workflow](https://www.atlassian.com/git/tutorials/comparing-workflows/gitflow-workflow) 
