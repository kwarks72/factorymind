# FactoryMind · 설비 고장 대응 AI 도우미

> **Java(Spring Boot) 업무시스템 + RAG(검색 증강 생성)** 으로 만든 제조 AI 웹 애플리케이션
> 설비에서 알람이 나면, AI가 **설비 매뉴얼과 과거 수리기록을 검색해서 근거(출처)와 함께** 조치 방법을 알려줍니다.

- 개발: 곽수연 · sooyeon0702@naver.com · [github.com/kwarks72](https://github.com/kwarks72)
- 기술: Java 17, Spring Boot 3.5, Spring Data JPA, JdbcTemplate, PostgreSQL 16 + **pgvector**, Gemini API(LLM·임베딩), Thymeleaf, Chart.js, Docker

---

## 1. 왜 만들었나 (채용공고와 연결)

| 공고 요구사항 | 이 프로젝트에서 구현한 것 |
|---|---|
| 산업용 제조 AI 플랫폼의 Web Application 개발 | 설비 대시보드·알람·수리기록·지식베이스·AI 도우미 5개 화면의 웹 앱 |
| 생성형 AI와 **Java 기반 업무시스템** 연계 | Spring Boot 업무시스템(설비/알람/수리기록) 안에서 LLM 호출. **수리기록을 저장하면 자동으로 AI 지식베이스에 추가** |
| **RAG 기반 문서·지식 검색** 및 AI 응답 연계 | 문서 조각내기(Chunking) → 임베딩 → pgvector 저장 → 벡터 검색 → 근거 번호를 단 LLM 답변 |
| Cloud 환경 Application 개발 | Docker Compose로 패키징, 클라우드 VM 배포 가이드 포함 |
| **Vector DB** 관련 기술 | PostgreSQL + pgvector, 코사인 거리 검색, HNSW 인덱스 |
| 산업용 데이터 / **Digital Twin** | 가상 센서(온도·진동) 시뮬레이터 + 실시간 설비 상태 대시보드(미니 디지털 트윈) |

## 2. 시연 시나리오 (2분)

1. **대시보드**: 설비 4대의 온도·진동이 3초마다 움직입니다.
2. "사출기 2호기" 카드를 누르고 **[과열 고장 발생시키기]** → 온도가 치솟아 카드가 빨갛게 바뀌고, `E-203 실린더/금형 과열` **알람이 자동 생성**됩니다.
3. 알람 상세 → **[AI에게 물어보기]** → AI가 매뉴얼의 E-203 조치 절차와 **"사출기 1호기의 지난달 수리 사례(냉각수 필터 막힘)"** 를 찾아 근거 번호와 함께 답합니다.
4. 조치 후 **수리기록 저장** → 알람 종료 + 이 기록이 지식베이스에 자동 추가 → 다음 고장 때 AI가 참고합니다. (**지식이 쌓이는 업무시스템**)

## 3. 구조

```
[가상 센서 시뮬레이터] ─3초마다 온도/진동─▶ ┌──────────── Spring Boot ────────────┐
   (실제 공장이라면 PLC·IoT 센서)            │ 업무시스템(JPA)   │ RAG(JdbcTemplate) │
                                            │ 설비·알람·수리기록 │ 조각내기·임베딩·검색│
[웹 화면(Thymeleaf + JS)] ◀──REST/HTML──▶   └─────────┬──────────┴────────┬────────┘
 대시보드 · 알람 · AI 도우미 · 지식베이스              │                   │
                                          ┌──────────▼───────────┐   ┌───▼────────────┐
                                          │ PostgreSQL + pgvector│   │ Gemini API     │
                                          │ 업무 테이블 + 벡터 색인│   │ 임베딩 · LLM   │
                                          └──────────────────────┘   └────────────────┘
```

### RAG 흐름 (`RagService.java`)
```
질문 ──① 임베딩──▶ 질문 벡터 ──② pgvector 검색(코사인 거리)──▶ 관련 조각 4개
                                                         │
     답변 + 출처 ◀──③ LLM 생성 (프롬프트: "아래 [자료]만 근거로, 근거 번호를 달아 답하라")
```

### DB 테이블
| 테이블 | 만든 방법 | 내용 |
|---|---|---|
| `equipment` | JPA | 설비 (이름, 모델, 위치, 상태, 최근 센서값) |
| `sensor_reading` | JPA | 센서 측정값 (2시간 지난 값은 자동 삭제) |
| `alarm` | JPA | 알람 (코드, 측정값/기준값, 상태) |
| `repair_log` | JPA | 수리기록 (원인, 조치, 작업자) |
| `kb_document` | SQL | 지식베이스 원문 (매뉴얼 / 수리기록) |
| `kb_chunk` | SQL | 문서 조각 + `embedding vector(768)` + HNSW 인덱스 |

## 4. 실행 방법 (Windows)

### 준비물: Docker Desktop 하나만 있으면 됩니다
Java나 PostgreSQL을 따로 설치할 필요가 없습니다. Docker가 "필요한 프로그램이 다 들어 있는 상자"를 만들어 실행해 줍니다.

1. [Docker Desktop](https://www.docker.com/products/docker-desktop/) 설치 → 실행 (처음이면 WSL2 설치 안내가 나오면 따라 하고 재부팅)
2. 이 폴더(`factorymind`)를 VS Code로 열기
3. VS Code 메뉴 **터미널 → 새 터미널** 을 열고 입력:

```powershell
copy .env.example .env
docker compose up --build
```

4. 처음에는 라이브러리 다운로드 때문에 3~5분 걸립니다. 로그에 `FactoryMind 준비 완료` 가 보이면
   브라우저에서 **http://localhost:8080** 접속
5. 끄기: 터미널에서 `Ctrl + C`, 완전히 정리하려면 `docker compose down`
   (DB까지 초기화하려면 `docker compose down -v`)

### AI 모드 2가지
| 모드 | 조건 | 동작 |
|---|---|---|
| **데모 모드** (기본) | `.env`의 `GEMINI_API_KEY`가 비어 있음 | 무료·인터넷 불필요. 해싱 기반 임베딩으로 벡터 검색을 하고, 찾은 자료를 정리해 보여줌 |
| **Gemini 모드** | `GEMINI_API_KEY` 입력 | 진짜 임베딩(gemini-embedding-001) + LLM(gemini-2.5-flash)이 근거를 달아 문장으로 답변 |

Gemini 키 넣는 법:
1. https://aistudio.google.com/apikey 에서 구글 계정으로 로그인 → **Create API key**
2. `.env` 파일의 `GEMINI_API_KEY=` 뒤에 붙여넣고 저장
3. `docker compose up --build` 다시 실행 → 기존 문서가 **자동으로 Gemini 임베딩으로 재색인**됩니다

> 데모 모드의 한계(면접에서 설명하기 좋은 포인트): 데모 임베딩은 같은 글자를 공유해야 찾을 수 있어서 "떨려요"와 "진동"을 연결하지 못합니다. Gemini 임베딩은 **뜻**으로 비교하기 때문에 연결합니다. 이것이 의미 기반(시맨틱) 검색이 필요한 이유입니다.

## 5. 폴더 구조

```
factorymind/
├─ docker-compose.yml          DB(pgvector) + 앱 함께 실행
├─ Dockerfile                  Java 앱을 상자(이미지)로 만드는 방법
├─ .env.example                API 키·비밀번호 설정 견본
└─ src/main/
   ├─ java/com/factorymind/
   │  ├─ domain/               설비·센서값·알람·수리기록 (JPA 엔티티)
   │  ├─ repository/           DB 조회 (Spring Data JPA)
   │  ├─ sensor/SensorSimulator.java   가상 센서 + 알람 자동 생성 (디지털 트윈 데이터)
   │  ├─ service/AlarmService.java     알람·수리 업무 로직 + 수리기록 → 지식베이스 연계
   │  ├─ rag/
   │  │  ├─ TextChunker.java           문서 조각내기
   │  │  ├─ DemoEmbedder.java          무료 데모 임베딩
   │  │  ├─ GeminiEmbedder.java        Gemini 임베딩 API
   │  │  ├─ GeminiGenerator.java       Gemini LLM 답변 (프롬프트)
   │  │  ├─ KnowledgeBaseService.java  pgvector 저장·검색·재색인
   │  │  └─ RagService.java            RAG 전체 흐름
   │  ├─ web/                  화면 컨트롤러 + REST API
   │  └─ config/DataInitializer.java   시연용 설비·매뉴얼·과거 수리기록 자동 등록
   └─ resources/
      ├─ application.yml       설정
      ├─ manuals/              가상 설비 매뉴얼 3종 (직접 작성)
      ├─ templates/            화면 HTML (Thymeleaf)
      └─ static/               CSS, JavaScript(대시보드 그래프, AI 답변 표시)
```

## 6. 설계하면서 고민한 점 (면접 대비)

**① 왜 문서를 "## 섹션" 단위로 잘랐나?**
처음엔 소제목(### 주요 원인 / ### 조치 순서)마다 잘랐더니, "E-203 조치 방법"을 물었을 때 *원인* 조각만 찾고 *조치 순서* 조각은 놓치는 문제가 있었습니다. 알람 하나의 원인·조치·주의가 한 조각에 들어가도록 섹션 단위로 바꾸고, 조각마다 `[문서 > 섹션 제목]` 머리말을 붙여 검색 정확도를 높였습니다.

**② 환각(지어내기)을 어떻게 줄였나?**
프롬프트에서 "제공한 [자료]만 근거로 답하고, 없으면 '자료에 없음'이라고 말하라", "문장마다 근거 번호 [1]을 달아라"라고 지시했습니다. 화면에는 근거 조각 원문과 유사도를 함께 보여줘서 작업자가 직접 확인할 수 있습니다. temperature도 0.2로 낮췄습니다.

**③ 왜 별도 벡터 DB가 아니라 PostgreSQL + pgvector인가?**
업무 데이터(설비·알람)와 지식 데이터를 한 DB에서 관리하면 운영이 단순하고, 무료 클라우드 VM 한 대에서도 돌아갑니다. 데이터가 많아지면 HNSW 인덱스로 검색 속도를 유지합니다.

**④ 업무시스템과 AI를 어떻게 "연계"했나?**
`AlarmService.repair()`에서 수리기록을 저장하는 순간 지식베이스에 자동으로 추가됩니다. AI 색인에 실패해도 업무(수리기록 저장)는 성공하도록 예외를 분리했습니다. 또 임베딩 방식이 바뀌면(데모 → Gemini) 앱 시작 시 자동으로 재색인합니다.

**⑤ 디지털 트윈 데이터는?**
실제 공장의 PLC·센서 대신 `SensorSimulator`가 3초마다 온도·진동을 만들고, 기준값을 넘으면 알람을 만듭니다. 설비 모델마다 정상값·기준값·알람 코드를 정의했고, 이 알람 코드는 매뉴얼의 "알람 코드표"와 일치시켰습니다.

## 7. 클라우드 배포 (Oracle Cloud / Google Cloud 무료 VM, Ubuntu)

```bash
# 1) Docker 설치
curl -fsSL https://get.docker.com | sudo sh
sudo usermod -aG docker $USER   # 적용하려면 로그아웃 후 다시 접속

# 2) 메모리 1GB VM이면 스왑(가상 메모리) 2GB 추가 - 빌드 중 멈춤 방지
sudo fallocate -l 2G /swapfile && sudo chmod 600 /swapfile
sudo mkswap /swapfile && sudo swapon /swapfile
echo '/swapfile none swap sw 0 0' | sudo tee -a /etc/fstab

# 3) 코드 받기 & 실행
git clone https://github.com/kwarks72/factorymind.git
cd factorymind
cp .env.example .env && nano .env      # GEMINI_API_KEY, DB_PASSWORD 입력
docker compose up -d --build

# 4) 로그 보기
docker compose logs -f app
```
- 클라우드 콘솔의 방화벽(보안 목록)에서 **8080 포트 인바운드 허용**, Ubuntu 내부 방화벽도 열기
  (`sudo iptables -I INPUT -p tcp --dport 8080 -j ACCEPT`)
- 접속: `http://서버공인IP:8080`
- 공개 서버에서는 `docker-compose.yml`의 DB 포트(`5433:5432`) 줄을 지워 DB를 외부에 노출하지 마세요.

## 8. 다음 단계 (확장 아이디어)

- **Spring AI** 로 RAG 파이프라인 교체 (현재는 원리를 이해하기 위해 직접 구현)
- **Knowledge Graph**: Neo4j로 `설비 → 부품 → 고장유형 → 조치` 관계를 저장하고, 벡터 검색 결과와 함께 LLM에 제공 (GraphRAG)
- **하이브리드 검색**: 벡터 검색 + 키워드(알람 코드) 검색 점수 결합
- PDF 매뉴얼 업로드, 로그인·권한(Spring Security), 질문/답변 이력 저장 및 피드백(👍/👎)

## 9. 테스트

`src/test/java/com/factorymind/rag/RagCoreTest.java` — 조각내기, 임베딩 차원·정규화, 같은 알람 코드끼리의 유사도, pgvector 문자열 변환을 검사합니다. Docker 빌드(`mvn package`) 때 자동 실행됩니다.

---
※ `manuals/` 의 설비 매뉴얼과 수리기록은 포트폴리오 시연을 위해 직접 작성한 **가상의 자료**입니다.
