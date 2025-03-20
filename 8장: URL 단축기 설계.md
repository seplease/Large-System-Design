8장: URL 단축기 설계
=
## 1단계: 문제 이해 및 설계 범위 확정
### 개략적 추정
- 쓰기 연산: 매일 1억 개의 단축 URL 생성
- 초당 쓰기 연산: 1억(100million)/24/3600 = 1160
- 읽기 연산: 읽기 연산과 쓰기 연산 비율은 10:1이라고 하자. 그 경우 읽기 연산은 초당 11,600회 발생
- URL 단축 서비스를 10년간 운영한다고 가정하면 1억(100million)*365*10 = 3650억(365billion) 개의 레코드를 보관해야 함
- 축약 전 URL의 평균 길이는 100.
- 따라서 10년 동안 필요한 저장 용량은 3650억(365billion)*100바이트 = 36.5TB

## 2단계: 개략적 설계안 제시 및 동의 구하기
### API 엔드포인트
URL 단축기는 기본적으로 두 개의 엔드포인트를 필요로 함
1. URL 단축용 엔드포인트
  - 새 단축 URL을 생성하고자 하는 클라이언트는 이 엔드포인트에 단축할 URL을 인자로 실어서 POST 요청을 보내야 함<br>
  <img width="238" alt="image" src="https://github.com/user-attachments/assets/eac00a09-75e1-44bb-8839-97ead2debc72" /><br>
2. URL 리디렉션용 엔드포인트
  - 단축 URL에 대해서 HTTP 요청이 오면 원래 URL로 보내주기 위한 용도의 엔드포인트<br>
  <img width="327" alt="image" src="https://github.com/user-attachments/assets/b955815c-9a46-4582-93df-690d45d037f5" /><br>

### URL 리디렉션
<img width="532" alt="image" src="https://github.com/user-attachments/assets/b0eb3d08-8a4c-4394-96f6-1e36d8896099" /><br>
- 브라우저에 단축 URL을 입력하면 생기는 과정
- 단축 URL을 받은 서버는 그 URL을 원래 URL로 바꿔서 301 응답의 Location 헤더에 넣어 반환함<br>
<img width="387" alt="image" src="https://github.com/user-attachments/assets/cf702f82-e178-44c4-9b79-360aa8c7d092" /><br>
* 301/302 응답의 차이를 유의해야 함
- 301 Permanently Moved
  - 해당 URL에 대한 HTTP 요청의 처리 책임이 영구적으로 Location 헤더에 반환된 URL로 이전되었다는 응답
  - 영구적으로 이전되었으므로, 브라우저는 이 응답을 캐시함
  - &rarr; 추후 같은 단축 URL에 요청을 보낼 필요가 있을 때 브라우저는 캐시된 원래 URL로 요청을 보내게 됨
- 302 Found
  - 주어진 URL로의 요청이 '일시적으로' Location 헤더가 지정하는 URL에 의해 처리되어야 한다는 응답
  - &rarr; 클라이언트의 요청은 언제나 단축 URL 서버에 먼저 보내진 후에 원래 URL로 리디렉션되어야 함
- 두 방법은 각기 다른 장단점을 가짐
  - 첫 번째 요청만 단축 URL 서버로 전송될 것이기 때문에, 서버 부하를 줄이는 것이 중요하다면 301 Permanent Moved
  - 트래픽 분석이 중요할 때는 302 Found를 쓰는 쪽이 클릭 발생률이나 발생 위치를 추적하는 데 유리함
- URL 리디렉션을 구현하는 가장 직관적인 방법은 해시 테이블을 사용하는 것. 해시 테이블에 <단축 URL, 원래 URL>의 쌍을 저장한다고 가정하면, URL 리디렉션은 아래와 같이 구현될 수 있음<br>
  <img width="435" alt="image" src="https://github.com/user-attachments/assets/0ccdc61c-51a9-4984-80e0-b732cebad2b2" /><br>

### URL 단축
- 단축 URL이 `www.tinyurl.com/{hashValue}` 같은 형태라고 해보자. 중요한 것은 긴 URL을 이 해시 값으로 대응시킬 해시 함수 fx를 찾는 일.<br>
  <img width="223" alt="image" src="https://github.com/user-attachments/assets/fbcca1f8-e1bf-4fe7-9631-a985987f5cee" /><br>
  - 이 해시 함수는 다음 요구사항을 만족해야 함
    - 입력으로 주어지는 긴 URL이 다른 값이면 해시 값도 달라야 함
    - 계산된 해시 값은 원래 입력으로 주어졌던 긴 URL로 복원될 수 있어야 함

## 3단계: 상세 설계
### 데이터 모델
<img width="169" alt="image" src="https://github.com/user-attachments/assets/4f5ce72a-6728-4e73-b244-cf2baf7ddfef" /><br>
- `<단축 URL, 원래 URL>`의 순서쌍을 관계형 데이터베이스에 저장

### 해시 함수
해시 함수는 원래 URL을 단축 URL로 변환하는 데 쓰임
**편의상 해시 함수가 계산하는 단축 URL 값을 hashValue라고 지칭**

#### 해시 값 길이
- hashValue는 [0-9, a-z, A-Z]의 문자로 구성됨 &rarr; 사용할 수 있는 문자 개수는 62개
- hashValue의 길이를 정하기 위해서는 62^n >= 3650억(365billion)인 n의 최솟값을 찾아야 함 (개략적 추정치에 따르면 이 시스템은 3650억 개의 URL을 만들어 낼 수 있어야 함)
- hashValue의 길이와, 해시 함수가 만들 수 있는 URL의 개수 사이의 관계<br>
  <img width="287" alt="image" src="https://github.com/user-attachments/assets/80548ce4-5ac7-4266-97f6-afe729d5208d" /><br>

#### 해시 후 충돌 해소
- 긴 URL을 줄이려면, 원래 URL을 7글자 문자열로 줄이는 해시 함수가 필요한데, 손쉬운 방법은 `CRC32`, `MD5`, `SHA-1`과 같이 잘 알려진 해시 함수를 이용하는 것
- 이 함수들을 사용하여 URL을 줄여도, CRC32가 계산한 가장 짧은 해시값조차도 7보다는 길다는 문제 발생
- 이 문제를 해결하기 위한 첫 번째 방법은 계산된 해시 값에서 처음 7개 글자만 이용하는 것 &rarr; 이렇게 하면 해시 결과가 서로 충돌할 확률이 높아짐 &rarr; 실제로 충돌이 발생했을 때는, 충돌이 해소될 때까지 사전에 정한 문자열을 해시값에 덧붙임<br>
<img width="537" alt="image" src="https://github.com/user-attachments/assets/eb96c0cb-8489-484e-84b7-b0f8bd8bba67" /><br>
- 이 방법을 쓰면 충돌을 해소할 수 있지만 단축 URL을 생성할 때 한 번 이상 데이터베이스 질의를 해야 하므로 오버헤드가 큼. &rarr; 데이터베이스 대신 블룸 필터를 사용하면 성능 향상 가능
- ** 블룸 필터는 어떤 집합에 특정 원소가 있는지 검사할 수 있도록 하는, 확률론에 기초한 공간 효율이 좋은 기술임

#### base-62 변환
- 진법 변환. URL 단축기를 구현할 때 흔히 사용된은 접근법 중 하나. 이 기법은 수의 표현 방식이 다르두 시스템이 같은 수를 공유하여야 하는 경우에 유용함
- 62진법을 쓰는 이유: hashValue에 사용할 수 있는 문자 개수가 62개이기 때문<br>
<img width="536" alt="image" src="https://github.com/user-attachments/assets/7421409d-042a-4a1c-8894-75ec4ed47ca5" /><br>

#### 두 접근법 비교
<img width="531" alt="image" src="https://github.com/user-attachments/assets/9f8f70ee-5a5d-4a83-a6ee-3cfa4694bc0e" /><br>

### URL 단축기 상세 설계
- URL 단축기는 시스템의 핵심 컴포넌트이므로, 그 처리 흐름이 논리적으로 단순해야 하고 기능적으로는 언제나 동작하는 상태로 유지되어야 함
- 본 예제에서는 62진법 변환 기법을 사용해 설계 예정
- 처리 흐름 순서도<br>
<img width="422" alt="image" src="https://github.com/user-attachments/assets/d0aede01-1021-4857-928b-ee7174ef6a7e" /><br>
<img width="543" alt="image" src="https://github.com/user-attachments/assets/86cb646d-dd7a-414e-a48a-b13125b16c93" /><br>

- 예제<br>
<img width="506" alt="image" src="https://github.com/user-attachments/assets/c57c9edd-df37-4ca4-827e-33c526afe91f" /><br>

**이 생성기의 주된 용도는, 단축 URL을 만들 때 사용할 ID를 만드는 것이고, 이 ID는 전역적 유일성이 보장되어야 하는 것**

### URL 리디렉션 상세 설계
- 쓰기보다 읽기를 더 자주하는 시스템<br>
<img width="537" alt="image" src="https://github.com/user-attachments/assets/cf951b4a-bd79-45e5-810a-49695aaa1658" /><br>
- 로드밸런서 동작 흐름<br>
<img width="545" alt="image" src="https://github.com/user-attachments/assets/263db22d-5c7b-4357-8799-d9eff696da64" /><br>

## 4단계: 마무리
### 처리율 제한 장치(rate limiter)
- 처리율 제한 장치를 두면, IP 주소를 비롯한 필터링 규칙들을 이용해 요청을 걸러낼 수 있을 것임

### 웹 서버의 규모 확장
- 본 설계에 포함된 웹 계층은 무상태(Stateless) 계층이므로, 웹 서버를 자유로이 증설하거나 삭제 가능

### 데이터베이스의 규모 확장
- 데이터베이스를 다중화하거나 샤딩하여 규모 확장성 달성 가능

### 데이터 분석 솔루션
- URL 단축기에 데이터 분석 솔루션을 통합해 두면 어떤 링크를 얼마나 많은 사용자가 클릭했는지, 언제 주로 클릭했는지 등 중요한 정보를 알아낼 수 있을 것임

### 가용성, 데이터 일관성, 안정성
