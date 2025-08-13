# 웹 워커의 구체적인 사용 사례

## On this page
- 웹 워커 없이 본 메인 스레드의 모습
- 웹 워커가 있을 때 메인 스레드의 모습
- 웹 워커 코드 살펴보기

> [!TIP]
> 이 섹션은 [데모](https://chrome.dev/learn-performance-exif-worker/)를 통해 웹 워커가 메인 스레드의 작업을 별도 스레드로 오프로드하는 방식을 보여줍니다. 직접 확인하며 따라와 보세요!

이전 모듈에서는 웹 워커 개요를 살펴봤습니다. 웹 워커는 JavaScript를 메인 스레드 밖, 별도의 웹 워커 스레드로 옮겨 입력 반응성을 개선할 수 있습니다. 메인 스레드에 직접 접근이 필요하지 않은 작업이 있을 때 웹사이트의 INP(Interaction to Next Paint)를 개선하는 데 도움이 됩니다. 이번 모듈에서는 개요를 넘어, 웹 워커의 구체적인 사용 사례를 다룹니다.

예시로, 이미지의 Exif 메타데이터를 제거하거나 확인해야 하는 웹사이트를 생각해봅시다. 전혀 낯선 요구가 아닙니다. 실제로 Flickr 같은 사이트는 색심도, 카메라 제조사와 모델 등 이미지의 기술적 정보를 확인할 수 있도록 Exif 메타데이터 보기를 제공합니다.

하지만 이미지를 가져오고 `ArrayBuffer`로 변환한 뒤 Exif 메타데이터를 추출하는 로직을 전부 메인 스레드에서 처리하면 비용이 커질 수 있습니다. 다행히 웹 워커 스코프를 사용하면 이 작업을 메인 스레드 밖에서 처리할 수 있습니다. 이후 웹 워커의 메시징 파이프라인을 이용해 Exif 메타데이터를 HTML 문자열로 메인 스레드로 전송하고, 사용자에게 표시할 수 있습니다.

## 웹 워커 없이 본 메인 스레드의 모습

먼저, 이 작업을 웹 워커 없이 수행할 때 메인 스레드가 어떻게 보이는지 관찰해 봅시다. 다음 단계를 따라 해보세요.

1. Chrome에서 새 탭을 열고 DevTools를 실행합니다.
2. Performance 패널을 엽니다.
3. https://chrome.dev/learn-performance-exif-worker/without-worker.html 로 이동합니다.
4. Performance 패널 우측 상단의 Record 버튼을 클릭합니다.
5. Exif 메타데이터가 포함된 이미지 링크(또는 임의의 링크)를 입력 필드에 붙여넣고 Get that JPEG! 버튼을 클릭합니다.
6. 인터페이스에 Exif 메타데이터가 표시되면 다시 Record 버튼을 눌러 기록을 중지합니다.

![main-thread-1](/img/main-thread-1.png)
*이미지 메타데이터 추출 앱의 메인 스레드 활동. 모든 작업이 메인 스레드에서 발생합니다.*

래스터라이저 스레드 등 다른 스레드가 있을 수는 있지만, 앱의 모든 작업은 메인 스레드에서 발생합니다. 메인 스레드에서는 다음이 이루어집니다.

1. 폼이 입력을 받아 Exif 메타데이터가 포함된 이미지의 초반부를 가져오기 위한 fetch 요청을 전송합니다.
2. 이미지 데이터가 `ArrayBuffer`로 변환됩니다.
3. exif-reader 스크립트로 이미지에서 Exif 메타데이터를 추출합니다.
4. 메타데이터를 스크랩하여 HTML 문자열을 구성하고, 메타데이터 뷰어에 채웁니다.

이제 동일한 동작을 웹 워커로 구현한 버전과 비교해봅시다!

## 웹 워커가 있을 때 메인 스레드의 모습

JPEG 파일에서 Exif 메타데이터를 메인 스레드로 추출하는 모습을 봤으니, 웹 워커를 사용했을 때는 어떻게 보이는지 살펴봅시다.

1. Chrome에서 다른 탭을 열고 DevTools를 실행합니다.
2. Performance 패널을 엽니다.
3. https://chrome.dev/learn-performance-exif-worker/with-worker.html 로 이동합니다.
4. Performance 패널 우측 상단의 Record 버튼을 클릭합니다.
5. 이미지 링크를 입력 필드에 붙여넣고 Get that JPEG! 버튼을 클릭합니다.
6. 인터페이스에 Exif 메타데이터가 표시되면 다시 Record 버튼을 눌러 기록을 중지합니다.

![main-thread-2](/img/main-thread-2.png)
*이미지 메타데이터 추출 앱의 메인 스레드 활동. 대부분의 작업이 수행되는 추가 웹 워커 스레드가 있습니다.*

이것이 웹 워커의 힘입니다. 메인 스레드에서 모든 것을 처리하는 대신, HTML로 메타데이터 뷰어를 채우는 일을 제외한 모든 작업을 별도 스레드에서 수행합니다. 그 결과 메인 스레드는 다른 작업을 처리할 여유가 생깁니다.

여기서 가장 큰 장점은, 웹 워커를 사용하지 않는 버전과 달리 exif-reader 스크립트를 메인 스레드가 아닌 웹 워커 스레드에서 로드한다는 점입니다. 즉, exif-reader 스크립트의 다운로드, 파싱, 컴파일 비용이 메인 스레드 밖에서 발생합니다.

이제 이러한 작업을 가능하게 하는 웹 워커 코드를 살펴봅시다!

## 웹 워커 코드 살펴보기

웹 워커의 효과를 확인하는 것만으로는 충분하지 않습니다. 여기서는 웹 워커 스코프에서 어떤 코드가 가능한지 이해를 돕기 위해, 관련된 코드 일부를 살펴봅니다.

> [!NOTE]
> 아래 코드는 전체가 아니라 핵심 부분만 담고 있습니다. 전체 동작은 데모의 소스를 참고하세요.

먼저, 웹 워커가 동작하기 전에 메인 스레드에서 수행되어야 하는 코드입니다.

```javascript
// scripts.js

// Register the Exif reader web worker:
const exifWorker = new Worker('/js/with-worker/exif-worker.js');

// We have to send image requests through this proxy due to CORS limitations:
const imageFetchPrefix = 'https://res.cloudinary.com/demo/image/fetch/';

// Necessary elements we need to select:
const imageFetchPanel = document.getElementById('image-fetch');
const imageExifDataPanel = document.getElementById('image-exif-data');
const exifDataPanel = document.getElementById('exif-data');
const imageInput = document.getElementById('image-url');

// What to do when the form is submitted.
document.getElementById('image-form').addEventListener('submit', event => {
  // Don't let the form submit by default:
  event.preventDefault();

  // Send the image URL to the web worker on submit:
  exifWorker.postMessage(`${imageFetchPrefix}${imageInput.value}`);
});

// This listens for the Exif metadata to come back from the web worker:
exifWorker.addEventListener('message', ({ data }) => {
  // This populates the Exif metadata viewer:
  exifDataPanel.innerHTML = data.message;
  imageFetchPanel.style.display = 'none';
  imageExifDataPanel.style.display = 'block';
});
```

이 코드는 메인 스레드에서 실행되며, 폼이 이미지 URL을 웹 워커로 전송하도록 설정합니다. 그 다음, 웹 워커 코드는 외부 exif-reader 스크립트를 로드하기 위해 `importScripts`를 사용하고, 메인 스레드와의 메시징 파이프라인을 구성합니다.

```javascript
// exif-worker.js

// Import the exif-reader script:
importScripts('/js/with-worker/exifreader.js');

// Set up a messaging pipeline to send the Exif data to the `window`:
self.addEventListener('message', ({ data }) => {
  getExifDataFromImage(data).then(status => {
    self.postMessage(status);
  });
});
```

> [!NOTE]
> 외부 스크립트를 웹 워커 스코프로 가져오는 `importScripts` 구문은 호환성이 넓습니다. 또한 대부분의 브라우저에서 정적 `import` 구문으로 모듈을 웹 워커로 가져오는 것도 가능합니다. 자세한 내용은 [Threading the web with module workers](https://web.dev/module-workers/)를 참고하세요.

위 JavaScript는 사용자가 JPEG 파일의 URL을 포함한 폼을 제출하면, 그 URL이 웹 워커에 도착하도록 메시징 파이프라인을 설정합니다. 이어지는 코드는 JPEG 파일에서 Exif 메타데이터를 추출하고, HTML 문자열을 만들어 `window`로 다시 보내 사용자에게 표시되도록 합니다.

```javascript
// Takes a blob to transform the image data into an `ArrayBuffer`:
// NOTE: these promises are simplified for readability, and don't include
// rejections on failures. Check out the complete web worker code:
// https://chrome.dev/learn-performance-exif-worker/js/with-worker/exif-worker.js
const readBlobAsArrayBuffer = blob => new Promise(resolve => {
  const reader = new FileReader();

  reader.onload = () => {
    resolve(reader.result);
  };

  reader.readAsArrayBuffer(blob);
});

// Takes the Exif metadata and converts it to a markup string to
// display in the Exif metadata viewer in the DOM:
const exifToMarkup = exif => Object.entries(exif).map(([exifNode, exifData]) => {
  return `
    <details>
      <summary>
        <h2>${exifNode}</h2>
      </summary>
      <p>${exifNode === 'base64' ? `<img src="data:image/jpeg;base64,${exifData}">` : typeof exifData.value === 'undefined' ? exifData : exifData.description || exifData.value}</p>
    </details>
  `;
}).join('');

// Fetches a partial image and gets its Exif data
const getExifDataFromImage = imageUrl => new Promise(resolve => {
  fetch(imageUrl, {
    headers: {
      // Use a range request to only download the first 64 KiB of an image.
      // This ensures bandwidth isn't wasted by downloading what may be a huge
      // JPEG file when all that's needed is the metadata.
      'Range': `bytes=0-${2 ** 10 * 64}`
    }
  }).then(response => {
    if (response.ok) {
      return response.clone().blob();
    }
  }).then(responseBlob => {
    readBlobAsArrayBuffer(responseBlob).then(arrayBuffer => {
      const tags = ExifReader.load(arrayBuffer, {
        expanded: true
      });

      resolve({
        status: true,
        message: Object.values(tags).map(tag => exifToMarkup(tag)).join('')
      });
    });
  });
});
```

코드가 제법 길지만, 그만큼 웹 워커의 활용이 깊은 사례이기도 합니다. 기대할 만한 결과를 얻을 수 있으며, 이 사용 사례에만 국한되지도 않습니다. 예를 들어 `fetch` 호출을 분리하고 응답을 처리하거나, 대량 데이터를 메인 스레드를 막지 않고 처리하는 등 다양한 작업에 웹 워커를 활용할 수 있습니다.

웹 애플리케이션의 성능을 개선할 때, 웹 워커 스코프에서 무리 없이 수행할 수 있는 작업이 무엇인지 먼저 생각해보세요. 상당한 성능 향상을 얻을 수 있으며, 궁극적으로 더 나은 사용자 경험으로 이어질 수 있습니다.


