# PixelCraft AI Email Manager

Sistema de automatización inteligente para la gestión de emails de clientes de **PixelCraft Studio**, desarrollado como proyecto final de automatización e inteligencia artificial.

El sistema recibe automáticamente emails de clientes, utiliza inteligencia artificial para clasificarlos y generar una respuesta personalizada utilizando información almacenada en una base de datos, y solicita aprobación humana antes de enviar cualquier respuesta al cliente.

##  Descripción del proyecto

PixelCraft AI Email Manager automatiza el proceso de atención inicial de emails de una agencia ficticia de diseño y desarrollo web.

El objetivo es reducir el trabajo manual del equipo de atención al cliente manteniendo un punto de control humano antes de cualquier comunicación automática con un cliente.

El sistema es capaz de:

* Recibir emails automáticamente.
* Validar que el mensaje contenga información.
* Registrar cada email en una base de datos.
* Analizar y clasificar el mensaje mediante IA.
* Determinar categoría, prioridad e intención.
* Generar un resumen del email.
* Consultar los servicios y precios disponibles.
* Generar una respuesta personalizada.
* Solicitar aprobación humana.
* Enviar la respuesta únicamente si es aprobada.
* Registrar el resultado final en la base de datos.
* Gestionar rechazos y casos de error.

# Arquitectura

```text
                         ┌──────────────────┐
                         │      CLIENTE     │
                         │  Envía un email  │
                         └────────┬─────────┘
                                  │
                                  ▼
                         ┌──────────────────┐
                         │      GMAIL       │
                         │   Trigger n8n    │
                         └────────┬─────────┘
                                  │
                                  ▼
                         ┌──────────────────┐
                         │ VALIDACIÓN EMAIL │
                         └────────┬─────────┘
                                  │
                         ┌────────┴────────┐
                         │                 │
                       ERROR               OK
                         │                 │
                         ▼                 ▼
                  ┌─────────────┐   ┌─────────────┐
                  │   AIRTABLE  │   │   OPENAI    │
                  │ Error       │   │ Clasifica   │
                  └─────────────┘   └──────┬──────┘
                                           │
                                           ▼
                                  ┌─────────────────┐
                                  │    AIRTABLE     │
                                  │ Servicios / DB  │
                                  └────────┬────────┘
                                           │
                                           ▼
                                  ┌─────────────────┐
                                  │     OPENAI      │
                                  │ Genera respuesta│
                                  └────────┬────────┘
                                           │
                                           ▼
                                  ┌─────────────────┐
                                  │ SOLICITUD DE    │
                                  │  APROBACIÓN     │
                                  └────────┬────────┘
                                           │
                              ┌────────────┴────────────┐
                              │                         │
                           APROBAR                   RECHAZAR
                              │                         │
                              ▼                         ▼
                       ┌─────────────┐          ┌─────────────┐
                       │    GMAIL    │          │   AIRTABLE  │
                       │ Respuesta   │          │ Rechazado   │
                       │ al cliente  │          └─────────────┘
                       └──────┬──────┘
                              │
                              ▼
                       ┌─────────────┐
                       │   AIRTABLE  │
                       │  Respondido │
                       └─────────────┘
```
#  Tecnologías utilizadas

| Tecnología   | Uso                                            |
| ------------ | ---------------------------------------------- |
| **n8n**      | Orquestación y automatización de los workflows |
| **Gmail**    | Recepción y envío de emails                    |
| **OpenAI**   | Clasificación y generación de respuestas       |
| **Airtable** | Base de datos y memoria del sistema            |
| **GitHub**   | Versionado y documentación del proyecto        |


# Workflows

El proyecto está dividido en **dos workflows principales** para implementar correctamente el proceso de aprobación humana.

## Workflow A — Procesamiento del email

Este workflow se ejecuta automáticamente cuando Gmail recibe un nuevo email.

### Flujo

```text
Gmail Trigger
     ↓
Ignorar emails de aprobación
     ↓
Validar email
     ↓
Guardar email en Airtable
     ↓
OpenAI - Analizar email
     ↓
Actualizar clasificación
     ↓
Consultar servicios
     ↓
Preparar información
     ↓
OpenAI - Generar respuesta
     ↓
Guardar respuesta IA
     ↓
Solicitar aprobación
```

### 1. Recepción

El workflow se inicia mediante un **Gmail Trigger**.

El email recibido contiene información como:

* Remitente
* Asunto
* Mensaje
* Fecha
* Identificador del mensaje

---

### 2. Prevención de loops

Antes de procesar el mensaje se verifica que no sea un email generado por el propio sistema de aprobación.

Los emails cuyo asunto contiene:

```text
[APROBACIÓN]
```

son ignorados.

Esto evita que las respuestas de aprobación vuelvan a entrar al Workflow A y generen nuevos procesos automáticamente.

### 3. Validación

Se verifica que el email contenga un mensaje.

Si el mensaje está vacío:

```text
Estado = Error
Error = El email no contiene un mensaje
```

El proceso no continúa hacia la IA.

### 4. Persistencia en Airtable

El email se almacena en la tabla `Emails`.

Los principales campos utilizados son:

* Cliente
* Email
* Fecha
* Asunto
* Mensaje
* Categoría
* Prioridad
* Intención
* Resumen IA
* Respuesta IA
* Estado
* Error

### 5. Clasificación mediante OpenAI

OpenAI analiza el email y devuelve un JSON estructurado.

Las categorías posibles son:

* `Consulta comercial`
* `Presupuesto`
* `Soporte`
* `Reclamo`
* `Otro`

Las prioridades posibles son:

* `Baja`
* `Media`
* `Alta`

Además se obtiene:

* Intención del cliente
* Resumen del mensaje

El modelo está configurado para devolver únicamente información presente en el email y evitar la invención de datos.

### 6. Consulta de servicios

La tabla `Servicios` funciona como fuente de información para las respuestas generadas por la IA.

Actualmente contiene servicios como:

| Servicio        | Precio |
| --------------- | -----: |
| Landing page    |   $300 |
| Web empresarial |   $600 |
| Tienda online   |   $900 |
| Mantenimiento   |   $100 |
| SEO básico      |   $200 |

Los datos se consultan dinámicamente desde Airtable.

La IA no recibe estos precios como información fija dentro del workflow, sino que los obtiene desde la base de datos.

### 7. Generación de respuesta

Un segundo proceso de OpenAI utiliza:

* Email original
* Categoría
* Prioridad
* Intención
* Resumen
* Servicios disponibles
* Precios registrados en Airtable

para generar una respuesta profesional y personalizada.

La IA tiene como regla no inventar:

* Servicios
* Precios
* Características
* Plazos

Si un servicio solicitado no existe en la base de datos, la respuesta indica que debe ser consultado con el equipo antes de confirmar un precio.

### 8. Solicitud de aprobación

Antes de enviar la respuesta al cliente, el sistema genera un email de aprobación para el responsable.

El asunto contiene el identificador único del registro de Airtable:

```text
[APROBACIÓN] [ID_AIRTABLE] Nueva respuesta para ...
```

Esto permite identificar exactamente qué registro debe procesarse.

# Human-in-the-Loop

Una característica fundamental del proyecto es el mecanismo **Human-in-the-Loop (HITL)**.

La IA puede analizar emails y generar respuestas, pero **no puede enviar automáticamente una respuesta al cliente sin autorización humana**.

El responsable puede responder:

```text
APROBAR
```

o:

```text
RECHAZAR
```

#  Workflow B — Aprobación humana

El segundo workflow se activa cuando Gmail recibe la respuesta del responsable.

### Flujo

```text
Gmail Trigger
     ↓
Extraer datos de aprobación
     ↓
Detectar decisión
     │
     ├── APROBAR
     │      ↓
     │ Obtener email original
     │      ↓
     │ Marcar como Aprobado
     │      ↓
     │ Enviar respuesta al cliente
     │      ↓
     │ Marcar como Respondido
     │
     └── RECHAZAR
            ↓
       Marcar como Rechazado
```

##  Identificación mediante Record ID

Para evitar asociar una aprobación con el email incorrecto, el sistema utiliza el `Record ID` de Airtable.

Ejemplo:

```text
[APROBACIÓN] [recuAO9GTeOzsnVa1] Nueva respuesta...
```

n8n extrae:

```text
recordId = recuAO9GTeOzsnVa1
```

y utiliza ese identificador para obtener directamente el registro correspondiente.

Esto evita utilizar el asunto del email como identificador único.

#  Base de datos

La base de datos de Airtable se denomina:

```text
PixelCraft AI CRM
```

## Tabla `Emails`

Contiene los emails procesados por el sistema.

### Estados

```text
Pendiente
Procesando
Procesado por IA
Esperando aprobacion
Aprobado
Rechazado
Respondido
Error
```

El flujo normal de un email aprobado es:

```text
Procesado por IA
       ↓
Esperando aprobacion
       ↓
Aprobado
       ↓
Respondido
```

En caso de rechazo:

```text
Procesado por IA
       ↓
Esperando aprobacion
       ↓
Rechazado
```

## Tabla `Servicios`

Contiene la información comercial utilizada por la IA.

Campos:

* Servicio
* Descripción
* Precio

Esta tabla funciona como una fuente de conocimiento externa al prompt, permitiendo modificar precios o servicios sin tener que modificar la lógica del workflow.

#  Seguridad y prevención de errores

El sistema incorpora diferentes mecanismos para reducir errores.

### Prevención de loops

Los emails de aprobación contienen `[APROBACIÓN]` y son excluidos del Workflow A.

### Validación de datos

Los emails sin contenido son detectados antes de enviarse a OpenAI.

### Identificación mediante ID

Las aprobaciones se relacionan utilizando el Record ID de Airtable en lugar del asunto.

### Respuestas estructuradas

La clasificación de OpenAI utiliza un esquema JSON con valores permitidos para categorías y prioridades.

### Información dinámica

Los precios y servicios se obtienen desde Airtable en tiempo de ejecución.

### Control humano

Las respuestas generadas por IA requieren aprobación antes de ser enviadas.

---

#  Pruebas

El sistema debe probarse con diferentes tipos de emails para comprobar tanto los casos normales como los casos de error.

### Casos de prueba sugeridos

| # | Caso                     | Resultado esperado                    |
| - | ------------------------ | ------------------------------------- |
| 1 | Consulta comercial       | Clasificación + respuesta             |
| 2 | Solicitud de presupuesto | Clasificación + precio desde Airtable |
| 3 | Problema de soporte      | Prioridad y respuesta adecuada        |
| 4 | Aprobación               | Email enviado al cliente              |
| 5 | Rechazo                  | Estado `Rechazado`                    |
| 6 | Email sin contenido      | Estado `Error`                        |
| 7 | Servicio inexistente     | No inventar precio                    |

---

#  Ejemplo de procesamiento

Un cliente puede enviar:

```text
Asunto:
Consulta tienda online

Mensaje:
Hola, estoy interesado en crear una tienda online para mi negocio.
Quería saber qué precio tiene y qué incluye.
Saludos.
```

OpenAI puede determinar:

```text
Categoría: Presupuesto
Prioridad: Media
Intención: Solicitar precio y alcance para crear una tienda online
```

Airtable proporciona:

```text
Tienda online
Precio: $900
```

La IA genera una respuesta utilizando esa información.

Posteriormente el responsable recibe una solicitud de aprobación.

Si responde:

```text
APROBAR
```

la respuesta se envía al cliente y el registro pasa a:

```text
Respondido
```

Si responde:

```text
RECHAZAR
```

el registro pasa a:

```text
Rechazado
```

# Estructura del repositorio

```text
pixelcraft-ai-automation/
│
├── README.md
│
├── workflow/
│   ├── workflow-procesamiento-email.json
│   └── workflow-aprobacion.json
│
├── docs/
│   ├── arquitectura.pdf
│   └── pruebas.pdf
│
├── screenshots/
│   ├── 01-workflow-principal.png
│   ├── 02-airtable.png
│   ├── 03-clasificacion-openai.png
│   ├── 04-generacion-respuesta.png
│   ├── 05-solicitud-aprobacion.png
│   ├── 06-aprobacion.png
│   ├── 07-rechazo.png
│   └── 08-error.png
│
└── demo/
    └── video-link.txt
```

#  Demostración

La demostración del proyecto muestra el funcionamiento completo:

1. Envío de un email de cliente.
2. Activación automática de n8n.
3. Clasificación mediante OpenAI.
4. Consulta de servicios en Airtable.
5. Generación de respuesta.
6. Solicitud de aprobación.
7. Aprobación humana.
8. Envío automático al cliente.
9. Actualización del estado en Airtable.

También se demuestra el flujo alternativo de rechazo.

#  Instalación y configuración

## Requisitos

* Cuenta de n8n
* Cuenta de Gmail
* Cuenta de Airtable
* API/credenciales de OpenAI
* Base de Airtable configurada
* Workflows de n8n importados

## Configuración

1. Crear la base `PixelCraft AI CRM` en Airtable.
2. Crear las tablas `Emails` y `Servicios`.
3. Cargar los servicios disponibles.
4. Configurar las credenciales de Gmail.
5. Configurar las credenciales de OpenAI.
6. Configurar las credenciales de Airtable.
7. Importar los workflows de n8n.
8. Verificar las expresiones dinámicas.
9. Ejecutar las pruebas.
10. Activar los workflows.

# Buenas prácticas

Las credenciales y claves API no deben almacenarse dentro del repositorio.

No se deben subir:

```text
API Keys
Tokens
Contraseñas
Credenciales OAuth
Datos personales reales de clientes
```

Las credenciales deben gestionarse mediante las credenciales seguras de n8n y las configuraciones correspondientes de cada servicio.

#  Objetivos alcanzados

El proyecto implementa un ecosistema de automatización que integra:

*  Orquestación mediante n8n
* Inteligencia artificial mediante OpenAI
*  Entrada y salida mediante Gmail
*  Memoria mediante Airtable
*  Procesamiento dinámico de información
*  Human-in-the-Loop
*  Validación de datos
*  Prevención de loops
*  Manejo de estados
*  Manejo de errores
*  Respuestas generadas dinámicamente
*  Identificación de registros mediante ID
*  Flujo de aprobación y rechazo

#  Proyecto académico

**Proyecto:** Ecosistema de Automatización IA Autónomo para Negocios
**Caso de uso:** Gestión inteligente de emails para agencia de diseño web
**Empresa ficticia:** PixelCraft Studio

El proyecto fue desarrollado con el objetivo de demostrar la integración de herramientas de automatización, bases de datos e inteligencia artificial dentro de un flujo empresarial realista.

##  Estado del proyecto

**Estado:** Funcional

El sistema cuenta con el flujo principal de procesamiento, generación de respuestas mediante IA, consulta de información comercial, aprobación humana, envío automático y registro del resultado en Airtable.
