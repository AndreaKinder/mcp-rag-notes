# mcp-notes-rag

Servidor MCP (Model Context Protocol) que permite a cualquier agente de IA hacer búsqueda semántica + full-text sobre tu bóveda de notas personales, además de gestionar notas diarias y tareas.

- **Transporte:** stdio (ejecutado localmente, consumido por Pi, Claude Code, OpenCode, etc.)
- **Búsqueda:** Híbrida vectorial + FTS5 con fusión por rango recíproco (RRF)
- **Embeddings:** Ollama (por defecto: `snowflake-arctic-embed2`)
- **Índice:** SQLite nativo (sin extensiones C, sin ChromaDB)
- **Bóvedas:** Soporta inglés (`~/Notes`) y español (`~/Notas`) mediante entry points separados

---

## Requisitos previos

- Python **3.12+**
- [Ollama](https://ollama.com) ejecutándose localmente (o accesible vía `OLLAMA_HOST`)
- El modelo de embeddings descargado: `ollama pull snowflake-arctic-embed2`
- Tu bóveda de notas en `~/Notes` (EN) o `~/Notas` (ES)

---

## Instalación

### Con `uv` (recomendado)

```bash
uv pip install -e .
```

### Con `pip`

```bash
pip install -e .
```

Esto registra dos puntos de entrada:
- `mcp-server-notes-en`
- `mcp-server-notes-es`

---

## Configuración

Toda la configuración se realiza mediante variables de entorno. Nada está hardcodeado.

| Variable | Default (EN) | Default (ES) | Descripción |
|----------|--------------|--------------|-------------|
| `NOTES_VAULT_PATH` | `~/Notes` | `~/Notas` | Ruta a tu bóveda de notas |
| `NOTES_INDEX_DB` | `~/.local/share/notes-rag/index.db` | `~/.local/share/notas-rag/index.db` | Ubicación del índice SQLite |
| `OLLAMA_HOST` | `http://localhost:11434` | igual | Endpoint de la API de Ollama |
| `NOTES_EMBED_MODEL` | `snowflake-arctic-embed2` | igual | Nombre del modelo de embeddings |

Ejemplo antes de ejecutar:

```bash
export NOTES_VAULT_PATH="$HOME/Notas"
export OLLAMA_HOST="http://localhost:11434"
export NOTES_EMBED_MODEL="snowflake-arctic-embed2"
```

---

## Herramientas disponibles

Las **descripciones** de todas las herramientas están localizadas al idioma del servidor, para que el LLM las invoque correctamente.

| Herramienta | Propósito |
|-------------|-----------|
| `query_notes` | Búsqueda híbrida semántica + por palabras clave, con filtros (tipo, etiqueta, rango de fechas). |
| `get_daily_note` | Lee una nota diaria por expresión de fecha. La crea desde plantilla si no existe. |
| `add_task` | Añade una tarea `- [ ] ...` bajo una categoría en la nota diaria. |
| `complete_task` | Marca una tarea pendiente como completada `[x]` buscando por su texto. |
| `get_note` | Lee el contenido completo de cualquier nota por ruta relativa en la bóveda. |
| `list_notes` | Lista notas de un directorio o de un tipo específico. |
| `index_vault` | (Re)construye el índice de búsqueda. Por defecto incremental; reconstrucción completa en background. |
| `index_status` | Muestra estado del índice: notas indexadas, chunks, cambios pendientes, modelo usado. |
| `create_zettel` | Crea una nueva nota atómica Zettelkasten con frontmatter YAML obligatorio. |

---

## Configuración en Pi

Copia `.pi/mcp.json` en tu proyecto o en la configuración de usuario de Pi, o añade el servidor directamente:

```bash
pi mcp add notes-rag-es --env OLLAMA_HOST='http://localhost:11434' -- uv run --project ~/Projects/mcp-rag-notes mcp-server-notes-es
```

O con `pip` (si está instalado en el entorno activo):

```bash
pi mcp add notes-rag-es -- mcp-server-notes-es
```

Verifica la conexión:

```bash
pi mcp list
```

Luego inicia Pi y las herramientas estarán disponibles inmediatamente (`exposure: direct` en la configuración de ejemplo).

---

## Prueba rápida

Tras instalar, puedes probar el servidor directamente:

```bash
# Servidor en español
mcp-server-notes-es

# Servidor en inglés
mcp-server-notes-en
```

El servidor espera mensajes MCP por stdio. Para probar end-to-end, conecta vía Pi o el inspector MCP:

```bash
npx -y @anthropics/mcp-inspector mcp-server-notes-es
```

---

## Notas de arquitectura

- **Un solo paquete, dos entry points.** `mcp-server-notes-en` y `mcp-server-notes-es` comparten el 100% del código; solo cambian el locale y las rutas por defecto.
- **Seguro para async.** El SDK de MCP es async, pero todo trabajo bloqueante (I/O de SQLite, HTTP a Ollama, lectura de archivos) se ejecuta en hilos mediante `asyncio.to_thread`. El event loop nunca se bloquea.
- **Tools a prueba de fallos.** Cada tool está envuelta en `try/except`. Las excepciones no capturadas nunca crashean el proceso stdio; devuelven un resultado con `isError=True` y un mensaje legible.
- **Indexación incremental.** `index_vault` escanea solo archivos modificados por hash SHA-256. Una reconstrucción completa se activa con `full_rebuild=true` y se ejecuta en un hilo en segundo plano para que la respuesta MCP sea inmediata.
- **Búsqueda híbrida.** La similitud vectorial (coseno) se fusiona con el ranking FTS5 mediante RRF. Los resultados incluyen un snippet resaltado.
- **Chunking por encabezados.** Las notas se dividen en H1/H2/H3; cada chunk se enriquece con título, aliases, etiquetas y descripción antes de generar el embedding.

---

## Estructura de bóveda esperada

Tu bóveda debería seguir un layout tipo Obsidian:

```
~/Notas o ~/Notes/
  ├── journal/           # Notas diarias: YYYY-MM-DD.md
  ├── zettelkasten/      # Notas atómicas
  ├── projects/          # Carpetas de proyecto con README.md
  ├── finanzas/          # Notas financieras por año (ES) / finance/ (EN)
  ├── templates/         # Plantillas
  └── inbox/             # Bandeja de entrada
```

Las carpetas de primer nivel `archived`, `assets`, y cualquiera que empiece por `.` (`.obsidian`, `.git`, `.trash`) son ignoradas por el indexador.

---

## Resolución de problemas

| Síntoma | Solución |
|---------|----------|
| `Ollama connection refused` | Inicia Ollama: `ollama serve` (o revisa `OLLAMA_HOST`). |
| Error `Model changed` | Ejecuta `index_vault` con `full_rebuild=true` tras cambiar `NOTES_EMBED_MODEL`. |
| `Sin resultados` | Ejecuta `index_vault` primero para construir el índice. |
| Primera query lenta | La primera `query_notes` sincroniza archivos cambiados automáticamente. Ejecuta `index_vault` periódicamente para mantenerlo rápido. |
| Nota diaria no se crea | Asegúrate de que existe `journal/` en la bóveda y que el servidor tiene permisos de escritura. |

---

## Licencia

MIT
