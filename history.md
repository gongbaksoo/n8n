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
- 노드 구성: Schedule Trigger → Keyword List → SplitInBatches Loop → HTTP Request → XML Parser → Article Extractor → Merge → Time Filter → Exclusion Filter → Deduplication → Type Classifier → AI Summary → 병렬 출력(Sheets/Notion/Slack/Email)
- Data Model, Node Specification, Error Handling, Test Plan 설계 완료

### Do Phase (구현)
- `workflow.json` 생성 (n8n 임포트용 완전한 워크플로우 JSON)
- `SETUP.md` 설정 가이드 작성
- n8n API를 통한 워크플로우 직접 배포 (ID: `6t0bgNHo7yGWM3PD`)

### Credentials 연결
| Credential | 방법 | 결과 |
|-----------|------|------|
| OpenAI API | API로 신규 생성 + 워크플로우 연결 | ✅ |
| Google Sheets | 기존 credential ID 발견 → 워크플로우 연결 | ✅ |
| Notion | 기존 credential ID 발견 → 워크플로우 연결 | ✅ |
| Send Email | Gmail OAuth2 → Google Service Account로 변경 연결 | ✅ |
| Slack | credential 존재 확인, ID 미확인으로 미연결 | ⏳ |

### 남은 작업
1. Slack credential 연결 (ID 확인 필요)
2. 플레이스홀더 값 설정 (Sheets ID, Notion DB ID, Slack Channel ID)
3. 수동 테스트 실행
4. 스케줄 활성화 (Active = true)
