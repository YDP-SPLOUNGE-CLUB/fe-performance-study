## 애니메이션 최적화

책의 올림픽 통계 서비스에는 설문 결과 그래프가 존재한다. 설문 결과는 막대 그래프로 표시되어 있고, 하나의 항목을 클릭하면 해당 답변에 응답한 사람들에 대해서만 필터링하여 그래프를 다시 보여준다.

그러나 새로운 결과를 보여줄 때, 막대 길이가 애니메이션을 통해 변화하는데 어딘가 끊기는 느낌이 생긴다. 이러한 끊김 현상을 `쟁크(jank)`라고 한다. 이 부분을 최적화해볼 예정이다.

```css
const BarGraph = styled.div`
  position: absolute;
  left: 0;
  top: 0;
  width: ${({ width }) => width}%;
  transition: width 1.5s ease;
  height: 100%;
  background: ${({ isSelected }) =>
    isSelected ? "rgba(126, 198, 81, 0.7)" : "rgb(198, 198, 198)"};
  z-index: 1;
`;
```

percent prop에 따라 막대 그래프의 가로 길이를 조절하는 컴포넌트의 스타일이다.

percent가 바뀌면, width 값이 바뀌면서 transition 속성에 의해 애니메이션이 일어난다. 여기서 **쟁크 현상이 발생하는 이유는 무엇일까?**

이를 알기 위해, 브라우저에서 애니메이션이 어떻게 동작하는지와, 브라우저가 어떤 과정을 거쳐 화면을 그리는지 이해해야 한다.

<br/>

### 애니메이션의 원리

애니메이션의 원리는 여러 장의 이미지를 빠르게 전환하여 우리 눈에 잔상을 남기고, 그로 인해 이미지가 움직이는 것처럼 느껴지게 하는 것이다.

다양한 주사율의 모니터가 있지만, 일반적으로 사용하는 디스플레이의 주사율은 60Hz이다. 즉, 1초에 60장의 정지된 화면을 빠르게 보여준다. 브라우저도 이에 맞춰서 최대 60FPS(Frames Per Second)로 1초에 60장의 화면을 새로 그린다.

**고로, 쟁크 현상이 발생한 이유는 브라우저가 정상적으로 60FPS로 화면을 그리지 못했기 때문이다.**

**브라우저는 왜 초당 60프레임을 그리지 못했을까?** 이를 알기 위해 브라우저가 화면을 그리는 과정을 알아야 한다.

<br/>

### 브라우저 렌더링 과정

(...)

<br/>

### 애니메이션 최적화

문제의 원인과 방법을 알았으니, width로 되어 있는 애니메이션을 transform 최적화해보자.

transform 속성에는 다양한 값이 있다. 위치를 이동시키는 translate, 크기를 변경하는 scale, 요소를 회전시키는 rotate가 대표적이다. 이 서비스에서는 scale을 활용하였다.

```css
const BarGraph = styled.div`
  /* 생략 */
  width: 100%;
  transform: scaleX(${({width}) => width / 100});
  transform-origin: center left;
  transition: width 1.5s ease;
  /* 생략 */
`;
```

미리 막대의 너비를 100%로 채워두고, scale을 이용하여 비율에 따라 줄이는 방식 사용했다.

이때 scaleX 안에 있는 width가 퍼센트이기 때문에 scaleX 함수의 인자로 쓰일 수 있도록 1 이하의 실수 값으로 변경해주었다.

기본적인 scale의 기준점이 중앙에 있기 때문에, 중앙을 중심으로 정렬해주는 코드도 추가했다.

이를 통해, 최적화 후 렌더링 작업이 확실히 여유로워졌다. **레이아웃과 페인트 작업이 생략되었기 때문이다.**

<br/>

## 컴포넌트 지연로딩

책의 서비스에서는 이미지 모달을 띄우기 위해, react-image-gallery라는 외부 라이브러리를 사용하여 이미지 데이터를 라이브러리에 넘겨 화면에 표시했다.

외부 라이브러리를 사용한다는 건 해당 라이브러리의 사이즈만큼 최종 번들링된 자바스크립트의 사이즈도 커진다는 것을 의미하고, 이는 곧 서비스의 자바스크립트를 로드하는 데 시간이 오래 걸린다는 뜻이다.

`cra-bundle-ananlyzer`를 통해 번들 분석 결과를 확인해보면, 서비스의 첫 화면부터 라이브러리가 필요하지 않음에도 자리를 차지하고 있음을 알 수 있다.

용량이 크지는 않지만, 코드를 분할하여 지연로딩을 적용해보자.

```jsx
import React, { useState } from "react";
// import ImageModal from './components/ImageModal'

const LazyImageModal = lazy(() => import("./components/ImageModal"));

function App() {
  const [showModal, setShowModal] = useState(false);

  return (
    <div className="App">
      ...
      <Suspense fallback={null}>
        {showModal ? (
          <LazyImageModal
            closeModal={() => {
              setShowModal(false);
            }}
          />
        ) : null}
      </Suspense>
    </div>
  );
}
```

lazy를 통해, 정적으로 import되어서 번들 파일에 포함되었던 ImageModal 컴포넌트와 그 안의 라이브러리까지 청크 파일에서 분리되었다.

ImageModal이 로드 되기 전에 발생하는 에러를 방지하기 위해, Suspense 컴포넌트로 감싸주기도 했다.

이렇게 하면, 처음 ImageModal 컴포넌트가 완전히 로드되지 않은 상태에서는 fallback에 넣어 준 null로 렌더링되고, 로드가 완료되면 제대로 된 모달이 렌더링될 것이다.

<br/>

## 컴포넌트 사전로딩

위에서 설명한 지연 로딩을 적용하면, 최초 페이지를 로드할 때 당장 필요없는 모달과 코드가 번들에 포함되지 않아, 로드할 파일의 크기가 작아지고 초기 로딩 속도나 자바스크립트의 실행 타이밍이 빨라져서 화면에 표시된다는 장점이 있다.

하지만, 초기 화면 로딩 시에는 효과적일지 몰라도 모달을 띄우는 시점에는 한계가 있다. 모달 코드를 분리했기 때문에 모달을 띄울 때, 코드를 새로 로드해야 하며 로드가 완료되어야 모달이 뜬다. 즉, 모달이 뜨기 전까지 약간의 지연이 발생할 수 있다.

이러한 문제를 해결하기 위해 `사전 로딩(Preloading) 기법`을 이용한다. **사전 로딩은 말그대로 나중에 필요한 모듈을 필요해지기 전에 미리 로드하는 기법을 말한다.**

<br/>

### 사전로딩 타이밍

사용자가 모달을 열기 위한 버튼을 언제 클릭할까?

이를 모르니, 모달 코드를 언제 로드해 둘지 정하기가 애매하다. 그래서 두 가지 타이밍을 고려해볼 수 있다.

하나는 사용자가 버튼 위에 마우스를 올려놨을 때(mouseenter)이고, 다른 하나는 최초에 페이지가 로드되고 모든 컴포넌트의 마운트가 끝났을 때이다.

<br/>

**1
) 버튼 위에 마우스를 올려놓았을 때 사전 로딩**

리액트에서 마우스가 버튼에 올라왔는지 아닌지는 Button 컴포넌트의 onMouseEnter 이벤트를 통해 알 수 있다.

```jsx
function App() {
    const [showModal, setShowModal] = useState(false)

    const handleMouseEnter = () => {
        const component = import('./component/ImageModal')
    }

    return (
        <div className="App">
            ... 생략 ...
            <ButtonModal
                onClick={() => {
                    setShowModal(true)
                }}
                onMouseEnter={handleMouseEnter}>
            올림픽 사진 보기
            </ButtonModal>
            <... 생략 ...
        </div>
    )
}

```

원래는 버튼 클릭 후 모달을 띄우려고 할 때 로드 되어야 하는데, onMouseEnter 이벤트에서 미리 코드를 로드했기 때문에 커서를 올려놓기만 해도 모달 코드가 로드된다. (Network 패널 확인)

커서를 올리고, 버튼을 클릭하기까지 대략 300~600밀리초 정도의 시간차가 있다. 아주 찰나의 시간이지만, 브라우저가 새로운 파일을 로드하기에는 충분하다.

<br/>

**2
) 컴포넌트의 마운트 완료 후 사전 로딩**

만약 모달 컴포넌트 크기가 커서 로드하는데 1초 또는 그 이상의 시간이 필요할 수도 있다.

이런 경우, 모든 컴포넌트의 마운트가 완료된 후 브라우저에 여유가 생겼을 때 뒤이어 모달을 추가 로드하는 방법이 있다.

클래스형 컴포넌트라면 componentDidMount 시점이고, 함수형 컴포넌트에서는 useEffect 시점이라고 할 수 있다.

```jsx
function App() {
  const [showModal, setShowModal] = useState(false);

  useEffect(() => {
    const component = import("./component/ImageModal");
  }, []);

  return <div className="App">... 생략 ...</div>;
}
```

마찬가지로 네트워크 패널을 확인해보면, 초기 페이지 로드에 필요한 파일(0.chunk.js, bundle.js)을 우선 다운로드하고 페이지 로드가 완료된 후에야 모달 코드를 다운하는 것을 볼 수 있다.

<br/>

## 이미지 사전로딩

컴포넌트는 import 함수를 이용하여 로드되지만, 이미지는 이미지가 화면에 그려지는 시점, 즉 HTML 또는 CSS에서 이미지를 사용하는 시점에 로드된다.

하지만 이런 경우 외에 자바스크립트로 이미지를 직접 로드하는 방법이 한 가지가 있다. 바로 자바스크립트의 Image 객체를 사용하는 방법이다.

```jsx
const img = new Image();
img.src = "{이미지 주소}";
```

위의 주제에 이어서, 코드를 모달이 사전로드하는 타이밍인 useEffect에 넣어주면

모달 코드와 함께 이미지가 다운된다. 즉, 나중에 모달 위에 표시될 대표 이미지를 미리 다운로드한 것이다. (아래 코드 확인)

```jsx
useEffect(() => {
  const component = import("./component/ImageModal");

  const img = new Image();
  img.src =
    "https://github.com/performance-lecture/lecture-2/blob/1e75da192571be7764a97a86be7aabe9e137bed9/src/assets/rio-2016.jpg";
}, []);
```

<br/>

❗❗ 주의해야 할 점 ❗❗

테스트를 할 때, 'Disable cache' 옵션을 체크 해제해야 한다.

이미지 사전 로딩이 가능한 이유는 이미지를 로드할 때 브라우저가 해당 이미지를 캐싱해 두기 때문이다.

그러나 'Disable cache' 옵션이 체크되어 있으면, 이미지 리소스에 대해 캐시를 하지 않아 매번 새로 불러온다.

하지만~ 캐시 활성화 시, 다른 리소스도 캐시를 사용하기 때문에 정확한 분석이 어려워질 수도 있다.^^ 따라서, 새로고침 할 때, `캐시 비우기 및 강력 새로고침`도 같이 하자.

<br/>

📚 **고민해보자**

몇 장의 이미지까지 사전 로드해둘 것인지 고민해볼 필요가 있다.

대표 이미지뿐만 아니라 하단 섬네일 이미지까지 사전로딩할 수도 있지만,

그럴 경우 페이지가 로드될 때, 즉 사전 로딩을 하는 순간 브라우저의 리소스를 그만큼 많이 사용하기 때문에 다른 성능 문제를 야기할 수도 있다.
