---
name: cnki-journal-toc
description: "CNKI 해외판에서 한 학술지의 호(期)별 목차를 훑을 때 쓴다 — 연도·호를 골라 수록 논문 제목·저자·쪽을 뽑는다. 성균관대 프록시 경유. 학술지 찾기는 cnki-advanced-search의 LY 검색."
license: "MIT (upstream README 표기 — LICENSE 파일 없음)"
argument-hint: "[학술지명] [연도] [호 — 예: No.01]"
metadata:
  author: cookjohn
  version: 1.0.0
  category: cjk-research
  upstream: https://github.com/cookjohn/cnki-skills @ 20d65f660456daf53ad0f7c74494ac3b829b925f
  local_divergence: |
    (1) 대상·도구·숨은 문자 정리는 cnki-search와 같다(성균관대 프록시 해외판, claude-in-chrome).
    (2) 학술지 페이지로 가는 길을 cnki-journal-search 대신 「LY 검색 → 결과의 출처 링크」로 바꿨다 — journal-search는 들이지 않았고,
        해외판 화면의 학술지 내비 링크(oversea.cnki.net/knavi/…)는 프록시로 바뀌지 않은 채 나온다(2026-10-22 실측).
    (3) 원판 목차 PDF(原版目录) 내려받기는 실측하지 않아 절차에서 뺐다. AUD-20261022-123013-5e888b76.
---

# CNKI 학술지 호별 목차 (해외판·SKKU 프록시)

## 환경

`cnki-search`의 《환경》과 같다. 학술지 페이지 주소는 `https://oversea-cnki-net-ssl.ca.skku.edu/knavi/detail?p=…`이다.

## 절차

### 1. 학술지 페이지로 가기

1. `cnki-advanced-search`로 `LY='학술지명'` 검색(간체로. 예: `LY='哲学研究'`).
2. 결과 첫 행의 출처 링크 주소를 얻어 `navigate`한다(클릭은 새 탭을 연다):

```javascript
document.querySelector('.result-table-list tbody td.source a')?.href || 'no-source-link'
```

화면 머리의 「期刊」 같은 내비 링크는 프록시 밖 주소(`oversea.cnki.net`)라 쓰지 않는다.

### 2. 연도·호 고르고 목차 추출 (한 호출)

```javascript
await (async () => {
  const year = "2025", issue = "No.01";
  const S = x => (x || '').replace(/[\u200B-\u200F\u202A-\u202E\u2060-\u206F\uFEFF]/g, '').trim();
  const dls = [...document.querySelectorAll('#yearissue0 dl.s-dataList')];
  const dl = dls.find(d => S(d.querySelector('dt')?.innerText) === year);
  const target = dl && [...dl.querySelectorAll('dd a')].find(a => S(a.innerText) === issue);
  if (!target) return { error: 'issue_not_found',
    available: dls.slice(0, 5).map(d => ({ year: S(d.querySelector('dt')?.innerText), issues: [...d.querySelectorAll('dd a')].map(a => S(a.innerText)) })) };
  target.click();
  await new Promise((r, j) => { let n = 0; const c = () => document.querySelectorAll('#CataLogContent dd.row').length ? r() : (++n > 30 ? j('timeout') : setTimeout(c, 500)); setTimeout(c, 1000); });
  return {
    journal: S(document.title.replace(/\s*-\s*中国知网.*$/, '')),
    issueLabel: S(document.querySelector('span.date-list')?.innerText),
    papers: [...document.querySelectorAll('#CataLogContent dd.row')].map((dd, i) => ({
      no: i + 1,
      title: S(dd.querySelector('span.name a')?.innerText),
      href: dd.querySelector('span.name a')?.href || '',
      authors: S(dd.querySelector('span.author')?.innerText).replace(/;$/, ''),
      pages: S(dd.querySelector('span.company')?.innerText)
    }))
  };
})()
```

`issue_not_found`면 `available`(최근 5년의 호 목록)을 보이고 다시 고르게 한다.

### 3. 보고

```
## {journal} — {issueLabel}  (N편)
1. {title}  [{pages}]  {authors}
```

한 편을 자세히 보려면 그 `href`로 `cnki-paper-detail`, 서지를 모으려면 그 편들의 상세 페이지에서 `cnki-export`.

## 확인한 선택자 (2026-10-22, 해외판 — 《哲学研究》 2025년 01기 15편 추출)

| 요소 | 선택자 |
|---|---|
| 연도 묶음 | `#yearissue0 dl.s-dataList`, 연도 `dt`, 호 `dd a`(「No.01」) |
| 호 표시 | `span.date-list`(「2025年01期」) |
| 목차 행 | `#CataLogContent dd.row` |
| 제목·저자·쪽 | `span.name a`·`span.author`·`span.company` |
