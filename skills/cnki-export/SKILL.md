---
name: cnki-export
description: "CNKI 해외판 검색 결과·상세 페이지의 서지를 내보낼 때 쓴다 — ENDNOTE를 받아 vault-pipeline 목록 JSON(CJK RefManager 등록용)·RIS·GB/T 7714로 저장. 성균관대 프록시 경유."
license: "MIT (upstream README 표기 — LICENSE 파일 없음)"
argument-hint: "[list|ris|gb] [결과 번호들, 비우면 이 쪽 전부]"
metadata:
  author: cookjohn
  version: 1.0.0
  category: cjk-research
  upstream: https://github.com/cookjohn/cnki-skills @ 20d65f660456daf53ad0f7c74494ac3b829b925f
  local_divergence: |
    (1) Zotero 연동 스크립트(상류 push_to_zotero.py)를 들이지 않았다 — 이 PC에 Zotero가 없고(2026-10-22 확인),
        그 스크립트는 세션 쿠키를 받아 PDF를 따로 내려받는다. 서지 정본은 CJK RefManager다.
    (2) 내보내기 API: 절대 주소 https://kns.cnki.net/dm8/API/GetExport 는 프록시 오리진에서 막힌다(Failed to fetch).
        상대 경로 /dm8/API/GetExport 와 uniplatform=OVERSEA 로 바꿨다(NZKPT 는 code −1). displaymode 의 elearning 은 소문자여야 한다.
    (3) 출력을 hwobsidian 적재 흐름에 맞췄다 — vault-pipeline batch 목록 JSON(language=chinese 명시)·RIS·GB/T 7714.
    (4) 대상·도구·숨은 문자 정리는 cnki-search와 같다. AUD-20261022-123013-5e888b76.
---

# CNKI 서지 내보내기 (해외판·SKKU 프록시)

## 환경

`cnki-search`의 《환경》과 같다(검색 결과는 `https://oversea-cnki-net-ssl.ca.skku.edu/kns8s/…`, 상세는 `…/kcms2/article/abstract`). API는 그 페이지 안에서 상대 경로로 부른다.

## 어디로 가나

이 PC의 논문 적재는 둘로 갈린다(hwobsidian `AGENTS.md` 《RefManager 연동》).

- **서지** → CJK RefManager. 정본 경로는 cjk-refmanager `vault-pipeline`의 `batch new`·`batch plan`이고, 이 스킬의 **목록 JSON**이 그 입력이다.
- **원문·구조분석** → hwobsidian. PDF는 `cnki-download`로 받아 `batch match`·`extract`·`write`·`check`·`load`로 넘긴다.

그래서 기본 출력은 목록 JSON이다. RIS는 다른 서지 도구용, GB/T 7714는 인용문용이다.

## 절차

### 1. ENDNOTE 받기 (한 호출)

검색 결과 페이지(`cnki-search`·`cnki-advanced-search` 뒤)에서는 결과 행의 `input.cbItem`을, 상세 페이지에서는 `#export-id`를 쓴다. `pick`에 1부터 센 결과 번호를 넣으면 그것만, 비우면 이 쪽 전부.

```javascript
await (async () => {
  const pick = [];   // 예: [1, 3, 5]
  const S = x => (x || '').replace(/[\u200B-\u200F\u202A-\u202E\u2060-\u206F\uFEFF]/g, '').trim();
  const dec = s => S((s || '').replace(/<br>/g, '\n').replace(/&lt;/g, '<').replace(/&gt;/g, '>').replace(/&quot;/g, '"').replace(/&amp;/g, '&'));
  const detailId = document.querySelector('#export-id')?.value;
  const ids = detailId ? [detailId]
    : [...document.querySelectorAll('.result-table-list tbody input.cbItem')].map(c => c.value)
        .filter((_, i) => !pick.length || pick.includes(i + 1));
  if (!ids.length) return { error: 'no-records' };
  const out = [];
  for (const id of ids) {
    const r = await fetch('/dm8/API/GetExport', { method: 'POST', credentials: 'include',
      headers: { 'Content-Type': 'application/x-www-form-urlencoded' },
      body: new URLSearchParams({ filename: id, displaymode: 'GBTREFER,elearning,EndNote', uniplatform: 'OVERSEA', language: 'CHS' }) });
    const j = await r.json();
    if (j.code !== 1) { out.push({ error: j.message || j.code }); continue; }
    const m = k => dec(j.data.find(d => d.mode === k)?.value?.[0]);
    out.push({ endnote: m('ENDNOTE'), gbt7714: m('GBTREFER').replace(/^\[\d+\]/, '') });
  }
  return out;
})()
```

### 2. 변환 (응답을 읽고 직접)

ENDNOTE 한 건은 `%태그 값` 줄의 묶음이다. 같은 태그가 여러 번 나오면(저자 `%A`) 배열로 모은다.

**목록 JSON 항목** (vault-pipeline `batch new --list`):

```json
{ "title": "%T", "authors": ["%A", "…"], "year": "%D(정수)",
  "type": "journal | thesis",   // %0 이 Thesis 면 thesis, 그 밖은 journal
  "language": "chinese",        // 반드시 명시 — 빼면 CJK 제목이라 korean 으로 잡힌다
  "venue": "%J (학위논문은 %I)", "url": "%U", "why": "사용자가 말한 수집 이유", "priority": 1 }
```

`url`은 `%U`(`link.oversea.cnki.net/doi/…` 같은 공개 주소)를 쓴다. 프록시 주소(`*.ca.skku.edu`)는 세션에 묶여 나중에 열리지 않는다.

**RIS** 대응: `%0`→`TY`(Journal Article→JOUR, Thesis→THES), `%A`→`AU`, `%T`→`TI`, `%J`→`JO`, `%D`→`PY`, `%N`→`IS`, `%P`→`SP`·`EP`(하이픈으로 나눔), `%@`→`SN`, `%K`→`KW`(세미콜론마다 한 줄), `%X`→`AB`, `%U`→`UR`, `%R`→`DO`, `%I`→`PB`, 끝에 `ER  -`.

### 3. 저장

Write 도구로 UTF-8(BOM 없음)으로 쓴다. 경로는 사용자에게 묻되, 기본은 이번 배치의 작업 폴더다(예: `…\논문\<주제>\cnki-<날짜>.json`·`.ris`). 같은 이름 파일이 있으면 덮어쓰지 않고 묻는다.

### 4. 넘기기

목록 JSON을 만들었으면 다음 명령을 사용자에게 보인다. 실행은 cjk-refmanager `vault-pipeline` 폴더에서 하고, RefManager 쓰기이므로 사용자의 확인을 받은 뒤 돌린다.

```powershell
node cli.mjs batch new  --vault <hwobsidian 경로> --id <배치 id> --topic <주제> --list <목록 JSON>
node cli.mjs batch plan --vault <hwobsidian 경로> --id <배치 id>
```

## 확인한 사실 (2026-10-22, 해외판)

| 항목 | 값 |
|---|---|
| API | `POST /dm8/API/GetExport` (상대 경로, `credentials: 'include'`) |
| 필수 인자 | `filename`=내보내기 ID, `displaymode`=`GBTREFER,elearning,EndNote`, `uniplatform`=`OVERSEA`, `language`=`CHS` |
| ID 출처 | 결과 페이지 `input.cbItem` 값 = 상세 페이지 `#export-id` |
| ENDNOTE 태그 | 학술지 `%0 %A %T %J %D %N %K %X %P %@ %L %U %R %W` / 학위논문 `%0 %A %T %Y %I %9 %D %K %X %U %R %W` |
| 응답 | `code: 1`이 성공. 실패는 `code: -1`, 「非常抱歉，请稍后重试」 |
