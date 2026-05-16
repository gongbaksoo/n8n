# n8n 뉴스 스크래퍼 — 원문 스크래핑 + 중복 클러스터링 적용 가이드

> 이 문서는 Shopping Platform News Scraper(6t0bgNHo7yGWM3PD)에서 구현/검증된 내용을 다른 n8n 워크플로우에 적용하기 위한 참고 자료입니다.

---

## 1. Google News 원문 스크래핑 (3단계)

Google News RSS의 기사 링크(`news.google.com/rss/articles/CBMi...`)는 실제 기사 URL이 아니라 protobuf로 인코딩된 리다이렉트 URL입니다. 실제 기사 본문을 가져오려면 3단계가 필요합니다.

### Step 1: Google News 페이지에서 signature/timestamp 추출

Google News 기사 페이지 HTML에 디코딩에 필요한 인증값이 숨어 있습니다.

```javascript
// article ID 추출 (URL에서)
const articleId = link.match(/\/(?:rss\/)?articles\/([^?]+)/)?.[1];

// Google News 기사 페이지 접근 (두 URL 중 하나 시도)
const urls = [
  'https://news.google.com/articles/' + articleId,
  'https://news.google.com/rss/articles/' + articleId
];

const html = await this.helpers.httpRequest({
  method: 'GET', url: urls[0],
  headers: {
    'User-Agent': 'Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/120.0.0.0 Safari/537.36',
    'Accept-Language': 'ko-KR,ko;q=0.9'
  },
  followRedirect: true, timeout: 15000, json: false
});

// HTML에서 signature, timestamp 추출
const signature = html.match(/data-n-a-sg="([^"]+)"/)?.[1];
const timestamp = html.match(/data-n-a-ts="([^"]+)"/)?.[1];
```

- 첫 번째 URL(`/articles/`)이 실패하면 두 번째(`/rss/articles/`)로 폴백
- signature/timestamp가 없으면 디코딩 불가 → 해당 기사 스크래핑 포기

### Step 2: batchexecute API로 실제 기사 URL 디코딩

Google 내부 API에 signature/timestamp와 함께 요청하면 실제 기사 URL을 반환합니다.

```javascript
// payload 구성 (Python googlenewsdecoder 소스 코드 기반, 이 형식 그대로 사용할 것)
const paramsStr = '["garturlreq",[["X","X",["X","X"],null,null,1,1,"US:en",null,1,null,null,null,null,null,0,1],"X","X",1,[1,1,1],1,1,null,0,0,null,0],"' + articleId + '",' + timestamp + ',"' + signature + '"]';
const payload = ['Fbv4je', paramsStr];
const reqBody = 'f.req=' + encodeURIComponent(JSON.stringify([[payload]]));

const resp = await this.helpers.httpRequest({
  method: 'POST',
  url: 'https://news.google.com/_/DotsSplashUi/data/batchexecute',
  headers: {
    'Content-Type': 'application/x-www-form-urlencoded;charset=utf-8',
    'Referer': 'https://news.google.com/',
    'User-Agent': 'Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36'
  },
  body: reqBody,
  json: false
});

// 응답 파싱
const text = resp;
const jsonPart = text.split('\n\n')[1];       // 첫 번째 줄은 ")]}'" 접두어
const parsed = JSON.parse(jsonPart);
const trimmed = parsed.slice(0, -2);           // 마지막 2개 요소 제거
const innerData = JSON.parse(trimmed[0][2]);   // 내부 JSON 파싱
const realUrl = innerData[1];                  // 실제 기사 URL
```

**주의사항:**
- paramsStr의 배열 구조를 절대 변경하지 말 것. `articleId`, `timestamp`, `signature`의 위치가 핵심
- `timestamp`는 숫자(따옴표 없이), `signature`는 문자열(따옴표 포함)
- locale 배열의 `"X"` 값은 플레이스홀더로 그대로 사용

### Step 3: Jina Reader API로 기사 본문 마크다운 추출

실제 기사 URL을 Jina Reader에 전달하면 깨끗한 마크다운으로 반환됩니다.

```javascript
const jinaResp = await this.helpers.httpRequest({
  method: 'GET',
  url: 'https://r.jina.ai/' + realUrl,    // URL 앞에 r.jina.ai/ 붙이기만 하면 됨
  headers: {
    'Accept': 'text/markdown',
    'X-Return-Format': 'markdown',
    'X-Timeout': '20'
  },
  timeout: 30000, json: false
});

const articleMarkdown = jinaResp;  // 깨끗한 마크다운 텍스트
```

**Jina Reader 무료 사용 조건:**
- API 키 없이: 분당 20회
- 무료 API 키 발급 시: 분당 200회
- 기사 10~20건 처리에는 무료 티어로 충분

### 주의사항 및 실패 케이스

| 실패 원인 | 증상 | 대응 |
|----------|------|------|
| signature/timestamp 추출 실패 | HTML에 `data-n-a-sg` 없음 | 해당 기사 스크래핑 포기 |
| batchexecute 응답에 URL 없음 | `parsed[0][2]`가 null | 해당 기사 스크래핑 포기 |
| Jina Reader HTTP 451 | Google News URL 직접 전달 시 | 반드시 실제 URL로 디코딩 후 전달 |
| Jina Reader 콘텐츠 부족 | 반환 텍스트 < 300자 | 페이월/로그인 필요 사이트 |
| 특정 언론사 반복 실패 | 뉴시스, 브릿지경제 등 | Exclusion Filter에서 source 기반 제외 |

### 스크래핑 불가 언론사 제외 방법

Exclusion Filter 코드에 source 기반 필터 추가:

```javascript
const excludeSources = ["뉴시스", "브릿지경제"];

return $input.all().filter(item => {
  const source = item.json.source || '';
  if (excludeSources.some(s => source.includes(s))) return false;
  // ... 기존 필터 로직
  return true;
});
```

---

## 2. 중복 기사 클러스터링 (Jaccard 단어 유사도)

같은 이슈를 여러 언론사가 보도하면 중복 기사가 쌓입니다. 제목의 단어 유사도를 비교하여 같은 이슈끼리 묶고, 대표 기사 1건만 남깁니다.

### 알고리즘

```
입력: 기사 배열 (title, channel 필드 필수)
출력: 클러스터 대표 기사 배열 (issueScore 추가)
```

### n8n Code 노드 구현

```javascript
const SIMILARITY_THRESHOLD = 0.3;  // 0.3 이상이면 같은 이슈로 판단

// Jaccard 단어 유사도: 두 제목의 단어 집합 비교
function getWordSimilarity(str1, str2) {
  const words1 = new Set(str1.replace(/[^가-힣a-zA-Z0-9\s]/g, '').split(/\s+/).filter(w => w.length > 1));
  const words2 = new Set(str2.replace(/[^가-힣a-zA-Z0-9\s]/g, '').split(/\s+/).filter(w => w.length > 1));
  if (words1.size === 0 || words2.size === 0) return 0;
  let intersection = 0;
  for (const w of words1) { if (words2.has(w)) intersection++; }
  return intersection / (words1.size + words2.size - intersection);
}

// 채널별 그룹화 (채널 내에서만 비교하여 성능 확보)
const channelArticles = {};
for (const item of $input.all()) {
  const ch = item.json.channel;
  if (!channelArticles[ch]) channelArticles[ch] = [];
  channelArticles[ch].push(item);
}

const results = [];

for (const [channel, articles] of Object.entries(channelArticles)) {
  const clusters = [];  // [{ representative, repTitle, count }]

  for (const item of articles) {
    const title = item.json.title.replace(/ - [^-]+$/, "");  // 언론사 정보 제거
    let merged = false;

    // 기존 클러스터들과 비교
    for (const cluster of clusters) {
      if (getWordSimilarity(cluster.repTitle, title) > SIMILARITY_THRESHOLD) {
        cluster.count++;     // 같은 이슈 → 카운트만 증가
        merged = true;
        break;
      }
    }

    if (!merged) {
      // 새 클러스터 생성 (이 기사가 대표)
      clusters.push({ representative: item, repTitle: title, count: 1 });
    }
  }

  // 이슈점수(=보도 건수) 내림차순 정렬
  clusters.sort((a, b) => b.count - a.count);

  for (const cluster of clusters) {
    results.push({
      json: {
        ...cluster.representative.json,
        title: cluster.repTitle,
        issueScore: cluster.count,        // 같은 이슈 보도 건수
        similarCount: cluster.count - 1   // 제거된 중복 건수
      }
    });
  }
}

results.sort((a, b) => b.json.issueScore - a.json.issueScore);
return results;
```

### 동작 예시

```
입력 6건:
① 이마트, 신세계건설에 5000억 수혈 (뉴스핌)
② 이마트, 신세계건설에 5000억원 유상증자 (조선비즈)
③ 이마트, 신세계건설에 5천억 유상증자 (다음)
④ "고환율에도 해외보다 싸다"…이마트, 와인장터 연다
⑤ 이마트·롯데마트, AI로 '성장판' 넓힌다
⑥ 이마트, 신세계건설에 5000억원 유상증자…현금·부동산 (다음)

클러스터링 결과:
- 클러스터 A: ①②③⑥ (issueScore: 4) → 대표: ① (가장 먼저 수집된 기사)
- 클러스터 B: ④ (issueScore: 1)
- 클러스터 C: ⑤ (issueScore: 1)

출력 3건: ①, ④, ⑤
```

### 임계값 튜닝 가이드

| 임계값 | 효과 | 위험 |
|--------|------|------|
| 0.2 | 공격적 병합, 중복 많이 제거 | 다른 이슈가 잘못 합쳐질 수 있음 |
| **0.3** | **균형** (현재 사용 중) | 일부 유사 기사가 남을 수 있음 |
| 0.4 | 보수적 병합 | 같은 이슈의 유사 기사가 남음 |

### Levenshtein 대신 Jaccard를 쓰는 이유

| | Levenshtein | Jaccard |
|---|---|---|
| 비교 방식 | 문자 단위 편집거리 | 단어 집합 교집합/합집합 |
| 시간복잡도 | O(m * n) per pair | O(m + n) per pair |
| 700건 기사 | 타임아웃 발생 | 61ms 완료 |
| 적합한 상황 | 짧은 문자열, 오타 검출 | 뉴스 제목 유사도 비교 |

---

## 3. 워크플로우 적용 위치

기존 워크플로우에 적용할 때의 노드 위치:

```
... → Exclusion Filter
        → [교체] Deduplication (Jaccard 클러스터링)
        → Type Classifier
        → [수정] AI Summary (원문 스크래핑 + 마크다운 생성 추가)
        → Prepare Output Data
        → Notion (페이지 생성)
        → [신규] Build Notion Blocks (마크다운 → 블록 변환)
        → [신규] Notion Add Content (PATCH blocks/{id}/children)
```

### 노드별 변경 요약

| 노드 | 변경 | 설명 |
|------|------|------|
| Exclusion Filter | 수정 | `excludeSources` 배열 추가 |
| Deduplication | 교체 | Levenshtein → Jaccard 클러스터링 코드로 교체 |
| AI Summary | 수정 | 3차 호출 추가 (원문 스크래핑 + Gemini 마크다운 생성) |
| Build Notion Blocks | 신규 | 마크다운 텍스트 → Notion 블록 JSON 변환 |
| Notion Add Content | 신규 | HTTP Request로 `PATCH /v1/blocks/{pageId}/children` 호출 |

---

## 4. Notion 마크다운 본문 블록

Notion API는 마크다운을 직접 받지 않으므로, 마크다운 텍스트를 Notion 블록 객체로 변환해야 합니다.

### 본문 구조

```markdown
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
- 📅 발행일: ...
- 📂 카테고리: ...
- 🏷️ 태그: ...
- ⭐ 유형: ...
- 🔗 원본: ...
```

### Notion Add Content 노드 설정 (HTTP Request)

| 설정 | 값 |
|------|-----|
| Method | PATCH |
| URL | `https://api.notion.com/v1/blocks/{{ $json.pageId }}/children` |
| Authentication | Predefined Credential Type → notionApi |
| Headers | `Notion-Version: 2022-06-28` |
| Body | JSON → `{{ JSON.stringify($json.requestBody) }}` |

### 스크래핑 실패 시

본문에 callout 블록으로 "원문 스크래핑 실패" 표시:

```json
{
  "object": "block",
  "type": "callout",
  "callout": {
    "icon": { "type": "emoji", "emoji": "⚠️" },
    "rich_text": [{ "type": "text", "text": { "content": "원문 스크래핑 실패로 요약을 생성하지 못했습니다." } }]
  }
}
```

---

## 5. 참고 자료

- [googlenewsdecoder Python 패키지](https://github.com/SSujitX/google-news-url-decoder) — batchexecute payload 형식의 원본
- [Jina Reader API](https://jina.ai/reader/) — URL → 마크다운 변환
- [n8n 워크플로우 #3150](https://n8n.io/workflows/3150) — Google News URL 디코딩 워크플로우
- [n8n 커뮤니티 토론](https://community.n8n.io/t/solved-base64-decode-google-news-urls-with-a-function-node/29019)
