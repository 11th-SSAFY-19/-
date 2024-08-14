# URL 단축기 설계
고전적인 시스템 설계 문제인 tiny url과 같은 url 단축기 설계에 대해 얘기한다.

## 문제 이해 및 설계 범위 확정
기본 기능
- URL 단축 : 주어진 긴 URl을 훨씬 짧게 줄인다.
- URL 리디렉션 : 축약된 URl로 HTTP 요청이 오면 원래 URL로 안내
- 높은 가용성과 규모 확장성, 장애 감내 요구

개략적 추정
- 쓰기 연산 : 매일 1억개의 단축 URL 생성
- 초당 쓰기 연산 1억(100million) / 24 / 3600 = 1160
- 읽기 연산 : 읽기 연산과 쓰기 연산 비율은 10:1. 그 경우 읽기 연산은 초당 11,600회 발생한다. (1160 x 10 = 11,600)
- URL 단축 서비스를 10년간 운영한다고 가정하면 1억 x 365 x 10 = 3650억 개의 레코드를 보관해야 한다.
- 축약 전 URL의 평균 길이는 100.
- 10년 동안 필요한 저장 용량은 3650억 x 100바이트 = 36.5TB이다.

## 개략적 설계안 제시 및 동의 구하기
API 엔드 포인트, URL 리디렉션, URL 단축 플로에 대해 알아본다.

### 💡 API 엔드 포인트
- 클라이언트는 서버가 제공하는 API 엔드포인트를 통해 서버와 통신한다(REST 스타일)
1. URL 단축용 엔드 포인트 : 새 단축 URl을 생성하고자 하는 클라이언트는 이 엔드 포인트에 단축할 URL을 인자로 실어서 POST요청 보내야함.<br>
ex) `POST /api/v1/data/shorten`
인자 : {longURL : longURLstring}
반환 : 단축 URL

2. URL 리디렉션용 엔드포인트 : 단축 URL에 대해 HTTP 요청이 오면 원래 URL로 보내주기 위한 용도의 엔드포인트.<br>
ex) `GET /api/v1/shortUrl`
반환 : HTTP 리디렉션 목적지가 될 원래 URL

### 💡 URL 리디렉션
![image](https://github.com/user-attachments/assets/5ca23c5c-97b8-45ad-961d-9ae12780282a)
- 단축 URL을 받은 서버는 URL을 원래 URL로 바꾸어 301 응답의 LOcation 헤더에 넣어 반환한다.

![image](https://github.com/user-attachments/assets/3dee7734-ceb3-4142-80eb-a1d43a5924b9)
- 유의해야할 점은 301과 302응답의 차이다.

✨ **301 Permanently Moved**
- 해당 URL에 대한 HTTp 요청의 책임이 영구적으로 Location 헤더에 반환된 URL로 이전되었다는 응답이다.
- 브라우저는 이 응답을 캐시하고, 추후 단축 URl에 요청을 보낼 필요가 있을 때 브라우저는 캐시된 원래 URL로 요청을 보내게 된다.

✨ **302 Found**
- 요청이 일시적으로 Location 헤더가 지정하는 URL에 의해 처리되어야 한다는 응답이다.
- 언제나 단축 URL 서버에 먼저 보내진 후에 원래 URL로 리디렉션 되어야한다.

-> 301은 서버 부하를 줄이는데 좋고, 302는 트래픽 분석이 중요할 때 좋다.


### 💡 URL 단축
- 단축 URL이 `www.tinyurl.com/{hashValue}`같은 형태라고 하면 URL을 해시 값으로 대응시킬 해시 함수 fx를 찾는 일이다.
![image](https://github.com/user-attachments/assets/1cd604d8-4ec8-4e1f-9dee-cf10e9b13864)
- 입력으로 주어진 긴 URL이 다른 값이면 해시값도 달하야한다.
- 계산된 해시 값은 원래 입력으로 주어진 긴 URL로 복원될 수 있어야한다.

## 상세 설계
URL을 단축하는 방법과 리디렉션 처리에 관계된 설계안에서 실제 구체적인 설계안을 제시한다.

### 💡 데이터 모델
- 모든 값을 해시 테이블에 두었었는데 실제 시스템에 사용하기에는 힘들다(메모리는 유한한데다 비싸기 때문)
- 따라서, 순서쌍을 관계형 데이터베이스에 저장하는 것이 더 좋다.
![image](https://github.com/user-attachments/assets/7588e832-3014-4fd2-b0ff-a38b6da836ef)

### 💡 해시 함수
- 원래 URL을 단축 URL로 변환하는 데 쓰인다. 

✨ **해시 값 길이**
- hashValue는 숫자, 영어의 문자들로 구성된다. 따라서 사용가능한 문자는 62개이다.
- hashValue의 길이를 정하기 위해서는 3650억인 n의 최소값을 찾아야한다.
- 해시함수가 만들 수 있는 URL의 개수사이의 관계를 나타내는 표이다.
![image](https://github.com/user-attachments/assets/152dd6bc-251d-4a95-9136-e58b6dbc53bc)

- n=7이면 3.5조개의 URL을 만들 수 있으므로 요구사항을 만족시킬 수 있다.
- 해시 함수 구현에 쓰일 기술은 `해시후 충돌 해소` 방법과 `base-62변환`방법이 있다.

✨ **해시 후 충돌 해소**
- 긴 URL을 줄이려면 원래 URL을 7글자 문자열로 줄이는 해시 함수가 필요한데 `CRC32`, `MD5`, `SHA-1`등과 같은 방법을 사용할 수 있다.

![image](https://github.com/user-attachments/assets/ea14e294-2e10-4157-bb43-8b536571bffb)
- CRC가 계산한 가장 짧은 해시 값조차도 7보다 길기 때문에 이를 줄여야한다.

![image](https://github.com/user-attachments/assets/d380c1b8-8a78-4ca9-a414-4482c71358fe)
- 1번방법은 처음 7개 글자만 이용하는 것이다. 하지만 충돌 확률이 높아진다.
- 따라서, 충돌이 실제로 발생하였을 때 충돌이 해소될 때까지 사전에 정한 문자열을 해시값에 덧붙인다.
- 이 방법을 쓰면 단축 URL을 생성할 때 한번 이상 데이터베이스 질의를 해야 하므로 오버헤드가 크다.
- DB대신 블룸 필터를 사용하면 성능을 높일 수 있다.

✨ **base-62 변환**
- hashValue에서 사용할 수 있는 문자의 개수가 62개 이기 때문에 진법을 변환할 수 있다.

✨ **두 접근법 비교**
![image](https://github.com/user-attachments/assets/91e5b497-79eb-47fe-b2f7-52512615fa88)

### 💡 URL 단축

