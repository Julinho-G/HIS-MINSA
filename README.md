# HIS MINSA

Plataforma de consulta y análisis HIS MINSA para explorar pacientes, producción asistencial y catálogos institucionales almacenados en SQL Server. La interfaz prioriza la consulta y no incluye captura de nuevas atenciones.

## Descripción general

El proyecto está organizado en dos áreas de responsabilidad:

- **Frontend:** presenta el constructor de consultas, resultados, fichas de consulta, reportes y catálogos; consume la API HTTP y no se conecta directamente a la base de datos.
- **Backend:** expone la API, aplica las reglas de negocio y ejecuta consultas parametrizadas contra SQL Server.

En el modo local, Express sirve también los archivos estáticos del frontend. Esta disposición conserva un único origen para la aplicación y permite separar la publicación del frontend y la API cuando el entorno lo requiera.

## Arquitectura y estructura

```text
backend/
  app.js                   Configuración de Express y montaje de rutas
  server.js                Carga del entorno y arranque del servidor
  config/db.js             Conexión y acceso a SQL Server
  controllers/             Consultas y reglas por dominio
  routes/                  Definición de endpoints HTTP
  check_schema.js          Inspección de tablas y columnas
frontend/
  index.html               Estructura semántica de la interfaz
  css/app.css              Estilos y reglas adaptables
  js/app.js                Interacciones y consumo de la API
.env                       Configuración local, no versionada
package.json               Dependencias y comandos del proyecto
```

### Responsabilidades del backend

- `app.js` registra middleware, rutas y la carpeta estática del frontend.
- `server.js` carga `.env` desde la raíz y abre el puerto definido por `PORT` (3000 por defecto).
- `routes/` asocia cada URL con la función de controlador que la atiende.
- `controllers/` agrupa el acceso a datos y la transformación de resultados por dominio: pacientes, maestros y atenciones.
- `config/db.js` centraliza el pool de SQL Server y limita los reintentos a fallos transitorios de lectura.
- `check_schema.js` permite consultar el esquema de la base de datos conectada.

### Responsabilidades del frontend

- `index.html` contiene la estructura de las vistas y sus controles.
- `css/app.css` define variables de estilo, componentes y comportamiento adaptable a pantallas pequeñas.
- `js/app.js` inicializa la página y coordina navegación, filtros, resultados, tablas y solicitudes HTTP.

## Funcionalidades

- Construir consultas visuales por medida, campos HIS, operadores y periodos. Cada grupo combina sus criterios con Y/O y cada conexión entre grupos tiene su propio Y/O.
- Agrupar los resultados por establecimiento, departamento, provincia, profesional, periodo, código HIS, diagnóstico, tipo de diagnóstico, financiador, sexo, UPS o datos identificatorios del paciente.
- Consultar pacientes, producción por profesional, indicadores de producción y reportes agregados.
- Consultar todo el padrón `MaestroPaciente` con paginación de servidor, buscar por documento, nombre, apellido o historia clínica y abrir la ficha de producción cuando existan registros HIS.
- Consultar resumen, atenciones con filtros de fecha y texto, diagnósticos frecuentes y detalles de cada atención.
- Revisar producción por profesional, incluyendo pacientes atendidos, códigos frecuentes y actividad por mes.
- Generar reportes de producción por periodo, establecimiento, código y profesional.
- Explorar catálogos de establecimientos, UPS, CIE-10, CPMS y laboratorio. Los establecimientos, servicios y códigos HIS pueden enviarse al constructor como filtros de producción.
- Guardar consultas frecuentes en el navegador y exportar resultados a CSV o JSON.

Las consultas y reportes utilizan `NProduccion` y se enriquecen con `MaestroPaciente`, `MaestroPersonal` y las tablas maestras relacionadas. Las medidas del explorador tienen semánticas diferenciadas: **pacientes únicos** cuenta `Id_Paciente` distinto; **citas atendidas** cuenta `Id_Cita` distinto; **registros HIS** cuenta filas de `NProduccion`. También se pueden contar códigos HIS distintos y calcular sumas/promedios LAB, edad, peso, talla y hemoglobina cuando existan valores. La consulta devuelve las tres métricas base junto con la medida seleccionada y calcula el total general directamente sobre los filtros; los grupos del resultado pueden compartir pacientes si se agrupa por dimensiones distintas. En el constructor, Y exige que se cumplan todos los criterios del grupo y O permite que se cumpla cualquiera; además, cada conexión entre grupos se configura individualmente. Si se mezclan conectores, Y tiene precedencia sobre O, como en SQL. El periodo de atención siempre se conserva como filtro general. El KPI de pacientes registrados y el buscador del padrón cuentan `MaestroPaciente` completo, incluyendo personas sin filas en `NProduccion`. La edad se toma de `Edad_Reg` únicamente cuando `Tipo_Edad = 'A'` (años registrados en la atención), no se recalcula con la fecha actual. Los agregados se ejecutan en SQL Server; los valores LAB no numéricos y la categoría `10+` se gestionan de forma segura para evitar errores de conversión. Al sumar o promediar LAB, `10+` se computa como `10` (límite inferior de la categoría, no el valor real), y la interfaz lo identifica para evitar presentar ese resultado como exacto. La API de esta aplicación no expone operaciones para registrar o modificar pacientes ni atenciones.

Los archivos fuente incluyen comentarios por módulo, vista, función y bloque funcional para explicar responsabilidades y reglas de negocio. Los archivos JSON de configuración se mantienen sin comentarios porque el formato JSON no los admite.

## Requisitos

- Node.js 22 o superior.
- Acceso de red a una instancia de SQL Server compatible con la configuración de `mssql`.
- Dependencias del proyecto instaladas con npm.

## Configuración del entorno

Crea un archivo `.env` en la raíz del repositorio con las variables necesarias:

```dotenv
DB_USER=
DB_PASSWORD=
DB_SERVER=
DB_DATABASE=
DB_PORT=1433
PORT=3000
```

Completa los valores de acuerdo con la instancia local. No incluyas credenciales reales en archivos versionados, capturas o registros. `.env` está excluido mediante `.gitignore`; el backend carga este archivo desde una ruta explícita, independiente del directorio desde el que se inicie.

## Instalación y ejecución

Desde la raíz del proyecto, instala las dependencias y arranca el servidor:

```powershell
npm install
npm start
```

Abre `http://localhost:3000` en el navegador. Para desarrollo con reinicio automático al detectar cambios:

```powershell
npm run dev
```

En Windows, si la política de PowerShell bloquea `npm.ps1`, usa el ejecutable `.cmd`:

```powershell
npm.cmd install
npm.cmd start
npm.cmd run dev
```

El comando `npm start` ejecuta `backend/server.js`; `npm run dev` utiliza el modo de observación de Node.js. Los archivos `server.js` y `check_schema.js` de la raíz se mantienen como puntos de entrada de compatibilidad para comandos anteriores.

## API HTTP

La API usa el prefijo `/api`. Endpoints de uso frecuente:

| Método | Endpoint | Descripción |
| --- | --- | --- |
| `GET` | `/api/health` | Comprueba que el servidor HTTP está activo. |
| `GET` | `/api/pacientes?page=1&pageSize=50&q=...` | Devuelve una página del padrón completo `MaestroPaciente` y el total coincidente. `q` es opcional; sin él se puede recorrer todo el padrón. |
| `GET` | `/api/pacientes?tipoDocumento=...&documento=...` | Busca un paciente por tipo y número de documento. |
| `GET` | `/api/pacientes/:id/panel` | Devuelve resumen, historial y diagnósticos del paciente. Acepta ID o documento. |
| `GET` | `/api/maestros/personal?q=...` | Busca profesionales en el catálogo de personal. |
| `GET` | `/api/maestros/personal/:id/actividad` | Devuelve actividad HIS del profesional; acepta ID o documento. |
| `GET` | `/api/atenciones/reportes` | Genera métricas de filas HIS, citas y pacientes distintos, con desgloses por mes, código, establecimiento y profesional. Acepta `fechaInicio` y `fechaFin` en formato `AAAA-MM-DD`; fechas inválidas o un rango invertido devuelven HTTP 400. |
| `GET` | `/api/consultas/metadata` | Devuelve los campos, medidas y agrupaciones permitidas en el constructor. |
| `GET` | `/api/consultas/dashboard` | Devuelve los indicadores globales y tendencias del tablero. |
| `POST` | `/api/consultas/ejecutar` | Valida y ejecuta una consulta analítica parametrizada. |

Los filtros de fecha utilizan `fechaInicio` y `fechaFin` en formato `AAAA-MM-DD`; la fecha final incluye el día completo, también en los reportes por paciente y profesional. En producción, `Registros HIS` cuenta filas por ítem, `Citas` cuenta `Id_Cita` distintos y `Pacientes` cuenta `Id_Paciente` distintos; estos totales no son intercambiables y los pacientes pueden aparecer en más de una fila agrupada. La API HIS no incluye rutas para insertar atenciones o pacientes. El listado completo de rutas de cada dominio está en `backend/routes/`.

## Datos y consultas

- `MaestroPaciente` aporta identificación y datos demográficos del paciente.
- `NProduccion` almacena los registros HIS por ítem, el identificador de cita (`Id_Cita`), los códigos, resultados LAB, edad registrada y relaciones con personal, establecimiento y UPS.
- El padrón de pacientes y la producción son universos distintos: la búsqueda lista el maestro completo sin exigir que el paciente tenga filas en `NProduccion`; el historial clínico solo puede mostrar producción que esté registrada.
- `MaestroPersonal` identifica al profesional que produjo cada registro.
- Las tablas `MAESTRO_HIS_*` aportan descripciones de códigos, servicios, establecimientos, profesiones y otros catálogos.

Los campos, medidas, agrupaciones, operadores y conectores lógicos del constructor se restringen a listas permitidas. Cada grupo tiene su propio conector Y/O y cada conexión entre grupos también tiene un operador independiente; cuando se mezclan operadores, Y tiene precedencia sobre O. El periodo se aplica globalmente y no puede quedar excluido por una condición O. Las consultas guardadas se almacenan localmente con el nombre elegido por el usuario y conservan sus grupos y conectores. Los valores se envían como parámetros SQL; el cliente no envía expresiones SQL ni recibe las expresiones internas del motor. Para proteger SQL Server y el navegador, una consulta admite hasta 100 criterios, 20 grupos, 500 valores por lista y devuelve como máximo 5,000 filas agrupadas; cuando el resultado supera ese límite, la interfaz lo indica para que se afinen los filtros.

## Verificación y resolución de problemas

1. Abre `http://localhost:3000/api/health`. Una respuesta HTTP 200 confirma que Express está activo.
2. Si la interfaz informa que no puede consultar la base, comprueba que SQL Server esté disponible y revisa `DB_SERVER`, `DB_DATABASE`, el puerto y las credenciales locales.
3. Para revisar los nombres de tablas y columnas visibles en la conexión configurada, ejecuta `node backend/check_schema.js` (también está disponible `node check_schema.js` desde la raíz).
4. Los errores de conexión transitorios en lecturas se reintentan de forma limitada. Si el servicio de base de datos sigue sin estar disponible, la API devuelve un error recuperable y registra el detalle en la consola del servidor.
5. Si el frontend y la API se alojan en direcciones diferentes, actualiza la URL base en la vista **Configuración** y restringe CORS a los orígenes que correspondan.

## Seguridad y operación

La información HIS puede contener datos personales y clínicos. Limita el acceso a la aplicación, protege las credenciales de SQL Server y evita compartir datos reales en reportes de depuración. Antes de publicar el servicio fuera de un entorno local controlado, configura autenticación y autorización, restringe CORS, habilita HTTPS y define controles de copias de seguridad, auditoría y conservación de datos.
