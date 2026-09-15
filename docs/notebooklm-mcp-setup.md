# NotebookLM MCP — guía de configuración

Procedimiento para instalar y usar el servidor MCP
[`PleasePrompto/notebooklm-mcp`](https://github.com/PleasePrompto/notebooklm-mcp)
(v2.0.0, licencia MIT, autor Gérôme Dexheimer).

> **Importante:** esto **no es una skill** de este repo. Es un **servidor MCP**
> que se configura aparte en tu cliente (Claude Code, Cursor, etc.) y le agrega
> herramientas para operar NotebookLM. No vive en `.agents/skills/`. Este
> documento queda en `docs/` solo como referencia.

---

## Qué es

Un servidor MCP que **automatiza el sitio web de NotebookLM** manejando un
Chrome real vía **Patchright** (un fork "stealth" de Playwright). Permite que un
agente:

- chatee contra un notebook (`ask_question`, con extracción de citas),
- ingiera fuentes (`add_source`: URL con crawl, o texto pegado),
- genere y descargue *Audio Overviews*,
- administre una librería local de notebooks.

NotebookLM **no tiene API pública oficial**: todo se hace conduciendo la interfaz
web. Por eso el MCP depende del HTML del sitio y puede romperse cuando Google
cambie algo.

---

## ⚠️ Antes de engancharlo — leé esto

1. **Riesgo de cuenta de Google / ToS.** Usa automatización *stealth*
   (Patchright) para evadir la detección de bots. Automatizar NotebookLM así
   **va contra los términos de Google** y puede marcar o suspender tu cuenta.
   Evaluá el riesgo antes de usarlo con una cuenta que te importe.
2. **Corre en tu máquina local, no en una sesión remota efímera.** El login
   necesita abrir una **ventana visible de Chrome** donde vos te logueás a Google
   a mano. En un contenedor remoto sin display interactivo **no se puede
   completar** (mismo bloqueo que tuvimos con Strix Cloud).
3. **Requisitos:** Node.js instalado; en Linux headless, `xvfb` para el login
   único.
4. **Credenciales:** no hay almacén cifrado. La sesión de Google se guarda como
   cookies dentro de un **perfil de Chrome por usuario** en disco. El aislamiento
   entre cuentas es solo por carpeta. **Nunca** subas ese perfil a git.

---

## 1. Instalar

Opción recomendada (paquete publicado, se autoactualiza con `@latest`):

```bash
npx notebooklm-mcp@latest
```

Desde el código fuente:

```bash
git clone https://github.com/PleasePrompto/notebooklm-mcp.git
cd notebooklm-mcp
npm install
npm run build
node dist/index.js
```

## 2. Conectar a Claude Code

Forma CLI:

```bash
claude mcp add notebooklm -- npx notebooklm-mcp@latest
# o, desde un clon local ya compilado:
claude mcp add notebooklm -- node /ruta/absoluta/a/notebooklm-mcp/dist/index.js
```

Forma manual — agregar a `~/.claude.json`:

```json
{
  "mcpServers": {
    "notebooklm": {
      "command": "npx",
      "args": ["notebooklm-mcp@latest"]
    }
  }
}
```

Para un build local, usar `"command": "node"` y
`"args": ["/ruta/absoluta/a/dist/index.js"]`.

## 3. Login (una sola vez)

La herramienta `setup_auth` abre un Chrome visible; te logueás a tu cuenta de
Google y las cookies quedan persistidas en el perfil. Tenés hasta ~10 minutos
para completar el login. Las corridas siguientes reutilizan el perfil y ya no
piden login (pueden correr headless).

- **Linux headless:** corré el login una vez bajo xvfb:
  `xvfb-run -a npx notebooklm-mcp`
- `re_auth` — borra el auth guardado y vuelve a empezar (cambiar de cuenta o
  arreglar auth roto).

Ubicación del perfil de Chrome (vía `env-paths`):

| SO | Ruta |
|----|------|
| Linux | `~/.local/share/notebooklm-mcp/chrome_profile/` |
| macOS | `~/Library/Application Support/notebooklm-mcp/chrome_profile/` |
| Windows | `%APPDATA%\notebooklm-mcp\chrome_profile\` |

## 4. Verificar

- `get_health` — estado de auth, cantidad de sesiones, snapshot de config y
  pista de troubleshooting.

---

## Herramientas principales (perfil `full`)

| Herramienta | Para qué |
|---|---|
| `ask_question` | Preguntar contra un notebook; reutiliza sesión y extrae citas. |
| `add_source` | Agregar fuente (`type=url` con crawl, o `type=text`). |
| `generate_audio` | Generar un Audio Overview (`custom_prompt` opcional). |
| `download_audio` | Guardar el último Audio Overview en `destination_dir`. |
| `add_notebook` / `list_notebooks` / `get_notebook` / `select_notebook` / `update_notebook` / `remove_notebook` / `search_notebooks` / `get_library_stats` | Administrar la librería local de notebooks. |
| `list_sessions` / `close_session` / `reset_session` | Manejo de sesiones de navegador. |
| `setup_auth` / `re_auth` | Login inicial / re-login. |
| `get_health` | Diagnóstico. |
| `cleanup_data` | Previsualizar y borrar datos guardados (`preserve_library=true` conserva `library.json`). |

### Perfiles (recortan la lista de tools para ahorrar contexto)

| Perfil | Tools |
|---|---|
| `minimal` | `ask_question`, `get_health`, `list_notebooks`, `select_notebook`, `get_notebook` |
| `standard` | `minimal` + `setup_auth`, `list_sessions`, `add_notebook`, `update_notebook`, `search_notebooks` |
| `full` (default) | todas |

```bash
npx notebooklm-mcp config set profile minimal
npx notebooklm-mcp config set disabled-tools cleanup_data,re_auth
```

---

## Transportes

- `stdio` (default) — para clientes MCP locales como Claude Code.
- Streamable-HTTP — expone `/mcp` con manejo de sesión vía header
  `Mcp-Session-Id`; útil para correrlo como servicio.

## Referencias

- Repo: <https://github.com/PleasePrompto/notebooklm-mcp>
- Docs del repo: `docs/configuration.md`, `docs/tools.md`,
  `docs/usage-guide.md`, `docs/troubleshooting.md`
