# Sistema de Recomendación Multimodal: Early Fusion Transformer

Este repositorio maestro contiene la implementación, optimización y puesta en producción de un modelo de recomendación de productos para canastas de compra. 

El núcleo del proyecto es una arquitectura **Early Fusion Transformer** que trasciende la simple predicción secuencial al integrar de forma temprana tres señales externas (multimodales) para cada artículo:
* **Score RNN:** Patrones secuenciales históricos.
* **Score TDA:** Relaciones estructurales y análisis de datos topológicos.
* **Score TS:** Predicciones temporales de demanda.

---

## Estructura Global del Proyecto

El repositorio está dividido en cuatro módulos principales. Cada directorio fue diseñado para operar de manera independiente y contiene su propio `README.md` con instrucciones detalladas, dependencias (`requirements.txt`) y scripts.

| Directorio | Propósito Principal | Archivos Clave |
| :--- | :--- | :--- |
| **Data** | Almacenamiento de diccionarios precalculados y mapeos necesarios para la red neuronal. | `transformer_vocab.pkl`, `catalogo.pkl`, `scores_dict.pkl` |
| **Model** | Preprocesamiento de transacciones, cruce de scores multimodales y entrenamiento base. | `Transformer_fusion.ipynb`, carpeta `docs/` |
| **Model Optimization** | Búsqueda de hiperparámetros con Optuna para maximizar la métrica MRR@10. | `Optuna_optimized_model.ipynb`, Base SQLite |
| **Recommendation Inference Function** | Script ligero y aislado para ejecutar predicciones en tiempo real sobre nuevas canastas. | `Inference Function.ipynb` |

---

## ⚠️ Confidencialidad y Requisitos de Datos

Por motivos de seguridad, confidencialidad empresarial y peso de almacenamiento, **las bases de datos transaccionales crudas no se incluyen en este repositorio**.

Para reproducir el procesamiento de datos o entrenar los modelos desde cero (en las carpetas `Model/` o `Model Optimization/`), es estrictamente necesario solicitar y colocar localmente los siguientes archivos en sus respectivos directorios:
* `pdvhdr.parquet`: Cabeceras de los tickets de venta.
* `pdvdet.parquet`: Detalle de artículos por ticket.
* `inviar.parquet` e `invars.parquet`: Tablas de inventario y catálogo físico.

> Si tu objetivo es únicamente probar la **inferencia** con el modelo ya optimizado, no necesitas los archivos `.parquet`. Solo requieres los archivos serializados `.pkl` (ubicados en `Data/`) y los pesos del modelo `.pt`.

---

## Flujo de Trabajo Recomendado

Si vas a explorar el proyecto completo, el orden lógico de ejecución es el siguiente:

1. **Entrenamiento Base (`Model/`):** Procesa los `.parquet`, genera la matriz de fusión unificada con los scores de RNN/TDA/TS (aplicando normalización Min-Max y relleno de ceros) y entrena la primera versión del Transformer.
2. **Refinamiento (`Model Optimization/`):** Toma las configuraciones base y utiliza Optuna para explorar el espacio de hiperparámetros. En el experimento actual, el modelo ganador (`optuna_trial_18`) logró un MRR@10 de validación de 0.1886.
3. **Producción (`Recommendation Inference Function/`):** Despliega el modelo ganador. Carga únicamente la arquitectura del Transformer, los pesos entrenados y los diccionarios `.pkl` para retornar el ranking de recomendaciones con una latencia mínima.