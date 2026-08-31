# ADR-0001: impact — Orquesta, no escribe

**Estado:** Propuesto **Fecha:** 2026-08-31

**Lo dispara** [`bilinker/proposals/cierre-de-firma.md`](../../../../../bilinker/proposals/cierre-de-firma.md), que necesita un compositor: el vecindario de nivel 1 lo calcula lattice, la aceptación la guarda bilinker, y ninguno de los dos puede llamar al otro sin invertir las capas. Este ADR decide quién los compone y con qué límite.

---

## Contexto

Impact hoy se declara de solo lectura, en dos lugares:

`overview.md` § Principios de diseño:

> **No invasivo**: no modifica código ni specs, solo lee y reporta

`bilinker/integration/impact.md`:

> Impact no lee los `.bilink` ni invoca bilinker: consulta el grafo a través de lattice.

Y el flujo post-resolución reparte el trabajo entre dos herramientas y una persona: `bilinker accept <uuid>.<N>` y después `impact thread resolve`.

### El problema que aparece con el vecindario de nivel 1

La propuesta del cierre de firma agrega dos campos a `accepted` —`hash_n1` y `hash_ast_n1`— cuyo valor **bilinker no puede calcular**: resolver un tipo hasta su declaración es trabajo de language server, y la frontera de bilinker es *sólo git y tree-sitter*.

Eso deja tres piezas que hay que juntar y ningún lugar donde juntarlas:

| Pieza | Dueño |
|---|---|
| encontrar los vecinos (`textDocument/definition`, un salto) | lattice, proveedor `lsp` |
| hashear con el recorte de bordes y escribir `accepted` | bilinker |
| decidir si el cambio es coherente con una decisión declarada | impact |

**Y bilinker no puede pedirle a lattice.** No por el ciclo —un query scopeado con `--via` no invoca al proveedor `bilink`, así que en runtime no hay lazo— sino por la inversión de capas y su consecuencia concreta: `bilinker check` hoy funciona con git y tree-sitter, siempre, offline, en cualquier repo. Un `check` que necesita un language server indexando queda condicionalmente degradado, y eso contamina el subsistema entero por un campo. Lattice consume bilinker vía `bilinker graph`; al revés no.

### Qué protegía en realidad la regla de "no invoca bilinker"

`integration/lattice.md` lo dice cuando explica por qué se sacó el Chain Resolver:

> La versión anterior de impact escaneaba los `.bilink` por su cuenta. Eso metía **el formato de bilinker dentro de impact**: cualquier cambio en la topología de cadena había que arreglarlo en dos lugares.

La regla prohíbe **reimplementar el formato**, no delegarle al dueño. Correr `bilinker accept` es exactamente lo contrario de duplicar el formato: es no saber nada de él. La frase quedó escrita en términos de "no invoca" cuando lo que defendía era "no parsea".

### Y hay una demanda de uso que va en la misma dirección

El trabajo real sobre un drift es un recorrido, no un batch: mirar el grafo, profundizar, comparar contra el commit aceptado, y decidir. Hoy eso son tres invocaciones de dos herramientas para una sola decisión, y no hay ningún lugar donde el recorrido y la decisión estén juntos — ni para una persona ni para una IA.

---

## Decisión

### 1. El principio pasa a ser "orquesta, no escribe"

Impact **puede invocar** a bilinker. Impact **no puede parsear** su formato ni escribir sus archivos.

La distinción es la que preserva el invariante que `bilink.md` defiende como estructura:

> **`apply` escribe `link`. `accept` escribe `accepted`. `check` no escribe nada en el bilink.**

Un segundo escritor de `accepted` rompería eso. Un segundo *invocador* de `bilinker accept` no rompe nada: bilinker sigue siendo el único que escribe un byte de `accepted`, y sigue siendo el único que conoce el formato. Un campo, un escritor.

### 2. Impact es el compositor del vecindario de nivel 1

```
impact  →  lattice: dame los vecinos de nivel 1 de este nodo
impact  →  bilinker: hasheá estas ubicaciones, foldeá y escribí accepted
```

Bilinker **recibe** las ubicaciones; no sale a buscarlas. El hasheo queda de su lado porque el recorte de bordes es su regla y vive *"en el único lugar donde un nodo se convierte en rango"*. Bilinker no adquiere ninguna dependencia nueva: no habla con un language server ni sabe que lattice existe.

Para bilinker esos dos campos son **valores opacos** que guarda y compara sin poder derivar. No es una excepción nueva: es el patrón de `capture.md` invariante 6, donde un `accepted.link` de endpoint layer o repo contiene *"una copia opaca de un id ajeno, que no se resuelve localmente"*.

Cuando nadie le pasa el valor de hoy, `check` **no dice OK ni dice drift: dice no verificado.** Es la familia de `LAYER_UNREACHABLE` y `REMOTE_UNREACHABLE` — no pude ver el otro lado — y es la misma distinción que impact ya defiende para sí: *"un reporte que no distingue 'no encontré impacto' de 'no pude buscarlo' es peor que no tener reporte"*.

### 3. La superficie interactiva es de impact

Recorrer el grafo, profundizar, comparar entre commits, y aceptar o rechazar al final del recorrido. Impact es el único subsistema que ya consume los dos, y las piezas están:

| Lo que ya tiene | Para qué sirve en el recorrido |
|---|---|
| `commit` en cada arista | el baseline de `git log <commit>..HEAD` sin reabrir archivos del proveedor |
| `guarantee` | no presentar una inferencia del LSP como vínculo verificado |
| propagación de degradación | que quien mira sepa si vio menos grafo |
| `lattice graph --via … --up/--both` | el recorrido con profundización, ya especificado |

Lo que falta son los verbos: los comandos actuales —`scan`, `report`, `thread`, `watch`— son de consultora autónoma en batch. **Este ADR no decide el juego de comandos**, sólo que su lugar es impact. Ese diseño es trabajo de spec posterior.

### 4. Lo que no cambia

- Impact no resuelve paths Stratum ni habla con language servers. Sigue consultando lattice.
- Lattice sigue sin persistir nada.
- Bilinker sigue siendo git y tree-sitter.
- El formato de bilinker no cambia por este ADR. Los dos campos nuevos son de la propuesta del cierre de firma, y son aditivos.

---

## Consecuencias

**Dos frases propias que hay que corregir.** `overview.md` § Principios: *"No invasivo: no modifica código ni specs, solo lee y reporta"* → *"Orquesta, no escribe: puede invocar a las herramientas dueñas de cada formato, no escribe sus archivos ni parsea sus formatos"*. Y `bilinker/integration/impact.md`: *"no lee los `.bilink` ni invoca bilinker"* → *"no parsea los `.bilink`"*, con el flujo post-resolución pasando de dos pasos manuales a uno orquestado.

**Un estado nuevo que hay que nombrar.** *No verificado* para un campo que bilinker guarda y no puede derivar. `RESTYLED` y `ALTERED` no aplican porque el eje es otro: no es que el valor difiera, es que no hay con qué compararlo. Queda anotado como abierto en la propuesta del cierre de firma junto al vocabulario de los estados del vecindario.

**Impact gana una dependencia de ejecución sobre bilinker.** Hoy sólo depende de lattice. Un impact sin bilinker en el PATH puede seguir reportando —lectura— y no puede orquestar una aceptación. Es degradación, no falla, y entra en el esquema de degradación que ya propaga.

**Trabajo que dispara en las specs.** En impact: el principio en `overview.md`, y el juego de comandos interactivo, que es diseño nuevo. En bilinker: la sección de `cierre-de-firma.md` que reparte quién calcula, quién decide y quién guarda —ya escrita—, y la forma en que `accept` y `check` reciben ubicaciones de afuera. En lattice: un `kind` sobre el proveedor `lsp` que no es `call`, que exprese *"este fragmento menciona este tipo en su firma"*. Las tres son independientes; la de lattice es la más chica y no bloquea a las otras.

**No hace falta migración.** Nada de esto cambia un archivo existente: los campos son aditivos y la orquestación es una capa por encima de comandos que ya existen.

---

## Alternativas descartadas

- **Bilinker le pregunta a lattice.** El ciclo se evita con un query scopeado —`--via signature` no invoca al proveedor `bilink`—, pero la inversión de capas queda: `bilinker check` pasa a necesitar un language server para verificar un campo, y bilinker deja de funcionar solo. Es la propiedad que lo hace usable en cualquier repo y no se cambia por un campo.
- **La aceptación del nivel 1 como artefacto propio de impact**, sin tocar bilinker. Falla en el caso que motiva todo: lo que atraviesa una frontera de repos es la copia opaca de `accepted` del endpoint `repo`, así que un valor guardado del lado de impact no llega nunca al consumidor. Y la tabla de cuatro cuadrantes habría que armarla cruzando dos archivos de dos subsistemas.
- **Impact escribe `accepted` directamente.** Ahorra una invocación y rompe *un campo, un escritor*, que es lo que vuelve verificable el formato. Además le mete el formato de bilinker adentro, que es exactamente lo que la regla vieja protegía y que este ADR conserva.
- **Accreta como la fachada.** Es el dueño principiado —`overview.md` de impact lo ubica como *"gobierna la resolución del cambio"*, y `bilinker/integration/acreta.md` dice que la aceptación es un acto de gobernanza que accreta puede querer someter a votación. Pero accreta no existe como subsistema, y impact ya consume los dos. Cuando accreta exista, la política de gobernanza se apoya sobre esta orquestación en vez de reemplazarla.
- **Dejar el flujo en tres invocaciones.** Es el statu quo y no resuelve el problema que dispara el ADR: el vecindario de nivel 1 necesita un compositor, porque ninguno de los dos subsistemas puede llamar al otro. Aun sin la superficie interactiva, el compositor tiene que existir en algún lado.
- **Un cuarto subsistema sólo para orquestar.** Un lugar nuevo sin nada propio: consumiría lattice y bilinker igual que impact, y duplicaría su Event Collector y su propagación de degradación para no cambiarle una frase a un documento.
