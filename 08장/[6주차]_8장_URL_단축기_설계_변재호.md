> 1. URL 단축 : 주어진 긴 URL을 훨씬 짧게 줄이기
> 2. URL 리디렉션(redirection) : 축약된 URL로 HTTP 요청이 오면 원래 URL로 안내
> 3. 높은 가용성, 규모 확장성, 장애 감내

<br>

# 문제 이해 및 설계 범위 확정

<aside>
💡 면접관과의 소통을 통해 정확한 환경과 범위와 요구사항을 이해하기

</aside>

- 예시
    - https://www.longurl.com/q=chatsystem&c=loggedin&v=v3&l=long
    ⇒ https://www.tinyurl.com/y7ke-ocwj 로 줄이기
    - 트래픽 규모 : 매일 1억개의 단축 url을 만들어야함
    - 단축 url의 길이 : 짧을수록 좋음
    - 단축 url에 포함될 문자에 제한 : 숫자&알파벳만 가능
    - 단축된 url을 시스템에서 지우거나 갱신 가능 여부 : 시스템 단순화를 위해 불가능하다고 가정

<br>

# URL 단축기 설계 방안

## API 엔드포인트 (endpoint)

<aside>
💡 API endpoint ⇒ REST API

</aside>

1. POST 요청 : 긴 url → 짧은 url 로 단축 요청
    - ex)
        - POST /api/v1/data/shorten
            - parameter : [longURL: longURLstring]
            - return : short url
2. GET 요청 : 짧은 url → 긴 url 의 원래 endpoint 가져오기 요청
    - ex)
        - GET /api/v1/shortUrl
            - return : http redirection 목적지가 될 원래 url

## URL Redirection

<aside>
💡 보통 해시 테이블 사용 (`{단축 url : 원래 url}`) 
⇒ 메모리가 유한하고 비싸므로, 관계형 DB에 저장 (id, short url, long url)

</aside>

![8-1  redirection](https://github.com/user-attachments/assets/4ae8a22b-83fc-48b3-9eaa-9f7bf745af9c)


1. 단축 url 요청 받음
2. 해당 요청 url을 원래 url 로 바꾸어 `301` 응답의 Location Header 에 넣어 반환
3. 원래 url 로 redirection

### 301 response vs 302 response

- 301 Permanently Moved
    - 해당 url에 대한 http 요청의 처리 책임이 영구적으로 Location 헤더에 반환된 url로 이전됨
    - 브라우저가 이 응답을 **캐시**
        - 추후 같은 단축 url에 대한 요청을 다시 같은 곳으로 보내지 않고, 캐시 된 원래 url로 요청
    - 장점 : 서버 부하 줄이기
- 302 Found
    - 해당 url에 대한 http 요청이 ‘일시적’으로 Location 헤더에 반환된 url에 의해 처리됨
    - 매 요청은 언제나 단축 url 서버에 먼저 보내짐
    - 장점 : 트래픽 분석 용이 (클릭 발생률, 발생 위치 추적 등)

## URL 단축

![8-2  단축](https://github.com/user-attachments/assets/9c235d99-28b0-45eb-a660-2550263956cc)

- 해시 값 길이 = URL 길이
    - 해시 값은 [0-9, a-z, A-Z] 의 문자들로 구성 ⇒ 62개
    
    ![8-3  해시 길이](https://github.com/user-attachments/assets/453946d0-3e95-4628-9196-e46e850b8f83)
    
    - ex) n=7 이면, 3.5조 개의 url 을 만들 수 있음 (요구사항에 따라 결정)

### 해시 함수 구현 기술 2가지

1. **해시 후 충돌 해소**
    - 원리
        - 계산된 해시 값에서 앞7글자만 사용
        - 충돌이 발생하면, 기존url에 임의의 문자열을 덧붙여 다시 해시
        - 반복
    
    ![8-4  해시 후 충돌 해소](https://github.com/user-attachments/assets/b43c602c-f62e-407f-a1a1-0292094175dc)
    
    - 장점
        - 단축 url의 길이가 고정됨
        - 유일성이 보장되는 ID 생성기가 필요없음
        - ID로부터 단축 url을 계산하는 방식이 아니라, 다음에 쓸 수 있는 url을 알아내는 것이 불가능 ⇒ 보안 good
    - 단점
        - 단축 url을 생성할 때 한번 이상 DB 질의가 필요 ⇒ 오버헤드가 큼
            - 블룸 필터를 사용하여 성능 높일 수 있음

1. **base-62 변환**
    - 원리
        - ID 생성기로 만들어진 ID를 62 진수로 표현
        - 62진법인 이유 : 해시값으로 62개의 문자가 들어올 수 있기 때문에
        - ex) 10진수로 11157 → 62진수로 XT2
            
            ![8-5  base62](https://github.com/user-attachments/assets/f4a9dd13-ef4e-431b-adfd-83920ae4537d)
            
    
    ![8-6  base62 설계](https://github.com/user-attachments/assets/52988d00-b542-48a6-82b9-90bb1ea1d5d1)
    
    - 장점
        - ID의 유일성이 보장된 후에 변환하므로 충돌이 아예 없음
    - 단점
        - 단축 url의 길이가 가변적, ID 값이 커지면 같이 길어짐
        - 유일성 보장 ID 생성기가 필요
        - ID가 1씩 증가하는 경우, 다음에 쓸 단축 url이 뭔지 쉽게 알수 있음 ⇒ 보안 bad
