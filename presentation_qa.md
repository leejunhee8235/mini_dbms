# Mini DBMS API Server 발표 Q&A

이 문서는 발표 Q&A 대비용입니다. 답변은 현재 프로젝트 구현 기준으로 정리했습니다.

발표장에서 답변할 때는 먼저 짧게 결론을 말하고, 이어서 “우리 코드에서는 어떻게 되어 있는지”를 함수명과 함께 설명하면 좋습니다.

## 0. 10초 요약 답변

**Q. 이번 프로젝트를 한 문장으로 설명하면?**

A. 기존 C 기반 SQL 처리기와 B+Tree 인덱스를 재사용해서, 외부 클라이언트가 HTTP API로 SQL을 실행할 수 있는 멀티스레드 미니 DBMS 서버를 만든 프로젝트입니다.

**꼬리질문: 핵심 구현은 무엇인가요?**

A. 핵심은 세 가지입니다.

- `GET /health`, `POST /query`를 제공하는 TCP/HTTP API 서버
- `ThreadPool + JobQueue`로 요청을 병렬 처리하는 구조
- `db_engine_facade`와 `DbResult`로 API 서버와 내부 DB 엔진을 분리한 구조

## 1. 과제 요구사항과 우리 구현 매핑

**Q. 과제 요구사항 중 무엇을 구현했나요?**

A. 요구사항 기준으로 다음을 구현했습니다.

| 과제 요구사항 | 우리 프로젝트 구현 |
| --- | --- |
| 미니 DBMS API 서버 | `src/api`의 socket 기반 HTTP API 서버 |
| 외부 클라이언트에서 DBMS 사용 | `POST /query`로 SQL 실행 가능 |
| 스레드 풀 구성 | `src/concurrency/thread_pool.c` |
| 요청마다 스레드 할당 | 요청을 queue에 넣고 worker thread가 가져가 처리 |
| SQL 처리기 재사용 | `tokenizer -> parser -> executor` 재사용 |
| B+Tree 인덱스 활용 | `SELECT ... WHERE id = n`에서 `bptree_search` 사용 |
| C 언어 구현 | Makefile 기반 C99, pthread 사용 |
| 테스트 | DB 단위 테스트, concurrency 테스트, API smoke/concurrency 테스트 |
| 엣지 케이스 고려 | 잘못된 route/method/body, queue full, explicit id, unsupported DELETE 등 |

**꼬리질문: 발표에서 가장 강조할 차별점은?**

A. 단순히 SQL 처리기에 HTTP만 붙인 것이 아니라, API 서버와 DB 엔진 사이를 `DbResult`라는 구조화된 결과 객체로 분리했고, thread pool, rwlock, tokenizer cache mutex, file lock까지 넣어 멀티스레드 환경에서 동작하도록 설계한 점입니다.

## 2. 기본 개념 질문

**Q. DBMS가 정확히 뭔가요?**

A. DBMS는 Database Management System의 약자로, 데이터를 저장하고 SQL 같은 질의 언어로 조회/삽입/수정/삭제할 수 있게 해주는 시스템입니다. 우리 프로젝트는 MySQL 같은 완성형 DB는 아니고, SQL 파싱, 실행, 저장, 인덱스, API 서버를 작게 구현한 mini DBMS입니다.

**Q. API 서버는 뭔가요?**

A. 외부 프로그램이 정해진 방식으로 기능을 호출할 수 있게 열어둔 서버입니다. 우리 프로젝트에서는 클라이언트가 HTTP 요청으로 SQL을 보내면 서버가 DB 엔진을 실행하고 JSON 응답을 돌려줍니다.

**Q. TCP는 뭔가요?**

A. TCP는 인터넷에서 데이터를 안정적으로 주고받기 위한 연결 기반 통신 방식입니다. HTTP 요청도 보통 TCP 연결 위에서 오갑니다. 우리 서버는 `socket`, `bind`, `listen`, `accept`를 사용해서 TCP 연결을 받고 그 위에서 HTTP 요청을 읽습니다.

**Q. HTTP는 뭔가요?**

A. 클라이언트와 서버가 요청과 응답을 주고받는 규칙입니다. 예를 들어 `GET /health`는 서버 상태를 묻는 요청이고, `POST /query`는 SQL을 body에 담아 보내는 요청입니다.

**Q. CLI와 REPL은 뭔가요?**

A. CLI는 Command Line Interface, 즉 터미널에서 명령어로 프로그램을 쓰는 방식입니다. REPL은 Read-Eval-Print Loop의 약자로, 사용자가 한 줄씩 입력하면 읽고 실행하고 결과를 출력한 뒤 다시 입력을 기다리는 방식입니다. 우리 CLI는 `./sql_processor`를 실행하면 `SQL>` 프롬프트가 뜨고, `.sql` 파일을 넘기면 파일 안의 SQL을 실행합니다.

**Q. Thread Pool이 뭔가요?**

A. 요청이 올 때마다 새 thread를 만들지 않고, 미리 worker thread들을 만들어 둔 뒤 요청을 queue에 넣고 worker가 하나씩 가져가 처리하는 구조입니다. thread 생성 비용과 무제한 thread 증가를 막을 수 있습니다.

**Q. Bounded Queue는 뭔가요?**

A. 크기가 제한된 queue입니다. 우리 서버는 queue capacity를 정해두고, queue가 가득 차면 더 기다리지 않고 `503 Server is busy`를 반환합니다. 이것은 서버가 과부하 상황에서 무한히 요청을 쌓지 않도록 하는 backpressure입니다.

**Q. Mutex와 RWLock은 뭐가 다른가요?**

A. mutex는 한 번에 하나의 thread만 들어오게 막는 lock입니다. rwlock은 읽기 작업은 여러 thread가 동시에 들어올 수 있고, 쓰기 작업은 단독으로 들어오게 하는 lock입니다. 우리 프로젝트는 SELECT는 read lock, INSERT/DELETE 계열은 write lock 경로를 사용합니다.

**Q. Projection이 뭔가요?**

A. SELECT 결과에서 필요한 컬럼만 골라내는 과정입니다. 예를 들어 `SELECT name FROM users`라면 전체 row에서 `name` 컬럼만 결과로 복사합니다. 우리 코드에서는 `executor_prepare_projection`, `executor_copy_projected_row`가 이 역할을 합니다.

## 3. 전체 아키텍처 질문

**Q. 전체 흐름을 설명해 주세요.**

A. API 기준 흐름은 다음과 같습니다.

```text
Client
-> TCP 연결
-> api_server_run accept loop
-> thread_pool_submit
-> JobQueue
-> worker thread
-> api_server_read_http_request
-> parse_http_request
-> route_request
-> execute_query_with_lock
-> db_execute_sql
-> tokenizer -> parser -> executor
-> TableRuntime / B+Tree / CSV storage
-> DbResult
-> JSON HTTP response
-> Client
```

**Q. CLI와 API는 서로 다른 엔진을 쓰나요?**

A. 아닙니다. 둘 다 같은 DB 엔진 경로를 사용합니다. CLI는 `db_execute_sql` 결과를 `executor_render_result_for_cli`로 터미널 출력하고, API는 `execute_query_with_lock` 결과를 `build_query_json_response`로 JSON 응답으로 바꿉니다.

**Q. 왜 `db_engine_facade`를 만들었나요?**

A. API 서버가 tokenizer, parser, executor 내부 구조를 직접 알게 되면 결합도가 커집니다. 그래서 API 서버는 SQL 문자열을 넘기고 `DbResult`만 받도록 `db_engine_facade`를 만들었습니다. 이 덕분에 CLI와 API가 같은 실행 엔진을 공유하면서도 출력 방식만 다르게 가져갈 수 있습니다.

**꼬리질문: `DbResult`에는 무엇이 들어가나요?**

A. 성공 여부, 결과 타입, 메시지, SELECT 컬럼 목록, SELECT row 목록, row count, affected rows, B+Tree 인덱스 사용 여부인 `used_id_index`가 들어갑니다.

## 4. API 서버 질문

**Q. API endpoint는 무엇이 있나요?**

A. 두 개입니다.

| Method | Path | 역할 |
| --- | --- | --- |
| `GET` | `/health` | 서버 상태, worker/queue 상태 확인 |
| `POST` | `/query` | JSON body의 `sql`을 실행 |

**Q. `/query` 요청 예시는?**

```bash
curl -i -X POST http://127.0.0.1:8080/query \
  -H "Content-Type: application/json" \
  --data '{"sql":"SELECT id, name FROM users WHERE id = 1;"}'
```

**Q. `/health`는 무엇을 반환하나요?**

A. 서버 생존 여부뿐 아니라 thread pool과 queue 상태도 반환합니다.

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

**꼬리질문: 왜 `/health`에 worker와 queue 상태를 넣었나요?**

A. 단순히 서버가 살아 있는지만 보는 것보다, 현재 부하가 worker 부족인지 queue 포화인지 확인할 수 있어야 하기 때문입니다. 특히 부하 테스트에서 병목을 설명할 때 `busy_workers`, `queue_length`, `available_queue_slots`가 유용합니다.

**꼬리질문: `/health` fast path는 왜 넣었나요?**

A. queue가 꽉 찬 상황에서도 서버 상태를 관찰할 수 있게 하기 위해서입니다. `api_server_is_health_fast_path_request`가 `GET /health`를 먼저 확인하고, 이 요청은 queue에 넣지 않고 바로 처리합니다. 실제 SQL 실행 요청인 `/query`는 thread pool을 통해 처리됩니다.

**Q. HTTP 처리는 어디까지 구현했나요?**

A. 범용 웹서버 수준은 아니고 과제에 필요한 MVP 수준입니다. request line, method, path, `Content-Length`, body를 읽고, 응답은 JSON body와 함께 `HTTP/1.1`, `Content-Type`, `Content-Length`, `Connection: close`를 붙여 보냅니다.

**꼬리질문: keep-alive를 지원하나요?**

A. 지원하지 않습니다. 요청 하나를 처리한 뒤 `Connection: close`로 소켓을 닫습니다. 과제의 핵심은 DBMS API, thread pool, 동시성 제어이므로 연결 재사용보다는 구현 범위를 명확히 했습니다.

**Q. 잘못된 요청은 어떻게 처리하나요?**

A. 잘못된 HTTP 요청이나 JSON body는 400, 없는 route는 404, 잘못된 method는 405, queue full은 503, 내부 응답 생성 실패는 500 계열로 JSON error를 반환합니다.

## 5. Thread Pool과 병렬 처리 질문

**Q. 요청마다 thread를 새로 만드나요?**

A. 아닙니다. 미리 만든 worker thread를 재사용합니다. main thread는 `accept`로 client socket을 받고 `thread_pool_submit`으로 queue에 넣습니다. worker는 `queue_pop`으로 socket fd를 가져와 요청을 처리합니다.

**Q. Thread pool 흐름을 함수명으로 설명하면?**

```text
api_server_run
-> accept
-> thread_pool_submit
-> queue_push
-> thread_pool_worker_main
-> queue_pop
-> api_server_worker_handle_client
-> api_server_handle_client
-> close(client_fd)
```

**Q. 왜 thread pool을 썼나요?**

A. 요청마다 thread를 만들면 생성/정리 비용이 크고, 요청 폭주 시 thread 수가 무제한으로 늘 수 있습니다. thread pool은 worker 수와 queue 크기를 제한해서 서버 자원 사용량을 예측 가능하게 만듭니다.

**Q. queue가 꽉 차면 어떻게 되나요?**

A. `queue_push`가 실패하고, API 서버는 해당 client에게 `503 Server is busy`를 보낸 뒤 소켓을 닫습니다. 대기열을 무한히 늘리지 않는 것이 과부하 상황에서 더 안전합니다.

**Q. queue는 thread-safe한가요?**

A. 네. `JobQueue`는 circular buffer이고, `queue_push`, `queue_pop`에서 `pthread_mutex_t`로 `head`, `tail`, `count`를 보호합니다. worker가 기다릴 때는 `pthread_cond_t not_empty`를 사용합니다.

**꼬리질문: worker 상태 통계는 race condition 없이 읽나요?**

A. `thread_pool_get_stats`가 queue mutex를 잡은 상태에서 `active_worker_count`, `queue.count`, `queue.capacity`를 읽습니다. worker가 active count를 증가/감소할 때도 같은 mutex를 사용합니다.

## 6. 멀티스레드 동시성 질문

**Q. 이 프로젝트에서 race condition이 생길 수 있는 공유 자원은 무엇인가요?**

A. 대표적으로 다음이 있습니다.

| 공유 자원 | 위험 | 보호 방식 |
| --- | --- | --- |
| `TableRuntime.rows` | INSERT 중 배열 변경 | DB write lock |
| `next_id` | 동시 INSERT 시 id 중복 | DB write lock |
| B+Tree root/node | insert 중 tree 구조 변경 | DB write lock |
| SELECT | 읽기 병렬화 필요 | DB read lock |
| tokenizer cache | 전역 cache list 변경 | tokenizer cache mutex |
| CSV file | 동시 파일 읽기/쓰기 | `flock` file lock |
| JobQueue | head/tail/count 경쟁 | queue mutex |

**Q. DB lock 정책은 어떻게 되나요?**

A. `init_lock_manager(LOCK_POLICY_SPLIT_RWLOCK)`로 초기화하고, `execute_query_with_lock`에서 SQL을 먼저 파싱해서 lock mode를 정합니다. SELECT는 read lock, INSERT/DELETE는 write lock입니다.

**꼬리질문: SELECT는 항상 read lock인가요?**

A. 대부분은 read lock입니다. 다만 아직 해당 table이 memory runtime에 load되지 않았다면 SELECT라도 CSV를 읽고 `TableRuntime`과 B+Tree를 구성해야 하므로 상태 변경이 발생합니다. 이 경우 read lock을 풀고 write lock으로 다시 잡습니다.

**꼬리질문: read lock에서 write lock으로 바로 upgrade하면 위험하지 않나요?**

A. 그래서 직접 upgrade하지 않습니다. read lock을 먼저 풀고 write lock을 다시 잡습니다. 직접 upgrade 방식은 deadlock 위험이 있으므로 피했습니다.

**Q. INSERT는 왜 write lock인가요?**

A. INSERT는 `next_id`, `rows`, B+Tree, CSV storage를 모두 바꿉니다. 동시에 여러 INSERT가 들어오면 id 중복이나 B+Tree 구조 깨짐이 생길 수 있으므로 write lock으로 직렬화합니다.

**Q. SELECT끼리는 병렬 실행되나요?**

A. table이 이미 load되어 있고 단순 조회라면 read lock으로 여러 SELECT가 동시에 들어올 수 있습니다. 단, 첫 load가 필요한 SELECT는 write lock 경로를 탑니다.

## 7. 내부 DB 엔진 질문

**Q. SQL 한 문장은 내부에서 어떻게 처리되나요?**

A. 흐름은 다음과 같습니다.

```text
SQL 문자열
-> tokenizer_tokenize
-> parser_parse
-> SqlStatement
-> executor_execute_into_result
-> TableRuntime / B+Tree / Storage
-> DbResult
```

**Q. tokenizer는 무엇을 하나요?**

A. SQL 문자열을 keyword, identifier, literal, operator, comma, semicolon 같은 token 배열로 나눕니다. 같은 SQL이 반복될 때를 대비해 soft parser cache도 사용합니다.

**Q. parser는 무엇을 하나요?**

A. token 배열을 `SqlStatement` 구조체로 바꿉니다. 예를 들어 SELECT라면 projection column, table name, WHERE 조건을 구조화합니다.

**Q. executor는 무엇을 하나요?**

A. `SqlStatement`를 실제 데이터 처리로 연결합니다. INSERT는 row를 만들고 B+Tree와 CSV에 반영합니다. SELECT는 projection을 준비하고, 조건에 따라 B+Tree 검색 또는 linear scan으로 row를 모아 `DbResult`를 만듭니다.

**Q. 데이터는 어디에 저장되나요?**

A. 영구 저장은 `data/<table>.csv`입니다. 실행 중에는 현재 활성 table 하나를 `TableRuntime`에 올려서 빠르게 접근하고, `id` 인덱스는 B+Tree로 메모리에 구성합니다.

**꼬리질문: `data` 파일이 안 보이면 데이터가 없는 건가요?**

A. 꼭 그렇지는 않습니다. CSV는 INSERT가 실행될 때 생성됩니다. 또한 `.gitignore`에 `data/*.csv`가 들어 있어서 GitHub에는 데이터 파일이 안 올라갈 수 있습니다. 즉 `data`는 실행 중 생성되는 runtime data 위치입니다.

**Q. 현재 table runtime의 한계는?**

A. 현재는 활성 table 하나를 memory runtime으로 들고 있습니다. 다른 table을 요청하면 기존 runtime을 비우고 새 table을 load합니다. 여러 table을 동시에 메모리에 유지하려면 table registry가 필요합니다.

## 8. B+Tree 인덱스 질문

**Q. B+Tree는 왜 사용했나요?**

A. DB 인덱스에서 많이 쓰이는 균형 트리 구조이고, 정렬된 key를 기반으로 빠르게 검색할 수 있습니다. 과제 요구사항도 이전 차수의 B+Tree를 활용하는 것이어서 `id` 조회 최적화에 사용했습니다.

**Q. 우리 프로젝트에서 B+Tree에는 무엇이 저장되나요?**

A. `id -> row_index`가 저장됩니다. `id`는 자동 증가 key이고, `row_index`는 `TableRuntime.rows` 배열에서 해당 row의 위치입니다.

**Q. 언제 B+Tree를 사용하나요?**

A. SELECT의 WHERE 조건이 정확히 `id = 정수` 형태일 때 사용합니다.

```sql
SELECT name FROM users WHERE id = 1;
```

이 경우 `executor_can_use_id_index`가 true를 반환하고 `bptree_search`로 row_index를 찾습니다.

**꼬리질문: 다른 컬럼 조건에서는 왜 B+Tree를 안 쓰나요?**

A. 현재 B+Tree는 `id` 전용 인덱스입니다. 다른 컬럼은 중복 값이 있을 수 있고 column별 index registry가 필요합니다. 이번 과제에서는 기존 B+Tree를 확실히 재사용하는 데 집중해 `id` exact lookup만 최적화했습니다.

**Q. B+Tree를 실제로 썼는지 어떻게 확인하나요?**

A. API SELECT 응답에 `"used_id_index": true`가 들어갑니다. 테스트에서도 `WHERE id` 조회 시 이 값이 true인지 확인합니다.

**꼬리질문: hash table이 더 빠르지 않나요?**

A. 단순 equality lookup만 보면 hash table도 좋은 선택입니다. 하지만 과제 요구사항이 B+Tree 재사용이었고, B+Tree는 정렬된 key 기반이라 이후 range query로 확장하기 좋습니다.

## 9. 지원 SQL과 한계 질문

**Q. 어떤 SQL을 지원하나요?**

A. 현재 핵심 지원은 INSERT와 SELECT입니다. SELECT는 전체 조회, 특정 컬럼 projection, 일반 WHERE 조건, `WHERE id = n` 인덱스 조회를 지원합니다.

**Q. DELETE는 지원하나요?**

A. parser와 일부 storage 테스트에는 DELETE 흐름이 있지만, 현재 memory runtime executor에서는 지원하지 않습니다. 실행하면 `"DELETE is not supported in memory runtime mode."` 에러를 반환합니다.

**꼬리질문: 왜 DELETE를 미지원으로 뒀나요?**

A. DELETE를 제대로 하려면 CSV뿐 아니라 memory rows, row_index, B+Tree까지 일관되게 갱신해야 합니다. 특히 row를 물리 삭제하면 row_index가 바뀌므로 B+Tree 전체 재구성이 필요합니다. 이번 과제의 핵심은 API 서버, thread pool, 내부 엔진 연결이라 DELETE는 명확히 unsupported로 처리했습니다.

**Q. transaction은 있나요?**

A. 없습니다. 현재는 SQL 한 문장 단위 실행입니다. rollback, commit, isolation level 같은 transaction 기능은 지원하지 않습니다.

**Q. JOIN은 되나요?**

A. 지원하지 않습니다. 현재는 단일 table 중심의 INSERT/SELECT 기능입니다.

**Q. 보안은 고려했나요?**

A. 인증, 권한, TLS는 과제 범위 밖이라 구현하지 않았습니다. 로컬 데모와 기능 검증용 API 서버입니다. 운영 환경이라면 인증, TLS, request size 제한, SQL 권한 제어가 필요합니다.

## 10. 왜 이렇게 설계했는지

**Q. 왜 직접 socket 서버를 만들었나요?**

A. 구현 언어가 C이고, 과제 중점이 API 서버 아키텍처와 thread pool, 동시성 이슈였기 때문입니다. socket을 직접 다루면 accept loop, worker 분배, queue full 처리 같은 서버 구조를 명확히 보여줄 수 있습니다.

**Q. 왜 외부 JSON 라이브러리를 안 썼나요?**

A. 현재 API body는 `{"sql":"..."}` 하나만 필요합니다. 그래서 과제 범위에서는 `"sql"` string field만 추출하는 작은 parser로 충분하다고 판단했습니다. 다만 production 수준이라면 cJSON 같은 검증된 JSON parser를 쓰는 것이 더 안전합니다.

**Q. 왜 stdout 출력 대신 `DbResult`를 만들었나요?**

A. CLI는 터미널 출력이 필요하지만 API 서버는 JSON 응답이 필요합니다. executor가 바로 `printf`하면 API로 재사용하기 어렵습니다. 그래서 실행 결과를 `DbResult`로 구조화하고, CLI와 API가 각각 원하는 방식으로 렌더링하도록 분리했습니다.

**Q. 왜 queue full에서 기다리지 않고 503을 반환하나요?**

A. 서버가 감당할 수 없는 요청을 계속 쌓으면 메모리와 socket 자원이 고갈될 수 있습니다. queue capacity를 넘으면 빠르게 실패시키는 정책이 과부하 상황에서 더 예측 가능하고 테스트하기도 좋습니다.

**Q. worker 수를 늘리면 항상 성능이 좋아지나요?**

A. 아닙니다. 특히 INSERT는 DB write lock을 잡기 때문에 worker가 많아도 실제 DB 변경 구간은 직렬화됩니다. 부하 테스트에서 worker/queue를 늘려도 처리량이 크게 늘지 않는다면 병목이 thread pool이 아니라 DB write lock 구간이라는 뜻입니다.

## 11. 테스트와 품질 질문

**Q. 어떤 테스트를 작성했나요?**

A. 크게 네 종류입니다.

| 테스트 종류 | 확인 내용 |
| --- | --- |
| DB 단위 테스트 | tokenizer, parser, storage, B+Tree, executor, facade |
| concurrency 테스트 | thread pool, tokenizer cache thread safety |
| integration 테스트 | `.sql` 파일을 CLI로 실행 |
| API 테스트 | `/health`, `/query`, 동시 INSERT, 병렬 SELECT |

**Q. 전체 테스트 명령어는?**

```bash
make tests
```

**Q. API 서버 실행 명령어는?**

```bash
make
./api_server 8080 4 16
```

여기서 `8080`은 port, `4`는 worker 수, `16`은 queue capacity입니다.

**Q. CLI 실행 명령어는?**

```bash
./sql_processor
./sql_processor tests/integration/test_cases/basic_insert.sql
```

**Q. 부하 테스트는 무엇을 확인했나요?**

A. `scripts/k6_queue_health_demo.js`로 많은 `/query` 요청을 넣고, `/health`로 worker와 queue 상태를 관찰했습니다. 이를 통해 “worker 수 문제인지, queue 문제인지, DB write lock 병목인지”를 구분하려고 했습니다.

**Q. 엣지 케이스는 무엇을 고려했나요?**

A. 잘못된 SQL, 잘못된 route, 잘못된 method, JSON body에 `sql` 없음, explicit id 삽입 거부, 문자열 안 comma 처리, DELETE unsupported, queue full, 동시 INSERT/SELECT 상황을 고려했습니다.

## 12. 자주 나올 꼬리질문 빠른 답변

**Q. 이게 진짜 DBMS인가요?**

A. production DBMS는 아니고 mini DBMS입니다. 그래도 SQL 파싱, 실행, 저장, 인덱스, API 서버, 동시성 제어라는 DBMS 핵심 요소를 작게 연결했습니다.

**Q. race condition이 완전히 없다고 말할 수 있나요?**

A. 과제 범위에서 확인한 공유 자원은 lock으로 보호했습니다. 다만 production 수준의 검증은 더 긴 stress test와 sanitizer, 더 많은 장애 케이스 테스트가 필요합니다.

**Q. API 요청은 정말 병렬 처리되나요?**

A. 네. 여러 client 연결은 thread pool worker가 병렬로 처리합니다. 다만 DB 내부에서 write lock이 필요한 구간은 정합성을 위해 직렬화됩니다.

**Q. `/health`는 thread pool을 안 타는데 요구사항과 충돌하지 않나요?**

A. 핵심 SQL 요청인 `/query`는 thread pool을 탑니다. `/health`는 서버 관찰용 endpoint라 queue가 꽉 찬 상황에서도 상태를 확인할 수 있도록 fast path로 처리했습니다. 기능 요구사항과 운영 관찰성을 함께 고려한 선택입니다.

**Q. CSV와 B+Tree가 불일치하면 어떻게 하나요?**

A. table load 시 CSV를 기준으로 memory rows와 B+Tree를 다시 구성합니다. INSERT 중 storage 저장이 실패하면 runtime table을 unloaded 상태로 되돌리고 storage에서 다시 load하려고 합니다.

**Q. 가장 어려웠던 부분은?**

A. 기존 SQL 처리기가 CLI 출력 중심이었기 때문에 API 응답으로 쓰기 어려웠습니다. 그래서 executor 결과를 `DbResult`로 구조화하고 CLI/API 렌더링을 분리한 부분이 핵심 설계 포인트였습니다.

**Q. 개선한다면 무엇을 먼저 하겠나요?**

A. table registry를 만들어 여러 table runtime을 동시에 유지하고, table별 lock을 적용해 병렬성을 높이겠습니다. 그 다음 DELETE의 B+Tree 재구성, secondary index, production 수준 JSON parser를 추가하겠습니다.

## 13. 데모용 명령어 모음

빌드:

```bash
make
```

서버 실행:

```bash
./api_server 8080 4 16
```

health check:

```bash
curl -i http://127.0.0.1:8080/health
```

INSERT:

```bash
curl -i -X POST http://127.0.0.1:8080/query \
  -H "Content-Type: application/json" \
  --data '{"sql":"INSERT INTO demo_users (name, age) VALUES (\"Alice\", 30);"}'
```

B+Tree 사용 SELECT:

```bash
curl -i -X POST http://127.0.0.1:8080/query \
  -H "Content-Type: application/json" \
  --data '{"sql":"SELECT id, name FROM demo_users WHERE id = 1;"}'
```

응답에서 확인할 부분:

```json
{
  "ok": true,
  "type": "select",
  "used_id_index": true,
  "row_count": 1
}
```

전체 테스트:

```bash
make tests
```

