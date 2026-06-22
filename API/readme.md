# Basket Recommender — Next-Product Recommendation API

A sequence-based "customers who bought these also bought…" recommender. Given a
shopping basket of 1–10 product codes, it returns the **top-10 most likely next
products**, ranked. The model is an **early-fusion Transformer** that combines
product-ID embeddings with three behavioral score streams (RNN, TDA, and
Time-Series), served as a containerized REST API on Azure Container Apps.

---

## Live API

> ⚠️ Deployed on a student Azure subscription — the live URL may be taken down after
> evaluation. Everything needed to rebuild and redeploy it is in this repository.

## USE functioning API: https://reco-basket-app.gentlesea-cb2510e3.eastus.azurecontainerapps.io/view

Base URL:
```
https://reco-basket-app.gentlesea-cb2510e3.eastus.azurecontainerapps.io
```

| Surface | Add to base URL | Purpose |
|---|---|---|
| **Visual demo** | `/view` | Browser table of recommendations — no code needed |
| **Swagger UI** | `/docs` | Interactive tester + auto-generated API documentation |
| **Endpoint** | `POST /recommend` | The integration endpoint that apps call |
| **Health check** | `/` | Returns `{"status":"ok",...}` — a heartbeat, not a webpage |

Full integration instructions for front-end developers are in
[`api/Guide_API_Into_AmazonMockUp.md`](api/Guide_API_Into_AmazonMockUp.md).

**HOWEVER, the front end WE have developed is in the folder called Front-end**

---

## The model

The core is the `EarlyFusionTransformer`. For each item in the basket it builds a
representation from two halves: one half is a learned **product-ID embedding**, the
other half is a projection of three precomputed **behavioral scores** (RNN, TDA,
Time-Series). These are concatenated ("early fusion"), combined with positional
embeddings, and passed through a Transformer encoder. The representation at the final
sequence position is projected over the full product vocabulary (~3,000 products) to
predict the next item.

Because it reads the basket as an ordered **sequence**, the order in which products
are added can change the recommendations — this is expected behavior, not a bug.

Hyperparameters were tuned with **Optuna** (TPE sampler + MedianPruner), optimizing
**validation MRR**; promising configurations were retrained for 15 epochs and the
best was selected. See [`notebooks/1_training_optuna.ipynb`](notebooks/1_training_optuna.ipynb).

---

## Repository structure

```
basket-recommender/
├── api/                          The deployable API (self-contained build context)
│   ├── app.py                    FastAPI app: loads the model, serves /recommend, /view, /docs
│   ├── Dockerfile                Builds the container image (pins torch 2.11.0 CPU)
│   ├── requirements.txt          Python dependencies (torch installed separately in Dockerfile)
│   ├── HANDOFF_GUIDE.md          Integration guide for front-end developers
│   ├── transformer_fusion.pt     Trained model weights
│   ├── transformer_vocab.pkl     Product-code ↔ index vocabulary
│   ├── catalogo.pkl              Product catalog (descriptions, prices)
│   └── scores_dict.pkl           Per-product RNN / TDA / Time-Series scores
│
├── notebooks/
│   ├── 1_training_optuna.ipynb   Model architecture + Optuna hyperparameter search (training)
│   └── 2_inference_demo.ipynb    Loading the model and generating recommendations (exploration)
│
└── reports/
    ├── Technical_Report.pdf      Detailed methodology, architecture, and results
    └── Executive_Report.pdf      High-level summary for non-technical stakeholders
```

The model artifacts live inside `api/` (next to the `Dockerfile`) on purpose: the app
loads them by filename and the Dockerfile copies them from the same folder, so the
deployable unit is fully self-contained.

---

## Run it locally

Requires Docker Desktop. From the repository root:

```bash
cd api
docker build -t reco-api .
docker run -p 8000:8000 reco-api
```

Then open:
- **http://localhost:8000/view** — the visual table
- **http://localhost:8000/docs** — the Swagger tester

The Docker build runs a smoke test that loads the model and performs one
recommendation; if the model can't load, the build fails immediately rather than at
runtime.

> Running `app.py` outside Docker also requires the CPU build of PyTorch:
> `pip install torch==2.11.0 --index-url https://download.pytorch.org/whl/cpu`,
> then `pip install -r requirements.txt`.

---

## Deploy (Azure Container Apps)

The image is pushed to Azure Container Registry and deployed to Azure Container Apps.
Outline of the flow (names from this project):

```bash
# Push the image to the registry
az acr login --name acrrecobasket
docker tag reco-api acrrecobasket.azurecr.io/reco-api:latest
docker push acrrecobasket.azurecr.io/reco-api:latest

# Deploy (creates the environment + app, pulls the image)
az containerapp up \
  --name reco-basket-app \
  --resource-group rg-reco-basket \
  --location eastus \
  --image acrrecobasket.azurecr.io/reco-api:latest \
  --ingress external \
  --target-port 8000 \
  --registry-server acrrecobasket.azurecr.io \
  --registry-username acrrecobasket \
  --registry-password <ACR_PASSWORD>
```

The app scales to zero when idle, so the first request after a quiet period takes
~10–20 seconds (cold start) while the model loads, then is fast.

---

## API usage

**Request** — `POST /recommend`, header `Content-Type: application/json`:
```json
{ "products": ["028253", "028593"] }
```
- 1 to 10 product codes, as **strings**. Order matters (sequence model).

**Response:**
```json
{
  "canasta": ["028253", "028593"],
  "recomendaciones": [
    {
      "Ranking": 1,
      "CVE_Articulo": "028253",
      "Descripción": "PASTELITO MINI GANSITO MARINELA 25 GR",
      "Precio": 8.9,
      "Score_Relativo": 1.0,
      "Salto_vs_Siguiente": 12.69
    }
    // ... 9 more rows
  ]
}
```

CORS is enabled and no API key is required. The `Descripción` key carries an accent,
so in JavaScript use bracket access: `row["Descripción"]`. See
[`api/HANDOFF_GUIDE.md`](api/HANDOFF_GUIDE.md) for a full walkthrough and a
copy-paste `fetch()` example.

---

## Tech stack

- **Model:** PyTorch (`EarlyFusionTransformer`), tuned with Optuna
- **API:** FastAPI + Uvicorn
- **Container:** Docker (Python 3.12, CPU-only PyTorch 2.11.0)
- **Hosting:** Azure Container Apps + Azure Container Registry

---

## Authors

-   _Diego Armando Mijares Ledezma (A01722421)_
-   _Daniel Eduardo Arana Bodart (A01741202)_
-   _Mauricio Octavio Valencia Gonzalez (A01234397)_
-   _David Alejandro Acuña Orozco (A00571187)_
-   _Diego Gutiérrez Vargas (A01285421)_
-   _Alfonso Elizondo Partida (A01285151)_
