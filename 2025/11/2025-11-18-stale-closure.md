# TIL: Stale Closure

## 클로저(Closure)란?

함수가 선언될 당시의 외부 변수를 기억하는 것임. JavaScript 중요한 개념 중 하나.

```
function outer() {
    const value = 1;

    return function inner() {
        console.log(value); // value를 기억함
    };
}

const fn = outer();
fn(); // 1 출력
```

`inner`는 `outer`가 끝나도 `value`를 기억함. 이게 클로저 구조임.

---

## 문제 상황

부서 목록을 조회한 뒤 테이블 행을 클릭했을 때:

- **첫 번째 클릭:** 부서 선택 안 됨, `departmentOptions`가 `[]`
- **두 번째 클릭부터:** 정상 작동

데이터는 로딩됐는데 **첫 클릭에서만 빈 배열을 참조하는 오류**가 발생함.

---

## 원인: Stale Closure

### Stale Closure란?

함수가 **옛날 값**을 계속 들고 있는 상태임.  
외부 값은 갱신됐는데, 함수는 생성 당시의 값만 기억하고 있어서 생기는 문제.

발생하기 쉬운 곳:

- useCallback / useEffect 의존성 누락
- setTimeout, 이벤트 리스너 같은 비동기 콜백
- state를 읽는 핸들러가 재생성되지 않는 구조

### 예시

```
let count = 0;

const logCount = () => {
    console.log(count); // 0 기억함
};

count = 10;

logCount(); // 0 출력 → stale closure
```

### React에서 문제화되는 방식

```
const departmentOptions = []; // 초기값

const handleClick = () => {
    console.log(departmentOptions); // [] 캡처됨
};

const MemoizedComponent = useCallback(() => {
    return <div onClick={handleClick} />;
}, []); // ❌ handleClick 누락

// 이후 departmentOptions = [데이터]로 업데이트됨
// 하지만 MemoizedComponent는 초기 handleClick을 계속 씀
```

---

## 실제 코드에서의 문제

### Table 컴포넌트

```
// Table/index.tsx
const RenderRow = useCallback(
    ({ row, style }) => {
        return (
            <div
                onClick={e => {
                    onTrClick(row.original, e); // onTrClick 사용
                }}
            >
                {/* ... */}
            </div>
        );
    },
    [prepareRow, page], // ❌ onTrClick 없음
);
```

### 문제 요약

- RenderRow가 `onTrClick`을 사용하고 있음
- 하지만 의존성 배열에서 `onTrClick`이 빠져 있음
- 그래서 `handleTrClick`이 최신 상태 기준으로 재생성되더라도  
  RenderRow는 **초기 함수**를 계속 사용함
- 결국 **빈 배열을 기억하는 오래된 handleTrClick**이 실행됨

### 흐름

```
1. 초기: departmentOptions = []
2. handleTrClick 생성 ([] 기억)
3. RenderRow 생성 + handleTrClick 캡처
4. 데이터 로딩: departmentOptions = [데이터]
5. handleTrClick 재생성 ([데이터] 기억)
6. ❌ RenderRow는 재생성 안 됨
7. 클릭 → 오래된 handleTrClick 실행됨
```

## 해결 방법

### **1\. 의존성 배열 정확히 선언 (핵심 해결책)**

```
const RenderRow = useCallback(
    ({ row, style }) => {
        return <div onClick={e => onTrClick(row.original, e)}>{/* ... */}</div>;
    },
    [prepareRow, page, onTrClick], // ✅ onTrClick 추가
);
```

---

### **2\. 콜백에서 state 직접 참조 줄이기**

핸들러가 특정 state에 묶이는 구조를 아예 없애는 방식.

**문제되는 방식**

```
const handleClick = () => {
    console.log(departmentOptions); // stale 가능
};
```

**안전한 방식 (필요한 값만 인자로 전달)**

```
const handleClick = item => {
    console.log(item.department); // 외부 state 직접 참조 안 함
};

<div onClick={() => handleClick(row.original)} />;
```

핸들러가 외부 상태를 기억하지 않으면 stale closure 자체가 생기지 않음.

---

### **3\. 함수형 업데이트 사용 (stale state 방지)**

비동기 업데이트에서 이전 state가 필요할 때는 항상 함수형 업데이트 사용하는 게 안전함.

**stale 발생 가능**

```
setCount(count + 1); // 오래된 count를 기억할 위험 있음
```

**안전**

```
setCount(prev => prev + 1); // 항상 최신 prev 보장
```

---

### **4\. 최신 값을 항상 유지해야 하면 useRef 활용**

특정 state 값을 “항상 최신 값”으로 유지하고 싶을 때는 ref 사용.

```
const latestOptions = useRef([]);

useEffect(() => {
    latestOptions.current = departmentOptions; // 항상 최신 값 저장
}, [departmentOptions]);

const handleClick = () => {
    console.log(latestOptions.current);
};
```

단점: ref는 렌더 트리거 안 함.

---

### **5\. React 19 이상이면 useEvent로 항상 최신 상태 참조**

React 19의 `useEvent`는 내부에서 state를 읽어도 stale이 절대 안 생김.

```
const handleClick = useEvent(() => {
    console.log(departmentOptions); // 항상 최신 state
});

return <button onClick={handleClick}>클릭</button>;
```

---

## 배운 점

- useCallback 쓸 때 **핸들러 내부에서 쓰는 값**은 무조건 의존성에 넣어야 함.
- 의존성 하나만 빠져도 stale closure로 실제 버그가 터질 수 있음.
- 비동기 로딩 데이터는 특히 stale 리스크가 큼.
- exhaustive-deps 린트는 괜히 있는 규칙이 아님.
- 클로저를 개념적으로만 알고 있었는데, 실제로 어떻게 버그로 이어지는지 확실히 체감함.

---

## 참고

- MDN - Closures
- React Docs - useCallback
