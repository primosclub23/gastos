# NOTAS TÉCNICAS · Panorama (Dashboard de Gastos)

Documento vivo. Se actualiza en el mismo commit que el cambio que documenta.

---

## 1. Qué es esto

Dashboard personal de control de gastos/ingresos. Un único archivo `index.html`
autocontenido (Tailwind CDN + Chart.js CDN, sin build step, sin frameworks).
Categorías, movimientos, recurrentes, recordatorios de vencimiento, importación
de CSV bancario con autocategorización, y sincronización de datos vía GitHub.

## 2. Arquitectura (estado actual)

```
┌─────────────────┐        ┌──────────────────────┐        ┌───────────────────────┐
│  Mac (local)     │        │  github.com/          │        │  github.com/           │
│  Claude Code     │──push─▶│  primosclub23/gastos  │        │  primosclub23/         │
│  ~/Desktop/gastos│        │  (público, solo código)│        │  gastos-data (privado) │
└─────────────────┘        └──────────┬────────────┘        └───────────┬────────────┘
                                       │ GitHub Pages                     │ API REST (fetch)
                                       │ auto-build (~1 min)              │ desde el navegador
                                       ▼                                  ▲
                     primosclub23.github.io/gastos/  ───── lee/escribe ──┘
                     (el dashboard que abres tú)          data.json
```

Dos repos, con roles completamente distintos — no confundirlos:

| Repo | Visibilidad | Contiene | Cómo se actualiza |
|---|---|---|---|
| `gastos` | **Público** | Solo el código (`index.html`). Sin datos. | `git push` desde Claude Code en la Mac |
| `gastos-data` | **Privado** | `data.json` con tus movimientos/categorías/recurrentes reales | Automático, desde el propio dashboard (API de GitHub) |

## 3. Por qué esta arquitectura y no otra (decisiones + motivos)

- **GitHub Pages en vez de un VPS/hosting con PHP**: se descartó porque GitHub
  Pages solo sirve estático (HTML/CSS/JS), no ejecuta PHP. Al principio se
  montó una versión con backend en PHP (pensada para Hostinger o para una NAS
  UGREEN futura, ver sección 7) pero mientras no se tenga esa NAS, no hacía
  falta un servidor propio para lo que se necesitaba: solo una URL fija +
  guardado automático real.
- **GitHub como "base de datos" en vez de backend propio**: usar la API REST
  de GitHub (`PUT /repos/{owner}/{repo}/contents/{path}`) para leer/escribir
  un `data.json` en un repo privado, directamente desde el navegador con
  `fetch()`. Sin servidor, sin coste, y de regalo queda un historial completo
  de commits = historial de cómo ha evolucionado la economía mes a mes.
- **Repo de datos separado del repo de código**: el código (`gastos`) es
  público a propósito, para poder usar GitHub Pages gratis. Los datos NUNCA
  deben estar ahí — de ahí el repo `gastos-data`, privado, solo para el JSON.
- **Token fine-grained, no classic**: el token de acceso está limitado
  *únicamente* al repo `gastos-data`, con permiso "Contents: Read and write"
  y nada más. Si algún día se filtrara, el daño máximo posible es ese repo,
  no la cuenta de GitHub completa.
- **El token vive solo en `localStorage` del navegador**, nunca en el código
  ni en ningún repo. Se configura una vez desde el botón "Sincronización"
  del propio dashboard. Si cambias de navegador/dispositivo, hay que
  volver a introducirlo ahí.

## 4. Cómo funciona el guardado automático (por dentro)

- Cada vez que cambia algo (añadir/editar/borrar movimiento, categoría,
  recurrente...) se llama a `render()`, que al final dispara
  `scheduleGhSave()` — un debounce de 1.5s antes de escribir, para no
  hacer una llamada a la API por cada tecla.
- `ghSave()` hace `PUT` a la API de GitHub con el JSON completo
  (categorías + movimientos + recurrentes + contadores de secuencia)
  codificado en base64 (UTF-8 seguro, por los acentos).
- Necesita el `sha` del archivo actual para poder sobrescribirlo (así
  funciona la API de GitHub, es su forma de evitar sobrescrituras
  accidentales). Se guarda en la variable `ghFileSha` tras cada
  lectura/escritura. Si llega un 409 (conflicto de `sha`), se reintenta
  una vez releyendo el `sha` más reciente.
- Al cargar la página, `ghLoad()` hace `GET` del `data.json`, lo decodifica,
  y puebla `CATEGORIES` / `TRANSACTIONS` / `RECURRING` antes del primer
  render. Si el archivo no existe todavía (primera vez), no falla — arranca
  vacío y lo crea en el primer guardado.
- Indicador visual junto al título ("☁ Sincronizado" / "● Cambios sin
  guardar…" / "⚠ Error de sincronización") para saber en todo momento si
  el último cambio ya se guardó.

## 5. Estructura de datos (`data.json`)

```json
{
  "categories": { "clave": { "label", "color", "kind": "expense|income", "group": "fijo|variable|null", "order", "deleted", "icon" } },
  "transactions": [ { "id", "date", "concept", "cat", "freq": "Fijo|Variable", "amount", "seriesId" } ],
  "recurring": [ { "id", "concept", "cat", "amount", "dayOfMonth", "startDate", "endDate", "reminderDays" } ],
  "txSeq": 0,
  "recSeq": 0,
  "orderSeq": 100
}
```

- `transactions` **no incluye** las filas generadas por recurrentes
  (`seriesId` no nulo) — esas se regeneran siempre a partir de `recurring`
  vía `regenerateAllRecurring()`, para no duplicar datos.
- `amount` siempre viene con signo: negativo = gasto, positivo = ingreso.
- Categorías eliminadas no se borran del todo (`deleted: true`) — quedan
  restaurables desde el panel de "Categorías".

## 6. Flujo de actualización del código (a partir de ahora)

Ya NO se reemplaza el `index.html` entero en cada cambio. El ciclo real es:

1. Se pide el cambio en el chat de Claude (claude.ai).
2. Claude da el fragmento exacto a buscar/sustituir dentro de `index.html`
   (no el archivo completo).
3. Se le pasa a **Claude Code**, ya con el repo clonado en
   `~/Desktop/gastos`, pidiéndole explícitamente que enseñe el `git diff`
   antes de comitear.
4. Revisión del diff línea por línea (no del resumen).
5. `git add index.html` (nunca `git add .` a ciegas).
6. Commit descriptivo del qué y el porqué.
7. `git push`.
8. GitHub Pages reconstruye solo (~1 min). Refrescar la URL para ver el
   cambio.

Los datos (`gastos-data`) no se tocan nunca en este proceso — son
completamente independientes del ciclo de despliegue del código.

## 7. Plan a futuro: migración a NAS propia

Hay pendiente la compra de una NAS (UGREEN DXP2800 barajada, con Docker
y VPN Server —WireGuard/OpenVPN— nativos en UGOS Pro). Cuando se tenga:

- Existe ya preparada una versión del dashboard con backend PHP propio
  (login con contraseña + guardado en un `data.json` en el propio servidor,
  vía `api/load.php` / `api/save.php`), pensada tanto para Hostinger como
  para Docker en la NAS.
- Migración de datos: exportar el `data.json` actual desde `gastos-data`
  (o usar el botón "Exportar" del dashboard) e importarlo en la nueva
  instalación — mismo formato, sin conversión necesaria.
- Ventaja de la NAS sobre el esquema actual: acceso también por VPN sin
  depender de GitHub como intermediario, y backups a nivel de RAID/snapshot
  del propio NAS.

## 8. Cosas que se rompieron y por qué (lecciones)

- **Bug: "Guardar movimiento" no guardaba nada.** Causa: cada toggle
  (Fijo/Variable, Gasto/Ingreso...) volvía a pintar el formulario entero
  (`innerHTML`) desde cero, y los campos de texto/fecha/importe no estaban
  en variables persistentes — se leían directo del DOM. Si el usuario
  tocaba un toggle *después* de rellenar el formulario, esos valores se
  perdían silenciosamente. Fix: todos los campos de texto ahora viven en
  variables de cierre (`dateVal`, `conceptVal`, `amountVal`...) que se
  reinyectan en cada repintado.
- **Bug: categorías creadas desde "Añadir movimiento" no aparecían en el
  banner.** Causa: el callback de creación no llamaba a la función que
  reconstruye el listado de categorías visibles. Fix: cada punto de
  creación de categoría refresca explícitamente la UI tras crear.
- **Decisión: eliminar categorías es "soft delete", no borrado real.**
  Motivo: un borrado real habría dejado movimientos históricos apuntando
  a una categoría inexistente (rompe el render). Con `deleted: true` se
  pueden ocultar y restaurar sin perder ni romper nada.
- **Import CSV vs Restaurar JSON — mismo botón por error de usuario.**
  Al confundir "Importar CSV" (para extractos bancarios) con "Restaurar"
  (para copias de seguridad `.json`), el parser de CSV fallaba con un
  mensaje confuso. Fix: el importador de CSV ahora detecta si el archivo
  es en realidad JSON y redirige automáticamente a la restauración.

## 9. Cosas a vigilar / deuda técnica conocida

- El repo `gastos-data` acumula un commit por cada guardado automático.
  No es un problema técnico (son diffs de texto pequeños), pero con el
  tiempo el historial de commits será largo — es solo estético, no requiere
  acción.
- El token de sincronización se guarda por navegador/dispositivo. Si se
  usa el dashboard desde el móvil también, hay que configurar la
  sincronización ahí por separado con el mismo token (o generar uno nuevo).
- No hay control de "quién guardó qué y cuándo" más allá del propio
  historial de commits de `gastos-data` (autor siempre es el dueño del
  token, no hay multi-usuario).
