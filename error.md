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

## 미해결 이슈

### Slack Credential ID 미확인
- **상태**: n8n에 "Slack account" (slackApi) credential 존재 확인됨
- **문제**: API로 credential ID를 조회할 수 없음
- **해결 방안**: n8n UI에서 Slack credential 클릭 → URL의 ID 확인 후 워크플로우에 연결
