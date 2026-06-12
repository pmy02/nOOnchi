[English](README.md) | **한국어**

# nOOnchi (눈치) — 실시간 보이스피싱 탐지 서비스

![SW Maestro](https://img.shields.io/badge/SW%20Maestro-13th%20(2022)-blue)
![Model](https://img.shields.io/badge/Model-KoBERT-orange)
![PyTorch](https://img.shields.io/badge/PyTorch-1.x-EE4C2C?logo=pytorch&logoColor=white)
![Backend](https://img.shields.io/badge/Backend-Nest.js%20%C2%B7%20Kubernetes%20%C2%B7%20AWS%20SQS-green)
![License](https://img.shields.io/badge/License-MIT-lightgrey)

> **상태: 아카이브 (2022.04.08 – 2022.12.15).** 이 README는 2026년에 시스템을 *정확하게* 기록하기 위해 개정되었으며, 사후에 발견한 방법론적 한계까지 그대로 담고 있습니다. 이 한계들은 데이터 분할·라벨·평가를 처음부터 다시 설계한 후속 프로젝트 **[EarShield](https://github.com/pmy02/earshield)** <!-- TODO: 후속 리포 푸시 후 URL 갱신 --> 의 출발점이 되었습니다.

**눈치(nOOnchi)** 는 **소프트웨어 마에스트로 13기**에서 진행한 프로젝트로, 진행 중인 통화를 전사하고 KoBERT 기반 모델로 분류하여 보이스피싱으로 판단되면 사용자에게 알려주는 API 서비스이자 애플리케이션입니다. 매년 피해 규모가 커지는 보이스피싱을, 피해가 발생하기 *전에* 막아 공공의 이익에 기여하는 것이 목표였습니다.

![image](https://github.com/pmy02/nOOnchi/assets/62882579/d35d8313-555f-4e9e-9efc-f82e809e5a45)
![image](https://github.com/pmy02/nOOnchi/assets/62882579/7f5917b1-d259-4feb-a4b6-ee4bf9e618b7)

시스템은 KoBERT 기반 탐지 모델, Kubernetes와 SQS로 구축한 데이터 파이프라인, Nest.js API 서버로 구성됩니다. 모든 연구·프로토타입 리포지토리는 팀 조직 **[SWMTeamCuriosity](https://github.com/SWMTeamCuriosity)** 에 있습니다.

---

## 시스템 아키텍처

통화 음성이 백엔드로 스트리밍되면 스트리밍 STT 엔진이 실시간 전사하고, 전사 결과는 비동기 파이프라인을 거쳐 모델 서버에서 분류됩니다. 분류 결과는 클라이언트 앱으로 반환되어, 피싱 점수가 임계값을 넘으면 경고를 띄웁니다.

- **컨테이너 오케스트레이션:** Kubernetes 기반 컨테이너 아키텍처
- **비동기 처리:** Amazon SQS로 STT 출력과 모델 추론을 분리
- **API 서버:** Nest.js MVC 패턴

![image](https://github.com/pmy02/nOOnchi/assets/62882579/0dcd6f34-139b-4eed-a452-bead5ba00d7e)

![image](https://github.com/pmy02/nOOnchi/assets/62882579/d4abd062-f76a-444e-a3bd-52d53d04d7cf)

### STT 엔진 선정

보이스피싱 통화 음성에 대해 여러 한국어 STT API의 전사 정확도와 스트리밍 지원 여부를 비교했습니다.

![image](https://github.com/pmy02/nOOnchi/assets/62882579/99aa342e-8fde-4957-8781-74526f2ac66c)

**VITO STT**는 준수한 정확도를 유지하면서 실시간 스트리밍을 지원한다는 *기술적 근거*로 선정했습니다. (협의를 통한 20% 할인은 서비스 운영상의 비용 요인이었을 뿐, 기술적 선정 기준은 아닙니다.)

![image](https://github.com/pmy02/nOOnchi/assets/62882579/398fe901-96e9-4d90-a77e-f2bff884c3bf)

---

## 데이터

| 출처 | 원천 수집 | 학습 사용 |
|---|---|---|
| **피싱** — 금융감독원 [‘그놈 목소리’](https://www.fss.or.kr/fss/bbs/B0000206/list.do?menuNo=200690) / [‘바로 이 목소리’](https://www.fss.or.kr/fss/bbs/B0000203/list.do?menuNo=200686) (실제 사기 통화 공개 음성) | 546개 통화 크롤링 후 VITO STT 전사 | 발화 18,000건 (라벨 `True`) |
| **정상** — [AI Hub](https://aihub.or.kr/aihubdata/data/view.do?currMenu=116&topMenu=100&aihubDataSe=ty&dataSetSn=123) 한국어 대화 코퍼스 (금융 상담, 콜센터, 일상 대화) | 약 78,000건의 대화 레코드 | 발화 20,000건 (라벨 `False`) |

![image](https://github.com/pmy02/nOOnchi/assets/62882579/9d4da9b7-8ed4-4150-beef-aaad69b1fbff)

![image](https://github.com/pmy02/nOOnchi/assets/62882579/caba59e1-75ab-47b1-adff-e99f14092d41)

**38,000건 학습 데이터가 실제로 만들어진 방식** ([`text_refactoring/sentence_to_csv`](https://github.com/SWMTeamCuriosity/text_refactoring), [`transcript_data`](https://github.com/SWMTeamCuriosity/transcript_data) 참고): 피싱 통화 546건을 STT 발화 단위로 분할하고, 2자 초과 발화 중 앞에서부터 18,000건에 `True`를 부여했습니다. 정상 발화도 같은 방식으로 추출해 앞에서부터 20,000건에 `False`를 부여했습니다. 당시 기록에서는 이 단계를 "Augmentation"이라 표현했으나, 정확히는 **통화 단위 데이터를 발화 단위로 분할**한 것이며, 모든 발화는 자신이 속한 통화의 라벨을 그대로 상속합니다.

> AI Hub 원천 데이터는 라이선스상 재배포가 불가하며, 리포지토리에는 프로그램 기간 중 사용한 파생 파일만 포함되어 있습니다.

---

## 모델

**[KoBERT](https://github.com/SKTBrain/KoBERT)** (SKTBrain) 위에 표준적인 파인튜닝 구성을 얹었습니다. 입력 발화를 KoBERT의 **SentencePiece 서브워드 토크나이저**로 토큰화하고, 인코딩된 풀링 표현을 드롭아웃 + 선형 레이어에 통과시켜 이진 분류합니다.

| 하이퍼파라미터 | 값 ([`KoBERT_test`](https://github.com/SWMTeamCuriosity/KoBERT_test) 노트북 기준) |
|---|---|
| 최대 시퀀스 길이 | 64 |
| 배치 크기 | 64 |
| 옵티마이저 / 학습률 | AdamW / 1e-6 (워밍업 10%) |
| 드롭아웃 | 0.5 |
| 손실 함수 | CrossEntropyLoss (2-way) |
| 에폭 | 노트북 설정 기준 50 <!-- TODO: 배포 체크포인트는 6 에폭으로 보고됨; 실제 배포본 확인 필요 --> |
| 분할 | `train_test_split(test_size=0.2, random_state=0)` — **발화 단위** |

모델 선정 단계에서 [BiLSTM/RNN 분류기](https://github.com/SWMTeamCuriosity/BiLSTM_RNN_Text_Classification)와 [사전학습 fastText](https://github.com/SWMTeamCuriosity/pretrained_fastText)도 프로토타이핑했으며, 팀 검증 데이터에서는 KoBERT가 가장 우수했습니다.

![image](https://github.com/pmy02/nOOnchi/assets/62882579/df63ff97-ad6d-47ea-9001-c321c034a988)

---

## 보고된 결과

홀드아웃 발화에 대한 혼동 행렬 기반 지표:

| Accuracy | Precision | Recall | F1 |
|:---:|:---:|:---:|:---:|
| 94.8% | 81.3% | 96.2% | 88.1% |

재현율을 우선시했습니다. 피싱 경고 시스템에서는 놓친 피싱 통화(거짓 음성)의 비용이 오경보보다 훨씬 크기 때문입니다.

![image](https://github.com/pmy02/nOOnchi/assets/62882579/ef0394b9-4728-46cd-bbd4-0fcff9052583)

### ⚠️ 한계와 회고 (수치를 인용하기 전에 반드시 읽어주세요)

프로그램 종료 후 코드와 데이터를 재감사하면서 네 가지 방법론적 문제를 확인했습니다. 후속 프로젝트의 설계 근거가 되었기에 솔직하게 기록합니다.

1. **동일 통화 누수(leakage)로 지표가 부풀려졌습니다.** 학습/평가 분할이 *발화 단위 무작위* 분할이어서, *같은 피싱 통화*에서 나온 — 서로 거의 중복인 — 발화들이 양쪽에 모두 존재합니다. 따라서 보고된 수치는 새로운 통화에 대한 일반화보다 이미 본 통화의 재인식에 가깝습니다. 올바른 프로토콜은 **통화 단위로 분리된(call-disjoint)** 그룹 분할입니다.
2. **라벨 상속 노이즈.** 모든 발화가 통화 라벨을 상속하므로, 피싱 통화 속 내용 없는 조각들(예: "아파서", "다름이 아니라")까지 `True`로 라벨됩니다. 모델이 중립적 추임새를 "피싱"이라 부르도록 일부 학습되어 실제 트래픽에서 정밀도를 해칩니다.
3. **평가 프로토콜이 문서화되어 있지 않고, 보고 지표가 문서화된 분할과 내적으로 일치하지 않습니다.** Accuracy 94.8 / Precision 81.3 / Recall 96.2 를 동시에 만족하려면 테스트셋의 음성:양성 비율이 약 **4 : 1**이어야 하지만, 문서화된 20,000 / 18,000 코퍼스의 80/20 분할은 약 1.1 : 1 입니다. 공개 수치가 산출된 평가셋은 현재 리포지토리에서 재구성할 수 없으므로, 해당 수치는 재현 가능한 결과가 아닌 참고치로 취급해야 합니다.
4. **학습-서빙 불일치.** 모델은 완결된 발화로 학습되었지만, 서빙 시에는 *부분적인 스트리밍* 전사를 채점하며, 발화 점수를 통화 수준 경고로 집계하는 규칙은 정식화·평가되지 않았습니다.

**올바른 재설계** — 통화 단위 분할, 발화 라벨 상속 대신 통화 수준 라벨의 약지도 학습, 명시적인 스트리밍/조기 탐지 평가, 정상 콜센터 대화에 대한 하드 네거티브 테스트 — 가 바로 **[EarShield](https://github.com/pmy02/earshield)** <!-- TODO: URL 갱신 --> 의 설계입니다.

---

## 탐지 화면

![image](https://github.com/pmy02/nOOnchi/assets/62882579/331c45e7-7dd2-4c9b-9430-1fdbad4998ec)

---

## 관련 리포지토리

| 리포지토리 | 역할 |
|---|---|
| [KoBERT_test](https://github.com/SWMTeamCuriosity/KoBERT_test) | KoBERT 파인튜닝 노트북 (최종 모델) |
| [BiLSTM_RNN_Text_Classification](https://github.com/SWMTeamCuriosity/BiLSTM_RNN_Text_Classification) | BiLSTM/RNN 베이스라인 |
| [pretrained_fastText](https://github.com/SWMTeamCuriosity/pretrained_fastText) | fastText 베이스라인 |
| [transcript_data](https://github.com/SWMTeamCuriosity/transcript_data) | 전사 데이터 및 CSV 학습 데이터 |
| [text_refactoring](https://github.com/SWMTeamCuriosity/text_refactoring) / [text_preprocessing](https://github.com/SWMTeamCuriosity/text_preprocessing) / [normal_data_preprocess](https://github.com/SWMTeamCuriosity/normal_data_preprocess) / [MeCab_and_Stopword](https://github.com/SWMTeamCuriosity/MeCab_and_Stopword) | 데이터 구축·전처리 파이프라인 |
| [streaming_stt_test](https://github.com/SWMTeamCuriosity/streaming_stt_test) / [mic_input_test](https://github.com/SWMTeamCuriosity/mic_input_test) / [Kospeech_test](https://github.com/SWMTeamCuriosity/Kospeech_test) | STT / 오디오 입력 실험 |
| [noonchi_api](https://github.com/SWMTeamCuriosity/noonchi_api) | Nest.js API 서버 |

## 재현성

학습 노트북(`KoBERT_test/KoBERT_Test.ipynb`, Colab 기반, SKTBrain KoBERT + gluonnlp/mxnet 스택)과 데이터셋 CSV(`transcript_data/csv_datas/result.csv`, 38,000행)는 기록 보존을 위해 그대로 유지됩니다. 이를 재실행하면 위에 기술한 *결함 있는* 프로토콜이 재현됩니다 — 의도적으로 수정하지 않았으며, 교정된 파이프라인은 후속 리포지토리에 있습니다.

## 인용

```bibtex
@misc{park2022noonchi,
  author       = {Park, Minyoung},
  title        = {nOOnchi: Real-Time Voice Phishing Detection Service},
  year         = {2022},
  howpublished = {\url{https://github.com/pmy02/nOOnchi}},
  note         = {SW Maestro 13th program project}
}
```

## 라이선스 및 연락처

MIT License — [LICENSE](LICENSE) 참고.
관리자: **박민영** ([@pmy02](https://github.com/pmy02)). <!-- TODO: 공개할 이메일이 있으면 추가 -->
