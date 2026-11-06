---
title: ADR - 005 시각 비교 픽셀 diff (pixelmatch)
path:  euta-workspace / 40.설계 / ADR - 005 시각-비교-픽셀-diff
ref: ARCH 측정·분석(화면 증거·시각 비교) · SPEC visual
---

# ADR - 005 시각 비교 픽셀 diff (pixelmatch)

## 상태

채택 (2026-09-29)

## 맥락

- `visual` 스텝은 기준 이미지와 현재 화면을 픽셀 단위로 비교하고, 임계치를 넘으면 실패시키며 diff 이미지를 남긴다. 기준이 없으면 만든다.
- 엔진은 Playwright **라이브러리**만 쓴다(`@playwright/test` 러너를 쓰지 않음).

## 결정

- **pixelmatch + pngjs** 로 비교한다. 픽셀 단위 색 차이 허용치는 pixelmatch `threshold: 0.1`(기본값)로 고정하고, 스텝의 `threshold` 는 **다른 픽셀의 비율**(0~1, 기본 0.01)로 정의한다. 크기가 다르면 즉시 실패.
- 캡처는 `animations: 'disabled'` · `caret: 'hide'` · 마스크(스텝 `mask` + secrets 입력란)를 적용한다.
- 기준 이미지는 실행 폴더 밖 `<스크립트 폴더>/baselines/<스크립트 slug>/<name>.png`(옵션 `baselineDir` / CLI `--baseline-dir`). 실행 폴더에는 실제 · 기준 사본 · diff 를 남긴다. `visual.<name>.diffRatio` 를 지표로 기록한다.

## 이유

- 순수 JS · 의존 작음 · 널리 쓰임(Playwright 내부 비교도 pixelmatch 계열). 네이티브 빌드가 없어 Electron 에서도 그대로 동작.
- "다른 픽셀 비율" 은 사람이 이해하기 쉽고 화면 크기와 무관하게 쓸 수 있다.

## 고려한 대안

| 대안 | 제외 이유 |
|---|---|
| `expect(page).toHaveScreenshot()` | `@playwright/test` 러너 전용 — 엔진 실행 모델과 맞지 않음 |
| odiff | 빠르지만 네이티브 바이너리 |
| sharp 기반 비교 | 네이티브(libvips), 재빌드 부담 |
| resemblejs | canvas 의존 |

## 결과

- 기준 이미지는 OS · 브라우저 · 폰트 · 뷰포트에 따라 달라진다. 기준은 같은 환경에서 만든 것을 써야 한다(예제의 `examples/baselines/` 는 git 에서 제외).
- 안티앨리어싱 차이를 따로 무시하는 옵션·영역 단위 허용치는 1단계 범위 밖.
