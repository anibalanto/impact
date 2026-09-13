# Impact Report

Un Impact Report es el artefacto central que produce impact. Describe de forma estructurada qué cambió, en qué archivos, qué cadenas de bilinks se vieron afectadas, y qué commits intercedieron desde el último estado conocido.

## Estructura

### Un reporte es un archivo `<uuid>.impact` con frontmatter YAML y cuerpo Markdown

```yaml
---
id: <uuid>
generated_at: <iso8601-utc>
trigger:
  kind: bilinker_event | git_hook | manual
  file: <path-relativo>
  commit: <sha1>        # presente si kind == git_hook o manual con commit
chains:
  - uuid: <chain-uuid>
    node: <layer-path>  # el nodo de la cadena que referencia el archivo
    endpoint: 0 | 1
    state: ALTERED | DELETED | ...
    file: <path>
    commit_anchor: <sha1>  # el commit en que el contenido aceptado quedó establecido
    commits_since:
      - sha: <sha1>
        author: <name>
        date: <iso8601>
        message: <primera línea>
---

## Cambio

<descripción del diff o resumen del cambio>

## Cadenas afectadas

<lista de cadenas con su estado y los commits que intercedieron>
```

## Generación

### Un Impact Report se genera ante un evento de bilinker, un git hook o una invocación manual

1. bilinker emite un evento `ALTERED` o `DELETED` para un archivo
2. Un git hook detecta un commit que modifica archivos linkeados
3. El usuario invoca `impact report <file>` manualmente

### La generación no modifica ningún archivo fuera de `.impact/reports/`

## Los comandos que generan reportes

### `impact scan` busca los archivos con commits no analizados desde el último estado aceptado

```
impact scan [--path <dir>] [--chain <uuid>]
```

Consulta el grafo del proyecto vía lattice y calcula qué archivos tienen commits no analizados desde el último estado aceptado:

1. Consulta el grafo bajo el directorio raíz: `lattice graph . --recursive --guarantee accepted --format json`.
2. Para cada arista con `commit` presente, toma el commit de cada extremo y ejecuta `git log <edge.commit[n]>..HEAD -- <file>`.
3. Si el resultado es no vacío, el archivo tiene commits intercedidos desde el anchor: se genera un Impact Report en `.impact/reports/<uuid>.impact`.
4. Si ya existe un reporte abierto para la misma combinación (chain + file + commit_anchor), no genera uno nuevo.
5. Imprime el resumen en stdout.

| Flag | Descripción |
|------|-------------|
| `--path <dir>` | Restringe el scan a `.bilink` files bajo ese subdirectorio |
| `--chain <uuid>` | Restringe el scan a una cadena específica |

```
scan: 2 bilink(s) con cambios desde anchor

  src/Persona.java   chain abc12345  2 commits
  specs/voting.yaml  chain abc12345  1 commit

generados: .impact/reports/f3a1...  .impact/reports/9c2b...
```

Si no hay cambios:

```
scan: todos los bilinks están en su anchor  (12 revisados)
```

| Código | Condición |
|---|---|
| `0` | sin cambios detectados |
| `1` | se generaron reportes (hay cambios para revisar) |
| `2` | error en la ejecución |

`impact scan` es la entrada principal del ciclo de análisis. Puede invocarse manualmente antes de una revisión, como parte de un git hook post-merge o post-fetch, o en un CI pipeline para detectar drift entre ramas. No requiere `bilinker watch` en ejecución: la fuente de verdad es el historial de git accesible localmente.

### `impact report` genera un reporte puntual para un archivo o un vínculo

```
impact report <file>
impact report --bilink <uuid>
```

1. Resuelve el target: archivo en el filesystem o UUID de un vínculo.
2. Consulta las aristas que lo alcanzan: `lattice graph <target> --guarantee accepted --format json`.
3. Para cada arista con `commit` presente, toma el commit de cada extremo y ejecuta `git log <edge.commit[n]>..HEAD -- <file>`.
4. Construye el Impact Report con la lista de chains afectadas, el diff acumulado y los commits intercedidos.
5. Guarda el reporte en `.impact/reports/<uuid>.impact`.
6. Imprime el reporte en stdout.

| Flag | Descripción |
|------|-------------|
| `--bilink <uuid>` | Genera el reporte para un bilink específico por UUID |
| `--no-save` | Imprime el reporte sin guardarlo en `.impact/reports/` |

```
Impact Report  f3a19c2b

trigger:  manual
file:     src/Persona.java
commit:   a1b2c3d4

chains afectadas:
  abc12345  src/Persona.java ↔ specs/voting.yaml
    ALTERED  src/Persona.java
    2 commits desde anchor a1b2c3d4:
      d5e6f7a8  Anibal  2026-05-20  refactor vote method signature
      9b0c1d2e  Anibal  2026-05-22  add validation in vote()

reporte guardado: .impact/reports/f3a19c2b-...
```

| Código | Condición |
|---|---|
| `0` | reporte generado sin cambios significativos |
| `1` | reporte generado con cambios detectados |
| `2` | target no encontrado o error en la ejecución |

`impact report` es la versión puntual y manual de `impact scan`. Scan barre todo el proyecto; report se enfoca en un target específico. Ambos generan el mismo formato de artefacto `.impact`.

### `impact watch` corre `scan` ante cada cambio del filesystem

```
impact watch [--path <dir>]
```

1. Observa el directorio raíz del proyecto (o `--path`) con un watcher de filesystem.
2. Cuando un archivo modificado es referenciado por algún `.bilink`, ejecuta `impact scan --path <dir-del-archivo>` de forma automática.
3. Imprime en stdout cada scan disparado y su resultado.
4. Corre indefinidamente hasta Ctrl-C.

```
watching /home/anibal/proyecto  (Ctrl-C to stop)

[2026-05-20T14:32:10Z] cambio detectado: src/Persona.java
  → impact scan  →  1 reporte generado: f3a19c2b
[2026-05-20T14:45:02Z] cambio detectado: specs/voting.yaml
  → impact scan  →  sin cambios
```

| Código | Condición |
|---|---|
| `0` | detenido con Ctrl-C |
| `2` | error al inicializar el watcher |

`bilinker watch` detecta drift estructural en los `.bilink` files. `impact watch` reacciona a esos mismos cambios pero con un análisis más profundo: calcula el blast radius, obtiene el historial de git y genera Impact Reports. Pueden correr en paralelo o de forma independiente: `impact watch` no depende de que `bilinker watch` esté activo, observa el filesystem directamente.

Sirve en un servidor de integración continua o en un entorno de desarrollo donde se quiere análisis de impacto en tiempo real. Para análisis puntuales o en CI, `impact scan` es suficiente.

## Garantía

### Un Impact Report no presenta una inferencia del LSP como si fuera un vínculo verificado

Cada arista llega con su `guarantee`, y el reporte la lee así:

- `accepted` → sostiene una afirmación de drift; hay estado anterior aceptado.
- `derived` → sugiere revisión; el call graph falla con dispatch dinámico, macros y callbacks.
- `asserted` → contexto, sin verificación.

## Degradación

### Un reporte generado sin el proveedor LSP dice que vio menos grafo

Todo resultado de lattice lleva el estado de cada proveedor. Impact lo propaga al Impact Report: uno generado sin el proveedor LSP vio menos grafo, y el lector tiene que poder distinguirlo de uno completo.

Un reporte que no distingue "no encontré impacto" de "no pude buscarlo" es peor que no tener reporte.

## Ciclo de vida

```
generado → abierto en thread → resuelto | descartado
```

### Un reporte sin thread asociado es información pasiva

Al abrirse un [thread](thread.md), el reporte se convierte en el mensaje inicial de ese thread y queda disponible para discusión en Accreta.

## Invariantes

### `id` es UUID v4, y coincide con el nombre del archivo

### `commit_anchor` es el `commit` que la arista reportaba para ese extremo al momento de la detección

### `commits_since` son exactamente los commits en `git log <commit_anchor>..HEAD -- <file>`

Si `commits_since` está vacío, el cambio ocurrió pero no hay commits posteriores al anchor (archivo modificado sin commitear).

### Un reporte no se modifica después de ser generado

Si el estado vuelve a cambiar, se genera un nuevo reporte.
