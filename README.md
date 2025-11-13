# azureproject112025

🧾 INFORME TÉCNICO

Proyecto: Spotify End-To-End Azure Data Engineering
Entorno: Azure Cloud

1️⃣ OBJETIVO DEL PROYECTO

1 Implementar una arquitectura integral de ingeniería de datos en Azure, aplicando buenas prácticas de entornos productivos:
2 Flujo completo Ingesta → Procesamiento → Curación (Bronze → Silver → Gold).
3 Integración modular de servicios Azure: Data Factory, Data Lake, Databricks y SQL Database.
4 Implementación de cargas incrementales y metadatos dinámicos.
5 Control de versiones mediante GitHub y CI/CD.
6 Validación continua y monitoreo con logging detallado.

Recursos	Data Factory, Data Lake, SQL DB, Databricks	Servicios principales.
Seguridad: autenticación SQL + Entra ID y firewall con IP local.

##  DATOS FUENTE

Tablas creadas desde script SQL en GitHub:
DimArtist, DimUser, DimDate, FactStream, etc.

##  DESARROLLO DEL PIPELINE INCREMENTAL (BRONZE LAYER)
🔸 Pipeline principal: Incremental_loop

Objetivo: mover datos desde SQL → Data Lake con control de cambios basado en columna CDC.

# Componentes:

🔎 Parámetro loop_input → array JSON con definición dinámica de cada tabla.

🔁 ForEach1 (secuencial) → itera sobre cada tabla, garantizando ejecución ordenada.

🔎 Lookup last_cdc → obtiene la fecha del último CDC registrada en cdc.json.

🖨 Copy azureSQLtoLake → extrae solo filas nuevas.

✅ IfCondition Ifincrimental_data → evalúa si hubo datos nuevos.

⏲ Script max_cdc → calcula la fecha máxima del nuevo lote.

🖨 Copy update_last_cdc → actualiza cdc.json con la nueva fecha.

🗑 Delete Delete_empty_file → elimina archivos vacíos si no hay nuevos registros.

### 🧠 INCIDENTES Y SOLUCIONES TÉCNICAS APLICADAS

Durante la implementación, se detectaron diversos errores de configuración, los cuales se documentan a continuación:

# Nº Descripción del error	Causa raíz	Solución aplicada

❗	Error array index '0' cannot be selected from empty array	El archivo cdc.json no existía o estaba vacío, provocando que activity('last_cdc').output.value devolviera un array vacío.	Se creó un valor por defecto en caso de CDC vacío (1900-01-01) y se añadió validación con @if(or(equals(...))).

❗		BadRequest en IfCondition	Métrica usada: dataRead (a veces null).	Se reemplazó por: @greater(int(coalesce(activity('azureSQLtoLake').output.rowsCopied, 0)), 0) para robustez.

❗		Concurrency alta en ForEach provocaba colisiones	Ejecución simultánea de tablas.	Se habilitó modo Secuencial (Concurrency = 1) durante depuración.

🧩 EXPRESIONES CLAVE FINALMENTE ADOPTADAS

## Carga incremental (Copy – azureSQLtoLake):

select *
from @{item().schema}.@{item().table}
where @{item().cdc_col} > '@{variables('cdc_cutoff')}'


## Condición IF:

@greater(int(coalesce(activity('azureSQLtoLake').output.rowsCopied, 0)), 0)


## Ruta dinámica en datasets:

@concat(item().table, '_cdc')


Actualización de CDC (update_last_cdc):

{
  "name": "cdc",
  "value": {
    "value": "@{activity('max_cdc').output.resultSets[0].rows[0].cdc}",
    "type": "Expression"
  }
}

✅ RESULTADO FINAL ESPERADO

Tras un run , cada carpeta de tabla en el contenedor bronze mantiene:

bronze/
 ├── DimUser_cdc/
 │   ├── empty.json
 │   └── cdc.json  ← {"cdc": "2025-11-12T14:38:21Z"}
 ├── DimTrack_cdc/
 │   ├── empty.json
 │   └── cdc.json
 ...


En la siguiente ejecución, el Lookup leerá esa fecha(actualizada), permitiendo una ingesta incremental totalmente automatizada.
