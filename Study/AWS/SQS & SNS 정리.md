# SQS & SNS 정리

## 1. SQS (Simple Queue Service)

AWS에서 제공하는 큐 서비스. **하나의 메시지를 한 번(가능한 한 정확히 한 번)만 처리**하는 것을 목표로 하는 **Pull(polling) 기반** 메시징 서비스.

### 1-1. 동작 방식

- Producer가 보낸 메시지를 최대 **14일간 임시 저장**하고, Consumer가 요청할 때 들어온 순서대로 전달
- Consumer가 메시지를 처리 완료(`DeleteMessage`)하면 큐에서 삭제
- 오류 등으로 처리에 실패하면 메시지는 삭제되지 않고 다시 큐로 반환되어 재수신 가능 상태가 됨
- 여러 Consumer가 동시에 요청하더라도, 하나의 메시지는 한 번에 한 Consumer에게만 전달됨

> SQS는 **push 기능이 없다.** 서버가 능동적으로 메시지를 밀어주는 게 아니라, Consumer가 `ReceiveMessage` API를 반복 호출(polling)해서 가져가는 구조다. Lambda + SQS 트리거처럼 "이벤트가 오면 실행되는" 것처럼 보이는 경우도, 내부적으로는 Lambda의 폴러(poller)가 백그라운드에서 계속 polling하고 있는 것뿐이다.

### 1-2. 구성 요소

|요소|설명|
|---|---|
|메시지|SQS에서 전달하는 데이터의 단위|
|Producer|메시지를 생산하고 SQS에 전달하는 주체|
|Queue|메시지를 저장하고 Consumer에게 전달하는 기능을 담당|
|Consumer|메시지를 받아 처리하고 소비하는 주체|
|Dead Letter Queue(DLQ)|처리에 반복 실패한 메시지를 모아두는 2차 큐|

### 1-3. 큐 타입

|구분|Standard|FIFO|
|---|---|---|
|순서 보장|X|O (단, **MessageGroupId 단위**)|
|중복 전달 가능성|있음 (최소 1회 이상 전달)|없음 (정확히 1회 전달 보장)|
|성능|높음|제약 있음|
|멱등성 확보|필요 (재전달 대비)|상대적으로 덜 필요|

**FIFO 큐의 Message Group**

- FIFO 큐 내부는 다시 `MessageGroupId` 단위로 나뉘며, **같은 그룹의 메시지는 순서대로 직렬 처리**되고, **다른 그룹끼리는 병렬 처리** 가능
- `MessageGroupId`는 큐 URL(큐 이름/계정ID/리전 정보만 포함)에 있는 게 아니라, 메시지 발행 시 요청 파라미터로 개별 메시지에 실어 보낸다
- 모든 메시지를 하나의 그룹으로 묶으면 순서는 완벽히 보장되지만 사실상 직렬 처리라 처리량이 크게 떨어짐 → **"순서가 실제로 의미 있는 최소 단위"**(예: `userId`, `orderId` 등)로 그룹을 나누는 것이 일반적

**큐를 여러 개로 나누는 기준**

- 도메인/책임 단위 (컨슈머·처리 로직이 다르면 분리)
- 순서 보장 필요 여부 (Standard vs FIFO)
- 처리량/우선순위 차이 (느린 메시지가 빠른 메시지를 막지 않도록)
- 재시도/DLQ 정책 차이
- 서비스 경계 (발행자-구독자 관계당 큐 하나가 자연스러움)

### 1-4. SQS 메시지

- 최대 사이즈 **1MiB**, 최대 **14일** 저장 가능
- 메시지 상태
    - **Stored**: Producer가 전달을 완료하여 대기 중
    - **In Flight**: Consumer가 가져가서 처리 중인 상태 (Visibility Timeout 동안 다른 Consumer에게 숨겨짐)
    - **Deleted**: Consumer가 처리 후 삭제한 상태

**Visibility Timeout 참고**: FIFO 큐에서는 정확히는 큐 전체가 아니라 **같은 MessageGroupId 그룹 단위**로 잠긴다. 한 그룹의 메시지가 처리 중(Visibility Timeout이 끝나지 않음)이면 같은 그룹의 다음 메시지는 다른 Consumer가 가져갈 수 없지만, 다른 그룹의 메시지는 동시에 처리 가능하다.

### 1-5. Consumer & Polling

기본 workflow: **Poll → 처리 → 삭제**

Consumer는 API로 큐에 주기적으로 메시지를 요청(Poll)해서 가져간다. HTTP 요청 하나를 서버가 오래 붙잡아두는 방식이며(HTTP Long Polling 기법), 웹소켓처럼 별도의 양방향 지속 연결이 열리는 것은 아니다.

**Short Polling (기본값, `WaitTimeSeconds=0`)**

- 요청 시 큐의 일부만 검색해서 가능한 메시지를 빠르게 찾아 전달
- 메시지를 못 찾아도 응답을 즉시 전달
- 딜레이 없이 루프를 돌리면 메시지가 없을 때도 초당 수십~수백 번씩 빈 응답을 받는 요청을 계속 날리게 되고, 이건 전부 과금 대상이 됨 (요청 횟수 기준 과금이라 비용 폭탄의 흔한 원인)
- 참고로 이렇게 계속 요청을 보낸다고 해서 SQS가 이를 공격으로 판단해 차단하지는 않는다. 계정/큐 단위 요청 할당량(quota)을 넘으면 `ThrottlingException` 같은 에러 응답만 돌아올 뿐, 별도의 보안 위협 탐지가 작동하는 건 아니다. 실무에서 문제가 되는 건 차단이 아니라 **불필요한 비용과 리소스 낭비**다.
- 이 문제 때문에 AWS는 Long Polling을 강력히 권장하고, 대부분의 SDK(Spring Cloud AWS 포함)도 기본값을 Long Polling에 가깝게 세팅하거나 그렇게 쓰도록 강조한다. 실무에서 Short Polling을 쓰는 경우는 거의 없다 (아주 가끔 큐 상태만 체크하는 배치성 용도가 아니라면).

**Long Polling (`WaitTimeSeconds: 1~20초`)**

- 요청 시 큐 전체를 검색하여 적어도 하나의 메시지를 찾아 전달
- 메시지가 없으면 찾을 때까지 대기하거나, Timeout을 넘기면 그때 빈 응답 전달
- 메시지가 도착하면 20초를 다 기다리지 않고 즉시 응답
- 응답을 받으면(메시지가 있든 없든) 컨슈머는 곧바로 다음 `ReceiveMessage`를 재호출 — 요청 사이에 별도의 sleep이 끼는 게 아니라, "대기(최대 20초) → 응답 → 즉시 재요청"이 반복되는 구조
- **동작 원리**: SQS는 도착한 `ReceiveMessage` 요청을 즉시 처리하지 않고 "대기 중인 요청" 목록에 걸어둔다. 이후 메시지가 새로 들어오거나 처리 가능한 상태가 되면, 대기 중이던 요청 중 하나(또는 `MaxNumberOfMessages` 설정에 따라 여러 개)를 골라 그 요청의 응답으로 실어 보낸다. 여러 Consumer가 동시에 대기 중이면 자연스럽게 로드밸런싱처럼 분배됨
- Batch 처리 가능 (한 번에 최대 10개)

---

## 2. SNS (Simple Notification Service)

**Pub/Sub 기반**의 메시징 서비스. 하나의 메시지를 **여러 서비스가 동시에** 처리하도록 하는 것이 목적이며, 전달 방식은 **PUSH**다.

### 2-1. 주요 사용 사례

- 하나의 메시지를 다수의 주체가 각자 처리하고 싶을 때 (Fan-out)
- 서비스에서 서비스로 메시지를 전달하고 싶을 때
- 메시지를 임시로 저장하거나 재생(replay)하여 동일한 로직을 다시 처리하고 싶을 때

### 2-2. 사용 방식

- 하나의 토픽을 여러 주체가 구독
- 토픽에 발행된 메시지는 구독한 모든 주체에게 전달 → 하나의 메시지를 여러 주체가 각각 처리

### 2-3. 구성 요소

|요소|설명|
|---|---|
|Topic|SNS의 커뮤니케이션 채널|
|Subscription|토픽으로 들어온 메시지를 받아볼 수 있는 리소스|
|Publisher|메시지를 생산하고 SNS에 전달하는 주체|
|Subscriber|실제 구독을 통해 메시지를 받아 처리하는 주체 (SQS, Lambda, HTTP(S), Email, SMS 등)|

### 2-4. SNS는 내부에 "큐"가 없다

SNS는 메시지를 큐에 쌓아뒀다가 Consumer가 꺼내가는 구조가 아니라, 발행 즉시 구독자에게 **밀어넣기(push)를 시도**하는 구조다. HTTP(S) 엔드포인트처럼 즉시 응답을 못 받는 구독자의 경우, SNS는 정해진 **재시도 정책(exponential backoff)**에 따라 몇 차례 재시도하는데, 이건 영구 저장소가 아니라 일정 기간의 재시도용 임시 버퍼에 가깝다. 재시도 기간이 끝나도록 실패하면 별도 DLQ가 없는 한 메시지는 유실된다.

**DLQ는 있는가?** → 구독(subscription) 단위로 DLQ를 지정할 수 있지만, 이 DLQ도 결국 **SQS 큐**를 만들어서 붙이는 것이다. 즉 SNS는 "실패 감지 + 재시도" 로직만 담당하고, 메시지를 안전하게 오래 보관하는 durability(내구성)는 여전히 SQS의 몫이다.

|기능|SQS|SNS|
|---|---|---|
|Visibility Timeout|O|개념 자체 없음 (push라 "처리 중" 상태가 없음)|
|메시지 장기 보관(최대 14일)|O|X (짧은 재시도 기간 내에서만)|
|Pull 방식|O|X (Push만)|
|Delay Queue|O (최대 15분)|X|
|FIFO + MessageGroupId|O|SNS FIFO 토픽에도 존재|
|재처리(재수신)|O|X (DLQ로 간 것 제외)|

이런 이유로 Fan-out 아키텍처에서는 **SNS 뒤에 항상 SQS를 붙이는 것이 표준 패턴**이다. SNS는 빠른 배달부, SQS는 안전한 창고 역할을 나눠 맡는 셈이다.

### 2-5. Fan-out 패턴 (SNS → 다수의 SQS)

```
Producer → SNS Topic (order-events)
              ├─→ SQS Queue (store-service)    → 매장 컨슈머
              ├─→ SQS Queue (delivery-service) → 배달 컨슈머
              ├─→ SQS Queue (coupon-service)   → 쿠폰 컨슈머
              └─→ SQS Queue (point-service)    → 포인트 컨슈머
```

- SQS의 "하나의 메시지는 하나의 Consumer가 처리" 원칙은 그대로 유지하면서, 서비스별로 각자 큐를 두어 사실상 여러 서비스가 같은 이벤트를 동시에 소비하게 만드는 구조
- **장점**: 느슨한 결합(새 구독자 추가 시 Producer 코드 변경 불필요), 장애 격리(한 서비스 장애가 다른 서비스로 전파되지 않음), 서비스별로 독립적인 Visibility Timeout/재시도/DLQ 정책 설정 가능
- **주의**: FIFO로 순서를 지키려면 SNS FIFO 토픽 + SQS FIFO 큐 조합 필요. 모든 구독자가 전체 메시지를 받을 필요 없다면 SNS의 메시지 필터링 정책으로 구독 단위 필터 적용 가능

### 2-6. 실패 처리 (Fan-out 구조에서)

|실패 지점|영향 범위|대응|
|---|---|---|
|SNS → 특정 구독자 전달 실패|해당 구독자만, 다른 구독자는 무관|SNS 재시도 → 실패 시 SNS DLQ|
|SQS 도착 후 컨슈머 처리 실패|해당 큐/서비스만, 다른 서비스는 무관|Visibility Timeout 재시도 → `maxReceiveCount` 초과 시 SQS DLQ|

각 구독은 독립적인 재시도/DLQ 파이프라인을 가지므로, 하나의 실패가 다른 구독자로 전파(cascade)되지 않는다는 것이 Fan-out 구조의 핵심 장점이다.

---

## 3. Kafka와의 비교 (참고)

Kafka는 **컨슈머 그룹(Consumer Group)** 개념을 기본 내장하고 있어, SNS+SQS의 Fan-out 역할 상당 부분을 Kafka 하나로 대체할 수 있다.

- 토픽 하나에 여러 컨슈머 그룹이 각자 독립 구독 → SNS가 여러 SQS 큐로 뿌려주는 것과 유사한 효과
- 같은 그룹 안에 여러 인스턴스(파티션 수만큼) → SQS처럼 "하나의 메시지는 그룹 내 하나만 처리"하는 병렬 워커 풀 효과

다만 완전히 동일하지는 않다.

|항목|SQS|Kafka|
|---|---|---|
|삭제/커밋 방식|명시적 삭제(ack), 처리 완료 보장이 명확|오프셋 커밋, 상대적으로 관리 복잡|
|지연 발행|Delay Queue 내장(최대 15분)|기본 기능 없음|
|운영 복잡도|완전 관리형|브로커/파티션/리밸런싱 등 운영 부담|
|보존/재처리(replay)|최대 14일, 재처리 제한적|장기 보존 가능, replay에 강함|
|처리량|상대적으로 낮음|초고처리량|

→ 실무에서는 **재처리(replay)가 필요한 핵심 도메인 이벤트는 Kafka**, **단순 1:1 비동기 작업 큐(알림 발송, 후처리 등)는 SQS**로 나눠 쓰는 경우가 많다.

---

## 4. Java/Spring 예시 코드 (Spring Cloud AWS 3.x 기준)

### 의존성

```xml
<dependency>
    <groupId>io.awspring.cloud</groupId>
    <artifactId>spring-cloud-aws-starter-sqs</artifactId>
    <version>3.1.1</version>
</dependency>
<dependency>
    <groupId>io.awspring.cloud</groupId>
    <artifactId>spring-cloud-aws-starter-sns</artifactId>
    <version>3.1.1</version>
</dependency>
```

### SQS Publisher (Producer)

```java
@Service
@RequiredArgsConstructor
public class OrderQueueProducer {

    private final SqsTemplate sqsTemplate;

    public void send(String userId, OrderEvent event) {
        sqsTemplate.send(to -> to
            .queueName("order-events.fifo")
            .payload(event)
            .messageGroupId("user-" + userId)                 // FIFO 큐 순서 보장 단위
            .messageDeduplicationId(UUID.randomUUID().toString())
        );
    }
}
```

### SQS Consumer

```java
@Component
public class OrderQueueConsumer {

    @SqsListener(value = "order-events.fifo")
    public void listen(OrderEvent event) {
        // 처리 로직
        // 예외 발생 시 메시지가 삭제되지 않고 Visibility Timeout 이후 재수신됨
    }
}
```

### SNS Publisher (Producer)

```java
@Service
@RequiredArgsConstructor
public class OrderTopicPublisher {

    private final SnsTemplate snsTemplate;

    public void publish(OrderEvent event) {
        snsTemplate.send("order-events-topic", MessageBuilder.withPayload(event).build());
    }
}
```

### SNS Subscriber — SQS를 구독자로 붙이는 경우 (Fan-out 표준 패턴)

SNS 토픽을 구독하는 SQS 큐는 결국 **SQS Consumer 코드와 동일**하다. 실제 "구독"은 인프라(콘솔/CDK/Terraform 등)에서 SNS Topic ↔ SQS Queue를 연결하는 설정이며, 애플리케이션 코드에서는 그 SQS 큐를 그냥 리스닝하면 된다.

```java
@Component
public class CouponServiceConsumer {

    @SqsListener(value = "coupon-service-queue") // 이 큐는 order-events-topic을 구독 중
    public void listen(SnsNotification<OrderEvent> notification) {
        OrderEvent event = notification.getMessage();
        // 쿠폰 발급 처리 로직
    }
}
```

> 매장/배달/포인트 서비스도 각자의 SQS 큐(`store-service-queue`, `delivery-service-queue`, `point-service-queue`)를 같은 방식으로 리스닝하면 하나의 SNS 발행 메시지를 각자 독립적으로 소비하는 Fan-out 구조가 완성된다.