<p align="center">
  <img src="https://github.com/team-agent-5/marieum-app/raw/a12aa847efa5794825da9e6e61cc05c48be30381/assets/branding/marieum_logo.png" width="120" alt="마리음 로고" />
</p>

<h1 align="center">마리음 · Marieum</h1>
<p align="center"><strong>당신의 말을 더 또렷하게 이어주는 AI 기반 음성 AAC 서비스</strong></p>
<p align="center">구음장애가 있는 사용자의 발화를 텍스트로 확인하고,<br/>상황에 맞는 문장을 선택해 음성으로 전달합니다.</p>

<p align="center">
  <a href="#서비스-소개">서비스 소개</a> ·
  <a href="#서비스-아키텍처">아키텍처</a> ·
  <a href="#파트별-역할과-책임">파트별 역할</a> ·
  <a href="#repositories">저장소</a>
</p>

---

## 서비스 소개

**마리음은 구음장애인의 일상적인 의사소통을 돕는 보완대체의사소통(AAC) 서비스입니다.**

발음이 명확하지 않아 같은 말을 반복해야 하는 순간, 하고 싶은 말을 문장판에서 찾는 데 시간이 걸리는 순간을 줄이고자 합니다. 사용자의 발화를 출발점으로 **음성 인식 → 내용 확인·문장 추천 → 음성 전달**을 연결합니다.

AI가 사용자의 의도를 대신 확정하기보다, 인식한 내용과 추천 문장을 확인하고 선택할 수 있는 경험을 지향합니다. 정확도 모드와 실시간 모드는 서로 다른 대화 상황에 맞춘 두 가지 사용 흐름입니다.

### 두 가지 대화 방식

| | 정확도 모드 | 실시간 모드 |
| :--- | :--- | :--- |
| 사용 상황 | 한 문장씩 확인하며 전달하고 싶을 때 | 대화의 흐름을 이어가고 싶을 때 |
| 입력 | 버튼을 누르는 동안 녹음 후 WAV 업로드 | 마이크 음성을 WebSocket으로 연속 전송 |
| 처리 | 전사 후 서버 응답에 따라 원문·추천·후보·재발화 안내 표시 | 중간 전사와 발화 구간별 최종 전사 표시 |
| 전달 | 사용자가 확인·선택한 문장을 TTS로 재생 | 확정 문장을 재생 큐에 넣어 순차적으로 전달 |

> ‘정확도 모드’는 확인·추천 중심의 사용 흐름을 가리키는 이름이며, 실시간 모드보다 높은 인식 정확도를 보장한다는 뜻은 아닙니다.

### 대화를 돕는 기능

- **상황에 맞는 문장 추천:** 인식 결과와 개인화 문맥을 바탕으로 추천 문구와 이유를 제공합니다.
- **빠른 문장:** 자주 쓰는 문장을 저장하고 한 번 눌러 재생합니다.
- **요청 기록과 개인화 현황:** 이전 문장을 다시 전달하고 사용 기록을 확인합니다.
- **사용자에게 맞춘 화면·음성:** 큰 글자, 한국어 목소리, 재생 속도와 음량을 설정합니다.
- **모바일 음성 경험:** Android·iOS의 마이크 권한, 녹음·재생 상태, 인증 만료와 재시도를 처리합니다.

### 서비스 화면

<table>
  <tr>
    <th align="center">말하기</th>
    <th align="center">추천 문장 선택</th>
    <th align="center">실시간 대화</th>
  </tr>
  <tr>
    <td align="center"><img src="https://github.com/team-agent-5/marieum-app/raw/a12aa847efa5794825da9e6e61cc05c48be30381/assets/readme/speak-precise.png" width="230" alt="마리음 정확도 모드 말하기 화면" /></td>
    <td align="center"><img src="https://github.com/team-agent-5/marieum-app/raw/a12aa847efa5794825da9e6e61cc05c48be30381/assets/readme/precise-candidates.png" width="230" alt="마리음 추천 후보 선택 화면" /></td>
    <td align="center"><img src="https://github.com/team-agent-5/marieum-app/raw/a12aa847efa5794825da9e6e61cc05c48be30381/assets/readme/live-conversation.png" width="230" alt="마리음 실시간 대화 화면" /></td>
  </tr>
</table>

<sub>App 저장소에 공개된 iOS 시뮬레이터 캡처입니다. 캡처 시점과 최신 구현 사이에 문구·배치 차이가 있을 수 있습니다.</sub>

## 서비스 아키텍처

마리음은 **모바일 경험**, **서비스 API·개인화 추천**, **GPU 음성 인식**을 분리합니다. 앱은 백엔드를 통해 음성 인식 서버와 통신하고, 추천·기록·인증은 백엔드가 담당합니다.

```mermaid
flowchart LR
    U[사용자] -->|발화·문장 선택| APP["Mobile App<br/>Flutter · Android / iOS"]
    APP -->|HTTPS · WebSocket| API["Service Backend<br/>FastAPI"]
    API -->|음성 업로드·스트리밍 중계| ASR["ASR Inference<br/>Whisper · Silero VAD<br/>PyTorch · NVIDIA L4"]
    ASR -->|전사·신뢰도| API
    API -->|개인화 문맥 검색·기록| DB[("PostgreSQL<br/>pgvector")]
    API -->|문장 추천 요청| LLM["Recommendation<br/>PydanticAI · Claude"]
    API -->|전사·추천 결과| APP
    APP -->|기기 TTS / 준비된 음성 캐시| OUT[상대방에게 음성 전달]
```

### 1. 정확도 모드: 확인하고 선택하는 흐름

1. **앱**이 PCM 음성을 메모리에 모아 WAV로 구성하고 `POST /phrase/transcribe`로 보냅니다.
2. **백엔드**가 ASR 서버에 전사를 요청하고, 전사 결과·신뢰도·추적 ID를 앱에 전달합니다.
3. 앱이 `POST /phrase/suggest`를 호출하면 **추천 계층**이 신뢰도와 개인화 문맥에 따라 응답을 구성합니다.
4. 앱은 `proactive / candidates / single / retry` 응답에 맞춰 화면과 재생을 처리합니다. 신뢰도 임계값을 앱에서 다시 계산하지 않습니다.
5. 선택한 문장을 TTS로 전달하고 `POST /phrase/selection`으로 선택·거부 기록을 보냅니다.

### 2. 실시간 모드: 대화가 이어지는 흐름

1. **앱**이 PCM16·16 kHz·mono 음성을 100 ms 프레임으로 `/ws/phrase/stream`에 전송합니다.
2. **백엔드**가 별도 GPU ASR 서버로 스트리밍을 중계합니다.
3. **ASR 서버**가 VAD와 반복 전사를 이용해 중간 결과와 발화 구간별 최종 결과를 반환합니다.
4. **앱**이 중간 결과를 표시하고 확정 문장을 순차 재생합니다. 사용 가능한 음성 캐시가 없으면 기기 TTS를 사용합니다.
5. 재생 후 `POST /phrase/bypass`로 기록합니다. 기록 전송이 다음 음성 재생을 막지 않도록 분리합니다.

> 실시간 기본 경로는 ASR 결과의 중계·표시·음성 전달입니다. 모든 실시간 발화가 LLM 문장 추천이나 교정을 거치는 구조는 아닙니다.

## 파트별 역할과 책임

기술을 나열하는 데 그치지 않고, **각 파트가 어떤 문제를 해결하고 어디까지 책임지는지**를 기준으로 구분합니다.

| 파트 | 해결하는 문제와 책임 | 주요 구현·산출물 |
| :--- | :--- | :--- |
| **서비스 기획 · UX** | 구음장애 사용자의 의사소통 흐름을 정의하고, 확인 중심 대화와 실시간 대화를 구분 | 모드별 사용자 흐름, 큰 말하기 버튼·후보 선택·재시도 안내, 접근성을 고려한 화면 구성 |
| **모바일 앱** | 녹음부터 문장 확인·선택·음성 재생까지의 사용자 경험을 안정적으로 연결 | Flutter UI, BLoC/Cubit 상태 관리, HTTP·WebSocket 연결, 플랫폼별 오디오 처리, TTS, 보안 토큰 저장 |
| **백엔드 · 서비스 연동** | 앱과 AI 서버 사이의 계약을 관리하고 인증·기록·설정 기능 제공 | FastAPI API, JWT 인증, ASR HTTP·WebSocket 프록시, 데이터 저장, 요청 추적 |
| **개인화 추천** | 인식 결과만으로 부족한 문맥을 보완하고 선택 가능한 문장 제공 | pgvector 기반 문맥 검색, PydanticAI·Claude 추천, 응답 모드 분기, 선택 피드백과 실패 시 대체 응답 |
| **AI · 음성 인식** | 구음장애 발화를 텍스트로 변환하고 대화 중 전사 결과 제공 | 구음장애 특화 Whisper 모델 연동, Silero VAD, GPU 추론, 중간·최종 전사, 배치 처리와 추론 검증 |

<sub>위 표는 시스템의 책임 영역입니다. 개인별 담당이나 기여 규모를 임의로 배정하지 않으며, 구체적인 구현 이력은 각 저장소의 커밋·PR에서 확인할 수 있습니다.</sub>

### 함께 맞춘 설계 기준

- **음성 인식과 추천의 분리:** ASR은 음성→텍스트를, 백엔드는 문맥 검색·추천·기록을 담당합니다.
- **명확한 API 계약:** 앱이 서버의 응답 모드를 해석해 원문·후보·추천·재시도 화면을 구성합니다.
- **음성 전달 우선:** 재생과 로그 전송을 분리하고, 실시간 모드에서 새 클라우드 합성을 기다리지 않습니다.
- **모델 출처와 검증 범위 구분:** 공개 AI 서버는 공개된 구음장애 특화 Whisper 파인튜닝 모델을 사용하는 추론·서빙 구현입니다. 이 서버의 구현을 팀의 자체 모델 학습 성과로 표현하지 않습니다.

## Repositories

| 저장소 | 담당 영역 | 주요 기술·문서 |
| :--- | :--- | :--- |
| [marieum-app](https://github.com/team-agent-5/marieum-app) | Android·iOS 앱, 녹음·확인·선택·음성 전달 | Flutter · Dart · BLoC/Cubit · 네이티브 오디오 / [실행 및 화면 안내](https://github.com/team-agent-5/marieum-app#readme) |
| [marieum-backend](https://github.com/team-agent-5/marieum-backend) | 인증·기록·설정, ASR 중계, 개인화 추천 | Python · FastAPI · PostgreSQL/pgvector · PydanticAI · Claude / [서버 안내](https://github.com/team-agent-5/marieum-backend#readme) |
| [marieum-ai](https://github.com/team-agent-5/marieum-ai) | 구음장애 음성 인식, 실시간 GPU 추론·서빙 | Whisper · PyTorch · Transformers · Silero VAD / [역할별 연동 가이드](https://github.com/team-agent-5/marieum-ai/blob/main/TEAM_GUIDE.md) |

로컬 실행·환경변수·상세 API 계약은 각 저장소의 문서를 참고해 주세요. 이 페이지는 서비스 전체의 구조와 책임 경계를 소개합니다.

<details>
<summary>구현 근거와 검증 범위</summary>

- [App README](https://github.com/team-agent-5/marieum-app/blob/a12aa847efa5794825da9e6e61cc05c48be30381/README.md): 화면, 정확도·실시간 통신 흐름, TTS와 기록 분리.
- [Backend API](https://github.com/team-agent-5/marieum-backend/blob/0183d995817bc2249ed50bd256401632b603a0bf/src/app/api/phrase.py) · [추천 오케스트레이터](https://github.com/team-agent-5/marieum-backend/blob/0183d995817bc2249ed50bd256401632b603a0bf/src/app/agent/orchestrator.py): 전사·추천·선택·bypass와 개인화 추천 경계.
- [실시간 파이프라인](https://github.com/team-agent-5/marieum-backend/blob/0183d995817bc2249ed50bd256401632b603a0bf/src/app/agent/streaming/pipeline.py): 기본 실시간 중계 경로.
- [AI 연동 가이드](https://github.com/team-agent-5/marieum-ai/blob/2ccebf6627259e6260d17beab6de96889a67f72b/TEAM_GUIDE.md) · [검증 결과](https://github.com/team-agent-5/marieum-ai/blob/2ccebf6627259e6260d17beab6de96889a67f72b/TEST_RESULTS.md): 모델 출처, 추론 방식과 측정 조건.

공개 저장소의 확인된 구현을 기준으로 작성했습니다. 현재 운영 상태, 앱스토어 배포 여부, 의료적 효능이나 특정 정확도·지연 시간을 보장하지 않습니다.

</details>
