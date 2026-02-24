# DataRobot AutoML vs RHOAI AutoML — Summary

This comparison is based on **DataRobot's automated ML offering** (marketed as **Autopilot**): the mode where DataRobot automatically builds and evaluates many models for a given dataset. It uses DataRobot public documentation (modeling, blueprints, tuning, deployment) and the RHOAI AutoML ADR and pipeline implementation.

## 1. Differences

### **Delivery & licensing**

| Aspect | DataRobot | RHOAI AutoML |
|--------|-----------|--------------|
| **Model** | Commercial (SaaS + on-prem), custom pricing, multiple license tiers | Part of Red Hat OpenShift AI; no separate AutoML license |
| **Pricing** | Contact sales; license scope can be unclear (e.g. what "MLDev" allows) | Included with RHOAI / OpenShift; no per-seat AutoML fee |
| **Deployment** | DataRobot platform on your infra (e.g. OpenShift via Helm) or SaaS | Your OpenShift cluster; pipelines and models run inside RHOAI |

### **Optimization approach**

| Aspect | DataRobot | RHOAI AutoML |
|--------|-----------|--------------|
| **Strategy** | **HPO-centric**: Autopilot runs a curated set of blueprints and tests thousands of combinations of preprocessing and parameter settings; optional Advanced Tuning (grid search, Smart Search, brute force) for overrides | **Ensembling-centric**: AutoGluon stacking/bagging, no traditional HPO; sample → train many model types → select top N → refit on full data |
| **Blueprints / pipeline** | Blueprints = preprocessing + model + post-processing (data by type: numeric, categorical, text, image, geospatial); many use DataRobot proprietary preprocessing | Single defined KFP pipeline: load → split → model selection → refit → leaderboard (and optional register/serve) |
| **Compute** | Tuning many hyperparameter grids can be heavy | Fewer "search" steps; two-stage (sample then full refit) keeps cost predictable |

*References: [DataRobot Advanced Tuning](https://docs.datarobot.com/en/docs/modeling/analyze-models/evaluate/adv-tuning.html), [DataRobot Blueprints](https://docs.datarobot.com/en/modeling/analyze-models/describe/blueprints.html); [ODH-ADR-0001-automl](https://github.com/LukaszCmielowski/architecture-decision-records/blob/main/architecture-decision-records/automl/ODH-ADR-0001-automl.md); [autogluon_tabular_training_pipeline README](https://github.com/LukaszCmielowski/pipelines-components/tree/rhoai_automl/pipelines/training/automl/autogluon_tabular_training_pipeline).*

### **Scope & features**

| Aspect | DataRobot | RHOAI AutoML |
|--------|-----------|--------------|
| **Data** | Tabular + time series + text, image, geospatial; broad EDA; optional image augmentation | Tabular only (CSV, Parquet, XLSX); time-series pipeline separate; no images/text/audio |
| **Tasks** | Classification, regression, time series (forecasting + nowcasting), clustering, anomaly detection; advanced options (bias/fairness, partitioning, etc.) | Classification (binary/multiclass), regression, time-series forecasting (dedicated pipeline) |
| **Extras** | Leaderboard (ranked models); bias/fairness, monotonic constraints; quantile regression & custom task hyperparameters (preview) | Leaderboard, feature importance, confusion matrix; fairness/explainability on roadmap |

### **Integration & deployment**

| Aspect | DataRobot | RHOAI AutoML |
|--------|-----------|--------------|
| **Orchestration** | DataRobot's own engine and UI | AutoML's own UI, Kubeflow Pipelines (KFP); same as rest of RHOAI; API + RHOAI Dashboard (AI Pipelines UI) |
| **Experiment tracking** | DataRobot-native | MLflow (experiments, metrics, artifacts) |
| **Model registry** | DataRobot Model Registry | MLflow / RHOAI Model Registry |
| **Serving** | Portable Prediction Server (PPS), Scoring Code (e.g. JAR), MLOps Agent | KServe with custom AutoGluon runtime |
| **Data access** | Platform-managed connectors | RHOAI Connections (Kubernetes secrets, e.g. S3); namespace-scoped |

### **Transparency & portability**

| Aspect | DataRobot | RHOAI AutoML |
|--------|-----------|--------------|
| **Core engine** | Proprietary blueprints and tuning | **AutoGluon** (open-source); [docs](https://auto.gluon.ai/stable/index.html) |
| **Pipeline definition** | Internal to platform | **Code & YAML** (KFP pipeline in [pipelines-components](https://github.com/LukaszCmielowski/pipelines-components/tree/rhoai_automl/pipelines/training/automl/autogluon_tabular_training_pipeline)); GitOps-friendly |
| **Artifacts** | Scoring Code (Java JAR), Portable Prediction Server (Docker/.mlpkg); no standard open model format | Standard AutoGluon predictors on object storage; load with `TabularPredictor.load(...)` |

---

## 2. Pros of DataRobot vs RHOAI AutoML

1. **Broader problem and data scope** — Supports tabular, time series (forecasting + nowcasting), text, image, and geospatial data in one platform; clustering and anomaly detection. RHOAI AutoML is tabular-only (plus a separate time-series pipeline) with no image, text, or geospatial support.

2. **Richer EDA and preprocessing** — Broad exploratory data analysis and preprocessing by data type (numeric, categorical, text, image, geospatial), including proprietary preprocessing. RHOAI uses a single pipeline with AutoGluon’s built-in preprocessing.

3. **Mature enterprise platform** — Single vendor for modeling, deployment, and MLOps; commercial support, SLAs, and documented deployment options (PPS, Scoring Code, MLOps Agent). Open stack (RHOAI + AutoGluon) relies on community and Red Hat support.

4. **One integrated experience** — EDA, Autopilot, leaderboard, deployment, and monitoring in a single product and UI. RHOAI spreads workflows across Pipelines UI, Model Registry, and KServe.


---

## 3. Pros of RHOAI's approach vs DataRobot

1. **No vendor lock-in** — Stack is open (AutoGluon, Kubeflow Pipelines, MLflow, KServe). Models are standard AutoGluon predictors; you can run and serve them outside RHOAI if needed.

2. **No separate AutoML licensing** — No per-seat or opaque "MLDev"-style limits for building or downloading models; AutoML is part of RHOAI on OpenShift.

3. **Efficiency for tabular** — Ensembling (stacking/bagging) avoids large hyperparameter grids. Two-stage design (sample → select top N → refit on full data) keeps runs more predictable and often competitive with heavy HPO on tabular (as in the ADR rationale).

4. **Data and control stay on your cluster** — Training and serving run in your OpenShift cluster with RHOAI Connections and namespace isolation; no dependency on a vendor SaaS for core workflows.

5. **Transparent, reproducible pipeline** — The pipeline is a visible KFP DAG and code (e.g. `pipeline.py` + README). You can version it, change it, and reuse the same pattern for other projects.

6. **Unified platform** — Same environment for pipelines, Model Registry, and KServe; one place for data connections, RBAC, and audit.

7. **Community-driven ML core** — AutoGluon (AWS-origin, open-source) is documented and extensible; behavior and limits are inspectable, unlike a proprietary blueprint engine.

8. **Operational consistency** — One Kubernetes/OpenShift stack for training and serving (KServe); no separate PPS or JAR-based deployment story to maintain.

9. **API + UI** — Fits programmatic use (pipelines API) and the RHOAI Dashboard (AI Pipelines UI), similar in spirit to DataRobot's UI + API but inside your own platform.

10. **Roadmap aligned with open ecosystem** — ADR mentions Kubeflow Katib for scaling, ONNX for portability, and MCP for tooling—all in the open-source / standard ecosystem rather than vendor-specific extensions.

---

**Bottom line:** RHOAI AutoML is an **open, platform-integrated, ensembling-based** tabular (and time-series) AutoML that runs entirely in your OpenShift environment with no separate AutoML vendor or license. DataRobot is a **broader, HPO-heavy commercial platform** with more vendor lock-in and licensing complexity. For tabular use cases where control, cost, and openness matter, RHOAI's approach is a strong alternative.

---

## Sources (DataRobot automated ML)

- [DataRobot Blueprints](https://docs.datarobot.com/en/modeling/analyze-models/describe/blueprints.html) — blueprints as ML pipelines (preprocessing + model + post-processing), thousands of combinations, proprietary preprocessing
- [DataRobot Advanced Tuning](https://docs.datarobot.com/en/docs/modeling/analyze-models/evaluate/adv-tuning.html) — hyperparameter tuning (grid search, Smart Search, brute force)
- [DataRobot Modeling / Build models](https://docs.datarobot.com/en/modeling/build-models/index.html) — basic workflow, target setting, modeling modes
- [DataRobot Model Leaderboard](https://docs.datarobot.com/en/docs/modeling/reference/model-detail/leaderboard-ref.html) — ranked models, Workbench leaderboard
- [DataRobot Time series modeling](https://docs.datarobot.com/en/modeling/time/index.html) — time series forecasting and nowcasting
- [DataRobot Scoring Code / Portable Predictions](https://docs.datarobot.com/en/predictions/port-pred/scoring-code/index.html) — Java JAR export, PPS (Docker/.mlpkg)
- [DataRobot Deploy methods](https://docs.datarobot.com/en/mlops/deployment/deploy-methods/index.html) — PPS, Scoring Code, MLOps Agent
- DataRobot public docs (modeling FAQ, Autopilot behavior, problem types) — classification, regression, time series, clustering, anomaly detection
