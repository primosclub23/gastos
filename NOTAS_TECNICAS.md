# NOTAS TÉCNICAS · Numo (antes Panorama, Dashboard de Gastos)

Documento vivo. Se actualiza en el mismo commit que el cambio que documenta.

---

## 1. Qué es esto

Dashboard personal de finanzas, hoy llamado **Numo** (el `<title>` y el logo
`logo-numo.png.png`; el nombre anterior era Panorama). Un único archivo
`index.html` autocontenido (Tailwind CDN + Chart.js CDN, sin build step, sin
frameworks). Empezó como control de gastos/ingresos y desde julio de 2026
incluye también cuentas, inversiones, alertas y dividendos (ver sección 5b).
Sincronización de datos vía GitHub.

## 2. Arquitectura (estado actual)

```
┌─────────────────┐        ┌──────────────────────┐        ┌───────────────────────┐
│  Mac (local)     │        │  github.com/          │        │  github.com/           │
│  Claude Code     │──push─▶│  primosclub23/gastos  │        │  primosclub23/         │
│  Proyectos/gastos│        │  (público, solo código)│        │  gastos-data (privado) │
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

- Cada acción que modifica datos de verdad (añadir/editar/borrar movimiento,
  recurrente, categoría, alerta, cuenta, dividendo, importar CSV...) llama
  **explícitamente** a `scheduleGhSave()` — un debounce de 1.5s antes de
  escribir. **`render()` ya NO guarda** (desde el commit `100ed22`, 22 jul
  2026; ver sección 8). Si se añade una acción nueva que cambia datos, hay
  que llamar a `scheduleGhSave()` a mano o no se guardará.
- `scheduleGhSave()` no hace nada hasta que `ghDataLoaded` sea `true`, es
  decir, hasta que una carga desde GitHub se complete con éxito.
- `ghSave()` hace `PUT` a la API de GitHub con el JSON completo
  (categorías, movimientos, recurrentes, cuentas, alertas, dividendos,
  cobros de dividendos y contadores de secuencia) codificado en base64 (UTF-8 seguro, por los acentos).
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
  "accounts": [ ... ],
  "alerts": [ { "id", "title", "dayOfMonth", "active" } ],
  "dividends": [ { "id", "accountId", "symbol", "annualYieldPct", "infoUrl", "updatedAt" } ],
  "dividendReceipts": [ ... cobros reales por mes ... ],
  "txSeq": 0, "recSeq": 0, "orderSeq": 100,
  "accSeq": 0, "alertSeq": 0, "divSeq": 0, "receiptSeq": 0
}
```

- `transactions` **no incluye** las filas generadas por recurrentes
  (`seriesId` no nulo) — esas se regeneran siempre a partir de `recurring`
  vía `regenerateAllRecurring()`, para no duplicar datos.
- `amount` siempre viene con signo: negativo = gasto, positivo = ingreso.
- Categorías eliminadas no se borran del todo (`deleted: true`) — quedan
  restaurables desde el panel de "Categorías".

## 5b. Vistas y funciones (estado a 2026-10-16)

Reconstruido a partir de los commits y del código. Navegación lateral (cada
entrada es una vista independiente):

- **Panel** — gráfico de *Patrimonio Total* (suma de todas las cuentas, USD
  convertido a EUR; filtros Anual/Mensual, por año y por cuenta; una línea de
  color por cuenta más una discontinua con el total) y una tarjeta de
  **Libertad Financiera**.
- **Control de Gastos** — KPIs y gráficas (la vista de entrada). Con dos
  sub-ítems: **Categorías** (abre el modal de gestión) y **Transacciones**
  (tabla con buscador, filtros propios de año/mes/categoría, sumatorio y fila
  de total; el campo Concepto autocompleta con conceptos ya usados y rellena
  categoría, frecuencia, tipo y logo).
- **Cuentas** — tipos: Corriente, Tarjeta de Crédito, Inversión, Préstamo,
  Bienes Raíces, Jubilación/Pensión, Ahorros, Otros, **Acciones** y **Cripto**.
  Las dos últimas (`LOT_ACCOUNT_TYPES`) guardan **compras individuales**
  (fecha, precio, cantidad; editables) y calculan por activo: unidades,
  precio medio ponderado, valor actual y revalorización (PNL, € o $ y %).
  Cada cuenta puede ser en € o $ y tiene un color propio. Se guarda un
  histórico de saldos para la gráfica. Hay una migración automática de las
  cuentas antiguas de un solo activo al formato multi-activo.
- **Dividendos** — cada entrada es un activo conectado a una cuenta de
  Acciones/Cripto, con un **% anual de rentabilidad** (y enlace opcional a la
  web del fondo). La app estima el ingreso mensual/anual (valor actual × % /
  12). El botón **"Registrar cobro del mes"** guarda los cobros reales, que
  alimentan el gráfico de evolución (barras por activo + línea total, con %
  de variación y acumulado del año).
- **Alertas** — recordatorios mensuales por día del mes. Icono de campana en
  el header con el contador de pendientes. El estado "hecha" solo dura la
  sesión (no persiste al recargar).
- **Perfil / Sincronización** — Importar CSV, Exportar, Restaurar y la
  configuración del token de GitHub.

**Libertad Financiera**: compara los ingresos pasivos con los **Gastos Fijos
del último mes cerrado**. Usa el cobro real de ese mes si está registrado y,
si no, la estimación por % anual.

**Header**: el botón principal depende de la vista ("Añadir movimiento" o
"Crear Cuenta"). Modo oscuro (variables CSS `--color-*` y
`html[data-theme="dark"]`, guardado en localStorage) y **modo privado** (difumina
importes).

**APIs externas** (llamadas directas desde el navegador, sin clave):
CoinGecko (precios cripto, en EUR o USD), Yahoo Finance (cotización de
acciones por ticker), Frankfurter (tipo de cambio USD→EUR) y jsDelivr (iconos
de criptomonedas, con emoji de respaldo).

## 6. Flujo de actualización del código (a partir de ahora)

Ya NO se reemplaza el `index.html` entero en cada cambio. El ciclo real es:

1. Se pide el cambio en el chat de Claude (claude.ai).
2. Claude da el fragmento exacto a buscar/sustituir dentro de `index.html`
   (no el archivo completo).
3. Se le pasa a **Claude Code**, ya con el repo clonado en
   `~/Desktop/IA/Cerebro/Proyectos/gastos`, pidiéndole explícitamente que enseñe el `git diff`
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
- **Divergencia de historial git (19 jul 2026).** En algún momento se subió
  `index.html` directamente desde la web de GitHub (botón "Upload files"),
  en vez de por `git push` desde la Mac. Eso creó un commit en `origin/main`
  sin ningún ancestro común con el historial local, y el siguiente push
  normal fue rechazado. Se resolvió con `git reset --hard origin/main` +
  `cherry-pick` del commit que faltaba (sin pérdida de contenido, el archivo
  ya era idéntico en ambos sitios). Regla a partir de ahora: `index.html`
  SIEMPRE se actualiza vía `git push` desde Claude Code, nunca subiendo el
  archivo a mano desde la web de GitHub — evita que el historial local y
  el remoto se desincronicen.

- **Fallo crítico: la app borró datos sincronizados con GitHub (corregido
  el 22 jul 2026, commit `100ed22`).** Dos bugs combinados: (1) `render()`
  guardaba en GitHub en cada refresco de pantalla, aunque no hubiera
  cambios; (2) si la carga inicial desde GitHub fallaba (red, token...), la
  app marcaba igualmente los datos como cargados con las listas vacías, y
  el siguiente refresco sobrescribía el archivo remoto con ese estado vacío.
  Fix: se quitó el guardado de `render()`, solo guardan las acciones que
  modifican datos, y no se programa ningún guardado hasta que una carga
  termine con éxito (`ghDataLoaded`). Regla: **un guardado nunca debe poder
  ocurrir antes de una carga correcta.** El historial de `gastos-data` en
  GitHub permite recuperar versiones anteriores.
- **Cripto mezclaba activos.** Una cuenta guardaba un único símbolo/precio
  para toda la cuenta, así que al juntar BTC y XRP los totales no tenían
  sentido. Ahora cada compra lleva su propio activo y hay `currentPrices`
  por activo; las cuentas antiguas se migran solas al abrirlas.
- **Compras antiguas sin `_id`.** Las compras ya guardadas no recibían `_id`
  al reabrir la cuenta (salía `undefined`), lo que rompía editar/borrar
  compras concretas.
- **`<input type="number">` rechaza la coma decimal** según el idioma del
  navegador. Para el % de dividendo se usa texto libre con el parser
  flexible de la importación CSV ("7,75" o "7.20").
- **Modal cortado en pantalla** cuando tiene mucho contenido: el modal de
  cuentas lleva `max-h-[90vh]` con scroll interno.
- **Coma flotante al sumar cantidades** (0.1 + 0.05 salía 0.15000000000000002):
  corregido. Las cantidades se muestran con `fmtQty()` (hasta 4 decimales) pero
  se guardan con precisión completa.

## 9. Cosas a vigilar / deuda técnica conocida

- **`TODAY` ya no está fijada** (cambio del 16 oct 2026). Antes era
  `const TODAY = '2026-08-13'` y la app trataba esa fecha como "hoy". Ahora se
  calcula con la fecha del dispositivo, en zona horaria local y formato
  `AAAA-MM-DD`. Consecuencias a tener en cuenta: el "último mes cerrado" de
  Libertad Financiera es siempre el mes anterior al actual; si ese mes no
  tiene Gastos Fijos registrados, la tarjeta muestra un aviso en vez de un
  porcentaje; y los movimientos recurrentes se regeneran hasta el mes actual
  (no se guardan, se recalculan siempre a partir de `recurring`).
- Los precios dependen de APIs públicas (CoinGecko, Yahoo Finance,
  Frankfurter) llamadas desde el navegador. Si una cambia o limita el uso, el
  botón de precio automático dejará de funcionar (el precio siempre se puede
  introducir a mano).
- El tipo de cambio USD→EUR del Patrimonio Total se consulta en vivo a Frankfurter; si falla, se usa un valor de respaldo fijo de 0,92 (`usdToEurRate`).

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

## 10. Cómo mantener el Proyecto de Claude.ai al día

Este archivo se lee desde un Proyecto de Claude.ai vía la integración nativa
de GitHub (Project knowledge → conectar repositorio), NO subiéndolo a mano
cada vez. Flujo:

1. Claude Code actualiza NOTAS_TECNICAS.md localmente y hace push a GitHub
   (repo `gastos`, público).
2. En el Proyecto de Claude.ai, dentro de "Project knowledge", pulsar
   "Sync now" para traer la versión más reciente del archivo.
3. El sync es manual (no en tiempo real) — hay que acordarse de pulsarlo
   tras cada push relevante a este archivo.

Esto sustituye al método anterior de descargar el .md y volver a subirlo
a mano al Proyecto.
