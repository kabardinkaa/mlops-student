# Домашнее задание: докажите воспроизводимость изменения

## Шаг 1. Изменение параметра

Исходный Git SHA: `c6f7a5b`. В `params.yaml` был изменён ровно один параметр — `training.threshold`: было `0.5`, стало `0.55`. Теперь модель относит клиента к группе высокого риска при вероятности оттока от 55%, а не от 50%.

```yaml
training:
  test_size: 0.2
  random_state: 42
  model: logistic_regression
  threshold: 0.55
```

## Шаг 2. DVC

Команда `dvc repro` фактически пересчитала три стадии:

- `generate_data`;
- `train`;
- `monitor`.

Threshold был единственным ручным изменением. Однако существующий `dvc.lock` также содержал устаревшие хеши зависимостей, включая код и `requirements.txt`, поэтому фактический запуск обновил все три стадии, а не только обучение.

После выполнения pipeline команда `dvc status` показала:

```text
Data and pipelines are up to date.
```

Результаты `dvc metrics show`:

| Метрика | Значение |
|---|---:|
| ROC AUC | 0.7693 |
| Average Precision | 0.2919 |
| Sentiment Accuracy | 1.0 |
| Sentiment F1 | 1.0 |
| `train_rows` | 3200 |
| `test_rows` | 800 |

Цепочка DVC устроена так: значение меняется в `params.yaml`, стадии и их зависимости описаны в `dvc.yaml`, а `dvc.lock` после запуска фиксирует новые хеши зависимостей и созданных артефактов.

[Скриншот терминала: dvc repro, dvc status и dvc metrics show]

## Шаг 3. MLflow

Обучение создало новый MLflow run со следующими результатами:

| Поле | Значение |
|---|---|
| Run ID | `44ac9a9da8ef48019a2505d9650f296e` |
| Threshold | `0.55` |
| ROC AUC | `0.7693` |
| Average Precision | `0.2919` |
| Sentiment Accuracy | `1.0` |
| Sentiment F1 | `1.0` |
| Model Registry version | `1` |
| SHA256 модели | `d5bbcfcf5aa2f1e34a89928b88becd60eda47b4849c69d9e13c74469a4a53904` |

В `models/model_metadata.json` идентификаторы совпадают:

```json
{
  "version": "44ac9a9da8ef48019a2505d9650f296e",
  "mlflow_run_id": "44ac9a9da8ef48019a2505d9650f296e",
  "threshold": 0.55,
  "sha256": "d5bbcfcf5aa2f1e34a89928b88becd60eda47b4849c69d9e13c74469a4a53904"
}
```

[Скриншот MLflow: новый run, параметры и метрики]

## Шаг 4. Паспорт релиза

Скрипт `scripts/build_release_manifest.py` сформировал `reports/release_manifest.json`. Его ключевые поля:

```json
{
  "service": {
    "version": "0.2.0",
    "git_sha": "c6f7a5b6c2c7effe98858e01c1aa3cf1754fe630"
  },
  "pipeline": {
    "dvc_lock_sha256": "0d2f101e5ed6a4715eb5855b261e5476c046eed250504f2aeaf4384d4e307afa",
    "params_sha256": "1c3034d3adffd7ed691eedff19a50abb6549127a87972e0b705973650859661a"
  },
  "data": {
    "reference_sha256": "6e980605a34ef962e3d65a688c0fa020d844f8e49f6be17745f4f9aecbab6972",
    "current_sha256": "74b50492af61213c81c577546aa11d37eed6337708703453045449eb20e949fa"
  },
  "model": {
    "mlflow_run_id": "44ac9a9da8ef48019a2505d9650f296e",
    "threshold": 0.55,
    "sha256": "d5bbcfcf5aa2f1e34a89928b88becd60eda47b4849c69d9e13c74469a4a53904"
  },
  "metrics": {
    "roc_auc": 0.7693,
    "average_precision": 0.2919,
    "sentiment_accuracy": 1.0,
    "sentiment_f1": 1.0,
    "train_rows": 3200,
    "test_rows": 800
  },
  "image": {
    "tag": "mlops-student:c6f7a5b6c2c7effe98858e01c1aa3cf1754fe630"
  }
}
```

Паспорт связывает версию сервиса и кода с состоянием DVC, параметрами, данными, моделью и метриками. Он был сформирован локально до commit, поэтому Git SHA относится к текущему `HEAD`; в CI после commit используется точный SHA запуска GitHub Actions.

[Скриншот: ключевые поля reports/release_manifest.json]

## Шаг 5. CI и CD

Workflow `.github/workflows/ci.yml` выполняет:

1. Checkout репозитория.
2. Настройку Python 3.11.
3. Установку зависимостей из `requirements.txt`.
4. Запуск `dvc repro`.
5. Формирование release manifest.
6. Проверку `dvc status` и `dvc metrics show`.
7. Запуск `pytest`.
8. Проверку `docker build`.
9. Сохранение `dvc.lock`, metadata, метрик, отчёта мониторинга и паспорта как evidence.

В контуре вебинара №2 это CI, потому что workflow воспроизводит, проверяет и собирает поставку. Он не публикует Docker-образ и не разворачивает сервис. Отдельный `cd.yml` в современной версии репозитория является более поздним развитием проекта и в эту практическую часть не входит.

## Итоговая цепочка

`training.threshold: 0.5 → 0.55` → DVC пересчитал `generate_data`, `train` и `monitor` → MLflow создал run `44ac9a9da8ef48019a2505d9650f296e` → тот же run ID и threshold `0.55` попали в `models/model_metadata.json` → `reports/release_manifest.json` связал Git SHA, DVC, данные, модель и метрики → CI сможет повторить эти проверки после commit и отправки изменений.
