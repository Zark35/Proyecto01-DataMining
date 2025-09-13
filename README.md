# Proyecto 01 Data Mining 
# Estudiante: Andrés Bohórquez | Código: 00320727

# Descripción y diagrama de arquitectura
El proyecto se enfoca en implementar un pipeline que extráe datos desde la API QuickBoosks. El pipeline busca una ingesta histórica (backfill) para Invoices, Customers e Items.
Los datos son segmentados y posteriormente cargados en la capa **raw** de Postgres, para completa trazabilidad.

La arquitectura del proyecto es la siguiente:

```text
QuickBooks API
     │
     ▼
[ Data Loader ]
  - Refresh Token
  - Extract + Chunking
  - Save request metadata
     │
     ▼
[ Data Exporter ]
  - Upsert en Postgres (raw.qb_<entidad>)
  - Guarda payload + metadatos
     │
     ▼
Postgres (schema raw)
```
 

# Pasos para levantar contenedores y configurar el proyecto
## Levantar contenedor de Docker
La primera parte del proyecto es el levantamiento en Docker para tener Mage, PgAdmin, y Postgres.
Los tres servicios tendrán un puerto asignado para posteriormente poder trabajar con cada una, como en Mage para el código y la funcionalidad de los pipelines, o realizar validaciones en la base de datos con pgadmin.
Para levantar el contenedor, primero se debe descargar el proyecto desde gihthub con el comando:
git clone https://github.com/Zark35/Proyecto01-DataMining.git

Dentro del proyecto, ir a la carpeta de Proyecto completo original, en la carpeta Proyecto 1, y ahí abrir una consola de comando. Este proceso también se puede hacer en la consola para llegar hasta la carpeta Proyecto 1. En la consola, se debe ejecutar el siguiente comando para ejecutar el archivo docker-compose.yml:

```bash
docker compose up -d
```

Una vez finalizado, se puede acceder a los diferentes servicios.

Servicios (con su respectivo puerto):
Postgres: localhost:5433
pgAdmin: http://localhost:8070
Mage: http://localhost:6789

Un detalle adicional es que al ingresar en Postgres, el usuario y la contraseña para entrar son las siguientes:
- Usuario: admin@datamining.com
- Contraseña: admin


# Gestión de secretos (nombres, propósito, rotación, responsables)
Los valores secretos están guardados en el pipeline con ayuda de Mage UI. Sin embargo, en la arquitectura del proyecto está un archivo de secrets.yml donde se muestra el nombre de los secretos junto a unos valores de referencia.

El propósito y nombre de los secretos son los siguientes:
| Nombre           | Propósito                          |
|------------------|------------------------------------|
| qb_client_id     | Identificador de app en QuickBooks |
| qb_client_secret | Clave secreta de app en QuickBooks |
| qb_refresh_token | Token de actualización OAuth 2.0   |
| qb_realm_id      | ID de la compañía en QuickBooks    |
| pg_host          | Host de Postgres                   |
| pg_user          | Usuario de Postgres                |
| pg_password      | Contraseña de Postgres             |
| pg_port          | Puerto de Postgres                 |
| pg_db            | Base de datos destino              |

Los cuatro primeros secretos corresponden a los necesarios para trabajar con la Api. Asimismo, el Refresh Token es fundamental para el proceso de OAuth 2.0 y obtener el Acces Token al inicio de cada ejecución/tramo. A pesar de que el Refresh Token recide en los Secrets, al momento de emplearlo se debe actualizar con los valores que se devuelve al obtener el access token. Por esta razón, el código también refresca la variable de Refresh Token para evitar errores de ejecución.
Los últimos cinco secretos son para las credenciales de Postgres.
De esta forma, todos los tokens/lleaves de QBO y las credencias de Postgres están seguras.


# Detalle de los tres pipelines qb_<entidad>_backfill
Antes de mencionar los detalles y como se mencionó previamente, para reducir la cantidad de pipelines y trabajar con un único pipeline, se agregó en los parámetros una variable entidad para trabajar con Invoice (factura), Customer (cliente), o Item (artículo)-
No olvidar que antes de ejecutar los triggers es necesario modificar los valores de las Runtime variables para ingresar la nueva entidad y las fechas.

- **Parámetros:**
  - `entidad` (`Invoice`, `Customer`, `Item`)
  - `fecha_inicio` y `fecha_fin` (ISO 8601, UTC)
  
- **Segmentación(chunks):**
  - Invoices → diaria
  - Customers / Items → única carga
  
- **Límites y reintentos:**
  - Paginación con `STARTPOSITION` y `MAXRESULTS` en el query para los parámetros del query
  - Reintentos con backoff exponencial (2 elevado al número de intentos) hasta 5 veces

- **Runbook:**
  - Al ingresar las fechas fallida, el pipeline solo hará el chunking para ese día
  
  - Consultar resultados en Postgres. Para ello se pueden ejecutar dos Querys diferentes para confirmar. 
    - Por un lado el query para ver los días procesados es el siguiente:
```sql
		SELECT DISTINCT DATE(extract_window_start_utc) AS fecha, COUNT(*) AS registros
		FROM qb_invoices
		GROUP BY fecha
		ORDER BY fecha;
```
		
    - Por otro lado, el query para detectar los días con 0 registros es el siguiente:
```sql
		SELECT g.fecha
		FROM generate_series('2025-08-01'::date, '2025-08-07'::date, '1 day') g(fecha)
		LEFT JOIN (
		    SELECT DISTINCT DATE(extract_window_start_utc) AS fecha
		    FROM qb_invoices
		) d ON g.fecha = d.fecha
		WHERE d.fecha IS NULL;
```
  
  - Validar ausencia de duplicados con `id`. Además, gracias al ON CONFLICT (id) DO UPDATE, el reintento no duplicará datos, los actualizará. En este caso también se puede verificar que no existan ids duplicados en Postgres con el query:
```sql
  SELECT id, COUNT(*)
	FROM qb_invoices
	GROUP BY id
	HAVING COUNT(*) > 1;
```
	

# Trigger one-time
En Mage se configuró un trigger para ejecutarse una vez (once) tomando en consideración dos fechas/horas en UTC. Por ejemplo:
- **Fecha/hora UTC:** 2025-09-15 18:00:00
- **Equivalencia Guayaquil:** 2025-09-15 13:00:00

**Política:** Tras ejecución exitosa, el trigger se deshabilita para evitar relanzamientos accidentales.


# Esquema raw
El esquema raw se encuentrá en la base de datos "datamining"

Tablas: `qb_invoices`, `qb_customers`, `qb_items`

Metadatos obligatorios:
- `id` (PRIMARY KEY)
- `payload` (JSONB con respuesta cruda)
- `ingested_at_utc`
- `extract_window_start_utc`, `extract_window_end_utc`
- `page_number`, `page_size`
- `request_payload`

**Idempotencia:** Este apartado se cumple con el query de SQL mencionado previamente de `ON CONFLICT DO UPDATE`.


# Validaciones y volumetría
## Validaciones
- Algunas de las validaciones que se pueden realizar fueron mencionadas en el apartado de Runbook en la sección del Pipeline.

```sql
-- Conteo total para el volumen total de registros cargados
SELECT COUNT(*) FROM qb_invoices;

-- Fechas cubiertas, para validar que la cobertura temporal del pipeline corresponde al rango de las fechas ingresadas
SELECT MIN(extract_window_start_utc), MAX(extract_window_end_utc)
FROM qb_invoices;

-- Validar ausencia de duplicados en base a la clave primaria
SELECT id, COUNT(*) FROM qb_invoices
GROUP BY id HAVING COUNT(*) > 1;

-- Detectar días del rango de interés sin registros cargados
SELECT g.fecha
FROM generate_series('2025-08-01'::date, '2025-08-07'::date, '1 day') g(fecha)
LEFT JOIN (
    SELECT DISTINCT DATE(extract_window_start_utc) AS fecha
    FROM qb_invoices
) d ON g.fecha = d.fecha
WHERE d.fecha IS NULL;
```
- Con las validaciones se puede comprobar que la volumetría sea la correcta para el rango solicitado de fechas.
- Como una prueba de las validaciones, se tomaron y guardaron screenshots de las ejecuciones de los querys en la carpeta Pruebas de las Validaciones.


# Troubleshooting
- **Auth fallida (401/403):** Refrescar `qb_refresh_token` en Secrets.
- **DNS errors:** Forzar DNS en `docker-compose.yaml` (`8.8.8.8`).
- **Rate limits:** Revisar logs de reintentos con backoff.
- **Timezones:** Siempre usar UTC en parámetros; equivalencias documentadas en README.
- **Postgres sin tablas:** Query adicional para la creación del esquema raw. Además, revisar bloque data_exporter se ejecute tras data_loader.


# Checklist de implementación

- [x] Mage y Postgres se comunican por nombre de servicio.  
- [x] Todos los secretos (QBO y Postgres) están en Mage Secrets; no hay secretos en el repo/entorno expuesto.  
- [x] Pipelines `qb_<entidad>_backfill` aceptan `fecha_inicio` y `fecha_fin` (UTC) y segmentan el rango.  
- [x] Trigger one-time configurado, ejecutado y luego deshabilitado/marcado como completado.  
- [x] Esquema `raw` con tablas por entidad, payload completo y metadatos obligatorios.  
- [x] Idempotencia verificada: reejecución de un tramo no genera duplicados.  
- [x] Paginación y rate limits manejados y documentados.  
- [x] Volumetría y validaciones mínimas registradas y archivadas como evidencia.  
- [x] Runbook de reanudación y reintentos disponible y seguido.  
