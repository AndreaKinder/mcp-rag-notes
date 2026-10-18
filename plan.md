# Plan: mcp-notes-rag

Servidor MCP (Model Context Protocol) que expone RAG semántico + FTS5 sobre vaults de notas personales (Obsidian/Neovim). Un solo paquete Python, dos entry-points (`en`/`es`), transporte stdio.

---

## 1. Arquitectura General

| Aspecto | Decisión |
|---------|----------|
| Transporte | stdio (servers locales ejecutados por el agente) |
| Lenguaje | Python 3.12+ |
| Framework | SDK oficial de MCP: `pip install mcp` |
| Base de datos | SQLite nativo (sin extensiones C); índice híbrido: vectores BLOB + FTS5 |
| Embeddings | Ollama vía HTTP (`urllib`); modelo configurable, por defecto `snowflake-arctic-embed2` |
| Vaults | Variables de entorno con rutas relativas; nunca rutas absolutas del sistema en el código |
| DB fuera del vault | `~/.local/share/notes-rag/index.db` (EN) / `~/.local/share/notas-rag/index.db` (ES) |

---

## 2. Un Solo Paquete, Dos Entry-Points

No se crean dos proyectos. Un solo paquete `notes_rag` con dos binarios registrados en `pyproject.toml`:

```toml
[project.scripts]
mcp-server-notes-en = "notes_rag.server:main_en"
mcp-server-notes-es = "notes_rag.server:main_es"
```

Ambos llaman al mismo `server.py` con `lang="en"` o `lang="es"`, que carga:
- El locale correspondiente (`locales/en.json` o `locales/es.json`)
- Las variables de entorno por defecto del idioma

```
  mcp-notes-rag/              # repo
  ├── pyproject.toml
  ├── README.md
  ├── src/notes_rag/          # paquete Python (underscore)
  │   ├── __init__.py
  │   ├── server.py           # Entry point MCP (stdio, async)
  │   ├── tools.py            # Registro e implementación de tools (async wrappers)
  │   ├── index.py            # Lógica de embeddings + SQLite (portado de notes_index.py)
  │   ├── vault.py            # Lógica del vault: rutas, frontmatter, daily notes, tareas
  │   ├── resolve.py          # Resolución de entidades (helper interno, NO tool pública)
  │   └── locales/
  │       ├── en.json         # Textos, descripciones de tools, plantillas, stopwords
  │       └── es.json
  └── .pi/
      └── mcp.json            # Config de ejemplo para Pi
```

---

## 3. Variables de Entorno

| Variable | Default (EN) | Default (ES) |
|----------|--------------|--------------|
| `NOTES_VAULT_PATH` | `~/Notes` | `~/Notas` |
| `NOTES_INDEX_DB` | `~/.local/share/notes-rag/index.db` | `~/.local/share/notas-rag/index.db` |
| `OLLAMA_HOST` | `http://localhost:11434` | Ídem |
| `NOTES_EMBED_MODEL` | `snowflake-arctic-embed2` | Ídem |
| `NOTES_LANG` | `en` | `es` (sobreescrito por entry-point) |

---

## 4. Tools a Exponer (9 herramientas)

`resolve_entity` se elimina como tool pública; se usa como helper interno. Las descripciones de cada tool deben estar en el idioma del servidor (ES o EN) para que el LLM las invoque correctamente.

| # | Tool | Descripción |
|---|------|-------------|
| 1 | `query_notes` | Búsqueda semántica híbrida (vectorial + FTS5, fusión RRF). Parámetros: `query`, `limit`, `type`, `tag`, `since`, `until`. |
| 2 | `get_daily_note` | Lee la nota diaria de una fecha dada (hoy, ayer, ISO). Crea la nota si no existe (desde plantilla). |
| 3 | `add_task` | Añade una tarea `- [ ] ...` a la nota diaria de una fecha. Parámetros: `task`, `date`, `category`. |
| 4 | `complete_task` | Marca una tarea como hecha `[x]` en la nota diaria. Parámetros: `task_query`, `date`. |
| 5 | `get_note` | Lee el contenido completo de una nota por ruta relativa al vault. |
| 6 | `list_notes` | Lista notas de un directorio o tipo (`journal`, `zettelkasten`, `projects`, ...). |
| 7 | `index_vault` | (Re)construye el índice. Por defecto **incremental** y rápido (< 5s). Si `full_rebuild=true`, se ejecuta en background vía `asyncio.to_thread` y devuelve inmediatamente: "Indexación completa iniciada. Usa `index_status` para verificar". |
| 8 | `index_status` | Estado del índice: notas indexadas, chunks, cambios pendientes, modelo usado. |
| 9 | `create_zettel` | Crea una nota atómica en `zettelkasten/` con frontmatter YAML obligatorio. |

---

## 5. Resources (opcionales, post-MVP)

- `notes://status` — JSON con estado del índice y modelo usado.
- `notes://vault-structure` — Lista jerárquica de directorios del vault.

No son prioritarias sobre las Tools.

---

## 6. Dependencias

```toml
dependencies = ["mcp>=1.0.0"]
```

No se añade cliente Python de Ollama ni Poetry. Se mantiene `urllib` para embeddings (cero dependencias extra).

Opcionalmente `pyyaml` si se quiere parseo robusto de frontmatter; de lo contrario, fallback manual.

---

## 7. Async y Manejo de Errores

- El SDK MCP es **asíncrono** (`asyncio`).
- Toda lógica bloqueante (sqlite3, I/O de disco, HTTP a Ollama) se ejecuta en thread vía `asyncio.to_thread(...)`.
- **Cada tool** debe llevar `try/except` completo. Nunca debe subir una excepción sin capturar al servidor. Si hay error, devolver `CallToolResult(isError=True, ...)` con mensaje legible en el idioma del servidor.
- Timeouts de Ollama: 30s para embeddings por batch, con retry 1 vez. Si falla, mensaje claro al usuario.

---

## 8. Flujo de Implementación

| Paso | Qué hacer |
|------|-----------|
| 1 | Crear `pyproject.toml` con `mcp>=1.0.0`, dos entry-points `mcp-server-notes-en/es`. |
| 2 | Portar `notes_index.py` → `src/notes_rag/index.py`. Limpiar y desacoplar de rutas absolutas; hacer que use variables de entorno. |
| 3 | Portar `resolve_note.py` → `src/notes_rag/vault.py` (daily notes, tareas, frontmatter) + `src/notes_rag/resolve.py` (resolución de entidades como helper privado). |
| 4 | Implementar `server.py` con `Server("notes-rag", "1.0.0")` + `stdio_server`. Cargar locale según `--lang`. |
| 5 | Implementar `tools.py`: registrar las 9 tools con wrappers async, `asyncio.to_thread` para lógica bloqueante, y `try/except` inquebrantable en cada una. |
| 6 | Crear `locales/en.json` y `locales/es.json`: textos, descripciones de tools, mensajes de error, stopwords, plantillas de daily note. |
| 7 | Añadir `.pi/mcp.json` de ejemplo con `uv run` o binario en PATH. |
| 8 | Test local: `uv run mcp-server-notes-es` y conectar a Pi. Verificar que las tools se listen y `query_notes` devuelva resultados. |
| 9 | Pulir: logging, manejo de timeouts, mensajes de error traducidos, README. |

---

## 9. Decisiones Clave Resueltas

| Pregunta | Respuesta |
|----------|-----------|
| ¿Un solo entry-point con `--lang` o dos? | Dos entry-points en `pyproject.toml` que llaman a funciones `main_en()` / `main_es()`; cero duplicación de código. |
| ¿Poetry, uv o pip? | `pyproject.toml` estándar; compatible con uv, pip, o cualquier gestor. |
| ¿Cliente Python de Ollama? | No. Se mantiene `urllib` para evitar dependencias. |
| ¿OAuth? | No. Puro local con Ollama. |
| ¿`resolve_entity` como tool? | No. Helper interno; `get_daily_note` y otras tools hacen resolución implícita. |
| ¿Creación automática de daily note? | Sí, `get_daily_note` crea desde plantilla si no existe. |
| ¿Indexación completa bloqueante? | No. Por defecto incremental; full rebuild en background con respuesta inmediata. |
| ¿Descripciones de tools? | En el idioma activo del servidor (ES o EN). |

---

## 10. Cosas que NO cambian del plan original

- SQLite nativo (sin `sqlite-vec` ni ChromaDB).
- Chunking por headings.
- Índice híbrido (vectorial + FTS5 + RRF).
- Ollama como único proveedor de embeddings.
- Variables de entorno para rutas privadas.
- DB fuera del vault.

---

## 11. Configuración en Pi (ejemplo)

Con `uv`:
```json
{
  "mcpServers": {
    "notes-rag-es": {
      "command": "uv",
      "args": ["run", "--project", "/path/to/mcp-notes-rag", "mcp-server-notes-es"]
    }
  }
}
```

Con binario instalado en PATH:
```json
{
  "mcpServers": {
    "notes-rag-es": {
      "command": "mcp-server-notes-es"
    }
  }
}
```

---

## 12. Criterios de aceptación para "listo"

- [ ] `mcp-server-notes-en` y `mcp-server-notes-es` arrancan sin errores vía stdio.
- [ ] `query_notes` devuelve resultados híbridos (vectorial + FTS5) en < 3s.
- [ ] `index_vault` incremental termina en < 5s; full rebuild no bloquea el servidor.
- [ ] `get_daily_note` crea la nota desde plantilla si no existe.
- [ ] `add_task` / `complete_task` modifican la daily note correctamente.
- [ ] Un error en cualquier tool devuelve `isError=True` con mensaje legible; el proceso sigue vivo.
- [ ] Las descripciones de tools aparecen en el idioma correcto según el servidor.
- [ ] El paquete se instala con `pip install .` o `uv pip install .` sin errores.
