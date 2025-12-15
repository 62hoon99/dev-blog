# Coordinator 기반 분산 Key-Value 저장소 실습

## 실습 목적

- Redis 서버 여러 대(예: 3대)를 하나의 “분산 저장소”처럼 보이게 만든다.
- Coordinator가 데이터 처리의 중심이 되어 **일관성 있게 여러 노드에 데이터 읽기/쓰기를 관리**한다.
- 복잡한 Paxos/Raft 없이, **단순 Majority-Writes, First-Response-Read** 방식으로 구현한다.
- 분산 저장소에서 _Coordinator가 무엇을 하는지 이해한다.

## 설계 이유

### 1.  Coordinator 역할을 분명하게 체감할 수 있음

Redis 여러 개를 직접 운영하면 단순한 Key-Value 저장소가 바로 분산 시스템이 되지 않는다. 각 Redis는 다른 Redis가 뭘 하는지 모른다. 그래서 “중앙에서 조율하는 존재”가 필요하다.
이 조율을 담당하는 것이 **Coordinator** 이다.

#### Coordinator는 다음을 담당:
**Write 요청 시**
- 클라이언트로부터 key, value를 받음
- 모든 Redis 노드에 set 요청
- 최소 2/3 이상의 Redis가 성공하면 commit 성공
- 실패하면 “실패”로 처리하며 rollback(optional)

**Read 요청 시**
- 모든 Redis 노드에 get 요청
- 가장 먼저 응답 온 값을 반환
- 서버 간 값 불일치 감지 가능 → 이를 바탕으로 self-healing 가능

###  3. 현실 세계의 분산 Key-Value 시스템과 유사

Cassandra, DynamoDB, Riak, MongoDB 등의 동작 일부를 축약해 체험할 수 있음.

## 실습
``` Python
from flask import Flask, request, jsonify
import redis
import threading
import time

app = Flask(__name__)

# Redis 노드 목록
REDIS_NODES = [
    redis.Redis(host='127.0.0.1', port=6379),
    redis.Redis(host='127.0.0.1', port=6380),
    redis.Redis(host='127.0.0.1', port=6381),
]

MAJORITY = 2  # 3대 중 2대 성공하면 commit 성공


# -------------------------------------------------------------------
# Write: 모든 노드에 SET 요청 → Majority 성공 시 OK
# -------------------------------------------------------------------
@app.route("/set", methods=["POST"])
def set_value():
    data = request.json
    key = data["key"]
    value = data["value"]

    success_count = 0

    for node in REDIS_NODES:
        try:
            node.set(key, value)
            success_count += 1
        except Exception as e:
            print(f"[WARN] SET 실패: {e}")

    if success_count >= MAJORITY:
        return jsonify({"status": "OK", "success_nodes": success_count})
    else:
        return jsonify({"status": "FAIL", "success_nodes": success_count}), 500


# -------------------------------------------------------------------
# Read: 모든 노드에 GET 요청 후 가장 빠르게 응답한 값 반환
# -------------------------------------------------------------------
@app.route("/get", methods=["GET"])
def get_value():
    key = request.args.get("key")

    responses = []
    threads = []

    def fetch(node):
        try:
            val = node.get(key)
            if val is not None:
                responses.append(val.decode())
        except:
            pass

    # 병렬로 GET 요청
    for node in REDIS_NODES:
        t = threading.Thread(target=fetch, args=(node,))
        t.start()
        threads.append(t)

    # 응답을 300ms만 기다림 → 가장 빠른 응답만 사용
    start = time.time()
    while time.time() - start < 0.3:
        if responses:
            break
        time.sleep(0.01)

    # fallback: 모든 스레드 기다리기
    for t in threads:
        t.join(timeout=0.1)

    if responses:
        return jsonify({"status": "OK", "value": responses[0]})
    else:
        return jsonify({"status": "NOT_FOUND"}), 404


if __name__ == "__main__":
    app.run(port=5000)
```

