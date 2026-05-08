# Error Log

## 2026-05-05: 워크플로우 배포 중 발생한 에러

### Error 1: tags is read-only

- **시점**: n8n API로 워크플로우 생성 시
- **에러 메시지**: `request/body/tags is read-only`
- **원인**: workflow.json에 `tags` 필드를 포함하여 POST 요청
- **해결**: `tags` 필드를 제거한 후 재요청 → 성공

### Error 2: OpenAI Credential 생성 시 schema validation

- **시점**: POST /api/v1/credentials (OpenAI)
- **에러 메시지**: `request.body.data requires property "headerName"`, `requires property "headerValue"`
- **원인**: n8n의 openAiApi credential schema가 `headerName`, `headerValue` 필드를 필수로 요구
- **해결**: `"headerName": "", "headerValue": ""` 빈 값으로 추가하여 재요청 → 성공

### Error 3: Workflow PUT 시 additional properties

- **시점**: PUT /api/v1/workflows/{id} (credential 업데이트)
- **에러 메시지**: `request/body must NOT have additional properties`
- **원인**: GET으로 받은 워크플로우에 `id`, `createdAt`, `updatedAt`, `versionId`, `tags` 등 read-only 필드가 포함된 채로 PUT 요청
- **해결**: 허용 필드만 추출 (`name`, `nodes`, `connections`, `settings`, `staticData`) 후 PUT → 성공

### Error 4: Credentials API GET 불가

- **시점**: 기존 credentials 목록 조회 시
- **에러 메시지**: `GET method not allowed` (credentials endpoint)
- **원인**: n8n Public API에서 credentials GET 목록 조회가 제한됨 (보안 정책)
- **우회**: 기존 워크플로우들의 노드에서 credential ID를 추출하여 확인

---

## 2026-05-08: 워크플로우 수정 중 발생한 에러

### Error 5: Gmail 노드 느낌표 (Credential 타입 불일치)

- **시점**: Send Email 노드에 Google Service Account 연결 후
- **에러 메시지**: n8n UI에서 노드에 느낌표(!) 표시
- **원인**: `n8n-nodes-base.gmail` 노드는 `gmailOAuth2` 자격 증명만 지원. `googleApi` (Service Account)는 호환 불가
- **시도**: HTTP Request 노드로 변경하여 Gmail API 직접 호출 → Authorization failed (Service Account에 Gmail API 권한 없음)
- **최종 해결**: Gmail 노드 유지 + Gmail OAuth2 자격 증명 신규 생성

### Error 6: Google Service Account로 Gmail API 호출 실패

- **시점**: Send Email 노드를 HTTP Request로 변경하여 Gmail API 호출 시
- **에러 메시지**: `Authorization failed - please check your credentials`
- **원인**: Google Service Account는 서버 계정으로 자체 이메일 주소가 없음. Gmail 발송은 사용자 계정(OAuth2) 인증 필요
- **해결**: Gmail 노드 + Gmail OAuth2 방식으로 최종 결정

### Error 7: Loop Over Keywords 완료 출력 미연결

- **시점**: 첫 테스트 실행 시 — 루프 이후 노드가 전혀 실행되지 않음
- **에러 메시지**: 에러 없음 (status: success이나 후반부 노드 미실행)
- **원인**: SplitInBatches output 0 (완료)이 `null` — 아무 노드에도 연결되지 않음
- **해결**: output 0 → Time Filter 직접 연결. Merge All Articles 노드 제거

### Error 8: Format Email HTML — missing /

- **시점**: 이메일 포맷 노드 실행 시
- **에러 메시지**: `missing /`
- **원인**: Python에서 JS 코드를 JSON 직렬화할 때 `\n` 이스케이프가 실제 줄바꿈으로 변환. `/\n/g` 정규식이 깨짐
- **해결**: 별도 변수로 분리하여 이스케이프 문제 회피

### Error 9: Gmail OAuth2 자격 증명 생성 시 스키마 오류

- **시점**: POST /api/v1/credentials (gmailOAuth2)
- **에러 메시지**: `requires property "serverUrl"`, `requires property "additionalBodyProperties"`
- **원인**: n8n의 gmailOAuth2 credential schema가 여러 필수 필드 요구
- **해결**: `serverUrl`, `sendAdditionalBodyProperties`, `additionalBodyProperties`, `oauthTokenData` 추가 → 성공

### Error 10: Gmail API 미활성화

- **시점**: Send Email 노드 실행 시
- **에러 메시지**: `Gmail API has not been used in project 242591179518 before or it is disabled`
- **원인**: Google Cloud 프로젝트에서 Gmail API가 활성화되지 않음
- **해결**: Google Cloud Console → Gmail API → Enable → 정상 작동

### Error 11: n8n MCP 인증 실패

- **시점**: 세션 시작 시
- **에러 메시지**: `Failed to authenticate with n8n. Please check your API key.`
- **원인**: 이전 세션의 API 키 만료 또는 MCP 캐시 문제
- **해결**: 새 API 키 발급 후 curl 직접 호출로 우회

### Error 12: 워크플로우 실행 불가 (플레이스홀더 값)

- **시점**: Test Workflow 클릭 시
- **에러 메시지**: `The workflow has issues and cannot be executed`
- **원인**: Sheets/Notion/Slack 노드에 플레이스홀더 값 잔존
- **해결**: 해당 노드들 비활성화 후 이메일 경로만 테스트 진행

---

## 미해결 이슈

### Slack Credential ID 미확인
- **상태**: Slack 노드 비활성화 상태
- **해결 방안**: n8n UI에서 Slack credential 확인 후 연결

### Google Sheets / Notion 플레이스홀더
- **상태**: 노드 비활성화 상태, 실제 ID 설정 필요
- **해결 방안**: 스프레드시트 ID, Notion DB ID 입력 후 노드 활성화
