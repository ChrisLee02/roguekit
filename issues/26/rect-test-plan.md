# Rect 테스트 보강 계획

## 목적

- 첫 PR 범위는 `bracket-geometry/src/rect.rs` 테스트 보강
- 구조 변경 없이 기존 동작을 테스트로 더 단단하게 고정

## 대상 API

### `with_*` 계열

- `with_size`
  - 시작 좌표와 width/height로 `x2`, `y2`가 올바르게 계산되는지 확인
  - `w = 0`, `h = 0`일 때 zero-area rectangle 동작 확인
  - 음수 좌표에서도 의도한 필드값이 들어가는지 확인
- `with_exact`
  - 입력한 `x1`, `y1`, `x2`, `y2`가 그대로 저장되는지 확인
  - `with_size`와 `with_exact`가 같은 rectangle을 만들 수 있는 케이스 확인

### 그 외 `impl Rect` 함수

- `zero`
  - 모든 필드가 `0`인지 확인
  - `Default::default()`와 같은 값인지 확인
- `intersect`
  - 겹치는 경우
  - 완전히 떨어진 경우
  - 경계만 맞닿는 경우를 현재 구현이 어떻게 처리하는지 확인
- `center`
  - 짝수 크기 rectangle 중심 확인
  - 음수 좌표가 섞인 경우 확인
- `point_in_rect`
  - 왼쪽/위쪽 경계 포함 확인
  - 오른쪽/아래쪽 경계 제외 확인
  - 내부 점, 바깥 점, 모서리 점 확인
- `for_each`
  - 순회한 점 개수가 `width * height`와 맞는지 확인
  - 빈 rectangle이면 callback이 호출되지 않는지 확인
- `point_set`
  - 포함되는 점 집합이 예상과 같은지 확인
  - `for_each` 결과와 같은 집합을 만드는지 확인
- `width`, `height`
  - 일반 케이스 확인
  - `with_exact`로 축 순서가 뒤집힌 rectangle에서도 절댓값으로 계산되는지 확인

## 우선순위

1. `point_in_rect`
2. `with_size`, `with_exact`
3. `zero`
4. `width`, `height`
5. `point_set`, `for_each`
6. `intersect`
7. `center`

## 첫 PR에서 특히 넣고 싶은 것

- `point_in_rect` 경계값 테스트
- `with_size`와 `with_exact` 기본 생성 테스트
- `zero`와 `Default` 동치 테스트
- `width`, `height` 절댓값 계산 테스트
