   # Pobreza Extrema en Colombia — Proyecto ETL (G51)

**Universidad Autónoma de Occidente — Facultad de Ingeniería y Ciencias Básicas**
**Programa:** Ingeniería de Datos e Inteligencia Artificial
**Curso:** ETL (G51) — Primera Entrega

> - Sebastián Alejandro Jimenéz Solís —
> - Laura Valentina Loaiza Diaz —
> - Santiago Lourido Rodriguez —
> - Carlos Fabian Cordoba —                  

---

## Tabla de contenido

1. [Objetivo del proyecto](#1-objetivo-del-proyecto)
2. [Alineación con los ODS](#2-alineación-con-los-ods)
3. [Requerimientos y preguntas de negocio](#3-requerimientos-y-preguntas-de-negocio)
4. [Evaluación y descripción de las fuentes de datos](#4-evaluación-y-descripción-de-las-fuentes-de-datos)
5. [Arquitectura del proyecto (ETL)](#5-arquitectura-del-proyecto-etl)
6. [Modelo de datos: esquema de galaxia](#6-modelo-de-datos-esquema-de-galaxia)
7. [Stack tecnológico y justificación](#7-stack-tecnológico-y-justificación)
8. [Seguridad y manejo de credenciales](#8-seguridad-y-manejo-de-credenciales)
9. [Proceso de transformación — qué se hizo y por qué](#9-proceso-de-transformación--qué-se-hizo-y-por-qué)
10. [Análisis Exploratorio de Datos (EDA) — resultados](#10-análisis-exploratorio-de-datos-eda--resultados)
11. [Carga a PostgreSQL](#11-carga-a-postgresql)
12. [⚠️ Observaciones importantes / riesgos frente a la rúbrica](#12-️-observaciones-importantes--riesgos-frente-a-la-rúbrica)
13. [Estructura del repositorio](#13-estructura-del-repositorio)
14. [Cómo ejecutar el proyecto](#14-cómo-ejecutar-el-proyecto)
15. [Estado actual y pendientes](#15-estado-actual-y-pendientes)

---

## 1. Objetivo del proyecto

Analizar los determinantes y la distribución de la **pobreza y la pobreza extrema (indigencia)** en los hogares colombianos, a partir de datos de encuesta oficial, con el fin de responder preguntas relevantes para el diseño de política social: qué tan profunda es la pobreza, en qué zonas se concentra, y qué características de los hogares y las personas están asociadas a ella.

El proyecto cubre el ciclo completo de un pipeline de datos: extracción de los archivos fuente, diseño de arquitectura, modelado dimensional, migración a una base de datos relacional, limpieza/transformación con buenas prácticas de EDA, y construcción de visualizaciones que respondan directamente a las preguntas planteadas.

## 2. Alineación con los ODS

El proyecto se alinea con el **ODS 1: Fin de la Pobreza** de la Agenda 2030 de la ONU.

> A nivel mundial, el 6,9% de la población (cerca de 800 millones de personas) sigue viviendo en pobreza extrema, apenas 1,5 puntos porcentuales por debajo de 2015 (8,4%), y se prevé que la cifra se mantenga prácticamente estancada en los próximos años (World Economic Forum, 2024). Entender cómo se comporta este fenómeno a nivel de hogares y personas en Colombia permite dimensionar el problema localmente y aporta evidencia útil para el diseño de política pública.

## 3. Requerimientos y preguntas de negocio

Preguntas guía del proyecto (ya respondidas en el EDA — ver sección 10):

1. **¿Los hogares con mayor número de personas presentan mayor probabilidad de estar en pobreza extrema?**
   Se analiza la relación entre `nper`/`npersug` (tamaño del hogar) y la condición de indigencia, así como la correlación con la **brecha de pobreza**.
2. **¿En qué departamentos se concentran los hogares con menores ingresos y mayor pobreza extrema?**
   Uso del campo `dpto`, ponderando siempre por el factor de expansión `fex_c` / `fex_dpto` (no se puede contar filas directamente).
3. **¿La pobreza extrema se distribuye igual entre zonas urbanas y rurales?**
   Comparación usando `clase` (1 = urbano, 2 = rural) y `dominio` (que además distingue ciudades capitales del resto urbano).
4. **¿Quiénes están en pobreza extrema y qué tan lejos están de salir de ella?**
   Respondida mediante la variable calculada de **brecha de pobreza** (ver sección 8).

**Nota metodológica clave:** todas las tablas provienen de una **encuesta con muestreo complejo**, por lo que cualquier conteo, promedio o proporción agregada debe calcularse ponderando por la columna `fex_c` (factor de expansión). Una fila con `fex_c = 250` representa a 250 hogares/personas reales, no a 1.

## 4. Evaluación y descripción de las fuentes de datos

### 4.1. `Hogares.csv` — nivel hogar 

- **Filas:** 276.666 · **Columnas:** 23 · **Llave:** `directorio` + `secuencia_p` (verificado: 0 duplicados)
- Rango de `li` y `lp` verificado contra los valores oficiales del DANE para el período — **OK**, sin valores fuera de rango.
- Variables clave: `ingtotug`, `ingpcug`, `lp`, `li`, `pobre`, `indigente`, `npobres`, `nindigentes`, `fex_c`, `dpto`, `dominio`, `clase`, `p5090` (tenencia de vivienda).

| Campo | Significado |
|---|---|
| `directorio` | Llave de la vivienda |
| `secuencia_p` | Llave del hogar |
| `mes` | Mes de la encuesta (1–12) |
| `clase` | 1 = Urbano, 2 = Rural |
| `dominio` | Ciudad capital / resto urbano / rural |
| `ciudad_sinam` | Ciudad sin área metropolitana (33,4% nulo — solo aplica a ciertas capitales) |
| `nper` / `npersug` | Número de personas en el hogar / en la unidad de gasto |
| `ingtotug` | Ingreso total del hogar (unidad de gasto) |
| `ingpcug` | Ingreso per cápita del hogar |
| `lp` | Línea de pobreza |
| `li` | Línea de indigencia (pobreza extrema) |
| `pobre` | 1 = hogar pobre, 0 = no pobre |
| `indigente` | 1 = hogar en pobreza extrema, 0 = no |
| `npobres` / `nindigentes` | Número de personas pobres / indigentes en el hogar |
| `fex_c` / `fex_dpto` | Factores de expansión (nacional / departamental) |
| `dpto` | Departamento |

- **Calidad de los datos (chequeo inicial):**
  - `p5100` (cuota de vivienda) 96,7% nula y `p5130` (arriendo estimado) 40,5% nula — nulos **estructurales**, no error: solo aplican según el tipo de tenencia de vivienda (`p5090`).
  - `ciudad_sinam` 33,4% nula — estructural, solo aplica a hogares en ciertas capitales.
  - El resto de columnas (incluidas todas las usadas para el análisis de pobreza) **no tiene nulos**.
  - Distribución: 66.949 hogares pobres (24,2% de la muestra sin ponderar) y 21.605 en indigencia (7,8% sin ponderar) — **[COMPLETAR]** recalcular estos porcentajes ponderando por `fex_c` para el dato representativo de la población real.
    
### 4.2. `Personas.csv` — nivel persona (**pendiente de integrar**)

- 773.932 filas, 133 columnas, comparte `directorio`+`secuencia_p` con `Hogares.csv`. Ver sección 15 (pendientes).

## 5. Arquitectura del proyecto (ETL)

Implementada en Google Colab, con el siguiente flujo real:

```
Google Drive (Hogares.csv)
        │  pd.read_csv
        ▼
   Extracción y validación
   (duplicados, rangos oficiales lp/li)
        │
        ▼
   Transformación (pandas)
   nulos, categorías legibles, brecha de pobreza,
   flag de outliers, tipado a category
        │
        ▼
   Construcción de dim_geografia, dim_hogar, dim_tiempo
   y fact_hogares (esquema de galaxia)
        │
        ▼
   EDA (3 preguntas de negocio) — ejecutado sobre el
   DataFrame en memoria (ver sección 12, riesgo de rúbrica)
        │
        ▼
   Carga a PostgreSQL (Supabase), vía SQLAlchemy
   DROP + CREATE TABLE + to_sql()
```

## 6. Modelo de datos: esquema de galaxia

Se eligió un **esquema de galaxia (fact constellation)** porque el proyecto tiene **dos procesos de negocio con grano distinto** (el hogar y la persona) que comparten varias dimensiones. Un esquema de estrella único no encajaría bien porque forzaría a mezclar dos granularidades en una sola tabla de hechos.

| Tabla | Grano implementado | Columnas cargadas |
|---|---|---|
| `dim_geografia` | 1 fila por combinación única `dpto`+`dominio`+`ciudad_sinam`+`clase` | `id_geografia`, `dpto`, `dominio`, `ciudad_sinam`, `clase` |
| `dim_hogar` | 1 fila por hogar único (`directorio`+`secuencia_p`) | `id_hogar`, `directorio`, `secuencia_p`, `p5090` |
| `dim_tiempo` | 1 fila por mes | `id_tiempo`, `mes`, `nombre_mes` |
| `fact_hogares` | 1 fila por hogar | `id_hogar`, `id_geografia`, `id_tiempo`, `ingtotug`, `ingpcug`, `lp`, `li`, `brecha_pobreza`, `brecha_pct_pobreza`, `pobre`, `indigente`, `npobres`, `nindigentes`, `fex_c` |

`dim_geografia`, `dim_hogar` y `dim_tiempo` se construyeron como **dimensiones conformadas**: mismas llaves y mismo grano que usará el futuro `fact_personas`, para poder anexarlo más adelante (a partir de `Personas.csv`) sin duplicar estos atributos ni romper el modelo. Ese es justamente el criterio que hace que el esquema sea de **galaxia** y no una estrella aislada.

**Diferencias frente al diseño completo del documento de modelo de datos** (a decidir si se cierran antes de la entrega):
- `dim_hogar` no incluye todavía `tamano_hogar_rango` (el bucket de `nper`) aunque el comentario del notebook lo anunciaba — hoy esa columna solo vive en el DataFrame de EDA, no en la tabla que se cargó a Postgres.
- `dim_geografia` no incluye `tipo_zona` (Ciudad capital / Resto urbano / Rural) por la misma razón — se calculó y se usó para el EDA, pero no se persistió en la tabla dimensional.
- No se implementó la junk dimension `dim_condicion_pobreza`: `pobre`/`indigente` se dejaron como columnas enteras directamente en `fact_hogares` (una simplificación razonable para esta primera entrega).
- `fex_dpto` (factor de expansión departamental) no se cargó en `fact_hogares`, solo `fex_c` (factor nacional). Para las agregaciones por departamento de la Pregunta 2, en rigor debería usarse `fex_dpto`; usar `fex_c` es una aproximación aceptable pero vale la pena dejarlo anotado como decisión metodológica.

## 7. Stack tecnológico y justificación

| Capa | Tecnología | Justificación |
|---|---|---|
| Entorno de desarrollo | **Google Colab** + Google Drive | Sin instalación local, fácil de compartir entre el equipo, `drive.mount()` para leer el CSV directamente desde Drive |
| Lenguaje / ETL | **Python** (pandas, numpy) | Manejo maduro de datasets grandes (276k filas) y limpieza de datos |
| Base de datos | **PostgreSQL gestionado en Supabase** | Motor relacional en la nube, no requiere infraestructura propia del equipo, accesible para todos los integrantes |
| Conexión Python↔DB | **SQLAlchemy + psycopg2-binary** | `create_engine` + `to_sql()` para cargar los DataFrames de pandas directamente a las tablas de Postgres |
| Visualización (EDA) | **matplotlib** | Gráficos de barras e histograma usados en las 3 preguntas del EDA |
| Control de versiones | **Git / GitHub** | Exigido por la cátedra |

## 8. Seguridad y manejo de credenciales

> Esto se corrigió durante la revisión del notebook y es importante dejarlo documentado.

El borrador original tenía el **usuario y la contraseña de PostgreSQL escritos directamente en la cadena de conexión**, en texto plano dentro del notebook. Como el repositorio del proyecto es justamente lo que se entrega y evalúa (incluyendo README y `.gitignore`), esa contraseña habría quedado expuesta públicamente en GitHub.

**Corrección aplicada:**
- La contraseña ahora se solicita en tiempo de ejecución con `getpass.getpass()`, y nunca se escribe en el archivo `.ipynb`.
- El usuario y el host sí pueden quedar en el notebook (no son secretos por sí solos), pero la contraseña no.

**Acción pendiente para el equipo:** dado que la contraseña original ya estuvo escrita en texto plano en un archivo que se compartió, se recomienda **rotarla en el panel de Supabase** antes de la entrega final, aunque ya se haya corregido el notebook.

**Buenas prácticas adicionales sugeridas para cuando se suba el repo:**
- Agregar `.env` (con las credenciales) al `.gitignore`.
- Nunca commitear notebooks ya ejecutados que muestren la contraseña en el output de una celda (revisar outputs antes de subir).

## 9. Proceso de transformación — qué se hizo y por qué

| Transformación | Cómo | Por qué |
|---|---|---|
| Nulos en `p5100`, `p5130` | Se dejan como `NULL` numérico | Representan "no aplica" según el tipo de tenencia de vivienda (`p5090`), no un dato faltante real. Rellenarlos con texto tipo `"No aplica"` habría roto el tipo numérico de esas columnas de dinero, impidiendo sumarlas/promediarlas. |
| Nulos en `ciudad_sinam` | `fillna("No aplica")` | Es una columna de texto (categórica), así que rellenar con una etiqueta explícita sí es seguro y no rompe ningún cálculo numérico. |
| `p5090` → `categoria_tenencia` | `map()` con el diccionario oficial (1=Propia totalmente pagada … 7=Otra) | Traduce el código numérico de la encuesta a una etiqueta legible para el EDA y los gráficos, sin alterar la columna original `p5090` (que se conserva para la carga a `dim_hogar`). |
| `clase` → `clase_desc`, `tipo_zona` | `map()` / `np.where()` combinando `clase` + `dominio` | Da una categorización de negocio (Urbano/Rural, y luego Ciudad capital / Resto urbano / Rural) más útil para agrupar en el EDA que los códigos crudos. |
| `tamano_hogar_rango` | `pd.cut(nper, bins=[0,2,4,100])` → "1-2"/"3-4"/"5+" | Convierte una variable continua de conteo en categorías interpretables para responder directamente la Pregunta 1 (tamaño del hogar vs. indigencia) sin perder mucha granularidad. |
| `brecha_pobreza` / `brecha_pct_pobreza` | `(lp - ingpcug) / lp` | Mide qué tan lejos está el ingreso per cápita del hogar de la **línea de pobreza**. Valor positivo = por debajo de la línea (más alto = más profunda la pobreza); esta es la que se carga a `fact_hogares`. |
| `brecha_indigencia` / `brecha_pct_indigencia` | `(li - ingpcug) / li` | Misma lógica pero contra la **línea de indigencia**; se usa para responder la Pregunta 3 y se queda solo en el DataFrame de trabajo (no se persiste en el modelo dimensional, ya que la Pregunta 3 se resuelve directamente en el EDA). |
| Redondeo de variables monetarias a 2 decimales | `.round(2)` | Estándar del diccionario oficial del DANE para estas variables. |
| `ingtotug_outlier` (flag, no eliminación) | `ingtotug <= 0` → booleano | Se identificaron **2.195 hogares (0,79%)** con ingreso total ≤ 0. En vez de eliminarlos (lo que distorsionaría la representatividad poblacional del diseño muestral), se **marcan** con una bandera para que el análisis decida si los excluye puntualmente o no — buena práctica cuando se trabaja con datos de encuesta con factor de expansión. |
| Tipado a `category`: `mes`, `clase`, `p5090` | `.astype("category")` | Son columnas con pocos valores posibles que se repiten miles de veces (12, 2 y 7 categorías respectivamente); tipar como `category` reduce el uso de memoria y acelera los `groupby()` del EDA. |
| `pobre`/`indigente` se mantienen en `int64` (no `bool`) | — | Porque `fact_hogares` las carga en columnas `INTEGER` de PostgreSQL, y Postgres no convierte automáticamente `bool → integer` al insertar con `to_sql()`. |
| Chequeo `nper` vs. `npersug` | Comparación directa, solo impresión de resultado | Es un chequeo de **consistencia diagnóstico**: se encontraron **1.225 filas (0,44%)** donde difieren (el tamaño del hogar no coincide exactamente con el tamaño de la unidad de gasto — son conceptos distintos en la metodología DANE, así que la diferencia es esperable y **no se corrige**, solo se documenta). |

## 10. Análisis Exploratorio de Datos (EDA) — resultados

> Resultados reproducidos ejecutando las mismas operaciones del notebook sobre `Hogares.csv`; el notebook enviado tenía estas celdas sin ejecutar todavía — conviene correrlas y dejar los outputs guardados antes de la entrega final.

### Pregunta 1 — ¿Hogares más grandes tienen mayor probabilidad de pobreza extrema?

| Tamaño del hogar | % en indigencia (ponderado) |
|---|---|
| 1-2 personas | 4,44% |
| 3-4 personas | 8,17% |
| 5 o más personas | 16,84% |

**Correlación ponderada (tamaño del hogar vs. indigencia): r = 0,153** — positiva pero débil: hay una tendencia clara y consistente (a mayor tamaño del hogar, mayor tasa de indigencia, casi 4x entre el grupo más pequeño y el más grande), aunque el tamaño del hogar por sí solo no explica la mayor parte de la variación en la condición de indigencia.

### Pregunta 2 — Distribución geográfica y urbano/rural

**Departamentos con menor ingreso per cápita ponderado:** Chocó (~$596.400), La Guajira (~$597.965), Sucre (~$768.361), Córdoba (~$821.031), Cauca (~$876.781).

**Departamentos con mayor ingreso per cápita ponderado:** Bogotá D.C. (~$2.734.383), Antioquia (~$1.896.476), Cundinamarca (~$1.637.964), Valle del Cauca (~$1.586.424), Risaralda (~$1.513.259).

**Pobreza e indigencia por tipo de zona (ponderado):**

| Zona | % pobre | % indigente |
|---|---|---|
| Ciudad capital | 17,60% | 4,38% |
| Resto urbano | 23,30% | 7,58% |
| Rural | 32,30% | 14,77% |

La brecha es marcada: la tasa de indigencia rural (14,77%) es más de 3 veces la de ciudad capital (4,38%).

### Pregunta 3 — ¿Qué tan lejos están de salir de la indigencia los hogares que ya están en ella?

- **21.605 hogares** de la muestra están en indigencia (sin ponderar), que representan **~1.367.073 hogares reales** al ponderar por `fex_c`, sobre un total nacional estimado de ~18.199.756 hogares (**7,51% de indigencia ponderada a nivel nacional**, y **22,35% de pobreza ponderada**).
- **Brecha mediana ponderada: 34,1%** por debajo de la línea de indigencia — es decir, la mitad de los hogares indigentes tiene un ingreso per cápita que está a más de un tercio por debajo del umbral que los sacaría de la indigencia; no es un déficit marginal.

## 11. Carga a PostgreSQL

El notebook: (1) crea las tablas en Postgres con `DROP TABLE IF EXISTS` + `CREATE TABLE` vía `engine.connect()` y `text()`, (2) carga `dim_geografia`, `dim_hogar`, `dim_tiempo` con `to_sql(..., if_exists="append")`, (3) carga `fact_hogares` con `chunksize=10000` (por el volumen, ~277 mil filas), y (4) valida con un `SELECT COUNT(*)` por tabla.

**[COMPLETAR]** Pegar aquí los conteos reales una vez se ejecute contra la base de datos (deberían ser: `dim_hogar` y `fact_hogares` con 276.666 filas cada una, y `dim_geografia`/`dim_tiempo` con la cantidad de combinaciones únicas de cada una).

## 12. ⚠️ Observaciones importantes / riesgos frente a la rúbrica

Vale la pena resolver esto **antes de la entrega**, porque toca directamente ítems de la rúbrica:

1. **El EDA se ejecuta sobre el DataFrame de pandas (leído del CSV), no contra PostgreSQL.** El propio notebook lo dice explícitamente: *"El EDA trabaja sobre `df`... no sobre las tablas dimensionales, que solo existen para la carga a Postgres."* La rúbrica de la entrega pide explícitamente: *"Visualizations and analysis are powered exclusively by SQL queries or connections to the database"* (ítem "Data Extraction from DB", 10%). Tal como está hoy, ese ítem se perdería.
   **Sugerencia:** después de cargar las tablas a Postgres, releer los datos con `pd.read_sql("SELECT ... FROM fact_hogares JOIN dim_hogar ...", engine)` y rehacer las 3 agrupaciones del EDA a partir de ese resultado (los números no deberían cambiar, porque son los mismos datos — es un cambio de *dónde* se consulta, no de *qué* se calcula).
2. **`tipo_zona` y `tamano_hogar_rango`** se usaron para el EDA pero no quedaron en las tablas dimensionales cargadas a Postgres. Si se resuelve el punto 1 haciendo el EDA por SQL, hace falta que estas columnas (o su lógica equivalente en SQL con `CASE WHEN`) estén disponibles en la base de datos.
3. **`fex_dpto` no se cargó a `fact_hogares`** — si se quiere ser metodológicamente estricto en la Pregunta 2 (agregación por departamento), debería usarse `fex_dpto` en vez de `fex_c` para esos cálculos específicos.

## 13. Estructura del repositorio

```
├── data/
│   └── raw/                # Hogares.csv, Personas.csv (raw)
├── notebooks/
│   └── ETL___EDA_Hogares.ipynb
├── sql/
│   └── schema.sql
├── docs/
│   └── modelo_datos_esquema_galaxia.md
├── .gitignore               # debe excluir .env / credenciales
├── requirements.txt         # [COMPLETAR] pandas, numpy, matplotlib, sqlalchemy, psycopg2-binary
└── README.md
```

## 14. Cómo ejecutar el proyecto

1. Abrir `notebooks/ETL___EDA_Hogares.ipynb` en Google Colab.
2. Montar Google Drive y ajustar `ruta_base` a la carpeta donde esté `Hogares.csv`.
3. Ejecutar las celdas de instalación (`psycopg2-binary`, `sqlalchemy`).
4. Al llegar a la celda de conexión, ingresar la contraseña de Postgres cuando la pida `getpass` (no escribirla en el notebook).
5. Ejecutar el resto de celdas en orden: extracción → transformación → construcción de dimensiones/hechos → EDA → carga a Postgres.

## 15. Estado actual y pendientes

**Hecho:**
- ✅ Extracción y validación de `Hogares.csv` (llave única, rangos oficiales de `li`/`lp`).
- ✅ Transformaciones documentadas y justificadas (sección 9).
- ✅ Modelo de galaxia parcial construido: `dim_geografia`, `dim_hogar`, `dim_tiempo`, `fact_hogares`.
- ✅ EDA de las 3 preguntas de negocio, con resultados ponderados (sección 10).
- ✅ Carga a PostgreSQL (Supabase) vía SQLAlchemy.
- ✅ Manejo seguro de credenciales corregido (`getpass`).

**Pendiente:**
- ⬜ Resolver el punto de la sección 12 (EDA vía SQL contra la base de datos, no contra el CSV) — es lo más urgente frente a la rúbrica.
- ⬜ Incorporar `Personas.csv`: construir `dim_persona` y `fact_personas`, enlazadas por `dim_hogar` (el diseño completo ya está en `docs/modelo_datos_esquema_galaxia.md`).
- ⬜ Decidir si se persisten `tipo_zona` y `tamano_hogar_rango` en las tablas dimensionales.
- ⬜ Rotar la contraseña de Supabase que quedó expuesta en el borrador original.
- ⬜ Ejecutar el notebook completo y guardar los outputs reales (conteos de carga, gráficos).
- ⬜ Informe técnico final (puede reutilizar buena parte de este README).
- ⬜ `requirements.txt`, `.gitignore` con `.env` excluido, y nombres de los integrantes.

---

*Este README se actualiza a medida que avanza la implementación; los bloques marcados con **[COMPLETAR]** son los puntos que el equipo debe resolver antes de la entrega.*
