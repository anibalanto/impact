# Thread

Un thread es un hilo de discusión estructurado asociado a uno o más [Impact Reports](impact-report.md). Es el espacio donde humanos y agentes evalúan si un cambio es coherente con las capas linkedeadas y deciden cómo resolverlo.

## Estructura en el filesystem

```
.impact/threads/<uuid>/
  thread.md          ← metadata y estado del thread
  messages/
    0001.md          ← mensaje inicial (el Impact Report)
    0002.md          ← respuesta (humano o agente)
    0003.md
    ...
```

### `thread.md` lleva la metadata y el estado

```yaml
---
id: <uuid>
title: <string>
status: open | resolved | discarded
created_at: <iso8601-utc>
resolved_at: <iso8601-utc>   # solo si status != open
reports:
  - <impact-report-uuid>
chains:
  - <chain-uuid>
---
```

### Cada mensaje es un archivo Markdown con frontmatter

```yaml
---
seq: <número>
author: <did | "human">
kind: report | analysis | reply | resolution
created_at: <iso8601-utc>
---

<cuerpo en Markdown>
```

### `seq` define el orden, y nombra el archivo

Los archivos se nombran con `seq` zero-padded a 4 dígitos.

### `kind` dice qué es cada mensaje

- `kind: report` es siempre el mensaje `0001.md`: el Impact Report que abrió el thread.
- `kind: analysis` es una opinión generada por un agente.
- `kind: reply` es una respuesta humana o de agente al hilo.
- `kind: resolution` cierra el thread con una decisión explícita.

## Ciclo de vida

```
open → resolved   (se tomó una decisión)
     → discarded  (falsa alarma o irrelevante)
```

### Un thread `resolved` tiene siempre un mensaje de `kind: resolution` como último mensaje

## Relación con Impact Reports

### Un thread puede agrupar múltiples Impact Reports

Si describen el mismo evento o cambios relacionados en la misma cadena. El primer mensaje del thread es siempre el reporte que lo abrió.

## Relación con Accreta

### Un thread de impact puede convertirse en una `Discussion` de Accreta

Arrastrando su historial de mensajes. La resolución del thread puede generar una `Iteration` propuesta sobre la spec o el ADR afectado.

## Los comandos

```
impact thread new <report-uuid> [--title <string>]
impact thread list [--status open|resolved|discarded]
impact thread show <thread-uuid>
impact thread reply <thread-uuid> [--kind reply|analysis|resolution] [--message <text>]
impact thread resolve <thread-uuid>
impact thread discard <thread-uuid>
```

### `impact thread new` abre un thread a partir de un Impact Report existente

1. Verifica que el report existe en `.impact/reports/`.
2. Crea `.impact/threads/<uuid>/thread.md` con `status: open`.
3. Copia el Impact Report como `0001.md` (`kind: report`).
4. Imprime el UUID del thread creado.

```
thread abierto: 7f3d8e9a
  title: cambio en src/Persona.java afecta chain abc12345
  .impact/threads/7f3d8e9a-...
```

### `impact thread list` lista todos los threads con su estado y título

```
7f3d8e9a  open      cambio en src/Persona.java afecta chain abc12345
a1b2c3d4  resolved  actualizar spec voting.yaml tras refactor
```

### `impact thread show` muestra el contenido completo de un thread

Metadata y todos los mensajes en orden de secuencia.

### `impact thread reply` agrega el siguiente mensaje en secuencia

El texto puede venir de `--message` o de stdin si se omite.

```
impact thread reply 7f3d8e --kind analysis --message "El cambio es compatible, el invariante sigue válido"
```

Crea el siguiente archivo en secuencia (`0002.md`, `0003.md`, etc.).

### `impact thread resolve` cierra con una resolución, y sólo un thread `open`

Agrega un mensaje `kind: resolution` y cambia el estado a `resolved`. Requiere que el thread esté `open`.

### `impact thread discard` marca el thread como `discarded`

Para falsos positivos o cambios irrelevantes.

### Los flags y los códigos de salida son comunes a los subcomandos

| Flag | Descripción |
|------|-------------|
| `--status <estado>` | Filtra por estado en `list` |
| `--title <string>` | Título del thread en `new` |
| `--kind <kind>` | Tipo de mensaje en `reply` |
| `--message <text>` | Cuerpo del mensaje en `reply` |

| Código | Condición |
|---|---|
| `0` | operación exitosa |
| `1` | thread o report no encontrado, estado inválido, o error |

## Invariantes

### `id` del thread coincide con el nombre del directorio

### `0001.md` siempre existe y tiene `kind: report`

### Los números de secuencia son contiguos y sin gaps

### Un thread no puede pasar de `resolved` o `discarded` de vuelta a `open`
