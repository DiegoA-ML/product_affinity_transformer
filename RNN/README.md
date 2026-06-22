# ANA-03 — Red Neuronal Recurrente (RNN/GRU) para Cross-selling / Afinidad de Canasta

Este repositorio contiene el componente de **Red Neuronal Recurrente** desarrollado para el caso **ANA-03 — Cross-selling (Afinidad de Canasta)** del proyecto TCA Merksyst.

El objetivo de este componente es generar una señal de secuencia por SKU (`score_RNN`) que ayude a priorizar productos candidatos dentro de un sistema de recomendación de canastas. Esta señal no reemplaza a los modelos de afinidad, sino que los complementa respondiendo:

> De los productos que podrían complementar una canasta, ¿cuál es el más probable que el cliente compre a continuación dado lo que ya tiene en su ticket?

---

## 1. Estructura del repositorio

La estructura esperada del proyecto es:

```text
ANA03-RNN/
│
├── README.md
├── requirements.txt
├── .gitignore
│
├── data/
│   ├── pdvhdr.parquet
│   ├── pdvdet.parquet
│   └── inviar.parquet
│
├── notebooks/
│   └── RNN_LSTM_BasketRecommender.ipynb
│
├── outputs_rnn/
│   ├── rnn_scores.csv
│   ├── distribucion_canastas.png
│   ├── curvas_entrenamiento_gru.png
│   ├── embeddings_pca_gru.png
│   └── distribucion_score_rnn.png
│
└── docs/
    └── DOCUMENTACION_RNN.md
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

> Nota: estos archivos pueden contener información operativa sensible. Si el repositorio se sube a GitHub, se recomienda mantenerlo como **privado**.

### `notebooks/`

Contiene el notebook principal:

```text
notebooks/RNN_LSTM_BasketRecommender.ipynb
```

Este notebook ejecuta el flujo completo del componente de secuencias:

1. Carga de archivos parquet.
2. Filtrado de ventas válidas y construcción de canastas.
3. Construcción del vocabulario de artículos (`TOP_K_VOCAB`).
4. Generación de secuencias de entrenamiento (sliding window).
5. Definición de la arquitectura GRU.
6. Entrenamiento del modelo con early stopping.
7. Evaluación con métricas Hit@K y MRR.
8. Generación del `score_RNN` por artículo.
9. Exportación de resultados.

### `outputs_rnn/`

Contiene todos los resultados generados por el notebook.

| Archivo | Descripción |
|---|---|
| `rnn_scores.csv` | Score RNN final por SKU. Una fila por artículo. **Input del Transformer.** |
| `distribucion_canastas.png` | Histograma de tamaño de canastas y top-20 artículos más frecuentes. |
| `curvas_entrenamiento_gru.png` | Loss, Hit@K, MRR y Accuracy a lo largo de las épocas de entrenamiento. |
| `embeddings_pca_gru.png` | PCA 2D de los embeddings aprendidos, coloreado por categoría de producto. |
| `distribucion_score_rnn.png` | Distribución del score_RNN, scatter Hit Rate vs Score y top-20 por score. |

### `docs/`

Contiene la documentación técnica detallada del módulo:

```text
docs/DOCUMENTACION_RNN.md
```

Incluye descripción de todas las clases y funciones, variables configurables, decisiones de diseño, descripción completa de columnas del `rnn_scores.csv` y guía de solución de problemas.

---

## 3. Requisitos de instalación

Se recomienda usar Python 3.10 o una versión compatible.

Instalar dependencias con:

```bash
pip install -r requirements.txt
```

El archivo `requirements.txt` incluye todas las dependencias necesarias para ejecutar el notebook localmente o en Google Colab.

---

## 4. Cómo ejecutar el notebook

### Opción recomendada: Google Colab

1. Subir los tres archivos `.parquet` a Google Drive.
2. Abrir el notebook en Colab.
3. Ajustar la variable `BASE_PATH` en la celda de carga de datos:

```python
BASE_PATH = Path("/content/drive/MyDrive/tu-carpeta/")
```

4. Activar GPU para reducir el tiempo de entrenamiento:

```text
Runtime → Change runtime type → GPU
```

5. Ejecutar las celdas en orden.

**Tiempos estimados:**

| Hardware | Tiempo por época | Total (10 épocas) |
|---|---|---|
| CPU Colab | ~70 segundos | ~12 minutos |
| GPU Colab | ~6 segundos | ~1 minuto |

### Opción alternativa: VS Code

1. Abrir VS Code.
2. Seleccionar **File > Open Folder**.
3. Abrir la carpeta raíz del proyecto:

```text
ANA03-RNN/
```

4. Abrir el notebook:

```text
notebooks/RNN_LSTM_BasketRecommender.ipynb
```

5. Seleccionar el kernel de Python correspondiente.
6. Ejecutar las celdas en orden.

---

## 5. Configuración importante de rutas

El notebook detecta automáticamente si se ejecuta desde Google Colab o localmente. La única variable que debe ajustarse es `BASE_PATH` en la celda de carga de datos:

```python
from pathlib import Path

# Google Colab — ajusta a tu ruta en Drive
BASE_PATH = Path("/content/drive/MyDrive/tu-carpeta/")

# Local — ajusta a la carpeta data/ del proyecto
# BASE_PATH = Path("../data/")
```

Al ejecutar la celda de carga, los tres archivos deben cargarse sin error:

```text
pdvhdr: 688,175 filas
pdvdet: 3,989,985 filas
inviar: 30,035 filas
```

---

## 6. Limpieza de datos aplicada

La base limpia se construye a partir de la unión entre `pdvhdr` y `pdvdet` mediante la llave natural del ticket:

```text
Suc + FecOpe + Caja + NumTra
```

Se conservaron únicamente ventas válidas con los siguientes filtros:

| Filtro | Motivo |
|---|---|
| `pdvhdr.status = 'TI'` | Mantener solo tickets de ingreso / venta válida. |
| `pdvdet.status = '01'` | Mantener solo líneas activas del detalle. |
| `unidades_netas > 0` | Excluir devoluciones (`unidades_netas = cantidad - devueltas`). |
| `cve_art` no vacío | Excluir líneas sin producto identificable. |
| Canastas con ≥ 3 artículos | Garantizar contexto mínimo para el sliding window de entrenamiento. |

Después de la limpieza y filtrado, la base de entrenamiento contiene aproximadamente:

```text
437,488 canastas útiles (≥3 artículos)
Media de 8.3 artículos por ticket
```

---

## 7. Vocabulario de artículos

El modelo no trabaja con los `cve_art` directamente, sino con índices enteros. El vocabulario se construye seleccionando los **K artículos más frecuentes** del dataset:

| Parámetro | Valor actual | Descripción |
|---|---|---|
| `TOP_K_VOCAB` | 3,000 | Número de artículos incluidos en el vocabulario del modelo |
| `<PAD>` (índice 0) | — | Token de relleno para secuencias cortas en un batch |
| `<UNK>` (índice 1) | — | Token para artículos fuera del top-K |

Con `TOP_K_VOCAB=3000`, el vocabulario cubre aproximadamente el 80-90% de todas las ocurrencias en los tickets.

Los diccionarios `art2idx` y `idx2art` se guardan dentro de `basket_gru_model.pt` para poder usarlos en inferencia sin necesidad de volver a construirlos.

---

## 8. Archivo `rnn_scores.csv`

Este archivo resume la señal de secuencia del modelo en una fila por SKU.

Su columna principal es:

```text
score_RNN
```

`score_RNN` resume la relevancia secuencial de un producto usando tres señales calculadas sobre el conjunto de validación:

| Componente | Peso | Interpretación |
|---|---:|---|
| `prob_norm` | 40% | Probabilidad softmax promedio que el modelo asignó al artículo cuando era el target. |
| `hitrate_norm` | 35% | Fracción de veces que el modelo acertó exactamente el artículo (Top-1). |
| `top5_norm` | 25% | Fracción de veces que el artículo apareció en el Top-5 del ranking cuando era el target. |

Fórmula:

```text
score_RNN = 0.40 * prob_norm
          + 0.35 * hitrate_norm
          + 0.25 * top5_norm
```

Columnas completas del archivo:

| Columna | Descripción |
|---|---|
| `cve_art` | ID del producto o SKU. |
| `des_art` | Descripción del artículo. |
| `score_RNN` | Score principal normalizado [0, 1]. |
| `prob_norm` | Probabilidad softmax promedio normalizada. |
| `hitrate_norm` | Hit rate Top-1 normalizado. |
| `top5_norm` | Tasa Top-5 normalizada. |
| `prob_raw` | Probabilidad softmax promedio sin normalizar. |
| `freq_en_canastas` | Número de tickets donde aparece el artículo. |
| `freq_como_siguiente` | Veces que fue el target en el conjunto de entrenamiento. |
| `hit_rate_como_target` | Fracción de aciertos exactos (sin normalizar). |
| `top5_rate` | Fracción de apariciones en Top-5 (sin normalizar). |
| `top10_rate` | Fracción de apariciones en Top-10 (sin normalizar). |
| `lin`, `fam`, `linea_dsc` | Clasificación comercial del producto. |
| `in_vocab` | `True` si el artículo está en el vocabulario del modelo. |

Los artículos con `in_vocab=False` tienen `score_RNN=0` y pueden ser rankeados por los otros módulos del pipeline.

---

## 9. Arquitectura del modelo GRU

El modelo es una Red Neuronal Recurrente tipo **GRU (Gated Recurrent Unit)**. Se eligió GRU sobre LSTM porque ofrece la misma calidad con 25% menos parámetros y aproximadamente 30% más velocidad de entrenamiento.

```text
Canasta parcial [art_1, art_2, ..., art_n]
        ↓
Embedding Layer  (vocab_size → 64 dims)
        ↓
pack_padded_sequence  ← ignora los tokens PAD, acelera 2-3x
        ↓
GRU Layer  (hidden=128, 1 capa)
        ↓
Último estado oculto  (128 dims)
        ↓
Dropout + BatchNorm1d
        ↓
Linear  (128 → vocab_size)
        ↓
Softmax  →  P(artículo_j | canasta_actual)
```

Parámetros del modelo con la configuración actual:

| Parámetro | Valor |
|---|---|
| `vocab_size` | 3,002 (3,000 + PAD + UNK) |
| `embed_dim` | 64 |
| `hidden_dim` | 128 |
| `num_layers` | 1 |
| `dropout` | 0.3 |
| Total parámetros entrenables | ~590,000 |

---

## 10. Entrenamiento

| Componente | Configuración | Justificación |
|---|---|---|
| **Loss** | `CrossEntropyLoss(label_smoothing=0.05)` | Regularización suave sobre las 3,000 clases. |
| **Optimizador** | `AdamW(lr=1e-3, weight_decay=1e-4)` | Mejor generalización que Adam estándar. |
| **Scheduler** | `OneCycleLR(max_lr=1e-3, pct_start=0.3)` | Convergencia rápida en pocas épocas. |
| **Grad clipping** | `max_norm=1.0` | Previene explosión de gradientes en la RNN. |
| **Early stopping** | `PATIENCE=3` | Detiene si `val_loss` no mejora en 3 épocas. |
| **Batch size** | 512 | Gradientes más estables, mejor uso de GPU/CPU. |
| **Épocas máximas** | 10 | Suficiente con OneCycleLR; la curva converge antes. |

---

## 11. Métricas de evaluación

| Métrica | Descripción |
|---|---|
| **Train Acc** | Porcentaje de veces que el artículo predicho en Top-1 coincide exactamente con el artículo real. |
| **Hit@1** | Fracción de canastas donde el artículo real es la primera recomendación. |
| **Hit@5** | Fracción de canastas donde el artículo real aparece entre las 5 primeras recomendaciones. |
| **Hit@10** | Fracción de canastas donde el artículo real aparece entre las 10 primeras recomendaciones. |
| **MRR** | Mean Reciprocal Rank — posición promedio inversa del artículo correcto en el ranking. |
| **Val Loss** | CrossEntropy sobre el conjunto de validación — indica si hay overfitting. |

---

## 12. Resultados obtenidos

Corrida con `TOP_K_VOCAB=3000`, `NUM_EPOCHS=10`, CPU Google Colab:

| Métrica | Valor obtenido |
|---|---|
| Train Acc | 11.65% |
| Hit@1 | 12.39% |
| Hit@5 | 24.26% |
| Hit@10 | 31.20% |
| MRR | 0.1755 |
| Val Loss | 6.0013 |

Comparativa con baselines:

| Modelo | Hit@10 |
|---|---|
| Aleatorio puro | 0.33% |
| Frecuencia marginal (siempre recomendar los más vendidos) | ~10% |
| Apriori / Market Basket Analysis | ~18% |
| **GRU — este módulo** | **31.20%** |

> El GRU supera al baseline clásico en aproximadamente 3×. Train Loss y Val Loss se mantienen muy cercanos en todas las épocas (6.05 vs 6.00), lo que indica que el modelo generaliza correctamente sin overfitting.

---

## 13. Integración con el recomendador de canastas

El componente de RNN no recomienda productos de forma aislada.

Su función es tomar candidatos generados por otros módulos y agregar una señal de probabilidad secuencial:

```text
score_RNN
```

La fusión conceptual del sistema completo se representa así:

```text
score_final = 0.25 * score_TDA
            + 0.20 * score_TS
            + 0.25 * score_RNN
            + 0.30 * score_Transformer
```

Ejemplo de integración en el módulo Transformer:

```python
candidatos_con_rnn = candidatos.merge(
    rnn_scores[["cve_art", "score_RNN"]],
    on="cve_art",
    how="left"
).fillna({"score_RNN": 0.0})
```

---

## 14. Resultados principales

El módulo genera resultados útiles para el negocio:

- Aprende qué artículos suelen seguir a una combinación específica de productos en el ticket.
- Genera un ranking de recomendaciones contextuales por canasta parcial.
- Produce el `score_RNN` como señal combinable con TDA, Series de Tiempo y Transformer.
- Detecta relaciones secuenciales no evidentes entre SKUs que Apriori no puede capturar.
- Permite inferencia en tiempo real dado cualquier canasta parcial nueva.

---

## 15. Consideraciones importantes

| Consideración | Explicación |
|---|---|
| `cve_art` como texto | Algunos SKUs tienen ceros a la izquierda. Leerlos como número puede romper los cruces. |
| `score_RNN` no es recomendación autónoma | Un score alto indica probabilidad secuencial, pero no necesariamente afinidad temporal ni disponibilidad en inventario. |
| `in_vocab=False` | Artículos fuera del top-3000 tienen `score_RNN=0`. Deben rankearse con los otros módulos. |
| Orden de artículos en la canasta | El modelo asume que el orden en que se registran los artículos tiene información; en la práctica depende del sistema POS. |
| Reentrenamiento | El modelo debe reentrenarse periódicamente para capturar cambios en el catálogo o en los patrones de compra. |
| Confidencialidad | Los datos reales del ERP deben manejarse en repositorios privados o entornos controlados. |

---

## 16. Notas para subir a GitHub

Dado que los archivos `.parquet` contienen datos reales del proyecto, se recomienda:

1. Crear el repositorio como **privado**.
2. No compartir el enlace públicamente.
3. Confirmar con el equipo/profesor si los datos completos pueden versionarse.
4. Si se requiere un repositorio público, reemplazar `data/` por una muestra anonimizada o eliminar la carpeta `data/`.

---

## 17. Autoría

Proyecto desarrollado como parte del Reto TCA Merksyst — Tecnológico de Monterrey.

Componente documentado:

```text
ANA-03 | Red Neuronal Recurrente (RNN/GRU) para Afinidad de Canasta
```
