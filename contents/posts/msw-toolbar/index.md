---
title: "API 요청 시뮬레이션을 간편하게: 그로잉 팀의 MSW Toolbar 도입기"
description: "프론트엔드에서 네트워크의 응답 시간을 조절하거나 특정 API 요청에서 에러 상태를 유발하는 등의 상황을 테스트하는 건 번거로운 일이에요. 이를 해결하기 위한 그로잉 팀의 MSW Toolbar 제작기를 소개할게요."
date: 2024-02-25
update: 2024-02-25
tags:
  - growing
  - msw
  - mocking
series: "MSW Toolbar로 다양한 시나리오 테스트하기"
---

## 툴바를 만들자!

저희 그로잉팀은 런타임에서 여러 기능들의 성공과 실패의 조합에 따라 테스트를 할 수 있는 방식이 필요했고, 이를 MSW Toolbar를 만들어 해결했어요.

이번 글에서는 MSW Toolbar 제작 과정을 소개할게요.

<div align='center'>
    <img src='msw.gif' width='300px'  />
</div>

### 정의된 핸들러 리스트 받아오기

툴바 구현을 위해 먼저 정의해 둔 핸들러 리스트 정보들을 가져올 수 있는 지가 가장 중요했어요. 따라서 해당 정보를 받아올 수 있는지 확인하는 작업을 먼저 진행했어요.

```tsx
// App.ts
import { handlers } from "mocks/handlers"

// 각 핸들러의 경로와 메소드 정보를 콘솔에 출력
handlers.forEach(handler => {
  const { path, method } = handler.info // 핸들러의 정보에서 경로와 메소드 추출
  console.log({ path, method })
})
```

다행히도 MSW 핸들러에서 제공하는 정보를 활용하여, 간단한 코드 몇 줄로 원하는 핸들러 정보를 추출할 수 있었어요.

![](handler-list-image.png)
이렇게 콘솔에 출력된 정보를 바탕으로, 핸들러 정보를 관리하는 클래스나 객체를 구성하여 필요할 때마다 활용하면 될 것 같네요!

### UI

툴바의 UI는 피그마를 사용하여 디자인했습니다. 툴바에 포함될 주요 기능은 다음과 같아요.

1. **API 경로 검색**: 사용자가 특정 API 경로를 검색하여 해당하는 핸들러를 찾을 수 있습니다.
2. **상태 코드 변경**: 성공 또는 에러 상태 코드를 동적으로 변경할 수 있어, 다양한 응답 시나리오를 시뮬레이션할 수 있습니다.
3. **응답 지연 설정**: API 응답의 지연 시간을 설정하여 네트워크 지연 테스트를 할 수 있습니다.

![](ui-image.png)

### **핸들러 동적 변경 고려사항**

상태 코드나 응답 지연 시간을 변경하려면, 기존의 핸들러 코드를 동적으로 조절할 필요가 있었어요. 예를 들어, 성공 응답과 에러 응답을 다루는 두 가지 핸들러가 있다고 가정해 보겠습니다.

```tsx
// mocks/user/getUserHandler.ts
type Params = {
  userId: string;
};

const data: UserDto = {
	id: '1',
	nickName: '곰곰곰',
	birthDay: '2000-01-01',
	anniversaryDay: '2023-12-16',
};

// 성공 응답 핸들러
export const getUserSuccessHandler = () => {
	http.get<Params, null, UserDto, '/user/:userId'>(
    '/user/:userId',
    async () => {
			await delay(1000); // 1초 지연

      return HttpResponse.json(data, {
	      status: 200,
	      statusText: '요청에 상공했습니다.',
		});
  });
}

// 실패 응답 핸들러
export const getUserErrorHandler = () => {
	http.get<Params, null, null, '/user/:userId'>(
    '/user/:userId',
    async () => {
			await delay(500); // 0.5초 지연

      return HttpResponse.json(null, {
	      status: 400,
	      statusText: '요청에 실패했습니다.',
    }),
  });
}
```

세 부분에서 변경이 일어나는 것을 알 수 있어요.

1. **응답 타입**: 성공과 실패 응답의 타입이 다를 수 있습니다.
2. **지연 시간**: `delay` 함수의 인자를 통해 응답 지연 시간을 설정합니다.
3. **응답 데이터**: `HttpResponse.json` 메소드의 인자로 전달되는 데이터입니다.

따라서 이 요소들을 런타임에 동적으로 바꿔줄 수 있어야 했어요. 이를 하드코딩이 아닌 변수로 지정해두고 동적으로 바꿔주면 될 것 같아요.

### 해당하는 URL에 맞는 핸들러 찾는 법

그렇다면 검색한 API URL에 맞는 핸들러는 어떻게 찾을 수 있을까요?

같은 URL에 대해 다른 HTTP 메소드(GET, POST, DELETE 등)를 사용하는 경우가 많기 때문에, 단순히 URL만으로 핸들러를 찾는 것은 충분하지 않았아요.

이를 해결하기 위해 핸들러 정보를 저장할 때 URL 경로와 HTTP 메소드의 조합을 고유한 식별자로 사용하기로 결정했어요.

```tsx
interface HandlerInfoList {
  [path: string]: {
    [method: string]: HandlerInfo
  }
}
```

![](url-search-image.png)

### HandlerInfoManager 구현과 활용

핸들러들을 저장하고 꺼내오고 delay 변수값들을 변경시켜주는 작업을 저희 그로잉 팀은 HandlerInfoManager 클래스를 제작하여 해결하였어요.

```tsx
import { handlers } from "mocks/handlers"

const DEFAULT_STATUS = 200
const DEFAULT_DELAY = 1000

type StatusType = 200 | 400

interface HandlerInfo {
  status: StatusType
  delayTime: number
}

interface HandlerInfoList {
  [path: string]: {
    [method: string]: HandlerInfo
  }
}

interface HandlerInfoParams {
  path: string
  method: string
  code: StatusType
  time: number
}

class HandlerInfoManager {
  private handlerInfos: HandlerInfoList = {}

  // 핸들러 정보 초기화
  public initHandlerInfo(): void {
    handlers.forEach(handler => {
      const { path, method } = handler.info
      this.setHandlerInfo({
        path: path.toString(),
        method: method.toString(),
        code: 200, // 기본 상태 코드
        time: 1000, // 기본 지연 시간
      })
    })
  }

  // 핸들러 정보 설정
  public setHandlerInfo({ path, method, code, time }: HandlerInfoParams): void {
    if (!this.handlerInfos[path]) {
      this.handlerInfos[path] = {}
    }
    this.handlerInfos[path][method] = { status: code, delayTime: time }
  }

  // 핸들러 리스트 조회
  public getHandlerInfos(): HandlerInfoList {
    return this.handlerInfos
  }

  // 특정 핸들러 정보 조회
  public getHandlerInfo(path: string, method: string): HandlerInfo | undefined {
    return this.handlerInfos[path]?.[method]
  }
}

export const handlerInfoManager = new HandlerInfoManager()
```

이렇게 `HandlerInfoManager`를 사용하면 핸들러의 상태 코드와 지연 시간을 런타임에 변경할 수 있으며, 툴바 UI에서 사용자의 입력에 따라 이러한 변경을 적용할 수 있습니다.

### 사용 예시

사용 예시는 다음과 같아요.

```tsx
import { handlerInfoManager } from "mocks/HandlerInfoManager"
import { CoupleDto } from "models/couple"
import { delay, http, HttpResponse } from "msw"

interface Params {
  coupleId: string
}

const data: CoupleDto = {
  coupleId: "1",
  myName: "연주",
  partnerName: "민지",
  dayCount: 10,
  petId: null,
}

export const getCoupleHandler = http.get<Params, {}, CoupleDto | null>(
  "/couples/:coupleId",
  async () => {
    const responseList = {
      200: HttpResponse.json(data, {
        status: 200,
        statusText: "success",
      }),
      400: HttpResponse.json(null, {
        status: 400,
        statusText: "fail",
      }),
    }

    const handler = handlerInfoManager.getHandlerInfo(
      "/couples/:coupleId",
      "GET"
    )

    const status = handler?.status || 200
    const delayTime = handler?.delayTime || 0

    await delay(delayTime)
    return responseList[status]
  }
)
```

```tsx
const handleClick = (path: string, method: string) => {
  handlerInfoManager.setHandlerInfo({ path, method, code: 400, time: 3000 })
}
```

### 추상화를 해보자

모든 API 핸들러에서 반복적으로 나타나는 패턴들은 무엇이 있을까요?

코드를 분석해 본 결과 다음 코드가 반복해서 작성되어야 했어요. 이를 추상화하여 코드 중복을 줄여볼게요.

```tsx
const handler = handlerInfoManager.getHandlerInfo("/couples/:coupleId", "GET")

const status = handler?.status || 200
const delayTime = handler?.delayTime || 0

await delay(delayTime)
return responseList[status]
```

```tsx
import { handlerInfoManager } from "mocks/HandlerInfoManager"
import { delay, http } from "msw"

// 상태 코드별 응답 데이터를 매핑하는 객체 타입을 정의합니다.
type ResponseData<TResponse> = {
  [key: number]: TResponse | null
}

// API 요청 처리 함수 타입을 정의합니다.
type RequestHandler<TParams, TResponse> = (
  params: TParams
) => ResponseData<TResponse>

// 범용 API 핸들러 생성 함수를 정의합니다.
export const createApiHandler = <TParams, TResponse>(
  path: string,
  method: keyof typeof http,
  handleRequest: RequestHandler<TParams, TResponse>
) => {
  return http[method](path, async req => {
    const params = req.params as TParams
    const handler = handlerInfoManager.getHandlerInfo(path, method)
    const delayTime = handler?.delayTime || 0
    const responseStatus = handler?.status || 200

    await delay(delayTime)

    const responseData = handleRequest(params)
    return responseData[responseStatus]
  })
}
```

이 추상화를 통해 아래처럼 간편하게 개별 핸들러들을 구현할 수 있게 되었어요. 각 API 핸들러마다 상태 코드와 지연 시간을 처리하는 로직을 반복해서 작성할 필요가 없어진거죠.

```tsx
// 사용 예시: 커플 정보 조회 API 핸들러
interface Params {
  coupleId: string
}

const data: CoupleDto = {
  coupleId: "1",
  myName: "연주",
  partnerName: "민지",
  dayCount: 10,
  petId: "1",
}

export const getCoupleHandler = createApiHandler<Params, CoupleDto | null>(
  "/couples/:coupleId",
  "get",
  () => ({
    // 여기서는 단순화를 위해 바로 데이터를 반환하지만,
    // 실제로는 params를 기반으로 데이터를 조회하거나 처리할 수 있습니다.
    200: data, // 성공 응답
    400: null, // 실패 응답
  })
)
```

## 그러나,,

처음에 생각한 기능들 외에 추가로 필요한 기능들이 있었어요. 처음에는 Path Parameter만 필요했었지만 Query Parameter가 필요한 경우도 있었어요.

> Parameter type
>
> 1. Query parameters: `?a=1&b=2`;
> 2. Path parameters: `GET /user/:id`, where `id` is a path parameter.

그리고 API 호출 성공 후에 로직을 처리할 콜백 함수가 필요했어요. DELETE 요청 후에 기존 data를 삭제하거나, PATCH 후에 수정하는 코드가 필요했기 때문이에요.

이러한 추가 요구사항을 충족하기 위해 다음과 같이 `createApiHandler` 함수를 확장했어요.

```tsx
// ApiHandlerCreator.ts
import { DefaultBodyType, PathParams, delay, http, HttpResponse, StrictRequest } from 'msw';
import { handlerInfoManager } from './HandlerInfoManager';

// 응답 데이터 타입 정의: 상태 코드별로 응답 데이터를 매핑
type ResponseData<TResponse> = {
  [status: number]: NullableResponse<TResponse>;
};

// 요청 처리 핸들러 타입 정의: 경로 및 요청 매개변수를 받아 응답 데이터를 반환
type RequestHandler<TParams, TRequest, TResponse> = (
  params: TParams,
  request: TRequest
) => ResponseData<TResponse>;

// 요청 처리 후 실행할 콜백 함수 타입 정의
type AfterRequest<TParams, TRequest> = (
  params: TParams,
  request: TRequest
) => void;

// 범용 API 핸들러 생성 함수
export const createApiHandler = <
  TParams extends PathParams,
  TRequest extends DefaultBodyType,
  TResponse extends DefaultBodyType
>(
  path: string;
  method: keyof typeof http;
  requestHandler: RequestHandler<TParams, StrictRequest<TRequest>, TResponse>;
  onSuccess?: AfterRequest<TParams, StrictRequest<TRequest>>;
  ) => {
  return http[method]<TParams, TRequest, TResponse>(
    path,
    async ({ params, request }) => {
      // 요청 처리 함수를 호출하여 응답 데이터를 가져옴
      const responseData = requestHandler(params, request);

      // 핸들러 정보(응답 지연 시간 및 상태 코드) 조회
      const handlerInfo = handlerInfoManager.getHandlerInfo(path, method);
      const delayTime = handlerInfo?.delayTime || 0;
      const responseStatus = handlerInfo?.status || 200;

      // 응답 지연 시간 적용
      await delay(delayTime);

      // 성공적인 요청 처리 후 콜백 함수 실행
      if (responseStatus !== 400) {
        onSuccess?.(params, request);
      }

      // JSON 형태로 응답 반환
      return HttpResponse.json(responseData[responseStatus], {
        status: responseStatus,
      });
    }
  );
};
```

### 리팩토링: 파라미터 객체화

함수의 매개변수가 많아짐에 따라 가독성과 사용성 문제가 발생했어요. 몇번째 파라미터에 어떤 값을 넣어줘야 할지 시각적으로 알아보기 어려웠기 때문이에요.

이를 해결하기 위해 "객체 비구조화 할당(destructuring)" 기법을 적용하여 함수 매개변수를 객체화했습니다. 이 접근 방식을 통해 함수 호출 시 매개변수의 순서에 덜 의존할 수 있게 되었고, 코드의 가독성을 향상시킬 수 있었어요.

```tsx
// 변경전
export const deleteOurChatHandler = createApiHandler<
  ChatDeleteParams,
  {},
  null
>(
  "/couples/:coupleId/chattings/:chattingId/delete-ours",
  "delete",
  () => ({
    200: null,
    400: null,
  }),
  ({ chattingId }) => {
    chatData = chatData.filter(chat => chat.parentChatting.id !== chattingId)
  }
)
```

```tsx
// 변경후
export const deleteOurChatHandler = createApiHandler<
  ChatDeleteParams,
  {},
  null
>({
  path: "/couples/:coupleId/chattings/:chattingId/delete-ours",
  method: "delete",
  requestHandler: () => ({
    200: null,
    400: null,
  }),
  onSuccess: ({ chattingId }) => {
    chatData = chatData.filter(chat => chat.parentChatting.id !== chattingId)
  },
})
```

### MSW 툴바 컴포넌트 제작하기

이제 툴바 컴포넌트를 제작해 볼게요.

<img src='selector-image.png' width='300px' />

```tsx
//MSWToolbar.tsx
import ...

function MSWToolbar() {
  const queryClient = useQueryClient();
  const [open, setOpen] = useState(false)
  const [items, setItems] = useState(
    Object.entries(handlerInfoManager.getHandlerInfos())
  )
  const stagedValue = useRef<{
    [path: string]: {
      [method: string]: HandlerInfo
    }
  }>({})


  const inputChangeHandler = (e: React.ChangeEvent<HTMLInputElement>) => {
    const { value } = e.target
    setItems(
      Object.entries(handlerInfoManager.getHandlerInfos()).filter(([path]) =>
        path.includes(value)
      )
    )
  }

  const clickApplyBtnHandler = () => {
    Object.keys(stagedValue.current).forEach(path => {
      Object.keys(stagedValue.current[path]).forEach(method => {
        const { status, delayTime } = stagedValue.current[path][method]
        handlerInfoManager.setHandlerInfo({
          path,
          method,
          code: status,
          time: delayTime,
        })
      })
    })
    stagedValue.current = {};
    queryClient.invalidateQueries();
    setOpen(false)
  }

  return (
    <>
      {!open && (
        <S.ToolbarButton onClick={() => setOpen(true)}>MSW</S.ToolbarButton>
      )}
      {open && (
        <ToolbarBottomSheet open={open} setOpen={setOpen}>
          <S.SearchBar>
            <Icon icon="IconSearch" />
            <S.Input onChange={inputChangeHandler} />
          </S.SearchBar>
          {items.length === 0 && (
            <S.MessageBox>검색 결과가 없어요!</S.MessageBox>
          )}
          {items.length > 0 && (
            <S.ItemsContainer className="hidden-scrollbar">
              {items.flatMap(([path, methods]) =>
                Object.entries(methods).map(
                  ([method, { delayTime, status }]) => (
                    <ToolbarItem
                      key={`${path}-${method}`}
                      method={method}
                      path={path}
                      delayTime={delayTime}
                      status={status}
                      onChange={(time, code) => {
                        stagedValue.current[path] =
                          stagedValue.current[path] ?? {}
                        stagedValue.current[path][method] = {
                          status: code,
                          delayTime: time,
                        }
                      }}
                    />
                  )
                )
              )}
            </S.ItemsContainer>
          )}
          {items.length > 0 && (
            <S.ButtonArea>
              <S.Button onClick={clickApplyBtnHandler}>적용하기</S.Button>
            </S.ButtonArea>
          )}
        </ToolbarBottomSheet>
      )}
    </>
  )
}

export default MSWToolbar;
```

툴바 컴포넌트의 전체 코드와 툴바의 옵션 선택 부분입니다. 코드의 동작 과정을 자세히 살펴볼게요.

- `handlerInfoManager.getHandlerInfos()` 를 사용해 handler의 path와 method를 받아오고 있어요.
- `inputChangeHandler` 는 검색 창의 input value가 변경되었을 때 호출되는 함수에요. 텍스트를 입력할 때마다 입력된 경로가 포함되는 요소들을 보여줍니다.
- 각 핸들러에 대응되는 드롭다운 메뉴에서 옵션을 선택하면, 옵션을 stagedValue 변수에 저장합니다.
- 적용하기 버튼을 누르면, `clickApplyBtnHandler` 가 실행되어 stagedValue에 저장된 옵션들(delay와 status code)이 실제 핸들러에 적용됩니다. 현재 그로잉 프로젝트에서는 react-query를 사용하고 있어서 옵션들을 적용한 후 바로 데이터를 refetch하도록 만들기 위해 마지막으로 `queryClient.invalidateQueries()` 를 실행해주고 있어요.
- real은 현실적인 응답 시간 정도를 지연해주는 옵션입니다. infinite는 응답을 무한하게 delay 시켜주는 옵션입니다. 이는 로딩 상태가 잘 표시되는 지 등을 확인할 때 유용하게 사용할 수 있어요.

#### 코드에서 한 가지 생각해볼 점이 있었어요.

`const stagedValue = useRef<{[key: string]: SetHandlerParams;}>({});` 부분에서 왜 useRef 훅을 사용했을까요?

stagedValue를 useRef없이 사용하면, 검색을 할 때마다 MSWToolbar 컴포넌트가 재실행 되어 stagedValue가 빈 객체로 초기화됩니다. 이를 방지하고자 useState 훅을 사용하면, stagedValue가 변경 될 때마다 리렌더링이 발생해요.

그래서 컴포넌트가 재실행 되더라도 stagedValue값을 유지하면서, stagedValue가 변경되더라도 리렌더링이 일어나지 않게 하기 위해 useRef hook을 사용했어요.

### 툴바는 리액트 돔 트리에서 어디에 위치해야 할까?

```tsx
const root = ReactDOM.createRoot(document.getElementById("root") as HTMLElement)

enableMocking().then(() => {
  root.render(
    <React.StrictMode>
      <BrowserRouter>
        <QueryClientProvider client={queryClient}>
          <ReactQueryDevtools initialIsOpen={false} />
          <GlobalStyle />
          <ThemeProvider theme={myTheme}>
            {process.env.NODE_ENV === "development" && <MSWToolbar />}
            <AsyncBoundary
              pendingFallback={<FullScreenLoading />}
              rejectedFallback={({ error, resetErrorBoundary }) => (
                <FullScreenError
                  error={error}
                  resetErrorBoundary={resetErrorBoundary}
                />
              )}
            >
              <App />
            </AsyncBoundary>
          </ThemeProvider>
        </QueryClientProvider>
      </BrowserRouter>
    </React.StrictMode>
  )
})
```

**다음은 MSWToolbar 컴포넌트의 위치를 선택할 때 고려한 내용들이에요.**

1. MSWToolbar는 dev모드에서만 뜨도록 해야 합니다.
2. MSWToolbar는 런타임에 영향을 주지 않아야 합니다.
3. MSWToolbar에서 styled-component의 theme을 이용하고 있어 ThemeProvider 안에 들어가야 합니다.
4. react query로 관리되고 있는 서버 데이터의 invalidate해줄 수 있어야 하므로 QueryClientProvider 안에 들어가야 합니다.

1번을 위해 NODE_ENV에 따라 Toolbar를 표시해 주었고 2, 3번을 고려해 AsyncBoundary밖이면서 ThemeProvider와 QueryClientProvider의 안에 위치 시켰어요.

MSWToolbar를 이용해 의도적으로 error 상황을 테스트할 때, 그로잉 프로젝트에서 AsyncBoundary와 함께 사용하고 있는 FullScreenError가 뜰 수 있어요. 이때, MSWToolbar는 런타임에서 발생하는 에러 범위 밖에서 상태코드를 200으로 바꿔줄 수 있어야 하기 때문에 AsyncBoundary밖에 위치시켜두었어요.

## 결론

이 과정을 통해 MSW와 MSW를 활용한 Toolbar 도입에 성공할 수 있었어요.

MSW를 통한 API 모킹은 API 경로를 기반으로 이루어지므로, 각 요청을 명확하게 식별하고 관리할 수 있었어요.

MSW Toolbar는 네트워크 요청의 시뮬레이션 과정을 간소화해주는데 큰 도움을 주었어요. 기존에는 네트워크 상태를 모니터링하고 조정하기 위해 크롬의 개발자 도구에 의존했지만, 이제는 Toolbar를 통해 직접적으로 응답 시간을 조절하거나 특정 API 요청에서 에러 상태를 유발하는 등의 작업을 쉽게 처리할 수 있게 되었어요. 이에 따라 다양한 네트워크 환경과 에러 상황을 빠르게 시뮬레이션하고 대응할 수 있을 것으로 기대하고 있어요.

이를 통해 개발 과정에서의 시행착오를 줄이고, 개발 효율성을 높여가며, 최종적으로 사용자에게 더 나은 서비스를 제공하는 그로잉 팀이 되어볼게요. 감사합니다 :)
