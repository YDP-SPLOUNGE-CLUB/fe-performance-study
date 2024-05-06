# 학습할 최적화 기법

#### 이미지 지연 로딩

말 그대로 첫 화면에 당장 필요하지 않은 이미지가 먼저 로드되지 않도록 지연시키는 기법
사용자에게 가장 먼저 보이는 콘텐츠를 더 바르게 로드할 수 있다.

#### 이미지 사이즈 최적화

Unsplash에서 제공하는 CDN이 아닌 서버에 저장되어 있는 정적 이미지를 최적화해본다.

#### 폰트 최적화

기본 폰트만 사용하면 문제가 없겠지만, 커스텀 폰트를 적용하려고 한다면 몇 가지 성능을 야기할 수 있다. 커스텀 폰트를 적용할 때 발생할 수 있는 문제를 알아보고 최적화해본다.

#### 캐시 최적화

자주 사용되는 리소스를 브라우저에 저장해두고, 다음번에 사용할 때 새로 다운로드하지 않고 저장되어 있는 것을 사용하는 기술을 적용해본다.

#### 불필요한 CSS 제거

불필요한 코드(JS, CSS)가 빌드되는 경우가 있다.
CSS 코드가 서비스 코드에 포함되어 있을 경우 해당 코드를 제거하여 파일 사이즈를 줄이는 방법에 대해 알아본다.

# 분석 툴

#### 크롬 개발자 도구 Coverage 패널

- 웹 페이지를 렌더링하는 과정에서 어떤 코드가 실행되었는지 보여준다.
- 각 파일의 코드가 얼마나 실행됐는지 비율로 나타내준다.
- 특정 파일에서 극히 일부 코드만 실행되었다면(퍼센티지가 낮다면) 해당 파일에 불필요한 코드가 많이 포함되어 있다고 볼 수 있다.

#### Squoosh

- 웹에서 서비스되는 이미지 압축 도구
- 구글에서 만들었으며, 간편하게 이미지의 포맷이나 사이즈를 변경할 수 있다.

#### PurgeCSS

- 사용하지 않는 CSS를 제거해주는 툴
- CLI를 통해 실행하거나 webpack과 같은 번들러에서 플러그인으로 추가하여 사용

# 서비스 탐색 및 코드 분석

서비스는 일반적인 홈페이지 구조이다.
첫 페이지에 커다란 동영상 배너와 함께 커스텀 폰트가 적용된 텍스트가 있다.
그 아래로는 롱보드에 대한 간략한 소개와 각 페이지로 이동하는 버튼이 존재한다.

스타일링은 Tailwind CSS를 사용하고 있다.
서버는 Express로 간단하게 구현되어 있다.

# 이미지 지연 로딩

#### 네트워크 분석

서비스를 분석하고 최적화할 때 어떤 작업을 먼저 해야 한다는 규칙은 없다.
상황에 따라 원하는 분석을 진행하면 된다.

여기서는 네트워크를 먼저 살펴본다.

Network Throttling 옵션에 Add 를 선택하여 원하는 throttling 옵션을 설정할 수 있다.

![ch03-01.png](attached-file/ch03-01.png)

6000이라는 이름, 다운로드와 업로드 속도를 6000kb/s로 설정하여 진행한다.
참고로 기본적으로 설정되어있는 Fast 3G와 Slow 3G 속도는 다음과 같다.

|               | Fast 3G  | Slow 3G |
| ------------- | -------- | ------- |
| 다운로드 속도 | 1500kb/s | 780kb/s |
| 업로드 속도   | 780kb/s  | 330kb/s |

추가한 옵션으로 throttling을 적용하고 네트워크 분석을 진행해보자.

![ch03-02.png](attached-file/ch03-02.png)

당장 중요한 리소스인 bundle 파일이 다운로드되는 것을 볼 수 있고
main1, 2, 3 이미지와 폰트가 다운로드 되는 것을 볼 수 있다.

그리고 한동안 banner-video 파일이 pending상태(하얀 막대)로 존재하다가
일부 리소스(main-items.jpg)의 다운로드가 완료된 후에 다운로드되는 것을 볼 수 있다.

그런데 banner-video는 페이지에서 가장 처음으로 사용자에게 보이는 콘텐츠인데 가장 나중에 로드되면,
사용자가 첫 화면에서 아무것도 보지 못한 채로 오랫동안 머물게 되므로 사용자 경험에 좋지 않을 것이다.

**이 문제를 어떻게 해결할 수 있을까?**

동영상 다운로드를 방해하는 사용되지 않는 이미지를 나중에 다운로드되도록 하여 동영상이 먼저 다운로드하게 변경하면 될 것이다.

즉, 이미지를 지연로드 시킨다.

**이 이미지들을 페이지가 로드될 때 로드하지 않는다면 언제 로드해야할까?**

바로 이미지가 화면에 보이는 순간 또는 그 직전에 이미지를 로드해야할 것이다.

즉, 뷰포트에 이미지가 표시될 위치까지 스크롤 되었을 때

# Intersection Observer

뷰포트에 이미지를 보이게 할지 판단해야하는데,
scroll 이벤트를 적용하여 throttle과 같은 방식으로 처리할 수 있지만 근본적인 해결 방법이 될 수는 없다.

이런 스크롤 문제를 해결할 수 있는 방법이 있다.
Intersection Observer는 브라우저에서 제공하는 API이다.
이를 통해 웹 페이지의 특정 요소를 관찰하면 페이지 스크롤 시,
해당 요소가 화면에 들어왔는지 아닌지 알려준다.

즉 스크롤 이벤트처럼 스크롤할 때마다 함수를 호출하는 것이 아니라
요소가 화면에 들어왔을 때만 함수를 호출하는 것

성능면에서 scroll 이벤트로 판단하는 것보다 훨씬 효율적이다.

```js
const options = {
	/*
		객체의 가시성을 확인할 때 사용되는 뷰포트 요소
		기본값 null
		null로 설정 시 브라우저의 뷰포트로 설정된다.
	*/
	root: null,
	/*
		root 요소의 여백
		root의 가시 범위를 가상으로 확장하거나 축소할 수 있다.
	*/
	rootMargin: '0px',
	/*
		가시성 퍼센티지
		대상 요소가 어느정도로 보일 때 콜백을 실행할지 결정한다.
		1.0으로 설정하면 대상 요소가 모두 보일 때 실행되며,
		0으로 설정하면 1px이라도 보이는 경우 콜백이 실행된다.
	*/
	threshold: 1.0,
}

// 가시성이 변경될 때마다 실행되는 함수
const callback = (entries, observer) => {
	console.log('Entries', entries)
}

// observer 인스턴스
const observer = new InterectionsObserver(callback, options)

observer.observe(document.querySelector('#target-element1'))
observer.observe(document.querySelector('#target-element2'))
```


# Intersection Observer 적용하기

나란히 렌더링 되는 이미지 3장에 지연로딩을 적용해보자.

```jsx
// 적용 전
function Card(props) {  
    return (  
       <div className="Card text-center">  
          <img src={props.image}/>  
          <div className="p-5 font-semibold text-gray-700 text-xl md:text-lg lg:text-xl keep-all">  
             {props.children}  
          </div>  
       </div>  
    )  
}  
```

```jsx
// 적용후
function Card(props) {  
    const imgRef = useRef(null)  
  
    useEffect(() => {  
       const options = {}  
       const callback = (entries, observer) => {  
          entries.forEach(entry => {  
             if(entry.isIntersecting) {  
                entry.target.src = entry.target.dataset.src  
                observer.unobserve(entry.target)  
             }  
          })  
       }  
  
       const observer = new IntersectionObserver(callback, options)  
  
       observer.observe(imgRef.current)  
  
       return () => observer.disconnect()  
    },[])  
  
    return (  
       <div className="Card text-center">  
          <img data-src={props.image} ref={imgRef} />  
          <div className="p-5 font-semibold text-gray-700 text-xl md:text-lg lg:text-xl keep-all">  
             {props.children}  
          </div>  
       </div>  
    )  
}
```


![ch03-03.png](attached-file/ch03-03.png)

로그를 찍어서 살펴보면 3개의 로그가 출력된다.
그중 중요한 값은 `isIntersectiong` 값이다.

이 값은 해당 요소가 뷰포트 내에 들어왔는지를 나타내는 값이다.
이 값을 통해 요소가 화면에 보이는지 나간 것인지 알 수 있다.

이미지가 보이는 순간, 즉 콜백이 실행되는 순간 이미지를 로드하도록 변경해보자.
이미지 로딩은 img 태그에 src가 할당되는 순간 일어난다.

따라서 최초에는 img 태그에 src 값을 할당하지 않다가 콜백이 실행되는 순간
src를 할당함으로써 이미지 지연 로딩을 적용할 수 있다.

여기서는 data-src에 주소를 넣어두었다가 src로 옮겨 이미지를 로드하였다.
한번 이미지 로드 후에 다시 호출할 필요가 없으므로 `observe` 를 호출하여 observe를 해제한다.

# 이미지 사이즈 최적화

#### 느린 이미지 로딩 분석

앞서 배너의 동영상 콘텐츠가 별다른 지연 없이 바로 다운로드 할 수 있도록
이미지 지연 로딩을 적용하였다.

**하지만 막상 이미지가 로드되는 속도는 굉장히 느리다.**

로드되면서 이미지가 잘려보이기 까지 하는데, 이는 서비스가 느리다는 느낌을 줄 수 있다.

Network 패널을 통해 이미지 크기를 살펴보면, 파일 크기가 매우 큰 것을 볼 수 있다.

![ch03-04.png](attached-file/ch03-04.png)

이렇게 이미지 사이즈가 크면 다운로드에 많은 시간이 걸리고 다른 작업에 영향을 준다.
이미지 사이즈 최적화를 진행해보자.

#### 이미지 포맷 종류

이미지 사이즈 최적화는 간단히 말하면
가로, 세로 사이즈를 줄여 이미지 용량을 줄이고 그 만큼 더 빠르게 다운로드하는 기법이다.

이미지의 사이즈를 줄이기 전에, 이미지를 잘 다루기 위해 짚고 넘어가야 할 것이 있다.
**바로 이미지의 포맷이다.**

SVG와 같은 벡터 이미지가 아닌 비트맵 이미지 포맷 중 대표적인 세가지 포맷을 살펴보자.

- PNG
    - 무손실 압축 방식
    - 원본을 훼손 없이 압축
    - 알파 채널을 지원
- JPG(JPEG)
    - 압축과정에정보 손실 발생 (더 작은 사이즈로 줄일 수 있다.)
    - 고화질이어야 하거나 투명정보가 필요한게 아니면 JPG 권장
- WebP
    - 무손실 압축과 손실 압축을 모두 제공하는 최신 이미지 포맷
    - PNG,JPG에 비해 대단히 효율적으로 이미지를 압축할 수 있다.
    - 공식문서에 따르면 WebP 방식은 PNG 대비 26% JPG 대비 25~34% 더 나은 효율을 가지고 있다고 한다.

위 내용을 살펴보면 WebP를 사용하는 것이 마냥 좋을 것 같다.
하지만 간단한 문제는 아니다.

이유는 바로 브라우저 호환성 때문인데
꽤나 최신 이미지 파일 포맷이라서 아직 지원하지 않는 브라우저도 있다.

- 사이즈 : PNG > JPG > WebP
- 화질 : PNG = WebP > JPG
- 호환성 : PNG = JPG > WebP

![ch03-05.png](attached-file/ch03-05.png)

#### Squoosh를 사용하여 이미지 변환

WebP 포맷으로 변환하여 고화질, 저용량의 이미지로 최적화해보도록 하자.

그러러면 이미지를 변환해주면 컨버터가 필요한데,
여기서 사용할 컨버터는 바로 Squoosh라는 애플리케이션이다.

Squoosh는 구글에서 만든 이미지 컨버터 웹 애플리케이션이다.

별도의 프로그램 설치 없이 웹에서 이미지를 손쉽게 여러 가지 포맷으로 변환할 수 있고,
원본과 비교하는 등 다양한 기능을 이용할 수 있다.

resize를 통해 600x600 사이즈로 변경하고 WebP로 변환하여 적용해보도록 하자.

![ch03-06.png](attached-file/ch03-06.png)

이미지 사이즈를 Network 탭에서 확인해보면 다운로드 시간도 상당히 짧아진 것을 볼 수 있다.

문제는 WebP로만 이미지를 렌더링할 경우 특정 브라우저에서는 제대로 렌더링되지 않을 수 있다는 것인데, 이런 문제를 해결하려면 단순 Img 태그로만 이미지를 렌더링하면 안되며, picture 태그를 사용해야한다.

picture 태그는 다양한 타입의 이미지를 렌더링하는 컨테이너로 사용된다.
예를 들어 아래 코드처럼 브라우저 사이즈에 따라 지정된 이미지를 렌더링하거나 지원되는 타입의 이미지를 찾아 렌더링한다.

```jsx
// 뷰포트에 따라 구분
<picture>
	<soruce media="(min-width:650px)" srcset="img_pink_flowers.jpg" />
	<soruce media="(min-width:465px)" srcset="img_white_flower.jpg" />
	<img src="img.jpg" alt="Flowers" style="width: auto;" />
</picture>
// 이미지 포맷에 따라 구분
<picture>
	<soruce srcset="photo.avif" type="image/avif" />
	<soruce srcset="photo.webp" type="image/webp" />
	<img src="photo.jpg" alt="photo" />
</picture>
```

picture 태그를 사용하여 브라우저가 WebP를 렌더링하지 못할 때 JPG 이미지로 렌더링하도록 수정해보자.

```jsx
function Card(props) {  
    const imgRef = useLazyImageLoad();  
    return (  
       <div className="Card text-center">  
          <picture>  
             <source data-srcset={props.webp} type="image/webp" />  
             <img data-src={props.image} ref={imgRef} />  
          </picture>  
          <div className="p-5 font-semibold text-gray-700 text-xl md:text-lg lg:text-xl keep-all">  
             {props.children}  
          </div>  
       </div>  
    )  
}
```

가장 상위에 있는 WebP를 우선으로 로드하고, 브라우저가 WebP를 지원하지 않으면 img 태그에 있는 JPG 이미지를 렌더링한다.

최적화 전후를 Network 탭에서 직접 로딩 속도를 비교해보도록 하자.

# 동영상 최적화

**동영상 콘텐츠 분석**

Network 패널 에서 살펴보면 이미지처럼 하나의 요청으로 모든 영상을 다운로드하지않는다.
콘텐츠의 특성상 파일 크기가 크기 때문에 당장 재생이 필요한 앞부분을 먼저 다운로드한 뒤
순차적으로 나머지 내용을 다운로드한다.

그래서 동영상 콘텐츠의 다운로드 요청이 여러 개로 나뉘어 있는 것이다.

하지만 역시 문제가 있다.
아무리 여러 번 나눠서 다운로드를 해도 애초에 동영상 파일이 크다보니 재생하기 까지 꽤 오래 걸린다.

![ch03-07.png](attached-file/ch03-07.png)

Performance 패널을 통해 확인해보면 일정 시간 동안 동영상 콘텐츠가 다운로드되고, 그 이후에야 재생이 되는 것을 볼 수 있다.

동영상 파일 크기도 확인해보면 54MB인 것을 알 수 있다.

**동영상 압축**

동영상 최적화도 이미지와 비슷하다.
가로,세로 사이즈를 줄이고 압축 방식을 변경하여 동영상의 용량을 줄이는 것

물론 동영상에는 프레임 레이트(frame rate)등 이미지보다는 복잡한 설정이 있지만 그 정도까지 알 필요는 없다.

여기서는 단순히 압축하는 툴을 활용하여 최적화를 진행한다.

>동영상이 메인 콘텐츠인 서비스에서는 이 작업을 추천하지 않는다.

[Media.io](https://www.media.io/ko/convert/mp4-to-webm.html) 서비스를 사용하여 비디오를 압축한다.

Bitrate는 제일 낮은 512Kbps로 설정하고 WebM 포맷으로 변환한다.
또 오디오는 사용하지 않으니 제거한다.

>Bitrate는 특정 시간 단위마다 처리하는 비트의 수로 동영상에서는 1초에 얼마나 많은 정보를 포함하는가를 의미한다. 따라서 이 값이 크면 1초에 더 많은 정보를 포함하게 되므로 화질은 좋아지지만 파일의 사이즈는 커진다.

다운로드하여 적용해보록 하자.

**최적화 전후 비교**

Performance 패널로 분석해보면 동영상이 이전과 달리 매우 빠르게 로드되고 재생되는 것을 확인할 수 있다. 사이트에서 직접 새로고침을 해서 눈으로 확인해 봐도 큰 끊김 없이 영상이 로드되고 재생되는 것을 볼 수 있다.

> 팁
> 변환 후 화질이 전환되었는데 패턴을 적용하거나 필터를 씌워 좋지 화질이 좋지 않음을 인지할 수 없다.
> 혹은 css의 blur를 통해 흐림 효과를 적용해보도록 하자.
