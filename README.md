# Control de Puesto (APK)

App Android para llevar el inventario y la caja de un puesto de perros calientes. Funciona sin internet y guarda los datos dentro de la propia app.

## Obtener el APK (gratis, sin instalar nada)
1. Crea una cuenta en github.com y un repositorio nuevo (ej. `control-de-puesto`).
2. Sube TODO el contenido de esta carpeta, incluida la carpeta oculta `.github`.
   - Si `.github` no se sube al arrastrar, usa Add file > Create new file, escribe el nombre `.github/workflows/build-apk.yml` y pega el contenido del archivo.
3. Ve a la pestaña **Actions**. La compilación inicia sola (3 a 6 minutos). Si no, elige "Compilar APK" > Run workflow.
4. Al terminar (check verde), abre esa ejecución y descarga **control-de-puesto-apk** en la sección Artifacts. Es un .zip con `app-debug.apk` dentro.
5. Pasa el APK al teléfono, ábrelo y acepta "Instalar apps de origen desconocido" para tu administrador de archivos o navegador.

## Si falla la compilación
Abre la ejecución con la X roja, entra al paso que falló y copia el mensaje de error.

## Datos
Se guardan en el almacenamiento interno de la app. Se pierden si desinstalas la app o borras sus datos desde Ajustes. La opción de copia de seguridad dentro de Inicio sigue disponible por seguridad.

## Limitaciones
- APK de tipo "debug": sirve para instalar y usar, pero no para publicar en Google Play (requiere firma de release).
- El ícono es el predeterminado de Capacitor; se puede personalizar con @capacitor/assets.
