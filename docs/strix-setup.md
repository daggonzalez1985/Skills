# Configurar Strix Cloud (app.strix.ai) en este entorno

Las 9 skills de pentesting (`*-with-strix`, `penetration-testing-with-strix`, etc.)
conducen la herramienta **Strix**. Para usarlas contra la plataforma gestionada
(`app.strix.ai`) hacen falta dos cosas: la **CLI instalada** y una **sesión iniciada**.

Este documento deja anotado el procedimiento porque el entorno de Claude Code on
the web **bloquea `app.strix.ai` por política de red** (el proxy responde `403` al
CONNECT). Hasta habilitar el host, el login no es posible desde la sesión.

---

## 1. Habilitar el host en el environment (acción del usuario, una sola vez)

La política de red se define en la **interfaz web de Claude Code**, no en este repo
ni dentro de la sesión. Pasos:

1. Ir a **https://claude.ai/code** → lista de environments.
2. Editar el environment de este proyecto (`daggonzalez1985/Skills`).
3. En la sección de **red / network policy**, usar una política que permita hosts
   personalizados y agregar:

   ```
   app.strix.ai
   *.strix.ai
   ```

4. Guardar y **reiniciar la sesión** (el proxy carga la lista nueva al arrancar el
   container).

Referencia: https://code.claude.com/docs/en/claude-code-on-the-web

**Verificación rápida de que el host quedó abierto** (debe devolver algo distinto de
`403`):

```bash
curl -sS -o /dev/null -w "HTTP: %{http_code}\n" --max-time 20 https://app.strix.ai/
```

---

## 2. Instalar la CLI de Strix

El container es efímero: si se reinició la sesión, la CLI hay que reinstalarla. El
Python del sistema es 3.11, pero Strix requiere >=3.12, así que se instala con `uv`
(que trae su propio Python):

```bash
export UV_HTTP_TIMEOUT=300
uv tool install strix-agent --python 3.12
export PATH="$HOME/.local/bin:$PATH"
strix --version        # debe imprimir: strix 1.6.x
```

> Alternativa en una máquina con Python >=3.12: `pipx install strix-agent`.

---

## 3. Login (device flow — la aprobación la hace el usuario en el navegador)

```bash
export PATH="$HOME/.local/bin:$PATH"
strix cloud login --no-browser --scope-profile recommended --device-name "claude-code-remote"
```

- Imprime una **URL de verificación**. Abrirla en el navegador y aprobar con la
  cuenta de app.strix.ai. El comando hace polling hasta que se apruebe.
- El perfil `recommended` cubre scans, uploads, cambio de workspace y top-ups
  aprobados por el usuario; excluye `tokens:write` (gestión de credenciales), que
  se pide aparte solo si hace falta.
- Si hay más de un workspace, agregar `--workspace "<nombre-o-id>"` para evitar el
  selector interactivo.

### Alternativa sin navegador: token de API

Si preferís un token generado desde la web (Settings → API tokens en app.strix.ai):

```bash
export STRIX_API_TOKEN=<tu-token>     # NO pegar el token en el chat ni commitearlo
```

⚠️ El token igual pasa por el proxy del environment, así que **también requiere el
paso 1** (host habilitado). Nunca lo escribas en el chat, en el repo, ni en un
commit — cargarlo como variable de entorno del environment.

---

## 4. Verificar la sesión

```bash
strix cloud whoami            # estado local rápido
strix cloud session --json    # verifica la sesión remota
strix cloud session scopes    # scopes efectivos + techo de login
```

`whoami` debe pasar de `{"signed_in": false}` a mostrar la cuenta/workspace.

---

## Notas

- **Seguridad ofensiva:** estas skills ejecutan pentests reales. Usarlas solo contra
  sistemas con autorización explícita. Antes de correr un scan contra un objetivo,
  confirmar alcance y autorización.
- **Logout:** `strix cloud logout` revoca la sesión remota y borra el token local.
- **Licencia:** Strix es Apache-2.0 (`strix-agent` en PyPI, repo `usestrix/strix`).
