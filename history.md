# Project History

## 2026-05-05: 프로젝트 시작 및 워크플로우 배포

### Plan Plus (브레인스토밍 + 계획)
- 기존 Google Apps Script 뉴스 수집 시스템 코드 분석
- 기능: 17개 쇼핑 플랫폼 키워드 RSS 수집, 야구/할인 필터링, 유사도 중복 제거, 유형 분류
- n8n 마이그레이션 목표 확정: 기존 기능 + AI 요약 + Slack + Notion 추가
- 단일 워크플로우 방식(Approach A) 선택
- MVP 범위: 전체 기능 포함 (추가 소스만 v2로 연기)
- 스케줄: 평일 오전 7시 KST

### Design Phase
- Option B (Clean Architecture) 선택: 18개 노드, 역할별 완전 분리
- Data Model, Node Specification, Error Handling, Test Plan 설계 완료

### Do Phase (구현)
- `workflow.json` 생성 및 n8n API 직접 배포 (ID: `6t0bgNHo7yGWM3PD`)
- `SETUP.md` 설정 가이드 작성

### Credentials 연결 (1차)
| Credential | 방법 | 결과 |
|-----------|------|------|
| OpenAI API | API로 신규 생성 | ✅ |
| Google Sheets | 기존 credential ID 발견 | ✅ |
| Notion | 기존 credential ID 발견 | ✅ |
| Send Email | Google Service Account 연결 시도 | ⚠️ |
| Slack | ID 미확인 | ⏳ |

---

## 2026-05-08: 워크플로우 수정 및 이메일 발송 성공

### 주요 변경 사항

#### 1. Send Email 노드 수정 (Gmail OAuth2)
- **문제**: Google Service Account는 Gmail 노드와 호환 불가
- **시도 1**: HTTP Request + Gmail API 직접 호출 → Service Account에 Gmail 권한 없어 실패
- **시도 2**: Gmail 노드 복원 + Gmail OAuth2 자격 증명 신규 생성 → 성공
- **추가 조치**: Google Cloud Console에서 Gmail API 활성화

#### 2. Loop 연결 버그 수정
- SplitInBatches output 0 (완료)이 미연결 → Time Filter 직접 연결
- Merge All Articles 노드 제거 (불필요), 노드 수 20→19개

#### 3. AI Summary 모델 변경 (OpenAI → Gemini)
- 이유: OpenAI API 비용 초과
- OpenAI GPT-4o → Google Gemini 2.0 Flash로 변경
- n8n Gemini 자격 증명 타입: `googlePalmApi`

#### 4. Format Email HTML 정규식 버그 수정
- Python→JSON 직렬화 시 `\n` 깨짐 → 별도 변수 분리로 해결

#### 5. 비활성화 노드
- Google Sheets, Notion, Slack: 플레이스홀더 값 미설정으로 비활성화

### 테스트 결과
- 이메일 발송: ✅ 성공
- AI 요약 (Gemini): ✅ 성공
- 뉴스 수집 파이프라인: ✅ 성공

### Credentials 현황 (최종)
| Credential | Type | ID | Status |
|-----------|------|-----|--------|
| Google Gemini API | googlePalmApi | bOqjILXaQe93TSQa | ✅ |
| Google Sheets | googleSheetsOAuth2Api | u9biJnMmTMX61aw5 | ✅ (비활성화) |
| Notion | notionApi | I34bqtnWMHogzUqF | ✅ (비활성화) |
| Gmail OAuth2 | gmailOAuth2 | spDs5DdZcw6Xtski | ✅ 테스트 성공 |
| Slack | slackOAuth2Api | ? | ⏳ (비활성화) |

### 남은 작업 (당시)
1. Notion DB ID 설정 및 노드 활성화
2. Google Sheets 스프레드시트 ID 설정 및 노드 활성화
3. Slack credential 연결 및 노드 활성화

---

## 2026-05-10: Notion 연결 완료 및 AI Summary 안정화

### 주요 변경 사항

#### 1. Google Sheets 노드 제거
- 불필요하여 워크플로우에서 완전 삭제
- 노드 수 19 → 18개

#### 2. Notion 연결 완료
- 기존 "업계 동향" DB에 연결 (id: `1e26d523e164805b9950f5197b8b216e`)
- 링크된 DB가 아닌 원본 DB ID 사용 필수 (linked database API 미지원)
- n8n 통합("n8n-sync")에 DB 공유 필요
- 속성 key 형식: `속성명|타입` (예: `제목|title`, `URL|url`, `태그|multi_select`)
- 매핑: 제목, URL, 태그(keyword), 관련분야(channel→이커머스/오프라인 동향), Summary, 텍스트(기사별 3줄 요약)

#### 3. AI Summary 전면 재구축
- **n8n Gemini 노드**: 모델 호환 문제로 사용 불가
- **HTTP Request 노드**: 기사 제목 특수문자로 JSON 깨짐
- **최종 방식**: Code 노드 + `this.helpers.httpRequest()` (Gemini API 직접 호출)
- Gemini 2회 호출: 1차 전체 트렌드 요약 (이메일용) + 2차 기사별 3줄 요약 (Notion 텍스트 필드)
- `thinkingConfig: { thinkingBudget: 0 }` 필수 (thinking 토큰이 출력 예산 소비)
- `#N` 텍스트 마커 파싱 방식 (JSON 출력보다 안정적)

#### 4. n8n 에디터 캐시 이슈 발견
- API로 워크플로우 업데이트 후 브라우저 새로고침 필수
- Test Workflow는 서버 버전이 아닌 에디터 캐시 버전으로 실행됨

### 테스트 결과
- 이메일 발송: ✅ 성공
- AI 전체 요약 (Gemini): ✅ 성공
- AI 기사별 3줄 요약: ✅ 성공
- Notion 입력: ✅ 성공 (제목, URL, 태그, 관련분야, Summary, 텍스트)

### 현재 워크플로우 구조
```
Schedule Trigger → Keyword List → Loop Over Keywords
  → HTTP Request → Wait → XML Parser → Article Extractor → (loop back)
  → (done) Time Filter → Exclusion Filter → Deduplication → Type Classifier
  → AI Summary (Code, Gemini API 2회) → Prepare Output Data
  → Notion ✅
  → Format Email HTML → Send Email ✅
  → Format Slack Message → Slack ⏸️
```

### 남은 작업 (당시)
1. Slack credential 연결 및 노드 활성화
2. 전체 통합 테스트
3. 스케줄 활성화 (Active = true)

---

## 2026-05-11: Notion Summary 개선, 야구 필터링 강화, 스케줄 활성화

### 주요 변경 사항

#### 1. Notion Summary → 한줄 요약
- Summary 필드: `[유형] 제목` → AI 한줄 요약으로 변경
- 텍스트 필드: AI 3줄 요약 유지
- Gemini 프롬프트에 `한줄:` 마커 추가하여 한줄/3줄 분리 파싱

#### 2. 야구 필터링 강화
- 야구 키워드 31개 추가: 잠실, 두산, 잭팟, 친정, 야유, 솔로포, 투구, 연패, 연승, 스코어, 한화, LG, KT, 키움, NC, 롯데자이언츠, KIA, 삼성라이온즈, 잠실구장, 사직구장, 고척돔, 인천구장, 수원구장, 2연승~5연승, 꺾고, 상대전적, 승률
- SSG 선수명 4명 추가: 오명진, 배동현, 박준순, 잭 로그
- "타선" 키워드 추가
- 이전 수집 SSG 야구 기사 7건 중 7건 모두 필터링 확인

#### 3. 스케줄 활성화
- 워크플로우 Active = true
- 평일(월~금) 오전 7시 KST 자동 실행

#### 4. 한줄 요약 프롬프트 가이드 설정
- 형식: `[업체명] 핵심 내용` (예: [쿠팡] 전년비 쇼핑 결제액 30% 증가)
- 한글 30자 이내, 주어는 기사 메인 업체명

### 남은 작업
1. Slack credential 연결 및 노드 활성화
2. 자동 실행 모니터링

---

## 2026-05-15: Notion 마크다운 본문 + 원문 스크래핑 + Jaccard 클러스터링

### 주요 변경 사항

#### 1. Deduplication: Levenshtein → Jaccard 클러스터링
- 기존 Levenshtein 유사도(O(m*n) 문자 비교) → Jaccard 단어 유사도(O(n) 집합 비교)로 교체
- 채널별 그룹화 후 클러스터링, 대표 기사 1건 선정
- 임계값: 0.3 (같은 이슈 판별)
- TOP_N 제한 제거: 모든 고유 이슈 유지 (중복만 제거)

#### 2. 원문 스크래핑 파이프라인 구축 (3단계)
- **Step 1**: Google News 기사 페이지 HTML에서 signature/timestamp 추출
- **Step 2**: batchexecute API + signature/timestamp로 실제 기사 URL 디코딩
- **Step 3**: Jina Reader API(`r.jina.ai/`)로 실제 URL에서 마크다운 추출
- 성공률: ~80% (일부 언론사 디코딩/추출 실패)

#### 3. Notion 페이지 본문 마크다운 블록 추가
- 새 노드 2개 추가: Build Notion Blocks + Notion Add Content
- Notion 페이지 생성(속성) 후, API(`PATCH blocks/{id}/children`)로 본문 블록 추가
- 마크다운 → Notion 블록 변환: heading_2/3, paragraph, numbered_list_item, bulleted_list_item, divider, callout
- 볼드(**) 텍스트 annotations 지원
- 스크래핑 실패 시 ⚠️ callout 블록 표시

#### 4. Notion 본문 구조
```
## 📝 요약
[업체명] 핵심 내용 한줄 요약
1. 포인트1
2. 포인트2
3. 포인트3
---
## 📄 본문
(원문 기반 구조화된 마크다운)
---
## 💡 핵심 인사이트
- 인사이트1
- 인사이트2
---
## 📊 메타 정보
- 📅 발행일 / 📂 카테고리 / 🏷️ 태그 / ⭐ 유형 / 🔗 원본
```

#### 5. 스크래핑 불가 언론사 제외
- Exclusion Filter에 `excludeSources: ["뉴시스", "브릿지경제"]` 추가
- 같은 이슈의 다른 언론사 기사가 대표로 선정되도록 처리

### 시도했으나 실패한 방법들
| 방법 | 실패 원인 |
|------|----------|
| base64 직접 디코딩 | protobuf 인코딩으로 URL 추출 불가 |
| batchexecute (signature 없이) | Google이 `[3]` 에러코드로 거부 |
| Jina Reader 직접 (Google News URL) | HTTP 451 차단 |
| batchexecute (잘못된 payload 구조) | 내부 배열 위치 오류로 null 반환 |

### 최종 작동 방식
```
Google News URL
  → HTML에서 signature/timestamp 추출
  → batchexecute + sig/ts → 실제 기사 URL
  → Jina Reader → 깨끗한 마크다운
  → Gemini → 구조화된 마크다운 (요약/본문/인사이트/메타)
  → Notion 페이지 본문 블록 추가
```

### 현재 워크플로우 구조 (20개 노드)
```
Schedule Trigger → Keyword List → Loop Over Keywords
  → HTTP Request → Wait → XML Parser → Article Extractor → (loop back)
  → (done) Time Filter → Exclusion Filter (야구/할인/언론사)
  → Deduplication (Jaccard 0.3) → Type Classifier
  → AI Summary (Gemini: 전체요약 + 한줄/3줄 + 원문스크래핑+마크다운)
  → Prepare Output Data
  → Notion → Build Notion Blocks → Notion Add Content
  → Format Email HTML → Send Email
  → Format Slack Message → Slack (비활성화)
```

### 남은 작업
1. Slack credential 연결 및 노드 활성화
2. 자동 실행 모니터링
3. 스크래핑 실패 언론사 추가 모니터링
