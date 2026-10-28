---
name: cnki-advanced-search
description: "CNKI 해외판에서 필드·연도 조건으로 정밀 검색할 때 쓴다 — 저자·제목·학술지·주제 조합을 전문검색(专业检索) 식으로, 출판 기간 지정. 성균관대 프록시 경유. 결과 추출·다음 쪽·적재는 cnki-search와 같다."
license: "MIT (upstream README 표기 — LICENSE 파일 없음)"
argument-hint: "[조건을 자연어로 — 예: 저자 김아무개, 학술지 哲学研究, 2015～2025]"
metadata:
  author: cookjohn
  version: 1.0.0
  category: cjk-research
  upstream: https://github.com/cookjohn/cnki-skills @ 20d65f660456daf53ad0f7c74494ac3b829b925f
  local_divergence: |
    (1) 해외판은 구형 고급검색(/kns/AdvSearch, 来源类别 체크박스)을 신형 /kns8s/AdvSearch로 돌려보낸다(2026-10-22 실측).
        upstream의 select 색인·#txt_1_value1 양식이 없어 절차를 전문검색(专业检索) 식 입력으로 새로 썼다.
        CSSCI·北大核心 같은 来源类别 필터는 이 경로에 없다.
    (2) 대상·도구·숨은 문자 정리는 cnki-search와 같다(성균관대 프록시, claude-in-chrome). AUD-20261022-123013-5e888b76.
---

# CNKI 고급검색 — 전문검색 식 (해외판·SKKU 프록시)

## 환경

`cnki-search`의 《환경》과 같다. 주소만 다르다: `https://oversea-cnki-net-ssl.ca.skku.edu/kns8s/advsearch?language=chs`

## 검색식 만들기

사용자의 자연어 조건을 전문검색 식으로 옮긴다. 값은 작은따옴표로 싸고, 조건은 `AND`·`OR`·`NOT`(대문자, 앞뒤 공백)으로 잇는다.

| 필드 | 코드 | 예 |
|---|---|---|
| 주제 | `SU` | `SU='丁若镛'` |
| 篇关摘(제목·키워드·초록) | `TKA` | `TKA='实学'` |
| 제목 | `TI` | `TI='茶山'` |
| 키워드 | `KY` | `KY='心性论'` |
| 초록 | `AB` | `AB='朝鲜'` |
| 저자 | `AU` | `AU='刘敏'` |
| 제1저자 | `FI` | |
| 소속 | `AF` | `AF='延边大学'` |
| 문헌 출처(학술지·학위 수여 기관) | `LY` | `LY='哲学研究'` |
| 기금 | `FU` | |

- 한국 인명·서명은 간체로 넣는다(丁若鏞 → 丁若镛). 번체·간체 둘 다 걸려면 `SU='丁若镛' OR SU='丁若鏞'`.
- 실측: `SU='丁若镛' AND LY='哲学研究'` → 1건(《哲学研究》 2026년 9기), `SU='丁若镛'` + 2020-01-01～2023-12-31 → 28건(期刊·辑刊·博士·硕士·图书 섞임).
- 식이 틀리면 결과가 0건이거나 오류 문구가 뜬다. 0건이면 식을 사용자에게 보이고 조건을 줄여 다시 돈다.

## 절차

### 1. 이동

`navigate` → 위 주소.

### 2. 식·기간 입력과 검색 (한 호출)

```javascript
await (async () => {
  const expr = "SU='丁若镛' AND LY='哲学研究'";   // 검색식
  const from = "", to = "";                      // 출판 기간 'YYYY-MM-DD', 없으면 ""
  const cap = document.querySelector('#tcaptcha_transform_dy');
  if (cap && cap.getBoundingClientRect().top >= 0) return 'captcha';
  document.querySelector('li[name=majorSearch]')?.click();          // 「专业检索」 탭
  await new Promise(r => setTimeout(r, 1200));
  const ta = document.querySelector('textarea.textarea-major');
  if (!ta) return 'no-form';
  ta.value = expr; ta.dispatchEvent(new Event('input', {bubbles: true}));
  for (const [id, v] of [['#datebox0', from], ['#datebox1', to]]) {
    const d = document.querySelector(id);
    if (d && v) { d.value = v; d.dispatchEvent(new Event('input', {bubbles: true})); d.dispatchEvent(new Event('change', {bubbles: true})); }
  }
  setTimeout(() => document.querySelector('input.btn-search').click(), 50);
  return 'submitted';
})()
```

### 3. 결과 추출·다음 쪽·보고

결과 표는 기본 검색과 같은 모양이다. `cnki-search`의 3～5단계 스크립트를 그대로 쓴다(주소는 `/kns8s/advsearch`에 머문다). 보고 첫 줄에 쓴 검색식과 기간을 적는다.

## 확인한 선택자 (2026-10-22, 해외판 신형 고급검색)

| 요소 | 선택자 |
|---|---|
| 전문검색 탭 | `li[name=majorSearch]` |
| 검색식 입력 | `textarea.textarea-major` |
| 출판 기간 | `#datebox0`(시작)·`#datebox1`(끝) |
| 검색 단추 | `input.btn-search` |
| 결과 | `cnki-search` 표와 같음 |
