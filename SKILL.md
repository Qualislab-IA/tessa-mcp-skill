# TESSA MCP — Skill

**TESSA** (Test Execution & Smart Synthesis Agent) es una plataforma SaaS que genera casos de prueba con IA y los ejecuta vía agentes conectados por MCP. Esta skill te enseña a interactuar correctamente con el servidor MCP de TESSA.

## Cuándo usar esta skill

Usala cuando el usuario pida cualquiera de estas cosas:

- "Ejecutá el test case N" / "Corré el caso de prueba N en [URL]"
- "Listá mis proyectos / test cases de TESSA"
- "¿Qué pasos tiene el caso N?"
- "Probá el happy path de [proyecto] en staging"
- "Subí estos screenshots al proceso N" / "Reportá los resultados"
- Cualquier mención de TESSA, QualisLab, o test cases automatizados en este contexto.

## Servidor

- **URL de producción**: `https://agent.qualis-lab.com/mcp`
- **Autenticación**: Bearer token con prefijo `qai_` (API token de TESSA).
- Los 5 tools se descubren automáticamente vía `tools/list` al conectarse.

## Los 5 tools

### 1. `list_test_cases`

Lista paginada de test cases (llamados internamente "processes") del usuario autenticado.

**Input**:
```json
{ "page": 1, "pageSize": 10 }
```

**Output**:
```json
{
  "projects": [
    { "id": 375, "name": "Test QR Payment Flow with Predefined Amount" },
    { "id": 374, "name": "..." }
  ],
  "pagination": {
    "currentPage": 1, "pageSize": 10,
    "totalItems": 23, "totalPages": 3,
    "hasNextPage": true, "hasPreviousPage": false
  }
}
```

**Usalo primero** cuando el usuario no especifica un `caseId` concreto. Mostrale la lista y pediel confirmación.

### 2. `fetch_test_case`

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

### 3. `fetch_additional_cases`

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

### 4. `get_presigned_url`

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

### 5. `submit_test_result`

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

## Flujo recomendado (end-to-end)

```
1. (Si no hay caseId)  → list_test_cases
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

### Pattern 6 — Ownership

Los tools validan en el servidor que el `caseId` pertenezca al dueño del API token. Si recibís un error tipo `"Process not found or not owned by caller"`, es que:
- El `caseId` está mal.
- O el token pertenece a otro usuario.

No intentes workarounds. Pedile al usuario que verifique el `caseId` con `list_test_cases`.

## Errores comunes y cómo responderlos

| Error | Causa | Qué hacer |
|---|---|---|
| `401 Invalid API token` | Token inválido o revocado | Pedir al usuario que regenere el token en TESSA → Configuración → API Tokens |
| `Invalid caseId` | `caseId` no numérico o proceso inexistente | Llamar `list_test_cases` para encontrar IDs válidos |
| `Invalid content type` | Screenshot no es png/jpeg/webp | Convertir el screenshot a PNG antes de pedir presigned URL |
| `Process not found or not owned by caller` | IDOR: estás tratando de tocar un proceso de otro user | Verificar `caseId` con `list_test_cases` del token actual |
| `caseId not found or not owned by caller` (en `get_presigned_url`) | Mismo que el anterior pero para presigned | Mismo fix |

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
> 1. `list_test_cases({ page: 1, pageSize: 10 })`.
> 2. Responde con la lista formateada. Pregunta cuál querés correr.

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

*Skill v1.0 — mantener alineada con el schema real de los tools en `backend/src/features/mcp/provider/mcp-tools.provider.ts`.*
