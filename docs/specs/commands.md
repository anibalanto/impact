# Los comandos

`impact` expone cuatro comandos: dos que generan Impact Reports, uno que los genera al ritmo del filesystem, y uno con seis subcomandos que lleva los threads. El detalle de cada uno está en el concepto que lo explica.

| Comando | Uso | Qué hace |
|---|---|---|
| `scan` | `scan [--path <dir>] [--chain <uuid>]` | Consulta el grafo vía lattice, busca los archivos con commits no analizados desde su anchor y genera un Impact Report por cada uno ([impact-report.md](concepts/impact-report.md)). |
| `report` | `report <file>` · `report --bilink <uuid>` | Genera un Impact Report puntual para un archivo o para un vínculo ([impact-report.md](concepts/impact-report.md)). |
| `watch` | `watch [--path <dir>]` | Observa el filesystem y corre `scan` cada vez que cambia un archivo referenciado por un bilink ([impact-report.md](concepts/impact-report.md)). |
| `thread new` | `thread new <report-uuid> [--title <string>]` | Abre un thread a partir de un Impact Report existente ([thread.md](concepts/thread.md)). |
| `thread list` | `thread list [--status <estado>]` | Lista los threads con su estado y su título ([thread.md](concepts/thread.md)). |
| `thread show` | `thread show <thread-uuid>` | Muestra la metadata y todos los mensajes de un thread ([thread.md](concepts/thread.md)). |
| `thread reply` | `thread reply <thread-uuid> [--kind <kind>] [--message <text>]` | Agrega el siguiente mensaje en secuencia ([thread.md](concepts/thread.md)). |
| `thread resolve` | `thread resolve <thread-uuid>` | Cierra un thread `open` con un mensaje de resolución ([thread.md](concepts/thread.md)). |
| `thread discard` | `thread discard <thread-uuid>` | Marca un thread como `discarded` ([thread.md](concepts/thread.md)). |

Ninguno escribe fuera de `.impact/`: los elementos de impacto son bilinks, y los escribe bilinker ([flow.md](concepts/flow.md)).
