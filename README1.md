# Pobreza Extrema en Colombia — Proyecto ETL (G51)

**Universidad Autónoma de Occidente — Facultad de Ingeniería y Ciencias Básicas**
**Programa:** Ingeniería de Datos e Inteligencia Artificial
**Curso:** ETL (G51) — Primera Entrega

> - Sebastián Alejandro Jimenéz Solís
> - Laura Valentina Loaiza Diaz
> - Santiago Lourido Rodriguez
> - Carlos Fabian Cordoba

## Objetivo del proyecto
Analizar los determinantes y la distribución de la pobreza y la pobreza extrema (indigencia) en los hogares colombianos, a partir de datos de encuesta oficial. El proyecto cubre el ciclo completo de un pipeline de datos: extracción de archivos fuente, transformación, y migración a una base de datos relacional para su análisis.

## Stack tecnológico y justificación
El proyecto utiliza un ecosistema moderno orientado al procesamiento en la nube:
* **Entorno de ejecución:** Google Colab + Google Drive. Permite trabajar sin instalación local, facilita la colaboración en equipo y permite montar la unidad (`drive.mount()`) para leer los CSV pesados directamente.
* **Lenguaje y Procesamiento (ETL):** Python (Pandas, NumPy). Herramientas robustas y eficientes para manejar la limpieza de datasets grandes (más de 276 mil filas) y aplicar transformaciones complejas.
* **Base de Datos (Data Warehouse):** PostgreSQL gestionado en Supabase. Un motor relacional en la nube que no requiere infraestructura propia y es accesible para todos los integrantes para ejecutar las consultas SQL del EDA.
* **Conectividad:** SQLAlchemy + psycopg2-binary. Utilizados para crear el motor de conexión (`create_engine`) y cargar los DataFrames directamente a PostgreSQL mediante `.to_sql()`.

## Configuración de `.gitignore`
El repositorio cuenta con un archivo `.gitignore` correctamente configurado para excluir binarios, cachés de Python (`__pycache__/`, `.ipynb_checkpoints/`), archivos de entorno local (`.env`) y los datos crudos (`.csv`), garantizando la seguridad de las credenciales y manteniendo el repositorio ligero.

## Instrucciones de ejecución
1. **Clonar el repositorio y abrir el entorno:** Abrir el archivo `notebooks/ETL___EDA_Hogares.ipynb` en Google Colab.
2. **Montar volúmenes:** Ejecutar la celda para montar Google Drive y ajustar la variable `ruta_base` a la carpeta donde se encuentra alojado el archivo `Hogares.csv`.
3. **Instalar dependencias:** Ejecutar las celdas iniciales que instalan `psycopg2-binary` y `sqlalchemy`[cite: 7].
4. **Credenciales seguras:** Al llegar a la celda de conexión a la base de datos, el sistema solicitará la contraseña de PostgreSQL mediante la librería `getpass` (esto evita que la clave quede en texto plano)[cite: 7].
5. **Ejecución del Pipeline:** Correr el resto de las celdas en orden. El script realizará la extracción, transformación, construcción de dimensiones/hechos, carga a Postgres y finalmente ejecutará el Análisis Exploratorio de Datos (EDA) conectándose a la base de datos[cite: 7].
