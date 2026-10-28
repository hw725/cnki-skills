---
name: cnki-paper-detail
description: "CNKI 해외판 논문·학위논문 상세 페이지에서 제목·저자·소속·초록·키워드·기금·분류번호·인용 관계 수를 뽑을 때 쓴다 — 성균관대 프록시 경유. 서지 파일 저장은 cnki-export."
license: "MIT (upstream README 표기 — LICENSE 파일 없음)"
argument-hint: "[상세 페이지 URL, 비우면 지금 페이지]"
metadata:
  author: cookjohn
  version: 1.0.0
  category: cjk-research
  upstream: https://github.com/cookjohn/cnki-skills @ 20d65f660456daf53ad0f7c74494ac3b829b925f
  local_divergence: |
    (1) 대상·도구·숨은 문자 정리는 cnki-search와 같다(성균관대 프록시 해외판, claude-in-chrome).
    (2) 해외판 학위논문 페이지는 소속·출처 요소(.doc-top·두 번째 h3.author)가 비어 있다(2026-10-22 실측).
        그래서 DOM 추출에 더해 내보내기 API의 ENDNOTE를 함께 받아 출처·연도·권호·쪽·ISSN·DOI·학위 정보를 채운다.
    (3) 스냅숏 대체 경로는 claude-in-chrome의 get_page_text로 바꿨다. AUD-20261022-123013-5e888b76.
---

# CNKI 논문 상세 (해외판·SKKU 프록시)

## 환경

`cnki-search`의 《환경》과 같다. 상세 페이지 주소는 `https://oversea-cnki-net-ssl.ca.skku.edu/kcms2/article/abstract?v=…`이다.

## 절차

### 1. 이동 (URL이 주어졌을 때)

`navigate` → 그 URL. 검색 결과에서 왔다면 결과의 `href`를 쓴다.

### 2. 추출 (한 호출)

```javascript
await (async () => {
  const S = x => (x || '').replace(/[\u200B-\u200F\u202A-\u202E\u2060-\u206F\uFEFF]/g, '').trim();
  const q = s => document.querySelector(s);
  await new Promise((r, j) => { let n = 0; const c = () => q('.brief h1') ? r() : (++n > 30 ? j('timeout') : setTimeout(c, 500)); c(); });
  const cap = q('#tcaptcha_transform_dy');
  if (cap && cap.getBoundingClientRect().top >= 0) return { error: 'captcha' };
  const brief = q('.brief'), h3 = brief.querySelectorAll('h3.author');
  const out = {
    title: S(brief.querySelector('h1')?.innerText).replace(/\s*(附视频|网络首发)\s*$/, ''),
    authors: h3[0] ? [...h3[0].querySelectorAll('a')].map(a => S(a.innerText).replace(/\d+$/, '')) : [],
    affiliations: h3[1] ? [...h3[1].querySelectorAll('a')].map(a => S(a.innerText)) : [],
    abstract: S(q('.abstract-text')?.innerText),
    keywords: [...document.querySelectorAll('p.keywords a')].map(a => S(a.innerText).replace(/;$/, '')),
    fund: S(q('p.funds')?.innerText),
    clc: S(q('.clc-code')?.innerText),
    dbcode: q('#paramdbcode')?.value || '',          // CJFD·CMFD(석사)·CDFD(박사) 등
    citations: Object.fromEntries([...document.querySelectorAll('ul.module-tab.tpl_lieteratures li')]
      .map(li => [li.getAttribute('data-id'), +(S(li.innerText).match(/\d+/)?.[0] || 0)])),
    url: location.href
  };
  // ENDNOTE — 출처·권호·쪽·ISSN·DOI·학위 정보의 정본
  const resp = await fetch(q('#export-url')?.value || '/dm8/API/GetExport', { method: 'POST', credentials: 'include',
    headers: { 'Content-Type': 'application/x-www-form-urlencoded' },
    body: new URLSearchParams({ filename: q('#export-id')?.value || '', displaymode: 'GBTREFER,elearning,EndNote',
      uniplatform: 'OVERSEA', language: 'CHS' }) });
  const data = await resp.json();
  if (data.code === 1) {
    const mode = m => S((data.data.find(d => d.mode === m)?.value?.[0] || '').replace(/<br>/g, '\n')
      .replace(/&lt;/g, '<').replace(/&gt;/g, '>').replace(/&amp;/g, '&'));
    out.gbt7714 = mode('GBTREFER').replace(/^\[1\]/, '');
    out.endnote = mode('ENDNOTE');
  } else out.exportError = data.message || data.msg || data.code;
  return out;
})()
```

### 3. 보고

```
## {title}
저자: {authors} / 소속: {affiliations — 비면 ENDNOTE %I(학위 수여 기관)}
출처: {ENDNOTE %J 또는 %I} {%D}년 {%N}기 {%P}쪽  ISSN {%@}  DOI {%R}
학위: {%9}  지도교수: {%Y}        ← 학위논문일 때만
초록: {abstract}
키워드: {keywords}
기금: {fund} / 분류번호: {clc}
인용 관계: 참고문헌 {references} · 피인용 {citations} …
GB/T 7714: {gbt7714}
```

ENDNOTE 태그: `%0` 유형(Journal Article·Thesis), `%A` 저자, `%T` 제목, `%J` 학술지, `%D` 연도, `%N` 기(期), `%P` 쪽, `%@` ISSN, `%K` 키워드, `%X` 초록, `%U` 링크, `%R` DOI, `%I` 기관, `%9` 학위, `%Y` 지도교수.

### 4. 대체 경로

스크립트가 `timeout`이면 `mcp__claude-in-chrome__get_page_text`로 본문을 받아 「摘要」「关键词」「基金资助」「分类号」 뒤의 글을 읽는다.

## 확인한 선택자 (2026-10-22, 해외판)

| 요소 | 선택자 | 비고 |
|---|---|---|
| 본문 머리 | `.brief`, 제목 `.brief h1` | |
| 저자·소속 | `.brief h3.author` 첫째·둘째 | 학위논문은 소속이 비어 있다 |
| 초록 | `.abstract-text` | |
| 키워드 | `p.keywords a` | |
| 기금·분류번호 | `p.funds`·`.clc-code` | 없으면 빈 값 |
| 데이터베이스 코드 | `#paramdbcode` | |
| 인용 탭 | `ul.module-tab.tpl_lieteratures li[data-id]` | references·citations·coreferences·cocitations·secondreferences·secondcitations |
| 내보내기 | `#export-url`(= `/dm8/API/GetExport`)·`#export-id` | `uniplatform=OVERSEA` |
