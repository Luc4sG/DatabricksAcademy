# ClearTech Final Project – Iowa Liquor Sales en Databricks

## Descripción
Este proyecto implementa un flujo de ingeniería de datos y analítica de negocio sobre el dataset de ventas de licores del estado de Iowa, utilizando la arquitectura Medallion (Bronze → Silver → Gold) en Databricks con Delta Lake. Incluye:
- Ingesta cruda a Bronze desde Volumes.
- Limpieza, tipado y reglas de negocio en Silver.
- Modelo dimensional en Gold (dimensiones y hecho de ventas) con cargas idempotentes y SCD Tipo 1 vía MERGE.
- Capa de análisis de negocio con SQL y visualizaciones (tendencias, ranking de tiendas/condados, categorías y rentabilidad).

El contenido de este README se basa en los notebooks del proyecto y el documento de análisis adjunto.

## Arquitectura (Medallion)
- Bronze: almacenamiento crudo, esquema laxo en texto, auditoría y señalización de registros corruptos.
- Silver: datos limpios y tipados con reglas de calidad y derivación de métricas base.
- Gold: modelo estrella con dimensiones `dim_date`, `dim_vendor`, `dim_store`, `dim_product` y la tabla de hechos `fact_sales`.

## Estructura del repositorio
- `ClearTech_Final_Project/NB0_Initial_Setup.ipynb`: creación de catálogo, esquemas, volumes y `dim_date`.
- `ClearTech_Final_Project/bronze/NB1_Loading_Bronze.ipynb`: ingesta y archivado en Bronze.
- `ClearTech_Final_Project/silver/NB2_Loading_Silver.ipynb`: limpieza/transformación y escritura en Silver.
- `ClearTech_Final_Project/gold/NB3_Loading_Dimensions.ipynb`: creación y carga SCD1 de dimensiones.
- `ClearTech_Final_Project/gold/NB4_Loading_Facts.ipynb`: creación y carga de `fact_sales`.
- `ClearTech_Final_Project/NB5_Business_Analysis.ipynb`: queries y visualizaciones para responder preguntas de negocio.
- `ClearTech_Final_Project/Lucas_Gauna_Final_Project.pdf`: documento con análisis y guía de preguntas.

## Requisitos y entorno
- Databricks Runtime con soporte a Delta Lake y `dbutils`.
- Permisos para crear `Catalog`, `Schema`, `Volume` y tablas en Unity Catalog.
- Acceso a Volumes para rutas de proceso y archivo.

## Parámetros (widgets)
Los notebooks usan widgets para parametrizar la ejecución (pueden editarse en NB0):
- `catalog`: nombre del catálogo (p. ej. `iowa_sales`).
- `bronze_schema`: p. ej. `sales_bronze`.
- `silver_schema`: p. ej. `sales_silver`.
- `gold_schema`: p. ej. `sales_gold`.
- Volumes: `process_path` y `processed_path` en `Volumes/<catalog>/<bronze_schema>/...`.
- Tablas: `bronze_table` (`sales_raw`), `silver_table` (`sales_cleaned`).
- Gold: `fact_sales`, `dim_date`, `dim_vendor`, `dim_store`, `dim_product`.

## Ejecución paso a paso
1) NB0_Initial_Setup
   - Crea `CATALOG` y `SCHEMA` (Bronze, Silver, Gold).
   - Crea `VOLUME` de `process` y `processed` en Bronze.
   - Crea y puebla `gold.dim_date` (2012-01-01 a 2025-12-31) con claves surrogate `date_key`.

2) NB1_Loading_Bronze
   - Lee archivos CSV desde `process_path` con un esquema totalmente `STRING` para evitar pérdida de datos.
   - Crea `bronze.sales_raw` con columnas crudas + `saved_date` y `is_corrupted`.
   - Inserta datos con chequeos de casteo (marca `is_corrupted = true` si fallan tipados críticos).
   - Archiva los archivos procesados a `processed_path` con sufijo de fecha.

3) NB2_Loading_Silver
   - Crea `silver.sales_cleaned` con esquema tipado (INT/DECIMAL/DATE) y medidas derivadas.
   - Filtra `is_corrupted = false` y descarta nulos en columnas críticas.
   - Completa valores no críticos (dirección, ciudad, etc.).
   - Recalcula `sale_dollars`, `volume_sold_liters`, `volume_sold_gallons` y aplica validaciones (> 0 donde corresponde).
   - Inserta de forma idempotente en Silver (INSERT OVERWRITE desde vista temporal).

4) NB3_Loading_Dimensions
   - Crea dimensiones si no existen:
     - `gold.dim_vendor(vendor_key, vendor_business_key, vendor_name, saved_date)`
     - `gold.dim_product(product_key, product_business_key, item_description, pack, bottle_volume_ml, category_code, category_name, saved_date)`
     - `gold.dim_store(store_key, store_business_key, store_name, address, city, zip_code, county, saved_date)`
   - Carga SCD Tipo 1 vía `MERGE` desde Silver, tomando el registro más reciente por business key (`ROW_NUMBER() OVER (...) = 1`). Actualiza atributos si cambian.

5) NB4_Loading_Facts
   - Crea `gold.fact_sales` con surrogate key `sale_id`, degenerate key `sale_invoice_line_no`, FKs a dimensiones y medidas cuantitativas.
   - `MERGE` idempotente: deduplica por `invoice_line_no` y resuelve claves surrogate desde dimensiones (`COALESCE(..., -1)` para desconocidos).

6) NB5_Business_Analysis
   - Responde preguntas de negocio con SQL y gráficos (Seaborn/Matplotlib) sobre Gold:
     - Tendencias temporales: por año/mes/día (`fact_sales` + `dim_date`).
     - Ranking de tiendas y condados (`dim_store`).
     - Categorías más vendidas y de mayor crecimiento (`dim_product`).
     - Rentabilidad (margen bruto) por artículo, categoría y vendedor.
     - Análisis de puntos de precio vs. volumen para optimización de precios.

## Modelo dimensional (Gold)
- Dimensiones:
  - `dim_date(date_key, full_date, day, month, month_name, quarter, year, day_of_week, day_name, week_of_year, is_a_holliday)`
  - `dim_vendor(vendor_key, vendor_business_key, vendor_name, saved_date)`
  - `dim_store(store_key, store_business_key, store_name, address, city, zip_code, county, saved_date)`
  - `dim_product(product_key, product_business_key, item_description, pack, bottle_volume_ml, category_code, category_name, saved_date)`
- Hechos:
  - `fact_sales(sale_id, sale_invoice_line_no, date_key, product_key, store_key, vendor_key, state_bottle_cost, state_bottle_retail, bottles_sold, sale_dollars, volume_sold_liters, volume_sold_gallons, saved_date)`

## Calidad de datos y gobernanza
- Señalización de registros corruptos en Bronze mediante validación de cast a tipos numéricos.
- Reglas en Silver: drop de nulos críticos, relleno de no-críticos, tipado estricto, filtros > 0, derivación consistente de métricas.
- Cargas idempotentes (INSERT OVERWRITE/MERGE) y SCD1 para mantener atributos actuales por business key.
- Uso de Unity Catalog para separación por `catalog`/`schema` y Volumes para trazabilidad de archivos fuente.

## Orquestación sugerida
- Crear un Job en Databricks con tareas en cadena: NB0 → NB1 → NB2 → NB3 → NB4 → NB5.
- Pasar los mismos widgets a todas las tareas para consistencia.
- Programación periódica según frecuencia de llegada de archivos a `process_path`.

## Cómo preparar y ejecutar
1. Abrir y ejecutar `NB0_Initial_Setup.ipynb` para crear catálogo/esquemas/volumes y `dim_date`.
2. Depositar CSVs en `process_path` (Volume de Bronze). Formato encabezado `header=true`.
3. Ejecutar `bronze/NB1_Loading_Bronze.ipynb` para ingestar y archivar.
4. Ejecutar `silver/NB2_Loading_Silver.ipynb` para transformar y escribir en Silver.
5. Ejecutar `gold/NB3_Loading_Dimensions.ipynb` y `gold/NB4_Loading_Facts.ipynb` para poblar Gold.
6. Ejecutar `NB5_Business_Analysis.ipynb` para consultas y visualizaciones.

## Preguntas de negocio cubiertas (ejemplos)
- Tendencias de ventas YoY/YM/Día:
  - SUM(`sale_dollars`) agrupado por `year`, `year_month` o `full_date` desde `fact_sales` + `dim_date`.
- Ranking de tiendas/condados:
  - SUM(`sale_dollars`) por `store_name`/`county` desde `fact_sales` + `dim_store`.
- Categorías líderes y crecimiento:
  - SUM(`sale_dollars`) y LAG YoY por `category_name` desde `fact_sales` + `dim_product` + `dim_date`.
- Rentabilidad (margen bruto):
  - `gross_margin = SUM(sale_dollars - state_bottle_cost * bottles_sold)` y `margin_rate = gross_margin / SUM(sale_dollars)` por artículo/categoría/vendedor.
- Optimización de precios:
  - Volumen y revenue por punto de precio (`state_bottle_retail`) en el tiempo por producto.

## Hallazgos esperados (alto nivel)
- Estacionalidad marcada en ventas mensuales y patrones semanales estables.
- Concentración de ventas en categorías líderes (p. ej., vodkas/whiskies) y tiendas top.
- Variabilidad en `margin_rate` que permite priorizar surtido y promociones.
- Sensibilidad de volumen a precios en ciertos productos con múltiples puntos de precio.

## Notas
- Las rutas de Volumes y nombres de tablas son configurables vía widgets.
- El uso de `COALESCE(..., -1)` en hechos permite detectar llaves desconocidas y completar dimensiones posteriormente.
- Todos los notebooks son re-ejecutables sin duplicar datos gracias a `MERGE`/`INSERT OVERWRITE`.

## Autor
Lucas Gauna – ClearTech Academy (Databricks Academy)