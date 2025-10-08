# Классификация кризисно‑релевантных твитов (Informative vs Not informative)

Социальные сети становятся всё более важным источником данных для служб экстренного реагирования во время кризисов. Ключевая проблема — быстро распознавать **полезные** сообщения среди общего потока. В этом проекте мы анализируем твиты о кризисах (Twitter) и строим классификатор, который отбирает посты с релевантной информацией для управления последствиями катастроф.

Проект состоит из 4 ноутбуков: подготовка данных, LSTM+Word2Vec, BERT и XLNet. Цель — автоматически определять **информативные** сообщения во время кризисов, чтобы помочь службам и волонтёрам быстрее отбирать полезные твиты.

---

## 🧭 Краткое резюме

- **Задача.** Бинарная классификация твитов: *Related & Informative* (информативные) vs *Not informative* (неинформативные).
- **Данные.** Набор **CrisisLex T26** (26 событий разных типов). Тексты нормализуются и переводятся (для не‑английских), дубль‑твиты удаляются.
- **Модели.**
  1) **LSTM + Word2Vec** (from scratch tokenizer + padding)
  2) **BERT (`bert-base-uncased`)**
  3) **XLNet (`xlnet-base-cased`)**
- **Сценарии экспериментов.**
  - *Scenario 1 (specialized):* обучение и тест в пределах **одного типа кризиса** (например, наводнения).
  - *Scenario 2 (generic):* обучение на **всех типах**, тест на отложенных событиях (кросс‑типовая генерализация).
- **Результаты (репрезентативные для текущего проекта).**
  - **LSTM**: dev accuracy ~ **0.81**, test accuracy ~ **0.80**, macro‑F1 ~ **0.79–0.80**.
  - **BERT**: test accuracy ~ **0.86**, macro‑F1 ~ **0.86**, **ROC‑AUC ~ 0.93**.
  - **XLNet**: dev/test accuracy ~ **0.86**, **ROC‑AUC ~ 0.93**.
  - Поведение согласуется с отчётом: **XLNet ≥ BERT ≫ LSTM**; *generic* модель часто не хуже специализированной.

Числа слегка варьируются от сидов/сплитов/тюнинга. Сравнимость обеспечивается строгим разделением **train/val/test** и выбором гиперпараметров **только по val**.

---

## 📁 Структура репозитория

```
.
├── 01_AutomaticDetectionOfCrisisRelatedMessages.ipynb   # Подготовка данных, EDA, baseline (лексикон)
├── 02_LSTMword2vec.ipynb                                # LSTM + Word2Vec (PyTorch) + MLflow
├── 03_BERT.ipynb                                        # Дообучение BERT + MLflow + ROC/AUC
├── 04_XLNETFineTuning.ipynb                             # Дообучение XLNet + MLflow + ROC/AUC
└── README.md                                            # Отчёт по проекту
```

---

## 🧪 Данные и предобработка [![Open In Colab](https://colab.research.google.com/assets/colab-badge.svg)](https://colab.research.google.com/drive/1T4XiS8l6RgoAUB9ItGfS2u57tPVpATT1)

**Источник:** CrisisLex T26 — 26 событий, 12 типов кризисов (наводнения, землетрясения, крушения и т. п.).  
**Целевая переменная:** бинарная (Informative / Not informative).

**Пайплайн 01‑го ноутбука:**
- детект языка и **перевод только уникальных текстов** на английский (кэш в CSV);
- нормализация токенов: `<URL>`, `<USER>`, `<HASHTAG>`; маппинг смайликов/эмодзи;
- удаление дубликатов по нормализованному тексту;
- быстрая EDA;
- подготовка **train/test** сплитов (стратификация), сохранение на Google Drive (`/content/drive/MyDrive/crisis_lstm`).  
  Для последующих ноутбуков берём готовые `train_data.csv` и `test_data.csv`.

> В некоторых экспериментах исходный `test_data.csv` дополнительно делится на **validation** и **test**, чтобы подбирать гиперпараметры **только по validation**, а финальную метрику снимать на **test**.

---

## 🧩 Модели

### 1) LSTM + Word2Vec — `02_LSTMword2vec.ipynb` [![Open In Colab](https://colab.research.google.com/assets/colab-badge.svg)](https://colab.research.google.com/drive/1bYeuqP2WHV2BE2WymFs9oED8cUayG9aR)

**Текст → индексы.**
- `keras_preprocessing.Tokenizer`, `pad_sequences` до `max_length`, ограничение словаря `vocabulary_size`.
- Токенайзер обучаем **только на train**, чтобы не было утечки информации.

**Архитектура.**
```
Embedding(vocab_size, 50) → SpatialDropout1D(p≈0.3) → LSTM(256) → Dropout → Linear(256) + ReLU → Linear(1)
```
- `BCEWithLogitsLoss (бинарная кросс-энтропия)`, оптимизатор `Adam`.
- Реализация `SpatialDropout1D` через `nn.Dropout1d` (канал‑вайз на тензорах (N,T,E)).

**Обучение.**
- Грид/рандом‑поиск гиперпараметров (`ParameterSampler`, `scipy.stats.loguniform` для LR).
- Для каждой конфигурации: тренировка на **train**, оценка на **val**, лог кривых `loss/accuracy` (train/val) в MLflow.
- Выбор лучшей конфигурации по метрике (обычно `accuracy`/`F1` на val).
- Финальная оценка лучшей модели на **test** + ROC‑кривая, матрица ошибок.

**Метрики.**
- Accuracy, Precision, Recall, F1 (по классам/сводно), ROC‑AUC; Confusion Matrix, ROC curve.

**MLflow.**
- Лог параметров/метрик/кривых/ROC.
- Лог модели с `signature` и `input_example` (чтобы убрать предупреждения), при необходимости указываются `pip_requirements` («очищенные» версии без `+cu***`).

### 2) BERT — `03_BERT.ipynb` [![Open In Colab](https://colab.research.google.com/assets/colab-badge.svg)](https://colab.research.google.com/drive/1yQrwN64BG2YrpsLeWasCOgtsNVH4y2bk)

- `AutoModelForSequenceClassification("bert-base-uncased", num_labels=2)`.
- Оптимизатор `AdamW`, `get_linear_schedule_with_warmup` (warmup ≈ 10% шагов).
- **AMP**: `torch.amp.autocast(device_type='cuda')` + `GradScaler`.
- Взвешивание классов для `CrossEntropyLoss` по доле положительного класса в train.
- Логика обучения: **train → val** (лучший state) → **test**. Логирование кривых и ROC в MLflow.

### 3) XLNet — `04_XLNETFineTuning.ipynb` [![Open In Colab](https://colab.research.google.com/assets/colab-badge.svg)](https://colab.research.google.com/drive/1I6m1z4Wwe59C6-QhcsIFEEAPUauwLKkZ)

- `AutoModelForSequenceClassification("xlnet-base-cased", num_labels=2)`.
- Те же принципы: `AdamW` + linear warmup, AMP, class weights, MLflow, строгие сплиты.
- По отчёту и нашим повторным запускам XLNet даёт наивысший ROC‑AUC.

---

## 📊 Результаты

### Наблюдения из ноутбуков (репрезентативно)
- **LSTM+Word2Vec**: лучшая val accuracy ≈ **0.81**, на test ≈ **0.80**, macro‑F1 ≈ **0.79–0.80**.  
  Модель лучше распознаёт *Informative*, хуже — *Not informative* (ниже precision/recall).
- **BERT**: test accuracy ≈ **0.86**, macro‑F1 ≈ **0.86**, ROC‑AUC ≈ **0.93**. Баланс классов выровненнее.
- **XLNet**: на val/test ≈ **0.86** accuracy, ROC‑AUC ≈ **0.93**; стабильно конкурентнее или равен BERT.

> Точная величина метрик зависит от сида/макс‑длины/батча/LR и конкретного сета событий.

### Сводная таблица (пример)

| Модель               | Сплит | Accuracy | Macro‑F1 | ROC‑AUC | Комментарий |
|----------------------|:-----:|:--------:|:--------:|:-------:|-------------|
| **LSTM+Word2Vec**    | test  | ~0.80    | ~0.79–0.80| ~0.85   | +Dropout=0.3, Batch=128, Epochs≈25 |
| **BERT base**        | test  | ~0.86    | ~0.86    | **~0.93** | Threshold‑tuning по val даёт +F1 |
| **XLNet base**       | test  | ~0.86    | ~0.86    | **~0.93** | Часто топ‑AUC среди моделей |

† ROC‑AUC для LSTM зависит от набора признаков/порога; см. ноутбук для конкретной цифры.

---

## 🔁 Воспроизводимость и сплиты

- Жёсткое разделение: **train / val / test**. Гиперпараметры и пороги выбираются **только по val**. **Test** используется единожды в самом конце.
- Для сценариев *specialized* / *generic* следуйте протоколам из отчёта (какие события в train/val/test).

---

## 💡 Рекомендации по улучшению

- **Порог по dev.** Вместо фиксированного 0.5 подберите порог, максимизирующий macro‑F1/PR‑AUC на dev, и применяйте его на test.
- **Регуляризация LSTM.** `dropout=0.3`, `weight_decay=1e-4..5e-4`, gradient clipping `1.0`. Рассмотреть **BiLSTM** или attention‑слой.
- **Эмбеддинги.** Предобученные GloVe/word2vec (с постепенным unfreeze) нередко добавляют ~+0.5–1 п.п. F1.
- **Transformers.** Тонкая настройка LR/макс‑длины влияет сильнее батча; ранняя остановка по `val_loss` полезна против переобучения.

---

## 📈 Почему generic‑обучение часто выигрывает

По наблюдениям и отчёту: обучение на **всех типах кризисов** («generic») улучшает переносимость, особенно для типов с меньшим объёмом данных (землетрясения/сходы с рельс). Для наводнений «specialized» иногда лучше из‑за изначально богатого внутри‑типа корпуса. В целом **XLNet** показывает самый высокий ROC‑AUC, **BERT** рядом, **LSTM** — уверенный базовый уровень.
