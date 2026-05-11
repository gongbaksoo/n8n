# n8n 뉴스 스크래퍼 워크플로우 인수인계서

## 1. 프로젝트 개요

Google News RSS를 통해 키워드별 뉴스를 자동 수집하고, AI 요약 후 Notion + 이메일로 발송하는 n8n 워크플로우.

### 원본 워크플로우
- **n8n 인스턴스**: `https://n8n.gongbaksoo.com`
- **워크플로우 ID**: `6t0bgNHo7yGWM3PD`
- **워크플로우명**: Shopping Platform News Scraper
- **GitHub**: `https://github.com/gongbaksoo/n8n`

---

## 2. 워크플로우 구조 (18개 노드)

```
Schedule Trigger (평일 오전 7시 KST)
  → Keyword List (Code: 키워드 배열 반환)
  → Loop Over Keywords (SplitInBatches, batchSize: 1)
    → HTTP Request (Google News RSS 호출)
    → Wait (1초 딜레이, rate limit 방지)
    → XML Parser (RSS XML → JSON)
    → Article Extractor (Code: 기사 객체 추출)
    → (loop back to Loop Over Keywords)
  → (done) Time Filter (Code: 최근 24시간 필터)
  → Exclusion Filter (Code: 야구/할인 키워드 제외)
  → Deduplication (Code: Levenshtein 유사도 중복 제거)
  → Type Classifier (Code: 기사 유형 자동 분류)
  → AI Summary (Code: Gemini API 2회 호출)
  → Prepare Output Data (Code: 출력 데이터 정리)
  → Notion (DB 페이지 생성)
  → Format Email HTML (Code: HTML 테이블 생성)
  → Send Email (Gmail 노드)
  → Format Slack Message (Code, 비활성화)
  → Slack (비활성화)
```

---

## 3. 새 워크플로우 생성 시 변경해야 할 항목

### 3.1 반드시 변경
| 항목 | 위치 | 설명 |
|------|------|------|
| **키워드 목록** | Keyword List 노드 | `keyword`와 `channel` 값 변경 |
| **Notion DB ID** | Notion 노드 | 새 DB의 원본 ID (링크된 DB ID 아님!) |
| **이메일 수신자** | Send Email 노드 | `sendTo` 값 |
| **워크플로우 이름** | 워크플로우 속성 | 새 프로젝트에 맞는 이름 |

### 3.2 필요 시 변경
| 항목 | 위치 | 설명 |
|------|------|------|
| 제외 키워드 | Exclusion Filter 노드 | 새 키워드에 맞는 노이즈 필터 |
| 유형 분류 규칙 | Type Classifier 노드 | 기사 분류 키워드 |
| 스케줄 | Schedule Trigger 노드 | cron 표현식 |
| AI 프롬프트 | AI Summary 노드 | 분야별 전문가 역할 설명 |
| Notion 속성 매핑 | Notion 노드 | DB 스키마에 맞춤 |

---

## 4. 자격 증명 (Credentials) — 재사용 가능

이미 n8n에 등록된 자격 증명을 그대로 사용하면 됩니다.

| Credential | Type | ID | 용도 |
|-----------|------|-----|------|
| Gmail OAuth2 | gmailOAuth2 | `spDs5DdZcw6Xtski` | 이메일 발송 |
| Notion | notionApi | `I34bqtnWMHogzUqF` | Notion DB 입력 |
| Google Gemini API Key | - | (Code 노드 내 하드코딩) | AI 요약 |

**주의**: Gemini API Key는 Code 노드의 jsCode 안에 직접 포함되어 있습니다.
```
Key: AIzaSyBJPpQP0aDMWwTXrWOARUfz7Pd_B_KRsBQ
```

---

## 5. 핵심 기술 사항 (삽질 방지)

### 5.1 n8n API로 워크플로우 배포 시
```bash
# API Key
X-N8N-API-KEY: eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiI1ZjljYzZlOS1iZjBiLTRlZjAtYWFiMS00NjZkYjA0MmMzOTIiLCJpc3MiOiJuOG4iLCJhdWQiOiJwdWJsaWMtYXBpIiwianRpIjoiNDhhZTRlNTYtMTI1ZS00NmU1LTg4NzktMjBmNTMyZmIzNGI5IiwiaWF0IjoxNzc3OTgwNDM1fQ.mzf_sID1Y7czv9nfIbyGf4RUcktjmaIcOUJgfoNaoYQ

# POST 시 tags 필드 제거 (read-only)
# PUT 시 허용 필드만: name, nodes, connections, settings, staticData
# n8n MCP 도구는 인증 실패할 수 있음 → curl 직접 호출 권장
```

### 5.2 SplitInBatches (Loop) 연결 — 가장 중요!
```
Loop Over Keywords의 connections:
  output 0 (완료) → 다음 처리 노드 (Time Filter 등)  ← 반드시 연결!
  output 1 (루프) → HTTP Request

※ output 0을 null로 두면 루프 이후 노드가 전혀 실행되지 않음
※ Merge 노드 불필요 — SplitInBatches가 자동 누적
```

### 5.3 AI Summary — Code 노드 + Gemini API 직접 호출
```
❌ n8n Gemini 노드 (@n8n/n8n-nodes-langchain.googleGemini) → 모델 호환 문제
❌ HTTP Request 노드 → 기사 제목 특수문자로 JSON 깨짐
❌ Code 노드 + fetch() → n8n 샌드박스에서 fetch 미지원
✅ Code 노드 + this.helpers.httpRequest() → 정상 작동

필수 설정:
- thinkingConfig: { thinkingBudget: 0 }  ← thinking 토큰이 출력 예산 소비
- maxOutputTokens: 16000  ← 충분히 크게
- API 엔드포인트: https://generativelanguage.googleapis.com/v1beta/models/gemini-2.5-flash:generateContent
```

### 5.4 Notion 노드 설정
```
❌ 링크된 DB URL의 ID 사용 → "linked database not supported" 에러
✅ 원본 DB ID 사용 (fetch 도구로 ancestor-database URL에서 확인)

속성 key 형식: "속성명|타입"
  - 제목|title
  - URL|url
  - 태그|multi_select
  - 관련 분야|multi_select
  - Summary|rich_text
  - 텍스트|rich_text

Notion DB를 n8n 통합("n8n-sync")과 공유해야 함:
  DB → 우측상단 ... → Connections → "n8n-sync" 추가
```

### 5.5 Gmail 노드
```
❌ Google Service Account (googleApi) → Gmail 노드 미지원
✅ Gmail OAuth2 (gmailOAuth2) → 정상 작동

Gmail API가 Google Cloud 프로젝트에서 활성화되어 있어야 함
OAuth Redirect URI: https://n8n.gongbaksoo.com/rest/oauth2-credential/callback
```

### 5.6 n8n 에디터 캐시 주의
```
API로 워크플로우를 업데이트한 후, n8n 에디터에서 "Test Workflow" 실행하면
에디터에 캐시된 이전 버전으로 실행됨!

반드시 브라우저 새로고침(F5) 후 테스트할 것
```

---

## 6. 새 워크플로우 생성 절차 (Step by Step)

### Step 1: 기존 워크플로우 복제
```bash
# 기존 워크플로우 가져오기
curl -s -H "X-N8N-API-KEY: {API_KEY}" \
  "https://n8n.gongbaksoo.com/api/v1/workflows/6t0bgNHo7yGWM3PD" \
  > base_workflow.json

# 또는 GitHub에서 workflow.json 참조
```

### Step 2: 키워드 변경
Keyword List 노드의 jsCode에서 키워드 배열 수정:
```javascript
return [
  { json: { keyword: "새키워드1", channel: "카테고리A" } },
  { json: { keyword: "새키워드2", channel: "카테고리B" } },
  // ...
];
```

### Step 3: Exclusion Filter 수정
새 키워드에 맞는 노이즈 제외 키워드 설정.
현재 야구 관련 키워드 128개 + SSG 선수명 63명이 설정되어 있음.

### Step 4: Notion DB 준비
1. 기존 "업계 동향" DB를 사용하거나 새 DB 생성
2. DB를 "n8n-sync" 통합과 공유
3. **원본 DB ID** 확인 (링크된 DB ID가 아닌!)
4. Notion 노드의 `databaseId` 값 변경
5. 속성 매핑이 DB 스키마와 일치하는지 확인

### Step 5: 이메일 수신자 변경
Send Email 노드의 `sendTo` 값 수정.

### Step 6: n8n에 배포
```bash
# POST로 새 워크플로우 생성 (tags 필드 제거 필수)
python3 -c "
import json
with open('workflow.json') as f:
    wf = json.load(f)
# tags, id, createdAt 등 제거
for key in ['tags', 'id', 'createdAt', 'updatedAt', 'versionId']:
    wf.pop(key, None)
with open('deploy.json', 'w') as f:
    json.dump(wf, f, ensure_ascii=False)
"

curl -s -X POST \
  -H "X-N8N-API-KEY: {API_KEY}" \
  -H "Content-Type: application/json" \
  -d @deploy.json \
  "https://n8n.gongbaksoo.com/api/v1/workflows"
```

### Step 7: 자격 증명 연결
```bash
# 워크플로우 GET → credentials 업데이트 → PUT
# Gmail OAuth2: spDs5DdZcw6Xtski
# Notion: I34bqtnWMHogzUqF
```

### Step 8: 테스트 및 활성화
1. n8n UI에서 "Test Workflow" 실행
2. 이메일/Notion 정상 확인
3. Active = true로 변경

---

## 7. AI Summary Code 노드 전문

Gemini API를 2회 호출하는 핵심 코드. 새 워크플로우에서도 동일 구조 사용.

**1차 호출**: 전체 트렌드 요약 (이메일용)
**2차 호출**: 기사별 한줄 요약 + 3줄 요약 (Notion용)

한줄 요약 규칙:
- `[업체명] 핵심 내용` 형식
- 한글 30자 이내
- 주어 = 기사 메인 업체명

파싱 방식:
- `#N` 텍스트 마커로 기사별 분리
- `한줄:` 마커로 한줄 요약 추출
- 나머지 `1. 2. 3.`이 3줄 요약

---

## 8. 에러 대응 가이드 (총 21건의 경험)

| 에러 | 원인 | 해결 |
|------|------|------|
| tags is read-only | POST 시 tags 포함 | tags 필드 제거 |
| additional properties | PUT 시 read-only 필드 포함 | name/nodes/connections/settings/staticData만 |
| Gmail 느낌표 | gmailOAuth2 아닌 자격 증명 | Gmail OAuth2 자격 증명 사용 |
| Gmail Forbidden | Gmail API 미활성화 | Google Cloud Console에서 Enable |
| Loop 이후 미실행 | SplitInBatches output 0 미연결 | output 0 → 다음 노드 연결 |
| Format Email missing / | Python→JSON \n 깨짐 | 별도 변수 분리 |
| Notion linked DB | 링크된 DB ID 사용 | 원본 DB ID 사용 |
| Notion resource not found | DB가 n8n-sync와 미공유 | Connections에 n8n-sync 추가 |
| Notion 속성 undefined | key에 타입 미포함 | `속성명\|타입` 형식 사용 |
| Gemini 노드 미작동 | n8n 내장 Gemini 노드 호환 문제 | Code + httpRequest 사용 |
| HTTP Request JSON 깨짐 | 특수문자가 JSON 파괴 | Code 노드에서 JSON.stringify |
| fetch() 미지원 | n8n 샌드박스 제한 | this.helpers.httpRequest() |
| Gemini 출력 잘림 | thinking 토큰이 출력 예산 소비 | thinkingBudget: 0 |
| 에디터 캐시 | API 업데이트가 에디터에 미반영 | F5 새로고침 후 테스트 |
| onError 마스킹 | continueRegularOutput이 에러 숨김 | 출력 데이터의 error 필드 확인 |

---

## 9. 파일 구조

```
n8n_news_scrap/
├── workflow.json          # n8n 워크플로우 JSON (로컬 참조용)
├── SETUP.md               # 초기 설정 가이드
├── HANDOVER.md            # 이 인수인계서
├── error.md               # 에러 이력 (21건)
├── history.md             # 작업 이력
├── .gitignore
└── docs/
    ├── 01-plan/features/
    │   └── n8n-news-scraper.plan.md
    └── 02-design/features/
        └── n8n-news-scraper.design.md
```

---

## 10. 새 세션 시작 시 프롬프트 예시

```
이 폴더의 HANDOVER.md를 읽고, 동일한 구조의 n8n 뉴스 스크래퍼 워크플로우를
새로 만들어줘.

변경 사항:
- 키워드: [새 키워드 목록]
- Notion DB: [새 DB URL]
- 이메일 수신자: [새 이메일]
- 제외 키워드: [새 제외 키워드]

기존 워크플로우(6t0bgNHo7yGWM3PD)를 복제해서 위 항목만 변경하면 돼.
n8n API로 직접 배포해줘.
```
