# UseEffect 남용 금지

## 서론

useEffect는 리액트에서 제공하는 가장 보편적으로 사용되는 훅 중 하나입니다.
이 훅은 데이터 가져오기, 구독, 직접 변경 등을 포함한 외부 요인/서비스와 컴포넌트를 동기화시킬 수 있지만, 아주 쉽게 남용되기도 합니다.
이 글에서는 모든 개발자가 피해야 하는 몇 가지 상황과 리액트 팀이 새로운 리액트 문서(react.dev)에서 제공하는 해결책에 대해 다뤄보겠습니다.

<br />

## 경쟁 상태

```jsx
// 마지막 요청에서 response 상태값을 저장한다는 것을 보장할 수 있는가?
function RaceConditionExample() {
  const [counter, setCounter] = useState(0);
  const [response, setResponse] = useState(0);
  const [isLoading, setIsLoading] = useState(false);

  useEffect(() => {
    const request = async (requestId) => {
      setIsLoading(true);
      await sleep(Math.random() * 3000);
      setResponse(requestId);
      setIsLoading(false);
    };
    request(counter);
  }, [counter]);

  const handleClick = () => {
    setCounter((prev) => ++prev);
  };

  return (
    <>
      //....
      <button onClick={handleClick}>Increment</button> //...
    </>
  );
}
```

예상과 다르게 동작합니다. 이전에 실행한 request가 response를 덮어서 저장하게 됩니다.

새로운 리액트 문서 예시에 따르면 위의 경쟁 상태를 클린업 함수로 처리할 수 있습니다.

```jsx
// 클린업 함수로 경쟁 상태를 처리합니다
useEffect(() => {
  let ignore = false;
  const request = async (requestId) => {
    setIsLoading(true);
    await sleep(Math.random() * 3000);
    if (!ignore) {
      setResponse(requestId);
      setIsLoading(false);
    }
  };
  request(counter);

  return () => {
    ignore = true;
  };
}, [counter]);
```

<br />

## 리렌더링

다음 예제에서는 데이터를 잘못된 방향으로 전달했을 때 렌더링이 추가로 발생하는 경우에 대해 설명하겠습니다.

**리액트의 데이터는 폭포수처럼 전달되어야 합니다.**

<br />

#### 잘못된 사용 1: 사용자가 클릭한 후 몇 번 렌더링 되나요?

```jsx
function Parent() {
    const [someState, setSomeState] = useState();
    return <Child onChange={(...) => setSomeState(...)} />
}

function Child({ onChange }) {
  const [isOn, setIsOn] = useState(false);

  useEffect(() => {
    // 추가적인 렌더링을 유발합니다
    onChange(isOn);
  }, [isOn, onChange]);

  function handleClick() {
  ️// 클릭한 후에 첫 번째 렌더링을 일으킵니다
    setIsOn(!isOn);
  }

  return <button onClick={handleClick}>Toggle</button>;
}
```

클릭 이벤트는 (첫 번째 렌더링을 만드는) 로컬 상태를 업데이트합니다.
이펙트가 실행 중이며 부모 컴포넌트가 제공하는 콜백을 호출하고, (두 번째 렌더링이 될) someState도 업데이트합니다.

이렇게 해결합니다.

```jsx
function Parent() {
    const [someState, setSomeState] = useState();
    return <Child onChange={(...) => setSomeState(...)} />
}

function Child({ onChange }) {
  const [isOn, setIsOn] = useState(false);

  // ✅ 업데이트를 일으키는 이벤트 함수에서 모든 업데이트를 수행하는 것이 좋습니다
  function handleClick() {
    const newValue = !isOn;
    setIsOn(newValue);
    onChange(newValue);
  }

  return <button onClick={handleClick}>Toggle</button>;
}
```

useEffect에서 호출되는 onChange는 전혀 필요하지 않습니다.
onClick 핸들러 내에서 간단히 호출할 수 있으며, 동일한 결과를 얻으면서도 렌더링을 한 번 줄입니다.

<br />

#### 잘못된 사용 2: 데이터 흐름 체인을 망가뜨리기

```jsx
function Parent() {
  const [data, setData] = useState(null);

  return <Child onFetched={setData} />;
}

// Effect 내에서 부모에게 데이터를 전달하는 건 좋지 않습니다
function Child({ onFetched }) {
  const data = useFetchData();

  useEffect(() => {
    if (data) {
      // 스파게티 코드를 만드는 건가요?
      onFetched(data);
    }
  }, [onFetched, data]);

  return <>{JSON.stringify(data)}</>;
}
```

데이터가 부모에서 자식 방향으로 흐르도록 유지해야 에러 추적, 유지보수, 디버깅 측면에서 더욱 유리합니다.

게다가 코드의 가독성을 높이고 따라 하기 쉽게 만들어 줍니다. 실제 프로덕트 코드의 양은 훨씬 더 많습니다.
따라서 코드가 복잡해질수록 어떤 컴포넌트가 콜백을 부모 컴포넌트에 전달하는지, 그리고 얼마나 잘못된 방향으로 가고 있는지를 추적하는 건 더욱 어려워집니다.

데이터의 '진실의 출처(source of truth)'를 이해하는 것 또한 어렵습니다.

부모 컴포넌트에서 상태를 관리하도록 해결합니다!

```jsx
// 제안 #1 데이터를 가져오는 로직을 상위 컴포넌트에서 처리합니다
function Parent() {
  const data = useFetchData();

  // 올바른 방향으로 자식에게 데이터를 전달합니다
  return <Child data={data} />;
}

function Child({ data }) {
  return <>{JSON.stringify(data)}</>;
}
```

자식 컴포넌트에 데이터를 가져오는 로직을 유지할 수밖에 없다면, 별도의 useState 또는 useEffect를 정의하는 대신 부모에게 전달받은 핸들러를 사용하여 데이터와 상호작용 하는 것이 바람직합니다.

```jsx
// 제안 #2 - onSuccess/onError 핸들러를 자식 컴포넌트에 전달합니다
function Parent() {
    function handleSuccess = (data) => {
        // 성공을 처리하는 로직
    }
    function handleError = (error) => {
        // 실패를 처리하는 로직
        toast(error.messasge)
    }
    // 올바른 방향으로 자식에게 데이터를 전달합니다
    return <Child onSuccess={handleSuccess} onError={handleError} />;
}

function Child({ onSuccess, onError }) {
  // 다른 이펙트는 관여하지 않고 데이터를 가져왔을 때 핸들러를 호출하는 훅을 사용하는 것이 좋습니다.
  const mutate = useMutateData({ onSuccess, onError });

  return ...;
}
```

<br />

## 초기화를 해야 할 때

앱 런타임 중에 딱 한 번만 초기화를 실행하는 경우가 많습니다.

```jsx
function App() {
  useEffect(() => {
    someOneTimeLogic();
  }, []);
  // ...
}
```

'someOneTimeLogic' 함수가 (인증 프로바이더와 같은) 무언가를 초기화하는 중이며 한 번만 실행해야 한다고 가정합니다.

이 경우에서 useEffect를 사용해서는 안 되는 몇 가지 이유를 생각해 보겠습니다.

리액트는 개발 모드(strict 모드와 함께 사용할 때)에서 항상 컴포넌트를 다시 마운트 하므로, 클린업 함수가 필요합니다.
어차피 한 번만 적용될 텐데 왜 클린업 함수를 구현해야 할까요? 이에 대해서는 이렇게 말하고 싶습니다…

동시성 모드는 개발이 완료되지는 않았지만, 이에 대한 의미는 명확합니다. 또한 컴포넌트가 한 번만 렌더링 된다는 보장은 없으므로 우리는 이에 대비해야 합니다!

#### 대안 1. 이전에 마운트 되었는지 나타내는 최상위 플래그를 사용하기

```jsx
let didInit = false;

function App() {
  useEffect(() => {
    if (!didInit) {
      didInit = true;
      // 앱 로드 시 한 번만 실행됩니다
      someOneTimeLogic();
    }
  }, []);
  // ...
}
```

#### 대안 2. 앱이 렌더되기 전에 로직을 적용하기

```jsx
if (typeof window !== "undefined") {
  // 앱 로드 시 한 번만 실행됩니다
  someOneTimeLogic();
}

function App() {
  // ...
}
```

<br />

## 언마운트 되어도 괜찮습니다

리액트 팀은 "언마운트 된 컴포넌트에서 리액트 상태 업데이트를 수행할 수 없습니다"라는 경고를 제거했으므로 이제 이를 두려워할 필요가 없습니다. 이 문제를 우회하면서 구현하는 것이 원래 문제보다 더 나쁜 결과를 초래할 수 있습니다!

메모리 누수가 발생하지 않으며, 언마운트 된 컴포넌트에서 setState를 호출해도 에러가 발생하지 않습니다.
이러한 상황에서 setState를 막을 이유가 없습니다. 실제로 이를 피하려다 불필요한 useEffect를 추가하기도 합니다."

일반적인 우회 방법은 아래와 같았습니다.

```jsx
const useMountedRef = () => {
  const mounted = useRef(true);
  useEffect(() => {
    return () => {
      mounted.current = false;
    };
  }, []);
  return mounted;
};

const MyCompoenet = () => {
  const [loading, setLoading] = useState(false);
  const mountedRef = useMountedRef();

  const handleDeleteBill = async (id) => {
    setLoading(true);
    await axios.delete(`/bill/${id}`);
    // 언마운트된 컴포넌트에서 상태를 업데이트하지 않으려 했지만, 오히려 이 방법이 더 좋지 않습니다.
    if (mountedRef.current) {
      setLoading(false);
    }
  };

  return (
    <button onClick={handleDeleteBill} disabled={loading}>
      Delete Bill
    </button>
  );
};
```

언마운트된 컴포넌트에서 상태의 업데이트를 피하지 말아야 하는 이유는 다음과 같습니다.

1. 불필요한 useEffect를 사용합니다.
2. 코드의 복잡성이 커집니다.
3. 리액트는 상태를 보존하는 새로운 기능을 곧 출시할 예정입니다. 이 기능은 탭을 전환할 때 상태를 초기화하지 않고 유지해 줄 것입니다. 지금 업데이트를 하지 않으면, 미래에 관련된 코드를 유지보수 해야 할 가능성이 큽니다.
   예를 들어, 기존 탭을 전환하는 컴포넌트에 새로운 상태 보존 메커니즘(각 탭에 state-key를 추가하는 방식)을 지원하려고 한다고 가정해 봅시다. 사용자가 API 호출 중에 탭을 전환할 경우, 언마운트 되어서 응답에 대해 setState를 하지 않는다면, 사용자가 다시 돌아왔을 때 상태 보존 메커니즘 때문에 데이터가 정의되지 않은 상태로 유지될 것입니다. 이런 상황에서는 필연적으로 리팩터링을 해야만 합니다. (그리고 사실 언마운트 된 컴포넌트에서 setState를 우회할 필요가 전혀 없습니다 🤷🏻.)"

```jsx
const MyCompoenet = () => {
  const [loading, setLoading] = useState(false);

  const handleDeleteBill = async (id) => {
    setLoading(true);
    await axios.delete(`/bill/${id}`);
    // 상태를 업데이트하는 걸 두려워하지 마세요.
    setLoading(false);
  };

  return (
    <button onClick={handleDeleteBill} disabled={loading}>
      Delete Bill
    </button>
  );
};
```

<br />

## 정리

- 콜백으로 데이터를 상위 컴포넌트로 전달하려고 한다면, 전체 상태를 끌어올려 자식에게 전달하는 것을 고려해 보세요. 아래로 흐를 때 렌더링이 최적화되고, 데이터 흐름을 선언적이고 명확하게 유지할 수 있습니다.
- useEffect 내에서 데이터를 가져오거나 계산하는 것은 괜찮지만, 경쟁 상태를 피하려면 항상 클린업 로직을 실행해야 합니다.
- 특별한 이유가 없는 한, useEffect 내부에서 로컬 상태를 설정하지 마세요. 불필요한 추가 렌더링이 발생할 수 있습니다.
- useEffect를 남발하지 마세요. 렌더링 중에 계산할 수 있다면 그렇게 하세요.
- 언마운트 된 컴포넌트에서 상태를 업데이트하는 것을 두려워할 필요는 없습니다. 이를 피하려다가 오히려 리액트 생태계를 오용하게 될 것입니다.

<br />

> 그럼 useEffect는 언제 사용해야 하나요?

- WebSocket과 같은 외부 연결/구독/리소스, Firebase와 같은 SDK 도구, 분석 도구 또는 구글맵이나 ChartJS와 같은 API를 노출하는 라이브러리를 초기화해야 할 때 사용합니다.
- 외부 리소스에 대한 미처리된 열린 연결/구독이 있어 컴포넌트가 언마운트될 때 해당 연결을 닫거나 처리해야 할 때 사용합니다.
- 컴포넌트 생애 주기 동안 한 번만 실행되어야 하는 이펙트(useMount)지만 애플리케이션 생애 주기 동안 한 번만 실행되어야 하는 건 아닐 때 사용합니다.
- 실제로 최적화 문제가 발생했으나 다른 해결책이 없는 경우에도 사용할 수 있습니다. 예를 들어 아주 큰 표에서 사용자가 정렬/필터 옵션을 사용하는 데 문제가 있고, 페이지네이션이나 가상화는 선택지가 아닐 때, useEffect/useMemo를 사용한 메모이제이션이 필요한 경우입니다. (리액트 19를 사용하고 있나요? 리액트 컴파일러에 대해 읽어보세요.)

> 출처: https://velog.io/@typo/leave-useeffect-alone
