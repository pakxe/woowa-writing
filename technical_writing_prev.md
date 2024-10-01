### 에러 핸들링이란?

에러 핸들링이란 **코드 실행 중 발생할 수 있는 예기치 않은 오류나 문제를 탐지**하고, 이를 **적절히 처리하여 프로그램이 비정상적으로 종료되지 않도록 하는 기술**을 말합니다.

### 에러 핸들링의 중요성

이런 에러 핸들링 기술이 중요한 이유가 무엇일까요?

(각 중요성에 이미지 첨부)

- **사용자 경험을 개선**:  적절한 핸들링을 통해 사용자의 불편을 최소화할 수 있습니다.
- **프로그램 안정성 향상**: 애플리케이션이 중단되지 않고 계속 동작할 수 있습니다.
- **디버깅 용이**: 에러 로그를 통해 문제를 빠르게 파악하고 해결할 수 있습니다.

위 장점들을 누리기 위해 제가 참여하고 있는 행동대장 서비스에도 에러 핸들링이 되고 있는데요. 제가 이 서비스에  어떤 에러 핸들링 전략을 왜 적용했는지, 어떻게 개선했는지, 어떤 결과가 있었는지를 소개해보도록 하겠습니다.

### 에러 핸들링 도입 start! 우리 서비스의 에러 핸들링 전략은?

일단 저희 행동대장 서비스의 에러 핸들링 전략은 “에러가 발생하면 토스트로 안내”하는 것이었습니다.

(이미지 또는 영상)

그래서 보통 에러 핸들링으로 어떤 방법들을 사용하는지 찾아봤는데요. 일단 대중적으로는 두 가지 방법이 주로 사용되고 있었습니다. 

1. try catch
2. 에러 바운더리

다만 1번 방법으로만 하기에는 무리가 있을 것이라 판단했습니다. 그 이유는 서비스 개발이 어느정도 진행된 상태에서 에러 핸들링 기능을 도입하는거라 request에러(=api)가 발생될 수 있는 곳이 굉장히 많았습니다. 그래서 이 모든 곳에 try catch를 적용하는 건 시간도 오래 걸리고, 만약 요구사항이 바뀌면 빠르게 대응하기도 어려울 것이라 생각했어요. 

그래서 2번째 방법인 에러 바운더리는 어떨까 고민해보았는데요. 에러 바운더리라는 개념은 오류 발생 시 fallback UI를 띄우는 데에 최적화된 방법이었습니다. 저희 요구사항인 토스트만 띄우는 건 기능적으로 불가능했습니다. 

이유를 탐색하기 위해 에러 바운더리를 조금 뜯어 살펴보았는데요. 에러 바운더리의 동작은 쉽게 말하면 칠드런을 받고 칠드런 안에서 에러가 발생해 에러 상태가 변동되면 면 그 안에 render함수를 실행하게 됩니다. render함수는 hasError가 false가 되어도 render, 성공해도 render 함수가 실행됩니다. 보통 hasError라면 펄백 ui, hasError가 트루라면 칠드런을 return하는 식으로 에러 바운더리가 기능합니다.

만약 hasError가 트루가 되었을 때 children을 에러 발생 `이전의 모습` 그대로 둘 수 있을까요? 이건 불가능했습니다. 칠드런을 이전의 모습 그대로 남기기 위해 return children을 한다면 이 칠드런에서 또 에러를 던지게되고 또 render함수를 부르고 칠드런을 렌더링하고 무한 루프에 갇히게 됩니다. 

따라서 에러 바운더리 자체를 기존 모습을 유지하며 ToastUI를 띄우기 위한 방법으로 사용하기에는 무리가 있었습니다. 다만 최상단에서 한 번에 컨트롤할 수 있다는 특징은 꼭 활용하고 싶었는데요.

그래서 1, 2방법의 특징을 적당히 합쳐 제 3의 방법으로 에러 핸들링을 구현하게 되었습니다. 

### 에러 핸들링 v1 - 에러는 토스트로 안내해라!

(구조도 제공)

일단 이 에러 핸들링의 핵심 전략은 업데이터와 구독자입니다. 

저희 서비스는 모든 api요청은 request라는 함수를 거치게 됩니다. request함수 안에서 api에러가 발생했을 경우 RequestError라는 에러 코드가 담긴 에러 객체를 던집니다. 

그리고 업데이터가 이 던져진 에러를 잡아 에러 state를 던져진 에러로 업데이트 합니다. 

이후 이 에러 state를 구독하고 있는 구독자가 에러객체 안의 에러 코드를 확인해 ToastUI 또는 fallbackUI를 띄우게 됩니다. 

위 과정을 한 스텝씩 예시 코드와 함께 살펴보겠습니다. 

### 딥다이브

1. ..

request안에서 실제 api 요청을 보낼 때 만약 에러가 발생했다면, createError라는 함수가 RequestError를 만들어 던집니다. 

```jsx
const response: Response = await fetch(url, requestInit);

    if (!response.ok) {
      throw await createError({
        response,
        body: requestInit.body ? JSON.stringify(requestInit.body) : null,
        requestInit,
        errorHandlingStrategy,
      });
    }
    
const createError = async ({
  response,
  body,
  requestInit,
  errorHandlingStrategy,
}: WithErrorHandlingStrategy<CreateError>) => {
  const {errorCode, message}: ErrorInfo = await response.json();

  return new RequestError({
    status: response.status,
    requestBody: body,
    endpoint: response.url,
    name: errorCode,
    method: requestInit.method,
    message,
    errorCode,
  });
};
```

1. ..

탠스택쿼리 사용 시 에러 발생할 때 전역 콜백을 실행할 수 있는 기능이 제공되고 있는데요. 여기에 에러 상태를 업데이트하는 로직을 호출해 사용합니다.

```jsx
const QueryClientBoundary = ({children}: React.PropsWithChildren) => {
  const {updateAppError} = useAppErrorStore();

  const queryClient = new QueryClient({
    queryCache: new QueryCache({
      onError: (error: Error) => {
        updateAppError(error);
      },
    }),
    mutationCache: new MutationCache({
      onError: (error: Error) => {
        updateAppError(error);
      },
    }),
  });

  return <QueryClientProvider client={queryClient}>{children}</QueryClientProvider>;
};
```

1. …

그리고 이 업데이트하는 에러state를 구독하고 있는 컴포넌트를 두었는데요. 

```jsx
const ErrorCatcher = ({children}: React.PropsWithChildren) => {
  const {error} = useErrorStore();

  useEffect(() => {
    if (!error) return;

    logError(error);

    if (!isPredictableError(error)) throw error;

    toast.error(SERVER_ERROR_MESSAGES[error.errorCode], {
      showingTime: 3000,
      position: 'bottom',
      bottom: '8rem',
    });
  }, [error]);

  return children;
};

```

이 컴포넌트는 에러 객체의 에러 코드 프로퍼티에 따라 해당 에러가 예측 가능한 에러인지 불가능한 에러인지 판단합니다. 예측가능한 에러인 경우 토스트를 실행하고, 예측 불가능한 에러인 경우 그대로 던져 외부 에러 바운더리에서 잡도록 했습니다. 

이때 예측 가능한 에러는 무엇인지, 에러를 또 왜 던지는지 의문이 생기셨을텐데요.

예측 가능한 에러는 서버에서 명시적으로 에러코드 준 것 중에 일반적인 것들을 의미합니다. 이름은 10자 이내여야 한다는 NAME_LENGTH같은 에러 코드를 말합니다. 다만 INTERNAL_SERVER_ERROR같은 사용자에게 안내한다고 해서 사용자가 할 수 있는 것이 없는 경우는 에러코드임에도 불구하고 예측 불가능한 에러라고 결정했습니다. 

그리고 다시 던지는 이유는 앞서 말했듯이 에러가 발생해도 사용자가 이를 해결할 수 없는 에러인 경우 즉, 예측할 수 없는 에러인 경우.. TOastUI로 띄워주는 것이 무의미합니다. 왜냐면 사용자는 빈화면 또는 더이상 진행되지 않는 화면에 토스트만 올라갔다 내려갔다 하는 화면을 바라보게 되기 때문입니다. 따라서 이런 예측 불가능한 에러인 경우는 에러 바운더리가 잡도록 에러를 던져 대체 페이지 또는 문의하기 페이지를 띄워주는 것이 적합하다고 판단했어요. 

결과적으로는 위 구현물로 에러 핸들링 요구사항인 “에러 발생 시 토스트로 안내”를 만족할 수 있었습니다.

그리고 api를 사용하는 페이지를 개발할 때 에러 핸들링 코드를 페이지 내부 혹은 가까운 곳에 작성하는 번거로움을 겪지 않아도 되었습니다. 성공케이스와 실패케이스가 난잡하게 섞이는.. 제가 생각하기로는 별로인 코드를 피할 수 잇었습니다. 

그리고  최상단의 몇 개의 컴포넌트가 에러 핸들링 업무를 담당하여 책임 분리를 이룰 수 있었습니다. 에러 전략이 바뀌어도 이 안에서만 수정하면 되기 때문에 유지보수에도 유리하다고 생각했어요.

### 에러 핸들링 v2 - GET메서드에는 펄백UI를 사용해라! 그 외는 v1그대로~

이대로 변함이 없다면 좋겠지만, 안타깝게도 유지보수가 필요하게 되었어요.  

시간이 어느정도 지나고 나서 에러 핸들링 요구사항이 “GET에러 발생 시 FallbackUI를 사용하고 그 외는 기존대로 토스트로 안내”로 바뀌게 되었습니다.  

그래도 막막하진 않았어요. 이전에 만든 v1에서 만들었던 에러 핸들링 책임을 갖는 컴포넌트들을 조금 수정한다면 이 요구사항을 만족할 수 있을 것 같았습니다. 

### 개인적인 요구사항 추가

다만 한가지 의문이 들었습니다. 모든 에러 GET이 FallbackUI로 표시되어야 할까요? 예를 들어 채팅창 기능이 있다고 하면 채팅 목록은 GET으로 불러오고 있을 것 같습니다. 그런데 에러가 발생해 fallbackUI로 화면을 채운다면 이전에 주고받아 캐싱되어있는 채팅 목록도 불러오지 못한채로 화면이 대체되어 버립니다. 이 경우 ToastUI로 알려주어도 충분할 것 같았어요. 이렇듯 상황에 따라 ToastUI와 FallbackUI 둘 중 하나를 선택해서 사용하는 경우가 있지 않을까 생각했어요. 

그래서 주어진 요구사항을 좀 더 확장해 재정의하게 되었습니다. 

- GET메서드 사용한 api 요청에서 에러가 발생한 경우
- ToastUI 또는 FallbackUI를 api를 호출해 사용하는 곳에서 선택할 수 있도록 한다.
- 그 외는 v1 그대로~

### 요구사항대로 개발하기

이제 새로 정의한 요구사항대로 개발하는 과정을 실제 예시 코드와 함께 살펴보도록 하겠습니다.

![image.png](%E1%84%90%E1%85%A6%E1%84%8F%E1%85%A9%E1%84%90%E1%85%A9%E1%86%A8%2010f44c1e63c68044bc7bf9347a0e9978/image.png)

일단 요구사항을 만족했을 때 어떤 모습을 갖길 원하는지를 먼저 생각해보았는데요. 아래 코드같은 형태로 사용할 수 있길 원했습니다.

```jsx
const {data} = useRequestData({errorDisplayMode: 'toast'}) // ToastUI

const {data} = useRequestData({errorDisplayMode: 'errorBoundary'}) // FallBackUI
```

ToastUI 또는 FallbackUI를 선택할 수 있도록 하기 위해 api요청하는 코드에서 errorDisplayMode라는 어떤 에러 표시 UI를 사용할건지 인자를 받도록 했습니다. 같은 api요청 훅이어도 errorDisplayMode인자로 넘기는 값에 따라 에러 발생시 띄우는 UI가 달라집니다.

~~구현한 방법을 요약하자면 `커스텀에러 객체를 사용한 분기처리`입니다.~~ 

toast의 경우 v1에서 했던 것이니 따로 수정할 필요는 없습니다. 다만, errorBoundary라는 값을 넘기면 에러 바운더리를 사용해 fallbackUI를 렌더링하도록 해야합니다. 

그럼 이때의 에러 바운더리는 어떤 에러 바운더리일까요?

GET메서드에서 펄백 UI를 사용하고 싶다 라는 의미는 지역적인 에러 바운더리를 중첩해 이곳의 fallbackUI를 사용하겠다는 의미입니다. 따라서 `지역적인 에러 바운더리`를 사용할 수 있도록 v1의 코드를 수정해야합니다. 

v1의 코드는 에러가 발생하면 자동으로 업데이터-구독자의 구조가 트리거되어 toast 또는 최외곽 에러 바운더리가 트리거됩니다. 지역적인 에러 바운더리를 끼워넣어도 저 둘 중 하나가 실행되는 것을 막을 수 없습니다. 

따라서 v1에서 만든것들 트리거를 막기 위해 업데이터가 실행되는 곳인 queryClient에 분기문을 추가해주기로 했습니다. 이 분기문운 GET메서드이며 && `errorDisplayMode: 'errorBoundary'`일 때 얼리리턴을 해주어야  업데이터-구독자의 트리거를 막을 수 있습니다. 

queryClient는 에러 객체를 받고있습니다. 따라서 분기문도 에러 객체안의 특징적인 프로퍼티를 사용하여 걸어줘야합니다. ~~그래서 에러가 발생했을 때의 인자나 부가적인 정보는 커스텀 에러 객체에 담아서 넘기도록 했습니다.~~ 

그래서 GET메서드이며 && `errorDisplayMode: 'errorBoundary'` 조건문을 위해 RequestGetError라는 커스텀 에러 객체를 만들어 errorDisplayMode라는 프로퍼티를 보유하도록 했습니다. 이 에러 객체는 이전에 말했던, 모든 api가 겨쳐가는 함수인 request 에서 GET메서드를 사용하는 api에서 오류가 났을 경우 RequestGetError를 만들어 던지도록 했습니다. 그리고 아래와 같은 형태로 api를 호출할 경우 RequestGetError 객체에 errorDisplayMode에 넘겨줬던 값이 담기게 되죠.

```jsx
// 1
const {data} = useRequestData({errorDisplayMode: 'errorBoundary'}) // FallBackUI

// 2
 if (requestInit.method === 'GET') {
    return new RequestGetError({
      method: requestInit.method,
      errorDisplayMode,
      errorCode,
      // ... 
    });
  }
  
  // 3
     queryCache: new QueryCache({
      onError: (error: Error) => {
        if (isRequestGetError(error) && error.errorDisplayMode === 'errorBoundary') return;

        updateAppError(error);
      },
    }),
```

RequestGetError라는 에러 객체가 queryClient의 조건문을 위해 위 흐름으로 흘러들어오게 됩니다. 이렇게 errorDisplayMode가 errorBoundary일 경우 얼리리턴으로 업데이터-구독자의 트리거를 막게 됩니다. 

이렇게 지역적인 에러 바운더리를 사용할 수 있게 되었습니다.
