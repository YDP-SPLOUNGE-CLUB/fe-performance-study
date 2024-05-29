# 실습 내용

이미지 갤러리 서비스를 실습한다.

# 이 장에서 학습할 최적화 기법

- 이미지 지연 로딩
- 레이아웃 이동 피하기
- 리덕스 렌더링 최적화
- 병목 코드 최적화

# 분석 툴 소개

React Developer Tools(Profiler)
- Profiler 패널
- Component 패널

# 서비스 탐색

이미지 하나를 클릭하면 화면 위로 이미지가 뜬다.
눈에 띄는 것은 이미지뒤의 배경 색이 이미지의 전체적인 생상과 비슷하게 맞춰진다
- 성능 측면에서 살펴보았을 때 이미지가 늦게 뜨는 것과 이미지가 뜨고 한참 뒤에 배경 색이 변한다.

# 레이아웃 이동 피하기

레이아웃 이동(Layout Shift)이란 화면상의 요소 변화로 레이아웃이 갑자기 밀리는 현상을 말한다.
특히 이미지 로딩 과정에서 레이아웃 이동이 많이 발생한다.
이런 레이아웃 이동은 사용자 경험에 좋지 않은 영향을 준다.
따라서 이미지 갤러리 서비스에서 발생하는 레이아웃 이동을 분석하고 해결해본다.

이런 레이아웃 이동은 사용자의 주의를 산만하게 만들고 위치를 순간적으로 변경시키면서 의도와 다른 클릭을 유발할 수 있다.

Lighthouse에서 레이아웃 이동이 얼마나 발생하는지를 나타내는 지표
**CLS(Cumulative Layout Shift)라는 항목을 두고 성능 점수에 포함되어 있다.**

- Lighthouse
    - CLS 는 0부터 1까지의 값을 가진다.
    - 레이아웃 이동이 전혀 발생하지 않은 상태 0
    - 레이아웃 이동이 발생한 상태 1
    - 권장되는 점수는 0.1 이하이다.
    - 현 서비스 0.438
- Performance 패널
    - 검사 결과의 Experience 섹션을 살펴보면 Layout Shift 라는 이름의 빨간 막대가 표시된다.
    - 해당 시간에 레이아웃 이동이 발생하였다는 의미

### 레이아웃 이동의 원인

원인은 다양하며 그중 가장 흔한 경우는 다음과 같다.

- 사이즈가 미리 정의되지 않은 이미지 요소
- 사이즈가 미리 정의되지 않은 광고 요소
- 동적으로 삽입된 콘텐츠
- 웹 폰트 (FOIT, FOUT)

여기서는 '사이즈가 미리 정의되지 않은 이미지 요소' 때문에 레이아웃 이동이 발생했다.

### 레이아웃 이동 해결

사실 답은 나와있다. 요소의 사이즈를 지정하면 된다.
그러나 브라우저의 가로 사이즈에 따라 달라진다. 너비와 높이를 고정하는 것이 아니라 비율로 공간을 잡아두면된다.

여기서는 16:9 비율로 잡는다. 비율로 설정하는 방법은 크게 두가지가 있다.

#### padding을 이용하여 박스를 만든 뒤 그 안에 이미지를 absolute로 띄우는 방식

```jsx
<div class="wrapper">
	<img class="image" src="..." />
</div>
```

```css
.wrapper {
	position: relative;
	width: 160px;
	padding-top: 56.25%; /* 16:9 */
}
.image {
	position: absolute;
	width: 100%;
	height: 100%;
	top: 0;
	left: 0;
}
```

padding을 이용하여 비율을 맞추긴 하였지만 퍼센트를 매번 계산해야하고 코드가 직관적이지 않다.

#### aspect-ratio CSS 속성을 이용하는 방법

```css
.wrapper {
	width: 100%;
	aspect-ratio: 16 / 9;
}
.image {
	width: 100%;
	height: 100%;
}
```

단 브라우저 호환을 잘 체크한후 적용하도록 하자.

여기서는 padding을 이용한 방법을 사용한다.

### 최적화

```jsx
// PhotoItem.js

// before
const ImageWrap = styled.div``;  
  
const Image = styled.img`  
  cursor: pointer;
  width: 100%;
`;

// after
const ImageWrap = styled.div`  
  width: 100%;
  padding-bottom: 56.25%;
  position: relative;
`;  
  
const Image = styled.img`  
  cursor: pointer;
  width: 100%;
  height: 100%;
  position: absolute;
  top: 0;
  left: 0;
`;
```

적용후 새로고침해보면 고정적인 이미지가 렌더링되는 것을 볼 수 있다.
- Lighthouse의 CLS 도 0으로 변경되었다.

# 이미지 지연 로딩

이전 장에서 Intersection Observer API를 이용했다면 이번에는
npm에 등록되어 있는 이미지 지연 로딩 라이브러리를 이용하여 지연 로딩을 적용해본다.

`react-lazyload` 라는 라이브러리를 사용한다.
사용 방법은 간단하다. 지연 로드하고자 하는 컴포넌트를 감사주면 된다.

여기서 중요한 것은 단순히 이미지뿐만 아니라 일반 컴포넌트도 이 안에 넣어 지연 로드할 수 있다.

### 최적화

```jsx
import LazyLoad from 'react-lazyload'  
  
function PhotoItem({ photo: { urls, alt } }) {  
  // 생략
  return (  
    <ImageWrap>
		<LazyLoad>  
	        <Image
				src={urls.small + '&t=' + new Date().getTime()}
				alt={alt}
				onClick={openModal}
			/>  
      </LazyLoad>  
    </ImageWrap>  
  );  
}
```

이렇게 수정한 후 이미지 갤러리에서 스크롤해 보면 처음에는 로드되지 않았던 이미지들이 화면에 보일 때 하나씩 로드되는 것을 볼 수 있다.
Network 패널을 띄워 함께 보면 Network 패널에서 지연 로드되는 이미지들이 보일 것이다.

한 가지 아쉬운 점은 이미지가 지연 로드되기 때문에 초기 화면의 리소스를 절약할 수 있는 것은 좋으나,
스크롤을 내려 화면에 이미지가 들어올 대 이미지를 로드하기 때문에 이미지가 보이지 않고 시간이 지나야 이미지가 보인다는 점이다.

이 문제를 해결하기 위해 `react-lazyload` 라이브러리의 옵션 중 `offset` 이라는 옵션을 활용한다.
얼마나 미리 이미지를 로드할지 픽셀 값을 넣어주면 되며 예를 들어 100으로 설정하면 화면에 들어오기 100px 전에 이미지를 로드하는 방식이다.

```jsx
import LazyLoad from 'react-lazyload'  
  
function PhotoItem({ photo: { urls, alt } }) {  
  // 생략
  return (  
    <ImageWrap>
		<LazyLoad offset={1000}> 
	        <Image
				src={urls.small + '&t=' + new Date().getTime()}
				alt={alt}
				onClick={openModal}
			/>  
      </LazyLoad>  
    </ImageWrap>  
  );  
}
```

1000으로 설정한후 변화를 살펴보면 이미지가 미리 준비되어 있음을 확인할 수 있다.


# 리덕스 렌더링 최적화

요즘은 Recoil이나 ContextAPI 등 다양한 상태 관리 라이브러리가 있지만,
여기서는 리덕스에 대해 다룬다.

리덕스에 useSelector라는 훅이 사용되는 과정에서 다양한 성능 문제가 발생한다.
해당 성능 문제를 해결하고 리덕스를 더욱 효율적으로 사용할 수 있는 방법을 알아본다.

여기서는 React Developer Tools를 사용한다.
`Hightlight updates when components render` 항목을 체크하여 렌더링 영역을 확인할 수 있다.

이상한 점은 모달을 띄웠을때 모달과 관련없는 헤더와 리스트 컴포넌트까지 리렌더링된다.

총 세번 일어나는데 다음과 같다.
- 모달을 띄우는 순간
- 모달 이미지가 로드된 후 배경색이 바뀌는 순간
- 모달을 닫는 순간

### 리렌더링의 원인

결론부터 이야기하면 리덕스 때문이다.
구독하고 있는 상태가 변경했을 때를 감지하여 변경이 생기면 리렌더링 되기 때문

하지만 리덕스에서 변경된 상태는 모달에 관련된 상태이지, PhotoListCon-tainer 에서 구독하고 있는 category, photos 상태가 아니다.
상식적으로 영향을 주지 않아도 되는 것이다. 하지만 왜 리렌더링 될까?

이유는 useSelector 동작 방식에 있다.
useSelector는 서로 다른 상태를 참조할 때 리렌더링을 하지 않도록 구현되어 있다.
하지만 그 판단 기준이 useSelector에 인자로 넣은 함수의 반환 값이다.

반환 값이 이전과 값이 같다면 리덕스 상태 변화에 영향이 없다고 판단하여 리렌더링을 하지 않고,
반환 값이 이전 값과 다르면 영향이 있다고 판단하여 리렌더링을 한다.

이런 기준을 이해하고 살펴보면 함수가 객체를 반환하는 것을 볼 수 있다.
객체 내부의 photos와 loading의 값을 보면 달라진게 없어 보일 수 있지만 객체를 새로 만들어 참조 값을 반환하는 형태이므로 useSelector는 리덕스를 통해 구독한 값이 변했다고 판단한다.

이는 ImageModalContainer와 Header도 마찬가지이다.

### useSelector 문제 해결

해결 방법은 크게 두 가지가 있다.
#### 객체를 새로 만들지 않도록 반환 값을 사용하는 방법

첫 번째 방법은 객체로 묶어 반환하면 참조가 바뀌므로 객체를 반환하지 않는 형태로 useSelector를 나누는 방법이다. 이 방법으로 수정하면 다음과 같다.

```js
const modalVisible = useSelector(state => state.imageModal.modalVisible)
const bgColor = useSelector(state => state.imageModal.bgColor)
const src = useSelector(state => state.imageModal.src)
const alt = useSelector(state => state.imageModal.alt)
```

Header 컴포넌트의 useSelector를 객체 형태를 걷어내도록 하자.

#### Equality Function 을 사용하는 방법

Equality Function이란 useSelector의 옵션으로 넣는 함수로, 리덕스 상태가 변했을 때
useSelector가 반환해야 하는 값에도 영향을 미쳤는지 판단하는 함수이다.

**쉽게 말해 이전 반환값과 현재 반환 값을 비교하는 함수이다.**
만약 두 값이 동일하면 리렌더링을 하지 않고, 다르면 리렌더링을 하는 방식이다.

useSelector의 옵션으로 넣거나, 직접 구현하여 넣거나, 리덕스에서 제공하는 함수를 사용할 수도 있다.
여기서는 리덕스에서 제공하는 함수를 이용하여 ImageModalContainer에 다시 적용해보자.

```js
const { modalVisible, bgColor, src, alt } = useSelector(state => ({  
  modalVisible: state.imageModal.modalVisible,  
  bgColor: state.imageModal.bgColor,  
  src: state.imageModal.src,  
  alt: state.imageModal.alt,  
}),shallowEqual);
```

반환 방식은 이전과 다르지 않지만, shallowEqual이라는 값을 반환하고 있다.
리덕스에서 제공하는 객체를 얕은 비교하는 함수이다.
즉 참조값을 비교하는 것이 아니라 객체 내부에 있는 프로퍼티를 비교하여 동일한지 판단한다.

PhotoListContainer도 동일하게 적용하도록하자.

```js
const { photos, loading } = useSelector(state => ({  
  photos:  
    state.category.category === 'all'  
      ? state.photos.data  
      : state.photos.data.filter(  
          photo => photo.category === state.category.category  
        ),  
  loading: state.photos.loading,  
}), shallowEqual);
```

추가로 filter로 배열을 새로 만들고 있는 과정을 제거하도록 하자.
제거하지 않게되면 다른 카테고리에서 이미지 모달을 띄우면 리스트가 렌더링될 것이다.

```js
const { category, allPhotos, loading } = useSelector(state => ({  
  category: state.category.category,  
  allPhotos: state.photos.data,  
  loading: state.photos.loading,  
}), shallowEqual);  
  
const photos = category === 'all' ? allPhotos : allPhotos.filter(photo => photo.category === category)
```

이렇게 하면 모달을 띄워도 이미지 리스트가 리렌더링되지 않음을 확인할 수 있다.

# 병목 코드 최적화

병목 코드를 찾아 해당 코드를 최적화해 볼 것입니다.
1장에서도 병목 코드를 분석하고 최적화해 봤다.
그와 비슷하지만 여기서는 로직을 개선하여 최적화할 뿐만 아니라 메모이제이션이라는 방법을 적용하여 성능 문제를 해결해 본다.

무작정 Performance 패널로 검사하기 보다는, 서비스 이용과정에서 느리거나 문제가 있다고 판단되는 부분을 찾아 검사해 볼 것이다.

이미지 갤러리 서비스에서는 크게 3가지 지점을 확인해 볼 수 있다.
- 페이지가 최초 로드될 때
- 카테고리가 변경될 때
- 모달을 띄웟을 때

별로 느린 느낌이 없을 수 있지만 이미지 모달을 띄워보면 이미지도 늦게뜨고 배경색도 늦게 적용된다.

Performance 패널을 통해 기록 하여 살펴보면 getAverageColorOfImage 함수가 굉장히 느린것을 파악할 수 있다

이미지의 평균 픽셀 값을 계산하는 함수로
캔버스에 이미지를 올리고 픽셀 정보를 불러온 뒤 하나씩 더해 평균을 내고 있다.

즉 이미지 통쨰로 캔버스에 올린다는 점과 반복문을 통해 가져온 픽셀 정보를 하나하나 더하고 있다는 점에서 느린것이다.

두 가지 방법으로 최적화한다.

#### 메모이제이션으로 코드 최적화하기

해당 반환값을 기억해두었다가 똑같은 조건으로 실행되었을 때 코드를 모두 실행하지 않고 바로 전에 기억해 둔값을 반환하는 기술이다.

인자를 제곱하는 함수에 메모이제이션을 적용하면 다음과 같다.

```js
const cache = {}; //함수 반환값을 저장하기 위한 변수

function square(n) {
	if(cache[n]) {
		return cache[n]
	}
	const result = n * n;
	cache[n] = result;
	return result;
}
```

적용하는 대상이나 방식에 따라 코드가 달라질수는 있지만 재활용한다는 개념은 동일하다.
이제 함수에 적용해보도록하자.

인자 값이 문자열이나 숫자 형태가 아니라 객체이기 때문에 인자를 그대로 key로 사용하는 것이 아니라 고유값인 src를 사용한다.

```js
const cache = {}  
  
export function getAverageColorOfImage(imgElement) {  
  if(cache.hasOwnProperty(imgElement.src)) {  
    return cache[imgElement.src];  
  }
  
  // 중략 ...
    
  cache[imgElement.src] = averageColor;  
  
  return averageColor;  
}
```

두 번째 부터는 성능이 대폭 향상된다는 장점이 있다.
하지만 반대로 첫 번째는 여전히 느리다는 점이다.
또한 매번 새로운 값이 들어온다면 오히려 메모리만 낭비하게 될 것이다.

#### 함수의 로직 개선

이 함수에서 느린 이유는 다음과 같다.
- 캔버스에 이미지를 올리고 픽셀 정보를 불러오는 함수
- 모든 픽셀에 대해 실행되는 반복문

모두 이미지 사이즈에 따라 작업량이 결정된다.
즉, 이미지 크기를 줄이면 실행횟수도 적어질 것이다.

코드를 잘 살펴보면 현재는 원본 이미지로 배경색을 계산한다.
제공되고 있는 섬네일 이미지로 변경하면 작업량이 매우 단축될 것이다.

```jsx
function PhotoItem({ photo: { urls, alt } }) {  
  const dispatch = useDispatch();  
  
  const openModal = (e) => {  
    dispatch(showModal({ src: urls.full, alt }));  
  
    const averageColor = getAverageColorOfImage(e.target);  
    dispatch(setBgColor(averageColor));  
  };  
  
  return (  
    <ImageWrap>
		<LazyLoad offset={1000}>  
	        <Image
				src={urls.small + '&t=' + new Date().getTime()}
				alt={alt}
				onClick={openModal}
				crossOrigin="*"
			/>  
	     </LazyLoad>  
    </ImageWrap>  
  );  
}
```

위와 같이 모달을 띄울 때 썸네일 이미지를 제공하도록 변경하였다.
추가로 ImageModal 의 onLoad 이벤트를 제거하도록 하자.

작업 시간을 Performance 패널을 통해 살펴보면 작업시간이 매우 빠르게 완료된 것을 확인할 수 있다.
