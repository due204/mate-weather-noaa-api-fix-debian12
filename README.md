# Fix MATE Weather Applet (libmateweather) for Debian 12 (Bookworm)

Este repositorio documenta el proceso completo de ingeniería inversa, diagnóstico y parchado de código fuente en C para solucionar el fallo del applet del clima de MATE Desktop en Debian 12. El applet quedaba congelado de forma permanente mostrando nada debido a cambios críticos en la API externa de la NOAA (aviationweather.gov).

---

## 🔍 1. Fase de Investigación y Diagnóstico

### El rastro en las librerías dinámicas (.so)
El problema comenzó identificando qué componente del sistema se encargaba de las peticiones de red del clima. Usando herramientas de análisis de binarios sobre el applet del panel de MATE, rastreamos las cadenas de texto ocultas dentro de las librerías compartidas del sistema:

strings /usr/lib/x86_64-linux-gnu/libmateweather.so.1 | grep aviation

Este comando reveló que la librería compilada apuntaba de forma cableada (hardcoded) a un script obsoleto en el servidor de la NOAA:
http://weather.noaa.gov/pub/data/observations/metar/stations/%s.TXT

Al auditar la infraestructura de la NOAA, confirmamos que dicho servidor web y su estructura de directorios PHP fueron dados de baja por completo, migrando todo el servicio a una API REST moderna.

### Auditoría de la API en el Laboratorio (Prueba con cURL)
Para entender cómo responde la nueva API de la NOAA y verificar la conectividad de la máquina, simulamos la petición HTTP REST externa usando cURL apuntando al aeropuerto de Ezeiza, Buenos Aires (SAEZ):

```bash
curl -G "https://aviationweather.gov/api/data/dataserver" \
  --data-urlencode "requestType=retrieve" \
  --data-urlencode "dataSource=metars" \
  --data-urlencode "stationString=SAEZ" \
  --data-urlencode "hoursBeforeNow=2" \
  -o respuesta_noaa.xml
``` 

Al inspeccionar el archivo resultante (`cat respuesta_noaa.xml`), aislamos la estructura del nuevo estándar XML versión 2.0 enviado por el servidor:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<response version="2.0">
  <data num_results="1">
    <METAR>
      <raw_text>METAR SAEZ 282300Z 24002KT CAVOK 07/05 Q1018 NOSIG</raw_text>
      <station_id>SAEZ</station_id>
      <temp_c>7</temp_c>
      <dewpoint_c>5</dewpoint_c>
    </METAR>
  </data>
</response>
```
---

## 🛑 2. Análisis de los Fallos en el Código Fuente (.c)

Tras descargar las fuentes oficiales de `libmateweather-1.26.0`, inspeccionamos la lógica interna y descubrimos dos bugs de obsolescencia que impedían que el applet dibujara el clima:

### Bug A: Error de coincidencia de string en `weather-metar.c`
La función encargada de recortar el XML no parseaba el árbol de nodos, sino que buscaba texto plano de forma estricta asumiendo el estándar antiguo de la NOAA:
searchkey = g_strdup_printf ("<raw_text>%s", loc->code);

Como la nueva API inyecta la cabecera "METAR " antes del código del aeropuerto (`<raw_text>METAR SAEZ...`), la función `strstr` fallaba (retornaba NULL), rompiendo el flujo de lectura de las condiciones actuales.

### Bug B: Bloqueo por servidor muerto en `weather-iwin.c`
El módulo encargado del pronóstico extendido (IWIN) realizaba peticiones paralelas a un subdominio de la NOAA que fue desmantelado. Al fallar la conexión HTTP, la función callback retornaba un error de transporte (`info->network_error = TRUE`), bloqueando por completo la actualización de la barra de tareas.

---

## 🛠️ 3. Solución Avanzada: Parches Quirúrgicos en C

Para solucionar ambos fallos sin romper la compatibilidad con Debian Bookworm, aplicamos las siguientes modificaciones directamente sobre el código fuente:

### Modificación en `libmateweather/weather-metar.c`
Actualizamos las variables de búsqueda de strings para adaptarlas al XML 2.0 y recalculamos el desplazamiento del puntero de memoria (offset de 11 a 17 bytes) para saltar limpiamente la etiqueta combinada:


```c
static void
metar_finish (SoupSession *session, SoupMessage *msg, gpointer data)
{
    WeatherInfo *info = (WeatherInfo *)data;
    WeatherLocation *loc;
    const gchar *p, *endtag;
    gchar *searchkey, *metar;
    gboolean success = FALSE;

    g_return_if_fail (info != NULL);

    if (!SOUP_STATUS_IS_SUCCESSFUL (msg->status_code)) {
        if (SOUP_STATUS_IS_TRANSPORT_ERROR (msg->status_code))
            info->network_error = TRUE;
        else {
            /* Translators: %d is an error code, and %s the error string */
            g_warning (_("Failed to get METAR data: %d %s.\n"),
                        msg->status_code, msg->reason_phrase);
        }
        request_done (info, FALSE);
        return;
    }

    loc = info->location;

    /* PARCHE XML 2.0: Adaptado para buscar la nueva estructura de la NOAA con "METAR " incluido */
    searchkey = g_strdup_printf ("<raw_text>METAR %s", loc->code);
    p = strstr (msg->response_body->data, searchkey);
    g_free (searchkey);
    if (p) {
        /* Desplazamiento ajustado a 17 para saltar limpiamente "<raw_text>METAR " */
        p += WEATHER_LOCATION_CODE_LEN + 17;
        endtag = strstr (p, "</raw_text>");
        if (endtag)
            metar = g_strndup (p, endtag - p);
        else
            metar = g_strdup (p);
        success = metar_parse (metar, info);
        g_free (metar);
    } else if (!strstr (msg->response_body->data, "aviationweather.gov")) {
        /* The response doesn't even seem to have come from NOAA...
         * most likely it is a wifi hotspot login page. Call that a
         * network error.
         */
        info->network_error = TRUE;
    }

    info->valid = success;
    request_done (info, TRUE);
}
```


### Modificación en `libmateweather/weather-iwin.c` (Bypass de Pronóstico)
Interceptamos la función de respuesta `iwin_finish`. Al no existir un reemplazo directo para el backend de IWIN, inyectamos un bypass de retorno temprano que notifica un éxito simulado (`TRUE`). Esto aísla el código obsoleto inferior de manera segura sin corromper la jerarquía de llaves sintácticas exigidas por el compilador GCC:

```c 
static void
iwin_finish (SoupSession *session, SoupMessage *msg, gpointer data)
{
    WeatherInfo *info = (WeatherInfo *)data;

    g_return_if_fail (info != NULL);

    /* BYPASS DEFINITIVO: Forzamos el éxito local para evitar el cuelgue del panel por servidor muerto */
    request_done (info, TRUE);
    return;

    /* Todo el bloque inferior queda aislado como código muerto, preservando la sintaxis del compilador */
    if (!SOUP_STATUS_IS_SUCCESSFUL (msg->status_code)) {
        /* forecast data is not really interesting anyway ;) */
        g_warning ("Failed to get IWIN forecast data: %d %s\n",
                   msg->status_code, msg->reason_phrase);
        request_done (info, FALSE);
        return;
    }

    if (info->forecast_type == FORECAST_LIST)
        info->forecast_list = parseForecastXml (msg->response_body->data, info);
    else
        info->forecast = formatWeatherMsg (g_strdup (msg->response_body->data));

    request_done (info, TRUE);
}
``` 

## 🚀 4. Guía de Compilación, Empaquetado e Instalación

Para replicar esta solución desde cero en cualquier instalación de Debian 12, ejecuta los siguientes comandos ordenadamente en la terminal:

## 1. Instalar las dependencias del sistema y de construcción de Debian 12

```bash
sudo apt update
sudo apt install build-essential devscripts dh-make quilt libsoup2.4-dev libgtk-3-dev libxml2-dev
``` 

## 2. Crear un directorio de trabajo seguro y descargar las fuentes oficiales

```bash
mkdir ~/paso-mateweather && cd ~/paso-mateweather
apt-get source libmateweather
cd libmateweather-1.26.0/
```

## 3. Aplicar las modificaciones explicadas arriba en los archivos correspondientes

Afecta directamente a:
* `libmateweather/weather-metar.c`
* `libmateweather/weather-iwin.c`


## 4. Limpiar el entorno de compilación

```bash
dh_clean
``` 

## 5. Compilar el código fuente y empaquetar en binarios nativos .deb (.no-sign evita errores GPG locales)

```bash
dpkg-buildpackage -b -rfakeroot -us -uc --no-sign
```

## 6. Instalar los paquetes generados en el directorio superior

```bash
cd ..
sudo dpkg -i libmateweather1_*.deb libmateweather-common_*.deb
```

## 7. Forzar el reinicio del entorno del panel de MATE para actualizar las librerías dinámicas en memoria

```bash
pkill -f mateweather
mate-panel --replace &
``` 

---

## 📈 Resultado Final
Tras aplicar las modificaciones en la memoria activa del panel, la función de parsing por expresiones regulares nativas de `metar_parse` vuelve a recibir la cadena cruda de aviación de forma limpia. El applet procesa exitosamente los datos de temperatura, viento y presión barométrica directamente desde el nuevo formato XML 2.0 de la NOAA, devolviendo a la vida el indicador meteorológico del escritorio.
