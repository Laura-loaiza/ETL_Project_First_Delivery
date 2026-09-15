# Pobreza Extrema en Colombia — Proyecto ETL (G51)

**Universidad Autónoma de Occidente — Facultad de Ingeniería y Ciencias Básicas**
**Programa:** Ingeniería de Datos e Inteligencia Artificial
**Curso:** ETL (G51) — Primera Entrega

> - Sebastián Alejandro Jimenéz Solís
> - Laura Valentina Loaiza Diaz
> - Santiago Lourido Rodriguez
> - Carlos Fabian Cordoba

## Objetivo del proyecto

Analizar la distribución y la profundidad de la **pobreza y la pobreza extrema (indigencia)** en los hogares colombianos, a partir de datos de encuesta oficial, para responder preguntas útiles al diseño de política social: qué tan profunda es la pobreza, dónde se concentra, y qué características de los hogares y de las personas están asociadas a ella.

El proyecto cubre el ciclo completo de un pipeline de datos: extracción de los archivos fuente, diseño de arquitectura, modelado dimensional, migración a una base de datos relacional, limpieza y transformación documentadas, y visualizaciones construidas **a partir de consultas SQL contra la base de datos**, no con el CSV original.

## Preguntas

**Nivel hogar (resueltas en el notebook de hogares):**

1. ¿Los hogares con mayor número de personas presentan mayor probabilidad de estar en pobreza extrema?
2. ¿En qué departamentos se concentran los hogares con menores ingresos?
3. ¿La pobreza extrema se distribuye igual entre zonas urbanas y rurales, y qué tan lejos están de salir de ella los hogares que ya están en indigencia?

**Nivel persona (notebook de personas):**

4. ¿Las personas de hogares pobres tienen mayor presencia en trabajos de baja remuneración o en múltiples empleos?
5. ¿Dónde hay más pobreza extrema, a nivel de personas?
6. ¿Quiénes están en pobreza extrema (sexo, edad)?
7. ¿Qué relación tiene el nivel educativo con la pobreza extrema?
8. ¿Cómo se comportan los ingresos de las personas según la condición de pobreza de su hogar?

## Stack tecnológico y justificación

| Capa | Tecnología | Justificación |
|---|---|---|
| Entorno de desarrollo | **Google Colab** + Google Drive | Sin instalación local, fácil de compartir entre el equipo, `drive.mount()` para leer el CSV directamente desde Drive |
| Lenguaje / ETL | **Python** (pandas, numpy) | Manejo maduro de datasets grandes (276k filas) y limpieza de datos |
| Base de datos | **PostgreSQL gestionado en Supabase** | Motor relacional en la nube, no requiere infraestructura propia del equipo, accesible para todos los integrantes |
| Conexión Python↔DB | **SQLAlchemy + psycopg2-binary** | `create_engine` + `to_sql()` para cargar los DataFrames de pandas directamente a las tablas de Postgres |
| Visualización (EDA) | **matplotlib** | Gráficos de barras e histograma usados en las preguntas del EDA |
| Control de versiones | **Git / GitHub** | Exigido por la cátedra |

## Arquitectura del proyecto

El pipeline de **Hogares** sigue extracción → transformación → carga → EDA de forma directa. El pipeline de **Personas** usa además una **arquitectura de medallón** (Bronce → Plata → Oro) antes de llegar al mismo esquema final:

- **Bronce**: datos crudos del CSV de Personas, tal como vienen.
- **Plata**: limpieza, columnas legibles (`sexo`, `grupo_edad`, `nivel_educativo`) y construcción de `condicion_laboral` a partir de `oc`/`des`/`fft`.
- **Oro**: variables listas para análisis (ingresos en escala log, `tiene_segundo_empleo`), unidas contra `dim_hogar` para heredar la condición de pobreza del hogar.

El medallón describe **cómo se refinó el dato**; el esquema de galaxia (constelación) describe **cómo queda organizado el dato final para consulta**. Son complementarios, no alternativos.

```
  Google Drive (Hogares.csv, Personas.csv)
            │  pd.read_csv
            ▼
  ┌─────────────────────────────────────────────────────────────┐
  │ EXTRACCIÓN                                                   │
  │ lectura + validación: llave única, rangos oficiales lp/li,   │
  │ duplicados                                                   │
  └─────────────────────────────────────────────────────────────┘
            ▼
  ┌─────────────────────────────────────────────────────────────┐
  │ TRANSFORMACIÓN                                               │
  │ Hogares (pandas): nulos, categorías legibles, brecha de      │
  │ pobreza/indigencia, flag de outliers, tipado a category.     │
  │ Personas: arquitectura de medallón (Bronce → Plata → Oro).   │
  └─────────────────────────────────────────────────────────────┘
            ▼
  ┌─────────────────────────────────────────────────────────────┐
  │ MODELO DIMENSIONAL — esquema de galaxia (constelación)       │
  │ dim_geografia, dim_tiempo, dim_hogar (compartidas) +         │
  │ dim_persona propia · fact_hogares y fact_personas comparten  │
  │ dimensiones sin duplicar atributos entre sí                  │
  └─────────────────────────────────────────────────────────────┘
            ▼
  ┌─────────────────────────────────────────────────────────────┐
  │ CARGA — PostgreSQL (Supabase), vía SQLAlchemy                │
  │ DDL (DROP + CREATE) → prueba con 100 filas → to_sql()        │
  │ → verificación con SELECT COUNT(*)                           │
  └─────────────────────────────────────────────────────────────┘
            ▼
  ┌─────────────────────────────────────────────────────────────┐
  │ EDA Y VISUALIZACIÓN                                          │
  │ pd.read_sql() sobre el esquema de galaxia + matplotlib       │
  │ (todas las agregaciones ponderadas por fex_c dentro del SQL) │
  └─────────────────────────────────────────────────────────────┘
```

## Cómo ejecutar el proyecto

### Requisitos previos

- Cuenta de Google (para Colab) o Python 3.10+ local.
- Proyecto de PostgreSQL en Supabase con sus credenciales.
- `Hogares.csv` y `Personas.csv` descargados del portal de microdatos del DANE (GEIH) y ubicados en `data/raw/` o en la carpeta de Google Drive correspondiente.

### Pasos

1. Clonar el repositorio:
   ```bash
   git clone <url-del-repositorio>
   cd <nombre-del-repositorio>
   ```
2. Abrir `notebooks/ETL_EDA_Hogares_DEF.ipynb` en Google Colab.
3. Montar Google Drive (celda inicial) y ajustar la variable `ruta_base` a la carpeta donde está `Hogares.csv`.
4. Ejecutar las celdas iniciales que instalan `psycopg2-binary` y `sqlalchemy`.
5. En la celda de conexión, ingresar la contraseña de Postgres cuando la pida `getpass` (Hogares2026SegPass91).
6. Correr el resto de las celdas en orden: el notebook crea el esquema, hace una prueba de carga con 100 filas, carga las dimensiones y `fact_hogares`, verifica los conteos y ejecuta el EDA con `pd.read_sql`.
7. Repetir el mismo flujo con `notebooks/ETL_personas_DEF.ipynb`.

## Estructura del repositorio

```
├── README.md
├── .gitignore
├── data/
│   └── Enlaces datasets.txt
├── notebooks/
│   ├── ETL_EDA_Hogares_DEF.ipynb
│   └── ETL_personas_DEF.ipynb
├── documentos/
│   ├── ETL-Project_First Delivery.pdf
```
