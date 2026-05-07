# Issue #20 - Point / Point3 통합 분석

## 목적

이 문서는 이슈 `#20`의 요구사항인 “`Point`와 `Point3`를 하나로 묶는 방법 연구”를 코드베이스 기준으로 검토한 결과입니다.

핵심 질문은 두 가지입니다.

1. 이 저장소에서 정말로 **하나의 공개 구조체**로 합치는 것이 현실적인가?
2. 아니라면, 어떤 수준의 통합이 가장 현실적이고 안전한가?

이번 정리는 서브에이전트 3개를 병렬로 써서 아래 세 축으로 조사했습니다.

- `Point` / `Point3` API와 구현 중복
- 실제 사용처와 변경 파급 범위
- 가능한 Rust 설계 대안과 현실성

## 결론 요약

- 이 저장소에서 `Point`와 `Point3`를 **하나의 공개 구조체로 완전히 통합하는 것**은 현실성이 낮습니다.
- 이유는 `Point`가 워크스페이스 전반의 핵심 2D 공개 API이고, `Point3`는 상대적으로 제한된 3D 경로에서만 사용되기 때문입니다.
- 더 현실적인 방향은:
  - 공개 타입 `Point`와 `Point3`는 유지
  - 내부 구현만 매크로나 작은 helper trait로 공통화
  - 필요하면 변환/공통 로직을 추가

즉, 이 이슈는 실제 구현 관점에서는 “엄밀한 단일 공개 구조체”보다 **중복 구현 제거와 설계 정리** 쪽으로 해석하는 것이 더 적절합니다.

## 현재 구조

### 1. 타입 정의

- `Point`: `bracket-geometry/src/point.rs`
- `Point3`: `bracket-geometry/src/point3.rs`

둘 다 정수 좌표 타입이고, `Add/Sub/Mul/Div`, `AddAssign/SubAssign/MulAssign/DivAssign` 같은 성분별 연산을 거의 같은 방식으로 구현합니다.

하지만 공개 API는 이미 비대칭입니다.

### 2. API 비대칭

`Point`와 `Point3`는 연산 패턴은 닮았지만 생성/변환/helper에서 차이가 큽니다.

| 항목 | `Point` | `Point3` | 의미 |
|---|---|---|---|
| 좌표 | `x`, `y` | `x`, `y`, `z` | 차원 자체가 다름 |
| derive | `Hash` 포함 | `Hash` 없음 | 컬렉션 호환성 차이 |
| `new` 실패 정책 | 실패 시 `0` 대체 | 실패 시 panic | 공개 동작 차이 |
| const/helper | `constant`, `zero` 있음 | 없음 | 2D API가 더 큼 |
| tuple 변환 | `from_tuple`, `to_tuple`, `to_unsigned_tuple` | `from_tuple`만 있음 | 역변환 비대칭 |
| index helper | `to_index(width)` 있음 | 없음 | 2D 전용 의미 |
| UV 변환 | `to_vec2`, `from_vec2`, `From<Vec2>` | `to_vec3`, `From<Vec3>` | API 형태 차이 |

중요한 차이 몇 가지:

- `Point::new`는 변환 실패 시 `0`으로 대체합니다.
- `Point3::new`는 변환 실패 시 panic합니다.
- `Point`는 `Hash`, `zero`, `constant`, `to_index`를 갖고 있지만 `Point3`는 그렇지 않습니다.
- `Point`는 tuple/`Vec2` 변환 API가 더 풍부합니다.

이 차이 때문에 “같은 타입으로 묶는다”는 건 단순한 중복 제거가 아니라 **공개 API 재설계**가 됩니다.

## 실제 사용 범위

### 1. 사용량 차이

로컬 검색 기준으로 `.rs` 파일 안에서:

- `Point`: 약 `839`회
- `Point3`: 약 `153`회

`Point`가 훨씬 더 넓게 퍼져 있고, 2D API 중심 구조가 분명합니다.

### 2. `Point3` 핵심 사용처

`Point3` 직접 등장 파일은 많지 않습니다. 핵심은 아래 축에 몰려 있습니다.

- `bracket-geometry/src/point3.rs`
- `bracket-geometry/src/distance.rs`
- `bracket-algorithm-traits/src/algorithm3d.rs`
- `bracket-terminal/examples/dwarfmap.rs`
- `bracket-bevy/examples/bevy_dwarfmap.rs`
- `rltk/examples-deprecated/ex14-dwarfmap.rs`

즉, `Point3`는 넓게 퍼져 있지는 않지만, **공개 타입/trait 서명에 직접 들어가 있기 때문에** 바꾸면 외부 호환성 위험은 큽니다.

### 3. 공개 표면

특히 아래 API는 외부 사용자 코드까지 흔들 수 있습니다.

- `bracket_geometry::prelude::Point3`
- `DistanceAlg::distance3d`
- `Algorithm3D`의 `point3d_to_index`, `index_to_point3d`, `dimensions`, `in_bounds`
- `Point3::new`, `Point3::from_tuple`
- `Point3`의 공개 필드 `x`, `y`, `z`

또한 `Point3`는 최상위 `bracket-lib`, `bracket-terminal`, `bracket-bevy`, `rltk`를 통해 다시 re-export 됩니다. 따라서 `bracket-geometry` 내부 문제로 끝나지 않습니다.

## 설계 대안 비교

### 대안 1. `PointN<const D: usize>` + type alias

예시 방향:

```rust
pub struct PointN<const D: usize> {
    pub coords: [i32; D],
}

pub type Point = PointN<2>;
pub type Point3 = PointN<3>;
```

장점:

- 개념적으로는 가장 “하나의 타입”에 가깝습니다.
- 내부 구현 공유가 명확합니다.

단점:

- 현재 코드베이스는 `pt.x`, `pt.y`, `pt.z`, `Point::new(x, y)`, `Point3::new(x, y, z)` 같은 문법에 강하게 의존합니다.
- 배열 기반 generic은 ergonomics가 떨어집니다.
- alias만으로는 이름별로 다른 inherent constructor를 자연스럽게 유지하기 어렵습니다.
- 문서/예제/API 수정량이 큽니다.

판단:

- 이론적으로 가능
- 현재 저장소에서는 공개 API 비용이 너무 큼

### 대안 2. 단일 3D 구조체 + 2D는 `z = 0`

예시 방향:

```rust
pub struct Point {
    pub x: i32,
    pub y: i32,
    pub z: i32,
}
```

장점:

- 구조체는 정말 하나만 남습니다.

단점:

- 기존 2D API의 의미가 바뀝니다.
- `Point::new(x, y)`와 3D 생성자 공존 문제가 생깁니다.
- 2D 연산/거리/trait 전반에 “숨은 z” 개념이 들어옵니다.
- `Point3`를 alias/newtype로 유지해도 필드 접근/생성자 ergonomics가 깨집니다.

판단:

- 미학적으로는 단순해 보이지만 실제론 비현실적

### 대안 3. 공통 내부 표현 + 외부 `Point` / `Point3` façade 유지

예시 방향:

- 내부에 공통 좌표 표현 또는 helper를 두고
- 겉에는 `Point`, `Point3`를 그대로 유지

장점:

- 외부 호환성이 높습니다.
- 구현 중복을 어느 정도 줄일 수 있습니다.

단점:

- 공개적으로는 여전히 타입이 2개라서 “하나의 구조체”라는 목표와는 거리가 있습니다.
- 구현 복잡도 대비 이득이 아주 크진 않습니다.

판단:

- 타협안으로는 가능
- 하지만 더 간단한 대안이 있음

### 대안 4. trait 기반 공통화

예시 방향:

- 좌표 접근, 성분별 연산, 거리 축약 등을 내부 trait로 일반화

장점:

- 공개 타입을 거의 건드리지 않습니다.
- 알고리즘 공통화에 유리합니다.

단점:

- `Point` / `Point3` 자체 impl 중복을 완전히 해소하지는 못할 수 있습니다.
- trait 설계가 과하면 오히려 읽기 어려워질 수 있습니다.

판단:

- 현실성 높음
- 다만 이 저장소에서는 매크로보다 이득이 더 크다고 단정하기는 어려움

### 대안 5. 매크로 기반 공통 구현 공유

예시 방향:

- `Point`와 `Point3` 구조체는 유지
- 반복되는 arithmetic impl, assign impl, 테스트 패턴을 매크로로 생성

장점:

- 공개 API 파손이 거의 없습니다.
- 현재 중복의 대부분을 직접 줄일 수 있습니다.
- 구현 난이도가 가장 합리적입니다.

단점:

- 엄밀한 의미의 “하나의 공개 구조체”는 아닙니다.
- helper 메서드의 차원별 비대칭은 그대로 남습니다.

판단:

- 이 저장소 맥락에서 가장 현실적

## 왜 “진짜 단일 공개 구조체”가 비현실적인가

핵심 이유는 세 가지입니다.

1. `Point`가 이미 2D 공개 API의 중심입니다. 사용량과 사용처가 `Point3`보다 훨씬 큽니다.
2. `Algorithm2D` / `Algorithm3D`, `distance2d` / `distance3d`처럼 상위 API 경계 자체가 이미 차원별로 분리돼 있습니다.
3. Rust에서 alias/newtype/const generics를 써도 현재의 `Point::new(x, y)` / `Point3::new(x, y, z)` / `.x .y .z` ergonomics를 동시에 매끈하게 지키기 어렵습니다.

즉, “하나의 구조체”를 얻는 순간 잃는 것이 더 많습니다.

## 현실적인 권고안

### 권고

- 공개 타입 `Point`, `Point3`는 유지
- 내부 구현만 공통화
- 우선순위는 매크로 기반 공통화
- 필요하면 그 다음 단계로 작은 내부 helper trait 도입

### 이유

- public API 파손을 최소화할 수 있습니다.
- 현재 중복의 주된 영역이 산술 impl과 테스트 패턴이기 때문입니다.
- `Algorithm2D` / `Algorithm3D`, `distance2d` / `distance3d`처럼 이미 차원별로 갈라진 상위 구조와 충돌하지 않습니다.

## 실제 작업 시 예상 수정 범위

만약 이 이슈를 “현실적으로 구현 가능한 방향”으로 진행한다면, 수정 범위는 대체로 아래처럼 보입니다.

### 최소 범위

- `bracket-geometry/src/point.rs`
- `bracket-geometry/src/point3.rs`

여기서 반복되는 arithmetic impl, assign impl, 일부 테스트를 공통화

### 중간 범위

- `bracket-geometry/src/distance.rs`
- `bracket-algorithm-traits/src/algorithm3d.rs`
- `bracket-geometry/src/lib.rs`

문서와 helper 사용 패턴까지 정리

### 넓은 범위

- `bracket-geometry/README.md`
- `bracket-algorithm-traits/README.md`
- `bracket-terminal/examples/dwarfmap.rs`
- `bracket-bevy/examples/bevy_dwarfmap.rs`
- `rltk/examples-deprecated/ex14-dwarfmap.rs`

여기까지 가면 public naming/usage 가이드까지 바뀌는 수준입니다.

## 구현 전에 확인할 점

- 이 이슈의 의도가 “정말 하나의 공개 구조체”인지
- 아니면 “중복된 Point 계열 구현을 하나의 설계 아래로 묶는 것”인지

현재 코드베이스를 보면 후자 해석이 훨씬 자연스럽습니다.

원본 이슈 문구만 보면 전자처럼 읽히지만, 실제 저장소 구조와 호환성 비용을 보면 후자로 접근하는 편이 훨씬 방어적이고 실용적입니다.

## 추천 진행 방식

1. 이슈 코멘트나 PR 설명에서 “공개 API 통합”과 “내부 구현 통합”을 구분해서 설명
2. 첫 단계는 `Point` / `Point3`의 중복 impl 공통화에 집중
3. public API 이름(`Point`, `Point3`)은 유지
4. 필요하면 후속 이슈로 차원 일반화 설계를 별도 논의

## 한 줄 결론

이 저장소에서 `Point`와 `Point3`를 진짜 하나의 공개 구조체로 합치는 것은 비용 대비 이득이 작고, 가장 현실적인 해법은 **공개 타입은 유지한 채 내부 구현만 공통화하는 것**입니다.
