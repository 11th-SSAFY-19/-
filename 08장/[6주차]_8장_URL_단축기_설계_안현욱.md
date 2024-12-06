> **8장. URL 단축기 설계**

<details>
  <summary><b>문제 이해 및 설계 범위 확정</b></summary>

  ---
  
  ## 1단계: 문제 이해 및 설계 범위 확정
  
  ### 시스템 설계: 질문을 통한 요구사항 알아내기
  
  - **Q.** URL 단축기는 어떻게 동작해야 하나요?
  - **A.** 단축 URL을 결과로 제공해야 한다.
      - 입력 : `https://www.systeminterview.com/q=chatsystem&c=loggedin&v=v3&l=long`
      - 출력 : `https://tinyurl.com/y7ke-ocwj`
  - **Q.** 트래픽 규모는 어느 정도 인가요?
  - **A.** 매일 1억(100million)개의 단축 URL을 만들어 낼 있어야 한다.
  - **Q.** 단축 URL의 길이는 어느 정도여야 하나요?
  - **A.** 짧으면 짧을수록 좋다.
  - **Q.** 단축 URL에 포함될 문자에 제한이 있나요?
  - **A.** 숫자(0~9), 영문(a-z, A-Z)만 사용 가능
  
  ### 시스템의 기본적 기능
  
  - URL 단축: 주어진 긴 URL을 훨신 짧게 줄인다.
  - URL Redirection : 축약된 URL로 HTTP 요청이 오면 원래 URL로 안내
  - 높은 가용성과 규모 확장성, 장애 감내 요구
  
  ### 개략적 추정
  
  - 쓰기 연산: 매일 1억 개의 단축 URL 생성
  - 초당 쓰기 연산: 1억(100million)/24/3600 = 1160
  - 읽기 연산: 읽기 연산과 쓰기 연산 비율은 10:1, 1160x10 = 초당 11600
  - URL 단축 서비스 10년간 운영 가정: 1억x365x10 = 3650억(365billion) 개의 레코드를 보관
  - 축약전 URL 평균 길이는 100
  - 10년 동안 필요한 저장 용량: 3650억x100바이트 = 36.5TB
  
  ---
</details>
<details>
  <summary><b>개략적 설계안 제시 및 동의 구하기</b></summary>

  ---
  
  ## 2단계: 개략적 설계안 제시 및 동의 구하기
  
  ### API 엔드포인트
  
  - 클라이언트는 서버가 제공하는 API 엔드포인트를 통해 서버와 통신
  - 엔드포인트를 **REST 스타일**로 설계
  - **URL 단축기에서 필요한 엔드포인트**
      - **1️⃣ URL 단축용 엔드포인트**
          - 단축 URL을 생성하고자 하는 클라이언트는 단축할 URL을 POST 요청
          - **POST api/v1/data/shorten**
          - 인자: `{longURL: longURLstring}`
          - 반환: `단축 URL`
      - **2️⃣ URL Redirection 용도 엔드포인트**
          - 단축 URL에 대해 HTTP 요청이 오면 원래 URL로 보내주기 위한 엔트포인트
          - **GET /api/v1/shortUrl**
          - 반환: `HTTP Redirection 목적지가 될 원래 URL`
  
  ### URL Redirection
  
  - **브라우저에 단축 URL 입력**
      
    <img src="https://github.com/user-attachments/assets/0da73d47-bfba-4737-85db-e347bca13878" width="400px"/>

      
      - 단축 URL을 받은 서버는 원래 URL로 바꾸어 **301 응답의 Location Header**에 넣어 반환
  - **클라이언트와 서버 사이의 통신 절차**
      
    <img src="https://github.com/user-attachments/assets/9f125601-536d-42aa-863c-df4716412d3a" width="400px"/>
      
      - **301 Permanently Moved**
          - 해당 URL에 대한 HTTP 요청의 처리 **책임이 영구적으로 반환된 URL로 이전**
          - 영구적 이전되었으므로, 브라우저는 이 **응답을 캐시(Cache)**
          - 같은 단축 URL에 요청 시, 캐시된 원래 URL로 요청
          - 서버 부하를 줄일 수 있음
      - **302 Found**
          - 주어진 URL로의 요청이 **일시적으로 Location Header가 지정하는 URL에 의해 처리**
          - 클라이언트 요청은 **언제나 단축 URL 서버에 먼저** 보내진 후 원래 URL로 Redirection
          - 트래픽 분석(Analytics)에 유리 : 클릭 발생률, 발생 위치 추적]
  - **해시 테이블**
      - URL Redirection을 구현하는 가장 직관적인 방법
      - 해시 테이블에 `<단축 URL, 원래 URL>` 쌍을 저장
      - `원래 URL = hashTable.get(단축 URL)`
      - 301 또는 302 응답 Location 헤더에 원래 URL을 넣은 후 전송
  
  ### URL 단축
  
  - 단축 URL : `www.tinyurl.com/{hashValue}`
  - **긴 URL을 이 해시 값으로 대응시킬 해시 함수 fx 찾기**
      
    <img src="https://github.com/user-attachments/assets/86af5524-1f9a-45f5-b114-3ac38dd9be92" width="400px"/>


      
      - 입력으로 주어지는 긴 URL이 다른 값이면 해시 값도 달라야 한다
      - 계산된 해시 값은 원래 입력으로 주어졌던 긴 URL로 복원될 수 있어야 한다
  
  ---
</details>
<details>
  <summary><b>상세 설계</b></summary>

  ---
  
  ## 3단계: 상세 설계
  
  ### 데이터 모델
  
  - 해시 테이블에 저장 : 메모리는 유한하고 비쌈
  - **관계형 데이터베이스에 저장**
      
    <img src="https://github.com/user-attachments/assets/c3c47a97-c873-44f2-9dce-a1dd40fb75da" width="400px"/>

      
  
  ### 해시 함수
  
  - 원래 URL을 단축 URL로 변환하기 위해 **해시 함수(hash function)**을 사용
      - 해시 함수가 계산하는 단축 URL 값을 `hashValue`라고 지칭
  - **해시 값 길이**
      - hashValue는 [0-9, a-z, A-Z]의 문자들로 구성
      - 사용할 수 있는 문자의 개수 10+26+26=62개
      - hashValue의 길이를 정하기 위해 $62^n ≥ 3650억$인 $n$의 최솟값을 찾아야 함, $n=7$
          
        <img src="https://github.com/user-attachments/assets/c006f68b-83a3-468c-8414-5dc3097bbca1" width="400px"/>

          
  - **해시 후 충돌 해소**
      - 긴 URL을 줄이려면 **원래 URL을 7글자 문자열로 줄이는 해시 함수**가 필요
      - 손쉬운 방법 : `CRC32`, `MD5`, `SHA-1` 같은 잘 알려진 해시 함수 활용
      - 예시) `https://en.wikipedia.org.wiki.Systems_design` 축약
          
        <img src="https://github.com/user-attachments/assets/f369f6c4-5de4-416b-9476-d4685ef9f2d8" width="400px"/>

          
          - 7문자로 더 줄여야 함
      - **계산된 해시 값에서 처음 7개 글자만 이용**
          - 해시 결과가 충돌할 확률이 높아짐
          - 충돌이 실제로 발생했을 때는 충돌이 해소될 때까지 사전에 정한 문자열을 해시값데 덧붙임
          
        <img src="https://github.com/user-attachments/assets/058169d2-3fd3-4202-9560-4bb885daa30a" width="400px"/>

          
          - 한번 이상 **데이터베이스에 질의**가 필요 : 오버헤드가 큼
          - 데이터베이스 대신 **블룸 필터**를 사용하면 성능을 높일 수 있음
          - **블룸 필터**
              - 집합에 특정 원소가 있는지 검사
              - 확률론에 기초한 공간 효율이 좋은 기술
  - **base-62 변환**
      - 진법 변환(base conversion) :URL 단축기를 구현할 때 흔히 사용되는 접근법 중 하나
      - 수의 표현 방식이 다른 두 시스템이 같은 수를 공유해야 하는 경우에 유용
      - 62진법을 쓰는 이유 : hashValue에 사용할 수 있는 문자(Character) 개수가 62개
      - ex) $11157_{10} → 2TX_{62}$
          
        <img src="https://github.com/user-attachments/assets/c4d2563b-2481-41e0-9fcd-db1acad188fc" width="400px"/>

          
          - 단축 URL : `https://tinyurl.com/2TX`
  - **두 접근법 비교**
      
    <img src="https://github.com/user-attachments/assets/5cf791ca-cdab-4e81-80fc-5d7f383b04fc" width="400px"/>

      
  
  ### URL 단축기 상세 설계
  
  - URL 단축기는 시스템의 핵심 컴포넌트
      - 처리 흐름이 논리적으로 단순해야 함
      - 언제나 동작하는 상태로 유지되어야 함
  - **URL 단축기 처리 흐름**
      
    <img src="https://github.com/user-attachments/assets/2616344f-5d8d-4cf8-b636-7a72006de5e5" width="400px"/>

      
      - 1️⃣ 입력으로 긴 URL을 받음
      - 2️⃣ 데이터베이스에 해당 URL이 있는지 검사
      - 3️⃣ 데이터베이스에 있다면, 단축 URL을 가져와 클라이언트에게 반환
      - 4️⃣ 데이터베이스에 없다면, 유일한 ID(기본 키)를 생성
      - **5️⃣ 62진법 변환**을 적용, ID를 단축 URL로 변환
      - 6️⃣ ID, 단축 URL 원래 URL로 새 DB 레코드 생성, 단축 URL 클라이언트에 전달
  - **예시)**
      
    <img src="https://github.com/user-attachments/assets/8d08f91c-d9ce-4c01-a2e5-18d505d83ab4" width="400px"/>

      
  
  ### URL 리디렉션 상세 설계
  
  - **URL Redirection 메커니즘**
      
    <img src="https://github.com/user-attachments/assets/b399aaeb-68cd-4a37-95b9-73fa8c285586" width="400px"/>

      
  - **캐싱(Caching) 적용**
      - 쓰기보다 읽기가 더 많은 시스템
      - `<단축 URL, 원래 URL>` 저장
  - **로드밸런서의 동작 흐름**
      - 1️⃣ 사용자가 단축 URL을 클릭
      - 2️⃣ 로드밸런서가 해당 클릭으로 발생한 요청을 웹 서버에 전달
      - 3️⃣ 단축 URL이 이미 캐시에 있는 경우, 바로 꺼내서 클라이언트에 전달
      - 4️⃣ 캐시에 해당 단축 URL이 없는 경우, 데이터베이스에서 조회
      - 5️⃣ 조회한 URL 레코드를 캐시에 저장한 후, 사용자에게 반환
      
  
  ---
</details>
<details>
  <summary><b>마무리</b></summary>

  ---
  
  ## 마무리
  
  ### 더 고려해볼 사항
  
  - **처리율 제한 장치(Rate Limiter)**
      - 엄청난 양의 URL 단축 요청이 밀려들 경우 무력화될 수 있는 잠재적 보안 결함
      - IP 주소를 비롯한 **필터링 규칙(Filtering Rule)**들을 이용해 요청을 걸러낼 수 있음
  - **웹 서버의 규모 확장**
      - 본 서버에 포함된 웹 계층은 무상태(Stateless) 계층
      - 웹 서버를 자유로이 증설하거나 삭제할 수 있음
  - **데이터베이스의 규모 확장**
      - 데이터베이스를 다중화하거나 샤딩(Sharding)하여 규모 확장성 달성
  - **데이터 분석 솔루션(Analytics)**
      - URL 단축기에 데이터 분석 솔루션 통합
      - 어떤 링크를 얼마나 많은 사용자가 클릭했는지 분석
      - 언제 주로 클릭헀는지 분석
  - **가용성, 데이터 일관성, 안정성**
      - 시스템이 성공적으로 운영되기 위해서 반드시 갖추어야 할 속성
  
  ---
</details>
