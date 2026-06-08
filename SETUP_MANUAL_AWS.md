# Setup Manual — Cloud CV Mallorca (Consola AWS)

Guía paso a paso para desplegar toda la infraestructura desde la consola de AWS sin usar Terraform.  
Región recomendada: **us-east-1 (N. Virginia)**

---

## Índice

1. [DynamoDB — Tabla del contador](#1-dynamodb--tabla-del-contador)
2. [Lambda — Función serverless](#2-lambda--función-serverless)
3. [API Gateway — Endpoint REST](#3-api-gateway--endpoint-rest)
4. [Conectar API Gateway con Lambda](#4-conectar-api-gateway-con-lambda)
5. [Desplegar la API](#5-desplegar-la-api)
6. [AWS Amplify — Hosting del frontend](#6-aws-amplify--hosting-del-frontend)
7. [Verificación final](#7-verificación-final)
8. [Route 53 — Subdominio `cv.atercates.cat` y registro MX](#8-route-53--subdominio-cvatercatescat-y-registro-mx)

---

## 1. DynamoDB — Tabla del contador

**Servicio:** `DynamoDB` → **Tablas** → **Crear tabla**

| Campo | Valor |
|-------|-------|
| Nombre de la tabla | `cloud-cv-mallorca-visites` |
| Clave de partición | `id` (tipo **Cadena**) |
| Modo de capacidad | **Bajo demanda** (PAY_PER_REQUEST) |

En **Configuración adicional**:
- Activar **Recuperación a un momento dado** (Point-in-time recovery)
- Activar **Cifrado en reposo** (AWS managed key)

Haz clic en **Crear tabla**.

### Insertar el item inicial del contador

Una vez creada la tabla:

1. Abre la tabla → pestaña **Explorar los elementos de la tabla**
2. Haz clic en **Crear elemento**
3. Cambia la vista a **JSON** y pega:

```json
{
  "id": { "S": "contador_principal" },
  "visites": { "N": "0" }
}
```

4. Haz clic en **Crear elemento**

---

## 2. Lambda — Función serverless

### 2.1 Crear el archivo ZIP del código

En tu máquina local, comprime el fichero de la función:

```bash
cd backend/
zip lambda_function.zip lambda_function.py
```

### 2.2 Crear la función Lambda

**Servicio:** `Lambda` → **Crear una función**

| Campo | Valor |
|-------|-------|
| Opción | **Crear desde cero** |
| Nombre de la función | `cloud-cv-mallorca-counter` |
| Tiempo de ejecución | **Python 3.11** |
| Arquitectura | x86_64 |
| Rol de ejecución | **Usar un rol existente** → `LabRole` |

Haz clic en **Crear función**.

### 2.3 Subir el código

En la pestaña **Código** de la función recién creada:

1. **Origen del código** → **Cargar desde** → **.zip file**
2. Sube el archivo `lambda_function.zip`
3. El controlador debe quedar como: `lambda_function.lambda_handler`

### 2.4 Variables de entorno

Pestaña **Configuración** → **Variables de entorno** → **Editar** → **Agregar variable de entorno**:

| Clave | Valor |
|-------|-------|
| `DYNAMODB_TABLE` | `cloud-cv-mallorca-visites` |

Guarda los cambios.

### 2.5 Configuración general

Pestaña **Configuración** → **Configuración general** → **Editar**:

| Campo | Valor |
|-------|-------|
| Tiempo de espera | **10 segundos** |
| Memoria | **128 MB** |

### 2.6 Crear grupo de logs en CloudWatch (opcional pero recomendado)

**Servicio:** `CloudWatch` → **Grupos de registros** → **Crear grupo de registros**

| Campo | Valor |
|-------|-------|
| Nombre | `/aws/lambda/cloud-cv-mallorca-counter` |
| Período de retención | **14 días** |

> Lambda también lo crea automáticamente en la primera ejecución.

### 2.7 Probar la función Lambda

En la pestaña **Prueba** de la función, crea un evento de prueba con este JSON:

```json
{
  "httpMethod": "POST",
  "body": "{}"
}
```

Haz clic en **Probar**. Deberías ver `statusCode: 200` y el campo `visites` incrementado.

---

## 3. API Gateway — Endpoint REST

**Servicio:** `API Gateway` → **Crear API**

Elige **API REST** → **Compilar**

| Campo | Valor |
|-------|-------|
| Protocolo | REST |
| Crear nueva API | **API nueva** |
| Nombre | `cloud-cv-mallorca-api` |
| Descripción | `API per al comptador de visites del CV` |
| Tipo de endpoint | **Regional** |

Haz clic en **Crear API**.

### 3.1 Crear el recurso `/visites`

En el panel de la API → **Acciones** → **Crear recurso**

| Campo | Valor |
|-------|-------|
| Nombre del recurso | `visites` |
| Ruta del recurso | `/visites` |
| Habilitar CORS | NO (lo configuramos manualmente) |

Haz clic en **Crear recurso**.

---

## 4. Conectar API Gateway con Lambda

Con el recurso `/visites` seleccionado en el árbol de recursos:

### 4.1 Método POST

**Acciones** → **Crear método** → selecciona `POST` → confirma con el tick ✔

| Campo | Valor |
|-------|-------|
| Tipo de integración | **Función Lambda** |
| Usar integración de proxy de Lambda | ✅ Activado |
| Región | `us-east-1` |
| Función Lambda | `cloud-cv-mallorca-counter` |

Haz clic en **Guardar** → confirma el permiso para que API Gateway invoque Lambda.

### 4.2 Método GET

Repite el mismo proceso: **Acciones** → **Crear método** → `GET`

| Campo | Valor |
|-------|-------|
| Tipo de integración | **Función Lambda** |
| Usar integración de proxy de Lambda | ✅ Activado |
| Función Lambda | `cloud-cv-mallorca-counter` |

### 4.3 Método OPTIONS (CORS preflight)

**Acciones** → **Crear método** → `OPTIONS`

| Campo | Valor |
|-------|-------|
| Tipo de integración | **Simulacro (MOCK)** |

Haz clic en **Guardar**.

Ahora configura la respuesta de integración de OPTIONS:

1. Haz clic en **Respuesta de integración** → expande la fila `200`
2. En **Asignaciones de encabezado de respuesta** añade:

| Nombre del encabezado | Valor de asignación |
|-----------------------|---------------------|
| `Access-Control-Allow-Headers` | `'Content-Type,X-Amz-Date,Authorization,X-Api-Key'` |
| `Access-Control-Allow-Methods` | `'GET,POST,OPTIONS'` |
| `Access-Control-Allow-Origin` | `'*'` |

3. Haz clic en **Guardar**.

Después configura la respuesta de método de OPTIONS:

1. Haz clic en **Respuesta de método** → expande `200`
2. En **Encabezados de respuesta** añade los tres mismos encabezados:
   - `Access-Control-Allow-Headers`
   - `Access-Control-Allow-Methods`
   - `Access-Control-Allow-Origin`

### 4.4 Habilitar CORS en GET y POST

Para los métodos GET y POST con proxy Lambda, los headers CORS ya los devuelve la propia función Lambda (están en `get_cors_headers()` en el código Python). No es necesario configurarlos en API Gateway si usas integración proxy.

---

## 5. Desplegar la API

### 5.1 Crear el despliegue

**Acciones** → **Implementar API**

| Campo | Valor |
|-------|-------|
| Etapa de implementación | **[Nueva etapa]** |
| Nombre de la etapa | `prod` |

Haz clic en **Implementar**.

### 5.2 Anotar la URL

Al finalizar verás la **URL de invocación**, con este formato:

```
https://<API_ID>.execute-api.us-east-1.amazonaws.com/prod
```

El endpoint completo del contador es:

```
https://<API_ID>.execute-api.us-east-1.amazonaws.com/prod/visites
```

**Guarda esta URL**, la necesitarás para el frontend.

---

## 6. AWS Amplify — Hosting del frontend

> Amplify conecta directamente con GitHub y despliega automáticamente en cada push.  
> El fichero `amplify.yml` en la raíz del repositorio ya está configurado para inyectar la URL de la API.

**Servicio:** `AWS Amplify` → **Crear nueva app** → **Alojar aplicación web**

### 6.1 Conectar el repositorio

1. Elige **GitHub** como proveedor
2. Autoriza el acceso a GitHub si es la primera vez
3. Selecciona tu repositorio y la rama `main`
4. Haz clic en **Siguiente**

### 6.2 Configuración de compilación

Amplify detectará automáticamente el fichero `amplify.yml` de la raíz del repo. Verifica que la configuración sea:

```yaml
version: 1
applications:
  - appRoot: frontend
    frontend:
      phases:
        build:
          commands:
            - sed -i "s|__API_ENDPOINT__|$API_ENDPOINT|g" js/counter.js
      artifacts:
        baseDirectory: .
        files:
          - '**/*'
      cache:
        paths: []
```

### 6.3 Variables de entorno

En la pantalla de configuración (o después desde **App settings** → **Variables de entorno**):

| Variable | Valor |
|----------|-------|
| `API_ENDPOINT` | `https://<API_ID>.execute-api.us-east-1.amazonaws.com/prod/visites` |

Haz clic en **Siguiente** → **Guardar e implementar**.

### 6.4 Esperar el despliegue

El primer despliegue tarda 1-3 minutos. Una vez finalizado, Amplify muestra la URL pública:

```
https://main.<APP_ID>.amplifyapp.com
```

### 6.5 Redespliegues automáticos

A partir de ahora, cada `git push` a la rama `main` desencadena un redespliegue automático.

---

## 7. Verificación final

### Probar la API directamente

```bash
# Obtener el contador actual (GET)
curl https://<API_ID>.execute-api.us-east-1.amazonaws.com/prod/visites

# Incrementar el contador (POST)
curl -X POST https://<API_ID>.execute-api.us-east-1.amazonaws.com/prod/visites \
  -H "Content-Type: application/json" -d '{}'
```

Respuesta esperada:

```json
{
  "visites": 1,
  "message": "Comptador incrementat correctament"
}
```

### Verificar el frontend

1. Abre la URL de Amplify en el navegador
2. El contador debe aparecer con el número actual de visitas
3. Recarga la página — el número debe incrementarse

### Verificar los logs de Lambda

**CloudWatch** → **Grupos de registros** → `/aws/lambda/cloud-cv-mallorca-counter`

Cada invocación genera un log con el evento recibido y el resultado.

---

## 8. Route 53 — Subdominio `cv.atercates.cat` y registro MX

Esta sección explica cómo apuntar `cv.atercates.cat` a tu app de Amplify usando Route 53 como DNS, y cómo añadir un registro MX al dominio.

> **Prerequisito:** el dominio `atercates.cat` debe estar registrado y sus nameservers deben apuntar a Route 53 (o debes delegar solo el subdominio desde tu proveedor actual).

---

### 9.1 Crear la zona alojada en Route 53 (si no existe)

**Servicio:** `Route 53` → **Zonas alojadas** → **Crear zona alojada**

| Campo | Valor |
|-------|-------|
| Nombre de dominio | `atercates.cat` |
| Tipo | **Zona alojada pública** |

Haz clic en **Crear zona alojada**.

Route 53 generará 4 registros NS, por ejemplo:

```
ns-123.awsdns-45.com.
ns-678.awsdns-90.net.
ns-111.awsdns-22.org.
ns-222.awsdns-33.co.uk.
```

**En tu registrador de dominio** (donde compraste `atercates.cat`), sustituye los nameservers actuales por estos 4. Los cambios DNS pueden tardar hasta 48 h en propagarse.

> Si `atercates.cat` ya está en Route 53, omite este paso y usa la zona existente.

---

### 9.2 Delegar solo el subdominio (alternativa sin mover el dominio)

Si no quieres mover toda la zona de `atercates.cat` a Route 53, puedes delegar únicamente `cv.atercates.cat`:

1. Crea una zona alojada pública en Route 53 con nombre **`cv.atercates.cat`**
2. Route 53 te dará 4 NS para esa zona
3. En tu DNS actual de `atercates.cat`, añade un registro NS:

| Nombre | Tipo | Valor |
|--------|------|-------|
| `cv.atercates.cat` | `NS` | Los 4 NS que dio Route 53 |

Con esto, Route 53 gestiona todo lo relativo a `cv.atercates.cat` y el resto del dominio sigue en tu DNS actual.

---

### 9.3 Añadir el dominio personalizado en Amplify

**Servicio:** `AWS Amplify` → tu app → **App settings** → **Dominios personalizados** → **Añadir dominio**

1. Escribe `atercates.cat` y haz clic en **Configurar dominio**
2. En **Subdominios**, añade:

| Prefijo del subdominio | Rama |
|------------------------|------|
| `cv` | `main` |

3. Haz clic en **Guardar**

Amplify mostrará los registros DNS que necesitas crear. Serán de dos tipos:

- Un registro **CNAME** para verificar la propiedad del dominio
- Un registro **CNAME** para apuntar `cv.atercates.cat` a Amplify

Copia ambos valores — los añadirás en el siguiente paso.

---

### 9.4 Crear los registros DNS en Route 53

**Servicio:** `Route 53` → **Zonas alojadas** → `atercates.cat` (o `cv.atercates.cat`) → **Crear registro**

#### Registro de verificación (lo indica Amplify)

| Campo | Valor |
|-------|-------|
| Nombre | El que indica Amplify (p. ej. `_abc123.cv`) |
| Tipo | `CNAME` |
| Valor | El que indica Amplify (p. ej. `_xyz.acm-validations.aws`) |
| TTL | `300` |

#### Registro del subdominio

| Campo | Valor |
|-------|-------|
| Nombre | `cv` |
| Tipo | `CNAME` |
| Valor | El dominio de Amplify (p. ej. `main.<APP_ID>.amplifyapp.com`) |
| TTL | `300` |

Haz clic en **Crear registros**.

Una vez propagados, Amplify completará la verificación automáticamente (puede tardar entre 5 minutos y 1 hora) y el sitio estará accesible en:

```
https://cv.atercates.cat
```

Amplify provisiona el certificado SSL/TLS automáticamente vía ACM — no necesitas configurar nada más para HTTPS.

---

### 9.5 Añadir un registro MX

El registro MX indica qué servidores de correo gestionan el email de `atercates.cat` (o `cv.atercates.cat`).

**Route 53** → **Zonas alojadas** → `atercates.cat` → **Crear registro**

| Campo | Valor |
|-------|-------|
| Nombre | *(vacío, para `atercates.cat`)* o `cv` (para `cv.atercates.cat`) |
| Tipo | `MX` |
| TTL | `3600` |
| Valor | Prioridad + servidor de correo |

El campo **Valor** tiene el formato `<prioridad> <servidor>`. Ejemplos según proveedor:

**Google Workspace:**
```
1 ASPMX.L.GOOGLE.COM.
5 ALT1.ASPMX.L.GOOGLE.COM.
5 ALT2.ASPMX.L.GOOGLE.COM.
10 ALT3.ASPMX.L.GOOGLE.COM.
10 ALT4.ASPMX.L.GOOGLE.COM.
```

**Microsoft 365:**
```
0 atercates-cat.mail.protection.outlook.com.
```

**Proton Mail:**
```
10 mail.protonmail.ch.
20 mailsec.protonmail.ch.
```

Pon todos los registros MX en el mismo bloque de valor (uno por línea). Haz clic en **Crear registros**.

> El número más bajo en la prioridad = servidor preferido. Si solo tienes un servidor de correo, usa prioridad `10`.

---

### 9.6 Verificar la propagación DNS

```bash
# Comprobar que cv.atercates.cat resuelve correctamente
dig cv.atercates.cat CNAME

# Comprobar los registros MX
dig atercates.cat MX

# Alternativamente desde cualquier máquina
nslookup -type=MX atercates.cat
```

---

## Resumen de recursos creados

| Servicio | Nombre / ID |
|----------|-------------|
| DynamoDB | `cloud-cv-mallorca-visites` |
| Lambda | `cloud-cv-mallorca-counter` |
| API Gateway | `cloud-cv-mallorca-api` (stage: `prod`) |
| Amplify | App conectada a rama `main` de GitHub |
| CloudWatch Logs | `/aws/lambda/cloud-cv-mallorca-counter` (14 días) |
| IAM Role | `LabRole` (preexistente) |
| Route 53 | Zona alojada `atercates.cat` |
| DNS | `cv.atercates.cat` → Amplify (CNAME) |
| DNS | Registro MX en `atercates.cat` |
