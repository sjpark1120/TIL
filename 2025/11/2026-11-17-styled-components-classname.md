# TIL: 커스텀 컴포넌트에 styled-components 안 먹는 이유

오늘 공통 컴포넌트를 styled-components로 감싸려고 했는데 스타일이 전혀 적용이 안 됨.  
원인부터 말하면 **그 공통 컴포넌트 자체에 `className` props가 아예 없었다.**  
그러니까 styled-components가 알아서 className을 주입해줘도, 컴포넌트가 그걸 받을 수도 없고, DOM에도 넘길 수가 없었던 거임.

## 뭐가 문제?

styled-components는 내부에서 스타일을 클래스로 만들어서 컴포넌트에 `className`으로 넣어준다.  
근데 내가 감싸려던 컴포넌트가 그걸 받지도 않고 DOM에도 전달 안 하면 스타일이 먹을 리가 없음.

내가 쓰려던 구조

```
const MyBox = () => {
  return <div>hello</div>; // className 없음
};

const StyledBox = styled(MyBox)`
  color: red;
`;
```

styled-components 입장에서는 분명 className을 주입하지만  
MyBox가 그걸 무시하니까 `<div>`는 아무 변화 없이 렌더됨.

## 공식 문서

styled-components docs:  
**"컴포넌트가 전달된 className을 DOM에 붙여줘야 styled()가 제대로 동작함."**

공식 예제:

```
const Link = ({ className, children }) => (
  <a className={className}>{children}</a>
);

const StyledLink = styled(Link)`
  color: palevioletred;
  font-weight: bold;
`;
```

핵심: `Link`가 `className`을 받고 `<a>`에 붙여줌.

## 그래서 어떻게 해야 하냐

```
const MyBox = ({ className }) => {
  return <div className={className}>hello</div>;
};

const StyledBox = styled(MyBox)`
  color: red;
  padding: 8px;
`;
```

이제야 styled-components가 생성한 className이 DOM까지 제대로 전달되고 스타일이 먹는다.

## 결론

- styled로 감싸는 커스텀 컴포넌트에 **className props가 없으면 스타일 안 먹는다**
- className을 받고 그대로 DOM 요소에 붙여주는 건 필수
- 오늘의 삽질: **공통 컴포넌트에 className이 없었다 → styled-components가 적용될리가 없었다**
