# Scraping Elecciones Colombia — 8 de marzo de 2026

Web scraping de resultados electorales de las elecciones legislativas de Colombia del 8 de marzo de 2026, a partir del portal oficial de la [Registraduría Nacional del Estado Civil](https://resultados.registraduria.gov.co).

**Autor**: Alejandro Henao Ruiz

Este repositorio es un ejercicio de **Web Scraping** concebido como insumo para análisis político y periodístico, pensado tanto para el portal [El Armadillo](https://elarmadillo.co) como para mis estudiantes de **Estadística**, **Diseños Cuantitativos** y **Analítica de Datos y Machine Learning** del pregrado de Ciencia Política de la **Universidad de Antioquia**.

---

## Estructura del repositorio

```
scraping-elecciones-colombia-2026/
├── notebook/
│   └── extract/
│       ├── data_scraping_consultas.ipynb   # Notebook de extracción: Consultas Interpartidistas
│       ├── data_scraping_camara.ipynb      # Notebook de extracción: Cámara de Representantes
│       └── data_scraping_senado.ipynb      # Notebook de extracción: Senado de la República
├── data/
│   └── raw_data_extract/
│       ├── consultas_interpartidistas/     # Datos crudos: Consultas Interpartidistas
│       │   ├── cn_candidatos_municipio.csv
│       │   ├── cn_candidatos_municipio_semicolon.csv
│       │   ├── cn_candidatos_municipio.xlsx
│       │   ├── cn_candidatos_municipio.json
│       │   └── cn_candidatos_municipio.parquet
│       ├── camara/                         # Datos crudos: Cámara de Representantes
│       │   ├── ca_candidatos_municipio.csv / _semicolon.csv / .xlsx / .json / .parquet
│       │   ├── ca_candidatos_depto.csv / _semicolon.csv / .xlsx / .json / .parquet
│       │   └── ca_municipios_ganador.csv / _semicolon.csv / .xlsx / .json / .parquet
│       └── senado/                         # Datos crudos: Senado de la República
│           ├── se_candidatos_municipio.csv / _semicolon.csv / .json / .parquet
│           ├── se_candidatos_depto.csv / _semicolon.csv / .xlsx / .json / .parquet
│           └── se_municipios_ganador.csv / _semicolon.csv / .xlsx / .json / .parquet
├── config.json                             # Configuración centralizada (cookies, headers, URLs)
├── requirements.txt
├── .gitignore
└── README.md
```

---

## Configuración centralizada (`config.json`)

Todos los notebooks leen la cookie `cf_clearance`, los headers HTTP y las URLs base desde un único archivo `config.json` en la raíz del proyecto. Esto permite actualizar la cookie vencida en **un solo lugar** en vez de editar cada notebook por separado.

```json
{
  "cookies": {
    "cf_clearance": "NUEVA_COOKIE_AQUI"
  },
  "headers": { ... },
  "nomenclator_url": "https://resultados.registraduria.gov.co/json/nomenclator.json",
  "base_url": "https://resultados.registraduria.gov.co"
}
```

> **Cuando la cookie venza**: Abre el navegador → visita `resultados.registraduria.gov.co` → copia el valor de `cf_clearance` desde DevTools (Application → Cookies) → pégalo en `config.json`.

---

## Mejoras de robustez: Retry y Caching (librería estándar)

A partir de marzo 2026, todos los notebooks usan lógica de **retry** (reintentos automáticos) y **caching** simple en disco para las requests HTTP, usando solo la librería estándar de Python (`os`, `time`, `hashlib`, `json`).

- **Retry:** Si una petición falla por error de red o status >= 500, se reintenta hasta 3 veces con espera exponencial.
- **Caching:** Las respuestas exitosas se guardan en disco (`.cache/`), evitando repetir requests idénticos.

**¿Cómo funciona?**
- Todas las requests usan la función `cached_request(method, url, ...)` en vez de `requests.get`/`requests.post`.
- El cache se almacena en `.cache/` dentro del notebook.
- No requiere librerías externas.

> **Importante:** Si cambias la cookie en `config.json`, borra la carpeta `.cache/` para evitar usar respuestas viejas.

---

## Notebook: Extracción de Consultas Interpartidistas

📓 `notebook/extract/data_scraping_consultas.ipynb`

### ¿Qué hace?

Este notebook extrae, estructura y exporta los resultados de las **Consultas Interpartidistas** de las elecciones legislativas de Colombia 2026 directamente desde la API JSON de la Registraduría.

### Lógica de Python paso a paso

1. **Configuración y fetch del nomenclator**: Se realiza un `requests.get()` al endpoint `nomenclator.json` de la Registraduría, que funciona como diccionario maestro del sistema electoral. Se configura una cookie `cf_clearance` para pasar la protección Cloudflare.

2. **Construcción de tablas de dimensión**: A partir del JSON del nomenclator se construyen 7 DataFrames de pandas que funcionan como tablas de dimensión (patrón estrella):
   - `dim_elecciones` — las 4 elecciones con sus circunscripciones
   - `dim_niveles` — la jerarquía geográfica de 7 niveles (Colombia → Departamento → Municipio → ... → Mesa)
   - `dim_partidos` — los 348 partidos políticos con código, nombre y color
   - `dim_ambitos` — los 1,224 ámbitos geográficos con jerarquía padre resuelta
   - `dim_departamentos` / `dim_municipios` — vistas filtradas para departamentos (34) y municipios (1,189)
   - `dim_ambitos_citrep` — los 185 ámbitos de las Circunscripciones Transitorias Especiales de Paz

3. **Scraping de resultados por departamento (34 requests)**: Se descubre que la Registraduría expone resultados en endpoints paramétricos `/json/ACT/{SIGLA}/{AMBITO}.json`. Se consultan los 34 departamentos para obtener votos agregados por candidato y el partido ganador por municipio.

4. **Scraping paralelo de 1,189 municipios**: Para obtener votos de **cada candidato en cada municipio**, se consulta individualmente cada uno de los 1,189 municipios. Se usa `ThreadPoolExecutor` con 20 workers concurrentes y una `requests.Session` con connection pool, logrando ejecutar las 1,189 descargas en ~15 segundos (vs ~12 minutos en secuencial).

5. **Estructuración del DataFrame granular**: Se construye `cn_candidatos_municipio` (19,024 filas × 15 columnas), con votos de cada candidato en cada municipio, enriquecido con nombres de partidos desde las tablas de dimensión.

6. **Exportación en múltiples formatos**: El DataFrame se exporta en 5 formatos (.csv, .csv con punto y coma, .xlsx, .json, .parquet) al directorio `data/raw_data_extract/consultas_interpartidistas/`.

### Conceptos de Python aplicados

| Concepto | Uso en el notebook |
|----------|-------------------|
| `requests.get()` | Consumo de APIs REST (JSON) |
| `requests.Session` + `HTTPAdapter` | Connection pooling para reutilizar conexiones TCP |
| `concurrent.futures.ThreadPoolExecutor` | Paralelización de I/O-bound tasks (20 workers) |
| `threading.Lock` | Sincronización de un contador compartido entre threads |
| `pandas.DataFrame` | Estructuración tabular de datos |
| `pd.merge()` | Enriquecimiento por joins (lookup de nombres de partidos) |
| Comprensiones de listas y diccionarios | Transformación de JSON anidado a filas planas |
| `to_csv()`, `to_excel()`, `to_json()`, `to_parquet()` | Serialización en múltiples formatos |

---

## Notebook: Extracción de Cámara de Representantes

📓 `notebook/extract/data_scraping_camara.ipynb`

### ¿Qué hace?

Este notebook extrae, estructura y exporta los resultados de la elección de **Cámara de Representantes** de las elecciones legislativas de Colombia 2026 directamente desde la API JSON de la Registraduría.

### Diferencia clave con Consultas

La elección de Cámara tiene **3 circunscripciones** (cámaras) en cada response del API, a diferencia de Consultas que solo tenía una:

| `cam` | Circunscripción | Descripción |
|-------|----------------|-------------|
| `1` | Territorial Departamental | Candidatos específicos del departamento |
| `4` | Indígenas | Circunscripción especial indígena (nacional) |
| `5` | Afrodescendientes | Circunscripción especial afrodescendiente (nacional) |

Los endpoints municipales usan códigos de **7 dígitos** del nomenclator (`dim_ambitos.ambito_codigo` nivel 3), a diferencia de los departamentales que usan 4 dígitos. Esto permite obtener votos de **cada candidato en cada municipio**.

### Lógica de Python paso a paso

1. **Configuración y fetch del nomenclator**: Igual que el notebook de Consultas — se obtiene el `nomenclator.json` y se construyen las 7 tablas de dimensión.

2. **Exploración de la API de Cámara**: Se consultan endpoints a nivel nacional (`/json/ACT/CA/00.json`), departamental (`/json/ACT/CA/{depto_4dig}.json`) y municipal (`/json/ACT/CA/{mpio_7dig}.json`) para descubrir la estructura:
   - Cada response contiene un array `camaras[3]` con una entrada por circunscripción.
   - Cada cámara tiene `partotabla` (partidos → candidatos), `mapagan` (ganador por sub-ámbito) y `totales`.
   - Los 3 niveles devuelven datos de candidatos con las 3 circunscripciones.

3. **Scraping paralelo de 34 departamentos**: Se consultan los 34 departamentos con `ThreadPoolExecutor` (10 workers), obteniendo votos agregados por candidato y partido ganador por municipio.

4. **Scraping paralelo de 1,189 municipios**: Para obtener votos de **cada candidato en cada municipio**, se consulta cada uno de los 1,189 municipios (códigos de 7 dígitos de `dim_ambitos`). Se usa `ThreadPoolExecutor` con 20 workers concurrentes y `requests.Session` con connection pool, logrando ejecutar las 1,189 descargas en ~16 segundos.

5. **Construcción de DataFrames estructurados**: Se iteran las 3 circunscripciones de cada response municipal y se construyen 3 DataFrames:
   - `ca_candidatos_municipio` (327,104 filas) — Votos de cada candidato por partido, municipio y circunscripción (tabla más granular).
   - `ca_candidatos_depto` (8,776 filas) — Votos agregados por candidato a nivel departamento.
   - `ca_municipios_ganador` (3,567 filas) — Partido ganador por municipio y circunscripción.

6. **Exportación en múltiples formatos**: Los 3 DataFrames se exportan en 5 formatos (.csv, .csv con punto y coma, .xlsx, .json, .parquet) al directorio `data/raw_data_extract/camara/`.

---

## Notebook: Extracción de Senado de la República

📓 `notebook/extract/data_scraping_senado.ipynb`

### ¿Qué hace?

Este notebook extrae, estructura y exporta los resultados de la elección de **Senado de la República** de las elecciones legislativas de Colombia 2026 directamente desde la API JSON de la Registraduría.

### Diferencia clave con Cámara

La elección de Senado tiene **2 circunscripciones** (cámaras) en cada response del API, a diferencia de Cámara que tiene 3:

| `cam` | Circunscripción | Descripción |
|-------|----------------|-------------|
| `0` | Nacional | 100 curules — candidatos de todo el país |
| `4` | Indígenas | 2 curules — circunscripción especial indígena |

Al ser una elección **nacional**, los mismos candidatos aparecen en todos los departamentos y municipios (a diferencia de Cámara, donde los candidatos territoriales son específicos de cada departamento).

### Lógica de Python paso a paso

1. **Configuración y fetch del nomenclator**: Igual que los notebooks anteriores — se obtiene el `nomenclator.json` y se construyen las 7 tablas de dimensión.

2. **Exploración de la API de Senado**: Se consultan endpoints a nivel nacional (`/json/ACT/SE/00.json`), departamental (`/json/ACT/SE/{depto_4dig}.json`) y municipal (`/json/ACT/SE/{mpio_7dig}.json`) para descubrir la estructura:
   - Cada response contiene un array `camaras[2]` con una entrada por circunscripción.
   - Cada cámara tiene `partotabla` (partidos → candidatos), `mapagan` (ganador por sub-ámbito) y `totales`.
   - Los 3 niveles devuelven datos de candidatos con las 2 circunscripciones.

3. **Scraping paralelo de 34 departamentos**: Se consultan los 34 departamentos con `ThreadPoolExecutor` (10 workers), obteniendo votos agregados por candidato y partido ganador por municipio.

4. **Scraping paralelo de 1,189 municipios**: Se consulta cada uno de los 1,189 municipios (códigos de 7 dígitos de `dim_ambitos`). Se usa `ThreadPoolExecutor` con 20 workers concurrentes y `requests.Session` con connection pool, logrando ejecutar las 1,189 descargas en ~21 segundos.

5. **Construcción de DataFrames estructurados**: Se iteran las 2 circunscripciones de cada response municipal y se construyen 3 DataFrames:
   - `se_candidatos_municipio` (1,304,333 filas) — Votos de cada candidato por partido, municipio y circunscripción (tabla más granular).
   - `se_candidatos_depto` (37,298 filas) — Votos agregados por candidato a nivel departamento.
   - `se_municipios_ganador` (2,378 filas) — Partido ganador por municipio y circunscripción.

6. **Exportación en múltiples formatos**: Los 3 DataFrames se exportan en 5 formatos al directorio `data/raw_data_extract/senado/`. El XLSX del dataset municipal se omite por exceder el límite de 1,048,576 filas de Excel.

---

## Datos disponibles

### Consultas Interpartidistas

El DataFrame `cn_candidatos_municipio` contiene **19,024 filas** con los votos de cada candidato en cada municipio de Colombia para las Consultas Interpartidistas:

| Columna | Descripción |
|---------|-------------|
| `depto_nombre` | Departamento |
| `mpio_codigo` / `mpio_nombre` | Código y nombre del municipio |
| `partido_codigo` / `partido_nombre` | Partido político |
| `candidato_codigo` / `candidato_cedula` / `candidato_nombre` | Candidato |
| `candidato_votos` / `candidato_porcentaje` | Votos del candidato en el municipio |
| `votos_partido_mpio` | Total de votos del partido en el municipio |
| `mpio_votantes` / `mpio_votos_validos` / `mpio_votos_nulos` / `mpio_votos_no_marcados` | Totales del municipio |

### Cámara de Representantes

**`ca_candidatos_municipio`** — **327,104 filas**: votos de cada candidato por partido, municipio y circunscripción (tabla más granular).

| Columna | Descripción |
|---------|-------------|
| `depto_nombre` | Departamento |
| `mpio_codigo` / `mpio_nombre` | Código (7 dígitos) y nombre del municipio |
| `circunscripcion_id` / `circunscripcion` | Circunscripción (Territorial, Indígenas, Afrodescendientes) |
| `partido_codigo` / `partido_nombre` | Partido político |
| `votos_partido_mpio` | Votos totales del partido en el municipio |
| `candidato_codigo` / `candidato_cedula` / `candidato_nombre` | Candidato |
| `candidato_votos` / `candidato_porcentaje` | Votos del candidato en el municipio |
| `preferido` | Indicador de voto preferente |
| `mpio_votantes` / `mpio_votos_validos` / `mpio_votos_nulos` / `mpio_votos_no_marcados` | Totales del municipio |

**`ca_candidatos_depto`** — **8,776 filas**: votos agregados por candidato a nivel departamento y circunscripción.

**`ca_municipios_ganador`** — **3,567 filas**: partido ganador por municipio y circunscripción.

### Senado de la República

**`se_candidatos_municipio`** — **1,304,333 filas**: votos de cada candidato por partido, municipio y circunscripción (tabla más granular).

| Columna | Descripción |
|---------|-------------|
| `depto_nombre` | Departamento |
| `mpio_codigo` / `mpio_nombre` | Código (7 dígitos) y nombre del municipio |
| `circunscripcion_id` / `circunscripcion` | Circunscripción (Nacional, Indígenas) |
| `partido_codigo` / `partido_nombre` | Partido político |
| `votos_partido_mpio` | Votos totales del partido en el municipio |
| `candidato_codigo` / `candidato_cedula` / `candidato_nombre` | Candidato |
| `candidato_votos` / `candidato_porcentaje` | Votos del candidato en el municipio |
| `preferido` | Indicador de voto preferente |
| `mpio_votantes` / `mpio_votos_validos` / `mpio_votos_nulos` / `mpio_votos_no_marcados` | Totales del municipio |

**`se_candidatos_depto`** — **37,298 filas**: votos agregados por candidato a nivel departamento y circunscripción.

**`se_municipios_ganador`** — **2,378 filas**: partido ganador por municipio y circunscripción.

> **Nota**: El dataset `se_candidatos_municipio` no incluye formato XLSX porque supera el límite de 1,048,576 filas de Excel. Se recomienda usar Parquet o CSV.

Los datos están disponibles en los siguientes formatos para facilitar su uso en distintas herramientas de análisis:

- **CSV** (coma y punto y coma) — Excel, R, Python, cualquier herramienta
- **Excel (.xlsx)** — Análisis directo en hojas de cálculo
- **JSON** — Visualizaciones web, APIs
- **Parquet** — Alto rendimiento para pandas, PySpark, DuckDB

---

## Instalación

```bash
# Clonar el repositorio
git clone https://github.com/tu-usuario/scraping-elecciones-colombia-2026.git
cd scraping-elecciones-colombia-2026

# Crear y activar entorno virtual
python -m venv venv
source venv/bin/activate  # macOS/Linux

# Instalar dependencias
pip install -r requirements.txt
```

> **Nota**: Para ejecutar los notebooks de scraping se requiere una cookie `cf_clearance` válida del navegador (protección Cloudflare de la Registraduría). Actualiza el valor en `config.json` — todos los notebooks la leen desde ahí.

---

## Licencia

Ver [LICENSE](LICENSE).
