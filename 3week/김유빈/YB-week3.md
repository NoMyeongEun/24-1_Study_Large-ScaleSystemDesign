# 4장 처리율 제한 장치의 설계
4장 키워드: `처리율 제한 장치의 설계`
## 1. 문제 이해 및 설계 범위 확정
면접관과의 소통을 통한 제한 장치 구현의 명확성

## 2. 개략적 설계안 제시 및 동의 구하기 
### 처리율 제한 장치는 어디에 둘 것인가
- 처리율 제한 장치: 클라이언트 측에 두는것보다 서버측에 두는것이 좋다. (클라이언트 요청은 쉽게 위변조가 가능)
- 처리율 제한 미들웨어를 만들어 요청을 통제

### 처리율 제한 알고리즘
**토큰 버킷 알고리즘**
- 간단. 아마존/스트라이프가 사용
- 주기적으로 버킷안에 토큰이 채워진다.(꽉 찬 경우 더 이상 채워지지 않으며, 추가로 공급된 코튼은 overflow로 버려진다.) 요청이 도착하면 토큰이 있는지 검사, 요청이 처리될 때마다 하나의 토큰을 사용해서 시스템 요청을 전달한다. 토큰이 없다면 토큰이 채워질때까지 오는 API 요청은 버려진다.(dropped)

장점: 구현이 쉽고 메모리 사용 측면에서 효율적이다. 짧은 시간에 집중되는 트래픽도 처리 가능하다.
단점: 버킷 크기와 토큰 공급률 이 두 인자를 적절하게 튜닝하는게 쉽지 않다.

**누출 버킷 알고리즘**
- 요청 처리율이 고정. FIFO로 구현
- 요청이 도착하면 큐가 가득 차 있는지 본다. 빈자리가 있는 경우에는 큐에 요청을 추가한다. 
- 큐가 가득 차 있는 경우에는 새 요청은 버린다. 
- 지정된 시간마다 큐에서 요청을 꺼내어 처리한다.

장점: 메모리 사용량 측면에서 효율적이다. 고정된 처리율 -> 안정적 출력이 필요한 경우 적합하다.
단점: 단시간에 많은 트레픽이 모이면 최신 요청들이 버려진다. 두 개의 인자를 올바르게 튜닝하기 까다롭다.

**고정 윈도 카운터 알고리즘**
- 타임라인을 고전된 강격의 윈도로 나누고, 각 윈도마다 카운터를 붙인다.
- 요청이 접수될 때마다 이 카운터의 값은 1씩 증가한다.
- 이 카운터의 값이 사전에 설정된 임계치에 도달하면 새로운 요청은 새 윈도가 열릴 때까지 버려진다.

장점: 메모리 효율이 좋으며 이해하기 쉽다. 윈도가 닫히는 시점에 카운터를 초기화하는 방식은 특정한 트래픽 패턴을 처리하기에 적합하다.
단점: 윈도 경계 부근에서 일시적으로 많은 트래픽이 몰리는 경우, 기대했던 시스템의 처리 한도보다 많은 양의 요청을 처리하게 된다.

**이동 윈도 로깅 알고리즘**
- 요청의 타임스탬프를 추적한다. 타임스탬프 데이터는 보통 레디스의 정렬 집합 같은 캐시에 보관한다.
- 새 요청이 오면 만료된 타임스탬프는 제거한다. 만료된 타임스탬프는 그 값이 현재 윈도의 시작 지점보다 오래된 스탬프를 말한다.
- 새 요청의 타임스탬프를 로그에 추가한다.
- 로그의 크기가 혀용치보다 같거나 작으면 요청을 시스템에 전달한다. 그렇지 않은 경우, 처리를 거부한다.

장점: 처리율 제한 메커니즘은 아주 정교하다. 어느 순간의 윈도를 보더라도, 허용되는 요청의 개수는 시스템의 처리율 한도를 넘지 않는다.
단점: 거부된 요청의 타임스탬프도 보관하기 때문에 다량의 메모리를 사용한다.

### 개략적인 아키텍처
> 얼마나 많은 요청이 접수되었는지를 추적할 수 있는 카운터를 추적 대상별로 두고(사용자별로 추적할 것인가? IP 주소별로? API 엔드포인트나 서비스 단위로?), 이 카운터의 값이 어떤 한도를 넘어서면 한도를 넘어 도착한 요청은 거부하는 것
- 클라이언트가 처리율 제한 미들웨어(rate limiting middleware)에게 요청을 보낸다.
- 처리율 제한 미들웨어는 레디스의 지정 버킷에서 카운터를 가져와서 한도에 도달했는지 아닌지를 검사한다.
    - 한도에 도달했다면 요청은 거부된다.
    - 한도에 도달하지 않았다면 요청은 API 서버로 전달된다. 한편 미들웨어는 카운터의 값을 증가시킨 후 다시 레디스에 저장한다.

## 3. 상세 설계
> 처리율 제한 규칙은 어떻게 만들어지고 어디에 저장되는가? / 처리가 제한된 요청들은 어떻게 처리되는가?
### 처리율 한도 초과 트래픽의 처리
어떤 요청이 한도 제한에 걸리면 API는 HTTP 429 응답(too many requests)을 클라이언트에게 보낸다.
경우에 따라서는 한도 제한에 걸린 메시지를 나중에 처리하기 위해 큐에 보관할 수도 있다.

**처리율 제한 장치가 사용하는 HTTP 헤더**
- 클라이언트는 자기 요청이 처리율 제한에 걸리고 있는지 어떻게 감지할 수 있는가?
- 자기 요청이 처리율 제한에 걸리기까지 얼마나 많은 요청을 낼 수 있는지 어떻게 알 수 있는가?

-> HTTP 응답 해더를 통해!

- X-Ratelimit-Remaining: 윈도 내에 남은 처리 가능 요청 수
- X-Ratelimit-Limit: 매 윈도마다 클라이언트가 전송할 수 있는 요청의 수
- X-Ratelimit-Retry-After: 한도 제한에 걸리지 않으려면 몇 초 뒤에 요청을 다시 보내야하는지 알림

### 분산 환경에서의 처리율 제한 장치의 구현
**경쟁 조건**
처리율 제한 장치는 다음과 같이 동작한다.
- 레디스에서 카운터의 값을 읽는다.
- counter + 1의 값이 임계치를 넘는지 본다.
- 넘지 않는다면 레디스에 보관된 카운터 값을 1만큼 증가시킨다.

[병행성이 심한 환경에서의 경쟁 조건 이슈]  
- 레디스에 저장된 변수 counter의 값이 3이라고 가정하자.
- 두 개 요청을 처리하는 스레드가 각각 병렬로 counter값을 읽었으며, 그 둘 가운데 어느 쪽도 아직 변경된 값을 저장하지는 않은 상태라 가정하자.
- 둘 다 다른 요청의 처리 상태는 상관하지 않고 counter에 1을 더한 값을 레디스에 기록할 것이며, counter의 값은 올바르게 변경되었다고 믿을 것이다.

    -> **사실 counter의 값은 5가 되어야 한다.**

- 경쟁 조건 문제를 해결하는 가장 널리 알려진 해결책은 락(lock)이다. 하지만 락은 시스템의 성능을 상당히 떨어뜨린다는 문제가 있다.

[해결할 수 있는 방법]
- 루아 스크립트(Lua Script)
- 정렬 집합(sorted set)이라 불리는 레디스 자료구조를 사용하는 것

**동기화 이슈**
- 분산 환경에서 고려해야 할 또 다른 중요한 요소이다.
- 수백만 사용자를 지원하려면 한 대의 처리율 제한 장치 서버로는 충분하지 않을 수 있음 -> 처리율 제한 장치 서버를 여러 대 두게 되면 동기화가 필요해진다.
- 웹 계층은 무상태(stateless)이므로 클라이언트는 다음 요청을 오른쪽 그림처럼 각기 다른 제한 장치로 보내게 될 수 있다.
    - 이 때 동기화를 하지 않는다면 제한장치 1은 클라이언트 2에 대해서는 아무것도 모르기 때문에 처리율 제한을 올바르게 수행할 수 없을 것이다.

[해결책]
- 고정 세션(sticky session)을 활용하여 같은 클라이언트로부터의 요청은 항상 같은 처리율 제한 장치로 보낼 수 있도록 하는 것
    - Sticky Session: 특정 세션의 요청을 처음 처리한 서버로만 전송하는 것을 의미한다.
    - 그러나 이 방법은 규모면에서 확장 가능하지도 않고 유연하지도 않다.
- 더 나은 방법은 레디스와 같은 중앙 집중형 데이터 저장소를 쓰는 것이다. (token과 같은 작업 등)

**성능 최적화**
[지금까지 살펴본 설계의 두 가지 개선책]
1. 여러 데이터 센터를 지원하는 문제는 처리율 제한 장치에 매우 중요한 문제이다.
- 데이터 센터에서 멀리 떨어진 사용자를 지원하려다보면 지연시간이 증가할 수밖에 없기 때문이다.
- 대부분의 클라우드 서비스 사업자는 세계 곳곳에 에지 서버(edge server)를 심어놓고 있다.
2. 제한 장치 간에 데이터를 동기화할 때 최종 일관성 모델(eventual consistency model)을 사용한다.

**모니터링**
- 처리율 제한 장치를 설치한 이후에는 효과적으로 동작하고 있는지 보기 위해 데이터를 모을 필요가 있다.
- 기본적으로 모니터링을 통해 확인하려는 것은 다음과 같은 두 가지이다.
    1. 채택된 처리율 제한 알고리즘이 효과적이다.
    2. 정의한 처리율 제한 규칙이 효과적이다.

깜짝 세일 같은 이벤트로 인해 트래픽이 급증할 때 처리율 제한 장치가 비효율적으로 동작한다면? -> 토큰 버킷이 적합

## 4. 마무리
- 경성(hard) 또는 연성(soft) 처리율 제한
    - 경성 처리율 제한: 요청의 개수는 임계치를 절대 넘어설 수 없다.
    - 연성 처리율 제한: 요청 개수는 잠시 동안은 임계치를 넘어설 수 있다.
- 다양한 계층에서의 처리율 제한
    - 이번 장에서는 애플리케이션 계층(HTTP: OSI 네트워크 계층도 기준으로 7번)에서의 처리율 제한에 대해서만 살펴보았지만 다른 계층에서도 처리율 제한이 가능하다.  
    ex. IPtables를 사용하면 IP 주소에 대한 처리율 제한을 적용하는 것이 가능하다(3번 계층)
- 처리율 제한을 회피하는 방법, 클라이언트를 어떻게 설계하는 것이 최신인가?
    - 클라이언트 측 캐시를 사용하여 API 호출 수를 줄인다.
    - 처리율 제한의 임계치를 이해하고, 짧은 시간동안 너무 많은 메시지를 보내지 않도록 한다.
    - 예외나 에러를 처리하는 코드를 도입하여 클라이언트가 예외적 상황으로부터 우아하게(gracefully) 복구될 수 있도록 한다.
    - 재시도(retry) 로직을 구현할 땐 충분한 백오프(back-off) 시간을 둔다.

# 5장 안정해시 설계
5장 키워드: `요청이나 데이터를 서버에 균등하게 나누는 법`

## 안정 해시
안정해시: 분산되어 있는 서버 혹은 서비스에 데이터를 균등하게 나누기 위한 기술로, k/n의 키만 재배치.

**해시 공간과 해시 링**
해시 함수 f로 SHA-1을 사용한다고 가정할 떄
이 함수의 출력 값 범위는 X0 ~ Xn이 라고 하면, SHA-1의 해시 공간 범위는 0 ~ 2^160 - 1이므로 X0는 0, Xn은 2^160 - 1이고 나머지 X1부터 Xn-1까지는 그 사이 값이 된다.

## 기본구현법
- 서버와 키를 균등 분포 해시 함수를 사용해 해시 링에 배치한다.
- 키의 위치에서 링을 시계방향으로 탐색 후 최초 발견 서버가 키가 저장될 서버로 선택한다.

**기본 구현법의 두 가지 문제**
- 서버와 키를 균등 분포 해시 함수를 사용해 해시 링에 배치한다.
- 키의 위치에서 링을 시계 방향으로 탐색하다 만나는 최초의 서버가 키가 저장될 서버다.

## 가상노드 구현
위의 2가지 문제점을 해결하기 위한 기법

실제 노드 또는 서버를 가리키는 노드, 하나의 서버는 링 위에 여러 개의 가상 노드를 가질 수 있다.
- 서버를 배치하기 위해 아래와 같이 여러 개의 가상 노드를 사용한다.

ex) 세 개의 가상 노드 사용 [ s0_0, s0_1, s0_2 ]
- 키가 저장되는 서버: 키의 위치로부터 시계방향을 탐색하다 만나는 최초의 가상 노드

가상 노드 수를 늘리면 분포는 더 균등해지지만 저장할 공간이 많이 필요함으로 타협적 결정이 필요.

## 안정해시가 주는 이로움
- 서버 수의 변경이 있을 때 재배치되는 키의 수 최소화
- 핫스팟 키 문제 감소
    - 특정한 샤드에 대한 접근이 지나치게 빈번한 경우를 방지