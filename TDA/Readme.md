# ANA-03 — Topological Data Analysis (TDA Mapper) para Afinidad de Canasta

Este repositorio contiene el componente de **Topological Data Analysis (TDA)** desarrollado para el caso **ANA-03 — Cross-selling (Afinidad de Canasta)** del proyecto TCA Merksyst.

El objetivo de este componente es identificar relaciones estructurales entre productos utilizando técnicas de análisis topológico de datos. A diferencia de los métodos tradicionales de Market Basket Analysis, TDA permite descubrir agrupaciones, conexiones y patrones complejos entre artículos mediante la construcción de un grafo Mapper.

La salida principal del módulo es un conjunto de **scores de afinidad topológica entre productos (`score_TDA`)**, que posteriormente pueden integrarse dentro del sistema global de recomendación.

---

# 1. Estructura del repositorio

La estructura esperada del proyecto es:

```text
ANA03-TDA/
│
├── README.md
├── .gitignore
│
├── notebook
│   ├── TDA_Mapper_Documentado.ipynb
│   ├── pdvhdr.parquet
│   ├── pdvdet.parquet
│   ├── inviar.parquet
│   └── invars.parquet
│
└── outputs_tda/
    ├── mapper_dbscan/
    └── mapper_kmeans/


```

---

# 2. Contenido de las carpetas

## `data/`

Contiene los archivos de entrada requeridos para construir el modelo topológico.

| Archivo          | Descripción                                              |
| ---------------- | -------------------------------------------------------- |
| `pdvhdr.parquet` | Encabezado de tickets de venta.                          |
| `pdvdet.parquet` | Detalle de artículos vendidos por ticket.                |
| `inviar.parquet` | Catálogo maestro de artículos.                           |
| `invars.parquet` | Variables complementarias de clasificación de productos. |

> Nota: estos archivos contienen información operativa del negocio y se recomienda mantener el repositorio privado.

---

## `notebooks/`

Contiene el notebook principal:

```text
notebooks/TDA_Mapper_Documentado.ipynb
```

Este notebook ejecuta el flujo completo del módulo TDA:

1. Carga de archivos parquet.
2. Integración y limpieza de datos.
3. Construcción del dataframe de productos.
4. Codificación de variables categóricas.
5. Reducción de dimensionalidad con UMAP e ISOMAP.
6. Construcción del grafo Mapper mediante KeplerMapper.
7. Clusterización con DBSCAN.
8. Clusterización alternativa con KMeans.
9. Evaluación de K mediante Silhouette Score.
10. Evaluación mediante Elbow Method.
11. Construcción del inverse graph.
12. Cálculo de scores topológicos entre productos.
13. Exportación de resultados.

---

## `outputs_tda/`

Contiene todos los resultados generados por el notebook.

| Archivo                 | Descripción                                                     |
| ----------------------- | --------------------------------------------------------------- |
| `tda_scores.csv`        | Score de afinidad topológica por artículo.                      |
| `mapper_dbscan/`        | Visualizaciones generadas usando DBSCAN.                        |
| `mapper_kmeans/`        | Visualizaciones generadas usando KMeans.                        |

---

## `docs/`

Contiene documentación técnica detallada:

```text
docs/DOCUMENTACION_TDA.md
```

Incluye explicación de algoritmos, parámetros, estructuras de datos y metodología de cálculo de scores.

---

# 3. Requisitos de instalación

Se recomienda utilizar Python 3.10 o superior.

Instalar dependencias:

```bash
pip install -r requirements.txt
```

Dependencias principales:

```text
pandas
numpy
scikit-learn
umap-learn
kmapper
networkx
matplotlib
plotly
pyarrow
```

---

# 4. Cómo ejecutar el notebook

## Opción recomendada: Google Colab

1. Subir los archivos parquet a Google Drive.
2. Abrir el notebook en Colab.
3. Configurar las rutas de carga.
4. Ejecutar las celdas en orden.

Para mejorar el rendimiento se recomienda activar GPU:

```text
Runtime → Change runtime type → GPU
```

---

## Opción alternativa: VS Code

1. Abrir la carpeta raíz del proyecto.
2. Abrir:

```text
notebooks/TDA_Mapper_Documentado.ipynb
```

3. Seleccionar el kernel de Python.
4. Ejecutar todas las celdas secuencialmente.

---

# 5. Construcción de la base analítica

La información transaccional se construye uniendo:

```text
pdvhdr + pdvdet + inviar + invars
```

La llave principal de integración es:

```text
Suc + FecOpe + Caja + NumTra
```

Posteriormente se genera un dataframe consolidado que contiene:

* Información de artículos.
* Clasificaciones comerciales.
* Variables de línea.
* Variables de familia.
* Variables de subfamilia.

---

# 6. Selección de características

Las características utilizadas para representar productos son:

| Variable  | Descripción         |
| --------- | ------------------- |
| `cve_art` | Código del artículo |
| `des_art` | Descripción         |
| `lin`     | Línea comercial     |
| `fam`     | Familia             |
| `s_fam`   | Subfamilia          |

Estas variables son transformadas mediante Label Encoding para permitir su utilización dentro de los algoritmos matemáticos posteriores.

---

# 7. Reducción de dimensionalidad

Antes de construir el Mapper se reduce la dimensionalidad utilizando dos técnicas complementarias:

## UMAP

Permite:

* Preservar estructura local.
* Mantener relaciones de vecindad.
* Generar una representación compacta.

## ISOMAP

Permite:

* Preservar distancias geodésicas.
* Capturar estructura global.
* Facilitar la construcción topológica.

Parámetros principales utilizados:

| Parámetro         | Valor |
| ----------------- | ----- |
| UMAP Components   | 2     |
| ISOMAP Components | 3     |
| ISOMAP Neighbors  | 20    |

---

# 8. Construcción del grafo Mapper

La representación topológica se construye utilizando:

```text
KeplerMapper
```

El proceso consiste en:

```text
Datos
   ↓
UMAP + ISOMAP
   ↓
Covering (cubos)
   ↓
Clusterización
   ↓
Mapper Graph
```

Cada nodo representa un subconjunto de productos similares.

Las conexiones entre nodos representan regiones compartidas dentro del espacio reducido.

---

# 9. Clusterización con DBSCAN

DBSCAN se utiliza como primera estrategia de agrupamiento.

Ventajas:

* No requiere definir número de clusters.
* Detecta formas arbitrarias.
* Tolera ruido.
* Identifica regiones densas.

El resultado se almacena dentro de:

```text
Mapper_Visualizations/
```

---

# 10. Clusterización con KMeans

Como alternativa se evalúa KMeans sobre la representación reducida.

Para determinar el mejor valor de K se utilizan dos criterios:

## Silhouette Score

Evalúa:

```text
Separación entre clusters
```

y

```text
Cohesión interna
```

Los valores más prometedores encontrados fueron:

```text
K = 10, 11 y 12
```

---

## Elbow Method

Analiza:

```text
Número de clusters vs Inercia
```

La gráfica sugiere:

```text
K = 3 a 5
```

Debido a la diferencia entre ambos métodos se exploran los valores:

```text
K = 3, 4, 11 y 12
```

para construir distintas variantes del Mapper.

---

# 11. Construcción del Inverse Graph

Una vez generado el Mapper se construye una estructura auxiliar denominada:

```text
inverse_graph
```

Esta estructura permite:

* Encontrar rápidamente los nodos que contienen un producto.
* Reducir tiempos de búsqueda.
* Optimizar cálculos de conectividad.

Conceptualmente:

```text
Producto
    ↓
Lista de nodos donde aparece
```

---

# 12. Cálculo de scores topológicos

El objetivo principal del módulo es calcular afinidad entre productos.

Para ello se utilizan:

* Distancias mínimas dentro del grafo.
* Número de conexiones.
* Relación entre nodos compartidos.
* Alcance topológico dentro de la red Mapper.

Las distancias se calculan utilizando:

```text
Breadth First Search (BFS)
```

sobre el grafo generado.

La intuición es:

> Dos productos que aparecen en regiones cercanas del Mapper poseen mayor afinidad topológica.

---

# 13. Archivo `tda_scores.csv`

El resultado final contiene una fila por producto.

Columnas sugeridas:

| Columna              | Descripción                      |
| -------------------- | -------------------------------- |
| `cve_art`            | SKU                              |
| `des_art`            | Descripción                      |
| `score_TDA`          | Afinidad topológica normalizada  |
| `distancia_promedio` | Distancia media dentro del grafo |
| `nodos_compartidos`  | Número de nodos compartidos      |
| `grado_conectividad` | Nivel de conectividad            |
| `lin`                | Línea comercial                  |
| `fam`                | Familia                          |
| `s_fam`              | Subfamilia                       |

---

# 14. Integración con el recomendador

El score generado por TDA se utiliza como una señal adicional dentro del sistema de recomendación.


El componente TDA aporta:

* Relaciones estructurales.
* Afinidad topológica.
* Conexiones no evidentes.
* Complementariedad entre productos.

---

# 15. Beneficios del enfoque TDA

El enfoque Mapper ofrece ventajas frente a métodos tradicionales:

* Captura relaciones complejas entre productos.
* Identifica estructuras no lineales.
* Descubre comunidades ocultas.
* Reduce dependencia de reglas Apriori.
* Facilita visualización de la topología comercial.

---

# 16. Consideraciones importantes

| Consideración     | Explicación                                                                       |
| ----------------- | --------------------------------------------------------------------------------- |
| Calidad de datos  | Los resultados dependen de la correcta clasificación de productos.                |
| Parámetros Mapper | Cubos, overlap y clusterización afectan la topología generada.                    |
| Escalabilidad     | Mapper puede incrementar su costo computacional con catálogos muy grandes.        |
| Reentrenamiento   | Se recomienda reconstruir periódicamente el grafo ante cambios del catálogo.      |
| Interpretabilidad | Los nodos representan regiones de similitud, no categorías de negocio explícitas. |

---

# 17. Autoría

Proyecto desarrollado como parte del Reto TCA Merksyst — Tecnológico de Monterrey.

Componente documentado:

```text
ANA-03 | Topological Data Analysis (TDA Mapper) para Afinidad de Canasta
```
