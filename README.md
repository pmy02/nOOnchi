# nOOnchi
- 프로젝트 소개 : 보이스피싱 탐지 솔루션
- 프로젝트 기간 : 2022.04.08 ~ 2022.12.15 


# 프로젝트 설명

![image](https://github.com/pmy02/nOOnchi/assets/62882579/d35d8313-555f-4e9e-9efc-f82e809e5a45)
![image](https://github.com/pmy02/nOOnchi/assets/62882579/7f5917b1-d259-4feb-a4b6-ee4bf9e618b7)

<strong>눈치(nOOnchi)</strong>는 <strong>소프트웨어 마에스트로 13기</strong>에서 진행한 프로젝트로, 통화 내용을 기반으로 보이스피싱 여부를 판별하고, 이를 사용자에게 알려주는 API 서비스이자 Application입니다. <br>
매년 피해와 규모가 증가하고 있는 보이스피싱을 사전에 예방하여 공공의 이익을 실천하고자 하였습니다. <br><br>
 
KoBERT 기반 탐지 모델, Kubernetes와 SQS를 활용한 아키텍처로 구축한 데이터 파이프라인, Nest.js를 사용한 API 서버가 있습니다. <br>
[링크](https://github.com/SWMTeamCuriosity)


# 개발 내용 - 모델

금융감독원의 [‘그놈 목소리’](https://www.fss.or.kr/fss/bbs/B0000206/list.do?menuNo=200690)와 [‘바로 이 목소리’](https://www.fss.or.kr/fss/bbs/B0000203/list.do?menuNo=200686) 서비스에서 제공하는 546개의 보이스피싱 데이터를 크롤링하여 수집하였습니다. <br><br>
![image](https://github.com/pmy02/nOOnchi/assets/62882579/9d4da9b7-8ed4-4150-beef-aaad69b1fbff) <br>

[AI Hub](https://aihub.or.kr/aihubdata/data/view.do?currMenu=116&topMenu=100&aihubDataSe=ty&dataSetSn=123)에서 일반 대화 데이터를 수집하였습니다. 금융 상담을 비롯한 일상 대화 내용 78,000건을 수집하였습니다. <br><br>
![image](https://github.com/pmy02/nOOnchi/assets/62882579/caba59e1-75ab-47b1-adff-e99f14092d41) <br>

보이스피싱 데이터를 텍스트로 변환하기 위해 여러 종류의 STT API를 비교 분석한 결과, 가장 적합한 STT API를 선정하였습니다. <br> 

아래는 STT API들의 비교 결과를 나타낸 표입니다. <br>

![image](https://github.com/pmy02/nOOnchi/assets/62882579/99aa342e-8fde-4957-8781-74526f2ac66c)

위 표를 통해 VITO STT 서비스가 준수한 정확도를 유지하면서도 실시간 스트리밍이 가능함을 확인할 수 있었습니다. <br>
또한, VITO 측과 협의를 통해 해당 서비스를 20% 저렴한 가격으로 이용할 수 있게 되어, 최종적으로 VITO STT를 선택하게 되었습니다. <br>

![image](https://github.com/pmy02/nOOnchi/assets/62882579/573db6a6-ec65-464a-a196-5de3d8bf506b)

모델 선정을 위해 다양한 모델을 테스트하였으며, 그 결과 BERT 기반의 KoBERT 모델이 가장 우수한 성능을 보였습니다. 또한, 보이스피싱 데이터가 부족한 문제를 Augmentation 기법을 활용하여 해결하였습니다. 
이를 바탕으로, 각 단어를 토큰화한 후 KoBERT와 Binary Classification 모델을 연결하여 보이스피싱 여부를 판별할 수 있는 모델을 설계하였습니다.

![image](https://github.com/pmy02/nOOnchi/assets/62882579/df63ff97-ad6d-47ea-9001-c321c034a988)

모델 학습은 총 6 Epoch로 진행되었고, Confusion matrix에서 나온 값들을 통해 모델 성능 지표를 확인해 보았습니다. <br>
이에 따르면, 모델의 정확도는 94.8%, 정밀도는 81.3%, 재현율은 96.2%, f1 score는 88.1%로 측정되었습니다.

이 결과는 보이스피싱 여부를 판별하는데 중요한 재현율이 높은 신뢰도 높은 모델이라는 것을 나타냅니다.

![image](https://github.com/pmy02/nOOnchi/assets/62882579/ef0394b9-4728-46cd-bbd4-0fcff9052583)


# 탐지 화면

![image](https://github.com/pmy02/nOOnchi/assets/62882579/331c45e7-7dd2-4c9b-9430-1fdbad4998ec)


# 개발 내용 - 서버

Kubernetes를 기반의 컨테이너 아키텍처 설계 <br>
Amazon SQS를 이용한 비동기식 데이터 처리

![image](https://github.com/pmy02/nOOnchi/assets/62882579/0dcd6f34-139b-4eed-a452-bead5ba00d7e)

Nest.js 를 이용한 MVC 패턴 구현

![image](https://github.com/pmy02/nOOnchi/assets/62882579/d4abd062-f76a-444e-a3bd-52d53d04d7cf)
