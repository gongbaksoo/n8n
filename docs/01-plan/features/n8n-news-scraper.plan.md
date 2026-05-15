# Plan: n8n News Scraper

## Executive Summary

| 관점 | 설명 |
|------|------|
| **Problem** | 기존 Google Apps Script 기반 뉴스 수집 시스템의 확장성 한계 (AI 분석 불가, 단일 출력 채널, 아카이빙 부재) |
| **Solution** | n8n 워크플로우로 마이그레이션하여 AI 요약, 멀티채널 알림, 구조화된 저장소(Sheets+Notion) 통합 |
| **UX Effect** | 팀원들이 매일 오전 7시 Slack+이메일로 AI 요약 포함 뉴스 브리핑을 받고, Notion/Sheets에서 이력 조회 가능 |
| **Core Value** | 쇼핑 플랫폼 경쟁사 동향을 빠르게 파악하여 비즈니스 의사결정 속도 향상 |

---

## 1. User Intent Discovery

### 1.1 Core Problem
- 기존 Apps Script 뉴스 수집 시스템을 n8n으로 이전하면서 기능을 확장
- AI 요약, 멀티채널 알림, 구조화된 아카이빙 추가

### 1.2 Target Users
- 팀/조직 내부 구성원 (비즈니스 의사결정자)
- 수신자: jh@avent.co.kr, aspark@avent.co.kr (기존 수신자 유지)

### 1.3 Success Criteria
- 기존 Apps Script와 동일한 수집 품질 유지 (키워드, 필터링, 분류)
- AI 요약이 포함된 뉴스 브리핑이 매일 평일 오전 7시 발송
- Slack + 이메일 동시 발송
- Google Sheets + Notion에 기사 아카이빙

### 1.4 Constraints
- n8n 워크플로우 단일 구성 (멀티 워크플로우 불필요)
- Google News RSS API rate limit 고려 (키워드 간 딜레이)
- 기존 야구 뉴스 필터링 로직 반드시 유지 (SSG 랜더스 오탐 방지)

---

## 2. Architecture Overview

```
[Schedule Trigger: 평일 오전 7시]
    |
    v
[Loop: 17개 키워드별 Google News RSS 호출]
    |
    v
[Code Node: 필터링 + 분류 + 중복 제거]
    |
    v
[AI Node: 전체 뉴스 트렌드 요약]
    |
    v
[Split: 병렬 출력]
  |--- [Google Sheets: 개별 기사 행 추가]
  |       |
  |       v
  |    [Notion: DB 항목 생성]
  |--- [Slack: AI 요약 + 주요 기사]
  |--- [Email: HTML 테이블 (기존 형식)]
```

---

## 3. Component Specification

### 3.1 Schedule Trigger
- **Type**: Cron Trigger
- **Schedule**: 평일(월-금) 오전 7:00 KST
- **Timezone**: Asia/Seoul

### 3.2 RSS 수집 (HTTP Request Loop)
- **Source**: Google News RSS (`https://news.google.com/rss/search?q={keyword}&hl=ko&gl=KR`)
- **Keywords** (17개):
  - 온라인: 쿠팡, 11번가, 지마켓, G마켓, 컬리, 마켓컬리, 네이버쇼핑, 배민, 배달의민족, 무신사
  - 오프라인: 이마트, SSG, 롯데마트, 홈플러스, 코스트코, 다이소, 올리브영
- **Rate Limit**: 키워드 간 1초 딜레이

### 3.3 필터링 및 분류 (Code Node)
- **시간 필터**: 최근 24시간 내 발행 기사만
- **제외 필터**:
  - "할인" 포함 기사 제외
  - 야구 관련 키워드 (약 130개+) 포함 기사 제외
  - SSG 랜더스 선수명 (약 60명) 포함 기사 제외
  - 스크래핑 불가 언론사 제외 (뉴시스, 브릿지경제)
- **중복 제거**: Jaccard 단어 유사도 > 0.3인 기사 클러스터링 (채널별 그룹화, 대표 기사 1건 선정)
- **유형 자동 분류**:
  - 신제품 출시 / 사업 확장 / 물류배송 / 투자매출 / 인재채용 / 협업제휴 / 고객 관련 / 일반 뉴스
- **출력 스키마**:
  ```json
  {
    "keyword": "string",
    "channel": "온라인|오프라인",
    "type": "string",
    "title": "string",
    "link": "string",
    "pubDate": "ISO8601"
  }
  ```

### 3.4 AI 요약 (Gemini Code Node)
- **Input**: 필터링된 전체 기사 목록
- **Output**: 트렌드 요약 (3-5문장), 주요 인사이트, 키워드별 동향, 기사별 한줄/3줄 요약
- **Model**: Google Gemini 2.5 Flash (무료 티어)
- **원문 스크래핑**: Google News URL 디코딩(batchexecute) + Jina Reader API로 원문 마크다운 추출
- **마크다운 생성**: 원문 기반 구조화된 마크다운 (요약/본문/핵심인사이트/메타정보)

### 3.5 Notion 등록
- **Target**: Notion Database (업계 동향)
- **Properties**: 제목, URL, 태그(keyword), 관련분야(channel), Summary(한줄요약), 텍스트(3줄요약)
- **Page Content**: AI 마크다운 본문 블록 (Notion API blocks/children)
- **Mode**: Create page + Append content blocks

### 3.7 Slack 알림
- **Channel**: 지정된 Slack 채널
- **Format**: AI 요약 + 온라인/오프라인 구분 주요 기사 목록
- **Mention**: 없음 (정보 공유 목적)

### 3.8 이메일 발송
- **To**: jh@avent.co.kr, aspark@avent.co.kr
- **Subject**: `쇼핑 플랫폼 뉴스 (최근 하루) - {날짜}`
- **Body**: HTML 테이블 (기존 형식 유지: #, 채널, 키워드, 유형, 제목, 발행일)

---

## 4. Alternatives Explored

| 접근법 | 설명 | 선택 여부 |
|--------|------|-----------|
| **A: 단일 워크플로우** | 전체 파이프라인을 하나의 워크플로우로 구현 | **선택** |
| B: 멀티 워크플로우 | 수집/처리/출력 단계별 분리 | 미채택 (현재 규모에 과잉) |
| C: 하이브리드 | 소스별 수집 분리 + 공통 처리 | 미채택 (소스 확대 시 재검토) |

**선택 이유**: 17개 키워드의 현재 규모에서 단일 워크플로우가 관리 용이하고 디버깅이 편리함.

---

## 5. YAGNI Review

### MVP 포함 (v1)
- [x] Google News RSS 키워드 수집
- [x] 야구/할인 필터링 + Jaccard 클러스터링 중복 제거 + 유형 분류
- [x] AI 전체 뉴스 요약 + 기사별 한줄/3줄 요약
- [x] 원문 스크래핑 (Google News URL 디코딩 + Jina Reader)
- [x] Notion DB 등록 + 마크다운 본문 블록
- [x] 이메일 발송 (HTML 테이블)

### Out of Scope (v2+)
- [ ] 추가 뉴스 소스 (특정 사이트 스크래핑, SNS)
- [ ] Slack 알림 연결
- [ ] 사용자별 키워드 커스터마이징
- [ ] 대시보드/웹 UI

---

## 6. Technical Notes

### n8n 필요 Credentials
- Google Sheets OAuth2
- Notion API Key
- Slack Bot Token (or OAuth2)
- Gmail OAuth2 (Google Cloud OAuth2 Client ID 필요)
- Google Gemini API Key (Google AI Studio 발급)

### 주의사항
- Google News RSS는 공식 API가 아니므로 차단 가능성 있음 → User-Agent 설정 필요
- SSG 키워드는 야구팀과 겹치므로 필터링 로직 필수
- Levenshtein 유사도 계산은 Code Node 내에서 JavaScript로 구현

---

## 7. Brainstorming Log

| Phase | 결정 사항 |
|-------|-----------|
| Intent Discovery | 핵심: 기존 Apps Script → n8n 이전 + 기능 확장 |
| Target Users | 팀/조직 내부 (비즈니스팀) |
| Approach | 단일 워크플로우 (A안 채택) |
| MVP Scope | 전체 기능 포함 (소스 확대만 제외) |
| Schedule | 평일 오전 7시 KST |
| Output | 이메일 + Slack 병행, Sheets + Notion 저장 |

---

## 8. Implementation Status

| 항목 | 상태 | 비고 |
|------|------|------|
| n8n 워크플로우 생성 | ✅ 완료 | ID: 6t0bgNHo7yGWM3PD, 20개 노드 |
| Jaccard 클러스터링 | ✅ 완료 | Levenshtein → Jaccard 단어 유사도 (임계값 0.3) |
| AI Summary (Gemini 2.5 Flash) | ✅ 작동 | 전체 요약 + 한줄/3줄 요약 + 원문 마크다운 생성 |
| 원문 스크래핑 | ✅ 작동 | Google News URL 디코딩(batchexecute) + Jina Reader (성공률 ~80%) |
| Notion 마크다운 본문 | ✅ 작동 | Build Notion Blocks + Notion Add Content (API blocks/children) |
| Gmail OAuth2 Credential | ✅ 연결 | id: spDs5DdZcw6Xtski |
| Notion Credential | ✅ 연결 | id: I34bqtnWMHogzUqF |
| Notion DB 연결 | ✅ 성공 | 업계 동향 DB (id: 1e26d523e164805b9950f5197b8b216e) |
| 스크래핑 불가 언론사 제외 | ✅ 완료 | 뉴시스, 브릿지경제 |
| Google Sheets | ❌ 제거 | 불필요하여 노드 삭제 |
| 스케줄 활성화 | ✅ 완료 | 평일 오전 7시 KST, Active = true |
| Slack Credential | ⏳ 비활성화 | 노드 비활성화 상태, 추후 연결 |

---

## Next Step

```
1. Slack credential 연결 및 노드 활성화
2. 자동 실행 모니터링 (평일 오전 7시)
3. 스크래핑 실패 언론사 추가 모니터링 및 excludeSources 업데이트
```
