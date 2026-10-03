# Contabilidad Comida Fast (APK)

App Android de inventario y caja para un puesto de comida rápida. Funciona sin internet y guarda los datos dentro de la propia app.

## Obtener el APK
1. Crea un repositorio en github.com (mejor **privado**, porque incluye la llave de firma `debug.keystore`).
2. Sube TODO el contenido de esta carpeta, incluida la carpeta oculta `.github`. Si no se sube, créala con Add file > Create new file, nombre `.github/workflows/build-apk.yml`, y pega su contenido.
3. Pestaña **Actions**: la compilación inicia sola (3 a 6 min).
4. Con el check verde, abre la ejecución y descarga **comida-fast-apk** (Artifacts). Dentro está `app-debug.apk`.
5. Instálalo en el teléfono (acepta "origen desconocido" y, si aparece, Play Protect > Instalar de todos modos).

## Actualizaciones
Mientras uses siempre el archivo `debug.keystore` incluido, cada APK nuevo se instala **encima del anterior sin perder los datos**. No lo borres ni lo cambies. Si lo pierdes, guárdalo fuera de GitHub también.

## Ícono y pantalla de inicio
Están ya hechos en la carpeta `android-res/` y se copian solos en cada compilación (paso "Ícono y pantalla de inicio" de Actions). No hace falta tocar nada.

## Datos
Se guardan en el almacenamiento interno de la app. Se pierden si desinstalas la app o borras sus datos en Ajustes de Android. La app ya no incluye copia de seguridad.
