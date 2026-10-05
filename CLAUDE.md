# Outbound Productivity Bridge — 대시보드 저장소

이 저장소는 **배포되는 대시보드 파일만** 담고 있습니다. 엑셀 모델·구버전 파일 등 나머지 산출물은 사용자의 로컬 PC(Google Drive)에만 있습니다. (최종 갱신: 2026-10-05)

## 사용자에 대해

- 사용자(Ian)는 물류 대시보드 기획·데이터에는 능숙하지만 git/배포 같은 개발 인프라는 낯설어함.
- 기술 설명은 쉬운 말로, 절차는 클릭/명령 단위로 안내할 것. 안 되는 게 있으면 "고장이 아니라 원래 설계"임을 먼저 말할 것.
- 한국어로 대화.
- 사용자 PC 터미널은 Windows PowerShell 5.1 — `&&`가 동작하지 않는다. 사용자에게 줄 명령은 한 줄에 하나씩.

## 프로젝트 개요

센터별 출고 생산성 브릿지 대시보드. 설계 기준 생산성에서 물량·인력·운영 제약을 순차 차감해 최종 예상 생산성을 구하고, 실제 실적과 대조한다.

- 생산성 단위 표기는 **HTP** (개/인·시, pcs/man·hr).
- 계산식: `최종 예상 = 기준 × Π(1 − 제약률ᵢ)`. 각 손실 = 직전 단계 잔여 × 제약률. 적용 순서: Volume → Workforce → Operations.
- **실적 HTP** = 선택한 기간의 `Σ물량 ÷ Σ공수` (Actuals 시트 기준, 일/주/월 집계).
- **실행 갭** = 실적 − 전체 제약 적용 시 예상. 제약 ON/OFF 토글과 무관하게 고정(전체 적용 기준). 양수 = 현장이 모델보다 잘함. 실적이 기간 선택에 따라 바뀌므로 실행 갭도 기간에 따라 달라진다.
- **제약별 최우수 센터** = 그 제약의 제약률이 가장 낮은 센터 (벤치마킹 대상).
- 현재 데이터는 전부 샘플. 실데이터 교체 예정.

## 이 저장소의 파일

| 파일 | 역할 |
|---|---|
| `index.html` | 대시보드 본체 (단일 파일, 약 1,620줄) |
| `404.html` | index.html과 **완전히 동일한 사본** — index를 고치면 반드시 함께 복사할 것 |
| `data.xlsx` | 대시보드가 진입 시 자동으로 읽는 사이트 데이터. Data/Actuals/Guide 시트. **공개 배포됨** |
| `upload_sample.xlsx` | 화면의 "⬇ 샘플 파일" 다운로드 대상. 현재 data.xlsx와 내용 동일. **공개 배포됨** |
| `.assetsignore` | `.md`·`.vscode`를 공개 배포에서 제외 |
| `robots.txt` | 검색엔진 차단 |
| `_headers` | Cloudflare 헤더 설정 |
| `실행방법.md` | PC/랩탑에서 로컬 실행하는 법 (사용자용) |
| `.vscode/` | VS Code 실행 태스크·권장 확장 |

## 배포

Cloudflare Workers 정적 배포가 GitHub `main`에 연동 — **main에 push하면 약 45초 뒤 자동 재배포**. 빌드 없음.

- 라이브: https://performance-view.nglim-psa.workers.dev
- 저장소: https://github.com/Rocket-Ian-lab/Performance-view (**private**)
- **저장소는 비공개지만 사이트는 공개다.** `.assetsignore`에 없는 파일은 URL로 그대로 열린다.
  - 문서(`.md`)와 `.vscode/`는 자동으로 제외됨.
  - `data.xlsx`, `upload_sample.xlsx`는 대시보드가 써야 해서 의도적으로 공개 — 누구나 내려받을 수 있다.
  - 그 밖의 민감 파일을 올릴 때는 `.assetsignore`에 반드시 추가할 것.
- Cloudflare Access(사내 이메일 제한)는 아직 미적용. **실데이터를 data.xlsx에 넣어 올리기 전에 필수.**

## 대시보드 기능

- 단일 HTML, 외부 라이브러리 없음(차트는 인라인 SVG, 폰트만 Google Fonts).
- 구성: 센터/제약 칩 + 기간 선택 → KPI 6종 → 브릿지 워터폴 → 센터 비교 + 히트맵 → 실행 갭 → 제약별 대응력 → 실적 HTP 추이 → 상세 표. 차트 6개. 라이트/다크 자동.
- 제약 추가/삭제 UI: ＋ 제약 추가 폼, 관리 목록에서 삭제.

### 데이터가 들어오는 경로 (우선순위 순)

1. **브라우저에 저장된 수정본** — localStorage `opb-data-v1`. 엑셀 업로드나 제약 수정 시 `DB._local = true`로 저장.
2. **사이트의 `data.xlsx`** — `loadSite()`가 `fetch("data.xlsx?t=...")`로 읽음. 수정본이 없으면 `adoptSite()`로 바로 적용, 있으면 덮어쓰지 않고 "↻ 사이트 데이터 불러오기" 버튼(`#siteBtn`)만 표시.
3. **내장 `SAMPLE` 상수** — data.xlsx를 못 읽을 때의 폴백. 일별 실적은 `actualsPacked`(2026-06-15부터 84일).

### 엑셀 형식 (data.xlsx = 업로드 템플릿)

- 파서: 순수 JS zip 파서 + `DecompressionStream('deflate-raw')`.
- **Data 시트**: "ID" 헤더 행으로 센터 표(ID, Name, Base, Target, Actual, Headcount, Note)와 제약 표(ID, Group, Name, Active, 센터ID별 제약률 열) 인식. 제약률이 전부 1 미만이면 분수로 간주해 ×100. rate 셀이 전부 빈 행에서 표 종료.
- **Actuals 시트**: 센터·일자별 1행 — Date, ID, Volume, Hours. `YYYY-MM-DD` 문자열·엑셀 날짜 숫자 모두 인식 (`extractActuals()`, `excelDate()`).
- Data 시트의 Actual 열은 Actuals 시트가 없을 때만 쓰는 폴백.
- 한/영 헤더 모두 지원. 헤더 동의어는 `H`, `HA_SYN` 상수.

### 기간 엔진

- 단위: 일 / 주(ISO 주, `2026-W36`) / 월. 기본은 주별, 가장 최근 기간. 상태는 `state.gran`, `state.period`.
- `buildPeriods()`가 Actuals를 기간별로 합산 → `applyPeriod()`가 선택 기간의 실적 HTP를 각 센터의 `actual`에 써넣음 → KPI·실행 갭·표가 그 값을 사용. 추이 차트는 `renderTrend()`.
- Actuals가 없으면 기간 선택 줄(`#periodRow`)과 추이 패널(`#trendPanel`)이 자동으로 숨겨짐.

### 언어 전환 (한/영)

- 헤더 우측 EN/한국어 토글. 선택은 localStorage `opb-lang`, 없으면 `navigator.language`로 자동 판별.
- 사전은 `I18N` 상수(en/ko). 헬퍼: `T()`(단순) / `TF()`(`{0}` 치환) / `LX(obj,field)`(데이터 필드).
- HTML 정적 텍스트는 `data-i18n` · `data-i18n-html` · `data-i18n-ph` 속성으로 `applyStatic()`이 채움.
- 내장 SAMPLE은 `nameKo` 등 한국어 필드를 병기. **업로드·사이트 데이터는 번역하지 않고 원문 유지.** 그래서 라이브 사이트는 한국어 모드에서도 센터명·제약명이 영어로 나온다(data.xlsx에 영문명만 있음) — 설계대로.
- 주의: 차트 함수들이 지역변수 `L`을 좌표로 쓰므로 데이터 헬퍼 이름은 반드시 `LX` 유지.
- i18n 키를 추가할 때는 en/ko 양쪽에 모두 넣을 것.

### 문구 표기 규칙 (쿠팡 IR 표기 기준)

- 섹션 제목은 지표명 명사구, 영문은 Title Case.
- 범위 한정은 en dash (`동탄 1센터 – 생산성 브릿지`).
- 정의·방법론은 `Note:` / `주:`로 분리해 완전한 문장으로.
- KPI 증감은 부호(+/−) 대신 방향어(Up/Down, 상회/하회/감소) + 단위 포함.
- 한/영 어순이 다른 문장은 `TF()` 템플릿으로 분리.
- 구분자 `·` 대신 괄호·쉼표.

### 검증 기준값 (샘플 데이터, 최신 주 2026-W36)

가중 평균 기준 195.4 → 최종 예상 142.1, 실적 143.8 HTP. 수정 후 이 값이 유지되는지 확인할 것.

## 작업 규칙

- 한 파일(single-file HTML) 원칙 유지. 외부 CDN 스크립트 추가 금지.
- `index.html`을 고치면 **반드시 `404.html`에 복사** (`cp index.html 404.html`).
- 커밋 메시지는 한국어로 간결하게.
- 지표 정의(실행 갭 = 전체 제약 기준 고정)를 바꾸지 말 것 — 웹·엑셀이 같은 정의를 공유해야 함.
- `data.xlsx`를 바꾸면 `upload_sample.xlsx`도 함께 볼 것. 단, 샘플 파일에 실데이터가 들어가면 안 된다.

## 모바일(claude.ai/code)에서 작업할 때

폰에서는 브라우저로 렌더링을 눈으로 확인하기 어렵다. 그래서:

1. **큰 구조 변경은 피하고** 문구 수정·수치 교체·색상 조정 같은 작은 변경 위주로.
2. push 전에 **문법 검증**을 반드시 거칠 것 — `<script>` 블록을 추출해 검사하거나, 최소한 태그 짝·따옴표 짝을 확인.
3. push하면 45초 뒤 실제 사이트가 바뀐다. 되돌리려면 `git revert HEAD && git push`.
4. 로컬 PC에만 있는 것(엑셀 모델, 구버전 한국어 대시보드)은 폰에서 볼 수 없음 — 필요하면 사용자에게 요청할 것.
5. 사용자가 PC에서 이어갈 때는 먼저 `git pull`이 필요하다고 알려줄 것.

## 남은 TODO

1. **Cloudflare Access** 적용 (사내 이메일 제한) — 실데이터 넣기 전 필수. 사용자가 Cloudflare 대시보드에서 직접 설정해야 함.
2. 실데이터 수령 시: `data.xlsx` 교체 후 push (또는 사용자가 화면에서 엑셀 업로드).

---

**참고**: 사용자의 로컬 PC 프로젝트 루트에도 별도 `CLAUDE.md`가 있습니다(엑셀 모델 등 전체 산출물 관리용). 대시보드 코드에 관한 내용을 고칠 때는 양쪽이 어긋나지 않게 주의하세요.
