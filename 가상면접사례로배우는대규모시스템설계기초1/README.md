# 가상 면접 사례로 배우는 대규모 시스템 설계 기초1

워낙에 유명한 책이고 나도 뜨문뜨문 봤던 책이라 정리하면서 읽어볼까해서 적어본다.

## 2장. 개략적인 규모 추정

- 설계 면접에서는 시스템 용량이나 성능 요구사항을 개략적으로 추정해보라는 요구를 받게 되는데 이게 "개략적인 규모 추정" 이라고 책에서 정의내리고 있다.
- 개략적 규모 추정은 정밀한 측정이 아니기에 추정값이고 면접자의 논리력이 뒷받침되어야 하는데 논리의 근거가 되는 소스들이 "2의 제곱수", "응답지연(latency)", "가용성에 관계된 수치"들이다.
  - 수치는 책을 통해 확인해보자!
- 읽으면서 '아맞다' 했던 부분이 "압축"이었다. 압축을 통해서 데이터 크기를 줄이고 보내는 응답지연을 줄이는 방법에 대해선 간과했던 부분이었던 것 같다.
- QPS : Query Per Second

## 3장. 시스템 설계 면접 공략법

- 문제 이해 및 설계 범위 확정
  - 질문을 통해 모호함을 없애고 스타트!
- 개략적인 설계안제시 및 동의 구하기
- 상세 설계
- 마무리

## 4장. 처리율 제한 장치의 설계

- rate limiter는 말그대로 트래픽의 처리율(Rate)를 제어하는 장치다
- 제어함으로써 DDoS 공격도 막을 수 있고, 서버의 과부하 또한 막을 수 있을 것이다.
  - 예시로, 네이버 메일 페이지에서 새로고침 50번쯤하면 조회가 제한되는 것을 볼 수 있다.
- 문제 이해 및 설계 범위 확정!
  - 클라이언트측 제한? 서버측 제한?
  - 어떤 기준으로 제한? endpoint? IP? 사용자ID? etc
  - 설계한 시스템의 내구성(tolerance)은 어느정도?
  - 분산환경에서 동작?
- 개략적 설계안 제시 및 동의 구하기
  - 읽으면서 좋았던 부분은 "클라이언트 측 rate limiter는 위변조가 가능하기때문에..." 부분이다. 내가 간과하고 있던 부분이다. 설계를 할 때에 rate limiter는 클라이언트가 아닌 서버에 두는 것이 좋다.
  - rate limit 알고리즘에는 여러 가지가 있는데 그들중 각각의 장단점을 비교해 선택하면 좋다.
    - Rate limiter로 많이 쓰이는 bucket4j, resilience4j 라이브러리 모두 토큰버킷 알고리즘을 사용하고 있다. 물론 이게 제일 좋아서는 아니겠지만 간단한 구현방식과 트래픽 과부하에도 내성이 있기 때문에 많이 차용되는 것이 아닐까 싶다.
    - 이건 몰랐는데 Redis로도 많이 구현하는 것 같다.
- 상세 설계
  - 만약 분산환경에서 rate limiter를 레디스로 구현한다했을 때, 문제가 생길 수 있는데 이건 "경쟁조건"과 "동기화" 이다.
  - 이런 문제를 바로 해결할 수 있는 방법은 사실 "락"인데 이는 너무 많은 성능 저하를 불러올 수 있다. 다만 트레이드오프로 쓰기의 성능은 크게 개의치않고, 데이터의 조회 성능과 조회 시 데이터의 정합성이 더욱 큰 이슈라면 쓰기 성능을 포기할수도 있다.
  - 락으로 쓰기 성능도 포기하고 싶지 않다면 레디스의 "루아 스크립트(Lua script)", "정렬 집합(sorted set)"을 활용하는 것도 방법이다.
  - 만약 rate limiter를 분산형으로 운영하고 싶다면 중앙집중형으로 redis를 활용하는 것 또한 방법이다. sticky session 방식을 사용하는것은 확장성에 좋지 않다는 필자의 의견에 동의한다!

## 5장. 안정 해시 (Consistent hash) 설계

- 서버의 수평적 확장(scale out)을 위해선, 요청을 균등하게 분배시켜주는 "안정해시"가 필요하다.
  - 개인적인 생각으론 "라운드로빈" 방식처럼 단순한 방법을 사용하면 가능하지만, 동일한 스펙이 아닌 서버를 수평적 확장할 때에는 "안정해시" 기술을 통해 수평확장하는 것 같다. (ex. DB Sharding, Cache Sharding)
- 일반 해시 함수와 모듈러 연산(% N (서버 수)) 을 통해 요청을 분배하고 있다면, 추후 서버 수가 줄어들거나 추가될 때 문제가 생긴다. = 해시 키 재배치 문제 = rehash
- 안정해시(Consistent hash)
  - 해시 테이블 크기가 고정되어 있을 때, 오직 k/n개의 키만 재배치하는 기술 (k : 키의 개수, n : 슬롯 개수)
  - 일반 해시는 슬롯의 수가 바뀌면 모든 키를 재배치하게 된다

> 해시 관련 용어
>
> - 해시공간 : 해시 함수를 통해 생성되는 모든 가능한 결과 값의 집합
> - 해시 테이블 : 해시 테이블은 해시 함수를 사용하여 데이터를 저장하고 검색하는 데 사용되는 자료 구조
> - 해시 링 : 원형 모양의 해시 공간. 분산 캐싱 시스템, 분산 DB 등에서 요청을 고르게 분산시킬 때 사용된다

- 안정해시도 만능은 아니다!

  - 서버 추가/삭제되는 상황에서, 균등한 분배를 유지하기 어렵다. 파티션 크기가 서버마다 다르다.
  - 키의 균등 분포를 달성하기 어렵다. 보관 데이터의 양이 서버마다 다르다.

- 문제 해결방법!

  - 가상노드 (virual node) or 복제 (Replica) 기법
    - 실제 서버가 1개라면, 가상노드는 여러 개를 만듦으로써 키의 분포를 균등하게한다. 가상노드가 많을수록 키의 분포는 점점 더 균등해진다. 다만 관리포인트가 많아지고 복잡해진다는 단점이 있는 것 같다.

- 안정해시가 쓰이는 기술들
  - AWS DynamoDB
  - Apache Cassandra 클러스터에서의 데이터 파티셔닝
  - 디스코드
  - Akamai CDN
  - Meglev Network Load Balancer

## 6장. 키-값 저장소 설계

- 문제이해 및 설계 범위 확정
  - 고가용성
  - 키-값 쌍 크기는 10KB 이하
  - 큰 데이터 저장 가능
  - 확장성 (auto scaling)
  - 데이터 일관성 수준은 조정 가능
  - latency 짧음
- 단일 서버로 키-값 저장소 설계는 쉽다. 단순히 키-값 쌍을 모두 메모리에 해시테이블로 저장하면된다. 다만 이렇게 되면 메모리를 많이 차지하게 되는 문제가 있다.
  - 개선점으로는 자주 안쓰이는 데이터는 디스크에 저장해두는 것이다. 레디스의 스냅샷같은?
- 분산 키-값 저장소(분산 해시테이블)
  - CAP 정리 : 일관성, 가용성, 파티션 감내성 모두 동시에 만족하는 시스템 설계는 불가능하다
  - 두가지만 만족할 수 있다.
  - CP : 가용성 희생
  - AP : 일관성 희생
  - CA : 파티션감내 희생. 네트워크 에러는 항상 있기에 해당 시스템은 선택되지 않음.
- 데이터를 다중화해서 저장한 뒤, 쓰기/읽기 연산 이전에 낡은 데이터를 읽지 못하도록 강한 일관성 모델을 채택할 수 있지만, 이는 가용성에 타격을 준다.
- 그래서 데이터를 다중화해서 저장한다면 일관성이 깨질 수 있는데, 책에서는 비일관성 해소 기법 중 하나인 "데이터 버저닝"를 소개하고 있다.
- 버저닝은 데이터를 변경할 때마다 해당 데이터의 새로운버전을 만드는 것으로 각 버전의 데이터는 변경 불가능하게 만든다. S3의 객체 버저닝과 비슷한 개념이다!
  - 이런 문제를 해결하는 데에 "벡터 시계(vector clock)"이 있다.
  - 벡터 시계는 [서버, 버전] 순서쌍을 데이터에 매단 것이다. HTTP 헤더처럼.
  - 충돌이 있는지 판단하는 방법은, A의 벡터 시계 구성요소 가운데 B의 벡터 시계 동일 서버 구성요소보다 작은 값을 갖는 것이 있는지 보면 된다.
  - "벡터 시계" 단점
    - 충돌을 다루는 클라이언트 구현이 복잡해짐
    - [서버, 버전] 순서쌍이 무한정 쌓이면서 메모리를 많이 차지하게 됨
- 장애 감지하는 방법
  - 멀티캐스팅 채널을 구축. > 이 방법은 서버 많을 때는 비효율적이라고함.
    - 모든 서버에 변경사항을 전달하기에 많은 수의 서버 환경에서는 비효율적인 선택이다
  - 가십 프로토콜을 많이 선택한다고 함
    - 모든 서버에 캐스팅하기보단, log N개의 서버에만 전달하고 전달받은 서버들에서 또 다시 전달하는 방식으로 구현. 이름처럼 '소문이 퍼진다~' 라고 생각하면 됨. 각 서버에서는 log N개의 응답만 체크해주면 가능하기에 확장성이 좋다.
- 일시적 장애 처리
  - 서버가 잠깐 죽었다고 아예 서버 기능을 못하는건 아니니, 서버 복구됐을 때의 액션도 있어야한다.
  - strict quorum 접근법은 데이터 일관성을 지키기 위해 쓰기/읽기를 막는 방법인데 이건 모든 분산 시스템에서 도입하지 않을 것이다
  - sloppy quorum 접근법은 조건을 완화해 가용성을 높이는 것이다. 쓰기 연산 수행할 W개의 online 서버와 읽기 연산 수행할 R개의 online 서버를 해시 링에서 골라 읽기/쓰기를 처리한다. 그 이후 복구된 서버에는 변경사항을 일괄반영하여 데이터 일관성을 보존한다.
  - 이 방법을 "단서 후 임시 위탁(hinted handoff)" 기법이라고 한다고 함.
- 영구적 장애 처리
  - 반-엔트로피(anti-entropy) 프로토콜을 사용해 사본을 동기화한다
  - 사본간 일관성이 망가진 상태를 탐지하고 전송 데이터의 양을 줄이기 위해 머클트리(Merkle Tree)라는 자료구조 사용함
- 카산드라(Cassandra) 에서는, 데이터를 메모리 캐시에 적재해두다가 임계치에 도달하면, 디스크에 있는 "SSTable(Sorted-String Table)"에 기록한다고 한다
  - [SSTable 참고문서1](https://www.igvita.com/2012/02/06/sstable-and-log-structured-storage-leveldb/)
  - [SSTable 참고문서2](https://sjo200.tistory.com/56)
- 카산드라(Cassandra) 에서는, 메모리에 데이터가 없는 경우 디스크롤 조회하게 되는데 이 때 "블룸 필터(Bloom Filter)"가 사용된다고 한다.
  - [Bloom Filter 참고문서1](https://stackoverflow.com/questions/39327427/what-is-role-of-bloom-filter-in-cassandra)
  - [Bloom Filter 참고문서2](https://en.wikipedia.org/wiki/Bloom_filter)
