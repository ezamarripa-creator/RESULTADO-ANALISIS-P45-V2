# Sistema de Evaluación de Planeaciones Didácticas — COBAED 2026B

Dashboard ejecutivo estático (HTML/JS) para consolidar y monitorear el registro de
planeaciones didácticas de los 35 planteles del Colegio de Bachilleres del Estado de
Durango, con base en el archivo `RESULTADO_DEL_P45_VS_RIA.xlsx`.

## Descripción

Una sola página (`index.html`) que:

- Trae los 929 registros del archivo original embebidos como snapshot (`DATASET`).
- Calcula en el navegador 25 indicadores institucionales (KPIs), gráficos, un mapa
  de calor, un treemap y una tabla inteligente con exportación.
- Ofrece un botón **Sincronizar** que intenta traer datos frescos desde un Google
  Sheet publicado como CSV (ver sección *Sincronización*), y si no está disponible,
  conserva el snapshot local sin romper la vista.
- Incluye modo claro/oscuro, sidebar colapsable, filtros dependientes, loader y
  toasts, siguiendo la paleta institucional (azul plumbago, marrón tierra, beige,
  crema, azules y verdes suaves).

## Arquitectura

Es una aplicación 100% front-end, sin backend propio:

```
index.html
 ├─ <style>   → tokens de color / tema claro-oscuro / layout
 ├─ DATASET   → snapshot JSON embebido (929 registros)
 └─ <script>  → capa de datos, KPIs, gráficos y tabla
     ├─ enrichRecord / validateAndNormalize   (normalización y cálculo derivado)
     ├─ syncFromGoogleSheets / parseCSV        (sincronización opcional)
     ├─ computeKPIs / KPI_DEFS                 (25 indicadores)
     ├─ renderCharts / renderHeatmap / renderTreemap  (Chart.js + nativos)
     ├─ renderTable (DataTables)
     └─ init()                                  (arranque, smoke tests)
```

No hay "MVC" con backend real porque no hay servidor: los datos viven en el
snapshot y, opcionalmente, en un Google Sheet publicado. Si más adelante quieres
un backend de verdad (control de acceso por dominio `@cobaed.mx`, escritura de
datos, etc.), la opción recomendada es un Web App de Google Apps Script — ver
la sección *Extender con Apps Script* más abajo.

## Tecnologías

- HTML5 / CSS3 (variables CSS para tema claro/oscuro) / JavaScript (ES2023)
- [Chart.js 4](https://www.chartjs.org/) — gauge, donut, barras, radar, bubble
- [DataTables](https://datatables.net/) + Buttons (Excel, CSV, PDF, imprimir, copiar)
- jQuery (requerido por DataTables)
- Sin build step: todo corre directo en el navegador, vía CDN (cdnjs)

## Instalación / Despliegue en GitHub Pages

1. Crea un repositorio nuevo en GitHub (por ejemplo `cobaed-planeaciones-2026b`).
2. Sube `index.html` (y este `README.md`) a la raíz del repositorio.
3. Ve a **Settings → Pages**, selecciona la rama `main` y la carpeta `/root`.
4. En un par de minutos tu dashboard queda disponible en
   `https://<tu-usuario>.github.io/<repo>/`.

También puedes abrir `index.html` localmente con doble clic, o servirlo desde
cualquier hosting estático (Netlify, Vercel, Google Cloud Storage, etc.).

## Sincronización con Google Sheets

El botón **Sincronizar** llama a `syncFromGoogleSheets()`, que intenta leer un
CSV publicado desde el Google Sheet de origen. Para activarlo con datos en vivo:

1. Abre el Google Sheet vinculado (mismo `Numero de plantel`, `plantel`,
   `matricula`, `docente`, `Turno`, columnas de parciales, etc. que el archivo
   original).
2. Ve a **Archivo → Compartir → Publicar en la web**, elige la hoja correcta y
   el formato **CSV**, y publica.
3. Copia el enlace generado (tiene la forma
   `https://docs.google.com/spreadsheets/d/<ID>/gviz/tq?tqx=out:csv&gid=<GID>`).
4. En `index.html`, busca la constante `GOOGLE_SHEET_CSV_URL` (dentro de la
   sección `0. CONFIGURACIÓN DE SINCRONIZACIÓN` del `<script>`) y pégalo ahí.

Si el Sheet no está publicado, o el navegador bloquea la petición por CORS, el
dashboard muestra un aviso y sigue funcionando con el snapshot local — nunca se
queda sin datos.

**Nota importante:** publicar una hoja "en la web" la hace accesible a cualquiera
con el enlace. Si el archivo contiene datos sensibles de docentes, valora
publicar solo las columnas necesarias en una hoja separada, o migrar a un Web
App de Apps Script con control de acceso (ver abajo).

## Extender con Apps Script (opcional, control de acceso real)

Si necesitas validar el dominio `@cobaed.mx` y restringir el acceso, la ruta
recomendada es publicar un **Web App de Google Apps Script** que:

1. Lea el Google Sheet con `SpreadsheetApp.openById(ID)`.
2. Verifique `Session.getActiveUser().getEmail()` contra el dominio institucional
   antes de responder.
3. Exponga un `doGet(e)` que devuelva el mismo JSON que espera `DATASET` en este
   `index.html` (mismas llaves: `num_plantel`, `plantel`, `matricula`, `docente`,
   `turno`, `registradas`, `p1`, `p2`, `p3`, `materias`, `esperadas`,
   `diferencia`, `dictamen`, `tipo_docente`).
4. En el dashboard, cambia `syncFromGoogleSheets` para hacer `fetch` a la URL
   `/exec` de ese Web App en vez del CSV público.

Esto requiere una cuenta de Google Workspace con permisos de despliegue, así que
no está incluido como código listo para pegar aquí — pero la estructura de datos
ya está preparada para recibirlo sin cambios adicionales en el front-end.

## Guía de indicadores

Los 25 KPIs se calculan en `computeKPIs()` y se documentan uno a uno en el
arreglo `KPI_DEFS` (mismo archivo), donde cada tarjeta incluye su fórmula,
tooltip, semáforo (verde ≥90%, amarillo ≥70%, rojo <70%) y color institucional.
Los más relevantes:

| Indicador | Fórmula |
|---|---|
| Índice General de Cumplimiento (IGCP) | Registradas ÷ Esperadas × 100 |
| % Docentes Cumplidos | Docentes con dictamen "Accede" ÷ total × 100 |
| Índice Gestión Académica | 50% cumplimiento general + 30% % docentes cumplidos + 20% cumplimiento titulares |
| Índice Cobertura Materias | Registradas ÷ (Materias asignadas × 3) × 100 |

## Guía de administración

- **Actualizar el snapshot embebido:** vuelve a exportar el `.xlsx` a JSON con
  las mismas llaves de columna y reemplaza el arreglo `DATASET` en `index.html`.
- **Cambiar la paleta:** todos los colores institucionales están declarados como
  variables CSS (`--plumbago`, `--marron`, `--beige`, etc.) al inicio del
  `<style>`; cambiarlas ahí actualiza todo el dashboard.
- **Agregar un KPI:** añade una entrada a `KPI_DEFS` y, si necesita un cálculo
  nuevo, una línea en `computeKPIs()`.
- **Agregar un gráfico:** sigue el patrón de `renderCharts()` (una función
  `destroyChart` + `new Chart(...)` por visualización) para que se reconstruya
  correctamente al cambiar de tema o al filtrar.

## Estructura del proyecto

```
/
├─ index.html   → dashboard completo (única página, autocontenida)
└─ README.md    → este documento
```

## Aviso de confidencialidad

El pie de página del dashboard reproduce el aviso de confidencialidad
institucional solicitado. Si publicas este repositorio en un GitHub **público**,
recuerda que los datos de docentes (nombre, matrícula, plantel) quedarán
visibles para cualquiera con el enlace — considera un repositorio **privado** o
la variante de Apps Script con control de acceso si el archivo contiene
información que deba tratarse como reservada.
