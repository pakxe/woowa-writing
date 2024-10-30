# [React] 행동대장 서비스로 알아보는 프론트엔드 에러 핸들링 전략과 개선 과정

<img src="https://velog.velcdn.com/images/pakxe/post/891d28af-a07a-4d60-bed9-100ef1c109a4/image.jpg" />

# ■ 시작하며
프론트엔드 개발을 하다 보면, 에러 핸들링 코드를 작성해야만 하는 순간을 마주하게 됩니다. 혹시 그럴 때마다 try-catch 또는 반복적인 코드를 작성하고 계신가요? 만약 그렇다면 이 글을 가볍게 읽어보시길 추천합니다.

이번 글에서는 제가 참여하고 있는 [행동 대장](https://haengdong.pro/) 프로젝트에 적용된 에러 핸들링 전략을 소개하고자 합니다. 어떤 방식으로 에러를 처리했는지, 왜 그 방식을 선택했는지, 결과는 어땠는지, 그리고 개선 과정까지 함께 살펴볼 예정입니다.

참고로, 에러 바운더리와 [TanStack Query](https://tanstack.com/query/latest)에 대한 기본적인 이해가 있다면 본 글을 더 수월하게 읽으실 수 있습니다.

이 글에 적힌 것이 완벽한 답이 아니니 '이런 에러 핸들링 방법도 있구나' 라고 생각해주시면 좋을 것 같아요.

# ■ 에러 핸들링이 뭐에요?
시작은 에러 핸들링이 무엇인지에 대해서 먼저 알아보겠습니다.

에러 핸들링이란 코드 실행 중 발생할 수 있는 예기치 않은 오류나 문제를 탐지하고, 이를 적절히 처리하여 프로그램이 비정상적으로 종료되지 않도록 하는 기술을 말합니다.

이 글에서는 Toast UI 또는 에러 바운더리를 사용한 Fallback UI에만 한정해 설명해 드릴 예정입니다.

![](https://velog.velcdn.com/images/pakxe/post/34137a87-5fbd-41f9-87e7-9165c01fbde9/image.png)

위 이미지처럼 처리하는 거라고 생각하시면 됩니다.

# ■ 에러 핸들링 왜 하나요?
에러를 적절히 핸들링해주면 애플리케이션이 비정상적으로 종료되는 것을 막을 수 있습니다. 에러가 발생했다고 알려주지도 않고, 그냥 흰 화면만 보이면 사용자 입장에서는 굉장히 당황스러울 수 있어요.

그리고 개발자 입장에서도 에러 로그를 심어 모니터링 하면 어떤 에러가 자주 발생하고 있는지 파악하기 용이합니다. 버그 유발 요소들을 금방 찾고 빠르게 해결할 수 있어요.

# ■ 에러 핸들링 v1
## ▌ 요구사항
이런 장점들을 누리기 위해서 제가 참여하고 있는 행동대장 서비스에도 에러 핸들링 버전 1을 도입하게 되었습니다. 

요구 사항은 아래와 같았어요.
> **"에러가 발생하면 Toast UI로 안내해주세요."**

![](https://velog.velcdn.com/images/pakxe/post/54a329af-fcb4-473d-87b4-1bc2cc0ffb83/image.gif)

위 gif처럼 화면 어딘가에 슝하고 나타나서 슝하고 사라지는 걸 Toast UI라고 합니다.


## ▌ 주로 사용되고 있는 에러 핸들링 방법은?
그래서 프론트엔드에서 주로 사용되고 있는 에러 핸들링 방법은 어떤 것들이 있는지 먼저 찾아봤는데요.

2가지 방법이 제일 많이 사용되고 있었습니다.

![](https://velog.velcdn.com/images/pakxe/post/4de31112-c85d-4d5a-a099-de113346ed05/image.png)

### ▎ 1. try-catch
다만 try-catch로는 저희 서비스의 에러 핸들링을 하기엔 무리가 있으리라 판단했는데요.

그 이유는 서비스 개발이 어느 정도 진행된 상태에서 에러 핸들링 기능을 도입하는 거라 api에러가 발생할 수 있는 곳이 굉장히 많았습니다. 

그래서 에러가 발생할 수 있는 모든 위치에 try-catch를  건 무리였어요. 시간도 굉장히 오래 걸릴 거고, 요구 사항이 바뀌면 그 모든 곳을 찾아서 수정해줘야 하니 유지보수도 어려우리라 판단했습니다. 가장 간단하긴 하지만요.

### ▎ 2. 에러 바운더리
그래서 2번째 방법인 에러 바운더리는 어떨까 고민해보았는데요. 

에러 바운더리라는 개념은 오류가 발생했을 때 Fallback UI를 띄우는 데에만 특화된 방법이었습니다. 에러 바운더리를 사용하면 에러 발생 시 Fallback UI로 화면을 갈아 끼우기 때문에 기존의 화면 그대로를 유지하는 Toast UI를 띄우는 건 불가능했어요.

이유는 에러 바운더리 내부 구현에 있는데요.

![](https://velog.velcdn.com/images/pakxe/post/1a3a2217-0a49-4f25-bd50-328ffa7bcc9b/image.png)

에러 바운더리는 hasError라는 상태를 갖고 있습니다. 에러가 발생해 이 hasError 값이 변경될 때마다 render함수를 실행합니다. 기본적으로는 render함수에서 children을 return하고, 에러가 발생했을 때 Fallback UI를 return합니다.

그럼 에러가 발생했을 때도 children을 return하면 기존 화면 그대로를 유지할 수 있지 않냐 라고 생각할 수 있는데요. 아래처럼요.
![](https://velog.velcdn.com/images/pakxe/post/c5625180-8e89-431b-b540-3813cfa074af/image.png)


아래 이미지를 보고 읽어보면 더 이해하기 쉽습니다. 이렇게 바꾸면 결과적으로는 무한루프에 갇히게 되는데요.

일단 처음에 render함수가 실행되어 children을 return 합니다. 만약 children에 있는 api요청에서 계속 500 에러가 발생하고 있는 상황이라면 이 에러 바운더리의 hasError가 true가 됩니다. hasError 상태가 바뀌었으니 다시 render함수가 실행되겠지만, 또 children을 return합니다. 그리고 이 children에서는 또 500에러가 던져지겠죠. 결국 무한 루프에 갇히게 됩니다. 

![](https://velog.velcdn.com/images/pakxe/post/6dceeec5-b33f-499a-8217-8543fa014572/image.png)

따라서 에러 바운더리를 이 서비스의 에러 핸들링 방법으로 사용하긴 어려웠어요. 다만 에러 바운더리가 가진 특징인 최상단에서 한 번에 에러를 핸들링할 수 있다는 건 굉장히 활용하고 싶다고 생각했습니다. 반복되는 코드를 줄일 수 있고 책임도 뭉쳐있게 되니까요.

그래서 이 특징을 갖는 에러 핸들링 방법을 생각해 구현하게 되었습니다.

## ▌ 핵심 구조
일단 이 에러 핸들링 v1의 핵심 전략은 아래와 같습니다.
> 핵심 전략: **최상단의 업데이터와 구독자**

![](https://velog.velcdn.com/images/pakxe/post/314b7b24-ebf5-480a-8ed7-dd37b8a73019/image.png)

저희 서비스의 모든 api요청은 `request`라는 함수를 거치게 됩니다. 이 `requst`함수에서는 api요청 중 에러가 발생했을 경우 RequestError라는 에러 코드가 담긴 에러 객체를 던집니다. 

그리고 `업데이터`는 request함수에서 던져진 RequestError를 잡아 에러 상태를 던져졌던 RequestError로 업데이트합니다.

이후 이 에러 상태를 구독하고 있는 `구독자`가 RequestError안의 에러 코드를 확인해 적절하게 Toast UI또는 전역 에러 바운더리의 Fallback UI를 띄우게 됩니다.

위 과정을 한 스텝씩 실제 구현 코드와 함께 살펴봅시다.

## ▌ 실제 구현
방금 말씀드렸던 구조는 실제 코드에서 아래와 같은 계층 구조로 사용할 수 있습니다. 
![](https://velog.velcdn.com/images/pakxe/post/f4716937-d0d5-456e-b2c0-1ac0f0771b04/image.png)
`전역 에러 바운더리` 하위에 `업데이터` 역할인 queryClient를 둡니다. 그리고 더 하위에 `구독자` 역할인 ErrorCatcher를 둡니다.
그리고 에러가 발생할 수도 있는 페이지 또는 컴포넌트를 안에 위치시킵니다.

앞서 말씀드렸다시피 저희 서비스의 모든 api요청은 request라는 함수를 거칩니다. 이 안에서 api요청을 보낼 때 에러가 발생했다면 `RequestError` 객체를 생성해 throw합니다.

![](https://velog.velcdn.com/images/pakxe/post/56aa8f75-30dd-4ae2-81ea-70a41d49947c/image.png)

다음은 `업데이터`입니다. 이 기능은 QueryClientBoundary라는 컴포넌트에 존재하는데요. (참고: 이 컴포넌트는 탠스택 쿼리 라이브러리를 사용한다면 캐싱된 값에 접근하기 위한 컴포넌트입니다.)

이 탠스택 쿼리에서는 query, mutation의 요청 수행 중 `에러가 발생했을 때 실행할 콜백`을 넘겨줄 수 있습니다. 따라서 이곳에 `에러 객체를 받아 에러 상태를 업데이트`하는 updateError를 호출하는 콜백을 넘겨주었어요.

![](https://velog.velcdn.com/images/pakxe/post/f361807e-290d-42ea-ab70-14ffec0d522e/image.png)

updateError는 useErrorStore에서 return하고 있는 함수인데요. useErrorStore는 그냥 에러 상태와 에러 상태를 업데이트하는 코드를 반환하는 훅입니다. useState와 동일해요.

마지막으로는 `구독자`입니다. 이름은 ErrorCatcher로, 업데이트되는 에러 상태를 useEffect로 구독하고 있습니다.

![](https://velog.velcdn.com/images/pakxe/post/1353c378-b797-41a7-8ab0-7134ad68efa4/image.png)

useEffect의 내부 코드를 읽어보면 isPredictableError라는 함수에 errorCode를 넘기고 있는 걸 볼 수 있는데요. 이름 그대로 예측 가능한 에러인지를 확인하는 함수입니다. 

예측 가능한 에러인 경우는 Toast UI를 띄웁니다. 예측 불가능한 에러인 경우는 그대로 throw해 ErrorCatcher를 감싸고 있는 전역 에러 바운더리에서 잡혀 Fallback UI를 띄우게 됩니다.

![](https://velog.velcdn.com/images/pakxe/post/d4af7c6f-4d1e-45c2-9d30-8e93c939c1c9/image.png)

예측 가능하다는 게 무슨 말인지 궁금하실 텐데요. `예측이 가능하다` 라는건 백엔드에서 명확하게 전달해준 에러 코드들을 의미합니다. 예로는 이름 길이가 제한보다 긴 경우 `INVALID_NAME_LENGTH와 같은 에러 코드가 전달되는데요. 이런 에러 코드는 백엔드가 이런 에러가 날 수 있음을 ' 예측해서' 만들어진 것이기 때문에 `예측 가능한` 이라는 표현을 사용하게 되었습니다.

반면 예측 불가능한 에러는 `INTERNAL_SERVER_ERROR`와 같은 에러 코드 또는 `서버에서 정의한 에러 코드 목록 안에 없는 에러` 를 의미합니다. 예로 서버에서 에러가 발생한 경우는 Toast UI로 안내해도 빈 화면만 남을 것이고 사용자는 해결할 방법이 없어 당황할 수 있습니다. 따라서 예측 불가능한 에러인 경우는 Fallback UI를 사용하도록 전역 에러 바운더리로 감싸주었어요.

## ▌ 결과
결과적으로는 위 구현물로 서비스의 에러 핸들링 요구사항인 `에러 발생 시 Toast UI로 안내`를 만족할 수 있었습니다.

사용하며 느꼈던 장점이 여러 개 있는데요. 

일단 성공 케이스와 실패 케이스를 한 곳에 작성하지 않고 분리할 수 있어 핵심 로직에만 집중할 수 있었어요. 새로운 기능이 추가된다고 해도 에러 처리를 어떻게 할지 고민할 필요가 없어졌습니다. 이미 자동으로 에러 처리가 적용될 것이니까요.

그리고 일부 컴포넌트가 에러 핸들링 업무를 전담하기 때문에 책임 분리 측면에서도 좋았습니다. 이 때문에 에러 핸들링 전략이 바뀌어도 유지 보수에도 용이했어요.

만약 에러 로깅이 필요한 경우에도 한 곳에만 작성해주면 되기 때문에 편리했습니다. 

# ■ 에러 핸들링 v2
## ▌ 요구사항
이대로 변함이 없다면 좋겠지만, 안타깝게도 새로운 요구사항이 들어왔습니다. 

> **"페이지 초기 렌더링 중 데이터를 받아오는 데 오류가 발생했을 경우 Fallback UI를 사용해 주세요"**

이 말은 결국 GET 메서드에서 오류가 발생했을 경우 에러 바운더리를 사용해 Fallback UI를 띄우라는 뜻입니다.

다만 GET 메서드에서 에러가 발생했을 때 모두 Fallback UI를 띄우는 것보단 에러 발생 맥락에 맞게 어떤 UI를 사용할 것인지 선택할 수 있도록 자유를 주는게 좋을 것 같아요.

그래서 주어진 요구사항을 좀 더 확장해 재정의했습니다.

> **"GET 메서드에서 오류가 발생했을 경우, Toast UI 또는 Fallback UI 중 하나를 선택할 수 있도록 합니다. 그 외는 v1 그대로 유지합니다."**

지금부터는 이 요구 사항을 만족하는 새로운 에러 핸들링 버전 2를 개발해보겠습니다.

## ▌ 핵심 구조
v2의 핵심 전략은 아래와 같습니다.
> 핵심 전략: **커스텀 에러 객체를 사용한 분기**

![](https://velog.velcdn.com/images/pakxe/post/50d528b7-19d4-4689-8103-25590980941c/image.png)

처음으로는 에러 발생 시 Toast UI, Fallback UI중 어떤 UI를 사용할 것인지에 대해 인자를 받습니다. 

그리고 이 인자를 커스텀 에러 객체에 담습니다. 이 에러 객체는 탠스택 쿼리에서 제공하는 에러가 발생했을 때 실행하는 콜백의 인자로 넘어갑니다. 

만약 fallback일 경우 조건문으로 얼리 리턴해 v1에서의 에러 상태를 업데이트하는 updateError함수의 호출을 막습니다. Toast UI가 뜨는 것을 막아야 하기 때문입니다. 

만약 toast일 경우 v1에서 구현한 것들이 그대로 실행되도록 합니다.

위 과정을 한 스텝씩 실제 구현 코드와 함께 살펴보도록 하겠습니다.

## ▌ 실제 구현
api 요청에서 에러가 발생했을 때 어떤 UI를 띄울지는 api 요청 훅의 errorDisplayMode 인자로 넘겨 조작할 수 있도록 했습니다.

![](https://velog.velcdn.com/images/pakxe/post/750dac44-4558-4880-ba24-f6c5a30a8340/image.png)

'toast'를 넘길 경우 v1을 그대로 실행합니다. 
'fallback'을 넘길 경우 에러 바운더리를 사용해 Fallback UI를 보여주도록 합니다.

이때 "GET 메서드에서 오류가 발생했을 때 Fallback UI를 사용해라" 라는 의미에 대해서 생각해볼 필요가 있는데요. 이 의미는 `지역적인 에러 바운더리`를 중첩해 해당 에러 바운더리의 Fallback UI를 사용하겠다는 겁니다.
![](https://velog.velcdn.com/images/pakxe/post/1b7f3f6e-2473-4375-8df4-027560c11573/image.png)
다만 v1의 코드 그 대로로는 지역적인 에러 바운더리를 사용할 수 없습니다. 

`Page1` 컴포넌트와 이 Page1 컴포넌트를 감싸는 `LocalErrorBoundary1`이 있다고 해봅시다. 그리고 이 외부의 최상단에는 v1에서 구현한 `전역 에러 바운더리`, `업데이터인 queryClient`, `구독자인 ErrorCatcher`가 위치하고 있습니다. 이런 상황에서는 에러가 발생했을 경우 LocalErrorBoundary1로 에러가 던져지는 게 아니라 바로 업데이터, 구독자로 진입하게 됩니다. 탠스택 쿼리에서 throwOnError를 켜도 그렇습니다.

따라서 v1 그대로 둔다면 지역적인 에러 바운더리로 감싸도 에러 바운더리를 사용할 수 없습니다.

v1도 사용할 수 있도록 하면서 지역적인 에러 바운더리도 사용하기 위해선 실제 업데이터-구독자의 진입을 막으면 됩니다. 진입을 막고 throwOnError 옵션을 킨다면 에러는 에러 발생 컴포넌트로부터 상위 컨텍스트로 
자연스레 흐를 수 있게 됩니다. 

진입을 막는 방법은 업데이터 코드가 있는 queryClient에 분기 문을 추가해주는 것입니다.

![](https://velog.velcdn.com/images/pakxe/post/265dee73-fb74-4c94-afb6-ff5511685562/image.png)

v1 구현 코드에서 보았던 업데이터 코드입니다. query 실행 중 에러가 발생했다면 updateError가 호출되고 있습니다. 

updateError가 호출되기 전에 조건문을 추가해 `GET메서드에서 발생한 에러면서 fallback UI를 사용하겠다고 선언된 에러`라면 얼리 리턴하도록 합니다.
![](https://velog.velcdn.com/images/pakxe/post/f96bb3e8-3fb4-4fa2-ae03-4424be8e19ef/image.png)
이 콜백함수는 첫 번째 인자로 에러 객체가 주입되고 있기 때문에 실제 코드로는 아래처럼 조건문을 구현할 수 있을 것 같아요.
![](https://velog.velcdn.com/images/pakxe/post/62b82ca7-7d15-4f68-b2d9-c3b232a597e3/image.png)
이 코드는 에러 객체로 조건문을 걸고 있습니다. 따라서 에러 객체가 Toast UI를 사용하는 에러인지, Fallback UI를 사용하는 에러인지 정보를 담고 있어야 해요.

그래서 아래처럼 RequestGetError라는 커스텀 에러 객체를 제작했습니다.

![](https://velog.velcdn.com/images/pakxe/post/33f9b923-532b-4a16-9a64-a6023cea7467/image.png)
 
이 RequestGetError는 생성자의 인자로 errorDisplayMode를 넘겨 생성할 수 있습니다. errorDisplayMode 인자의 값으로 가능한 건 계속 말했듯 'toast'와 'fallback'입니다.

이렇게 구현된 RequestGetError 는 이 서비스의 모든 api가 거쳐 가는 곳인 request함수에서 생성되어 throw됩니다.

![](https://velog.velcdn.com/images/pakxe/post/472f0760-fc3b-4586-8049-4ee1e475af55/image.png)

## ▌ 결과 
### ▎ 1. GET 메서드 에러 시 Fallback UI
이제 GET 메서드에서 에러가 발생했을 경우 Fallback UI와 Toast UI 중 선택해서 띄울 수 있게 되었습니다. 

Fallback UI를 사용하는 경우 아래 코드처럼 사용합니다. 
![](https://velog.velcdn.com/images/pakxe/post/e7f60f2b-a164-435c-9c2b-e67de0aed03b/image.png)
에러가 발생할 수 있는 컴포넌트인 TestComponent를 지역적인 에러 바운더리로 감쌉니다. 그리고 api요청 훅에 errorDisplayMode 인자를 'fallback'값을 넘깁니다. 
![](https://velog.velcdn.com/images/pakxe/post/b382cfd9-6121-488f-8f59-af44cee8d967/image.gif)
Fallback UI가 잘 보입니다.

### ▎ 2. GET 메서드 에러 시 Toast UI
Toast UI를 사용하는 경우 아래 코드처럼 사용합니다.
![](https://velog.velcdn.com/images/pakxe/post/ec370bdf-2c62-4f08-af24-ca4b0f98bf93/image.png)
에러 바운더리로 감싸줄 필요 없고, errorDisplayMode인자의 값만 'toast'로 잘 넘겨주면 됩니다.

![](https://velog.velcdn.com/images/pakxe/post/afc7dfd3-a882-4b03-89f3-8737be523745/image.gif)

Toast UI가 잘 보입니다.

---
v2를 구현하는 건 v1보단 어렵지 않았는데요. 아마 책임을 잘 분리해두었기 때문에 빠르게 구현할 수 있던 것 같습니다.

v2로 오면서 지역적인 에러 바운더리를 사용할 수 있게 되었고, 같은 api여도 상황에 맞게 에러 UI를 선택할 수 있는 기능이 추가되었습니다. 

# ■ 마무리
긴 글 읽어주시느라 고생 많으셨습니다. 🙇

이렇게 행동대장 서비스에서 사용하고 있는 에러 핸들링 전략에 대해서 알아보았습니다. 에러는 개발 과정에서 피할 수 없는 존재이지만, 어떻게 대응하고 처리하느냐에 따라 사용자 경험과 서비스의 안정성에 큰 영향을 미칩니다. 이 글에서 다룬 사례와 전략들이 여러분의 프로젝트에 작은 도움이 되었기를 바랍니다.

글을 읽으며 이해가 어려웠던 부분이나 질문하고 싶은 내용이 있으시다면 <a href= "mailto:pigkill40@naver.com" >이메일</a>또는 댓글 남겨주세요.

감사합니다.

> 관련 [PR 링크](https://github.com/woowacourse-teams/2024-haeng-dong/pull/567). 글에서 사용되고 있는 용어와 실제 코드에서의 용어가 다르니 이 점 참고하시길 바랍니다.
