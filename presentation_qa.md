# Mini DBMS API Server 발표 예상 질문 & 답변

이 문서는 목요일 발표 QnA 대비용입니다.
답변은 실제 현재 프로젝트 구현 기준으로 작성했습니다.

## 0. 10초 요약 답변

**Q. 이번 프로젝트 한 문장으로 설명하면?**

A. C로 만든 미니 SQL 처리기에 HTTP API 서버, thread pool, 동시성 lock, B+Tree 기반 `id` 인덱스를 붙여서 외부 클라이언트가 SQL을 API로 실행할 수 있게 만든 프로젝트입니다.

**꼬리질문: 핵심 구현은 어디인가요?**

A. 핵심은 세 부분입니다.

- API 서버: socket accept, HTTP request 읽기, route 처리, JSON 응답 생성
- 병렬 처리: thread pool과 bounded job queue
- DB 엔진 연결: `db_engine_facade`가 tokenizer, parser, executor, table runtime, B+Tree를 하나로 묶음

## 1. 과제 요구사항 매핑

**Q. 과제 요구사항 중 무엇을 구현했나요?**

A. 요구사항 기준으로 보면 다음을 구현했습니다.

- 외부 클라이언트용 API 서버: `GET /health`, `POST /query`
- SQL 요청 병렬 처리: thread pool + job queue
- 기존 SQL 처리기 활용: tokenizer, parser, executor 재사용
- B+Tree 인덱스 활용: `WHERE id = ?` 조회에서 B+Tree 사용
- C 언어 구현: 전체 서버/DB 엔진 C99
- 테스트: DB 단위 테스트, 동시성 테스트, API smoke/concurrency 테스트

**꼬리질문: 발표에서 가장 강조할 차별점은?**

A. 단순히 API를 붙인 것이 아니라, 기존 stdout 중심 executor를 `DbResult`라는 구조화된 결과로 바꾸어 CLI와 API가 같은 실행 결과를 공유하도록 만든 점입니다. 또 API 응답에 `"used_id_index": true/false`를 넣어서 실제 B+Tree 인덱스 사용 여부를 검증 가능하게 했습니다.

## 2. 전체 아키텍처

**Q. 전체 흐름을 설명해 주세요.**

A. 외부 요청 기준 흐름은 다음과 같습니다.

```text
client
-> api_server accept
-> thread_pool_submit
-> worker thread
-> HTTP parser
-> request router
-> db_engine_facade
-> tokenizer
-> parser
-> executor
-> table_runtime / B+Tree / storage
-> DbResult
-> JSON response
-> client
```

**꼬리질문: 왜 `db_engine_facade`가 필요한가요?**

A. API 서버가 tokenizer, parser, executor 내부 구조를 직접 알면 결합도가 커집니다. `db_engine_facade`는 SQL 문자열 하나를 받아 `DbResult`를 반환하는 단일 진입점 역할을 하므로, API 서버는 DB 내부 구현을 몰라도 SQL 실행 결과를 받을 수 있습니다.

**꼬리질문: CLI와 API는 같은 DB 엔진을 쓰나요?**

A. 네. CLI는 `db_execute_sql`을 호출하고 결과를 `executor_render_result_for_cli`로 출력합니다. API는 `execute_query_with_lock`을 호출하고 결과를 `build_query_json_response`로 JSON 변환합니다. 실행 엔진은 공유하고 표현 방식만 다릅니다.

## 3. API 서버

**Q. API 엔드포인트는 무엇인가요?**

A. 두 개입니다.

- `GET /health`: 서버 상태 확인, `{"status":"ok"}` 반환
- `POST /query`: JSON body의 `sql` 문자열을 실행

예시:

```bash
curl -i http://127.0.0.1:8080/health
```

```bash
curl -i -X POST http://127.0.0.1:8080/query \
  -H "Content-Type: application/json" \
  --data '{"sql":"SELECT id, name FROM users WHERE id = 1;"}'
```

**꼬리질문: 왜 JSON body를 `{"sql":"..."}`로 단순화했나요?**

A. 과제 핵심은 DBMS 기능을 API로 노출하고 병렬 처리하는 것입니다. 그래서 MVP에서는 request schema를 단순하게 고정했습니다. 확장한다면 query id, timeout, transaction option 등을 body에 추가할 수 있습니다.

**Q. HTTP 처리는 어떤 수준까지 구현했나요?**

A. 완전한 범용 HTTP 서버가 아니라 과제용 MVP 서버입니다. request line, path, method, `Content-Length`, body를 읽고, 응답은 `HTTP/1.1`, `Content-Type: application/json`, `Content-Length`, `Connection: close`를 붙여 보냅니다.

**꼬리질문: keep-alive는 지원하나요?**

A. 지원하지 않습니다. 응답에 `Connection: close`를 넣고 요청 하나 처리 후 socket을 닫습니다. 구현 범위를 줄이고 동시성 검증을 명확히 하기 위한 선택입니다.

**Q. 잘못된 요청은 어떻게 처리하나요?**

A. 잘못된 HTTP request는 400, 잘못된 method는 405, 없는 route는 404, queue가 가득 찬 경우는 503, 내부 응답 생성 실패는 500으로 JSON 에러를 반환합니다.

## 4. Thread Pool

**Q. Thread pool이 뭔가요?**

A. 요청이 들어올 때마다 thread를 새로 만들지 않고, 미리 worker thread들을 만들어 둔 뒤 요청을 queue에 넣고 worker가 하나씩 가져가 처리하는 구조입니다.

**꼬리질문: 왜 요청마다 thread를 새로 만들지 않았나요?**

A. 매 요청마다 thread를 생성하면 생성/소멸 비용이 크고, 요청이 몰릴 때 thread 수가 무한히 늘어날 수 있습니다. thread pool은 worker 수와 queue 용량을 제한해서 서버 자원 사용량을 예측 가능하게 만듭니다.

**Q. 이 프로젝트의 thread pool 흐름은?**

A.

```text
api_server_run
-> accept client_fd
-> thread_pool_submit
-> queue_push
-> worker thread가 queue_pop
-> api_server_worker_handle_client
-> 요청 처리 후 close(client_fd)
```

**꼬리질문: queue가 꽉 차면 어떻게 하나요?**

A. blocking하지 않고 즉시 실패합니다. API 서버는 해당 client에게 `503 Server is busy.` JSON 응답을 보내고 socket을 닫습니다. 이것은 서버가 과부하일 때 무한 대기하지 않도록 하는 backpressure입니다.

**Q. worker thread 수는 어떻게 설정하나요?**

A. 기본은 `4`개이고 실행 인자로 바꿀 수 있습니다.

```bash
./api_server 8080 4 16
```

여기서 `8080`은 port, `4`는 worker 수, `16`은 queue capacity입니다.

## 5. 동시성 이슈

**Q. 이 프로젝트에서 동시성 이슈가 생길 수 있는 지점은?**

A. 대표적으로 네 곳입니다.

- `TableRuntime`: 전역 활성 테이블, rows, `next_id`, B+Tree root를 공유
- `B+Tree`: INSERT 시 구조가 바뀔 수 있음
- `tokenizer cache`: 전역 linked list cache
- CSV storage: 여러 요청이 같은 파일을 읽고 쓸 수 있음

**Q. 동시성 문제를 어떻게 막았나요?**

A. 두 레벨로 막았습니다.

- DB 엔진 내부 상태는 `lock_manager`의 DB lock으로 보호합니다.
- tokenizer cache는 별도 mutex로 보호합니다.
- storage 파일 접근은 `flock`으로 파일 lock을 겁니다.

**꼬리질문: DB lock은 mutex인가요 rwlock인가요?**

A. 현재 `db_engine_init`에서 `LOCK_POLICY_SPLIT_RWLOCK`으로 초기화합니다. 그래서 SELECT는 read lock, INSERT/DELETE는 write lock을 사용합니다. 다만 SELECT라도 아직 table이 memory에 load되지 않은 경우에는 상태를 바꾸므로 write lock으로 승격합니다.

**꼬리질문: read lock에서 write lock으로 바로 upgrade하면 deadlock 위험이 있지 않나요?**

A. 직접 upgrade하지 않고 read lock을 먼저 풀고 write lock을 다시 잡습니다. 그래서 upgrade deadlock을 피합니다. 그 사이 다른 thread가 먼저 table을 load할 수 있지만, 이후 실행 단계에서 다시 상태를 확인하므로 안전합니다.

**Q. SELECT끼리는 동시에 실행되나요?**

A. table이 이미 memory에 로드되어 있으면 SELECT는 read lock이므로 동시에 실행될 수 있습니다. INSERT는 write lock이라 SELECT와 동시에 실행되지 않습니다.

**Q. INSERT끼리는 동시에 실행되나요?**

A. INSERT는 write lock이므로 한 번에 하나만 실행됩니다. 그래야 `next_id`, rows 배열, B+Tree 구조가 깨지지 않습니다.

## 6. DB 엔진 연결

**Q. API 서버와 DB 엔진은 어떻게 연결되어 있나요?**

A. API 서버는 SQL 문자열을 추출한 뒤 `execute_query_with_lock(engine, sql, &result)`를 호출합니다. 이 함수가 lock을 잡고 `db_execute_sql`을 실행합니다. `db_execute_sql`은 `tokenizer -> parser -> executor` 순서로 처리하고 `DbResult`를 채웁니다.

**꼬리질문: 기존 SQL 처리기를 어떻게 API 친화적으로 바꿨나요?**

A. 기존처럼 executor가 바로 `printf`로 출력하면 API 응답으로 쓰기 어렵습니다. 그래서 실행 결과를 `DbResult`에 담도록 `executor_execute_into_result`를 만들고, CLI는 그 결과를 표로 출력하고 API는 JSON으로 변환하게 분리했습니다.

**Q. `DbResult`에는 무엇이 들어가나요?**

A. 성공 여부, 결과 타입, 메시지, SELECT 컬럼/row, row count, affected rows, B+Tree 인덱스 사용 여부가 들어갑니다.

## 7. SQL 처리 흐름

**Q. SQL 한 문장은 내부에서 어떻게 처리되나요?**

A.

```text
SQL 문자열
-> tokenizer_tokenize
-> parser_parse
-> SqlStatement
-> executor_execute_into_result
-> DbResult
```

**Q. tokenizer는 무슨 역할인가요?**

A. SQL 문자열을 keyword, identifier, literal, operator, 괄호, comma, semicolon 같은 token 배열로 나눕니다.

**꼬리질문: tokenizer cache는 왜 있나요?**

A. 같은 SQL 문자열이 반복되면 tokenizing 비용을 줄일 수 있습니다. cache hit 시 token 배열을 새로 분석하지 않고 복사본을 반환합니다.

**Q. parser는 무슨 역할인가요?**

A. token 배열을 `SqlStatement` 구조체로 변환합니다. 예를 들어 INSERT는 table name, column list, value list로 나누고 SELECT는 projection, table, where 조건으로 나눕니다.

**Q. executor는 무슨 역할인가요?**

A. `SqlStatement`를 실제 데이터 처리로 연결합니다. INSERT면 runtime table에 row를 넣고 storage에 저장합니다. SELECT면 projection을 준비하고 B+Tree 또는 linear scan으로 row를 찾아 `DbResult`를 만듭니다.

## 8. B+Tree 인덱스

**Q. B+Tree를 왜 사용했나요?**

A. DB 인덱스에서 자주 쓰이는 균형 트리 구조이고, 정렬된 key를 유지하면서 검색 비용을 줄일 수 있습니다. 현재 프로젝트에서는 `id` 값을 key로 해서 row 위치를 빠르게 찾습니다.

**꼬리질문: 이 프로젝트에서 B+Tree는 정확히 무엇을 저장하나요?**

A. `id -> row_index`를 저장합니다. `id`는 자동 증가 값이고, `row_index`는 `TableRuntime.rows` 배열에서 해당 row의 위치입니다.

**Q. 언제 B+Tree를 사용하나요?**

A. SELECT에 WHERE가 있고 조건이 정확히 `id = 정수` 형태일 때만 사용합니다.

```sql
SELECT name FROM users WHERE id = 1;
```

이 경우 `executor_can_use_id_index`가 true를 반환하고 `bptree_search`를 호출합니다.

**꼬리질문: 다른 컬럼 WHERE는 왜 B+Tree를 안 쓰나요?**

A. 현재 B+Tree는 id 전용 인덱스로 설계되어 있습니다. 다른 컬럼은 schema마다 다르고 중복 값도 많을 수 있어서 column별 index registry가 필요합니다. 이번 과제 범위에서는 기존 B+Tree 인덱스를 활용하는 것을 우선해 id exact lookup에 집중했습니다.

**Q. B+Tree가 실제로 사용됐는지 어떻게 확인하나요?**

A. API SELECT 응답에 `"used_id_index": true`가 들어갑니다. API smoke test에서도 이 값을 검증합니다.

**Q. B+Tree 삽입 중 leaf가 꽉 차면 어떻게 하나요?**

A. `bptree_split_leaf`가 새 leaf를 만들고 key를 절반으로 나눈 뒤, 새 leaf의 첫 key를 parent로 올립니다. parent도 꽉 차면 `bptree_split_internal`로 internal node를 분할합니다.

## 9. TableRuntime과 Storage

**Q. 데이터는 어디에 저장되나요?**

A. 영구 저장은 `data/<table>.csv` 파일에 합니다. 실행 중에는 현재 활성 table 하나를 `TableRuntime`에 메모리로 올려 사용합니다.

**꼬리질문: 서버를 껐다 켜면 데이터가 유지되나요?**

A. 네. CSV 파일에 저장되므로 유지됩니다. 다음에 같은 table을 조회하거나 INSERT하면 `table_load_from_storage_if_needed`가 CSV를 읽어 runtime table과 B+Tree를 재구성합니다.

**Q. 왜 메모리에도 올리고 CSV에도 저장하나요?**

A. CSV는 persistence를 위한 저장소이고, 메모리는 빠른 SELECT와 B+Tree 인덱스 사용을 위한 runtime 상태입니다.

**Q. 현재 table runtime의 한계는?**

A. 전역 활성 table 하나만 유지합니다. 여러 table을 번갈아 요청하면 기존 table runtime을 비우고 새 table을 로드합니다. 포트폴리오용 확장으로는 table registry를 만들어 여러 table runtime을 동시에 유지할 수 있습니다.

**Q. `id`는 어떻게 생성하나요?**

A. runtime table의 `next_id`가 자동 증가합니다. 첫 INSERT에서 schema 앞에 `id` 컬럼을 붙이고, row를 넣을 때마다 `next_id`를 문자열로 변환해 첫 컬럼에 저장합니다.

**꼬리질문: 명시적으로 id를 넣으면?**

A. 현재 runtime executor 경로에서는 명시적 `id` 삽입을 허용하지 않습니다. `table_initialize_schema`에서 INSERT 컬럼 중 `id`가 있으면 실패 처리합니다.

## 10. DELETE 관련 질문

**Q. DELETE는 지원하나요?**

A. parser와 storage 계층에는 DELETE 관련 구현이 일부 있지만, 현재 메모리 runtime executor에서는 DELETE를 지원하지 않습니다. 실행하면 `DELETE is not supported in memory runtime mode.` 에러를 반환합니다.

**꼬리질문: 왜 막았나요?**

A. DELETE를 완전히 지원하려면 CSV 삭제뿐 아니라 runtime rows 배열, row_index, B+Tree 인덱스까지 일관되게 갱신해야 합니다. 현재 과제 핵심은 API 서버, thread pool, 기존 SQL/B+Tree 연결이므로 DELETE는 명확히 미지원으로 처리했습니다.

**꼬리질문: 나중에 구현한다면 어떻게 하나요?**

A. 두 가지 방식이 있습니다.

- physical delete: rows 배열에서 제거하고 모든 row_index와 B+Tree를 재구성
- logical delete: deleted flag를 두고 SELECT에서 제외

현재 구조에서는 단순하고 안전한 방법은 DELETE 후 table을 storage에서 다시 load하면서 B+Tree를 재구성하는 방식입니다.

## 11. 테스트와 검증

**Q. 어떤 테스트를 작성했나요?**

A. 크게 네 종류입니다.

- DB 단위 테스트: tokenizer, parser, storage, executor, B+Tree, facade
- 동시성 테스트: thread pool, tokenizer cache thread safety
- 통합 테스트: SQL 파일을 `sql_processor`로 실행
- API 테스트: health, query, 동시 INSERT, 병렬 SELECT

**Q. 전체 테스트 명령어는?**

```bash
make tests
```

**Q. API 서버 기능 테스트는 어떻게 하나요?**

```bash
bash tests/api/test_api_smoke.sh
bash tests/api/test_api_concurrency_smoke.sh
bash tests/api/test_api_parallel_select_smoke.sh
```

**꼬리질문: 동시성 테스트는 무엇을 확인하나요?**

A. `test_api_concurrency_smoke.sh`는 여러 INSERT 요청을 동시에 보내고 최종 row count가 맞는지 확인합니다. `test_api_parallel_select_smoke.sh`는 여러 SELECT 요청이 동시에 성공하고 row count가 일관적인지 확인합니다.

**Q. edge case는 어떤 것을 고려했나요?**

A. 잘못된 SQL, 누락된 semicolon, 없는 route, 잘못된 method, JSON body에 `sql` 필드 없음, explicit id 삽입 거부, 문자열에 comma 포함, DELETE 미지원, queue full 상황, 동시 요청 상황을 고려했습니다.

## 12. 성능과 병목

**Q. B+Tree 사용으로 어떤 성능 개선이 있나요?**

A. `WHERE id = ?` 조회에서 전체 rows를 훑지 않고 B+Tree로 row index를 찾습니다. 데이터가 많아질수록 선형 탐색 대비 조회 비용 차이가 커집니다. benchmark 모드에서 B+Tree lookup과 linear scan lookup 시간을 비교할 수 있습니다.

**Q. 병목은 어디인가요?**

A. 현재 구조에서 병목은 크게 세 가지입니다.

- INSERT는 write lock으로 직렬화됨
- runtime table이 하나라 multi-table workload에 약함
- JSON parser와 HTTP parser가 MVP 수준이라 복잡한 요청 처리에는 부적합

**꼬리질문: 개선한다면?**

A. table별 lock과 table registry를 도입하고, column별 index를 관리하며, HTTP/JSON 처리는 검증된 라이브러리나 더 완전한 parser로 확장할 수 있습니다.

## 13. 설계 선택 질문

**Q. 왜 직접 HTTP 서버를 만들었나요?**

A. 과제 구현 언어가 C이고 API 서버 아키텍처, thread pool, 동시성 이슈를 직접 다루는 것이 중점이어서 socket 기반으로 직접 구현했습니다.

**Q. 왜 완전한 JSON parser를 쓰지 않았나요?**

A. MVP 요구는 SQL 문자열 하나를 받는 것이므로 `"sql"` string field만 추출하는 작은 parser로 충분하다고 판단했습니다. 다만 production 수준이라면 cJSON 같은 검증된 JSON parser를 쓰는 것이 더 안전합니다.

**Q. 왜 stdout 출력 대신 `DbResult`를 만들었나요?**

A. API 서버는 stdout 문자열보다 구조화된 결과가 필요합니다. `DbResult`를 만들면 CLI는 표로 렌더링하고 API는 JSON으로 렌더링할 수 있어 실행 로직과 표현 로직이 분리됩니다.

**Q. 왜 queue full 때 기다리지 않고 503을 반환하나요?**

A. 과부하 상황에서 client connection을 무한히 붙잡지 않고 빠르게 실패시키면 서버 자원을 보호할 수 있습니다. 또한 테스트와 데모에서 동작이 명확합니다.

## 14. 발표 데모 흐름

4분 발표라면 이 순서를 추천합니다.

1. 한 문장 소개: "기존 SQL 처리기에 API 서버와 thread pool을 붙인 C 기반 mini DBMS입니다."
2. 아키텍처 그림 또는 README 흐름 설명
3. 서버 실행

```bash
make
./api_server 8080 4 16
```

4. health check

```bash
curl -i http://127.0.0.1:8080/health
```

5. INSERT

```bash
curl -i -X POST http://127.0.0.1:8080/query \
  -H "Content-Type: application/json" \
  --data '{"sql":"INSERT INTO demo_users (name, age) VALUES ('\''Alice'\'', 30);"}'
```

6. SELECT with B+Tree

```bash
curl -i -X POST http://127.0.0.1:8080/query \
  -H "Content-Type: application/json" \
  --data '{"sql":"SELECT id, name FROM demo_users WHERE id = 1;"}'
```

여기서 `"used_id_index":true`를 보여줍니다.

7. 테스트 검증 언급

```bash
make tests
```

## 15. 자주 나올 꼬리질문 빠른 답변

**Q. 이게 진짜 DBMS인가요?**

A. production DBMS는 아니고 과제 범위의 mini DBMS입니다. SQL parsing, execution, storage, index, API server, concurrency 제어라는 DBMS 핵심 요소를 작게 구현한 것입니다.

**Q. transaction은 있나요?**

A. 없습니다. 현재는 SQL 한 문장 단위 실행이고, transaction isolation/rollback은 지원하지 않습니다.

**Q. JOIN은 지원하나요?**

A. 지원하지 않습니다. 현재는 단일 table의 INSERT, SELECT 중심입니다.

**Q. 여러 SQL 문장을 한 API 요청으로 실행할 수 있나요?**

A. 현재 API는 SQL 문자열 하나 실행을 전제로 합니다. CLI file mode는 세미콜론 기준으로 여러 문장을 순차 실행할 수 있습니다.

**Q. 보안은 고려했나요?**

A. 인증, TLS, 권한 관리는 과제 범위 밖이라 구현하지 않았습니다. 다만 로컬 데모와 기능 검증을 위한 API 서버로 설계했습니다.

**Q. race condition이 정말 없나요?**

A. 공유 DB 상태 접근은 lock manager로 보호하고, tokenizer cache는 별도 mutex로 보호합니다. 다만 production 수준 검증은 더 많은 stress test가 필요합니다. 현재 과제 요구 수준에서는 동시 INSERT/SELECT smoke test로 기본 안정성을 검증했습니다.

**Q. B+Tree와 CSV 저장소가 불일치하면 어떻게 하나요?**

A. 서버 시작 또는 table load 시 CSV를 기준으로 runtime rows와 B+Tree를 재구성합니다. INSERT 중 CSV 저장 실패가 발생하면 runtime table을 unloaded로 되돌리고 storage에서 다시 로드하려고 시도합니다.

**Q. 가장 어려웠던 부분은?**

A. 기존 SQL 처리기는 CLI 출력 중심이었기 때문에 API 응답으로 쓰기 어려웠습니다. 그래서 executor 결과를 `DbResult`로 구조화하고, CLI/API 렌더링을 분리한 부분이 핵심 설계 포인트였습니다.

**Q. 포트폴리오에 어필할 포인트는?**

A. C socket 서버, thread pool, bounded queue, rwlock 기반 동시성 제어, SQL parser/executor, B+Tree index, CSV persistence, API 테스트까지 하나의 작은 시스템으로 연결했다는 점입니다.

