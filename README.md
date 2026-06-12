**English** | [한국어](README.ko.md)

# nOOnchi — Real-Time Voice Phishing Detection Service

![SW Maestro](https://img.shields.io/badge/SW%20Maestro-13th%20(2022)-blue)
![Model](https://img.shields.io/badge/Model-KoBERT-orange)
![PyTorch](https://img.shields.io/badge/PyTorch-1.x-EE4C2C?logo=pytorch&logoColor=white)
![Backend](https://img.shields.io/badge/Backend-Nest.js%20%C2%B7%20Kubernetes%20%C2%B7%20AWS%20SQS-green)
![License](https://img.shields.io/badge/License-MIT-lightgrey)

> **Status: archived (Apr 8 – Dec 15, 2022).** This README was revised in 2026 to document the system *accurately*, including methodological limitations identified in retrospect. The limitations are kept here deliberately — they motivated a methodologically rigorous successor project, **[EarShield](https://github.com/pmy02/earshield)** <!-- TODO: update URL after pushing the successor repo -->, which redesigns the data splits, labels, and evaluation from the ground up.

**nOOnchi** ("nunchi" — the Korean word for reading a situation) is a voice phishing detection solution built during the **13th SW Maestro program**: an API service and mobile application that transcribes an ongoing phone call, classifies the conversation with a KoBERT-based model, and alerts the user when the call looks like voice phishing. The motivation was the steady year-over-year growth of voice phishing damage in Korea and the fact that most victims realize what happened only after the money has moved.

![image](https://github.com/pmy02/nOOnchi/assets/62882579/d35d8313-555f-4e9e-9efc-f82e809e5a45)
![image](https://github.com/pmy02/nOOnchi/assets/62882579/7f5917b1-d259-4feb-a4b6-ee4bf9e618b7)

The system consists of three parts: a KoBERT-based detection model, a streaming data pipeline built on Kubernetes and Amazon SQS, and a Nest.js API server. All research and prototype repositories live in the team organization: **[SWMTeamCuriosity](https://github.com/SWMTeamCuriosity)**.

---

## System Architecture

A phone call is streamed to the backend, transcribed in real time by a streaming STT engine, and the transcript segments are pushed through an asynchronous pipeline to the model server for classification. Results are returned to the client app, which raises an alert when the phishing score crosses a threshold.

- **Container orchestration:** Kubernetes-based container architecture
- **Asynchronous processing:** Amazon SQS decouples STT output from model inference
- **API server:** Nest.js with an MVC layering

![image](https://github.com/pmy02/nOOnchi/assets/62882579/0dcd6f34-139b-4eed-a452-bead5ba00d7e)

![image](https://github.com/pmy02/nOOnchi/assets/62882579/d4abd062-f76a-444e-a3bd-52d53d04d7cf)

### STT engine selection

Several Korean STT APIs were benchmarked on phishing-call audio for transcription accuracy and streaming capability:

![image](https://github.com/pmy02/nOOnchi/assets/62882579/99aa342e-8fde-4957-8781-74526f2ac66c)

**VITO STT** was selected on technical grounds — competitive accuracy while supporting real-time streaming, which the detection scenario requires. (A negotiated 20% discount was an additional cost consideration for the service, not a technical criterion.)

![image](https://github.com/pmy02/nOOnchi/assets/62882579/398fe901-96e9-4d90-a77e-f2bff884c3bf)

---

## Data

| Source | Raw collection | Used for training |
|---|---|---|
| **Phishing** — FSS ["그놈 목소리"](https://www.fss.or.kr/fss/bbs/B0000206/list.do?menuNo=200690) / ["바로 이 목소리"](https://www.fss.or.kr/fss/bbs/B0000203/list.do?menuNo=200686) (real scam-call recordings released by the Financial Supervisory Service) | 546 calls, crawled and transcribed with VITO STT | 18,000 utterances (label `True`) |
| **Normal** — [AI Hub](https://aihub.or.kr/aihubdata/data/view.do?currMenu=116&topMenu=100&aihubDataSe=ty&dataSetSn=123) Korean dialogue corpora (financial counseling, call-center, daily conversation) | ~78,000 dialogue records | 20,000 utterances (label `False`) |

![image](https://github.com/pmy02/nOOnchi/assets/62882579/9d4da9b7-8ed4-4150-beef-aaad69b1fbff)

![image](https://github.com/pmy02/nOOnchi/assets/62882579/caba59e1-75ab-47b1-adff-e99f14092d41)

**How the 38,000-utterance training set was actually built** (see [`text_refactoring/sentence_to_csv`](https://github.com/SWMTeamCuriosity/text_refactoring) and [`transcript_data`](https://github.com/SWMTeamCuriosity/transcript_data)): the 546 phishing calls were segmented into STT utterances; utterances longer than 2 characters were kept and the first 18,000 were labeled `True`. Normal utterances were extracted from the AI Hub dialogues the same way and the first 20,000 were labeled `False`. The original write-up described this step as "augmentation"; **utterance-level segmentation of call-level data** is the precise description — every utterance inherits the label of its source call.

> The raw AI Hub corpora are license-restricted and cannot be redistributed; the repositories contain only derived files used during the program.

---

## Model

A standard fine-tuning setup on top of **[KoBERT](https://github.com/SKTBrain/KoBERT)** (SKTBrain): the input utterance is tokenized with KoBERT's **SentencePiece subword tokenizer**, encoded, and the pooled representation is passed through a dropout + linear layer for binary classification.

| Hyperparameter | Value (from [`KoBERT_test`](https://github.com/SWMTeamCuriosity/KoBERT_test) notebook) |
|---|---|
| Max sequence length | 64 |
| Batch size | 64 |
| Optimizer / LR | AdamW / 1e-6 (linear warmup 10%) |
| Dropout | 0.5 |
| Loss | CrossEntropyLoss (2-way) |
| Epochs | 50 in the notebook configuration <!-- TODO: the deployed checkpoint was reported as 6 epochs; confirm which checkpoint shipped --> |
| Split | `train_test_split(test_size=0.2, random_state=0)` at the **utterance level** |

Alternative models were prototyped during model selection — [BiLSTM/RNN classifiers](https://github.com/SWMTeamCuriosity/BiLSTM_RNN_Text_Classification) and [pretrained fastText](https://github.com/SWMTeamCuriosity/pretrained_fastText) — and KoBERT performed best among them on the team's validation data.

![image](https://github.com/pmy02/nOOnchi/assets/62882579/df63ff97-ad6d-47ea-9001-c321c034a988)

---

## Reported Results

Metrics computed from a confusion matrix on held-out utterances:

| Accuracy | Precision | Recall | F1 |
|:---:|:---:|:---:|:---:|
| 94.8% | 81.3% | 96.2% | 88.1% |

High recall was prioritized: for a phishing alert system, a missed phishing call (false negative) is far more costly than a false alarm.

![image](https://github.com/pmy02/nOOnchi/assets/62882579/ef0394b9-4728-46cd-bbd4-0fcff9052583)

### ⚠️ Limitations & retrospective (read this before citing the numbers)

Re-auditing the code and data after the program ended surfaced four methodological problems. They are documented here honestly because they shaped the successor project.

1. **Same-call leakage inflates the metrics.** The train/test split was a *random utterance-level* split, so utterances from the *same phishing call* — often near-duplicates of each other — appear on both sides. The reported numbers therefore measure recognition of seen calls more than generalization to new ones. A grouped, **call-disjoint** split is the correct protocol.
2. **Label inheritance noise.** Every utterance inherits its call's label, so contentless fragments from phishing calls (e.g., "아파서", "다름이 아니라") are labeled `True`. The model is partly trained to call neutral filler "phishing", which hurts precision on real traffic.
3. **The evaluation protocol is under-documented and the reported metrics are not internally consistent with the documented split.** Accuracy 94.8 / precision 81.3 / recall 96.2 jointly imply a test set with roughly a **4 : 1 negative-to-positive ratio**, while the documented 80/20 split of the 20,000 / 18,000 corpus yields ≈ 1.1 : 1. The exact evaluation set behind the published numbers can no longer be reconstructed from the repositories, so the figures should be treated as indicative, not reproducible.
4. **Train/serve mismatch.** The model was trained on complete utterances, but at serving time it scores *partial, streaming* transcripts, and the rule for aggregating utterance scores into a call-level alert was never formalized or evaluated.

**What a correct redesign looks like** — call-disjoint splits, call-level labels with weak supervision instead of inherited utterance labels, an explicit streaming/early-detection evaluation, and hard-negative testing against legitimate call-center dialogue — is exactly the design of **[EarShield](https://github.com/pmy02/earshield)** <!-- TODO: update URL -->.

---

## Detection UI

![image](https://github.com/pmy02/nOOnchi/assets/62882579/331c45e7-7dd2-4c9b-9430-1fdbad4998ec)

---

## Related Repositories

| Repository | Role |
|---|---|
| [KoBERT_test](https://github.com/SWMTeamCuriosity/KoBERT_test) | KoBERT fine-tuning notebook (final model) |
| [BiLSTM_RNN_Text_Classification](https://github.com/SWMTeamCuriosity/BiLSTM_RNN_Text_Classification) | BiLSTM/RNN baselines |
| [pretrained_fastText](https://github.com/SWMTeamCuriosity/pretrained_fastText) | fastText baseline |
| [transcript_data](https://github.com/SWMTeamCuriosity/transcript_data) | Transcripts and CSV training data |
| [text_refactoring](https://github.com/SWMTeamCuriosity/text_refactoring) / [text_preprocessing](https://github.com/SWMTeamCuriosity/text_preprocessing) / [normal_data_preprocess](https://github.com/SWMTeamCuriosity/normal_data_preprocess) / [MeCab_and_Stopword](https://github.com/SWMTeamCuriosity/MeCab_and_Stopword) | Data construction & preprocessing pipelines |
| [streaming_stt_test](https://github.com/SWMTeamCuriosity/streaming_stt_test) / [mic_input_test](https://github.com/SWMTeamCuriosity/mic_input_test) / [Kospeech_test](https://github.com/SWMTeamCuriosity/Kospeech_test) | STT / audio input experiments |
| [noonchi_api](https://github.com/SWMTeamCuriosity/noonchi_api) | Nest.js API server |

## Reproducibility

The training notebook (`KoBERT_test/KoBERT_Test.ipynb`, Colab-based, SKTBrain KoBERT + gluonnlp/mxnet stack) and the dataset CSVs (`transcript_data/csv_datas/result.csv`, 38,000 rows) are preserved as-is for the historical record. Re-running them reproduces the *flawed* protocol described above — by design they are kept unchanged; the corrected pipeline lives in the successor repository.

## Citation

```bibtex
@misc{park2022noonchi,
  author       = {Park, Minyoung},
  title        = {nOOnchi: Real-Time Voice Phishing Detection Service},
  year         = {2022},
  howpublished = {\url{https://github.com/pmy02/nOOnchi}},
  note         = {SW Maestro 13th program project}
}
```

## License & Contact

MIT License — see [LICENSE](LICENSE).
Maintained by **Park Minyoung** ([@pmy02](https://github.com/pmy02)). <!-- TODO: add a contact email if you want one public -->
