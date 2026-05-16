# n8n News Scraper Design Document

> **Summary**: 쇼핑 플랫폼 뉴스 자동 수집, AI 요약, 멀티채널 발송 n8n 워크플로우 상세 설계
>
> **Project**: n8n_news_scrap
> **Author**: Team
> **Date**: 2026-05-04
> **Status**: Draft
> **Planning Doc**: [n8n-news-scraper.plan.md](../../01-plan/features/n8n-news-scraper.plan.md)

---

## Context Anchor

| Key | Value |
|-----|-------|
| **WHY** | 기존 Apps Script의 확장성 한계 극복 — AI 분석, 멀티채널 알림, 구조화 아카이빙 필요 |
| **WHO** | 팀/조직 내부 비즈니스 의사결정자 (jh@avent.co.kr, aspark@avent.co.kr) |
| **RISK** | Google News RSS 비공식 API 차단 가능성, SSG 야구 오탐, API rate limit |
| **SUCCESS** | 매일 평일 7시 AI 요약 포함 뉴스 브리핑 발송 + Sheets/Notion 아카이빙 정상 동작 |
| **SCOPE** | 단일 n8n 워크플로우, 17개 키워드, Google News RSS만 (추가 소스 v2) |

---

## 1. Overview

### 1.1 Design Goals

- 각 처리 단계를 독립 노드로 분리하여 디버깅/수정 용이성 확보
- n8n의 시각적 워크플로우 장점을 최대한 활용
- 각 노드가 명확한 입출력 스키마를 가져 데이터 흐름 추적 가능
- 출력 채널(Sheets, Notion, Slack, Email)은 완전 독립 병렬 처리

### 1.2 Design Principles

- **Single Responsibility**: 각 노드는 하나의 역할만 수행
- **Clear Data Contract**: 노드 간 데이터 스키마 명시
- **Fail-Safe**: 개별 출력 채널 실패가 전체 워크플로우에 영향을 주지 않음
- **Observability**: 각 노드 출력에서 중간 결과 확인 가능

---

## 2. Architecture

### 2.0 Architecture Comparison

| Criteria | Option A: Minimal | Option B: Clean | Option C: Pragmatic |
|----------|:-:|:-:|:-:|
| **Code Node 수** | 1개 | 4개 | 2개 |
| **총 노드 수** | ~10개 | ~18개 | ~13개 |
| **복잡도** | Low | High | Medium |
| **유지보수성** | Medium | High | High |
| **디버깅** | 어려움 | 쉬움 | 양호 |

**Selected**: Option B: Clean — **Rationale**: 노드별 독립 수정/테스트 가능, n8n 시각적 디버깅 최대 활용, 향후 소스 확대 시 유리

### 2.1 Workflow Diagram

```
┌──────────────────────────────────────────────────────────────────────────────┐
│  n8n Workflow: Shopping Platform News Scraper                                 │
├──────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  [1. Schedule Trigger]                                                       │
│        │ (평일 07:00 KST)                                                    │
│        v                                                                     │
│  [2. Keyword List]  ──────────────────────────────────────┐                  │
│        │ (Static: 17 keywords + channel mapping)          │                  │
│        v                                                  │                  │
│  [3. Loop: SplitInBatches]                                │                  │
│        │                                                  │                  │
│        v                                                  │                  │
│  [4. HTTP Request]  (Google News RSS per keyword)         │                  │
│        │                                                  │                  │
│        v                                                  │                  │
│  [5. Wait Node]  (1s delay for rate limit)                │                  │
│        │                                                  │                  │
│        v                                                  │                  │
│  [6. XML Parser]  (RSS XML → JSON)                        │                  │
│        │                                                  │                  │
│        v                                                  │                  │
│  [7. Code: Article Extractor]  (title, link, pubDate 추출)│                  │
│        │                                                  │                  │
│        └──── (loop back to 3) ────────────────────────────┘                  │
│                                                                              │
│  [8. Merge Node]  (모든 키워드 결과 합치기)                                     │
│        │                                                                     │
│        v                                                                     │
│  [9. Code: Time Filter]  (24시간 내 기사만)                                    │
│        │                                                                     │
│        v                                                                     │
│  [10. Code: Exclusion Filter]  (야구/할인 제외)                                │
│        │                                                                     │
│        v                                                                     │
│  [11. Code: Deduplication]  (Levenshtein 유사도 체크)                          │
│        │                                                                     │
│        v                                                                     │
│  [12. Code: Type Classifier]  (기사 유형 분류)                                 │
│        │                                                                     │
│        v                                                                     │
│  [13. OpenAI Node: AI Summary]  (전체 뉴스 트렌드 요약)                         │
│        │                                                                     │
│        v                                                                     │
│  ┌─────┼─────────────────────────┐                                           │
│  │     │    [병렬 출력 분기]       │                                           │
│  │     ├──> [14. Google Sheets]   │                                           │
│  │     │          │               │                                           │
│  │     │          v               │                                           │
│  │     │    [15. Notion]          │                                           │
│  │     │                          │                                           │
│  │     ├──> [16. Slack]           │                                           │
│  │     │                          │                                           │
│  │     └──> [17. Email (Gmail)]   │                                           │
│  └────────────────────────────────┘                                           │
│                                                                              │
└──────────────────────────────────────────────────────────────────────────────┘
```

### 2.2 Data Flow

```
Schedule Trigger
  → Keyword List (static data injection)
    → Loop (SplitInBatches)
      → HTTP Request (RSS XML)
        → Wait (1s)
          → XML Parser (JSON conversion)
            → Article Extractor (structured objects)
    → Merge (all articles combined)
      → Time Filter (24h only)
        → Exclusion Filter (baseball/discount removed)
          → Deduplication (unique articles)
            → Type Classifier (categorized articles)
              → AI Summary (trend analysis)
                → [Parallel] Sheets → Notion
                → [Parallel] Slack
                → [Parallel] Email
```

### 2.3 Node Dependencies

| Node | Depends On | Output To |
|------|-----------|-----------|
| 1. Schedule Trigger | (cron) | 2. Keyword List |
| 2. Keyword List | 1 | 3. Loop |
| 3. Loop (SplitInBatches) | 2 | 4. HTTP Request |
| 4. HTTP Request | 3 | 5. Wait |
| 5. Wait | 4 | 6. XML Parser |
| 6. XML Parser | 5 | 7. Article Extractor |
| 7. Article Extractor | 6 | 3 (loop) or 8. Merge |
| 8. Merge | 7 (all iterations) | 9. Time Filter |
| 9. Time Filter | 8 | 10. Exclusion Filter |
| 10. Exclusion Filter | 9 | 11. Deduplication |
| 11. Deduplication | 10 | 12. Type Classifier |
| 12. Type Classifier | 11 | 13. AI Summary |
| 13. AI Summary | 12 | 14, 16, 17 (parallel) |
| 14. Google Sheets | 13 | 15. Notion |
| 15. Notion | 14 | (end) |
| 16. Slack | 13 | (end) |
| 17. Email | 13 | (end) |

---

## 3. Data Model

### 3.1 Keyword Configuration (Node 2 Input)

```typescript
interface KeywordConfig {
  keyword: string;       // "쿠팡", "11번가", etc.
  channel: "온라인" | "오프라인";
}
```

### 3.2 Raw Article (Node 7 Output)

```typescript
interface RawArticle {
  keyword: string;
  channel: string;
  title: string;
  link: string;
  pubDate: string;       // ISO8601
  source: string;        // RSS source attribution
}
```

### 3.3 Filtered Article (Node 12 Output — Final)

```typescript
interface FilteredArticle {
  keyword: string;
  channel: "온라인" | "오프라인";
  type: "신제품 출시" | "사업 확장" | "물류/배송 관련" | "투자/매출 관련"
      | "인재 채용" | "협업/제휴" | "고객 관련" | "일반 뉴스";
  title: string;
  link: string;
  pubDate: string;
}
```

### 3.4 AI Summary Output (Node 13 Output)

```typescript
interface AISummaryResult {
  summary: string;           // 전체 트렌드 요약 (3-5문장)
  keyInsights: string[];     // 주요 인사이트 (3-5개)
  keywordTrends: {           // 키워드별 동향
    keyword: string;
    trend: string;
  }[];
  articleCount: number;
  articles: FilteredArticle[];  // 원본 기사 목록 패스스루
}
```

---

## 4. Node Detailed Specification

### 4.1 Node 1: Schedule Trigger

| Setting | Value |
|---------|-------|
| Type | Schedule Trigger |
| Rule | Cron |
| Expression | `0 7 * * 1-5` |
| Timezone | Asia/Seoul |

### 4.2 Node 2: Keyword List (Code Node)

**Purpose**: 17개 키워드와 채널 매핑을 정적 데이터로 주입

```javascript
// Output: 17 items, one per keyword
return [
  { json: { keyword: "쿠팡", channel: "온라인" } },
  { json: { keyword: "11번가", channel: "온라인" } },
  { json: { keyword: "지마켓", channel: "온라인" } },
  { json: { keyword: "G마켓", channel: "온라인" } },
  { json: { keyword: "컬리", channel: "온라인" } },
  { json: { keyword: "마켓컬리", channel: "온라인" } },
  { json: { keyword: "네이버쇼핑", channel: "온라인" } },
  { json: { keyword: "배민", channel: "온라인" } },
  { json: { keyword: "배달의민족", channel: "온라인" } },
  { json: { keyword: "무신사", channel: "온라인" } },
  { json: { keyword: "이마트", channel: "오프라인" } },
  { json: { keyword: "SSG", channel: "오프라인" } },
  { json: { keyword: "롯데마트", channel: "오프라인" } },
  { json: { keyword: "홈플러스", channel: "오프라인" } },
  { json: { keyword: "코스트코", channel: "오프라인" } },
  { json: { keyword: "다이소", channel: "오프라인" } },
  { json: { keyword: "올리브영", channel: "오프라인" } },
];
```

### 4.3 Node 3: SplitInBatches

| Setting | Value |
|---------|-------|
| Batch Size | 1 |
| Purpose | 키워드를 하나씩 순차 처리 (rate limit 준수) |

### 4.4 Node 4: HTTP Request

| Setting | Value |
|---------|-------|
| Method | GET |
| URL | `https://news.google.com/rss/search?q={{ encodeURIComponent($json.keyword) }}&hl=ko&gl=KR` |
| Response Format | String (XML) |
| Headers | `User-Agent: Mozilla/5.0 (compatible; n8n-bot)` |
| Timeout | 10000ms |
| Continue On Fail | true |

### 4.5 Node 5: Wait

| Setting | Value |
|---------|-------|
| Amount | 1 |
| Unit | Seconds |
| Purpose | Google News rate limit 방지 |

### 4.6 Node 6: XML Parser (XML Node)

| Setting | Value |
|---------|-------|
| Property Name | data |
| Input: XML field | `{{ $json.data }}` |
| Output | Parsed JSON object with `rss.channel.item[]` |

### 4.7 Node 7: Article Extractor (Code Node)

**Purpose**: RSS JSON에서 기사 객체 추출, 키워드/채널 정보 병합

```javascript
// Input: parsed XML items + keyword context from earlier node
// Output: array of RawArticle objects
const items = $input.first().json.data?.rss?.channel?.item || [];
const keyword = $('Keyword List').first().json.keyword;  // from loop context
const channel = $('Keyword List').first().json.channel;

const articles = (Array.isArray(items) ? items : [items]).map(item => ({
  keyword,
  channel,
  title: item.title || '',
  link: item.link || '',
  pubDate: item.pubDate || '',
  source: (item.source?._text || item.source || '').toString()
}));

return articles.map(a => ({ json: a }));
```

### 4.8 Node 8: Merge

| Setting | Value |
|---------|-------|
| Mode | Append |
| Purpose | Loop에서 수집된 모든 기사를 하나의 배열로 합침 |

### 4.9 Node 9: Time Filter (Code Node)

**Purpose**: 최근 24시간 내 발행 기사만 유지

```javascript
const now = new Date();
const oneDayAgo = new Date(now.getTime() - 24 * 60 * 60 * 1000);

return $input.all().filter(item => {
  const pubDate = new Date(item.json.pubDate);
  return pubDate >= oneDayAgo;
});
```

### 4.10 Node 10: Exclusion Filter (Code Node)

**Purpose**: 야구 관련 키워드, SSG 선수명, "할인" 포함 기사 제외

```javascript
const baseballExcludeKeywords = [
  "랜더스", "야구", "투수", "타자", "홈런", "안타", "포수", "감독", "경기", "KBO",
  "선수", "드래프트", "이닝", "타율", "방어율", "불펜", "선발", "포스트시즌",
  // ... (기존 Apps Script의 전체 100+ 키워드 목록)
];

const ssgPlayerNames = [
  "김건우", "김광현", "김도현", "김민", "김준영",
  // ... (기존 Apps Script의 전체 60+ 선수명 목록)
];

return $input.all().filter(item => {
  const title = item.json.title;

  // "할인" 제외
  if (title.includes("할인")) return false;

  // 야구 키워드 제외
  const lowerTitle = title.toLowerCase();
  const hasBaseball = baseballExcludeKeywords.some(
    word => lowerTitle.includes(word.toLowerCase())
  );
  if (hasBaseball) return false;

  // SSG 선수명 제외
  const hasPlayer = ssgPlayerNames.some(name => title.includes(name));
  if (hasPlayer) return false;

  return true;
});
```

### 4.11 Node 11: Deduplication (Code Node)

**Purpose**: Levenshtein 유사도 기반 중복 기사 제거

```javascript
function getLevenshteinSimilarity(str1, str2) {
  const len1 = str1.length;
  const len2 = str2.length;
  const dp = Array(len1 + 1).fill(null).map(() => Array(len2 + 1).fill(0));

  for (let i = 0; i <= len1; i++) dp[i][0] = i;
  for (let j = 0; j <= len2; j++) dp[0][j] = j;

  for (let i = 1; i <= len1; i++) {
    for (let j = 1; j <= len2; j++) {
      const cost = str1[i-1] === str2[j-1] ? 0 : 1;
      dp[i][j] = Math.min(dp[i-1][j]+1, dp[i][j-1]+1, dp[i-1][j-1]+cost);
    }
  }

  const distance = dp[len1][len2];
  return (Math.max(len1, len2) - distance) / Math.max(len1, len2);
}

const processed = [];
const results = [];

for (const item of $input.all()) {
  // 제목에서 언론사 정보 제거
  const title = item.json.title.replace(/ - [^-]+$/, "");

  const isSimilar = processed.some(
    prev => getLevenshteinSimilarity(prev, title) > 0.2
  );

  if (!isSimilar) {
    processed.push(title);
    results.push({ json: { ...item.json, title } });
  }
}

return results;
```

### 4.12 Node 12: Type Classifier (Code Node)

**Purpose**: 기사 제목 기반 유형 자동 분류

```javascript
return $input.all().map(item => {
  const title = item.json.title;
  let type = "일반 뉴스";

  if (title.includes("출시") || title.includes("신제품")) type = "신제품 출시";
  else if (title.includes("확대") || title.includes("점포") || title.includes("매장")) type = "사업 확장";
  else if (title.includes("배송") || title.includes("물류") || title.includes("운송")) type = "물류/배송 관련";
  else if (title.includes("투자") || title.includes("자금") || title.includes("매출")) type = "투자/매출 관련";
  else if (title.includes("채용") || title.includes("인재") || title.includes("고용")) type = "인재 채용";
  else if (title.includes("협업") || title.includes("파트너십") || title.includes("제휴")) type = "협업/제휴";
  else if (title.includes("고객") || title.includes("이용자")) type = "고객 관련";

  return { json: { ...item.json, type } };
});
```

### 4.13 Node 13: AI Summary (OpenAI Node)

| Setting | Value |
|---------|-------|
| Resource | Chat Message |
| Model | gpt-4o |
| Max Tokens | 1000 |

**System Prompt**:
```
당신은 쇼핑/유통 업계 뉴스 분석 전문가입니다.
주어진 뉴스 기사 목록을 분석하여 다음을 제공하세요:
1. 전체 트렌드 요약 (3-5문장)
2. 주요 인사이트 3-5개 (bullet point)
3. 주목할 키워드별 동향

간결하고 비즈니스 의사결정에 도움이 되는 형태로 작성하세요.
```

**User Message**:
```
오늘의 쇼핑 플랫폼 뉴스 ({{ $input.all().length }}건):

{{ $input.all().map(item =>
  `[${item.json.channel}/${item.json.keyword}] ${item.json.title} (${item.json.type})`
).join('\n') }}
```

### 4.14 Node 14: Google Sheets (Append)

| Setting | Value |
|---------|-------|
| Operation | Append Row |
| Document | (사용자 지정 스프레드시트 ID) |
| Sheet | Sheet1 |
| Columns Mapping | 날짜, 채널, 키워드, 유형, 제목, 링크 |

**Column Mapping**:
| Sheet Column | Value |
|-------------|-------|
| A: 날짜 | `{{ $json.pubDate }}` |
| B: 채널 | `{{ $json.channel }}` |
| C: 키워드 | `{{ $json.keyword }}` |
| D: 유형 | `{{ $json.type }}` |
| E: 제목 | `{{ $json.title }}` |
| F: 링크 | `{{ $json.link }}` |

### 4.15 Node 15: Notion (Create Database Item)

| Setting | Value |
|---------|-------|
| Resource | Database Page |
| Operation | Create |
| Database ID | (사용자 지정 Notion DB ID) |

**Property Mapping**:
| Notion Property | Type | Value |
|----------------|------|-------|
| 제목 | Title | `{{ $json.title }}` |
| 날짜 | Date | `{{ $json.pubDate }}` |
| 채널 | Select | `{{ $json.channel }}` |
| 키워드 | Select | `{{ $json.keyword }}` |
| 유형 | Select | `{{ $json.type }}` |
| 링크 | URL | `{{ $json.link }}` |

### 4.16 Node 16: Slack (Send Message)

| Setting | Value |
|---------|-------|
| Channel | (사용자 지정 채널) |
| Message Type | Block Kit |

**Message Template**:
```
📰 *쇼핑 플랫폼 뉴스 브리핑* ({{ $now.format('yyyy-MM-dd') }})

*🤖 AI 요약:*
{{ $('AI Summary').first().json.message.content }}

━━━━━━━━━━━━━━━━━━━━━

*📋 주요 기사 (총 {{ $input.all().length }}건)*

*[온라인]*
{{ $input.all().filter(i => i.json.channel === '온라인').slice(0, 5).map(i =>
  `• <${i.json.link}|${i.json.title}> (${i.json.keyword})`
).join('\n') }}

*[오프라인]*
{{ $input.all().filter(i => i.json.channel === '오프라인').slice(0, 5).map(i =>
  `• <${i.json.link}|${i.json.title}> (${i.json.keyword})`
).join('\n') }}
```

### 4.17 Node 17: Email (Gmail/Send Email)

| Setting | Value |
|---------|-------|
| To | jh@avent.co.kr, aspark@avent.co.kr |
| Subject | `쇼핑 플랫폼 뉴스 (최근 하루) - {{ $now.format('yyyy-MM-dd HH:mm') }}` |
| Email Type | HTML |

**HTML Body**: 기존 Apps Script와 동일한 테이블 형식 유지

```html
<h2>쇼핑 플랫폼 뉴스 (최근 하루 내 발행분)</h2>
<p><strong>🤖 AI 요약:</strong><br>{{ $('AI Summary').first().json.message.content }}</p>
<hr>
<table border="1" cellspacing="0" cellpadding="8" style="border-collapse:collapse;width:100%;">
  <thead style="background-color:#f2f2f2;">
    <tr>
      <th>#</th><th>채널</th><th>키워드</th><th>유형</th><th>제목</th><th>발행일</th>
    </tr>
  </thead>
  <tbody>
    <!-- n8n expression으로 각 기사를 행으로 렌더링 -->
  </tbody>
</table>
```

> Note: HTML 테이블은 Code Node에서 전체 HTML을 생성하여 Gmail 노드에 전달하는 방식으로 구현

---

## 5. Error Handling

### 5.1 노드별 에러 처리

| Node | Error Type | Handling |
|------|-----------|----------|
| 4. HTTP Request | Timeout / 429 | Continue On Fail = true, 빈 응답 처리 |
| 6. XML Parser | Invalid XML | Continue On Fail, 해당 키워드 스킵 |
| 13. AI Summary | API Error | Fallback: 요약 없이 기사 목록만 발송 |
| 14. Google Sheets | Auth Error | Error workflow 알림 |
| 15. Notion | API Rate Limit | Retry (n8n 내장 retry) |
| 16. Slack | Channel Not Found | Error workflow 알림 |
| 17. Email | SMTP Error | Error workflow 알림 |

### 5.2 Error Notification

- n8n Error Workflow 설정으로 워크플로우 실패 시 관리자에게 알림
- 개별 출력 노드 실패는 다른 출력 채널에 영향을 주지 않음 (병렬 처리)

---

## 6. Security Considerations

- [x] Google Sheets OAuth2 토큰 안전 보관 (n8n credentials)
- [x] OpenAI API Key 암호화 저장 (n8n credentials)
- [x] Slack Bot Token 최소 권한 원칙 적용 (chat:write만)
- [x] Notion API Key 최소 스코프 (데이터베이스 write만)
- [x] HTTP Request User-Agent 설정으로 차단 방지
- [x] 이메일 수신자 하드코딩 (외부 입력 없음 → 인젝션 불가)

---

## 7. Required Credentials

| Credential | Type | Required Permissions |
|-----------|------|---------------------|
| Google Sheets | OAuth2 | spreadsheets (read/write) |
| Notion | API Key | Insert database items |
| Slack | Bot Token (OAuth2) | chat:write |
| Gmail | OAuth2 | Send email |
| OpenAI | API Key | Chat completions |

---

## 8. Test Plan

### 8.1 Test Scope

| Type | Target | Method | Phase |
|------|--------|--------|-------|
| Manual Trigger Test | 전체 워크플로우 | n8n Manual Execution | Do |
| Node Unit Test | 각 Code Node 로직 | 개별 노드 실행 | Do |
| Integration Test | 출력 채널 연결 | 실제 credential 테스트 | Do |

### 8.2 Test Scenarios

| # | Scenario | Expected Result | Verification |
|---|----------|----------------|--------------|
| 1 | 전체 워크플로우 수동 실행 | 17개 키워드 수집 완료, 출력 4채널 정상 | 각 노드 output 확인 |
| 2 | SSG 키워드 야구 기사 필터링 | 야구 관련 기사 0건 통과 | Node 10 output에 야구 기사 없음 |
| 3 | 유사 기사 중복 제거 | 유사도 >0.2 기사 제거됨 | Node 11 output count < Node 10 count |
| 4 | AI 요약 생성 | 3-5문장 요약 + 인사이트 생성 | Node 13 output 내용 확인 |
| 5 | Google Sheets 저장 | 스프레드시트에 행 추가됨 | 실제 시트 확인 |
| 6 | Notion 등록 | DB에 페이지 생성됨 | Notion DB 확인 |
| 7 | Slack 발송 | 채널에 메시지 도착 | Slack 채널 확인 |
| 8 | 이메일 발송 | 수신자에게 HTML 이메일 도착 | 이메일 수신 확인 |
| 9 | HTTP 에러 (키워드 하나 실패) | 나머지 키워드 정상 수집 | Continue On Fail 동작 확인 |
| 10 | AI 노드 실패 | 요약 없이 기사 목록만 발송 | Fallback 동작 확인 |

---

## 9. Implementation Guide

### 9.1 n8n Workflow JSON Structure

```
workflow.json (n8n export)
├── nodes[]          (18 nodes)
├── connections{}    (node linkage)
├── settings{}       (timezone, error workflow)
└── staticData{}     (none needed)
```

### 9.2 Implementation Order

1. [ ] n8n 인스턴스 확인 및 credentials 설정
2. [ ] Schedule Trigger + Keyword List 노드 구성
3. [ ] RSS 수집 루프 (HTTP Request + Wait + XML Parser + Extractor)
4. [ ] 필터링 파이프라인 (Time + Exclusion + Dedup + Classifier)
5. [ ] AI Summary 노드 구성
6. [ ] Google Sheets 출력 노드
7. [ ] Notion 출력 노드
8. [ ] Slack 출력 노드
9. [ ] Email 출력 노드 (HTML 생성 포함)
10. [ ] 에러 처리 및 Error Workflow 연결
11. [ ] 전체 테스트 및 스케줄 활성화

### 9.3 Session Guide

#### Module Map

| Module | Scope Key | Description | Estimated Effort |
|--------|-----------|-------------|:---------------:|
| Credential Setup | `module-1` | n8n credentials 5개 설정 + 연결 테스트 | 짧음 |
| RSS Collection Loop | `module-2` | Node 1-8 (Trigger → Merge) 구현 | 중간 |
| Filtering Pipeline | `module-3` | Node 9-12 (Time → Classifier) 구현 | 중간 |
| AI Summary | `module-4` | Node 13 OpenAI 노드 설정 + 프롬프트 | 짧음 |
| Output Channels | `module-5` | Node 14-17 (Sheets, Notion, Slack, Email) | 중간 |
| Testing & Activation | `module-6` | 전체 테스트 + 스케줄 활성화 | 짧음 |

#### Recommended Session Plan

| Session | Scope | Description |
|---------|-------|-------------|
| Session 1 | `module-1,module-2` | Credentials + RSS 수집 루프 구현 |
| Session 2 | `module-3,module-4` | 필터링 + AI 요약 |
| Session 3 | `module-5,module-6` | 출력 채널 + 테스트 |

---

## Version History

| Version | Date | Changes | Author |
|---------|------|---------|--------|
| 0.1 | 2026-05-04 | Initial draft — Option B (Clean) selected | Team |
| 0.2 | 2026-05-05 | 워크플로우 배포 완료, Credentials 연결 상태 반영 | Team |
| 0.3 | 2026-05-08 | AI Summary: OpenAI → Gemini 변경, Send Email: Gmail OAuth2로 변경, 루프 연결 수정, Merge 노드 제거 | Team |
| 0.4 | 2026-05-10 | Notion 연결 완료, Google Sheets 제거, AI Summary Code 노드 전환, 기사별 3줄 요약 추가 | Team |
| 0.5 | 2026-05-11 | Notion Summary 한줄요약 변경, 야구 필터링 강화 (키워드 31개+선수 4명+타선), 스케줄 활성화 | Team |
| 0.6 | 2026-05-15 | Dedup: Levenshtein→Jaccard(0.3), 원문스크래핑(batchexecute+Jina Reader), Notion 마크다운 본문 블록, 언론사 제외 필터 | Team |

---

## Deployment Status

- **Workflow ID**: `6t0bgNHo7yGWM3PD`
- **n8n Instance**: `n8n.gongbaksoo.com`
- **Deployed Nodes**: 20개
- **Active**: Yes (평일 오전 7시 KST)

### Credential Mapping (실제 연결 상태)

| Node | Credential Type | Credential Name | ID | Status |
|------|----------------|-----------------|-----|--------|
| AI Summary | (Code 노드 내 HTTP 호출) | Google Gemini API Key | - | ✅ 테스트 성공 |
| Notion | notionApi | Notion account | I34bqtnWMHogzUqF | ✅ 테스트 성공 |
| Send Email | gmailOAuth2 | Gmail OAuth2 | spDs5DdZcw6Xtski | ✅ 테스트 성공 |
| Slack | slackOAuth2Api | ? | ? | ⏳ (노드 비활성화) |

### Design Deviation Notes

- AI Summary 노드: OpenAI GPT-4o → Gemini 2.5 Flash로 변경. n8n Gemini 노드 호환 문제로 Code 노드 + HTTP Request 방식으로 전환. Gemini API 2회 호출 (전체 요약 + 기사별 3줄 요약). thinking 비활성화 필요 (thinkingBudget: 0)
- Send Email 노드: Gmail 노드 + Gmail OAuth2 자격 증명 사용 (Google Service Account는 Gmail API 미지원)
- Merge All Articles 노드: SplitInBatches가 자동 누적하므로 불필요하여 제거
- Google Sheets 노드: 불필요하여 제거 (Notion으로 대체)
- Loop Over Keywords: 완료 출력(output 0)이 Time Filter에 직접 연결
- Notion 노드: 링크된 DB가 아닌 원본 DB ID 사용 필수. 제목+URL만 입력하면 Notion AI가 태그/관련분야/Summary 자동 채움 → 하지만 n8n에서 직접 채우는 것으로 변경
- n8n 에디터 캐시: API로 워크플로우 업데이트 후 브라우저 새로고침(F5) 필수
- Notion Summary: 기사별 한줄 요약 (Gemini 생성), 텍스트: 3줄 요약
- 야구 필터링: 기존 97개 키워드 + 31개 추가 (잠실, 두산, 한화, LG 등 구단/구장) + SSG 선수명 4명 추가 (오명진, 배동현, 박준순, 잭 로그) + "타선"
- Deduplication: Levenshtein → Jaccard 단어 유사도 클러스터링 (임계값 0.3, 키워드별 그룹화, 키워드당 Top 5)
- 원문 스크래핑: Google News batchexecute(signature/timestamp 필수) → Jina Reader API(r.jina.ai) → Gemini 마크다운 구조화
- Notion 본문: Build Notion Blocks(마크다운→블록 변환) + Notion Add Content(PATCH blocks/{id}/children)
- 스크래핑 불가 언론사: Exclusion Filter에서 source 기반 제외 (11곳: 뉴시스, 브릿지경제, 헤럴드경제, v.daum.net, 네이트, 아주경제, 파이낸셜뉴스, 게임플, 바이오타임즈, 로이슈, 더스쿠프)
- Code 노드 타임아웃: N8N_RUNNERS_TASK_TIMEOUT=1800 (30분, Docker 환경변수)
- Gmail OAuth: Google Cloud Console에서 프로덕션 모드 전환 (7일 토큰 만료 방지)
