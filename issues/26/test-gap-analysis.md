# Issue #26 - ChrisLee02 작업 범위 정리

## 목적

이 문서는 이슈 `#26`을 `ChrisLee02` 관점에서 다시 정리한 작업 메모입니다.

- 목표는 **기존 동작에 대한 테스트 보강**
- 구조 개편 없이 `#[cfg(test)]` 보강 위주로 접근
- `neokkk`가 댓글에서 선점 의사를 밝힌 범위는 취소선으로 표시
- 따라서 이 문서의 실질 목적은 **충돌 가능성이 낮은 테스트 후보를 빠르게 고르는 것**

## 최신 댓글 기준 작업 상태

- `neokkk`는 `2026-05-05T12:59:51Z`에 `algorithm-traits`, `color`, `terminal` 작업 의사를 명시함
- `utilForever`는 `2026-05-06T07:47:40Z`에 기존 테스트 보강과 신규 테스트 추가 모두 가능하다고 확인함
- `ChrisLee02`는 `2026-05-06T11:23:47Z`에 안 겹치는 범위에서 작업하겠다고 답변함

현재 기준으로 `ChrisLee02`는 아래 원칙으로 범위를 잡는 것이 가장 안전합니다.

- 우선 `geometry`, `pathfinding`, `random`, `embedding`, `rex`, `noise` 쪽을 본다
- `algorithm-traits`, `color`, `terminal`은 가능하면 피한다
- 꼭 건드려야 한다면 세부 파일 단위로 먼저 범위를 명확히 적고 들어간다

## 저장소 테스트 구조

- 이 워크스페이스는 별도 `tests/` 디렉터리보다 각 소스 파일 내부의 `#[cfg(test)]` 인라인 테스트를 주로 사용합니다.
- 따라서 이슈 `#26` 작업도 새 테스트 폴더를 만드는 것보다 **해당 모듈 파일 안에 테스트를 추가하는 방식**이 현재 구조와 가장 잘 맞습니다.
- 첫 기여 기준으로는 pure logic, 경계조건, off-by-one, parsing, deterministic behavior를 다루는 파일이 가장 안전합니다.

## ChrisLee02 우선 후보

### 1. 가장 추천

- [bracket-pathfinding/src/dijkstra.rs](/home/chrislee/ossca-2026-rltk/bracket-pathfinding/src/dijkstra.rs)
  - 현재 `find_highest_exit` 위주로만 테스트가 있고 `find_lowest_exit`, `new_weighted`, `max_depth`, `clear`는 거의 비어 있음
  - public behavior를 직접 강화할 수 있어서 이슈 취지와 잘 맞음
- [bracket-embedding/src/embedding.rs](/home/chrislee/ossca-2026-rltk/bracket-embedding/src/embedding.rs)
  - 기본 폰트 preload와 경로 정규화가 무테스트
  - 파일이 작고 독립적이라 첫 PR로 리뷰받기 편함
- [bracket-rex/src/xpcolor.rs](/home/chrislee/ossca-2026-rltk/bracket-rex/src/xpcolor.rs)
  - 색상 변환, transparent 판정, read/write가 거의 잠겨 있지 않음
  - 파일 포맷 관련 핵심 로직이라 테스트 보강 가치가 높음
- [bracket-random/src/parsing.rs](/home/chrislee/ossca-2026-rltk/bracket-random/src/parsing.rs)
  - junk input rejection이 약함
  - `"abc1d6"` 또는 `"1d6xyz"` 같은 문자열 처리의 허점을 잡기 좋음

### 2. 가볍게 시작하기 좋은 후보

- [bracket-geometry/src/rect.rs](/home/chrislee/ossca-2026-rltk/bracket-geometry/src/rect.rs)
  - `point_in_rect`의 half-open 경계 규칙을 모서리 기준으로 더 강하게 잠글 수 있음
- [bracket-geometry/src/distance.rs](/home/chrislee/ossca-2026-rltk/bracket-geometry/src/distance.rs)
  - `Diagonal` 알고리즘이 public API에 있지만 직접 테스트되지 않음
- [bracket-rex/src/rex.rs](/home/chrislee/ossca-2026-rltk/bracket-rex/src/rex.rs)
  - `XpLayer::get/get_mut`의 인덱싱 규칙과 out-of-bounds `None` 계약 보강이 가능함
- [bracket-random/src/random.rs](/home/chrislee/ossca-2026-rltk/bracket-random/src/random.rs)
  - 동일 seed 재현성과 길이 1 슬라이스 경계값 테스트를 추가하기 좋음

### 3. 탐구형 후보

- [bracket-geometry/src/lines.rs](/home/chrislee/ossca-2026-rltk/bracket-geometry/src/lines.rs)
  - 대각선, 역방향, start/end 보존 테스트가 비어 있음
- [bracket-pathfinding/src/astar.rs](/home/chrislee/ossca-2026-rltk/bracket-pathfinding/src/astar.rs)
  - 실패 경로와 `start == end` 케이스가 비어 있음
- [bracket-noise/src/fastnoise.rs](/home/chrislee/ossca-2026-rltk/bracket-noise/src/fastnoise.rs)
  - deterministic behavior와 setter normalization 테스트가 약함
- [bracket-random/src/iterators.rs](/home/chrislee/ossca-2026-rltk/bracket-random/src/iterators.rs)
  - 모듈 전체가 무테스트
  - `DiceIterator` 범위와 재현성 검증이 쉬운 편임

## 선점 의사 표시된 범위

아래는 `neokkk`가 이미 작업 의사를 밝힌 범위입니다. `ChrisLee02` 기준으로는 기본적으로 제외 대상으로 봅니다.

### 크레이트/영역

- ~~`bracket-algorithm-traits`~~
- ~~`bracket-color`~~
- ~~`bracket-terminal`~~

### 개별 후보 중 제외 처리

- ~~[bracket-color/src/rgb.rs](/home/chrislee/ossca-2026-rltk/bracket-color/src/rgb.rs)~~
- ~~[bracket-color/src/rgba.rs](/home/chrislee/ossca-2026-rltk/bracket-color/src/rgba.rs)~~
- ~~[bracket-color/src/lerpit.rs](/home/chrislee/ossca-2026-rltk/bracket-color/src/lerpit.rs)~~
- ~~[bracket-terminal/src/consoles/text/format_string.rs](/home/chrislee/ossca-2026-rltk/bracket-terminal/src/consoles/text/format_string.rs)~~
- ~~[bracket-terminal/src/input/input_handler.rs](/home/chrislee/ossca-2026-rltk/bracket-terminal/src/input/input_handler.rs)~~
- ~~[bracket-terminal/src/consoles/text/textblock.rs](/home/chrislee/ossca-2026-rltk/bracket-terminal/src/consoles/text/textblock.rs)~~

## 바로 고르기 좋은 순서

1. `dijkstra.rs`
2. `embedding.rs`
3. `xpcolor.rs`
4. `parsing.rs`
5. `rect.rs`

## 메모

- 이 저장소는 인라인 테스트 문화가 강해서, 기존 파일 안에 `mod tests`를 보강하는 형태가 가장 자연스럽습니다.
- `ChrisLee02` 기준으로는 선점 영역을 피해서 PR 범위를 선명하게 만드는 편이 리뷰와 이슈 커뮤니케이션 모두 깔끔합니다.
- 가장 충돌이 적고 설명하기 쉬운 선택은 `pathfinding`, `embedding`, `rex`, `random` 계열입니다.
