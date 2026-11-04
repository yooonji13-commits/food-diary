# 형식

이 앱에는 다른 프로그램이 부르는 인터페이스가 없다. 바깥과 맞물리는 것은 저장 형식, 백업 파일, 카카오 응답 세 가지다.

## 저장 형식 2판 (`localStorage['fd_data_v2']`, 백업 파일의 본문도 같은 모양)

```js
{
  version: 2,
  places: [{
    id: string,
    name: string,                    // 필수(앞뒤 공백 뺀 뒤 비면 저장 불가)
    category: '' | '한식'|'일식'|'중식'|'양식'|'분식'|'고기'|'아시안'|'카페·디저트'|'술집'|'기타',
    area: string,
    location: null | { kakaoId: string, placeName: string, address: string, lat: number|null, lng: number|null },
    revisit: 'yes' | 'no' | '',
    links: string[],
    memo: string,
    fromWishIds: string[],           // 이 가게가 된 가고 싶은 곳 id들
    visits: [{
      id: string,
      visitedAt: 'YYYY-MM-DD' | '',
      rating: 0..5,                  // 0 = 안 매김
      oneLiner: string,
      menus: [{ id: string, name: string, rating: 0..5, note: string }],
      photos: string[],              // IndexedDB 사진 id, 첫 장이 대표
      memo: string,
      createdAt: ISO문자열, updatedAt: ISO문자열
    }],
    createdAt: ISO문자열, updatedAt: ISO문자열
  }],
  wishes: [{
    id, name, category, area, location,   // places와 같은 뜻
    link: string,
    wantMenus: string[],
    memo: string,
    createdAt: ISO문자열, updatedAt: ISO문자열
  }]
}
```

- 어느 객체든 위에 없는 필드는 읽고 쓰는 동안 그대로 남는다.
- 위치의 `kakaoId`가 빈 문자열이면 위치가 없는 것으로 본다. `lat`/`lng`가 숫자가 아니면 지도에 나오지 않는다.

## 1판 형식 (`localStorage['fd_data_v1']`, 읽기 전용)

```js
{ version: 1, places: [{ id, name, category, area, visitedAt, rating, oneLiner,
  menus: [{ id, name, rating, note }], photos: [id], revisit, memo, createdAt, updatedAt }] }
```

기록 하나가 방문 한 번이다. 2판으로 옮기는 규칙은 가게 id = `String(id)`, 방문 id = `String(id) + '_v1'`.

## 사진 (IndexedDB `fd_photos`, 버전 1, store `photos`, keyPath `id`)

`{ id: 'ph_…', full: Blob(JPEG, 긴 변 최대 1600px), thumb: Blob(JPEG, 정사각 360px) }`

## 백업 파일

- 파일 이름: `맛집기록서-YYYY-MM-DD.json`.
- 2판:
  ```js
  { app: 'food-diary', version: 2, exportedAt: ISO문자열,
    places: [...], wishes: [...],               // 위 저장 형식과 같다
    photos: { [사진 id]: { full: 'data:image/jpeg;base64,…', thumb: 'data:image/jpeg;base64,…' } } }
  ```
  `photos`에는 어떤 방문이든 쓰고 있는 사진만 담는다.
- 1판: `{ app: 'food-diary', version: 1(또는 없음), exportedAt, places: [1판 기록], photos }`. 불러올 때 2판으로 바꾼다.
- `app`이 `food-diary`가 아니거나 `places`가 배열이 아니면 "맛집 기록서 백업 파일이 아니에요"로 거절한다. JSON이 아니면 "읽을 수 없는 파일이에요".

## 카카오에서 쓰는 값

- SDK 주소: `https://dapi.kakao.com/v2/maps/sdk.js?appkey=<KAKAO_JS_KEY>&libraries=services&autoload=false`, 이어서 `kakao.maps.load(콜백)`.
- 장소 검색 `new kakao.maps.services.Places().keywordSearch(검색어, 콜백)`의 결과 항목에서 쓰는 필드:

| 카카오 필드 | 앱에서 |
|---|---|
| `id` | `location.kakaoId` (문자열로) |
| `place_name` | `location.placeName`, 이름 칸이 비었을 때 이름 |
| `road_address_name` → 없으면 `address_name` | `location.address` |
| `y`, `x` | `location.lat`, `location.lng` (숫자로) |
| `category_name`, `category_group_code` | 종류 자동 고르기 |

- 콜백 상태가 `OK`가 아니면 결과 없음(`ZERO_RESULT`)이나 실패로 보고 안내 문구만 띄운다.
