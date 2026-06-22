# This is a public demonstration of the private repository I managed with my teammates for our final bachelor project in the Monterrey Institute of Technology and Higher Education. You may however still find the reports in the Reports folder with all findings, but these will be in spanish. I can always chat about the project in english with those interested! 

# Basket Recommender — ANA-03 Cross-selling (Afinidad de Canasta)

Sistema de recomendación de productos complementarios para el caso de uso **ANA-03 Cross-selling** sobre la plataforma **TCA Merksyst**. Dada una canasta parcial de compra (de 1 a 10 productos), el sistema devuelve el **Top-10 de productos más probables a agregar**, ordenados.

El núcleo es un **Early Fusion Transformer** que combina, desde la entrada, el *embedding* de identidad de cada producto con tres señales conductuales precalculadas por SKU:

- **Score RNN** — patrones secuenciales históricos de canastas.
- **Score TDA** — relaciones estructurales del catálogo (análisis topológico de datos).
- **Score TS** — oportunidad temporal de demanda (series de tiempo + inventario).

El modelo se sirve como una **API REST contenedorizada** (FastAPI + Docker) desplegada en **Azure Container Apps**, y se consume desde la aplicación web **Canasta**.

---

## Autores

- Diego Armando Mijares Ledezma (A01722421)
- Daniel Eduardo Arana Bodart (A01741202)
- Mauricio Octavio Valencia Gonzalez (A01234397)
- David Alejandro Acuña Orozco (A00571187)
- Diego Gutiérrez Vargas (A01285421)
- Alfonso Elizondo Partida (A01285151)

> Reto · Tecnológico de Monterrey · Ingeniería en Ciencias de Datos y Matemáticas · Socio formador: **TCA Software Solutions**

---

## API en vivo

> ⚠️ Desplegada sobre una suscripción de Azure for Students — la URL en vivo puede desactivarse después de la evaluación. Todo lo necesario para reconstruirla y redesplegarla está en este repositorio.

URL base:

```
https://reco-basket-app.gentlesea-cb2510e3.eastus.azurecontainerapps.io/view
```

| Superficie | Ruta | Propósito |
|---|---|---|
| Demo visual | `/view` | Tabla de recomendaciones en el navegador (sin código) |
| Swagger UI | `/docs` | Probador interactivo + documentación autogenerada |
| Endpoint | `POST /recommend` | Punto de integración que consumen las aplicaciones |
| Health check | `/` | Devuelve `{"status":"ok",...}` — latido del servicio |

---

## Estructura del repositorio

Cada carpeta es independiente y contiene su propio `README` a manera de manual detallado.

```
basket_recommender/
|-- README.md                           Documentacion general del repositorio global
|-- LICENSE                             Terminos de licencia de software del proyecto
|-- gitignore                           Archivos y extensiones ignorados por Git
|
|-- API/                                Servicio de inferencia desplegable
|   |-- app.py                          Aplicacion FastAPI para el servicio
|   |-- Dockerfile                      Imagen Docker del servicio
|   |-- requirements.txt                Dependencias de Python de la API
|   |-- readme.md                       Documentacion de la API
|   |-- Guide_API_Into_AmazonMockUp.ipynb Guia de integracion y prueba
|   |-- transformer_fusion.pt           Pesos del modelo Transformer
|   |-- transformer_vocab.pkl           Vocabulario SKU-indice
|   |-- catalogo.pkl                    Catalogo serializado
|   |-- scores_dict.pkl                 Scores RNN / TDA / TS por producto
|
|-- Executive Presentation/             Presentacion ejecutable para stakeholders
|   |-- ANA-03_TCA_Eq2.pdf              Archivo de la presentacion ejecutiva
|
|-- Front-end/                          Aplicacion web Canasta
|   |-- readme.md                       Documentacion del frontend
|   |-- catalogo.pkl                    Catalogo serializado para la interfaz
|   |-- vocabulario.pkl                 Vocabulario serializado
|   |
|   |-- RecepcionFacturasWeb/           Proyecto ASP.NET Core
|       |-- Controllers/                Controladores del backend
|       |-- Models/                     Modelos de datos
|       |-- Properties/                 Configuracion del proyecto
|       |-- Services/                   Servicios internos
|       |-- wwwroot/                    Archivos estaticos del frontend
|       |-- bin/                        Artefactos generados de compilacion
|       |-- obj/                        Artefactos generados de compilacion
|       |-- AuthController.cs           Controlador de autenticacion
|       |-- EmailService.cs             Servicio auxiliar de correo
|       |-- Program.cs                  Punto de entrada de la aplicacion
|       |-- RecepcionFacturasWeb.csproj Proyecto .NET
|       |-- RecepcionFacturasWeb.csproj.user Configuracion local de usuario
|       |-- RecepcionFacturasWeb.http   Pruebas HTTP del backend
|       |-- RecepcionFacturasWeb.slnx   Solucion de Visual Studio
|       |-- appsettings.json            Configuracion general
|       |-- appsettings.Development.json Configuracion de desarrollo
|       |-- Catalogo front.json         Catalogo utilizado por la vista
|       |-- catalogo.json               Catalogo de productos
|       |-- Combos.json                 Datos de combos
|       |-- Destacados.json             Productos destacados
|       |-- usuarios.json               Usuarios de demostracion
|
|-- Kedro/                              Gestion de pipelines de datos con Kedro
|   |-- README.md                       Manual explicativo del componente
|   |-- KedroRepo.md                    Enlace al repositorio externo de Kedro
|
|-- RNN/                                Componente secuencial RNN/GRU
|   |-- README.md                       Documentacion del modulo RNN
|   |-- requirements.txt                Dependencias de Python para RNN
|   |-- docs/                           Documentos del modulo
|   |   |-- DOCUMENTACION_RNN.md        Archivo de documentacion de la RNN
|   |-- notebooks/                      Cuadernos de trabajo de RNN
|   |   |-- RNN_LSTM_BasketRecommender_FINAL.ipynb Cuaderno de desarrollo del modelo RNN
|   |-- outputs_rnn/                    Resultados y salidas del componente RNN
|       |-- rnn_scores.csv              Archivo de scores de RNN
|       |-- curvas_entrenamiento_gru.png Imagen de curvas de entrenamiento de GRU
|       |-- distribucion_canastas.png   Imagen de distribucion de canastas
|       |-- distribucion_score_rnn.png  Imagen de distribucion de scores de RNN
|       |-- embeddings_pca_gru.png      Imagen de embeddings PCA de GRU
|
|-- TDA/                                Componente de Analisis Topologico de Datos
|   |-- README.md                       Documentacion del modulo TDA
|   |-- requirements.txt                Dependencias de Python para TDA
|   |-- notebooks/                      Cuadernos de trabajo de TDA
|       |-- TDA_Mapper_Documentado.ipynb Cuaderno del algoritmo Mapper documentado
|
|-- Reports/                            Reportes finales del proyecto
|   |-- Technical Report.pdf             Archivo del reporte tecnico
|   |-- Executive Report.pdf           Archivo del reporte ejecutivo
|
|-- Time Series/                        Componente temporal score_TS
|   |-- README.md                       Documentacion del modulo de series de tiempo
|   |-- requirements.txt                Dependencias de Python para series de tiempo
|   |-- .gitignore                      Archivo gitignore especifico del modulo
|   |-- data/                           Carpeta de datos del modulo
|   |   |-- README_data.md              Documentacion de los datos de series de tiempo
|   |-- docs/                           Documentos del modulo
|   |   |-- Documentacion_series_tiempo_ANA03.docx Documento de texto de la documentacion
|   |-- notebooks/                      Cuadernos de trabajo de series de tiempo
|   |   |-- ANA03_series_tiempo_recomendador.ipynb Cuaderno de desarrollo de series de tiempo
|   |-- outputs_ana03/                  Resultados y salidas de series de tiempo
|       |-- weekly_series_ana03.csv     Archivo de series semanales ANA03
|       |-- ts_scores_ana03.csv         Archivo de scores de series de tiempo ANA03
|       |-- statistical_tests_by_sku.csv Archivo de pruebas estadisticas por SKU
|       |-- statistical_tests_summary.csv Archivo de resumen de pruebas estadisticas
|       |-- pie_estacionariedad_series.png Imagen de estacionariedad de las series
|       |-- pie_ciclicidad_series.png   Imagen de ciclicidad de las series
|       |-- bar_tipo_ciclo_series.png   Imagen de tipo de ciclo de las series
|
|-- Transformer-fusion/                 Modelo final Early Fusion Transformer
|   |-- README.md                       Documentacion del modulo Transformer-fusion
|   |
|   |-- Data/                           Datos del modulo Transformer-fusion
|   |   |-- README.md                   Documentacion de la carpeta de datos
|   |   |-- catalogo.pkl                Archivo de catalogo serializado
|   |   |-- scores_dict.pkl             Archivo de diccionario de scores serializado
|   |   |-- transformer_vocab.pkl       Archivo de vocabulario del Transformer serializado
|   |
|   |-- Model/                          Componentes del modelo Transformer
|   |   |-- README.md                   Documentacion de la carpeta del modelo
|   |   |-- requirements.txt            Dependencias de Python para el modelo
|   |   |-- Operation_Manual.pdf        Archivo del manual de operacion
|   |   |-- Transformer_fusion_final.ipynb Cuaderno del modelo final Transformer-fusion
|   |   |-- transformer_fusion.pt       Archivo de pesos del Transformer en PyTorch
|   |
|   |-- Model Optimization/             Optimizacion del modelo Transformer
|   |   |-- README.md                   Documentacion de la carpeta de optimizacion
|   |   |-- requirements.txt            Dependencias de Python para optimizacion
|   |   |-- data/                       Carpeta de datos de optimizacion
|   |   |-- notebooks/                  Cuadernos de trabajo de optimizacion
|   |       |-- Optuna_optimized_model.ipynb Cuaderno del modelo optimizado con Optuna
|   |       |-- outputs_optuna/         Carpeta de salidas de Optuna
|   |       |-- outputs_optuna_rnn_item_scores/ Carpeta de salidas de scores de RNN en Optuna
|   |
|   |-- Recommendation Inference Function/ Funcion de inferencia de recomendaciones
|       |-- README.md                   Documentacion de la funcion de inferencia
|       |-- requirements.txt            Dependencias de Python para la inferencia
|       |-- Inference Function.ipynb    Cuaderno de la funcion de inferencia
```

---

## Flujo de la solución

```
[ERP TCA Merksyst — Parquet]  pdvhdr · pdvdet · inviar · invars
        │  limpieza, reconstrucción de tickets, enriquecimiento
        ▼
   3 pipelines de señales (Kedro)
   ┌──────────┬───────────────┬──────────┐
   │  TDA     │ Series Tiempo │   RNN    │   → score_TDA, score_TS, score_RNN
   └──────────┴───────────────┴──────────┘
        │  outer merge por cve_art + normalización
        ▼
   Pipeline Transformer  →  EarlyFusionTransformer  →  transformer_fusion.pt
        │  empaquetado FastAPI + Docker
        ▼
   Azure Container Registry  →  Azure Container Apps  →  POST /recommend
        │  proxy del back-end
        ▼
   Aplicación web "Canasta"
```

| Componente | Aporta | Salida |
|---|---|---|
| **TDA** | Similitud estructural del catálogo (Mapper + DBSCAN, Jaccard mod. + BFS) | `tda_scores.csv` |
| **Series de Tiempo** | Tendencia, recencia, volumen, estacionalidad e inventario | `ts_scores_ana03.csv` |
| **RNN (GRU)** | Patrones secuenciales de canastas históricas | `rnn_scores.csv` |
| **Transformer** | Fusión temprana de las 3 señales + Top-K final | `transformer_fusion.pt` |

---

## Ejecución

### 1) Pipelines (Kedro)

```bash
pip install uv
uv pip install -r requirements.txt

# Los pipelines TDA, TS y RNN deben correr ANTES que el Transformer.
kedro run -p transformer_training
```

### 2) API en local (Docker Desktop)

```bash
cd API
docker build -t reco-api .
docker run -p 8000:8000 reco-api
# → http://localhost:8000/view   y   http://localhost:8000/docs
```

### 3) Consumo del endpoint

```bash
curl -X POST \
  https://reco-basket-app.gentlesea-cb2510e3.eastus.azurecontainerapps.io/recommend \
  -H "Content-Type: application/json" \
  -d '{ "products": ["028253", "028593"] }'
```

CORS habilitado, sin API key. El orden de los productos importa (modelo secuencial). La clave `Descripción` lleva acento: en JavaScript usa `row["Descripción"]`.

### 4) Despliegue (Azure)

```bash
az acr login --name acrrecobasket
docker tag reco-api acrrecobasket.azurecr.io/reco-api:latest
docker push acrrecobasket.azurecr.io/reco-api:latest

az containerapp up \
  --name reco-basket-app \
  --resource-group rg-reco-basket \
  --location eastus \
  --image acrrecobasket.azurecr.io/reco-api:latest \
  --ingress external --target-port 8000 \
  --registry-server acrrecobasket.azurecr.io \
  --registry-username acrrecobasket \
  --registry-password <ACR_PASSWORD>
```

El servicio escala a cero en inactividad: la primera solicitud tras un periodo sin tráfico tarda ~10–20 s (*cold start*).

### 5) Aplicación web "Canasta" (solo Windows)

Requiere Windows 10/11, Visual Studio 2022+ con la carga *ASP.NET y desarrollo web* y el SDK de .NET. Abrir el `.sln`, revisar `appsettings.json` (`CarpetaCsv` y `RecomendadorUrl`), restaurar NuGet, compilar (`Ctrl+Shift+B`) y ejecutar (`F5`, por defecto `https://localhost:7237`). Detalle completo en `Front-end/readme.md`.

---

## Stack tecnológico

- **Modelo:** PyTorch (`EarlyFusionTransformer`), optimizado con **Optuna** (TPE + MedianPruner). Ganador `optuna_trial_18` — MRR@10 ≈ 0.1886.
- **Orquestación:** Kedro (4 pipelines).
- **API:** FastAPI + Uvicorn.
- **Contenedor:** Docker (Python 3.12, PyTorch 2.11.0 CPU).
- **Hosting:** Azure Container Apps + Azure Container Registry.
- **Front-end:** ASP.NET Core (.NET) + HTML/JS.

---

## ⚠️ Confidencialidad y requisitos de datos

Por seguridad y confidencialidad empresarial, **las bases transaccionales crudas no se incluyen** en el repositorio. Para reproducir el procesamiento o reentrenar desde cero hay que solicitar y colocar localmente:

- `pdvhdr.parquet` — encabezados de tickets de venta.
- `pdvdet.parquet` — detalle de artículos por ticket.
- `inviar.parquet` e `invars.parquet` — catálogo e inventario.

Para probar **solo la inferencia** con el modelo ya optimizado basta con los `.pkl` serializados y los pesos `.pt`.

---

*Desarrollado para TCA Software Solutions — Reto ITESM, Ingeniería en Ciencias de Datos y Matemáticas.*
