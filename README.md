# blogstudio GEO Audit

운영 중인 콘텐츠 플랫폼 [blogstudio.co.kr](https://blogstudio.co.kr)의 글을 자동으로 모아 **SEO·AEO·GEO 규칙으로 점검하고, LLM(Claude)이 고칠 방법을 제안하는 n8n 워크플로**입니다.
점수는 규칙(코드)이 매기고, Claude는 실패·경고 항목의 개선안만 씁니다. Claude의 응답은 코드로 다시 검증하고, 문제가 있으면 사람이 보도록 표시합니다.
결과를 메일로 보고하는 단계를 이어서 만들고 있습니다.

검색이 AI 답변으로 옮겨 가면서 "검색 결과에 잘 나오는가(SEO)"만이 아니라 "질문에 대한 답으로 뽑히는가(AEO)", "생성형 AI가 읽고 인용하기 좋은가(GEO)"까지 봐야 합니다.
이 프로젝트는 그 점검을 사람이 매번 하지 않도록 반복 가능한 자동화로 만든 것입니다.

## 진행 상황

| 단계 | 내용 | 상태 |
|---|---|---|
| 수집 | 사이트맵에서 글 주소 수집 | 완료 |
| 가져오기 | 글마다 HTML 받기 (JS 미실행, 1초 간격) | 완료 |
| 추출 | 제목·설명·canonical·h1·소제목·OG·JSON-LD·본문·링크 추출 | 완료 |
| 판정 | 규칙 14개로 판정하고 SEO·AEO·GEO 점수 계산 | 완료 |
| 개선안 | Claude Sonnet 5.5가 실패·경고 항목의 개선안 작성 (JSON 스키마 구조화 출력) | 완료 |
| 검증 | 정상 종료·JSON·글자 수·원문에 없는 숫자 검사, 비용 계산 | 완료 |
| 보고 | HTML 메일 리포트 | 진행 중 |

## 구조

```
Manual Trigger (Schedule Trigger로 바꿀 예정)
 → Fetch Sitemap         sitemap.xml 받기
 → Parse Sitemap         XML → JSON
 → Split URLs            주소 하나 = item 하나
 → Only Articles         /article/ 주소만 남기기
 → Fetch Article         글 HTML 받기 (User-Agent: blogstudio-geo-audit-bot, 1초 간격)
 → Extract SEO Fields    HTML에서 점검할 값 추출
 → Check Rules           규칙 판정 + 점수 (JavaScript)
 → Limit                 Claude에 보낼 글 수 제한 (테스트용, 기본 2개)
 → Build Claude Request  실패·경고 항목과 글 내용으로 요청 만들기
 → Claude                Messages API 호출 (1초 간격, 실패하면 한 번 더 시도)
 → Validate              응답 검증 + 비용 계산
 → 메일 리포트            (진행 중)
```

## 점검 규칙

| 구분 | 규칙 | 통과 조건 | 가중치 |
|---|---|---|---|
| SEO | status | HTTP 200 | 3 |
| SEO | indexable | noindex 없음 | 3 |
| SEO | title | 10~60자 | 1 |
| SEO | description | 50~160자 | 2 |
| SEO | canonical | 자기 주소를 가리킴 | 2 |
| SEO | h1 | 정확히 1개 | 1 |
| SEO | open-graph | og:title·og:description·og:image 모두 있음 | 1 |
| SEO | json-ld | BlogPosting에 headline·datePublished·author 있음 | 2 |
| AEO | question-heading | 제목·소제목 중 질문형이 있음 | 1 |
| AEO | answer-first | 첫 문단 40~300자 (핵심 답을 먼저 줌) | 2 |
| AEO | paragraph-structure | 소제목 2개 이상, 800자 넘는 문단 없음 | 1 |
| GEO | metadata-in-head | 제목·설명·canonical이 `<head>` 안에 있음 | 2 |
| GEO | ssr-text | JS 없이 보이는 본문 500자 이상 | 3 |
| GEO | citations | 다른 사이트로 가는 출처 링크 1개 이상 | 1 |
| GEO | concrete-facts | 숫자·날짜 같은 구체적 정보 3개 이상 | 1 |

점수는 구분별로 `Σ(가중치 × PASS 1 / WARN 0.5 / FAIL 0) ÷ Σ가중치 × 100`입니다.
이 프로젝트가 정한 규칙의 통과율일 뿐, 검색 순위나 AI 답변 인용을 보장하지 않습니다.

## 설계에서 정한 것

### 점검
- **판정은 코드로, LLM은 개선안만**: 같은 글이면 언제나 같은 점수가 나와야 기준을 설명하고 검증할 수 있습니다. 그래서 LLM은 점수를 매기지 않고, 실패·경고 항목을 고칠 제안만 씁니다.
- **JS를 실행하지 않고 읽기**: 많은 AI 크롤러는 자바스크립트를 실행하지 않습니다. 서버가 내려준 HTML만으로 점검해야 AI가 보는 화면과 같아집니다.
- **크롤러로서 지킬 것**: User-Agent에 `bot`을 넣어 크롤러임을 밝히고, 글 사이에 1초 간격을 둡니다.

### LLM (Claude)
- **모델은 Sonnet 5.5, effort `low`**: 무엇이 문제인지는 규칙이 이미 정했고, Claude는 정해진 형식으로 짧은 제안만 씁니다. 그래서 가장 비싼 모델(Opus)은 쓰지 않았습니다. effort `low`에서는 단순한 요청이면 생각 단계를 건너뛰어, 첫 실행에서 thinking 토큰이 0이었습니다.
- **구조화 출력**: `output_config.format`에 JSON 스키마를 넣어 응답이 항상 같은 모양의 JSON으로 오게 했습니다. 스키마로 강제할 수 없는 글자 수 제한은 Validate에서 코드로 확인합니다.
- **글쓴이가 고칠 수 없는 문제는 넘기지 않기**: `metadata-in-head`는 사이트 전체의 Next.js 설정 문제라 Claude에게 보내지 않습니다.
- **프롬프트 인젝션 방어**: 글 내용은 `<page>` 태그 안에 데이터로만 넣고, 그 안의 지시는 따르지 말라고 시스템 프롬프트에 적었습니다. 본문에 `</page>`가 있으면 지워서 태그를 닫지 못하게 합니다.
- **지어내지 않기**: 원문에 있는 사실만 쓰게 했습니다. 근거가 필요하면 링크를 만들어 내지 말고 "출처 추가 필요"라고 쓰게 했습니다.

### 검증과 실패 처리
- **LLM 응답은 그대로 믿지 않기**: Validate가 아래 항목을 확인하고, 하나라도 걸리면 `needsReview: true`로 표시해 사람이 보게 합니다. 개선안을 글에 자동으로 반영하지는 않습니다.
  - `stop_reason`이 `end_turn`인지 (거절 `refusal`, 잘림 `max_tokens` 걸러내기)
  - JSON으로 읽히는지
  - 메타 설명 제안이 50~160자, 첫 문단 제안이 40~300자인지
  - 메타 설명·첫 문단 제안에 Claude가 본 원문에 없는 숫자가 있는지
- **글 하나가 실패해도 전체는 계속**: Claude 노드는 실패하면 5초 뒤 한 번 더 시도합니다. 그래도 실패하면 오류를 다음 노드로 넘기고(On Error: Continue), Validate가 그 글을 "검토 필요"로 표시합니다.

### 운영
- **n8n 운영**: 버전을 고정(2.47.7)하고 `127.0.0.1`에만 열어 둡니다. 2026년에 n8n의 치명적 취약점이 여러 번 공개됐기 때문입니다.
- **비밀값은 n8n 자격 증명에만**: API 키와 메일 계정은 n8n Credentials에 저장하므로 워크플로 JSON과 이 저장소에는 들어가지 않습니다.

## 첫 실행 결과 (2026-11-14, 글 17개)

평균 점수: **SEO 98.8 · AEO 78.1 · GEO 76.8**

| 걸린 규칙 | 글 수 | 의미 |
|---|---|---|
| metadata-in-head | 17 | AI 크롤러에게는 제목·설명·canonical이 `<head>`가 아니라 `<body>` 뒤쪽에 들어감 |
| citations | 17 | 다른 사이트로 가는 출처 링크가 없음 (예: 질병관리청 자료를 인용하지만 링크가 없음) |
| question-heading | 12 | 제목·소제목이 질문형이 아님 |
| answer-first | 6 | 첫 문단이 핵심 답을 바로 주지 않음 |
| paragraph-structure | 6 | 소제목이 부족하거나 문단이 너무 김 |
| open-graph | 6 | 대표 이미지(og:image)가 없음 |

### 가장 큰 발견: AI 크롤러에게는 메타데이터가 head 밖으로 나간다

같은 글을 User-Agent만 바꿔 받아 봤습니다.

| User-Agent | 제목·설명·canonical 위치 |
|---|---|
| 네이버 Yeti, 페이스북 | `<head>` 안 |
| Googlebot, GPTBot, ClaudeBot, PerplexityBot, 일반 브라우저 | `<body>` 뒤쪽 |

Next.js는 메타데이터를 페이지와 함께 스트리밍하고, HTML만 읽는 봇 목록(`htmlLimitedBots`)에 있는 UA에게만 `<head>`에 넣어 줍니다.
Googlebot은 JS를 실행하므로 문제가 없지만, GPTBot·ClaudeBot·PerplexityBot 같은 AI 크롤러는 이 목록에 없어 `<body>` 뒤쪽에서 받습니다.
`<head>`만 읽는 크롤러라면 제목과 설명을 놓칠 수 있습니다. 개선 방향은 `htmlLimitedBots`에 AI 크롤러 UA를 추가하는 것입니다.

결과 한 건의 전체 모양은 [examples/sample-audit-result.json](examples/sample-audit-result.json)에 있습니다.

## Claude 개선안 예시와 비용 (2026-11-14, 글 2개)

이유식 글(첫 문단 34자, 출처 링크 없음)에 대한 개선안 일부입니다.

| 항목 | 내용 |
|---|---|
| 메타 설명 제안 (82자) | 아기 이유식은 일반적으로 생후 6개월 이후 시작하도록 권장됩니다. 시작 시기와 발달 신호, 첫 음식, 새 음식을 하나씩 먹이는 방법을 정리했습니다. |
| 출처 제안 | '질병관리청 국가건강정보포털'이 처음 나오는 문장에 해당 자료 링크를 걸고, 글 끝에 참고 자료 목록을 추가하세요. 정확한 URL은 출처 추가 필요입니다. |
| 검증 | 통과 (`needsReview: false`) |

전체 응답은 [examples/sample-claude-suggestion.json](examples/sample-claude-suggestion.json)에 있습니다.

실측 비용 (Sonnet 5.5, 100만 토큰당 입력 $2·출력 $10):

| 글 | 입력 토큰 | 출력 토큰 | 비용 |
|---|---|---|---|
| 이유식 | 3,408 | 1,402 | $0.021 |
| 0~2세 육아 | 3,695 | 1,520 | $0.023 |

글 하나에 약 2센트이므로, 글 17개를 모두 보내도 약 $0.37입니다.

## 실행 방법

```bash
docker compose up -d
```

1. http://localhost:5678 에 접속해 계정을 만듭니다.
2. Workflows → Import from File로 `workflows/blogstudio-geo-audit.json`을 불러옵니다.
3. Anthropic API 키를 등록합니다. Credentials → Create credential → Header Auth를 고르고, Name은 `x-api-key`, Value는 API 키로 저장합니다. 그다음 Claude 노드에서 이 자격 증명을 선택합니다.
4. Execute workflow를 누릅니다. 규칙 점검은 모든 글에 하고, Claude에는 Limit 노드가 정한 2개만 보냅니다. 전체를 보내려면 Limit 노드를 지웁니다.

## 한계

- 질문형 판정과 숫자 세기는 정규식이라 대략적입니다.
- 숫자 검사는 그 숫자가 원문에 있는지만 봅니다. 단위나 문맥이 맞는지는 보지 않습니다.
- 개선안의 질은 사람이 읽고 판단합니다. 개선안 품질을 자동으로 평가하는 장치(eval)는 아직 없습니다.
- 규칙이 n8n Code 노드 안에 있어 단위 테스트가 어렵습니다. 규칙이 늘어나면 테스트할 수 있는 별도 서비스(Spring Boot)로 분리할 계획입니다.
- 대상 사이트는 blogstudio.co.kr 하나입니다.
