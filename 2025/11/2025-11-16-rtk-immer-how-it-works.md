# TIL: Redux Toolkit과 Immer 동작 원리

## Redux Toolkit 내부에서 Immer가 어떻게 동작하는가

Redux Toolkit(RTK)은 기본적으로 **Immer가 내장**되어 있다.
그래서 `createSlice` 안에서 작성한 리듀서는 겉으로 보면 **직접 state를 수정하는(mutate) 코드처럼 보이지만**,
실제로는 Immer가 내부에서 **불변성을 지키면서 새로운 state를 생성**한다.

```js
state.count += 1; // mutate처럼 보임
```

하지만 실제로 일어나는 일은 이렇다:

1. Immer가 state를 **draft 객체**로 감싼다.
2. draft에 가해진 변경을 Proxy로 감지한다.
3. 변경된 부분만 새로운 객체로 복사(구조적 공유).
4. 완전히 새로운 state를 반환한다.

### produce 개념

Immer의 핵심 함수는 `produce`.

```js
const nextState = produce(prevState, (draft) => {
  draft.count += 1;
});
```

- 첫 번째 인자: 실제 state
- 두 번째 인자: draft를 수정하는 함수
- 반환값: Immer가 만들어준 새로운 state

### draft vs 실제 state

- **draft**: "변경된 것처럼 보이게" Immer가 제공한 Proxy 객체
- **실제 state**: 불변성이 지켜진 새로운 객체

개발자가 변경하는 건 draft뿐이고,
실제 state 갱신은 전부 Immer가 한다.

---

## 직접 불변성 관리 vs Immer 사용하는 패턴 비교

### 1) 직접 불변성 지키는 코드

```js
const reducer = (state, action) => {
  return {
    ...state,
    user: {
      ...state.user,
      profile: {
        ...state.user.profile,
        age: action.payload,
      },
    },
  };
};
```

---

### 2) Immer 사용 (RTK 기본 패턴)

```js
const reducer = (state, action) => {
  state.user.profile.age = action.payload;
};
```

---

### 3) Immer 코드가 왜 안전하고 깔끔한가

- Proxy가 draft에서 일어난 변경만 감지
- 변경된 경로만 새로운 객체 생성 → 성능 최적화
- 나머지 부분은 참조 재사용(구조적 공유)
- 불변성 깨질 위험 없음
- 코드 복잡도 감소

결국 Immer가 해주는 핵심 작업은:

### 🔑 구조적 공유(Structural Sharing)

- 변경된 부분만 새로운 객체로 복사
- 변경되지 않은 레벨은 기존 참조 재사용

이 방식 덕분에
“겉으로는 mutate, 실제로는 immutable”
이라는 RTK의 깔끔한 상태 업데이트 방식이 가능해진다.

---

## 💡 전체 요약

### RTK + Immer 핵심 흐름

1. draft를 수정한다.
2. Immer가 변경을 추적한다.
3. Immer가 불변성 유지된 새 state를 만든다.

Redux Toolkit 에서는 직접 불변성 신경 쓰지 말고
draft에 mutate 하듯이 작성하면 되고
Immer가 알아서 불변성을 보장한다.
