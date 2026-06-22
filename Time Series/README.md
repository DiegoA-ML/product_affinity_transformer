# ANA-03 — Series de Tiempo para Cross-selling / Afinidad de Canasta

Este repositorio contiene el componente de **Series de Tiempo** desarrollado para el caso **ANA-03 — Cross-selling (Afinidad de Canasta)** del proyecto TCA Merksyst.

El objetivo de este componente es generar una señal temporal por SKU (`score_TS`) que ayude a priorizar productos candidatos dentro de un sistema de recomendación de canastas. Esta señal no reemplaza a los modelos de afinidad, sino que los complementa respondiendo:

> De los productos que podrían complementar una canasta, ¿cuáles conviene recomendar ahora?

---

## 1. Estructura del repositorio

La estructura esperada del proyecto es:

```text
ANA03-Series-Tiempo/
│
├── README.md
├── requirements.txt
├── .gitignore
│
├── data/
│   ├── pdvhdr.parquet
│   ├── pdvdet.parquet
│   ├── inviar.parquet
│   └── invars.parquet
│
├── notebooks/
│   └── ANA03_series_tiempo_recomendador.ipynb
│
├── outputs_ana03/
│   ├── ts_scores_ana03.csv
│   ├── weekly_series_ana03.csv
│   ├── statistical_tests_by_sku.csv
│   ├── statistical_tests_summary.csv
│   ├── pie_estacionariedad_series.png
│   ├── pie_ciclicidad_series.png
│   └── bar_tipo_ciclo_series.png
│
└── docs/
    └── Documentacion_series_tiempo_ANA03.docx
```

---

## 2. Contenido de las carpetas

### `data/`

Contiene los archivos `.parquet` necesarios para ejecutar el notebook:

| Archivo | Descripción |
|---|---|
| `pdvhdr.parquet` | Encabezado de tickets de venta. |
| `pdvdet.parquet` | Detalle de productos vendidos por ticket. |
| `inviar.parquet` | Catálogo maestro de artículos. |
| `invars.parquet` | Existencias/inventario por producto. |

> Nota: estos archivos pueden contener información operativa sensible. Si el repositorio se sube a GitHub, se recomienda mantenerlo como **privado**.

### `notebooks/`

Contiene el notebook principal:

```text
notebooks/ANA03_series_tiempo_recomendador.ipynb
```

Este notebook ejecuta el flujo completo del componente temporal:

1. Carga de archivos parquet.
2. Limpieza de ventas válidas.
3. Construcción de series semanales por SKU.
4. Cálculo de métricas temporales.
5. Generación de `score_TS`.
6. Integración de inventario.
7. Pruebas estadísticas de estacionariedad y ciclicidad.
8. Exportación de resultados.

### `outputs_ana03/`

Contiene los resultados ya generados por el notebook.

| Archivo | Descripción |
|---|---|
| `weekly_series_ana03.csv` | Serie semanal por SKU. Una fila por producto-semana. |
| `ts_scores_ana03.csv` | Score temporal final por SKU. Una fila por producto. |
| `statistical_tests_by_sku.csv` | Resultados de pruebas estadísticas por SKU. |
| `statistical_tests_summary.csv` | Resumen porcentual de estacionariedad, ciclicidad y tipo de ciclo. |
| `pie_estacionariedad_series.png` | Gráfica de porcentaje de series estacionarias/no estacionarias. |
| `pie_ciclicidad_series.png` | Gráfica de porcentaje de series con/sin evidencia cíclica. |
| `bar_tipo_ciclo_series.png` | Gráfica de tipo de patrón cíclico dominante detectado. |

### `docs/`

Contiene documentación metodológica y explicación de resultados:

```text
docs/Documentacion_series_tiempo_ANA03.docx
```

---

## 3. Requisitos de instalación

Se recomienda usar Python 3.11 o una versión compatible.

Instalar dependencias con:

```bash
pip install -r requirements.txt
```

El archivo `requirements.txt` incluye:

```text
pandas
numpy
duckdb
pyarrow
matplotlib
statsmodels
scikit-learn
jupyter
ipykernel
```

---

## 4. Cómo ejecutar el notebook

### Opción recomendada: VS Code

1. Abrir VS Code.
2. Seleccionar **File > Open Folder**.
3. Abrir la carpeta raíz del proyecto:

```text
ANA03-Series-Tiempo/
```

4. Abrir el notebook:

```text
notebooks/ANA03_series_tiempo_recomendador.ipynb
```

5. Seleccionar el kernel de Python correspondiente.
6. Ejecutar las celdas en orden.

---

## 5. Configuración importante de rutas

El notebook debe poder encontrar la carpeta `data/`, aunque se ejecute desde la carpeta `notebooks/`.

Para evitar errores de ruta, se recomienda que la primera celda de rutas use esta lógica:

```python
from pathlib import Path
import duckdb
import pandas as pd
import numpy as np
import matplotlib.pyplot as plt

# Directorio actual desde donde corre el notebook
CURRENT_DIR = Path.cwd()

# Si el notebook se está ejecutando desde la carpeta notebooks,
# subimos un nivel para llegar a la raíz del proyecto.
if CURRENT_DIR.name == "notebooks":
    PROJECT_DIR = CURRENT_DIR.parent
else:
    PROJECT_DIR = CURRENT_DIR

# Carpeta donde están los archivos parquet
DATA_DIR = PROJECT_DIR / "data"

# Rutas de los parquet
PATH_PDVHDR = DATA_DIR / "pdvhdr.parquet"
PATH_PDVDET = DATA_DIR / "pdvdet.parquet"
PATH_INVIAR = DATA_DIR / "inviar.parquet"
PATH_INVARS = DATA_DIR / "invars.parquet"

# Carpeta para guardar salidas
OUTPUT_DIR = PROJECT_DIR / "outputs_ana03"
OUTPUT_DIR.mkdir(parents=True, exist_ok=True)

print("Directorio actual:", CURRENT_DIR)
print("Directorio raíz del proyecto:", PROJECT_DIR)
print("Directorio de datos:", DATA_DIR)
print("Directorio de salida:", OUTPUT_DIR)

print("\nValidación de archivos:")
for p in [PATH_PDVHDR, PATH_PDVDET, PATH_INVIAR, PATH_INVARS]:
    print(p.name, "->", p.exists(), "|", p)
```

Al ejecutar esta celda, todos los archivos deberían aparecer como `True`.

---

## 6. Limpieza de datos aplicada

La base limpia se construye a partir de la unión entre `pdvhdr` y `pdvdet` mediante la llave natural del ticket:

```text
Suc + FecOpe + Caja + NumTra
```

Se conservaron únicamente ventas válidas con los siguientes filtros:

| Filtro | Motivo |
|---|---|
| `pdvhdr.Tipo = '00'` | Mantener solo operaciones de venta. |
| `pdvhdr.status IN ('TI', 'TIF')` | Conservar tickets válidos, normales o facturados. |
| `pdvdet.status = '01'` | Mantener solo líneas activas del detalle. |
| `cantidad > 0` | Excluir devoluciones, cancelaciones o registros sin venta positiva. |
| `cve_art` no vacío | Excluir líneas sin producto identificable. |
| `TRIM()` en campos clave | Evitar errores por espacios en IDs o estatus. |
| Conversión de `FecOpe` a fecha | Permitir análisis temporal por semana. |

Después de la limpieza, la base contiene aproximadamente:

```text
3,989,985 líneas de venta limpias
688,322 transacciones únicas
7,792 SKUs únicos vendidos
```

Para el análisis de series de tiempo se conservaron:

```text
5,757 SKUs con historial suficiente
97 semanas de observación
558,429 filas producto-semana
```

---

## 7. Archivo `weekly_series_ana03.csv`

Este archivo es la base histórica semanal.

Tiene una fila por:

```text
SKU + semana
```

Columnas principales:

| Columna | Descripción |
|---|---|
| `cve_art` | ID del producto o SKU. |
| `week_start` | Inicio de la semana analizada. |
| `qty` | Cantidad vendida en la semana. |
| `sales` | Venta monetaria semanal. |
| `tickets` | Número de tickets donde apareció el SKU. |
| `lines` | Número de líneas de venta. |
| `sucursales_con_venta` | Presencia/conteo de sucursales con venta. |
| `des_art` | Descripción del producto. |
| `lin`, `s_lin`, `fam`, `s_fam`, `marca` | Clasificación comercial del producto. |
| `status_articulo` | Estatus del producto en catálogo. |

Este archivo sirve para:

- Graficar series de productos específicos.
- Analizar crecimiento, caídas, intermitencia o estacionalidad.
- Validar visualmente el comportamiento temporal de SKUs.
- Servir como insumo para análisis o modelos posteriores.

---

## 8. Archivo `ts_scores_ana03.csv`

Este archivo resume cada serie semanal en una fila por SKU.

Su columna principal es:

```text
score_TS
```

`score_TS` resume la relevancia temporal de un producto usando cuatro señales:

| Componente | Peso | Interpretación |
|---|---:|---|
| `trend_score` | 40% | Premia productos con crecimiento reciente. |
| `recency_score` | 25% | Premia productos vendidos recientemente. |
| `volume_score` | 20% | Premia productos con volumen suficiente. |
| `seasonality_score` | 15% | Premia productos relevantes para el periodo actual. |

Fórmula:

```text
score_TS = 0.40*trend_score
         + 0.25*recency_score
         + 0.20*volume_score
         + 0.15*seasonality_score
```

Este archivo se usa para integrar la señal temporal al recomendador final.

---

## 9. Pruebas estadísticas

El notebook agrega una sección de pruebas estadísticas para caracterizar las series temporales por SKU.

### 9.1 Estacionariedad

Se utiliza la prueba **Augmented Dickey-Fuller (ADF)**.

La interpretación general es:

```text
p-value < 0.05  -> serie estacionaria
p-value >= 0.05 -> serie no estacionaria
```

Resultados obtenidos:

```text
63.2% Estacionaria
24.0% No estacionaria
12.9% Sin datos suficientes
```

Una serie estacionaria indica que el comportamiento del SKU tiende a mantenerse relativamente estable en el tiempo.

### 9.2 Ciclicidad

Se calculan autocorrelaciones en rezagos semanales para detectar evidencia de patrones repetitivos:

| Rezago | Interpretación |
|---:|---|
| 4 semanas | Patrón mensual aproximado. |
| 13 semanas | Patrón trimestral aproximado. |
| 26 semanas | Patrón semestral aproximado. |
| 52 semanas | Patrón anual aproximado. |

Resultados obtenidos:

```text
40.8% Con evidencia cíclica
46.4% Sin evidencia cíclica
12.9% Sin datos suficientes
```

El tipo de ciclo dominante detectado se resume en la gráfica:

```text
bar_tipo_ciclo_series.png
```

La mayor parte de los patrones cíclicos detectados se concentran en ciclos mensuales aproximados, seguidos por patrones anuales aproximados.

---

## 10. Integración con el recomendador de canastas

El componente de series de tiempo no recomienda productos de forma aislada.

Su función es tomar candidatos generados por otros módulos, por ejemplo:

- TDA / co-ocurrencia.
- RNN.
- Transformer.

Y agregar una señal temporal:

```text
score_TS
```

La fusión conceptual del sistema completo puede representarse así:

```text
score_final = alpha*score_TDA
            + beta*score_TS
            + gamma*score_RNN
            + delta*score_Transformer
```

Ejemplo de integración:

```python
candidatos_con_ts = candidatos.merge(
    ts_scores[["cve_art", "score_TS", "has_stock"]],
    on="cve_art",
    how="left"
)
```

Antes de mostrar recomendaciones al vendedor, se recomienda filtrar:

```text
has_stock = True
```

---

## 11. Resultados principales

El módulo genera resultados útiles para el negocio:

- Identifica productos con alta relevancia temporal.
- Prioriza productos activos recientemente.
- Reduce recomendaciones de productos sin movimiento actual.
- Detecta productos con patrones estacionales o cíclicos.
- Agrega un filtro de inventario para evitar recomendaciones inviables.
- Permite actualizar periódicamente el ranking temporal de SKUs.

---

## 12. Consideraciones importantes

| Consideración | Explicación |
|---|---|
| `cve_art` como texto | Algunos SKUs tienen ceros a la izquierda. Leerlos como número puede romper cruces. |
| `score_TS` no es recomendación autónoma | Un score alto indica relevancia temporal, pero no necesariamente afinidad con una canasta. |
| Productos de temporada | Pueden subir en `score_TS`; esto es deseable, pero debe interpretarse según la fecha de corte. |
| Inventario | `has_stock` debe usarse como filtro operacional antes de mostrar recomendaciones. |
| Reentrenamiento | El score debe recalcularse periódicamente para mantener actualidad. |
| Confidencialidad | Los datos reales del ERP deben manejarse en repositorios privados o entornos controlados. |

---

## 13. Notas para subir a GitHub

El repositorio completo pesa aproximadamente 90 MB comprimido y 161 MB descomprimido.

Ningún archivo individual supera los 100 MB, por lo que puede subirse a GitHub sin Git LFS. Sin embargo, dado que los archivos `.parquet` contienen datos reales del proyecto, se recomienda:

1. Crear el repositorio como **privado**.
2. No compartir el enlace públicamente.
3. Confirmar con el equipo/profesor si los datos completos pueden versionarse.
4. Si se requiere un repositorio público, reemplazar `data/` por una muestra anonimizada o eliminar la carpeta `data/`.

---

## 14. Autoría

Proyecto desarrollado como parte del Reto TCA Merksyst — Tecnológico de Monterrey.

Componente documentado:

```text
ANA-03 | Series de Tiempo para Afinidad de Canasta
```
