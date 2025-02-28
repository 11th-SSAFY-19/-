## 상황

아래 상황이 주어졌다고 가정하자.
- 쓰기 연산: 매일 `1억`개의 단축 URL 생성
- 초당 쓰기 연산: 1억/24/3600 = `1160`
- 읽기 연산: 읽기 연산과 쓰기 연산의 비율은 10:1 -> 그럼 읽기 연산은 초당 `11600`회 발생
- URL 단축 서비스를 10년간 운영한다고 가정 -> 1억 * 365 * 10년 = `3650억`
- 축약 전 URL의 평균 길이는 `100바이트`
- 따라서 10년동안 필요한 저장 용량은 3650억 * 100바이트 = `36.5TB`

## URL 단축기는 기본적으로 두 개의 엔드포인트를 필요로 한다.
1. URL 단축용 API
    - 단축 URL을 생성하고자 할 때, 단축할 URL을 인자로 넣어 POST 요청
    - ex. `POST /api/v1/data/shorten`
2. URL 리다이렉션 API
    - 축약된 URL에 대해 HTTP 요청이 오면, 원래의 URL로 보내주기 위해 리다이렉션 URL 반환
    - ex. `GET /api/vi/shortUrl`

## URL Redirection
단축 URL을 받은 서버는 그 URL을 원래 URL로 바꾸어서 301 응답의 Location 헤더에 넣어 반환한다. 

<img width="811" alt="스크린샷 2024-08-06 오후 9 05 55" src="https://github.com/user-attachments/assets/f57a83e7-7a3d-491a-876a-1850614840ec">

여기서 유의할 것은 301 응답과 302 응답의 차이!

### 301 Permanently Moved
- 해당 URL에 대한 HTTP 요청의 처리 책임이 영구적으로 Location 헤더에 반환된 URL로 이전됨
- 영구적으로 이전되었으므로, 브라우저는 이 응답을 캐시한다. 
- 따라서 추후 같은 단축 URL에 요청을 보낼 필요가 있을 때 브라우저는 캐시된 원래 URL로 응답을 보내게 됨

### 302 Found
- 주어진 URL로의 요청이 '일시적으로' Location 헤더가 지정하는 URL에 의해 처리되어야 함
- 따라서 클라이언트의 요청은 언제나 단축 URL 서버에 먼저 보내진 후에 원래 URL로 리디렉션 되어야 함 

이 둘은 각기 장단점이 있다. 서버 부하를 줄이는 것이 중요하면 캐시를 사용하는 301 Permanent Moved를 사용하는 것이 좋다. 하지만 트래픽 분석이 중요할 때는 302 Found를 쓰는 쪽이 클릭 발생률이나 발생 위치를 추적하는데 유리하다. 

## URL 단축
단축 URL이 `www.tinyurl.com/{hashValue}`와 같다면, 중요한 것은 긴 URL을 이 해시 값으로 대응시킬 해시 함수를 찾는 것이다.

<img width="483" alt="스크린샷 2024-08-06 오후 9 15 06" src="https://github.com/user-attachments/assets/de733e3a-4ad4-47ea-9382-cdd7222fab0d">

해시 함수는 다음 요구사항을 만족해야 한다.
- 입력이 다르면 해시 값도 달라야 한다.
- 원래 입력으로 주어졌던 긴 URL로 복원될 수 있어야 한다. 


## 상세 설계

### 1) 데이터 모델
지금까지는 `<단축 URL, 원래 URL>`의 순서쌍을 메모리에 두는걸로 설계했다. 하지만 메모리는 유한한데다 비싸기 때문에 실제 시스템에 쓰기에는 적합하지 않다.

더 나은 방법은 관계형 db에 저장하는 것이다.

URL 테이블이 있다면 `id`, `shortURL`, `longURL`을 저장하는 것이다.
(실제 테이블은 이것보다 더 많은 컬럼을 가질 수 있지만 중요한 컬럼만 추린 것이다.)

### 2) 해시 함수
해시 함수는 원래 URL을 단축 URL로 변환하는데 쓰인다. 
- 해시 함수가 계산한 단축 URL값을 hashValue라고 칭한다면, hashValue는 [0-9, a-z, A-Z]의 문자들로 구성됨 -> 따라서 사용할 수 있는 문자의 개수는 62(=10+26+26)개 
- 맨 처음에 가정한 상황에서, 이 시스템은 3650억개의 URL을 만들어 낼 수 있어야 한다.

<br>

그럼 hashValue(단축 URL)의 길이는 몇으로 정할까?
- n=7이면 62^7 = 3.5조개의 URL을 만들 수 있다. 
- 따라서 hashValue의 길이는 `7`

<br>

긴 URL을 7글자 문자열로 줄이는 해시 함수가 필요하다. 해시 함수 구현에는 두 가지 방법을 사용할 수 있다.

1. 해시 후 충돌 해소 방법

    해시함수의 결과에서 앞자리 7개만 사용하자.

- 충돌난 경우, 미리 사전에 정의해 둔 문자열을 원래의 URL에 추가해 다시 해시함수를 돌리자. 충돌나지 않을 때까지 반복.

- 충돌나는지 여부를 확인하기 위해 원래의 URL과 단축 URL을 저장한 DB에 쿼리를 날려야 함 → 작업 당 한번 이상의 쿼리는 무조건 발생 → 오버헤드가 크다.

- 블룸 필터를 사용 - 어떤 집합에 특정원소가 있는지 검사

2. base-62 변환법

    흔히 사용되는 접근법 중 하나이다. hashValue에 사용할 수 있는 문자가 62개이기 때문에 62진법 사용.

- 0은 0으로, 10은 a로, 11은 b로, 35는 z로, 61은 Z로 대응.

- 10진수 `11157`는 `2 * 62^2 + 55 * 62^1 + 59 * 62^0` = `[2, 55, 59]` -> `[2, T, X]` -> 62진수 `2TX`이다. 

- 따라서 단축 URL은 `https://tinyurl.com/2TX`가 된다.

<br>

base-62 변환법에 정리하자면 아래와 같다.
- 유일성 보장 ID를 단순히 전환하는 것이기에 `충돌날 염려는 없다.`
- 하지만, ID가 1씩 증가해 `다음 단축 URL이 예측`되는 보안적인 문제가 발생할 수 있다. 
- 유일성 보장 ID가 길어짐과 비례해서 `단축 URL 길이가 길어진다.`

### 3) URL 단축 상세 설계

1. 입력으로 긴 URL을 받는다.
2. 데이터베이스에 해당 URL이 있는지 검사한다.
3. 데이터베이스에 있다면 가져와서 클라이언트에게 반환.
4. 없다면 유일한 ID를 생성. (이 ID는 데이터베이스의 기본 키로 사용된다.)
5. 62진법 변환을 적용하여 ID를 단축 URL로 만든다.
6. `ID`, `단축 URL`, `원래 URL`로 새 데이터베이스 레코드를 만든 후 저장.

### 4) URL 리디렉션 상세 설계

아래는 리디렉션 매커니즘의 상세 설계이다. 쓰기보다 읽기를 더 자주하는 시스템이라, `<단축 URL, 원래 URL>`의 쌍을 캐시에 저장하여 성능을 높였다.

로드밸런서의 동작 흐름을 다음과 같이 요약할 수 있다.

1. 사용자가 단축 URL을 클릭
2. 로드밸런서가 해당 요청을 웹 서버에 전달
3. 단축 URL이 이미 캐시에 있는 경우에는 원래 URL을 꺼내서 클라이언트에게 바로 전달
4. 캐시에 없는 경우에는 데이터베이스에서 꺼낸다.
5. 데이터베이스에서 꺼낸 URL을 캐시에 넣은 후 사용자에게 반환

## 마무리

더 논의해보면 좋을 내용

- 처리율 제한 장치의 사용
  - 지금까지의 설계는 엄청난 양의 URL 단축 요청이 몰리면 무력화될 수 있음
  - 처리율 제한 장치를 두어 IP 주소를 비롯한 필터링 규칙을 이용해 앞단에서 요청을 걸러내자.
- 웹 서버/데이터베이스 규모 확장
  - 웹 서버: 스케일 아웃
  - 데이터베이스: 다중화 또는 샤딩
- 데이터 분석 솔루션 사용
  - 어떤 링크를 얼마나 많은 사용자가 선택했는지? 언제 주로 클릭했는지? 등 확인 가능


