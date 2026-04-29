---
layout: single
title: La unidad de permisos que el motor de seguridad realmente evalúa - Salesforce
excerpt: "Salesforce implementa un modelo de seguridad multicapa basado en el principio de acumulación de permisos y separación de responsabilidades.."
date: 2026-04-10
classes: wide
header:
  teaser: /assets/images/sfdc-permisos/logo.webp
  teaser_home_page: true
  icon: /assets/images/salesforce.png
categories:
  - salesforce
tags:
  - user
  - profile
  - permissionset
  - permissionsetgroup
  - ObjectPermissions
  - FieldPermissions
  - SetupEntityAccess
  - PermissionSetGroupComponent
  - PermissionSetAssignment
  - UserLicense
  - Permissionsetlicense
---

# About

`Profile`, `PermissionSet` y `PermissionSetGroup` El tema de conversación en un café de consultores salesforce “preocupados” e intentando responder la siguiente pregunta: **¿ Cuál es la mejor forma de gestionar y administrar los permisos de la plataforma ?**. 

La propuesta más frecuente es: Bueno llevemos todo en el `Profile` y solo creemos `permissionSet` para funcionalidades específicas que serán asignadas directamente a los usuarios independientemente de su perfil. Causando un caos de permisos en la plataforma cumpliendo con los accesos pero yendo en contra con el **principio de mínimos privilegios**.

Es un punto clave en la seguridad de cualquier aplicación de software y en la que salesforce nos brinda las herramientas y conceptos administrativos correctos para llevarlo a cabo.

A esta pregunta siempre suelo responder lo siguiente: *Bueno y qué tal si les digo que: Internamente para salesforce `profile`,  `permission set` y `permission set group` son lo **mismo!!***. Si suena loco..

Para entenderlo debemos hacernos una pregunta mas elemental **¿ Cómo internamente salesforce calcula los permisos ?**

Primero debemos entender cuál es la arquitectura del modelo de seguridad, salesforce implementa un modelo de seguridad multicapa basado en el **principio de acumulación de permisos y separación de responsabilidades**.

Las capas principales son:

- **Identidad** → User
- **Autorización Base** → Profile
- **Permisos Modulares** → Permission Set
- **Agrupación de Permisos** → Permission Set Group
- **Seguridad a Nivel de Objeto y Campo** → ObjectPermissions + FieldPermissions
- **Seguridad a Nivel de Registro** → OWD + Role Hierarchy + Sharing
- **Restricción Adicional** → Restriction Rules

Cada capa cumple una función específica dentro del cálculo del acceso efectivo. Documentaremos la *arquitectura del modelo de seguridad* para un usuario y entenderemos cómo se relaciona con los permisos en sus diferentes abstracciones a nivel de objetos, campos y licenciamientos.

# Contents
- [1. Identidad y Permiso Base](#1-identidad-y-permiso-base)
  - [1.1 User](#11-user)
  - [1.2 Profile](#12-profile)
- [2. Objeto PermissionSet – Unidad Básica de Permisos y Nucleo del modelo](#2-objeto-permissionset--unidad-básica-de-permisos-y-núcleo-del-modelo)
  - [2.1 Unidad Básica de Permisos](#21-unidad-básica-de-permisos)
    - [2.1.1 ObjectPermissions](#211-objectpermissions)
    - [2.1.2 FieldPermissions](#212-fieldpermissions)
    - [2.1.3 SetupEntityAccess](#213-setupentityaccess)
  - [2.2 Nucleo del modelo y Naturaleza Polimórfica](#22-núcleo-del-modelo-y-naturaleza-polimórfica)
    - [2.2.1 Profile](#221-profile)
    - [2.2.2 Permission Set](#222-permission-set)
    - [2.2.3 Permission Set Group](#223-permission-set-group)
- [3. Objecto PermissionSetGroup - Capa de abstracción orientada a la gobernanza y escalabilidad](#3-objecto-permissionsetgroup---capa-de-abstracción-orientada-a-la-gobernanza-y-escalabilidad)
  - [3.1 PermissionSetGroupComponent](#31-permissionsetgroupcomponent)
- [4. Asignación de Permisos a Usuarios](#4-asignación-de-permisos-a-usuarios)
  - [4.1 PermissionSetAssignment](#41-permissionsetassignment)
- [5. Licenciamiento](#5-licenciamiento)
  - [5.1 UserLicense](#51-userlicense)
  - [5.2 PermissionsetLicense(PSL)](#52-permissionsetlicense-psl)
  - [5.3 Objeto PermissionSet](#53-objeto-permissionset)
    - [5.3.1 Restricción por User License](#531-restricción-por-userlicense)
    - [5.3.2 Restricción por Permission Set License(PSL)](#532-restricción-por-permissionsetlicensepsl)


# 1. Identidad y Permiso Base

## 1.1 User 

Object API Name: [`User`](https://developer.salesforce.com/docs/atlas.en-us.object_reference.meta/object_reference/sforce_api_objects_user.htm)

Representa la identidad autenticada dentro del sistema.

**Campos importantes**

- `ProfileId`
- `UserRoleId`
- `IsActive`
- `UserType`

**Relaciones estructurales**

```
Profile (1) —— (1) User

User (1) —— (N) PermissionSetAssignment —— (1) PermissionSet

User (1) —— (N) PermissionSetAssignment —— (1) PermissionSetGroup
```

*El objeto `User` no almacena permisos directamente. Solo referencia estructuras que los contienen.*

## 1.2 Profile

API Name: [`Profile`](https://developer.salesforce.com/docs/atlas.en-us.object_reference.meta/object_reference/sforce_api_objects_profile.htm)

Define el conjunto base obligatorio de permisos asociado a cada usuario.

**Campos importantes**
- `Name`
- `UserLicenseId`
- `UserType`

**Concepto de Arquitectura Clave**

*Internamente, *cada Profile* tiene un registro especial en el objeto `PermissionSet`.* Es acá es donde está lo interesante lo veremos más adelante...

Es decir:

- Cada `Profile` tiene un registro correspondiente en el objeto `PermissionSet`
- Ese registro en el objeto `PermissionSet` contiene los permisos reales del `Profile` que el sistema valida automáticamente. No en el objeto `Profile`

Esto significa que: *Profile es técnicamente es un `PermissionSet`*

*Esto explica por qué muchas consultas técnicas deben realizarse sobre `PermissionSet` incluso cuando conceptualmente hablamos de Profile.*

ACLARACIÓN: **En este contexto `PermissionSet` hace referencia al mecanismo interno donde salesforce alcanema los permisos para ser evaluados de forma automática por su sistema**. 

# 2. Objeto PermissionSet – Unidad Básica de Permisos y Núcleo del modelo

API Name: [`PermissionSet`](https://developer.salesforce.com/docs/atlas.en-us.object_reference.meta/object_reference/sforce_api_objects_permissionset.htm)

Es el objeto central del modelo de permisos, la unidad estructural donde realmente los permisos se almacenan.

**Campos importantes**

- `Name`
- `Type`
- `IsOwnedByProfile`
- `ProfileId`
- `PermissionSetGroupId`
- `LicenseId`
- `PermissionsModifyAllData`

Contiene tanto permisos de sistema como relaciones a objetos hijos que representan permisos específicos.

Es en este objeto donde radica el poder del motor se seguridad de salesforce dado su comportamiento polimórfico.

## 2.1 Unidad Básica de Permisos

Los permisos reales se **almacenan en objetos relacionados**.

### 2.1.1 ObjectPermissions

API Name: [`ObjectPermissions`](https://developer.salesforce.com/docs/atlas.en-us.object_reference.meta/object_reference/sforce_api_objects_objectpermissions.htm)

Representa permisos **CRUD** sobre objetos.

**Campos importantes**

- `ParentId` → Lookup a **objeto** `PermissionSet`
- `SObjectType`
- `PermissionsRead`
- `PermissionsCreate`
- `PermissionsEdit`
- `PermissionsDelete`
- `PermissionsViewAllRecords`
- `PermissionsModifyAllRecords`

**Relaciones estructurales**

```
PermissionSet (1) —— (N) ObjectPermission
```

*Controla acceso a nivel de objeto.*

*No controla acceso a registros específicos.*

### 2.1.2 FieldPermissions

API Name: [`FieldPermissions`](https://developer.salesforce.com/docs/atlas.en-us.object_reference.meta/object_reference/sforce_api_objects_fieldpermissions.htm)

Representa seguridad a nivel de campo (Field Level Security).

**Campos importantes**

- `ParentId` → Lookup a **objeto** `PermissionSet`
- `SObjectType`
- `Field`
- `PermissionsRead`
- `PermissionsEdit`

**Relaciones estructurales**

```
PermissionSet (1) —— (N) FieldPermissions
```

*Controla visibilidad y edición de campos.*

*No controla acceso al objeto ni al registro.*

### 2.1.3 SetupEntityAccess

API Name: [SetupEntityAccess](https://developer.salesforce.com/docs/atlas.en-us.object_reference.meta/object_reference/sforce_api_objects_setupentityaccess.htm)

Permite acceso a componentes de metadata como:

- `ApexClass`
- `ApexPage`
- `CustomPermission`
- `CustomMetadata`
- Entro otros

**Campos importantes**

- `ParentId`
- `SetupEntityId`
- `SetupEntityType`


## 2.2 Núcleo del modelo y Naturaleza Polimórfica

Un registro en el objeto `PermissionSet` es la unidad real que el motor de seguridad evalúa.

**"Profiles"**, **"Permission Set"** y  **"Permission Set Groups"** son abstracciones administrativas que terminan materializando como registros en el objeto `PermissionSet`.

Todo permiso evaluado proviene de un `PermissionSet`.

Puede representar:

- `Profile`
- `Permission Set` independiente
- `Permission Set` Asociado a **Groupo**(PSG)

*Todos son registros del mismo objeto: `PermissionSet`.*

Campos que permiten identificar su rol:

- `Type`
- `IsOwnedByProfile`
- `ProfileId`
- `PermissionSetGroupId`

### 2.2.1 Profile

Internamente en salesforce por cada `Profile` existe un registro en el objeto `PermissionSet` que representa los permisos de cada `Profile` y que es administrado a partir de la [standard metadata type profile](https://developer.salesforce.com/docs/atlas.en-us.api_meta.meta/api_meta/meta_profile.htm) o desde la interfaz grafica. 

El mecanismo para obtener este registro es mediante el filtro: 

```
type = 'Profile' AND 
IsOwnedByProfile = true AND
ProfileId  != null
```

**Ejemplo SOQL Query CRUD por objeto**:

```SQL
SELECT 
    Parent.Name, 
    Parent.Label, 
    Parent.Description,
    Parent.Type,
    Parent.Profile.Name,
    parent.PermissionSetGroup.DeveloperName, 
    parent.LicenseId, 
    Parent.License.Name,
    PermissionsRead,
    PermissionsCreate,
    PermissionsEdit, 
    PermissionsDelete,
    PermissionsViewAllRecords,
    PermissionsModifyAllRecords
FROM ObjectPermissions
WHERE SobjectType = '{SobjectName}' AND
Parent.type = 'Profile' AND 
Parent.IsOwnedByProfile = true AND
Parent.ProfileId  != null
ORDER BY Parent.Type ASC
```

**Ejemplo SOQL Query Field Level Security por objeto**:

```SQL
SELECT 
    ...
    Field, 
    PermissionsRead, 
    PermissionsEdit 
FROM FieldPermissions 
WHERE SobjectType = '{SobjectName}'  AND
Parent.type = 'Profile' AND 
Parent.IsOwnedByProfile = true AND
Parent.ProfileId  != null
ORDER BY Parent.Name, Field
```

### 2.2.2 Permission Set

Internamente en salesforce por cada `PermissionSet` existe un registro en el objeto `PermissionSet` que representa los permisos de cada `permissionset` y que es administrado a partir de la [standard metadata type permissionSet](https://developer.salesforce.com/docs/atlas.en-us.api_meta.meta/api_meta/meta_permissionset.htm) o desde interfaz grafica.

jaja! si, así como lo lees, veámoslo con lupa para entenderlo:

Cuando creamos un `PermissionSet` desde la interfaz gráfica de salesforce para ahí agregar los permisos a objetos, clases, campos etc. Salesforce internamente crea un registro en el objeto `PermissionSet`pero también internamente crea una metadata de tipo `PermissionSet`, donde estos tres mecanismo representan una unidad, queriendo decir que tenemos tres formas de manipular los permisos: por la interfaz gráfica, por deploy o manipulando los registros. Y así mismo funciona cuando hablamos de perfiles.

El mecanismo para obtener este registro es mediante el filtro: 

```SQL
type = 'Regular' AND 
ProfileId  = null AND 
PermissionSetGroupId = null
```

**Ejemplo SOQL Query CRUD por objeto**:

```SQL
SELECT 
    ...
    PermissionsRead,
    PermissionsCreate,
    PermissionsEdit, 
    PermissionsDelete,
    PermissionsViewAllRecords,
    PermissionsModifyAllRecords
FROM ObjectPermissions
WHERE SobjectType = '{SobjectName}' AND
Parent.type = 'Regular' AND 
Parent.ProfileId  = null AND 
Parent.PermissionSetGroupId = null
ORDER BY Parent.Type ASC
```

**Ejemplo SOQL Query Field Level Security por objeto**:

```SQL
SELECT 
    ...
    Field, 
    PermissionsRead, 
    PermissionsEdit 
FROM FieldPermissions 
WHERE SobjectType = '{SobjectName}'  AND
Parent.type = 'Regular' AND  
Parent.ProfileId  = null AND
Parent.PermissionSetGroupId = null
ORDER BY Parent.Name, Field
```

### 2.2.3 Permission Set Group

Internamente en salesforce por cada `PermissionSetGroup` existe un registro en el objeto `PermissionSet` que representa los permisos del `PermissionSetGroup` y que es administrado a partir de la [standard metadata type PermissionSetGroup](https://developer.salesforce.com/docs/atlas.en-us.api_meta.meta/api_meta/meta_permissionsetgroup.htm) o desde la interfaz grafica.

El mecanismo para obtener este registro es mediante el filtro: 

```SQL
type = 'Group' AND 
ProfileId  = null AND 
PermissionSetGroupId != null
```

Me gusta llamarlo **`PermissionSet` agregado o consolidado por el sistema**. Porque cuando un `PermissionSetGroup` es asignado a un usuario, Salesforce de manera automática e internamente:

- Identifica los `PermissionSets` que componen al grupo, para sumar todos los accesos.
- Aplica las exclusiones definidas en el `MutingPermissionSet` (si existen).
- Generando o actualizando este registro en el objeto `PermissionSet`.

Este PermissionSet consolidado:

- Es un registro distinto en PermissionSet.
- Tiene `type = 'Group'` y `PermissionSetGroupId != null`. Como vimos en el filtro anterior.
- Es referenciado en `PermissionSetGroupAssignment.PermissionSetId`.
- Es el que realmente evalúa el motor de seguridad.

**Permission Sets componentes del grupo**.

Son los `PermissionSets` independientes existentes que ahora forman parte de un `PermissionSetGroup`.

Su relación con el grupo se define mediante el objeto: `PermissionSetGroupComponent`

- No se crea un nuevo registro en el objeto `PermissionSet`.
- Se reutiliza el mismo Id del `PermissionSet` original.
- El campo `PermissionSetGroupId` permanece en `null`.

**Muting Permission Set**:

Se usa cuando no se desea asignar a los usuarios todos los permisos incluidos en el `PermissionSetGroup`

- No modifica los `PermissionSets` originales.
- No elimina permisos estructuralmente.
- Define exclusiones aplicadas durante la consolidación del grupo.

# 3. Objecto PermissionSetGroup - Capa de abstracción orientada a la gobernanza y escalabilidad.

API Name: [`PermissionSetGroup`](https://developer.salesforce.com/docs/atlas.en-us.object_reference.meta/object_reference/sforce_api_objects_permissionsetgroup.htm)

Es un contenedor lógico de múltiples `PermissionSets`.

**Campos importantes**

- `DeveloperName`
- `Status`

**Características**

- No almacena permisos directamente.
- Reduce asignaciones manuales masivas.

## 3.1 PermissionSetGroupComponent

API Name: [`PermissionSetGroupComponent`](https://developer.salesforce.com/docs/atlas.en-us.object_reference.meta/object_reference/sforce_api_objects_permissionsetgroupcomponent.htm)

Define la composición interna del grupo.

**Campos importantes**

- `PermissionSetGroupId`
- `PermissionSetId`

**Relaciones estructurales**

```
PermissionSetGroup  (1) —— (N) PermissionSetGroupComponent —— (1) PermissionSet
```

Cada registro indica que un `PermissionSet` forma parte del grupo.

*Es la capa estructural que conecta agrupación con permisos reales.*

# 4. Asignación de Permisos a Usuarios

Los permisos nunca se asignan directamente al usuario, siempre se asignan mediante estructuras intermedias.

Existen dos mecanismos:

- Asignación directa de `PermissionSets`.
- Asignación mediante `PermissionSetGroups`

Ambos mecanismos son acumulativos.

## 4.1 PermissionSetAssignment

API Name: [`PermissionSetAssignment`](https://developer.salesforce.com/docs/atlas.en-us.object_reference.meta/object_reference/sforce_api_objects_permissionsetassignment.htm)

Une usuarios con `PermissionSets`, cada registro representa una asignación explícita de un `PermissionSet` o `PermissionSetGroup` al usuario.

**Campos importantes**

- `AssigneeId` → Lookup a **objeto** `User`
- `PermissionSetId`
- `PermissionSetGroupId`
- `IsActive`

**Relaciones estructurales**

```
User (1) —— (N) PermissionSetAssignment —— (1) PermissionSet

User (1) —— (N) PermissionSetAssignment —— (1) PermissionSetGroup
```

*Es la forma más simple y directa de extender permisos más allá del `Profile`.*

# 5. Licenciamiento

El modelo de licenciamiento define el marco dentro del cual pueden otorgarse permisos.

La licencia no otorga permisos directamente, define el límite superior de lo que un usuario puede llegar a tener.

## 5.1 UserLicense

ApiName: [`UserLicense`](https://developer.salesforce.com/docs/atlas.en-us.object_reference.meta/object_reference/sforce_api_objects_userlicense.htm)

Define el tipo de usuario y el conjunto máximo de capacidades disponibles.

- Es obligatoria
- Define el tipo de usuario
- Se asigna mediante el Profile
- Es el límite superior del modelo

Ejemplos:

- `Salesforce`
- `Salesforce Platform`
- `Identity`
- `Customer Community`
- `Partner Community`

Determina:

- Qué objetos estándar están disponibles
- Qué funcionalidades pueden usarse
- Qué perfiles pueden crearse
- Qué Permission Sets pueden asignarse

**Campos importantes**

- `Name`
- `MasterLabel`
- `Status`
- `TotalLicenses`
- `UsedLicenses`
- `UsedLicensesLastUpdated`


**Relaciones estructurales**

```
User (1) —— (1) Profile —— (1) UserLicense
```

Todos los Profiles tienen obligatoriamente una `UserLicense` por lo tanto, todo usuario tiene una User License definida por su Profile.

*Sin una User License válida, el usuario no puede existir*.

## 5.2 PermissionsetLicense (PSL)

ApiName: [`PermissionSetLicense`](https://developer.salesforce.com/docs/atlas.en-us.object_reference.meta/object_reference/sforce_api_objects_permissionsetlicense.htm)

Licencia adicional y opcional que habilita funcionalidades específicas.

No define el tipo de usuario. Habilita features avanzadas o add-ons.

Ejemplos:

- `CPQ`
- `Knowledge`
- `Inbox`
- `Field Service`
- `Service Cloud Voice`

**Campos importantes**

- `DeveloperName`
- `MasterLabel`
- `Status`
- `TotalLicenses`
- `UsedLicenses`

**Relaciones estructurales**

```
User (1) —— (N) PermissionSetLicenseAssign —— (1) PermissionSetLicense
```

Se asigna directamente al usuario por medio del objeto [`PermissionSetLicenseAssign`](https://developer.salesforce.com/docs/atlas.en-us.object_reference.meta/object_reference/sforce_api_objects_permissionsetlicenseassign.htm)

*Un usuario puede tener asignada la PSL mediante el `PermissionSet` que la otorga o directamente*.

SOQL Query para obtener lista de usuarios con licencias especificas asignadas: 

```SQL
SELECT Id, Name, Username, Profile.Name
FROM User
WHERE 
Id IN (
    SELECT AssigneeId
    FROM PermissionSetLicenseAssign
    WHERE PermissionSetLicense.DeveloperName IN 
        ('{PSL_DeveloperName}')
)
```

## 5.3 Objeto PermissionSet

Como vimos anteriormente es una entidad polimórfica. Un registro en `PermissionSet` puede representar:

- `Profile`
- `Permission Set` Independiente
- `Permission Set` Asociado a **Grupos**(PSG)

El campo `LicenseId` es un lookup polimórfico que puede referenciar a:

• `UserLicense`
• `PermissionSetLicense`

El objeto `PermissionSet` puede tener dos tipos de restricciones relacionadas con licenciamiento:

- Restricción por **User License**
- Restricción por **Permission Set License (PSL)**

Este único campo modela las restricciones de licenciamiento del PermissionSet.

Dependiendo del tipo de registro que represente el `PermissionSet`, el comportamiento de `LicenseId` cambia.

### 5.3.1 Restricción por UserLicense

Cuando `LicenseId` apunta a un registro de `UserLicense`:

- Define compatibilidad con el tipo de usuario.
- Controla qué usuarios pueden recibir ese `PermissionSet`.

Un  `PermissionSet` solo puede asignarse si:

- `User.Profile.UserLicenseId` es compatible con `PermissionSet.LicenseId`

Comportamiento según tipo de registro:

- Cuando el `PermissionSet` representa un `Profile` : `PermissionSet.LicenseId` es igual a `Profile.UserLicenseId`.
- Si es un `PermissionSet` independiente: `LicenseId` indica con qué `UserLicense` puede asignarse.
- El **Permission Set consolidado por el sistema** del `PermissionSetGroup` mantiene el mismo `LicenseId` (UserLicense) que los `PermissionSets` componentes del grupo. Salesforce no permite mezclar licencias incompatibles dentro de un mismo `PermissionSetGroup`.
- El `MutingPermissionSet` también está asociado al mismo UserLicense que el Group al que pertenece.

En todos estos casos, `LicenseId` actúa como restricción de compatibilidad base.

### 5.3.2 Restricción por PermissionSetLicense(PSL)

Cuando `LicenseId` apunta a un registro de `PermissionSetLicense`:

- El `PermissionSet` requiere una PSL específica.
- Funciona como dependencia de add-on.
- El usuario debe tener asignada esa PSL mediante PermissionSetLicenseAssign.

Regla:

Si `PermissionSet.LicenseId` referencia a un `PermissionSetLicense` y este se asigna al usuario: 
- Automaticamente salesforece crea registro del objecto `PermissionSetLicenseAssign`
- Aplica únicamente a:
  - `PermissionSets` independientes creados manualmente
  - `PermissionSets` gestionados por paquetes
  - `PermissionSets` estándar del sistema

**Permission Set consolidado por el sistema:**

- Mantiene `LicenseId` apuntando a `UserLicense`.
- No apunta a `PermissionSetLicense`(PSL).
- No consolida múltiples `PSL.
- No declara dependencia propia de `PSL`.

Si un `PermissionSetGroup` contiene componentes cuyo `LicenseId` apunta a `PermissionSetLicense`:

- Salesforce valida la dependencia de PSL sobre cada componente individual.
- La validación ocurre antes de generar o actualizar el **Permission Set consolidado por el sistema**.

# 6. Arquitectura del modelo de seguridad

```
User
├── Profile
│     ├── UserLicense
│     └── PermissionSet type Profile
│           ├── LicenseId → UserLicense
│           ├── ObjectPermissions
│           ├── FieldPermissions
│
├── PermissionSetAssignment
│     │└── PermissionSet Type Regular
│     │      ├── LicenseId → (UserLicense o PermissionSetLicense)
│     │      ├── ObjectPermissions
│     │      ├── FieldPermissions
│     │
│     ├── PermissionSetGroup
│     │     ├── PermissionSetGroupComponent
│     │     │     └── PermissionSet Type Regular (componentes existentes)
│     │     │           └── LicenseId → (UserLicense o PermissionSetLicense)
│     │     └── MutingPermissionSet
│     │           
│     └── PermissionSet Type Group (consolidado por el sistema)
│           ├── LicenseId → UserLicense
│           └── Evaluado por el motor de seguridad
│
└── PermissionSetLicenseAssign
      └── PermissionSetLicense (PSL)                  
```

Ahora si!! teniendo este entendimiento podemos pensar en la pregunta inicial **¿ Cuál es la mejor forma de gestionar y administrar los permisos de la plataforma ?**.

La arquitectura recomendada mediante la [Gestión de acceso y permisos de datos en salesforce](https://help.salesforce.com/s/articleView?id=platform.security_data_access_mgmt.htm&type=5) basado en un modelo modular y gobernable mediante el principio de **mínimos privilegios** es definir:

- **Profiles con mínimos accesos**
  - Definen solo acceso base.
  - No deben contener permisos amplios o excepcionales.

- **Permission Sets para extender acceso**
  - Cada PS representa una capacidad específica.
  - Permite asignar acceso adicional sin modificar el Profile.
  - Facilita trazabilidad y control de excepciones.

- **Permission Set Groups para modelar roles**
  - Agrupan capacidades funcionales.
  - Representan responsabilidades de negocio.
  - Permiten gestionar permisos de manera estructurada.

Para con esto adoptar:

- **Mayor trazabilidad**
  - Claridad sobre qué permiso otorga cada capacidad.
  - Facilidad para auditorías y revisiones.

- **Control más granular**
  - Menor dependencia en accesos globales.
  - Acceso alineado a necesidades reales del negocio.

- **Reducción de acumulación de privilegios**
  - Separación clara entre acceso base y acceso extendido.
  - Menor riesgo de sobreasignación.
  - Menor riesgo en detención de vulnerabilidades por pruebas de ethical hacking 

- **Arquitectura más sostenible**
  - Menor deuda técnica.
  - Modelo más simple de mantener.