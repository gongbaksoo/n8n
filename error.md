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

## 2026-05-10: Notion 연결 및 AI Summary 안정화

### Error 13: Notion Linked Database 미지원

- **시점**: Notion 노드 실행 시
- **에러 메시지**: `Database with ID ... is a linked database. Database retrievals do not support linked databases.`
- **원인**: 사용자가 제공한 URL이 링크된 DB (뷰)이고 원본 DB가 아님
- **해결**: 원본 DB ID (`1e26d523e164805b9950f5197b8b216e`) 사용

### Error 14: Notion "resource not found"

- **시점**: Notion 노드 실행 시
- **에러 메시지**: `Could not find database... Make sure shared with your integration "n8n-sync"`
- **원인**: Notion DB가 n8n 통합("n8n-sync")과 공유되지 않음
- **해결**: Notion에서 DB → Connections → "n8n-sync" 추가

### Error 15: Notion 속성 key 형식 오류

- **시점**: Notion 노드 실행 시
- **에러 메시지**: `body.properties.제목.title should be defined, instead was 'undefined'`
- **원인**: n8n Notion 노드 v2.2는 key가 `속성명|타입` 형식이어야 함 (예: `제목|title`, `URL|url`, `태그|multi_select`)
- **해결**: 모든 속성 key를 `이름|타입` 형식으로 변경

### Error 16: Notion onError 마스킹

- **시점**: Notion 노드 실행 시 — 28건 모두 Bad request이나 status: success
- **원인**: `onError: continueRegularOutput` 설정으로 에러가 성공으로 처리됨
- **교훈**: 디버깅 시 출력 데이터의 error 필드를 반드시 확인

### Error 17: n8n Gemini 노드 모델 미지원

- **시점**: AI Summary (n8n Gemini 노드) 실행 시
- **에러 메시지**: `The resource you are requesting could not be found`
- **원인**: n8n의 `@n8n/n8n-nodes-langchain.googleGemini` 노드가 내부적으로 다른 API 엔드포인트 사용
- **해결**: n8n Gemini 노드 포기 → Code 노드 + `this.helpers.httpRequest()`로 Gemini API 직접 호출

### Error 18: Gemini HTTP Request JSON 깨짐

- **시점**: AI Summary (HTTP Request 노드) 실행 시
- **에러 메시지**: `JSON parameter needs to be valid JSON`
- **원인**: HTTP Request 노드의 expression에서 기사 제목의 특수문자(따옴표 등)가 JSON을 깨뜨림
- **해결**: HTTP Request 노드 포기 → Code 노드로 변경 (JavaScript에서 JSON.stringify로 안전하게 처리)

### Error 19: Code 노드 fetch() 미지원

- **시점**: AI Summary (Code 노드) 실행 시
- **원인**: n8n Code 노드 샌드박스에서 `fetch()`를 사용할 수 없음
- **해결**: `this.helpers.httpRequest()` 사용

### Error 20: Gemini thinking 토큰으로 출력 잘림

- **시점**: 기사별 3줄 요약 생성 시
- **에러 메시지**: `Unterminated string in JSON at position 1124`
- **원인**: Gemini 2.5 Flash의 "thinking" 토큰이 출력 토큰 예산을 소비하여 응답이 잘림
- **해결**: `thinkingConfig: { thinkingBudget: 0 }` 추가 + `maxOutputTokens: 16000` 증가

### Error 21: n8n 에디터 캐시 — API 업데이트 미반영

- **시점**: API로 워크플로우 업데이트 후 Test Workflow 실행 시
- **원인**: n8n 에디터가 브라우저에 이전 버전 캐시. Test Workflow는 에디터 버전으로 실행
- **해결**: 브라우저 새로고침(F5) 후 Test Workflow 실행

---

---

## 2026-05-15: Notion 마크다운 본문 + 원문 스크래핑 구현

### Error 22: Google News URL base64 디코딩 실패

- **시점**: AI Summary 노드에서 원문 스크래핑 시도 시
- **원인**: Google News RSS 링크(`news.google.com/rss/articles/CBMi...`)는 protobuf 인코딩으로 단순 base64 디코딩 불가
- **시도**: Buffer.from(base64, 'base64').toString()으로 URL 추출 시도 → URL 패턴 미발견
- **해결**: batchexecute API 방식으로 전환 (Error 23 참조)

### Error 23: batchexecute API에서 null 응답 (signature/timestamp 누락)

- **시점**: batchexecute 호출 시 `[3]` 에러코드 반환
- **응답**: `[["wrb.fr","Fbv4je",null,null,null,[3],"generic"]]`
- **원인**: batchexecute 요청에 signature/timestamp 인증값이 없어서 Google이 거부
- **해결**: Google News 기사 페이지 HTML에서 `data-n-a-sg`(signature), `data-n-a-ts`(timestamp) 추출 후 요청에 포함

### Error 24: batchexecute payload 구조 오류

- **시점**: signature/timestamp 포함 후에도 null 응답
- **원인**: payload 내부 배열 구조가 Python googlenewsdecoder와 다름
  - 잘못된 형식: `["garturlreq", [[locale_array], articleId, ts, sg]]` (내부 배열에 포함)
  - 올바른 형식: `["garturlreq", [locale_array], "articleId", timestamp, "sg"]` (외부 별도 인자)
  - locale 배열도 달랐음: 복잡한 값 대신 `["X","X",["X","X"],...]` 사용
- **해결**: Python googlenewsdecoder 소스 코드의 정확한 payload 형식 적용 → 성공

### Error 25: Jina Reader API HTTP 451 (Google News URL 직접 접근)

- **시점**: Jina Reader에 Google News URL을 직접 전달 시
- **에러 메시지**: `Request failed with status code 451` (Unavailable For Legal Reasons)
- **원인**: Google News가 Jina Reader 서버의 접근을 법적 사유로 차단
- **해결**: Google News URL을 먼저 batchexecute로 디코딩하여 실제 기사 URL 확보 → Jina Reader에 실제 URL 전달

### Error 26: Build Notion Blocks 1건만 처리

- **시점**: Notion 페이지 10건 중 1건에만 본문 블록 추가
- **원인**: Code 노드에서 `$json.id`가 첫 번째 아이템만 참조 (Run Once for All Items 모드)
- **해결**: `$input.all()` + `$('Prepare Output Data').all()`로 전체 아이템 순회

### Error 27: 뉴시스/브릿지경제 스크래핑 실패

- **시점**: 10건 중 2건 markdownContent = "요약 실패"
- **원인**: 해당 언론사의 Google News URL이 batchexecute에서 디코딩 실패하거나 Jina Reader에서 콘텐츠 추출 불가
- **해결**: Exclusion Filter에 `excludeSources: ["뉴시스", "브릿지경제"]` 추가하여 사전 제외

### Error 28: Code 노드 타임아웃 (N8N_RUNNERS_TASK_TIMEOUT)

- **시점**: 17개 키워드 전체 실행 시 AI Summary 노드
- **에러 메시지**: `Task execution timed out after 300 seconds`
- **원인**: n8n Code 노드 전용 타임아웃(300초)이 워크플로우 타임아웃(1시간)과 별도. 68건 스크래핑에 ~25분 소요
- **해결**: Docker 환경변수 `N8N_RUNNERS_TASK_TIMEOUT=1800` 추가 (docker-compose.yml) + 컨테이너 재시작

### Error 29: Gmail OAuth2 토큰 만료

- **시점**: Send Email 노드 실행 시
- **에러 메시지**: `The provided authorization grant or refresh token is invalid, expired, revoked`
- **원인**: Google Cloud Console에서 OAuth 앱이 "Testing" 모드 → refresh token 7일 자동 만료
- **해결**: Google Cloud Console → OAuth 동의 화면 → "앱 게시(Publish App)" → 프로덕션 모드 전환 + n8n에서 Gmail OAuth2 재연결

### Error 30: 키워드 오타 "컴리" (에디터 캐시 관련)

- **시점**: 키워드 17개 복원 후 테스트 시
- **원인**: API로 업데이트한 "컬리"가 n8n 에디터 캐시에서 반영되지 않아 이전 버전 "컴리"로 실행됨
- **해결**: 에디터에서 직접 수정 후 Save. API 업데이트 후에는 반드시 에디터에서 노드를 열어 확인/저장 필요

### Error 31: Docker 재시작 후 스케줄 트리거 미작동

- **시점**: N8N_RUNNERS_TASK_TIMEOUT 설정을 위해 컨테이너 재시작 후 다음 날 오전 7시
- **원인**: 워크플로우 active: true 상태였지만 스케줄러가 cron을 재등록하지 못함
- **해결**: 워크플로우 비활성화 → 재활성화 (deactivate → activate)

---

## 미해결 이슈

### Slack Credential ID 미확인
- **상태**: Slack 노드 비활성화 상태
- **해결 방안**: n8n UI에서 Slack credential 확인 후 연결

### 원문 스크래핑 성공률 (~85%)
- **상태**: 일부 언론사에서 batchexecute 디코딩 또는 Jina Reader 추출 실패
- **해결 방안**: 2회 연속 성공 없는 언론사를 excludeSources에 추가 (현재 11곳 제외)
