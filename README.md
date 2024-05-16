## DOM 재사용하여 인피니티 스크롤 최적화하기

![ezgif-6-e99fa8d058](https://github.com/MochaChoco/infinite-scroll/assets/30750586/4472ce76-ce21-4f23-bb3b-9bee87a8fcb6)

이 프로젝트는 DOM을 재사용하여 인피니티 스크롤을 최적화하는 예시를 보여주는 프로젝트입니다. 데이터를 로드할 때마다 새로운 DOM을 생성하는 것이 아니라, 미리 생성된 DOM을 재활용하여 성능을 최적화하는 방법을 구현했습니다.

## 설치 및 실행 방법

### dependency 설치

yarn install

### front-end 실행

yarn dev

### back-end 실행

json-server를 사용하므로 별도의 터미널에서 실행해야 합니다.

yarn json-server --watch db.json --port 3001

## 상세 설명

보다 상세한 설명은 아래의 링크에서 확인하실 수 있습니다.

[Velog 바로가기](https://velog.io/@swj9077/Intersection-Observer-API%EB%A5%BC-%ED%99%9C%EC%9A%A9%ED%95%9C-Infinite-Scroll-%EC%B5%9C%EC%A0%81%ED%99%94%ED%95%98%EA%B8%B0)
