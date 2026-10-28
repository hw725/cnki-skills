---
name: cnki-search
description: "CNKI(中国知网) 해외판에서 키워드로 논문·학위논문을 찾을 때 쓴다 — 성균관대 프록시 경유, 결과 목록 추출·다음 쪽, vault-pipeline 목록 JSON 저장. 필드·연도 조건은 cnki-advanced-search."
license: "MIT (upstream README 표기 — LICENSE 파일 없음)"
argument-hint: "[검색어]"
metadata:
  author: cookjohn
  version: 1.0.0
  category: cjk-research
  upstream: https://github.com/cookjohn/cnki-skills @ 20d65f660456daf53ad0f7c74494ac3b829b925f
  local_divergence: |
    (1) 대상: 본토판 kns.cnki.net → 성균관대 프록시 경유 해외판 oversea-cnki-net-ssl.ca.skku.edu(2026-10-22 실측).
    (2) 도구: chrome-devtools MCP → claude-in-chrome(navigate·javascript_tool). 검색 단추가 페이지를 바꿀 수 있어
        「입력·클릭」과 「추출」을 두 호출로 나눴다.
    (3) 해외판 결과 제목에 숨은 서식 문자(U+200E·U+2060～206F)가 끼어 있어 추출 때 지운다.
    (4) 출력에 vault-pipeline 목록 JSON 형식을 더했다. AUD-20261022-123013-5e888b76.
---

# CNKI 기본 검색 (해외판·SKKU 프록시)

## 환경 (먼저 확인)

- 검색 주소: `https://oversea-cnki-net-ssl.ca.skku.edu/kns8s/search?language=chs`
- 인증 확인: `https://chn-oversea-cnki-net-ssl.ca.skku.edu/index/` 오른쪽 위에 「欢迎来自成均馆大学」가 보이면 기관 인증 상태다. 성균관대 로그인 화면이 나오면 **사용자가 직접 로그인**한다(비밀번호를 대신 넣지 않는다).
- 구독 범위: CAJ(학술지·집간)·CDMD(석박사 학위논문, 문사철 2000～). 인사캠 구독분이다.
- 도구: `mcp__claude-in-chrome__navigate`, `mcp__claude-in-chrome__javascript_tool`(REPL — 맨 위 `await` 가능, 마지막 식이 반환값. `return`은 `await (async () => { … })()` 안에서만).
- 텍스트 정리: 해외판은 제목·저자에 숨은 서식 문자를 끼운다. 추출한 문자열은 모두 `S()`(아래)로 거른다.

## 절차

### 1. 이동

`navigate` → 검색 주소.

### 2. 입력·검색 (한 호출 — 반환을 기다리지 않는다)

```javascript
(() => {
  const q = "검색어";
  const cap = document.querySelector('#tcaptcha_transform_dy');
  if (cap && cap.getBoundingClientRect().top >= 0) return 'captcha';
  const input = document.querySelector('input.search-input');
  if (!input) return 'no-input';   // 로그인 화면이면 사용자에게 로그인을 부탁한다
  input.value = q; input.dispatchEvent(new Event('input', {bubbles: true}));
  setTimeout(() => document.querySelector('input.search-btn').click(), 50);
  return 'submitted';
})()
```

`captcha`면 사용자에게 Chrome에서 슬라이드 인증을 끝내 달라고 하고 기다린다.

### 3. 결과 추출 (다음 호출)

```javascript
await (async () => {
  const S = x => (x || '').replace(/[\u200B-\u200F\u202A-\u202E\u2060-\u206F\uFEFF]/g, '').trim();
  await new Promise((r, j) => { let n = 0; const c = () => document.querySelectorAll('.result-table-list tbody tr').length ? r() : (++n > 30 ? j('timeout') : setTimeout(c, 500)); c(); });
  const rows = [...document.querySelectorAll('.result-table-list tbody tr')];
  return {
    total: S(document.querySelector('.pagerTitleCell')?.innerText).match(/[\d,]+/)?.[0] || '0',
    page: S(document.querySelector('.countPageMark')?.innerText) || '1/1',
    results: rows.map((row, i) => ({
      n: i + 1,
      title: S(row.querySelector('td.name a.fz14')?.innerText),
      href: row.querySelector('td.name a.fz14')?.href || '',
      exportId: row.querySelector('input.cbItem')?.value || '',
      authors: [...row.querySelectorAll('td.author a')].map(a => S(a.innerText)).filter(Boolean),
      source: S(row.querySelector('td.source a')?.innerText || row.querySelector('td.source')?.innerText),
      date: S(row.querySelector('td.date')?.innerText),
      db: S(row.querySelector('td.data')?.innerText)   // 期刊·辑刊·博士·硕士·图书
    }))
  };
})()
```

### 4. 다음 쪽

```javascript
await (async () => {
  const first = document.querySelector('.result-table-list tbody td.name a.fz14')?.innerText;
  document.querySelector('#PageNext')?.click();
  await new Promise((r, j) => { let n = 0; const c = () => { const t = document.querySelector('.result-table-list tbody td.name a.fz14')?.innerText; t && t !== first ? r() : (++n > 30 ? j('timeout') : setTimeout(c, 500)); }; setTimeout(c, 800); });
  return document.querySelector('.countPageMark')?.innerText;
})()
```

그다음 3단계를 다시 돌린다. `#PageNext`가 없으면 마지막 쪽이다.

### 5. 보고

번호 목록으로 제목·저자·출처·날짜·구분(db)을 보인다. 논문 한 편을 열 때는 결과의 `href`로 `navigate`한다(링크 클릭은 새 탭을 연다).

## 적재로 넘기기

사용자가 결과를 모아 두려 하면 `cnki-export`로 넘긴다. 결과 페이지에서 바로 서지(ENDNOTE)를 받아 vault-pipeline 목록 JSON·RIS·GB/T 7714를 만든다. 서지는 CJK RefManager가, 원문·구조분석은 hwobsidian이 맡는다(hwobsidian `AGENTS.md` 《RefManager 연동》).

## 확인한 선택자 (2026-10-22, 해외판)

| 요소 | 선택자 |
|---|---|
| 검색어 입력 | `input.search-input` (`#txt_SearchText`) |
| 검색 단추 | `input.search-btn` |
| 결과 수 | `.pagerTitleCell` 「共找到 N 条结果」 |
| 쪽 표시 | `.countPageMark` 「1/8」 |
| 결과 행 | `.result-table-list tbody tr` |
| 제목 링크 | `td.name a.fz14` → `/kcms2/article/abstract` |
| 내보내기 ID | `input.cbItem` 값 |
| 구분 | `td.data` |
| 다음 쪽 | `#PageNext` |
| 캡차 | `#tcaptcha_transform_dy` 의 `top >= 0` 일 때만(평소 −1000000에 숨어 있다) |
