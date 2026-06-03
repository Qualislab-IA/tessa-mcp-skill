# TESSA MCP — Skill

**TESSA** (Test Execution & Smart Synthesis Agent) es una plataforma SaaS que genera casos de prueba con IA y los ejecuta vía agentes conectados por MCP. Esta skill te enseña a interactuar correctamente con el servidor MCP de TESSA.

## Cuándo usar esta skill

Usala cuando el usuario pida cualquiera de estas cosas:

- "Ejecutá el test case N" / "Corré el caso de prueba N en [URL]"
- "Listá mis proyectos de TESSA" / "¿Qué casos tiene el proyecto X?"
- "¿Qué pasos tiene el caso N?"
- "Probá el happy path de [proyecto] en staging"
- "Subí estos screenshots al proceso N" / "Reportá los resultados"
- Cualquier mención de TESSA, QualisLab, o test cases automatizados en este contexto.

## Servidor

- **URL de producción**: `https://agent.qualis-lab.com/mcp`
- **Autenticación**: Bearer token con prefijo `qai_` (API token de TESSA).
- Los 8 tools se descubren automáticamente vía `tools/list` al conectarse.

## Los 8 tools

### 1. `list_projects`

Lista paginada de los **proyectos** a los que el usuario autenticado tiene acceso (id + nombre real del proyecto). El acceso se resuelve server-side: solo ves proyectos de tu empresa donde sos miembro.

**Input**:
```json
{ "page": 1, "pageSize": 10 }
```

**Output**:
```json
{
  "projects": [
    { "id": 12, "name": "Checkout Web" },
    { "id": 9, "name": "App Móvil - Pagos" }
  ],
  "pagination": {
    "currentPage": 1, "pageSize": 10,
    "totalItems": 4, "totalPages": 1,
    "hasNextPage": false, "hasPreviousPage": false
  }
}
```

**Usalo primero** cuando el usuario no especifica un `caseId` concreto. Identificá el proyecto y luego listá sus casos con `list_test_cases`.

### 2. `list_test_cases`

Lista paginada de los **casos de prueba** (internamente "processes") de un proyecto al que el usuario tiene acceso. Requiere el `projectId` obtenido de `list_projects`. Devuelve el nombre del proyecto y los casos en estado `PROCESADO` (listos para ejecutar).

**Input**:
```json
{ "projectId": 12, "page": 1, "pageSize": 10 }
```

**Output**:
```json
{
  "projectId": 12,
  "projectName": "Checkout Web",
  "cases": [
    { "caseId": 375, "title": "Test QR Payment Flow with Predefined Amount", "status": "PROCESADO" },
    { "caseId": 374, "title": "Checkout con cupón de descuento", "status": "PROCESADO" }
  ],
  "pagination": {
    "currentPage": 1, "pageSize": 10,
    "totalItems": 23, "totalPages": 3,
    "hasNextPage": true, "hasPreviousPage": false
  }
}
```

El `caseId` de cada caso es el ID que usan `fetch_test_case`, `fetch_additional_cases`, `get_presigned_url` y `submit_test_result`. Mostrale la lista al usuario y pedí confirmación antes de ejecutar.

### 3. `fetch_test_case`

Detalle del happy path de un test case.

**Input**:
```json
{ "caseId": "375" }
```

**Output**:
```json
{
  "testCaseId": 375,
  "title": "Test QR Payment Flow with Predefined Amount",
  "steps": [
    { "stepNumber": 1, "action": "Scan QR code with device camera" },
    { "stepNumber": 2, "action": "Verify predefined amount matches expected value" }
  ]
}
```

Los `steps` son **descripciones en lenguaje natural**. Vos (la IA) sos la responsable de traducirlas en acciones concretas del browser (click, type, wait, screenshot).

### 4. `fetch_additional_cases`

Casos alternativos asociados (edge cases, flujos de error, validaciones extra).

**Input**:
```json
{ "caseId": "375" }
```

**Output**:
```json
{
  "testCaseId": 375,
  "totalAdditionalCases": 4,
  "additionalCases": [
    {
      "id": 1,
      "title": "QR expirado",
      "precondition": "El código QR fue generado hace más de 5 minutos",
      "classification": "negative",
      "testType": "Functional",
      "cases": [
        { "stepNumber": 1, "description": "Escanear QR expirado" },
        { "stepNumber": 2, "description": "Verificar mensaje de error" }
      ],
      "expectedResults": { "message": "QR code expired" }
    }
  ]
}
```

Llamalo **después de `fetch_test_case`** si el usuario pidió explícitamente "ejecutá todos los casos" o si el happy path falló y hay que probar el camino negativo.

### 5. `get_presigned_url`

Genera una URL firmada para subir un screenshot directamente a S3. **Úsalo SIEMPRE antes de subir screenshots** — no envíes imágenes en base64 inline.

**Input**:
```json
{
  "fileName": "step-3-error.png",
  "contentType": "image/png",
  "caseId": "375"
}
```

**Output**:
```json
{
  "uploadUrl": "https://bucket.s3.amazonaws.com/executions/375/uuid_step-3-error.png?X-Amz-Algorithm=...",
  "publicUrl": "https://bucket.s3.amazonaws.com/executions/375/uuid_step-3-error.png"
}
```

- `contentType` debe ser uno de: `image/png`, `image/jpeg`, `image/jpg`, `image/webp`. Cualquier otro es rechazado.
- `caseId` es **opcional** pero **pásalo siempre que sea posible** — organiza las imágenes por proceso y valida ownership.
- Hacé un `PUT` HTTP al `uploadUrl` con el binario crudo del screenshot (no form-data) y el header `Content-Type` correcto.
- Guardá el `publicUrl` para pasarlo después a `submit_test_result`.

### 6. `submit_test_result`

Reporta el resultado final de la ejecución.

**Input**:
```json
{
  "caseId": "375",
  "status": "PASS",
  "executedUrl": "https://staging.example.com/payments",
  "totalDurationMs": 8420,
  "steps": [
    {
      "stepNumber": 1,
      "description": "Scan QR code",
      "status": "PASS",
      "durationMs": 1200,
      "screenshotUrl": "https://bucket.s3.amazonaws.com/executions/375/uuid_step-1.png"
    },
    {
      "stepNumber": 2,
      "description": "Verify amount",
      "status": "FAIL",
      "durationMs": 800,
      "screenshotUrl": "https://bucket.s3.amazonaws.com/executions/375/uuid_step-2-error.png",
      "errorMessage": "Expected $500, got $550"
    }
  ]
}
```

Reglas:
- `status` debe ser uno de: `PASS`, `FAIL`, `ERROR`, `SKIPPED`.
- Si al menos un step falló, el `status` general no puede ser `PASS`.
- `screenshotUrl` debe ser un `publicUrl` obtenido de `get_presigned_url`. **Nunca** inventes URLs ni mandes base64.
- `errorMessage` solo en steps con status distinto de `PASS`.
- Solo podés reportar sobre procesos de los que sos dueño (validación server-side).

### 7. `create_analysis_draft`

**Paso 1 de la generación de casos a partir de un documento funcional.** Crea un proceso en estado `DRAFT` y devuelve una URL firmada de S3 (`PUT`) para subir el documento. (Tool **solo MCP**: no tiene equivalente REST.)

**Input**:
```json
{
  "projectId": 12,
  "fileName": "spec-checkout.pdf",
  "contentType": "application/pdf"
}
```

**Output**:
```json
{
  "processId": 481,
  "uploadUrl": "https://bucket.s3.amazonaws.com/processes/481/uuid_spec-checkout.pdf?X-Amz-Algorithm=...",
  "publicUrl": "https://bucket.s3.amazonaws.com/processes/481/uuid_spec-checkout.pdf"
}
```

- `fileName`: máximo 255 caracteres.
- `contentType` debe ser **uno de**: `application/pdf` o `application/vnd.openxmlformats-officedocument.wordprocessingml.document` (docx). Cualquier otro es rechazado.
- Después de llamarlo, hacé un `PUT` HTTP del documento al `uploadUrl` con el header `Content-Type` que coincida con el `contentType` enviado.
- **Gotcha**: la URL está firmada con solo los headers `host;content-type`. **No envíes ningún header `x-amz-checksum-*`** (aunque aparezca en el query string) o S3 devuelve `403`.
- Valida acceso al proyecto + permiso `CREATE_EXECUTIONS` del lado del servidor.
- Guardá el `publicUrl` para pasarlo a `generate_analysis` en el paso 2.

### 8. `generate_analysis`

**Paso 2** (después de `create_analysis_draft` + subir el archivo). Genera casos de prueba de forma **asíncrona** a partir del documento subido. (Tool **solo MCP**: no tiene equivalente REST.)

**Input**:
```json
{
  "processId": 481,
  "fileUrl": "https://bucket.s3.amazonaws.com/processes/481/uuid_spec-checkout.pdf",
  "documentType": "pdf",
  "originalName": "spec-checkout.pdf",
  "prompt": "Enfocate en los flujos de pago con tarjeta",
  "industry": "Fintech",
  "functionality": "Checkout",
  "platform": "Web",
  "additionals": true,
  "gherkin": false,
  "uxUi": false
}
```

**Output**:
```json
{
  "processId": 481,
  "message": "Generation started"
}
```

- `fileUrl`: máximo 2000 caracteres. **Debe ser el `publicUrl`** devuelto por `create_analysis_draft` y apuntar a la carpeta de este mismo proceso (validación anti-SSRF del lado del servidor).
- `documentType` debe ser `pdf` o `word`.
- `originalName` (opcional), `prompt` (opcional, máximo 5000 caracteres), `industry`, `functionality`, `platform` (todos opcionales).
- `additionals`, `gherkin`, `uxUi` (opcionales, default `false`): flags que controlan qué tipos de casos se generan además del happy path.
- El proceso debe estar en estado `DRAFT` y pertenecerte (validación server-side). Usa el proveedor LLM activo de la empresa.
- La generación es **asíncrona**: el output solo confirma que arrancó. **Poleá después** (ej. con `list_test_cases`) hasta que el caso esté en estado `PROCESADO`.

## Flujo recomendado (end-to-end)

```
1. (Si no hay caseId)  → list_projects               (proyectos accesibles)
                        → list_test_cases(projectId)  (casos del proyecto elegido)
                        → mostrar al usuario y pedir confirmación

2. fetch_test_case     → obtener pasos del happy path
3. (Opcional)
   fetch_additional_cases → obtener casos alternativos si aplica

4. PRE-EJECUCIÓN:
   Confirmar con el usuario: URL objetivo, credenciales si aplican,
   y si aprueba la ejecución.

5. EJECUCIÓN (por cada step):
   a. Traducir el step.action a acciones concretas del browser
      (navigate, click, fill, wait, etc.)
   b. Ejecutar. Medir tiempo (durationMs).
   c. Tomar screenshot del resultado (antes o después según el caso).
   d. Llamar get_presigned_url con el screenshot
   e. PUT HTTP al uploadUrl con el binario
   f. Guardar publicUrl + status del step

6. submit_test_result con el array completo de steps
7. Resumir al usuario: status global, pasos que fallaron, links a screenshots
```

## Flujo de generación a partir de un documento

Para **generar** casos de prueba (no ejecutarlos) a partir de un PDF o docx funcional:

```
1. create_analysis_draft({ projectId, fileName, contentType })
                         → { processId, uploadUrl, publicUrl }

2. PUT HTTP del PDF/docx al uploadUrl
   - Solo header Content-Type (igual al contentType enviado)
   - NO mandes headers x-amz-checksum-*  (si no, S3 devuelve 403)

3. generate_analysis({ processId, fileUrl: publicUrl, documentType, ...flags })
                         → { processId, message }   (generación ASÍNCRONA)

4. Poleá list_test_cases hasta que el caso esté en status PROCESADO,
   luego fetch_test_case para obtener los pasos.
```

## Patrones anti-fallo

### Pattern 1 — Confirmar antes de ejecutar

**Los test cases pueden tocar sistemas reales** (pagos, creación de cuentas, etc.). **Siempre** pedí confirmación explícita antes de empezar:

> "Voy a ejecutar el caso 375 'Test QR Payment Flow' en `staging.example.com`, lo que incluye simular un pago de $500. ¿Confirmás?"

No arranques si no hay un "sí" explícito.

### Pattern 2 — URL de ejecución obligatoria

El usuario debe darte la URL donde ejecutar (staging, dev, prod). **Nunca asumas `https://example.com`** ni ninguna URL ficticia. Si no te la dieron, preguntala.

### Pattern 3 — Screenshots via presigned URL, no base64

- ❌ Mandar screenshot en `submit_test_result` como base64 inline.
- ✅ Primero `get_presigned_url` → PUT al S3 → guardar `publicUrl` → pasar `publicUrl` a `submit_test_result`.

Por qué: el campo `screenshotUrl` se almacena verbatim en DB y se expone al frontend. Un base64 inflaría la DB y no se puede servir con presigned URL para control de acceso.

### Pattern 4 — Manejo de errores

Si un step falla:
1. Tomá screenshot **del estado de error** (no del estado esperado).
2. Seteá `status: "FAIL"` en ese step, con `errorMessage` explicando qué pasó.
3. **Decidí**: ¿seguir ejecutando los pasos siguientes, o abortar? Depende del tipo de falla:
   - Falla "asertiva" (ej. valor esperado ≠ obtenido): seguí, puede que los pasos siguientes pasen.
   - Falla "estructural" (ej. elemento no existe, timeout, red): abortá el resto con `status: "SKIPPED"`.
4. El `status` global del test es el peor status encontrado: FAIL > ERROR > SKIPPED > PASS.

### Pattern 5 — Idempotencia y duplicados

Cada llamada a `submit_test_result` **crea una nueva ejecución**. No hay deduplicación. Si el usuario te pide "volvé a correr el test", eso es una ejecución nueva y correcta. Pero si tu código falla a medio camino, **no reenvíes** el resultado completo duplicando steps.

### Pattern 6 — Acceso por proyecto

Los tools validan en el servidor que el recurso pertenezca a un **proyecto al que el usuario del API token tiene acceso** (membresía + misma empresa). Si recibís:
- `"Project not accessible"` → el `projectId` existe pero no sos miembro de ese proyecto (o es de otra empresa).
- `"Test case (process) not found"` / `"... not accessible"` → el `caseId` está mal, o pertenece a un proyecto que no podés ver.

No intentes workarounds. Pedile al usuario que verifique con `list_projects` → `list_test_cases(projectId)` qué proyectos y casos tiene disponibles.

## Errores comunes y cómo responderlos

| Error | Causa | Qué hacer |
|---|---|---|
| `401 Invalid API token` | Token inválido o revocado | Pedir al usuario que regenere el token en TESSA → Configuración → API Tokens |
| `Authentication required` | La llamada llegó sin usuario autenticado | Verificar que el API token se envía en el header `Authorization: Bearer qai_...` |
| `Project not found` | El `projectId` no existe | Llamar `list_projects` para encontrar IDs válidos |
| `Project not accessible` | No sos miembro de ese proyecto (o es de otra empresa) | Usar solo `projectId` que aparezcan en `list_projects` |
| `Invalid caseId` | `caseId` no numérico o proceso inexistente | Llamar `list_test_cases(projectId)` para encontrar IDs válidos |
| `Invalid content type` | Screenshot no es png/jpeg/webp | Convertir el screenshot a PNG antes de pedir presigned URL |
| `Test case (process) not found` / `not accessible` | El caso no existe o pertenece a un proyecto que no podés ver | Verificar el `caseId` con `list_test_cases(projectId)` del token actual |

## Ejemplos de conversación

### Ejemplo 1 — Ejecución simple

> **Usuario**: Corré el test case 375 en staging.example.com
>
> **IA** (internamente):
> 1. `fetch_test_case({ caseId: "375" })` → obtiene 5 pasos.
> 2. Responde al usuario: "Voy a ejecutar 'Test QR Payment' (5 pasos) en staging.example.com. Incluye un pago simulado de $500. ¿Confirmás?"
> 3. Usuario confirma.
> 4. Por cada paso: ejecutar en browser → screenshot → `get_presigned_url` → PUT a S3.
> 5. `submit_test_result({...})` con los 5 steps.
> 6. Responde: "Ejecución completa. Status: PASS. 8.4s. Screenshots en..."

### Ejemplo 2 — Descubrimiento

> **Usuario**: ¿Qué tests tengo en TESSA?
>
> **IA**:
> 1. `list_projects({ page: 1, pageSize: 10 })` → muestra los proyectos accesibles.
> 2. Usuario elige "Checkout Web" (`id: 12`).
> 3. `list_test_cases({ projectId: 12 })` → casos del proyecto.
> 4. Responde con la lista formateada. Pregunta cuál querés correr.

### Ejemplo 3 — Happy path + alternativos

> **Usuario**: Ejecutá todos los casos del test 375.
>
> **IA**:
> 1. `fetch_test_case({ caseId: "375" })` → happy path.
> 2. `fetch_additional_cases({ caseId: "375" })` → 4 alternativos.
> 3. Confirma con el usuario: "Voy a correr 1 happy path + 4 alternativos = 5 ejecuciones. ¿OK?"
> 4. Ejecutá cada uno como `submit_test_result` separado.
> 5. Resume: "5/5 ejecutados. 3 PASS, 1 FAIL, 1 ERROR. Detalles..."

### Ejemplo 4 — Falla parcial

> **IA** ejecuta paso 3, encuentra un botón que no existe.
> - Screenshot del estado actual → presigned → PUT.
> - `status: "ERROR"` para ese step, `errorMessage: "Element 'button#pay-now' not found after 10s"`.
> - Pasos 4-5 como `status: "SKIPPED"`, sin screenshot.
> - `submit_test_result` con `status: "ERROR"` global.

## Límites y consideraciones

- **Tamaño de `steps[]`**: array razonable (~100 steps max). Evitá arrays gigantes.
- **Tamaño de screenshot**: idealmente <2 MB cada uno. PNG o WebP comprimido.
- **Rate limiting**: todavía no implementado server-side — usá sentido común, no hagas 50 ejecuciones paralelas.
- **Logs sensibles**: no incluyas passwords reales en `errorMessage` ni en steps. Si ejecutaste con credenciales, tachalas (`pass=***`).
- **Audit trail**: cada `submit_test_result` queda registrado con `executedBy` = owner del proceso. Tu identidad (como agente) no se guarda por separado — sé consciente.

## Recursos

- **Frontend TESSA**: `https://agent.qualis-lab.com`
- **API Tokens**: `https://agent.qualis-lab.com/api-tokens`
- **Soporte**: contactar al equipo de Qualis Lab.

---

*Skill v1.1 — mantener alineada con el schema real de los tools en `backend/src/features/mcp/provider/mcp-tools.provider.ts`.*
