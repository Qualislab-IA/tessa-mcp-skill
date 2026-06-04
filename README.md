# TESSA MCP Skill

Guía para que asistentes de IA en distintos IDEs usen correctamente el **servidor MCP de TESSA** (Test Execution & Smart Synthesis Agent).

TESSA expone 6 herramientas MCP para descubrir test cases, ejecutarlos en un navegador, subir screenshots, reportar resultados y generar análisis a partir de documentos. Esta skill le enseña a la IA el **flujo correcto** y los **patrones anti-fallo**.

## ¿Qué es esto?

El servidor MCP está deployado en:

```
https://agent.qualis-lab.com
```

Cualquier cliente compatible con Model Context Protocol (Claude Code, Cursor, Copilot, Antigravity, Claude Desktop, Windsurf, Cline, Continue, etc.) puede conectarse. Los tools se descubren automáticamente.

Esta carpeta contiene la **skill / rules** en formato nativo de cada IDE para que el agente:

1. Sepa el orden correcto de los tools.
2. Maneje screenshots vía presigned URLs (no base64).
3. Valide ownership antes de reportar.
4. Recupere errores comunes.

## Instalación por IDE

### 1. Claude Code

Copiá la carpeta al proyecto donde trabajás:

```bash
cp -r claude-code/.claude your-project/
```

La skill se activa automáticamente cuando abrís Claude Code en ese proyecto. Alternativamente, para instalación personal:

```bash
cp -r claude-code/.claude/skills/tessa-mcp ~/.claude/skills/
```

### 2. Cursor

Copiá las reglas al proyecto:

```bash
cp -r cursor/.cursor your-project/
```

Cursor cargará automáticamente `.cursor/rules/tessa-mcp.mdc` cuando detecte contexto relacionado (archivos de test, mención de TESSA, etc.).

### 3. GitHub Copilot

Copiá el archivo de instrucciones:

```bash
cp -r copilot/.github your-project/
```

Copilot leerá `.github/copilot-instructions.md` en cada request del workspace.

### 4. Google Antigravity

Copiá las reglas al proyecto:

```bash
cp -r antigravity/.antigravity your-project/
# o en la raíz:
cp antigravity/AGENTS.md your-project/
```

Antigravity inyecta automáticamente las rules del workspace en el contexto del agente.

### Otros IDEs compatibles con MCP

Si tu IDE no está listado pero soporta MCP (Windsurf, Cline, Continue, Zed, Aider, etc.):

1. Configurá el servidor MCP apuntando a `https://agent.qualis-lab.com` (ver sección "Configuración MCP" más abajo).
2. Copiá `generic/tessa-mcp-guide.md` a la ubicación de rules/context de tu IDE.

## Configuración MCP (común a todos los IDEs)

Antes de que la skill tenga efecto, el IDE tiene que estar conectado al MCP de TESSA.

### Paso 1 — Conseguir un API Token

Hacés login en el frontend de TESSA y vas a **Configuración → API Tokens → Crear token**.

O vía CURL (necesitás JWT de una sesión activa):

```bash
curl -X POST https://agent.qualis-lab.com/api/api-tokens \
  -H "Authorization: Bearer <TU_JWT>" \
  -H "Content-Type: application/json" \
  -d '{"name":"Mi Cursor"}'
```

La respuesta trae un `token` que empieza con `qai_`. **Copialo ahora** — solo se muestra una vez.

### Paso 2 — Configurar el cliente MCP

**Claude Code / Claude Desktop** (`~/.claude.json` o el `.mcp.json` del proyecto):

```json
{
  "mcpServers": {
    "tessa": {
      "type": "http",
      "url": "https://agent.qualis-lab.com/mcp",
      "headers": {
        "Authorization": "Bearer qai_XXXXXXXXXXXXXXXXXXXXXXXX"
      }
    }
  }
}
```

**Cursor** (`~/.cursor/mcp.json` o `.cursor/mcp.json` del proyecto):

```json
{
  "mcpServers": {
    "tessa": {
      "url": "https://agent.qualis-lab.com/mcp",
      "headers": {
        "Authorization": "Bearer qai_XXXXXXXXXXXXXXXXXXXXXXXX"
      }
    }
  }
}
```

**Antigravity**: agregá el server desde Settings → MCP Servers con la misma URL y header `Authorization`.

**GitHub Copilot**: Copilot todavía no soporta MCP nativo en todas las plataformas. Si tu versión lo soporta (Copilot Chat en VS Code con extensión MCP bridge), usá la misma config que Cursor. Si no, la skill igual funciona como instructions — Copilot no va a ejecutar los tools pero va a entender los conceptos y escribir código correcto contra la API REST equivalente (`/api/mcp/*`).

## Los 6 tools expuestos

| Tool | Propósito |
|---|---|
| `list_projects` | Lista paginada de los proyectos accesibles del usuario (id + nombre) |
| `list_test_cases` | Lista paginada de los casos de un proyecto (requiere `projectId`); devuelve casos en todos los estados (`DRAFT`, `INICIADO`, `AWAITING_APPROVAL`, `PROCESADO`, `ERROR`), cada uno con su `status` |
| `fetch_cases` | Trae happy path + adicionales + gherkin + uxui de un proceso en una sola llamada; filtros opcionales por tipo |
| `get_presigned_url` | Genera URL firmada para subir un screenshot a S3 |
| `submit_test_result` | Reporta el resultado de una ejecución + steps + screenshots |
| `generate_analysis` | Genera test cases de forma asíncrona a partir del texto de un documento funcional, en una sola llamada |

Detalles completos en `SKILL.md` (fuente canónica).

## Arquitectura de archivos

```
tessa-mcp-skill/
├── README.md                        ← Este archivo
├── SKILL.md                         ← Contenido canónico (fuente de verdad)
├── claude-code/
│   └── .claude/
│       └── skills/tessa-mcp/
│           └── SKILL.md             ← Formato Claude Code (frontmatter YAML)
├── cursor/
│   └── .cursor/
│       └── rules/
│           └── tessa-mcp.mdc        ← Formato Cursor (.mdc con frontmatter)
├── copilot/
│   └── .github/
│       └── copilot-instructions.md  ← Formato Copilot
├── antigravity/
│   ├── AGENTS.md                    ← Rules de Antigravity
│   └── .antigravity/
│       └── rules/tessa-mcp.md
└── generic/
    └── tessa-mcp-guide.md           ← Markdown genérico para cualquier IDE
```

