# Thread Pool 기반 API 서버로 확장한 Mini DBMS

## 1. 프로젝트 소개

이 프로젝트는 기존에 구현한 **Mini DBMS(SQL Processor + B+Tree 인덱스)** 를
**HTTP API 서버**로 확장한 결과물이다.

단순히 SQL 엔진만 구현한 것이 아니라,

- 외부 클라이언트가 HTTP로 질의할 수 있고
- 여러 요청을 Thread Pool로 병렬 처리할 수 있으며
- 동시성 제어를 통해 최소한의 정합성을 보장하고
- `/health` 엔드포인트로 서버 상태를 관찰할 수 있도록

구성한 것이 핵심이다.

즉, 이번 프로젝트의 핵심은 **"기존 DB 엔진을 재사용하면서, 멀티스레드 API 서버 환경으로 확장했다"** 는 점이다.

---

## 2. 핵심 목표

- 기존 SQL 처리기 재사용
- B+Tree 인덱스 재사용
- `POST /query` 기반 SQL 실행 API 제공
- `GET /health` 기반 서버 상태 확인 기능 제공
- Thread Pool + Bounded Queue 기반 병렬 요청 처리
- Lock Manager 기반 동시성 제어
- 부하 테스트를 통한 병목 분석

---

## 3. 핵심 기능

### 3-1. DB 기능
- `INSERT`
- `SELECT`
- `SELECT ... WHERE id = ?`
- 일반 `WHERE` 조건 조회
- B+Tree 기반 `id` 인덱스 탐색
- CSV 기반 저장소 연동

### 3-2. 서버 기능
- `POST /query` : SQL 실행
- `GET /health` : 서버 상태 조회
- JSON 요청 / JSON 응답
- Thread Pool 기반 요청 처리
- Bounded Queue 기반 overload 대응
- Busy 상태에서 빠른 거절(503)

### 3-3. 동시성 제어
- Job Queue는 `mutex + condition variable` 기반으로 동기화
- DB 실행 구간은 Lock Manager를 통해 보호
- `SELECT`는 기본적으로 read lock 경로를 타고,
  `INSERT` 등 변경 작업은 write lock 경로를 탐
- 단, 초기 로딩이 필요한 `SELECT`는 런타임 상태를 바꾸므로 write lock으로 승격될 수 있음

---

## 4. 전체 아키텍처
<img width="1672" height="941" alt="ChatGPT Image 2026년 4월 22일 오후 10_37_08" src="https://github.com/user-attachments/assets/8476e5c0-973d-42dd-accc-5a8103d0072a" />


---

## 5. 요청 처리 흐름

### 5-1. `/query` 처리 흐름

1. 클라이언트가 `POST /query` 요청을 보낸다.
2. 메인 스레드가 연결을 받아 job queue에 넣는다.
3. worker thread가 요청을 꺼내 HTTP 요청을 파싱한다.
4. JSON body에서 SQL 문자열을 추출한다.
5. Lock Manager를 통해 적절한 lock을 획득한다.
6. `DB engine facade`가 기존 SQL 실행 흐름에 진입한다.
7. 내부적으로 `tokenizer -> parser -> executor` 순서로 실행된다.
8. 실행 결과는 `DbResult`로 정리된다.
9. 결과를 JSON 응답으로 변환해 클라이언트에 반환한다.

### 5-2. `/health` 처리 흐름

`GET /health`는 DB 엔진을 실행하지 않고
현재 thread pool과 queue 상태를 JSON으로 내려준다.

예를 들어 다음과 같은 정보를 확인할 수 있다.

- `worker_count`
- `busy_workers`
- `idle_workers`
- `queue_length`
- `queue_capacity`
- `available_queue_slots`

이 엔드포인트는 부하 상황에서 **"지금 서버가 어디까지 찼는가?"** 를 확인하는 데 매우 유용하다.

---

## 6. 왜 이런 구조를 선택했는가

### 6-1. Thread Pool을 사용한 이유
요청마다 새 스레드를 만드는 방식은 생성/정리 비용이 크다.  
그래서 worker를 미리 만들어 두고 재사용하는 **Thread Pool** 구조를 채택했다.

### 6-2. Bounded Queue를 둔 이유
요청이 순간적으로 몰릴 때 바로 DB 쪽으로 밀어 넣는 대신,
중간에 **queue를 두어 완충 지점**으로 사용했다.

하지만 queue도 무한하지 않다.  
그래서 queue가 가득 차면 무한정 대기시키는 대신 **빠르게 실패 응답(503)** 을 줄 수 있게 했다.

### 6-3. Lock Manager를 둔 이유
기존 DB 엔진은 멀티스레드 환경을 전제로 만들어진 구조가 아니므로,
공유 상태를 보호하는 장치가 필요했다.

이를 위해 lock 정책을 별도 계층으로 분리했고,
현재는 read/write lock 기반으로 동시성 제어를 수행한다.

---

## 7. 프로젝트 구조

```text
week8_mini_dbms/
├── src/
│   ├── api/
│   │   ├── api_main.c
│   │   ├── api_server.c/h
│   │   ├── http_parser.c/h
│   │   ├── request_router.c/h
│   │   └── response_builder.c/h
│   ├── cli/
│   │   └── main.c
│   ├── common/
│   │   └── utils.c/h
│   ├── concurrency/
│   │   ├── job_queue.c/h
│   │   ├── lock_manager.c/h
│   │   └── thread_pool.c/h
│   └── db/
│       ├── benchmark.c/h
│       ├── bptree.c/h
│       ├── db_engine_facade.c/h
│       ├── executor.c/h
│       ├── executor_result.c/h
│       ├── index.c/h
│       ├── parser.c/h
│       ├── tokenizer.h
│       ├── table_runtime.h
│       └── ...
├── tests/
│   ├── api/
│   ├── concurrency/
│   ├── db/
│   └── integration/
├── docs/
│   ├── API_SERVER_DESIGN.md
│   ├── LOAD_TEST_COMPARISON.md
│   └── SPEC.md
├── scripts/
│   └── k6_queue_health_demo.js
└── Makefile
```

### 모듈별 역할

| 모듈 | 역할 |
|---|---|
| `api` | HTTP 요청 수신, 파싱, 라우팅, 응답 생성 |
| `concurrency` | thread pool, queue, lock 관리 |
| `db_engine_facade` | 서버 코드와 기존 DB 엔진 사이의 연결점 |
| `db` | tokenizer, parser, executor, table runtime, B+Tree |
| `tests` | 단위 테스트 + 통합 테스트 |
| `scripts` | k6 부하 테스트 스크립트 |

---

## 8. 빌드 및 실행 방법

### 8-1. 빌드

```bash
make
```

빌드 결과:

- `sql_processor`
- `api_server`

### 8-2. 테스트

```bash
make tests
```

### 8-3. API 서버 실행

```bash
./api_server
```

기본값:
- port: `8080`
- worker_count: `4`
- queue_capacity: `16`

직접 지정해서 실행할 수도 있다.

```bash
./api_server 8080 8 20
```

### 8-4. CLI 실행

```bash
./sql_processor
```

---

## 9. API 사용 예시

### 9-1. Health Check

```bash
curl http://127.0.0.1:8080/health
```

예시 응답:

```json
{
  "status": "ok",
  "worker_count": 4,
  "busy_workers": 0,
  "idle_workers": 4,
  "queue_length": 0,
  "queue_capacity": 16,
  "available_queue_slots": 16
}
```

### 9-2. INSERT 요청

```bash
curl -X POST http://127.0.0.1:8080/query \
  -H "Content-Type: application/json" \
  -d '{"sql":"INSERT INTO users (name, age) VALUES (\"kim\", 24);"}'
```

### 9-3. SELECT 요청

```bash
curl -X POST http://127.0.0.1:8080/query \
  -H "Content-Type: application/json" \
  -d '{"sql":"SELECT * FROM users WHERE id = 1;"}'
```

---

## 10. 성능 및 부하 테스트

이번 프로젝트에서는 단순히 "서버가 돌아간다" 수준에서 끝내지 않고,
실제로 부하를 걸었을 때 어떤 병목이 발생하는지를 확인했다.

부하 테스트는 `scripts/k6_queue_health_demo.js`를 사용했고,
`GET /health` fast-path가 적용된 상태에서
`POST /query`로 `INSERT` 부하를 가한 실험을 비교했다.

### 10-1. 비교 조건

- 실험 A: `worker_count = 4`, `queue_capacity = 16`
- 실험 B: `worker_count = 8`, `queue_capacity = 20`

### 10-2. 결과 요약

| 항목 | 4 workers / 16 queue | 8 workers / 20 queue |
|---|---:|---:|
| `query_ok` | 11,676 | 11,646 |
| `query_busy` | 6,046 | 6,076 |

### 10-3. 관찰 결과

겉보기에는 worker 수를 2배로 늘리고 queue도 키웠기 때문에
처리량이 꽤 증가할 것처럼 보인다.

하지만 실제 결과는 거의 차이가 없었다.

즉,

- `worker_count`를 늘려도 `query_ok`가 크게 증가하지 않았고
- `query_busy` 역시 거의 비슷하게 나타났다.

### 10-4. 왜 이런 결과가 나왔는가

핵심 원인은 `INSERT` 요청이 내부적으로 **write lock 경로**를 타기 때문이다.

즉, 앞단에서는 여러 worker가 병렬로 요청을 받더라도,
실제 DB 변경 구간에서는 write lock 때문에 **사실상 직렬화**된다.

이를 그림으로 표현하면 다음과 같다.

```mermaid
flowchart LR
    A["Client Requests"] --> B["API Server"]
    B --> C["Bounded Queue"]
    C --> D["Thread Pool"]
    D --> E["DB Engine"]
    E --> F["Write Lock"]
    F --> G["TableRuntime / CSV Storage"]
```

이 구조에서:

- `Thread Pool`과 `Queue`는 **요청을 받아두는 버퍼**
- 실제 병목은 뒤쪽의 **DB write lock 경로**
- 따라서 worker 수를 늘려도 write 구간이 그대로면 처리량 향상은 제한적

이라는 점을 확인할 수 있었다.

### 10-5. 이 실험의 의미

이 실험은 단순히
**"워커 수가 부족해서 느린가?"**
를 확인한 것이 아니라,

실제로는
**"DB write lock이 병목인가?"**
를 확인한 실험이었다.

즉, 이번 결과를 통해:

- 병목이 queue 크기 때문인지
- worker 개수 때문인지
- 아니면 DB 내부 직렬화 때문인지

를 구분해서 설명할 수 있게 되었다.

---

## 11. 테스트 및 검증

이 프로젝트는 기능 구현뿐 아니라 검증도 함께 수행했다.

- 단위 테스트
- API 관련 테스트
- 동시성 관련 테스트
- DB 관련 테스트
- integration 테스트
- 부하 테스트(k6)

즉, 단순히 "돌아간다"가 아니라,
**정상 동작 / 예외 상황 / 부하 상황** 까지 확인했다는 점이 중요하다.

---

## 12. 발표에서 강조할 포인트

발표에서는 아래 4가지를 핵심 메시지로 가져가면 좋다.

### 1) 기존 엔진을 재사용했다
처음부터 새로운 DB를 만든 것이 아니라,
기존 SQL 처리기와 B+Tree 인덱스를 재사용해서 API 서버로 확장했다.

### 2) 병렬 처리 구조를 도입했다
Thread Pool + Bounded Queue 구조로
멀티 요청을 처리할 수 있는 서버 구조를 만들었다.

### 3) 동시성 제어를 분리했다
DB lock을 별도 계층으로 분리해서
정합성을 지키면서도 이후 정책 변경이 가능하도록 설계했다.

### 4) 단순 구현이 아니라 병목까지 분석했다
부하 테스트를 통해
"왜 worker를 늘려도 성능이 크게 오르지 않는가?"를
write lock 병목 관점에서 설명할 수 있다.

---

## 13. 결론

이번 프로젝트는 단순한 SQL 처리기 구현을 넘어,

- 기존 DB 엔진 재사용
- HTTP API 서버 확장
- Thread Pool 기반 병렬 처리
- Lock 기반 동시성 제어
- 부하 테스트 기반 병목 분석

까지 연결한 프로젝트다.

즉, **"자료구조/DB 구현" + "서버 시스템 설계" + "성능 분석"** 을 한 번에 경험한 과제라고 정리할 수 있다.
