---
name: "a4-print-infographic"
description: "A4 세로 인쇄물(설문지·안내문·인포그래픽)을 Claude Design 캔버스에 브랜드 디자인 시스템으로 만들 때 사용. 'A4 인포그래픽', '설문지 만들어줘', '안내문 만들어줘' 요청 시. 인쇄·PDF에서 아래가 잘리지 않도록 자동 맞춤과 높이 측정을 포함한다."
---

# A4 세로 인쇄물 / 인포그래픽 제작

결과물은 **Design 캔버스(Artifact type) + 브랜드 디자인 시스템**으로 만든 A4 세로 한 장 이상, 인쇄·PDF 다운로드용이다.

## 브랜드 설정 (처음 한 번 채우기)

아래 값을 자기 브랜드에 맞게 바꿔 쓴다. 값이 비어 있으면 사용자에게 묻거나 기본값을 쓴다.

| 항목 | 설명 | 기본값 |
|---|---|---|
| `BRAND_NAME` | 기관·브랜드 이름 (하단 표기) | (사용자에게 확인) |
| `CONTACT` | 하단 문의 정보 (전화 등) | (생략 가능) |
| `DESIGN_SYSTEM_URL` | Claude 디자인 시스템 artifact URL (README.md, tokens.json 포함) | 없으면 아래 기본 팔레트 사용 |
| `LOGO_ASSET` | 로고 SVG 에셋 ID 또는 파일 | 없으면 텍스트 워드마크 |
| `BODY_FONT` | 본문 서체 (woff2, 재배포 라이선스 확인) | 'Nunito', sans-serif |
| `ACCENT` | 강조색 한 가지 | #9c4a5c |

기본 팔레트: 배경 #ffffff, 본문 #2f3a30, 카드 #fffefb + 1.5px #ece5d6 테두리, 안내 띠 #f1f5ec, 기록란 #f6f2e7, 주요색 #4a6f50, 강조 `ACCENT`는 한 군데만.

## 0. 준비 (매번)

1. Artifact `list` scope types → Design 타입 `type_url` 확인. 새 캔버스: `type_url` + `title`(한국어 이름) + `auto_open: "after_first_write"`.
2. `DESIGN_SYSTEM_URL`이 있으면 `project/README.md`, `project/tokens.json`을 읽는다. 목소리·색·서체·기관 정보는 README를 그대로 따른다.
3. 에셋을 새 캔버스로 가져오기
   - 서체: 디자인 시스템의 woff2를 `read`(path)로 받아 새 캔버스에 `asset:true`로 업로드 → `/_blob/<id>`.
   - 로고: `LOGO_ASSET`을 `from_url` + `asset_ids`로 복사 → 새 `/_blob/<id>`.
4. 디자인 시스템 설치: `files`에 `"project/ds/<brand>/tokens.json": {"artifact": "<DS url>", "path": "project/tokens.json"}`, canvas.json `designSystems`에 레코드 추가.

## 1. 페이지 규격 (고정)

- A4 세로 = **794×1123px**. canvas.json 각 board: `w:794, h:1123, paper:"a4"`, 가로 간격 80px(x = 0, 874, 1748…). 사용자가 틀을 늘려 놓았으면 다시 794×1123으로 되돌린다.
- 페이지가 여러 장이면 장마다 별도 artboard(Main.dc.html, P2.dc.html, P3.dc.html…). 각 장에 머리말(로고 + 우측 칩 "○○ · 1")과 하단(감사 문구 + `BRAND_NAME` · `CONTACT`)을 그린다.
- 배경은 인쇄용 흰색, 색은 브랜드 설정의 팔레트를 따른다.

## 2. 잘림 방지 (가장 중요)

원인: 일부 기기(예: iPad 큰 글자 설정, 기기 글꼴)는 글자를 25~35% 크게 그린다. 또 한자용 웹폰트가 늦게 로드되며 줄이 늘어난다. 내 화면에서 딱 맞으면 사용자 화면·다운로드 PDF에서 아래가 잘린다.

반드시 지킬 것:
1. **여유 높이**: 일반 크기 렌더 기준 콘텐츠 높이 ≤ 약 1000px (A4의 ~89%). 1123에 딱 맞추지 않는다.
2. **간격 기본값**: 루트 padding `30px 46px 24px`, 섹션 gap 11~14px, 로고 높이 34px, h1 26px, 본문 14~15px, 보조 12~13px(인쇄 최소 12px).
3. **한글 줄바꿈**: 루트에 `word-break: keep-all`.
4. **라벨 칩(동그란 pill)**: `display:inline-flex; align-items:center; justify-content:center; text-align:center; white-space:nowrap; line-height:1.2; padding:4px 10px`. 두 줄로 꺾여 왼쪽으로 쏠리는 문제 방지. 칩이 들어가는 칸은 128px 이상.
5. **글자 자동 확대 차단**: helmet style에 `html,body{-webkit-text-size-adjust:100%;text-size-adjust:100%}`.
6. **A4 자동 맞춤 래퍼**를 모든 페이지에 넣는다 (아래 코드). 넘칠 때만 비율대로 축소, 정상 화면·PDF에서는 원래 크기.
7. 긴 문구는 미리 줄인다: 팁·설명은 한두 줄, 중복 정보는 라벨에 합친다("TYPE 01 · 문항 ①").

### 자동 맞춤 래퍼 (구조)

```html
<x-dc>
<helmet>... @font-face(BODY_FONT, unicode-range:U+0080-10FFFF) ... </helmet>
<div style="width: 794px; height: 1123px; overflow: hidden; background: #ffffff; position: relative">
<div data-a4-fit="1" style="width: {{fw}}px; height: {{fh}}px; transform: scale({{fs}}); transform-origin: 0 0; word-break: keep-all; box-sizing: border-box; padding: 30px 46px 24px; background: #ffffff; color: #2f3a30; font-family: 'BODY_FONT', 'Nunito', sans-serif; display: flex; flex-direction: column; gap: 12px; overflow: hidden">
  ... 내용 ... <footer style="margin-top: auto; ...">...</footer>
</div>
</div>
</x-dc>
```

### 자동 맞춤 스크립트 (그대로 사용)

```js
class Component extends DCLogic {
constructor(props) { super(props); this.state = { s: 1 }; this.fit = this.fit.bind(this); }
componentDidMount() {
this.fit();
const el = document.querySelector('[data-a4-fit]');
if (el && window.ResizeObserver) { this.ro = new ResizeObserver(() => this.fit()); this.ro.observe(el); Array.prototype.forEach.call(el.querySelectorAll('section, article, header, footer, p, div'), (c) => this.ro.observe(c)); }
if (document.fonts && document.fonts.ready) { document.fonts.ready.then(this.fit); }
this.ts = [300, 1000, 2500, 5000].map((d) => setTimeout(this.fit, d));
}
componentDidUpdate() { this.fit(); }
componentWillUnmount() { if (this.ro) this.ro.disconnect(); (this.ts || []).forEach(clearTimeout); }
fit() {
const el = document.querySelector('[data-a4-fit]');
if (!el || this.busy) return;
this.busy = true;
const w0 = el.style.width, h0 = el.style.height;
const ok = (s) => { el.style.width = (794 / s) + 'px'; el.style.height = (1123 / s) + 'px'; return el.scrollHeight <= el.clientHeight + 1; };
let best = 1;
if (!ok(1)) { let lo = 0.55, hi = 1; for (let i = 0; i < 14; i++) { const mid = (lo + hi) / 2; if (ok(mid)) lo = mid; else hi = mid; } best = lo; }
el.style.width = w0; el.style.height = h0;
this.busy = false;
if (Math.abs(best - this.state.s) > 0.003) this.setState({ s: best });
}
renderVals() {
const s = this.state.s;
return { accent: this.props.accent ?? '#4a6f50', fs: s.toFixed(4), fw: (794 / s).toFixed(1), fh: (1123 / s).toFixed(1) };
}
}
```
`data-props`에는 `accent`(색 tweak)와 `$preview: {width:794,height:1123}`.

## 3. 게시 전 높이 측정 (필수)

로컬 Playwright(Chromium)로 각 artboard를 측정한다. 방법: `<x-dc>` 안 마크업만 꺼내 `{{accent}}`·`{{fw}}`(794)·`{{fh}}`(1123)·`{{fs}}`(1)을 치환하고, 본문 서체 woff2를 로컬 @font-face로 연결, 로고 blob은 로컬 svg로 바꿔 렌더 → 루트 height를 auto로 풀고 `scrollHeight` 측정. 추가로 모든 `font-size: Npx`를 ×1.3 해서 한 번 더 측정.

통과 기준:
- 1.0배: ≤ 약 1000px
- 1.3배: 자동 맞춤 시뮬레이션 결과 축소율 ≥ 0.85, 하단 문구까지 보임
- 스크린샷을 직접 눈으로 확인(줄바꿈 어색함, 칩 정렬, 한자 줄넘김)

## 4. 내용 규칙 (기본값, 디자인 시스템 README가 있으면 그쪽 우선)

- 합니다체, 독자를 존중하는 호칭. 효과·결과를 보장하는 표현 금지("도움이 될 수 있습니다"). 이모지 금지, 체크는 ✓만.
- 한자는 `font-family: 'Noto Serif KR', serif`로 감싼다. 숫자·영문은 Nunito. Google Fonts `css2` 링크로 Nunito + Noto Serif KR 로드.
- 연락처는 `white-space: nowrap`. 운영시간 등 기관 정보는 디자인 시스템 README 표 그대로.
- 설문지: 원본 문항 문구 보존, 기간 표현 통일(예: ISI는 최근 2주), 점수 해석은 응답자용 안내가 아닌 "담당자 기록란"에, 자가진단 고지 문구 포함("진단을 대신하지 않으며 정확한 판단은 전문가 상담").
- 안내문 흐름(예): 증상/유형 → 원인 → 관리·치료 → 생활 관리 → "이런 경우 먼저 말씀해 주세요"(강조색 박스). 전문 내용은 실제 담당자 판단과 다를 수 있다고 답변에서 알린다.
- 표시용 원(○)은 `border:1.5px solid #5b6157` 18px span, 문항 옵션 열은 68px 고정 grid.

## 5. 게시와 수정

- 파일은 scratchpad의 한 폴더 `<root>/project/...`에 Write → Artifact publish(`url`, `root`, `file_path`, `files`). 레이아웃(보드 추가·크기) 바뀔 때만 canvas.json 재전송, 기존 키(`attachments` 등) 유지.
- 수정 요청 시 먼저 게시본을 `read`해 사용자가 손댄 부분(틀 크기 등)을 확인.
- 사용자가 잘림을 다시 보고하면: 캡처에서 어느 페이지·어느 요소인지 확인 → 그 페이지 문구 축약·간격 축소 → 3단계 측정 재실시 → 그래도 안 되면 두 장으로 나누기 제안.
- 답변은 짧게: 무엇을 바꿨는지와 확인 부탁할 점만. 실제 사용자 화면에서 렌더를 확인 못 했으면 그렇게 말한다.
