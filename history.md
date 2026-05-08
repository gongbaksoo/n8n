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

### 남은 작업
1. Notion DB ID 설정 및 노드 활성화
2. Google Sheets 스프레드시트 ID 설정 및 노드 활성화
3. Slack credential 연결 및 노드 활성화
4. 전체 통합 테스트
5. 스케줄 활성화
