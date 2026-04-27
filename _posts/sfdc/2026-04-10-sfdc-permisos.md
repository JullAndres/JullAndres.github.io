---
layout: single
title: ¿ Cómo internamente salesforce calcula los permisos ? - Salesforce
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
  - profile
  - permission set
  - permission set group
---

# About

`Profile`, `PermissionSet` y `PermissionSetGroup` El tema de conversación en un café de consultores salesforce “preocupados”, intentando responder la siguiente pregunta: **¿ Cuál es la mejor forma de gestionar y administrar los permisos de la plataforma ?**. 

La propuesta más frecuente es: Bueno llevemos todo en el `Profile` y solo creemos `permissionSet` para funcionalidades específicas que serán asignadas directamente a los usuarios independientemente de su perfil. Causando un caos de permisos en la plataforma cumpliendo con los accesos pero yendo en contra con el **Principio de mínimos privilegios**.

Es un punto clave en la seguridad de cualquier aplicación de software y en la que salesforce nos brinda las herramientas y conceptos administrativos correctos para llevarlo a cabo.

La respuesta simple a esta pregunta llevando a cabo el uso adecuado de las herramientas administrativas para la [Gestión de acceso y permisos de datos en salesforce](https://help.salesforce.com/s/articleView?id=platform.security_data_access_mgmt.htm&type=5) es: Usar el `Profile` solo para definir los permisos básicos y evitar extenderlos agregando permisos de funcionalidades adicionales, usar `Permission Set` donde cada uno de ellos representa una funcionalidad adicional, usar un `Permission Set Group` por perfil y agregar en ellos estos `permission set`.  

Pero.. Pero te preguntaras y.. ¿ Por qué es esta una forma correcta de administrar los permisos y cuáles son sus ventajas? Bueno y qué tal si les digo que: *Internamente para salesforce `profile`,  `permission set` y `permission set group` son lo **mismo!!** *

Entonces la pregunta fundamental que debemos hacernos es: **¿ Cómo internamente salesforce calcula los permisos ?**

Primero debemos entender cuál es la arquitectura del modelo de seguridad, salesforce implementa un modelo de seguridad multicapa basado en el **principio de acumulación de permisos y separación de responsabilidades**.

Las capas principales son:

- **Identidad** → User
- **Autorización Base** → Profile
- **Permisos Modulares** → Permission Set
- **Agrupación de Permisos** → Permission Set Group
- **Seguridad a Nivel de Objeto y Campo** → ObjectPermissions + FieldPermissions
- **Seguridad a Nivel de Registro** → OWD + Role Hierarchy + Sharing
- **Restricción Adicional** → Restriction Rules

Cada capa cumple una función específica dentro del cálculo del acceso efectivo. Documentamos la *arquitectura del modelo de seguridad* para un usuario y cómo se relaciona con los permisos a nivel de objetos, campos y licenciamiento.

# Contents
- [1. Identidad y Permiso Base](#1-identidad-y-permiso-base)
  - [1.1 User](#11-user)
  - [1.2 Profile](#12-profile)
- [2. PermissionSet – Unidad Básica de Permisos y Nucleo del modelo](#2-permissionset--unidad-básica-de-permisos-y-nucleo-del-modelo)
  - [2.1 Unidad Básica de Permisos](#21-unidad-básica-de-permisos)
    - [2.1.1 ObjectPermissions](#211-objectpermissions)
    - [2.1.2 FieldPermissions](#212-fieldpermissions)
    - [2.1.3 SetupEntityAccess](#213-setupentityaccess)
  - [2.2 Nucleo del modelo y Naturaleza Polimórfica](#22-nucleo-del-modelo-y-naturaleza-polimórfica)
    - [2.2.1 Profile](#221-profile)
    - [2.2.2 Permission Set Independiente](#222-permission-set-independiente)
    - [2.2.3 Permission Set Asociado a Groupo](#223-permission-set-asociado-a-groupo)
- [3. Asignación de Permisos a Usuarios](#3-asignación-de-permisos-a-usuarios)
  - [3.1 Asignación Directa de Permission Sets](#31-asignación-directa-de-permission-sets)
    - [3.1.1 PermissionSetAssignment](#311-permissionsetassignment)
  - [3.2 Asignación Mediante Permission Set Groups](#32-asignación-mediante-permission-set-groups)
    - [3.2.1 PermissionSetGroup](#321-permissionsetgroup)
    - [3.2.2 PermissionSetGroupComponent](#322-permissionsetgroupcomponent)
- [4. Licenciamiento](#4-licenciamiento)
  - [4.1 User License](#41-user-license)
  - [4.2 Permission set license (PSL)](#42-permission-set-license-psl)
  - [4.3 Objeto PermissionSet](#43-objeto-permissionset)
    - [4.3.1 Restricción por User License](#431-restricción-por-user-license)
    - [4.3.2 Restricción por Permission Set License (PSL)](#432-restricción-por-permission-set-license-psl)


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

User (1) —— (N) PermissionSetGroupAssignment —— (1) PermissionSet
```

*El objeto `User` no almacena permisos directamente.*
*Solo referencia estructuras que los contienen.*

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
PermissionSet (1) —— (N) ObjectPermissions
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

Es internamente un PermissionSet especial.

SOQL Query CRUD por objeto:

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

SOQL Query Field Level Security por objeto:

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


### 2.2.2 Permission Set Independiente

- Creado manualmente
- No pertenece a ningún grupo
- Asignable directamente mediante `PermissionSetAssignment`.

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
Parent.type = 'Regular' AND 
Parent.ProfileId  = null AND 
Parent.PermissionSetGroupId = null
ORDER BY Parent.Type ASC
```

SOQL Query Field Level Security por objeto:

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

### 2.2.3 Permission Set Asociado a Grupo

Un “Permission Set” relacionado con un “Permission Set Group” puede participar en tres roles distintos:

- Permission Set Componente del grupo
- Muting Permission Set
- Permission Set agregado (consolidado)

Es importante distinguir cómo se relaciona cada uno con el Grupo.

**Permission Set Componente del grupo**

Es un “Permission Set” independiente existente que ahora forma parte de un Permission Set Group.

Su relación con el grupo se define mediante el objeto: `PermissionSetGroupComponent`

Importante:

- No se crea un nuevo registro en el objeto `PermissionSet`.
- Se reutiliza el mismo Id del `PermissionSet` original.
- El campo `PermissionSetGroupId` permanece en null.

El componente no está vinculado al grupo por campo, sino por la tabla intermedia `PermissionSetGroupComponent`.

**Muting Permission Set**

Cuando se crea un Permission Set Group, Salesforce genera automáticamente un PermissionSet adicional.

Este registro:

- Es distinto de los Permission Sets componentes.
- Tiene `PermissionSetGroupId != null`.
- Está estructuralmente vinculado al Grupo mediante ese campo.

El Muting:

- No modifica los Permission Sets originales.
- No elimina permisos estructuralmente.
- Define exclusiones aplicadas durante la consolidación del Grupo.

*Es un registro nuevo en el objeto PermissionSet.*

**Permission Set agregado (consolidado)**

Cuando un Permission Set Group es asignado a un usuario, Salesforce:

- Identifica los Permission Sets componentes.
- Aplica las exclusiones definidas en el Muting (si existen).
- Genera o actualiza un PermissionSet agregado.

Este PermissionSet agregado:

- Es un registro distinto en PermissionSet.
- Tiene `PermissionSetGroupId != null`.
- Es referenciado en `PermissionSetGroupAssignment.PermissionSetId`.
- Es el que realmente evalúa el motor de seguridad.

# 3. Asignación de Permisos a Usuarios

Los permisos nunca se asignan directamente al usuario.

Siempre se asignan mediante estructuras intermedias.

Existen dos mecanismos:

- Asignación directa de Permission Sets
- Asignación mediante Permission Set Groups

Ambos mecanismos son acumulativos.

## 3.1 Asignación Directa de Permission Sets

### 3.1.1 PermissionSetAssignment

API Name: `PermissionSetAssignment`

Une usuarios con PermissionSets. Cada registro representa una asignación explícita de un PermissionSet al usuario.

**Campos principales**

- AssigneeId (UserId)
- PermissionSetId

**Relación**

```
User (1) —— (N) PermissionSetAssignment —— (1) PermissionSet
```

*Es la forma más simple y directa de extender permisos más allá del Profile.*


## 3.2 Asignación Mediante Permission Set Groups

Este mecanismo introduce una capa de abstracción orientada a gobernanza y escalabilidad.

### 3.2.1 PermissionSetGroup

API Name: `PermissionSetGroup`

Es un contenedor lógico de múltiples PermissionSets.

**Características**

- No almacena permisos directamente.
- Permite gestionar permisos por rol funcional o dominio.
- Reduce asignaciones manuales masivas.

**Campos principales**
- DeveloperName
- Status

### 3.2.2 PermissionSetGroupComponent

API Name: `PermissionSetGroupComponent`

Define la composición interna del grupo.

**Campos principales**

- PermissionSetGroupId
- PermissionSetId

**Relación**

```
PermissionSetGroup  (1) —— (N) PermissionSetGroupComponent —— (1) PermissionSet
```

Cada registro indica que un `PermissionSet` forma parte del grupo.

*Es la capa estructural que conecta agrupación con permisos reales.*

### 3.2.3 PermissionSetGroupAssignment

API Name: `PermissionSetGroupAssignment`

Denine la asignacion del grupo a al usuario

**Campo importantes**

- AssigneeId(User)
- IsActive
- PermissionSetGroupId
- PermissionSetId - **Es un PermissionSet consolidado, generado por el sistema*

**Relacion**

```
User (1) —— (N) PermissionSetGroupAssignment —— (1) PermissionSetGroup
```

Cada registro indica que el usuario está vinculado al Permission Set consolidado generado por el sistema para ese grupo, el cual representa la suma efectiva de todos los PermissionSets contenidos en el Permission Set Group.

Importante:
- No puede mezclar Permission Sets creados bajo distintas User Licenses.
- Por lo tanto, el Permission Set agregado generado por el sistema siempre será compatible con la misma User License(Esto lo veremos mas adelante).

# 4. Licenciamiento

El modelo de licenciamiento en Salesforce define el marco dentro del cual pueden otorgarse permisos.

La licencia no otorga permisos directamente. Define el límite superior de lo que un usuario puede llegar a tener.

## 4.1 User License

ApiName: `UserLicense`

La User License define el tipo de usuario y el conjunto máximo de capacidades disponibles.

- Es obligatoria
- Define el tipo de usuario
- Se asigna mediante el Profile
- Es el límite superior del modelo

Ejemplos:

- Salesforce
- Salesforce Platform
- Identity
- Customer Community
- Partner Community

La User License determina:

- Qué objetos estándar están disponibles
- Qué funcionalidades pueden usarse
- Qué perfiles pueden crearse
- Qué Permission Sets pueden asignarse

Relación estructural:

```
User
└── Profile
└── UserLicense
```

Todos los Profiles tienen obligatoriamente una User License.
Por lo tanto, todo usuario tiene una User License definida por su Profile.

Sin una User License válida, el usuario no puede existir.

## 4.2 Permission set license (PSL)

ApiName: `PermissionSetLicense`

Un Permission Set License es una licencia adicional y opcional que habilita funcionalidades específicas.

No define el tipo de usuario. Habilita features avanzadas o add-ons.

Ejemplos:

- CPQ
- Knowledge
- Inbox
- Field Service
- Service Cloud Voice

Relación estructural:

````
User
└── PermissionSetLicenseAssign
└── PermissionSetLicense
````
Se asigna directamente al usuario por medio del objeto `PermissionSetLicenseAssign`

*Un usuario debe tener asignada la PSL antes de poder recibir un Permission Set que dependa de esa PSL.*

## 4.3 Objeto PermissionSet

El objeto PermissionSet es una entidad polimórfica. Un registro en PermissionSet puede representar:

- Un Profile
- Un Permission Set independiente
- Un Permission Set agregado de un Permission Set Group
- Un Muting Permission Set

*Los campos de licencia definidos en PermissionSet aplican al registro específico, independientemente de su rol lógico.*

Este campo `LicenseId` un lookup polimórfico que puede referenciar a:

• UserLicense
• PermissionSetLicense

El objeto PermissionSet puede tener dos tipos de restricciones relacionadas con licenciamiento:

- Restricción por **User License**
- Restricción por **Permission Set License (PSL)**

Este único campo modela las restricciones de licenciamiento del PermissionSet.

Dependiendo del tipo de registro que represente el PermissionSet, el comportamiento de LicenseId cambia.

### 4.3.1 Restricción por User License

Cuando `LicenseId` apunta a un registro de `UserLicense`:

- Define compatibilidad con el tipo de usuario.
- Controla qué usuarios pueden recibir ese PermissionSet.

Un PermissionSet solo puede asignarse si:

- `User.Profile.UserLicenseId` es compatible con `PermissionSet.LicenseId`

Comportamiento según tipo de registro:

- Cuando el PermissionSet representa un Profile: `PermissionSet.LicenseId = Profile.UserLicenseId`.
- Si es un Permission Set independiente (sin PSL): `LicenseId` indica con qué UserLicense puede asignarse.
- El Permission Set agregado (consolidado) mantiene el mismo `LicenseId` (UserLicense) que los Permission Sets componentes del grupo. Salesforce no permite mezclar licencias incompatibles dentro de un mismo Permission Set Group.
- El Muting Permission Set también está asociado al mismo UserLicense que el Group al que pertenece.

En todos estos casos, `LicenseId` actúa como restricción de compatibilidad base.

### 4.3.2 Restricción por Permission Set License (PSL)

Cuando `LicenseId` apunta a un registro de `PermissionSetLicense`:

- El PermissionSet requiere una PSL específica.
- Funciona como dependencia de add-on.
- El usuario debe tener asignada esa PSL mediante PermissionSetLicenseAssign.

Regla:

Si `PermissionSet.LicenseId` referencia a un `PermissionSetLicense`, el usuario debe tener esa PSL asignada antes de recibir el PermissionSet.
- Si no la tiene, Salesforce no permite asignar el PermissionSet.
- Aplica únicamente a:
  - Permission Sets independientes creados manualmente
  - Permission Sets gestionados por paquetes
  - Permission Sets estándar del sistema

**Permission Set consolidado por el sistema**

El Permission Set agregado o consolidado generado por el sistema:

- Mantiene LicenseId apuntando a UserLicense.
- No apunta a PermissionSetLicense.
- No consolida múltiples PSL.
- No declara dependencia propia de PSL.

Si un Permission Set Group contiene componentes cuyo `LicenseId` apunta a `PermissionSetLicense`:

- Salesforce valida la dependencia de PSL sobre cada componente individual.
- La validación ocurre antes de generar o evaluar el Permission Set agregado.
- El Permission Set agregado no participa en la validación de PSL.

#### 4.3.3 Modelo de Validación Completo

Al asignar un PermissionSet a un usuario, Salesforce valida el campo LicenseId.
El comportamiento depende del tipo de registro al que apunte:

- Si LicenseId apunta a UserLicense → Se valida que el Profile.UserLicenseId del usuario sea compatible.
-  Si LicenseId apunta a PermissionSetLicense → Se valida que el usuario tenga asignada esa PSL mediante PermissionSetLicenseAssign.

Estas validaciones son automáticas y previas a la asignación.

En el caso de Permission Set Groups, la validación se realiza sobre los Permission Sets componentes antes de consolidar el Permission Set agregado.


# 6. Modelo Consolidado de Relaciones

```
User
├── Profile
│     ├── UserLicense
│     └── PermissionSet (IsOwnedByProfile = true)
│           ├── LicenseId → UserLicense
│           ├── ObjectPermissions
│           ├── FieldPermissions
│           └── System Permissions
│
├── PermissionSetAssignment
│     └── PermissionSet (independiente)
│           ├── LicenseId → (UserLicense o PermissionSetLicense)
│           ├── ObjectPermissions
│           ├── FieldPermissions
│           └── System Permissions
│
├── PermissionSetGroupAssignment
│     ├── PermissionSetGroup
│     │     ├── PermissionSetGroupComponent
│     │     │     └── PermissionSet (componentes existentes)
│     │     │           ├── LicenseId → (UserLicense o PermissionSetLicense)
│     │     │
│     │     └── Muting PermissionSet
│     │           ├── PermissionSetGroupId != null
│     │           └── LicenseId → UserLicense
│     │
│     └── PermissionSet (Agregado generado por sistema)
│           ├── LicenseId → UserLicense
│           └── Evaluado por el motor de seguridad
│
└── PermissionSetLicenseAssign
      └── PermissionSetLicense (PSL)                  
```

# 7. Evaluación de Permisos Efectivos (Se debe actualizar)

El acceso efectivo es acumulativo.

- User License define el universo base.
- Permission Set License habilita funcionalidades adicionales.
- PermissionSets otorgan permisos dentro del marco permitido.
- Permission Set Groups consolidan permisos.
- El motor evalúa PermissionSets.
- CRUD y FLS determinan acceso estructural.
- Sharing determina acceso a registros.
- Restriction Rules pueden reducir visibilidad.

Claves del modelo:

- Profile es técnicamente un PermissionSet especial
- Todo permiso estructural vive en PermissionSet
- User nunca almacena permisos directamente
- Los Permission Set Groups son una capa de gobernanza, no de permisos.
- Los Permission Set Groups no agregan lógica adicional por sí mismos.
- Los permisos nunca se restan (excepto con mecanismos específicos como Restriction Rules o Muting en PSG).
- El resultado final es la suma de todos los permisos otorgados.
- Record-level security es independiente de CRUD y FLS
- La evaluación ocurre en capas secuenciales

```
User
↓
Profile
↓
UserLicense (define universo base)
↓
PermissionSet.LicenseId compatible
↓
Si existe PermissionSetLicenseId
↓
Usuario debe tener PermissionSetLicenseAssign
↓
Motor evalúa PermissionSets
↓
CRUD → FLS → Sharing → Restriction Rules
```

# Conclusión ((Se debe actualizar))

El modelo de permisos en Salesforce está diseñado bajo:

- Separación entre definición de permisos y asignación
- Modelo acumulativo
- Evaluación por capas
- Abstracción para escalabilidad (Permission Set Groups)
- Comprender el schema real permite:

Diseñar arquitecturas escalables

- Realizar auditorías precisas
- Resolver problemas de acceso complejos
- Evitar anti‑patterns en seguridad