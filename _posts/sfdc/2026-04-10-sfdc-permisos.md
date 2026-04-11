---
layout: single
title: Modelo de Datos de Permisos en Salesforce
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
  - Profile
  - permission set
  - permission set group
---

# About

Este documento describe el modelo de datos que representa los permisos en Salesforce, detallando:

- Los objetos involucrados en la definición de permisos
- Las relaciones entre dichos objetos
- Los mecanismos de asignación a usuarios
- Cómo se calcula el acceso efectivo

# 1. Arquitectura General del Modelo de Seguridad

Salesforce implementa un modelo de seguridad multicapa basado en el principio de acumulación de permisos y separación de responsabilidades.

Las capas principales son:

- **Identidad** → User
- **Autorización Base** → Profile
- **Permisos Modulares** → Permission Set
- **Agrupación de Permisos** → Permission Set Group
- **Seguridad a Nivel de Objeto y Campo** → ObjectPermissions + FieldPermissions
- **Seguridad a Nivel de Registro** → OWD + Role Hierarchy + Sharing
- **Restricción Adicional** → Restriction Rules

Cada capa cumple una función específica dentro del cálculo del acceso efectivo.

# 2. Identidad y Permiso Base

## 2.1 User 

API Name: `User`

Representa la identidad autenticada dentro del sistema.

**Campos clave**

- Id
- ProfileId
- UserRoleId
- IsActive
- UserType

**Relaciones estructurales**

```
User
 ├── Profile
 ├── PermissionSetAssignment
 └── PermissionSetGroupAssignment
```

Relación N:N con `PermissionSet` vía `PermissionSetAssignment`

*El objeto `User` no almacena permisos directamente.*
*Solo referencia estructuras que los contienen.*

## 2.2 Profile

API Name: `Profile`

Define el conjunto base obligatorio de permisos asociado a cada usuario.

**Concepto Arquitectónico Clave**

*Internamente, *cada Profile* es un `PermissionSet` especial.*

En el objeto `PermissionSet`:

```
- IsOwnedByProfile = true
- ProfileId ≠ null

Profile = PermissionSet (IsOwnedByProfile = true)
```

Es decir:

- Cada Profile tiene un registro correspondiente en PermissionSet
- Ese PermissionSet contiene los permisos reales del perfil

Esto significa que: *Profile → técnicamente es un `PermissionSet`*

*Esto explica por qué muchas consultas técnicas deben realizarse sobre `PermissionSet` incluso cuando conceptualmente hablamos de Profile.*

## 3. PermissionSet – Unidad Básica de Permisos y Nucleo del modelo

API Name: `PermissionSet`

Es el objeto central del modelo de permisos, la unidad estructural donde realmente los permisos se almacenan.

**Campos importantes**

- Id
- Name
- Label
- IsOwnedByProfile
- ProfileId
- PermissionSetGroupId
- PermissionsModifyAllData

Contiene tanto permisos de sistema como relaciones a objetos hijos que representan permisos específicos.

Es en este objeto donde radica el poder del motor se seguridad de salesforce dado su comportamiendo polimorfico

### 3.1 Unidad Básica de Permisos

Los permisos reales se **almacenan en objetos relacionados**.

#### 3.1.1 ObjectPermissions

API Name: `ObjectPermissions`

Representa permisos **CRUD** sobre objetos.

**Campos Principales**

- ParentId → Lookup a PermissionSet
- SObjectType
- PermissionsRead
- PermissionsCreate
- PermissionsEdit
- PermissionsDelete
- PermissionsViewAllRecords
- PermissionsModifyAllRecords

**Relación**

```
PermissionSet (1) —— (N) ObjectPermissions
```

*Controla acceso a nivel de objeto.*

*No controla acceso a registros específicos.*

#### 3.1.2 FieldPermissions

API Name: `FieldPermissions`

Representa seguridad a nivel de campo (Field Level Security).

**Campos Principales**

- ParentId
- SObjectType
- Field
- PermissionsRead
- PermissionsEdit

**Relación**

```
PermissionSet (1) —— (N) FieldPermissions
```

*Controla visibilidad y edición de campos.*

*No controla acceso al objeto ni al registro.*

#### 3.1.3 SetupEntityAccess

Permite acceso a componentes de metadata como:

- ApexClass
- ApexPage
- CustomPermission
- CustomMetadata

**Campos**

- ParentId
- SetupEntityId
- SetupEntityType

#### 3.1.4 ApexClassAccess

En algunos contextos aparece como entidad separada.

**Campos**

- ParentId
- ApexClassId

#### 3.1.5 CustomPermission

Permisos personalizados definidos por configuración.

**Objetos involucrados**

- CustomPermission
- SetupEntityAccess

**Uso típico**

- Feature toggles
- Validaciones condicionales en Apex
- Control de UI dinámico

### 3.2 Nucleo del modelo y Naturaleza Polimórfica

PermissionSet es la unidad real que el motor de seguridad evalúa.

Profiles, Permission Set y  Permission Set Groups son abstracciones administrativas que terminan materializándose como registros en el objeto PermissionSet.

Todo permiso evaluado proviene de un PermissionSet.

Puede representar:

- Profile
- Permission Set independiente
- Permission Set Asociado a Group

*Todos son registros del mismo objeto: PermissionSet.*

Campos que permiten identificar su rol:

- `Type`
- `IsOwnedByProfile`
- `ProfileId`
- `PermissionSetGroupId`

#### 3.2.1 Profile

Es internamente un PermissionSet especial

Campos tipicos:

```
- `Type = 'Profile'`
- `IsOwnedByProfile = true`
- `ProfileId != null`
```

#### 3.2.2 Permission Set Independiente

- Creado manualmente
- No pertenece a ningún grupo
- Asignable directamente mediante `PermissionSetAssignment`.

Campos tipicos:

```
- `Type = 'Regular'`
- `ProfileId = null`
- `PermissionSetGroupId = null`
```

#### 3.2.3 Permission Set Asociado a Group

Un Permission Set relacionado con un Permission Set Group puede participar en tres roles distintos:

- Permission Set Componente del grupo
- Muting Permission Set
- Permission Set agregado (consolidado)

Es importante distinguir cómo se relaciona cada uno con el Group.

**Permission Set Componente del grupo**

Es un Permission Set independiente existente que ahora forma parte de un Permission Set Group.

Su relación con el grupo se define mediante el objeto:

PermissionSetGroupComponent

Importante:

- No se crea un nuevo registro en PermissionSet.
- Se reutiliza el mismo Id del Permission Set original.
- El campo PermissionSetGroupId permanece en null.

El componente no está vinculado al grupo por campo, sino por la tabla intermedia PermissionSetGroupComponent.

Cuando el Group es asignado a un usuario:

- El motor de seguridad no evalúa directamente el Permission Set componente.
- Evalúa el PermissionSet agregado generado para el Group.

*No se crea un nuevo registro en PermissionSet. Se reutiliza el mismo Id.*

Aquí ocurre algo importante:
El motor NO evalúa Permission Set independiente para el usuario. Evalúa el PermissionSet Agregado generado por el sistema.

Ese PermissionSet agregado:

- Incluye los permisos de Permission Set independiente que se encuentran en el PSG
- Aplica exclusiones de Muting (si existen)

**Muting Permission Set**

Cuando se crea un Permission Set Group, Salesforce genera automáticamente un PermissionSet adicional.

Este registro:

- Es distinto de los Permission Sets componentes.
- Tiene PermissionSetGroupId != null.
- Está estructuralmente vinculado al Group mediante ese campo.

El Muting:

- No modifica los Permission Sets originales.
- No elimina permisos estructuralmente.
- Define exclusiones aplicadas durante la consolidación del Group.

*Es un registro nuevo en el objeto PermissionSet.*

**Permission Set agregado (consolidado)**

Cuando un Permission Set Group es asignado a un usuario, Salesforce:

- Identifica los Permission Sets componentes.
- Aplica las exclusiones definidas en el Muting (si existen).
- Genera o actualiza un PermissionSet agregado.

Este PermissionSet agregado:

- Es un registro distinto en PermissionSet.
- Tiene PermissionSetGroupId != null.
- Es referenciado en PermissionSetGroupAssignment.PermissionSetId.
- Es el que realmente evalúa el motor de seguridad.

# 4 Asignación de Permisos a Usuarios

Los permisos nunca se asignan directamente al usuario.

Siempre se asignan mediante estructuras intermedias.

Existen dos mecanismos:

- Asignación directa de Permission Sets
- Asignación mediante Permission Set Groups

Ambos mecanismos son acumulativos.

## 4.1 Asignación Directa de Permission Sets

### 4.1.1 PermissionSetAssignment

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


## 4.2 Asignación Mediante Permission Set Groups

Este mecanismo introduce una capa de abstracción orientada a gobernanza y escalabilidad.

### 4.2.1 PermissionSetGroup

API Name: `PermissionSetGroup`

Es un contenedor lógico de múltiples PermissionSets.

**Características**

- No almacena permisos directamente.
- Permite gestionar permisos por rol funcional o dominio.
- Reduce asignaciones manuales masivas.

**Campos principales**
- DeveloperName
- Status

### 4.2.2 PermissionSetGroupComponent

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

### 4.2.3 PermissionSetGroupAssignment

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

# 5. Modelo Consolidado de Relaciones

```
User
 ├── Profile
 │     └── PermissionSet (IsOwnedByProfile = true)
 │
 ├── PermissionSetAssignment
 │     └── PermissionSet
 │           ├── ObjectPermissions
 │           ├── FieldPermissions
 │           └── System Permissions
 │
 └── PermissionSetGroupAssignment
       └── PermissionSetGroup
             └── PermissionSetGroupComponent
                   └── PermissionSet
                         ├── ObjectPermissions
                         ├── FieldPermissions
                         └── System Permissions
```

# 6. Evaluación de Permisos Efectivos

El acceso efectivo es acumulativo.

```
Permisos efectivos =
    PermissionSet del Profile
  + PermissionSets asignados directamente
  + PermissionSets contenidos en PermissionSetGroups
```

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
Identidad (User)
  ↓
Profile (PermissionSet base)
  ↓
PermissionSets adicionales
  ↓
PermissionSetGroups
  ↓
CRUD (ObjectPermissions)
  ↓
FLS (FieldPermissions)
  ↓
Sharing (OWD + Roles + Rules)
  ↓
Restriction Rules
```

# Conclusión

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