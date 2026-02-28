---
layout: single
title: Developer Experience(DX). High-Volume Data Ingestion via Bulk API 2.0 - Salesforce
excerpt: "La Bulk API 2.0 representa la evolución de la carga masiva en Salesforce, diseñada para maximizar el rendimiento y simplificar la experiencia del desarrollador. A diferencia de las APIs basadas en REST o SOAP tradicionales, Bulk 2.0 utiliza un modelo de procesamiento asíncrono optimizado para datasets de millones de registros."
date: 2026-02-27
classes: wide
header:
  teaser: /assets/images/sfdc-dx-bulk-api/logo.webp
  teaser_home_page: true
  icon: /assets/images/salesforce.png
categories:
  - salesforce
  - platform api
tags:
  - salesforce dx
  - sf
  - bulk api 2.0
---

# About

La **Bulk API 2.0** representa la evolución de la carga masiva en Salesforce, diseñada para maximizar el rendimiento y simplificar la experiencia del desarrollador. A diferencia de las APIs basadas en REST o SOAP tradicionales, **Bulk 2.0** utiliza un modelo de procesamiento **asíncrono** optimizado para datasets de millones de registros. Gestionando la segmentación de forma automática `(10,000 registros por lote)`.

Para esta PoC, Utilizaremos el mecanismo de **External ID** para desacoplar la carga de los IDs internos de Salesforce y permitir relaciones dinámicas.

![](/assets/images/sfdc-dx-bulk-api/flow.png)

- **Protocolo**: Basado en **REST** (HTTP/S).
- **Autenticación**: **OAuth 2.0**, La CLI (`sf`) gestiona de forma transparente el *Access Token* y la persistencia de la sesión mediante el alias de la organización.
- **Mecanismo de Ingesta**:  El cliente envía el archivo completo a un endpoint de carga; el servidor lo encola y procesa en **paralelo** según disponibilidad de recursos.
- **Orden**: **No determinista.** No se garantiza que los registros se procesen en el mismo orden que el archivo CSV.


# Contents

- [1. Environment Setup](#1-environment-setup)
    - [1.1 Org Connectivity Validation](#11-org-connectivity-validation)
    - [1.2 Metadata Mapping & Schema](#12-metadata-mapping--schema)
- [2. Execution Flow ](#2-execution-flow)
    - [2.1  Job Submission & Execution](#21--job-submission--execution)
    - [2.2 Job Monitoring](#22-job-monitoring)
    - [2.3 Outcome Analysis & Result Retrieval](#23-outcome-analysis--result-retrieval)
      - [2.3.1 Programmatic Retrieval (CLI)](#231-programmatic-retrieval-cli)
      - [2.3.2 Visual Monitoring (Salesforce UI)](#232-visual-monitoring-salesforce-ui)
- [3. Operational Governance & Limits](#3-operational-governance--limits)

# 1. Environment Setup

## 1.1 Org Connectivity Validation

Antes de iniciar, verifica que el alias de tu sandbox esté activo y autenticado:

```
sf org list
```
![](/assets/images/sfdc-dx-bulk-api/org-list.png)

## 1.2 Metadata Mapping & Schema

- **Configuración en Salesforce**:
    - Objeto `Account`: Debe existir un campo con API Name `Codigo_Cliente__c` marcado como External ID.
    - Datos Previos: Deben existir cuentas con los valores `ACC-101` y `ACC-102`.

- **Resolución de Relaciones**: El servidor resuelve el `AccountId` en tiempo de ejecución al buscar la coincidencia del External ID proporcionado en el CSV.

![](/assets/images/sfdc-dx-bulk-api/config-metadata.png)

Creamos un archivo `contacts.csv` con los API Names de los campos en la cabecera.

```csv
FirstName,LastName,Email,Account.Codigo_Cliente__c
Laura,Gomez,lgomez@test.com,ACC-101
Carlos,Ruiz,cruiz@test.com,ACC-102
Juan,Perez,jperez@test.com,ACC-101
```

# 2. Execution Flow 

## 2.1  Job Submission & Execution

Utilizamos `upsert` para que, si el contacto ya existe (basado en el Email, **evita duplicados**), se actualice; de lo contrario, se cree. El flag `--wait 0` permite que el comando regrese el control de la consola de inmediato mientras el servidor trabaja.

```
sf data upsert bulk --file data/contacts.csv --sobject Contact --external-id Email --wait 0
```

**Resultado**: `JobId (ej. 750...)`

## 2.2 Job Monitoring

Consulta el progreso del trabajo hasta que el estado cambie a `JobComplete`:

```
sf data import resume --job-id ID_DEL_JOB
```

![](/assets/images/sfdc-dx-bulk-api/job-monitoring.png)

## 2.3 Outcome Analysis & Result Retrieval

Una vez que el trabajo alcanza el estado `JobComplete`, se debe procedemos con la auditoría de resultados para garantizar la integridad de los datos.

### 2.3.1 Programmatic Retrieval (CLI)

Este comando nos permite descargar localmente dos archivos CSV `(success y failed)` para un análisis detallado. El archivo de fallos incluye una columna adicional con el `Error Message` específico de Salesforce para cada registro.

```
sf data bulk results --job-id ID_DEL_JOB
```
![](/assets/images/sfdc-dx-bulk-api/result.png)

### 2.3.2 Visual Monitoring (Salesforce UI)
Como alternativa de inspección rápida, Salesforce ofrece una consola de administración para supervisar el rendimiento del motor de Bulk API:

- **Ir a**: Setup > Quick Find > Bulk Data Load Jobs.
- **Metricas**: Permite visualizar el tiempo de procesamiento por lote, el número de registros "retried", usuario ejecutor y mas.

![](/assets/images/sfdc-dx-bulk-api/iu-result.png)

*Tanto los resultados en la CLI como en la interfaz de usuario tienen una ventana de persistencia de **7 días**. Tras este periodo, los metadatos del Job y sus resultados detallados son **eliminados permanentemente** de los servidores de Salesforce.*

# 3. Operational Governance & Limits

- **Volumen**: Máximo 150 millones de registros por periodo de 24 horas.
- **Payload**: El archivo CSV no debe exceder los 150 MB (sin comprimir).
- **Densidad de Datos**: Máximo 5,000 caracteres por campo.
- **Ventana de Procesamiento**: Los jobs caducan a las 24 horas.
- **Segmentación/Batches**: Los lotes automáticos son de máximo 10,000 registros.
- **Nivel de profundidad**: Limitado a un nivel de profundidad (ej. Contacto -> Cuenta).
- **Polimorfismo**: No compatible con campos polimórficos (ej. WhoId, WhatId).