---
name: cnki-download
description: "CNKI 해외판 논문·학위논문 원문 PDF(없으면 CAJ)를 내려받을 때 쓴다 — 성균관대 프록시 기관 인증, 한 편씩 사용자 확인 뒤 클릭, 받은 파일을 각자 지정한 받는 자리(%PAPERS_ROOT%/_global_inbox)로 옮겨 vault-pipeline batch match에 넘긴다."
license: "MIT (upstream README 표기 — LICENSE 파일 없음)"
argument-hint: "[상세 페이지 URL, 비우면 지금 페이지] [pdf|caj]"
metadata:
  author: cookjohn
  version: 1.0.1
  category: cjk-research
  upstream: https://github.com/cookjohn/cnki-skills @ 20d65f660456daf53ad0f7c74494ac3b829b925f
  local_divergence: |
    (1) 대상·도구·숨은 문자 정리는 cnki-search와 같다(성균관대 프록시 해외판, claude-in-chrome).
        다운로드 링크는 프록시의 다른 호스트(o-oversea-cnki-net-ssl.ca.skku.edu)로 간다(2026-10-22 실측).
    (2) 다운로드마다 제목·형식을 사용자에게 밝히고 확인을 받은 뒤 클릭한다 — upstream은 곧바로 클릭한다.
    (3) 받은 뒤의 이름 붙이기·적재 폴더 이동을 더했다(CJK RefManager·hwobsidian 적재). AUD-20261022-123013-5e888b76.
    (4) 옮길 곳은 스킬에 경로를 적지 않고 환경변수 PAPERS_ROOT 아래 _global_inbox 로 한다 — 각자 로컬에서 지정(2026-10-22).
---

# CNKI 원문 내려받기 (해외판·SKKU 프록시)

## 환경

`cnki-search`의 《환경》과 같다. 원문은 기관 인증이 있어야 받는다. 상세 페이지에 `#pdfDown`·`#cajDown`이 있고 「未登录」 표시(`.downloadlink.icon-notlogged`)가 없으면 받을 수 있다.

- Chrome은 같은 출처의 두 번째 다운로드부터 조용히 막는다. `chrome://settings/content/automaticDownloads` 허용 목록에 `[*.]ca.skku.edu`가 있어야 한다(`paper-harvest`와 같은 조건).
- 형식은 **PDF가 기본**이다. CAJ는 CAJViewer가 있어야 열리고, 적재 파이프라인의 텍스트 추출이 읽지 못한다.

## 절차

### 1. 이동과 상태 확인 (한 호출 — 클릭하지 않는다)

URL이 있으면 먼저 `navigate`. 그다음:

```javascript
await (async () => {
  const S = x => (x || '').replace(/[\u200B-\u200F\u202A-\u202E\u2060-\u206F\uFEFF]/g, '').trim();
  const q = s => document.querySelector(s);
  await new Promise((r, j) => { let n = 0; const c = () => q('.brief h1') ? r() : (++n > 30 ? j('timeout') : setTimeout(c, 500)); c(); });
  const cap = q('#tcaptcha_transform_dy');
  if (cap && cap.getBoundingClientRect().top >= 0) return { error: 'captcha' };
  if (q('.downloadlink.icon-notlogged') || q('[class*="notlogged"]')) return { error: 'not_logged_in' };
  const host = el => { try { return new URL(el.href).host; } catch (e) { return null; } };
  return {
    title: S(q('.brief h1')?.innerText).replace(/\s*(附视频|网络首发)\s*$/, ''),
    authors: [...(q('.brief h3.author')?.querySelectorAll('a') || [])].map(a => S(a.innerText).replace(/\d+$/, '')),
    pdf: !!q('#pdfDown'), caj: !!q('#cajDown'), pdfHost: q('#pdfDown') ? host(q('#pdfDown')) : null
  };
})()
```

- `not_logged_in`: 기관 인증이 풀렸다. `chn-oversea-cnki-net-ssl.ca.skku.edu/index/`를 열어 사용자에게 로그인을 부탁한다.
- `captcha`: 사용자가 Chrome에서 슬라이드 인증을 끝낼 때까지 기다린다.

### 2. 확인 받기

사용자에게 한 줄로 묻는다: 《{title} ({authors}) — PDF를 CNKI 해외판(성균관대 프록시)에서 받을까요?》 크기는 받기 전에 알 수 없다고 밝힌다. 여러 편이면 목록을 한 번에 보이고 한 번에 승인받는다. 승인 없이는 클릭하지 않는다.

### 3. 클릭 (승인 뒤 — 스크립트가 아니라 실제 클릭)

`#pdfDown`은 `target=_blank` 링크라 `javascript_tool`의 `a.click()`은 팝업 차단에 막혀 아무 일도 일어나지 않는다(2026-10-22 실측). `mcp__claude-in-chrome__computer`로 화면을 직접 누른다.

1. `find`로 내려받기 단추를 찾아 `scroll_to` 한다. 해외판 화면에서는 PDF·CAJ가 드롭다운 안에 숨어 있다 — 학술지 논문은 초록색 「下载」, 학위논문은 「整本下载」(「章节下载」는 장별이라 쓰지 않는다).
2. 그 단추를 `left_click` → 펼쳐진 「PDF」를 `left_click`.
3. 빈 탭이 잠깐 생겼다 닫히고 다운로드 폴더에 파일이 생긴다. 단추 옆 「页数」「大小」로 쪽수·크기를 미리 알 수 있으니 2단계 확인 때 함께 밝힌다.

CAJ만 있으면 그 사실을 먼저 말하고 따로 승인받는다.

### 4. 받은 파일 정리

1. Chrome 다운로드 폴더에서 방금 받은 파일을 찾는다. CNKI가 붙이는 이름은 `제목_제1저자.pdf`이고 긴 제목은 가운데가 `...`로 줄어든다(예: `中日韩《孟子》学研究_王岩.pdf`). 파일 머리가 `%PDF`인지 확인한다. 못 찾으면 사용자에게 경로를 묻는다.
2. 이름을 `저자_연도_제목.pdf`로 바꿔(연도는 `cnki-export`가 받은 ENDNOTE `%D`) **받는 자리** `%PAPERS_ROOT%/_global_inbox/`로 옮긴다(삭제가 아니라 이동). 받는 자리는 스킬이 정하지 않는다 — 각자 환경변수 `PAPERS_ROOT`로 지정한다. 설정이 없으면 옮기지 않고 다운로드 폴더에 둔 채 그 사실을 알린다. 같은 이름이 있으면 덮어쓰지 않고 묻는다.
3. 받는 자리를 `--pdf-dir`로 vault-pipeline에 넘긴다 — 서지는 `cnki-export` 목록 JSON으로 `batch new`·`plan`이 먼저 등록돼 있어야 한다.

```powershell
node cli.mjs batch match   --vault <hwobsidian 경로> --id <배치 id> --pdf-dir <PDF 폴더>
node cli.mjs batch extract --vault <hwobsidian 경로> --id <배치 id>
```

이후 `write`(구조분석 작성)·`check`·`load`는 hwobsidian `AGENTS.md`와 vault-pipeline README의 절차를 따른다.

## 확인한 선택자 (2026-10-22, 해외판)

| 요소 | 선택자 | 비고 |
|---|---|---|
| PDF | `#pdfDown` | 링크 호스트 `o-oversea-cnki-net-ssl.ca.skku.edu`, `target=_blank` — 존재 확인용, 클릭은 화면에서 |
| CAJ | `#cajDown` | |
| 미인증 | `.downloadlink.icon-notlogged` | |
| 제목 | `.brief h1` | |

실측(2026-10-22): 《社会科学战线》 2019년 2기 논문 19쪽 2.9MB, 산시대학 2020년 박사논문 384쪽 4.9MB — 둘 다 「下载」·「整本下载」 → 「PDF」 실제 클릭으로 받았다.
