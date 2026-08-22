# Performance View — 센터별 출고 생산성 브릿지 대시보드

설계 기준 생산성(UPH)에서 출발해 물량·인력·운영 제약을 순차 적용하고, 최종 예상 생산성과 실제 실적을 대조하는 단일 파일 대시보드입니다. 제약별로 가장 잘 대응한 센터도 함께 보여줍니다.

- 외부 라이브러리 없음 (차트는 전부 인라인 SVG)
- 빌드 단계 없음 — `index.html` 하나가 전부
- 라이트/다크 테마 자동 대응

## 배포 (Cloudflare Pages)

`main`에 커밋하면 Cloudflare Pages가 자동 재배포합니다.

### 최초 1회 연결

1. https://dash.cloudflare.com → **Compute (Workers & Pages)** → **Create** → **Pages** 탭 → **Connect to Git**
2. GitHub 인증 → `Rocket-Ian-lab/Performance-view` 선택
3. 빌드 설정

   | 항목 | 값 |
   |---|---|
   | Framework preset | `None` |
   | Build command | *(비워 둠)* |
   | Build output directory | `/` |
   | Root directory | *(비워 둠)* |

4. **Save and Deploy** → 30초 내 `https://<프로젝트명>.pages.dev` 생성

이후 `main` 브랜치 커밋마다 자동 배포되며, Deployments 목록에서 이전 버전으로 롤백할 수 있습니다.

## 데이터 교체

`index.html` 안의 두 블록만 수정하면 전체 지표·차트가 자동 재계산됩니다. GitHub 웹에서 파일을 열고 연필 아이콘으로 바로 편집해도 되고, 커밋하는 순간 재배포됩니다.

```js
const CENTERS = [
  {id:"DT", name:"동탄 1센터", sname:"동탄", code:"DT-CFC",
   base:212,      // 기준 생산성 (설계 표준 UPH)
   target:165,    // 목표 UPH
   actual:162.4,  // 실제 실적 UPH
   hc:186,        // 인원 — 가중평균 계산에 사용
   note:"자동화 소터 운영"},
  ...
];

const RATES = {
  // 각 값 = 직전 단계까지 남은 생산성 대비 손실 비율(%)
  DT:{mix:4.5, peak:3.2, new:6.0, ojt:2.4, wait:5.5, stock:2.0, brk:6.5},
  ...
};
```

- **센터 추가** — `CENTERS`에 항목 하나, `RATES`에 같은 `id`의 행 하나
- **제약 추가** — `CONSTRAINTS`에 `{id, group, gname, name, short, desc}` 추가 후 `RATES`의 모든 센터에 그 `id` 값 입력 (`group`은 `vol` / `hr` / `ops`)

## 계산 규칙

```
최종 예상 = 기준 × Π(1 − 제약률ᵢ)
```

각 손실 UPH는 직전 단계까지 남은 생산성에 제약률을 곱한 값이며, 제약은 물량 → 인력 → 운영 순으로 적용됩니다.

**실행 갭** = 실적 − 기본 시나리오(전체 제약 적용) 최종 예상. 양수면 현장 실행이 모델 예상보다 좋았다는 뜻이고, 음수면 모델이 잡지 못한 손실이 추가로 있다는 신호입니다. 화면의 제약 토글은 이 값에 영향을 주지 않습니다.

**제약별 대응력** = 해당 제약의 제약률이 가장 낮은 센터를 최우수로 봅니다.

## 파일

| 경로 | 역할 |
|---|---|
| `index.html` | 대시보드 본체 |
| `404.html` | 잘못된 경로 접근 시에도 대시보드 표시 |
| `_headers` | 보안 헤더 · 캐시 무효화 · 검색엔진 색인 차단 |
| `robots.txt` | 크롤러 차단 |

현재 수치는 구조 확인용 샘플입니다.
