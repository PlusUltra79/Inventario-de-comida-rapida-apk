# Especificación maestra: "Control de Puesto" (plantilla de apps de inventario y caja)

> **Para Claude (instrucciones de lectura):** este documento describe una app ya construida y probada en la práctica, y cómo clonarla para otros negocios (postres, ropa, etc.). Léelo completo antes de responder. Si el usuario adjunta `index.html` (la implementación de referencia), úsalo como base y modifícalo en lugar de reescribir desde cero. Responde siempre en español, de forma directa y práctica: **el usuario es programador principiante**.

---

## 1. Qué es el producto

App móvil **offline, de un solo usuario**, para llevar inventario y caja de un negocio pequeño. Versión original: puesto de perros calientes.

Flujo de uso diario:
1. Ver el **stock** (unidades o kg) y qué falta por comprar.
2. **Comprar** mercancía: la cantidad se **suma** al stock existente.
3. **Vender** durante el día: botones + / − por combo y por producto suelto; cada venta **descuenta del stock** los ingredientes.
4. **Cierre del día:** el usuario cuenta lo que queda; la app calcula dinero generado, ganancia estimada y **diferencias** (conteo físico vs. sistema).
5. **Reportes** de hoy / 7 / 30 días y **copia de seguridad**.

Se entrega de 3 formas con **el mismo código**: página web, PWA instalable y APK Android (Capacitor).

## 2. Reglas y restricciones de diseño

- Sin servidor, sin cuentas, sin internet, sin dependencias externas. **HTML + CSS + JavaScript puro en un solo archivo `index.html`.**
- Interfaz en **español**, móvil primero, modo claro y oscuro, estados vacíos útiles ("Aún no hay productos. Crea el primero para comenzar").
- Confirmación antes de acciones destructivas (`confirm()`); validación antes de guardar; avisos con `toast`, nunca fallos silenciosos.
- Todo texto escrito por el usuario pasa por `esc()` antes de insertarse en HTML (evita XSS).
- Nunca prometer funciones que no existen. Si algo requiere cuentas, claves o pagos, decirlo claramente.
- **Claude no puede compilar un APK en el chat** (no hay Android SDK). Se entrega el proyecto Capacitor + workflow de GitHub Actions que compila en la nube. Dilo siempre con transparencia.
- Descargas de archivos no funcionan en la página publicada como artifact; por eso la exportación se hace **copiando texto** (CSV/JSON) al portapapeles.

## 3. Estructura de archivos

**Página web / PWA (carpeta `control-de-puesto-pwa/`):**
```
index.html            app completa
manifest.webmanifest  datos de instalación
sw.js                 service worker (offline)
icon.svg              ícono
README.md
```

**APK (carpeta `control-de-puesto-apk/`):**
```
www/index.html                      la misma app (sin manifest ni SW)
package.json                        dependencias Capacitor
capacitor.config.json               appId, appName, webDir
.gitignore                          node_modules/ y android/
.github/workflows/build-apk.yml     compila el APK en GitHub
README.md
```

## 4. Modelo de datos (un único objeto `S` guardado como JSON)

```js
const KEY = 'controlpuesto.v1';           // cambiar por cada nicho: 'postres.v1', 'ropa.v1'
S = {
  cur: '$',                                // símbolo de moneda
  products: [{ id, name, unit:'pza'|'kg', cost, price, min, stock }],
  combos:   [{ id, name, price, items:[{ pid, qty }] }],
  sales:    [{ id, date:'YYYY-MM-DD', type:'combo'|'prod', ref, name, qty:1, amount }],
  purchases:[{ date, pid, qty, cost }],
  closes:   [{ date, revenue, profit, diffs:[{ name, unit, expected, counted, diff }] }]
}
```

- `id = Date.now().toString(36)+Math.random().toString(36).slice(2,6)` (función `uid()`).
- `price` en un producto = precio de **venta suelta** (0 o vacío = no se vende solo).
- `cost` = costo por unidad/kg; al comprar con costo total, `cost = total / cantidad`.
- Fechas locales (no UTC) con `today()`.
- Persistencia: `localStorage.setItem(KEY, JSON.stringify(S))`, con `try/catch` en lectura y escritura.
- **Si se cambia la estructura de `S` en una versión nueva, mantener compatibilidad:** al cargar usar `Object.assign(blank(), datosGuardados)` para rellenar campos nuevos.

## 5. Funcionalidad por pestaña (contrato de funciones)

Navegación inferior fija con 6 pestañas: **Inicio, Vender, Comprar, Catálogo, Cierre, Reportes**. `render()` dibuja la pestaña activa; cada pestaña es una función que **devuelve HTML**.

| Pestaña | Función | Qué hace |
|---|---|---|
| Inicio | `inicio()`, `extras()` | Ventas de hoy, lista "Qué comprar" (stock ≤ mínimo), stock actual, historial de cierres, copia de seguridad, restaurar, cambiar moneda, borrar todo, botón "Cargar datos de ejemplo" (`demo()`) |
| Catálogo | `catalogo()`, `prodForm(id)`, `saveProd(id)`, `delProd(id)`, `comboForm(id)`, `saveCombo(id)`, `delCombo(id)` | CRUD de productos y combos. No se puede eliminar un producto usado en un combo. Nombre de producto único (sin distinguir mayúsculas) |
| Comprar | `comprar()`, `buy()`, `buyHint()` | Suma cantidad al stock; costo total opcional actualiza `cost`; muestra últimas compras |
| Vender | `vender()`, `sell(type,id,d)`, `applyItems(items,mult)`, `count()` | `d=+1` registra venta y descuenta; `d=-1` deshace la última venta de hoy de ese ítem y restaura stock. Avisa (sin bloquear) si el stock es insuficiente |
| Cierre | `cierre()`, `closeDay()` | Un campo de conteo por producto; vacío = coincide con el sistema; calcula diferencias, iguala stock al conteo y guarda el cierre; muestra resumen en diálogo |
| Reportes | `reportes()`, `setRp()`, `saleCost()`, `copyCsv()` | Hoy / 7 / 30 días: ingresos, ganancia estimada (usa costo actual), gasto en compras, nº de ventas, gráfico de barras CSS, más vendido, CSV copiable |

Utilidades: `$()`, `esc()`, `uid()`, `today()`, `dstr()`, `n()` (número tolerante a coma decimal), `fmt()` (redondeo a 2 decimales), `money()`, `unit()`, `prod()`, `toast()`, `dlg()`, `closeDlg()`, `save()`, `go()`.

## 6. Reglas de negocio

1. **Venta de combo:** por cada ítem, `stock -= item.qty`. Venta suelta: `stock -= 1`.
2. **Productos por kg** normalmente no se descuentan por venta; se ajustan en el cierre por conteo (a menos que estén dentro de un combo con cantidad en kg).
3. **Compra:** `stock += cantidad`.
4. **Cierre:** `diferencia = contado − stock_sistema`; negativa = faltante (merma, regalo, error). Luego `stock = contado`.
5. **Ganancia estimada** = ingresos − costo de lo vendido (costo actual de cada producto × cantidad consumida). Es una **estimación**; decirlo en pantalla.
6. Stock bajo cuando `stock <= min`.

## 7. Diseño visual

- Tokens CSS en `:root` (`--bg, --card, --tx, --mut, --pri, --pri2, --ok, --bad, --bd`); modo oscuro con `@media (prefers-color-scheme: dark)` redefiniendo solo los tokens.
- **Color primario cambia por nicho** (original naranja `#d9480f`): postres rosado/chocolate, ropa azul marino/negro, etc.
- Componentes: `.card`, `.row`, `.grid`, botones (`.sec`, `.bad`, `.sm`), `dialog` para formularios, `#toast`, barra `nav` fija, `.empty` para estados vacíos.
- `viewport-fit=cover` y `env(safe-area-inset-*)` para teléfonos con muescas.
- Inputs numéricos con `inputmode="decimal"`.

## 8. Archivos de plantilla (copiar tal cual, cambiando solo lo indicado)

**`package.json`**
```json
{
  "name": "control-de-puesto",
  "version": "1.0.0",
  "private": true,
  "dependencies": {
    "@capacitor/android": "^6.1.2",
    "@capacitor/cli": "^6.1.2",
    "@capacitor/core": "^6.1.2"
  }
}
```
Cambiar `name` por el nicho (ej. `control-de-postres`). Mantener **Capacitor 6 + Java 17** (compatibles entre sí).

**`capacitor.config.json`**
```json
{ "appId": "com.mipuesto.control", "appName": "Control de Puesto", "webDir": "www" }
```
Cambiar `appId` (único por app, ej. `com.mipuesto.postres`) y `appName` (nombre bajo el ícono).

**`.gitignore`**
```
node_modules/
android/
```

**`.github/workflows/build-apk.yml`**
```yaml
name: Compilar APK
on:
  push:
    branches: [main]
  workflow_dispatch:
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: 20
      - uses: actions/setup-java@v4
        with:
          distribution: temurin
          java-version: 17
      - name: Instalar dependencias
        run: npm install
      - name: Crear proyecto Android
        run: |
          npx cap add android
          npx cap sync android
      - name: Compilar APK
        run: |
          cd android
          chmod +x gradlew
          ./gradlew assembleDebug
      - name: Guardar APK
        uses: actions/upload-artifact@v4
        with:
          name: control-de-puesto-apk
          path: android/app/build/outputs/apk/debug/app-debug.apk
```

**`manifest.webmanifest`** (solo PWA)
```json
{"name":"Control de Puesto","short_name":"Mi Puesto","lang":"es","start_url":"./index.html","scope":"./","display":"standalone","background_color":"#faf6f0","theme_color":"#d9480f","icons":[{"src":"icon.svg","sizes":"any","type":"image/svg+xml","purpose":"any"}]}
```

**`sw.js`** (solo PWA; **subir la versión `v2`→`v3` en cada actualización**)
```js
const C='control-puesto-v2',F=['./','./index.html','./manifest.webmanifest','./icon.svg'];
self.addEventListener('install',e=>{e.waitUntil(caches.open(C).then(c=>c.addAll(F)));self.skipWaiting()});
self.addEventListener('activate',e=>{e.waitUntil(caches.keys().then(k=>Promise.all(k.filter(x=>x!==C).map(x=>caches.delete(x)))));self.clients.claim()});
self.addEventListener('fetch',e=>{if(e.request.method!=='GET')return;e.respondWith(caches.match(e.request).then(r=>r||fetch(e.request).catch(()=>caches.match('./index.html'))))});
```

**En `index.html` de la PWA** añadir en `<head>`: `<link rel="manifest" href="manifest.webmanifest"><link rel="icon" href="icon.svg"><meta name="theme-color" content="#d9480f">` y antes de `</body>`: `<script>if("serviceWorker"in navigator)addEventListener("load",()=>navigator.serviceWorker.register("sw.js").catch(()=>{}))</script>`. **La versión APK (`www/index.html`) NO lleva manifest ni service worker.**

**`icon.svg`**: cuadrado redondeado del color primario con un emoji del nicho (🌭, 🧁, 👕).

## 9. Adaptación por nicho (qué cambiar)

**Siempre cambiar:** `KEY` de almacenamiento, título y textos, emoji del ícono y favicon, color `--pri`, `appId`, `appName`, `name` del `package.json`, datos de `demo()`.

**Vocabulario:** "combo" puede pasar a "paquete", "kit", "caja" o "producto elaborado". "Puesto" a "tienda", "pastelería", etc.

### A. Puesto de postres / pastelería
- **Insumos** (harina, azúcar, huevos, en kg o pza) y **productos elaborados** (torta, brownie) = el "combo" actual funciona como **receta**: lista de insumos con cantidades por unidad producida.
- Cambio de lógica recomendada: separar **producir** de **vender**. Pestaña "Producir": al producir N unidades se descuentan insumos de la receta y se **suman** a un stock de producto terminado. Vender descuenta del stock terminado.
- Campos extra: fecha de vencimiento, merma por producto, porciones por receta.
- Demo: harina, azúcar, huevos, mantequilla, chocolate; recetas de brownie, torta de vainilla, cupcakes.

### B. Tienda de ropa
- No usa kg. Todo es `pza`; se elimina la unidad kg o se oculta.
- **Variantes:** cada producto tiene talla/color con su propio stock. Modelo sugerido: `products[i].variants=[{id, talla, color, stock}]` y la venta registra `variantId`. El stock total es la suma de variantes.
- Campos extra: SKU o código, precio de venta, **descuentos** por venta, apartados/fiado (cliente, saldo).
- "Combos" se reinterpretan como **conjuntos** (ej. camisa + pantalón con precio especial).
- Demo: 6 prendas con tallas S/M/L y 2 colores.

### C. Otros (cafetería, ferretería, accesorios)
Reusar el modelo tal cual; ajustar unidades, textos y demo. Para ferretería agregar venta por metros/litros (extender `unit`).

## 10. Procedimiento para generar una versión nueva (para Claude)

1. **Preguntar solo lo imprescindible** (máx. 4 preguntas cortas): ¿qué negocio es? ¿qué productos y unidades? ¿hay recetas/variantes/vencimientos? ¿hay venta combinada (combos/paquetes)? Si falta algo, **declarar suposiciones razonables** y avanzar.
2. **Proponer un plan breve:** cambios al modelo de datos, pestañas nuevas o modificadas, supuestos.
3. **Construir** modificando la implementación de referencia: mismo patrón *modificar datos → `save()` → `render()`*.
4. **Probar en el chat:** publicar el `index.html` como artifact para que el usuario lo pruebe en el teléfono (el almacenamiento del navegador funciona en la página publicada).
5. **Empaquetar:** carpeta PWA y carpeta APK con los archivos de la sección 8 adaptados, más README con instrucciones.
6. **Entregar** el zip y explicar cómo compilar con GitHub (sección 11).
7. **Validar** con la lista de la sección 12. No declarar "probado en teléfono" si no se probó.

## 11. Cómo obtiene el usuario el APK

1. Crear cuenta en github.com y un repositorio nuevo.
2. Subir **todo** el contenido de la carpeta APK, incluida la carpeta oculta `.github`. Si no se sube, crearla en GitHub: *Add file > Create new file*, nombre `.github/workflows/build-apk.yml`, y pegar el contenido.
3. Pestaña **Actions**: la compilación inicia sola (3 a 6 min). Si no, "Compilar APK" > *Run workflow*.
4. Al terminar (check verde), abrir la ejecución y descargar el artifact (zip con `app-debug.apk`).
5. Pasar el APK al teléfono e instalarlo. Android mostrará un aviso de "app desconocida" y Play Protect puede bloquear: es normal en APK *debug*. Usar *Más detalles > Instalar de todos modos* o desactivar temporalmente el análisis de Play Protect.

Para la PWA: publicar la carpeta en GitHub Pages (Settings > Pages > rama `main`) o arrastrarla a Netlify Drop, abrir la URL en Chrome y elegir *Instalar aplicación*.

## 12. Lista de pruebas manuales

- Crear producto sin nombre: debe avisar y no guardar.
- Crear producto duplicado: debe rechazarlo.
- Crear combo sin productos o sin precio: debe rechazarlo.
- Eliminar un producto que está en un combo: debe impedirlo.
- Vender un combo: el stock baja según sus ingredientes; el botón − lo revierte.
- Vender con stock insuficiente: aviso, sin bloqueo.
- Comprar mercancía: el stock suma sobre el existente.
- Cierre con conteo menor: aparece diferencia negativa; el stock queda igual al conteo.
- Reportes: totales coinciden con lo vendido; estado vacío sin ventas.
- Copia de seguridad: copiar, borrar todo, restaurar y verificar que todo vuelve.
- Restaurar texto inválido: error claro, sin borrar datos actuales.
- Pantalla móvil y modo oscuro: legibles, sin desbordes.
- (PWA) Modo avión: abre y funciona. (APK) Cerrar y reabrir: los datos persisten.

## 13. Lecciones aprendidas y trampas conocidas

- **Datos separados:** el navegador, la PWA instalada y el APK **no comparten datos** (cada uno tiene su propio `localStorage`). Migrar entre ellos = copia de seguridad.
- **Desinstalar el APK borra los datos.** La copia de seguridad (texto JSON copiable) es el seguro.
- **Actualizar el APK:** cada compilación *debug* se firma con una llave distinta, así que **no se instala encima de la anterior**. Solución profesional: keystore fija como secreto de GitHub y compilar *release* firmado (nunca subir la keystore al repositorio).
- **Caché del service worker:** si el cambio no se ve, subir la versión del caché en `sw.js`.
- **Carpeta `.github` oculta:** causa más común de "Actions no muestra nada".
- **Descargas:** los enlaces de descarga y `Blob` no funcionan en la página publicada; usar copiar al portapapeles. Dentro del APK, una exportación real a archivo requeriría plugins de Capacitor (Filesystem/Share), no implementados ni probados.
- **No verificado en dispositivo real:** `prompt()` y `confirm()` en el WebView de Capacitor, y el portapapeles dentro del APK. Si fallan, reemplazarlos por diálogos propios (`dialog`).
- **Ícono del APK:** por defecto es el de Capacitor; personalizar con `@capacitor/assets`.

## 14. Mejoras futuras (hoja de ruta)

1. Gastos fijos (gas, alquiler, empleados) para ganancia real.
2. Keystore propia y APK *release* firmado (actualizaciones sin perder datos).
3. Almacenamiento más robusto en el APK (plugin `@capacitor/preferences` o SQLite).
4. Exportar CSV/Excel a archivo con Filesystem + Share.
5. PIN de acceso, múltiples sucursales, sincronización en la nube (Supabase).
6. Ícono y pantalla de inicio personalizados por nicho.
7. Posible versión comercial (SaaS o plantilla de pago por nicho).

## 15. Prompt listo para pegar en un chat nuevo

> Te adjunto la **especificación maestra** y el archivo `index.html` de referencia de "Control de Puesto". Quiero crear una versión nueva para **[NICHO: ej. puesto de postres]**. Productos principales: **[lista]**. Unidades: **[pza/kg/otras]**. Particularidades: **[recetas, tallas y colores, vencimientos, fiado, etc.]**. Moneda: **[símbolo]**.
>
> Haz lo siguiente: (1) dime tus suposiciones y haz como máximo 3 preguntas si hace falta; (2) propón qué cambia del modelo de datos; (3) genera el `index.html` adaptado y publícalo para probarlo; (4) prepara las carpetas PWA y APK con los archivos de plantilla (cambia `KEY`, `appId`, `appName`, color, ícono y datos demo); (5) dame un zip y las instrucciones para compilar en GitHub; (6) incluye la lista de pruebas. Sé honesto sobre lo que no puedes probar. Soy programador principiante: explícame cada paso de forma sencilla.
