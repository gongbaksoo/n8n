# n8n News Scraper — 설정 가이드

## 빠른 시작

### 1. 워크플로우 임포트

1. n8n 대시보드에서 **Add Workflow** > **Import from File** 클릭
2. `workflow.json` 파일 선택하여 임포트
3. 워크플로우가 열리면 아래 Credentials 설정 진행

### 2. Credentials 설정 (5개)

| # | Credential | 설정 방법 |
|---|-----------|-----------|
| 1 | **OpenAI API** | Settings > Credentials > OpenAI > API Key 입력 |
| 2 | **Google Sheets OAuth2** | Google Cloud Console에서 OAuth2 클라이언트 생성 후 연결 |
| 3 | **Notion API** | [notion.so/my-integrations](https://www.notion.so/my-integrations)에서 Integration 생성 |
| 4 | **Slack OAuth2** | [api.slack.com](https://api.slack.com/apps)에서 App 생성, `chat:write` 스코프 추가 |
| 5 | **Gmail OAuth2** | Google Cloud Console에서 Gmail API 활성화 후 OAuth2 연결 |

### 3. 워크플로우 내 설정값 변경

워크플로우 임포트 후 다음 노드의 설정을 실제 값으로 변경해야 합니다:

#### Google Sheets 노드
- `YOUR_SPREADSHEET_ID` → 실제 스프레드시트 ID로 변경
- 스프레드시트에 헤더 행 추가: `날짜 | 채널 | 키워드 | 유형 | 제목 | 링크`

#### Notion 노드
- `YOUR_NOTION_DATABASE_ID` → 실제 Notion DB ID로 변경
- Notion DB에 다음 속성 생성:
  - 제목 (Title)
  - 날짜 (Date)
  - 채널 (Select: 온라인, 오프라인)
  - 키워드 (Select: 쿠팡, 11번가, ... 17개)
  - 유형 (Select: 신제품 출시, 사업 확장, ...)
  - 링크 (URL)

#### Slack 노드
- `YOUR_SLACK_CHANNEL_ID` → 실제 채널 ID로 변경

#### Credential ID 연결
- 각 노드의 Credentials 탭에서 위에서 생성한 credential을 선택

### 4. 테스트

1. 워크플로우 캔버스에서 **Test Workflow** 버튼 클릭 (수동 실행)
2. 각 노드의 output을 확인하여 데이터 흐름 검증
3. 출력 채널(Sheets, Notion, Slack, Email) 각각 확인

### 5. 활성화

- 모든 테스트 통과 후 우상단 **Active** 토글을 ON으로 변경
- 매주 월-금 오전 7시(KST)에 자동 실행됩니다

---

## 트러블슈팅

| 증상 | 원인 | 해결 |
|------|------|------|
| HTTP Request 실패 | Google News 차단 | User-Agent 확인, IP 변경 검토 |
| XML Parser 에러 | RSS 응답이 HTML | Continue On Fail로 스킵됨, 정상 |
| AI Summary 빈 응답 | OpenAI API 키 만료 | Credential 갱신 |
| Sheets 권한 에러 | OAuth 만료 | Google Sheets credential 재연결 |
| Notion 429 에러 | Rate limit | n8n 자동 retry로 처리됨 |
| Slack 채널 못 찾음 | 채널 ID 오류 | Slack App이 채널에 초대되었는지 확인 |

---

## 구조

```
n8n_news_scrap/
├── workflow.json          # n8n 워크플로우 (임포트용)
├── SETUP.md              # 이 파일
└── docs/
    ├── 01-plan/features/n8n-news-scraper.plan.md
    └── 02-design/features/n8n-news-scraper.design.md
```
