# Dijkstra 테스트 보강 계획

## 목적

- 다음 PR 범위는 `bracket-pathfinding/src/dijkstra.rs` 테스트 보강
- 공개 API 동작을 테스트로 고정하고 회귀를 막는 데 집중
- 구현 리팩터링 없이 현재 계약을 드러내는 방향으로 접근

## 현재 상태

- 현재 테스트는 `find_highest_exit` 기본 케이스 1개뿐임
- 아래 public behavior는 테스트 공백이 큼
  - `new`
  - `new_weighted`
  - `new_empty`
  - `clear`
  - `find_lowest_exit`
  - `max_depth` 제한 동작

## 대상 API

### 생성 계열

- `new`
  - 간단한 선형 맵에서 거리 전파 결과가 예상과 같은지 확인
  - 도달 가능한 타일과 도달 불가능한 타일이 `f32::MAX`로 구분되는지 확인
- `new_weighted`
  - 시작점 가중치가 인접 타일까지 올바르게 반영되는지 확인
  - 일반 `new`와 다르게 0이 아닌 시작 비용에서 탐색이 시작되는지 확인
- `new_empty`
  - 지정한 크기만큼 `map`이 생성되는지 확인
  - 초기값이 전부 `f32::MAX`인지 확인

### 상태 초기화

- `clear`
  - 기존에 값이 채워진 `map`을 전부 `f32::MAX`로 되돌리는지 확인
  - 크기 자체는 유지되는지 확인

### 경로 추적 helper

- `find_lowest_exit`
  - 더 낮은 cost를 가진 출구를 선택하는지 확인
  - 출구가 없으면 `None`을 반환하는지 확인
- `find_highest_exit`
  - 기존 테스트를 유지하되, 출구가 없을 때 `None`인 케이스를 보강할 수 있음

### 깊이 제한

- `max_depth`
  - 누적 cost가 `max_depth` 이상이면 전파되지 않는지 확인
  - 경계값에서 `>=` 비교가 적용되는 현재 구현을 고정

## 추천 테스트 시나리오

1. 1x3 또는 1x4 선형 맵에서 `new` 기본 거리 전파 확인
2. 동일 맵에서 `find_lowest_exit`가 시작점 방향으로 역추적되는지 확인
3. `new_weighted`에 `(start, 3.0)` 같은 입력을 주고 인접 타일 값이 `4.0`이 되는지 확인
4. 작은 `max_depth`를 주고 먼 타일이 여전히 `f32::MAX`로 남는지 확인
5. `new_empty` 이후 수동으로 값을 채운 뒤 `clear`가 초기화하는지 확인
6. 출구가 없는 타일에서 `find_lowest_exit`와 `find_highest_exit`가 모두 `None`인지 확인

## 테스트용 맵 구성

- 기본 맵은 현재 테스트처럼 작은 선형 `MiniMap`을 재사용 가능
- `None` 반환 검증용으로 출구가 없는 타일을 가진 작은 맵을 하나 더 두는 방식이 단순함
- weighted/max-depth 확인은 복잡한 2D 맵보다 1D 연결 구조가 기대값 계산이 쉬움

## 우선순위

1. `find_lowest_exit`
2. `new_weighted`
3. `max_depth`
4. `clear`
5. `new_empty`
6. `find_highest_exit`의 `None` 케이스

## 주의할 점

- 현재 구현은 시작 타일 자체를 `0.0`으로 기록하지 않고, 이웃 타일부터 값을 채움
- 따라서 테스트는 “일반적인 Dijkstra 기대”가 아니라 “현재 라이브러리의 실제 동작”을 기준으로 써야 함
- weighted 경로도 동일하게 시작 노드 기록보다는 전파 결과를 검증하는 편이 안전함
