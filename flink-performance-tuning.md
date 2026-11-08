# Apache Flink 개념과 성능 튜닝

『가상 면접 사례로 배우는 대규모 시스템 설계 기초 2』 6장 "광고 클릭 이벤트 집계"의 참고 문헌 중 하나인 Flink 공식 문서 [Performance Tuning](https://nightlies.apache.org/flink/flink-docs-stable/docs/dev/table/tuning/)을 정리했습니다. 튜닝 문서는 Flink를 안다는 전제로 쓰여 있어서, 앞부분에 Flink가 무엇이고 어디에 쓰는지를 먼저 정리합니다.

- 기준 버전: Apache Flink 2.3 (2026-11 기준 stable 문서)
- 그림 출처: Apache Flink 공식 문서 (Apache License 2.0)

---

## 1. Flink는 무엇인가

**Apache Flink는 끝없이 들어오는 데이터 스트림을 실시간으로 계산하는 분산 처리 엔진입니다.** 공식 소개 문구는 "unbounded / bounded 데이터 스트림 위의 상태 기반(stateful) 연산을 위한 프레임워크이자 분산 처리 엔진"입니다.

풀어 쓰면 이렇습니다.

- **스트림 처리**: Kafka 같은 곳에서 이벤트가 들어오는 즉시 하나씩 처리합니다. 데이터를 모아 두었다가 한 번에 돌리는 배치(일괄) 처리와 반대입니다.
- **상태 기반(stateful)**: "광고 A의 지금까지 클릭 수"처럼 이전 이벤트의 계산 결과를 기억해 두고 다음 이벤트에 이어서 씁니다. 이 상태를 Flink가 직접 저장하고 장애 시 복구까지 책임집니다.
- **분산**: 작업을 여러 서버(TaskManager)에 나눠 병렬로 처리합니다.
- **unbounded / bounded**: 끝이 없는 스트림(unbounded)도, 끝이 있는 데이터(bounded, 즉 배치)도 같은 엔진으로 처리합니다. 배치를 "끝이 있는 스트림"으로 봅니다.

### 6장 광고 클릭 집계와의 연결

6장에서 집계 서비스가 해야 했던 일을 거의 그대로 Flink가 기본 기능으로 제공합니다. 그래서 책이 Flink를 참고 문헌으로 든 것입니다.

| 6장에서 나온 요구사항                       | Flink가 제공하는 기능                           |
| ---------------------------------- | ---------------------------------------- |
| 카파 아키텍처: 실시간 처리와 재처리를 하나의 스트림 엔진으로 | 같은 Flink 잡으로 실시간 처리와 과거 데이터 재처리를 모두 수행   |
| 이벤트 시각 기준 집계                       | Event Time 처리                            |
| 늦게 도착한 이벤트 처리 (워터마크)               | Watermark 기본 지원                          |
| 1분 단위 텀블링 윈도, 슬라이딩 윈도              | Tumbling / Sliding / Session 윈도          |
| 정확히 한 번(exactly-once) 처리           | Checkpoint + 트랜잭션 싱크로 exactly-once 보장    |
| 집계 중 장애 시 복구                       | Checkpoint에서 상태와 Kafka 오프셋을 함께 복원        |
| 핫스팟(특정 광고에 클릭 집중)                  | Local-Global Aggregation 등 튜닝 옵션 (아래 3장) |

## 2. 핵심 개념

### 2.1 아키텍처: JobManager와 TaskManager

- **JobManager**: 잡을 받아 실행 계획을 짜고, 작업을 배분하고, 체크포인트를 조율하는 관리자입니다.
- **TaskManager**: 실제로 데이터를 처리하는 워커입니다. 각 TaskManager에는 작업을 올릴 수 있는 칸(task slot)이 여러 개 있습니다.
- **병렬도(parallelism)**: 연산자 하나를 몇 개의 병렬 태스크로 나눠 돌릴지 정하는 값입니다. Kafka 파티션 수와 맞추는 경우가 많습니다.

잡은 `소스 → 변환 연산자들 → 싱크`로 이어지는 데이터 흐름 그래프(dataflow)입니다. 예를 들어 `Kafka 소스 → ad_id별 keyBy → 1분 윈도 집계 → DB 싱크` 같은 모양입니다.

### 2.2 상태(State)와 State Backend

- `keyBy(ad_id)`처럼 키로 데이터를 나누면, 같은 키의 이벤트는 항상 같은 태스크로 갑니다. 각 태스크는 자기 키들의 상태(keyed state)를 로컬에 들고 있습니다.
- 상태를 어디에 저장할지는 **State Backend**가 정합니다.
  - **HashMap State Backend**: JVM 힙 메모리에 저장합니다. 빠르지만 상태 크기가 메모리에 묶입니다.
  - **RocksDB State Backend**: 로컬 디스크의 RocksDB에 저장합니다. 큰 상태를 감당하지만 접근할 때마다 직렬화/역직렬화 비용이 듭니다. 아래 튜닝 문서에서 "RocksDB에서 오버헤드가 크다"고 하는 이유가 이것입니다.

### 2.3 시간, 워터마크, 윈도

- **Event Time vs Processing Time**: 6장의 "이벤트 시각 / 처리 시각"과 같은 개념입니다. 정확도가 중요하면 Event Time을 씁니다.
- **Watermark**: "이 시각 이전 이벤트는 이제 다 왔다고 보자"는 신호입니다. 워터마크가 윈도 끝을 넘으면 그 윈도를 닫고 결과를 내보냅니다. 6장의 "윈도를 조금 늦게 닫는다"가 Flink에서는 `WATERMARK FOR ts AS ts - INTERVAL '15' SECOND`처럼 선언하는 것입니다.
- **Window**: Tumbling(겹치지 않는 고정 구간), Sliding(겹치는 구간), Session(활동 간격 기준) 윈도를 지원합니다.

### 2.4 Checkpoint와 Exactly-once

- Flink는 주기적으로 모든 연산자의 상태와 소스의 읽은 위치(Kafka 오프셋)를 **함께** 스냅숏으로 저장합니다. 이것이 **Checkpoint**입니다.
- 장애가 나면 마지막 체크포인트로 상태와 오프셋을 같이 되돌리고 거기서부터 다시 읽습니다. 상태와 오프셋이 항상 짝이 맞으므로 내부 상태 기준으로 중복이나 누락이 생기지 않습니다.
- 외부 싱크까지 exactly-once로 만들려면 Kafka 트랜잭션처럼 체크포인트와 연동되는 싱크(2단계 커밋)를 씁니다. 6장의 "다운스트림 확인 응답을 받은 뒤 오프셋을 저장해야 한다"는 고민을 Flink가 체크포인트로 대신 처리하는 셈입니다.
- **Savepoint**: 사람이 직접 찍는 체크포인트입니다. 코드를 고쳐 재배포하거나 과거 시점부터 재처리할 때 씁니다.

### 2.5 API 계층

| API | 설명 |
|---|---|
| Flink SQL / Table API | SQL로 스트림을 다룹니다. 가장 많이 쓰이고, 아래 튜닝 문서도 이 계층 대상입니다 |
| DataStream API | Java로 연산자를 직접 조립합니다. 세밀한 제어가 필요할 때 씁니다 |
| ProcessFunction | 상태와 타이머를 직접 다루는 가장 저수준 API입니다 |

6장의 "1분 단위 광고별 클릭 수"를 Flink SQL로 쓰면 대략 이렇습니다.

```sql
CREATE TABLE ad_clicks (
  ad_id STRING,
  user_id STRING,
  click_ts TIMESTAMP(3),
  WATERMARK FOR click_ts AS click_ts - INTERVAL '15' SECOND  -- 15초 늦게 온 이벤트까지 받음
) WITH ('connector' = 'kafka', ...);

SELECT ad_id, window_start, window_end, COUNT(*) AS click_count
FROM TABLE(
  TUMBLE(TABLE ad_clicks, DESCRIPTOR(click_ts), INTERVAL '1' MINUTE)  -- 1분 텀블링 윈도
)
GROUP BY ad_id, window_start, window_end;
```

## 3. 용도와 다른 도구 비교

### 3.1 어디에 쓰나

Flink 공식 문서가 꼽는 대표 용도는 세 가지입니다.

- **이벤트 기반 애플리케이션**: 이벤트가 오면 상태를 보고 바로 반응합니다. 예: 이상 거래 탐지, 실시간 추천, 규칙 기반 알림.
- **스트리밍 분석**: 실시간 대시보드와 지표 집계입니다. 예: 광고 클릭 집계(6장), 실시간 매출 집계, 모니터링 지표 계산(5장).
- **데이터 파이프라인 / ETL**: 데이터를 실시간으로 변환해 다른 저장소로 옮깁니다. 예: Kafka → 정제 → 데이터 레이크, CDC로 DB 변경분 동기화.

### 3.2 다른 도구와 비교

| 도구 | 처리 방식 | 특징 |
|---|---|---|
| Flink | 레코드 단위 스트리밍 | 지연이 낮고 상태 관리, 이벤트 시간, exactly-once가 강함. 별도 클러스터 운영 필요 |
| Spark Structured Streaming | 마이크로배치(기본) | Spark 배치 생태계와 통합. 지연은 보통 Flink보다 큼 |
| Kafka Streams | 라이브러리 | 별도 클러스터 없이 애플리케이션에 내장. 입출력이 Kafka 중심 |
| Hadoop MapReduce / 배치 Spark | 배치 | 하루치 재계산, 6장의 조정(reconciliation) 같은 작업에 적합 |

---

## 4. Performance Tuning (공식 문서 정리)

SQL은 데이터 분석에서 가장 널리 쓰이는 언어입니다. Flink의 Table API와 SQL을 쓰면 적은 코드로 효율적인 스트리밍 분석을 만들 수 있습니다. 다만 **모든 최적화가 기본으로 켜져 있지는 않아서**, 워크로드에 따라 아래 옵션을 켜면 성능이 좋아질 수 있습니다.

> 이 문서의 스트리밍 집계 최적화는 Group Aggregation과 Window TVF Aggregation(Session Window TVF Aggregation 제외)에 모두 적용됩니다.

### MiniBatch Aggregation

**문제**: 기본 Group Aggregation은 레코드를 하나씩 처리합니다. 레코드 하나마다 ① 상태에서 accumulator를 읽고 ② 누적/철회하고 ③ 다시 상태에 씁니다. 이 상태 접근 비용이 크고, RocksDB State Backend에서는 특히 큽니다. 데이터 쏠림(skew)이 있으면 백프레셔도 심해집니다.

**해결**: 입력을 연산자 안에 한 묶음(bundle) 모았다가 처리합니다. 그러면 같은 키의 상태는 묶음당 한 번만 접근합니다. **처리량은 크게 오르지만 지연이 늘어나는 트레이드오프**가 있습니다.

![MiniBatch Aggregation](https://nightlies.apache.org/flink/flink-docs-release-2.3/fig/table-streaming/minibatch_agg.png)

- Group Aggregation에서는 **기본 비활성화**입니다.
- Window TVF Aggregation은 설정과 상관없이 **항상 mini-batch를 씁니다.** 레코드를 JVM 힙이 아닌 managed memory에 버퍼링하므로 GC나 OOM 걱정이 없습니다.

```java
TableEnvironment tEnv = ...;
TableConfig configuration = tEnv.getConfig();
configuration.set("table.exec.mini-batch.enabled", "true");      // mini-batch 최적화 활성화
configuration.set("table.exec.mini-batch.allow-latency", "5 s"); // 입력을 최대 5초 버퍼링
configuration.set("table.exec.mini-batch.size", "5000");         // 집계 연산자 태스크당 최대 버퍼 레코드 수
```

```python
t_env = ...
configuration = t_env.get_config()
configuration.set("table.exec.mini-batch.enabled", "true")
configuration.set("table.exec.mini-batch.allow-latency", "5 s")
configuration.set("table.exec.mini-batch.size", "5000")
```

| 옵션 | 의미 |
|---|---|
| `table.exec.mini-batch.enabled` | mini-batch 활성화 여부 |
| `table.exec.mini-batch.allow-latency` | 버퍼링을 허용할 최대 지연 시간 |
| `table.exec.mini-batch.size` | 연산자 태스크당 버퍼링할 최대 레코드 수 |

### Local-Global Aggregation

**문제**: 데이터 쏠림입니다. 6장의 핫스팟 문제와 같습니다.

**해결**: 집계를 두 단계로 나눕니다. 먼저 업스트림에서 로컬 집계를 하고, 다운스트림에서 글로벌 집계를 합니다. MapReduce의 Combine + Reduce와 같은 구조입니다.

```sql
SELECT color, sum(id)
FROM T
GROUP BY color
```

- 로컬 단계는 mini-batch 안에서 같은 키의 입력을 미리 하나의 accumulator로 합칩니다.
- 글로벌 단계는 원본 레코드 대신 줄어든 accumulator만 받습니다.
- 그래서 **네트워크 셔플과 상태 접근이 크게 줄어듭니다.** 로컬 집계 한 번에 줄어드는 양은 mini-batch 크기에 달려 있습니다.

![Local-Global Aggregation](https://nightlies.apache.org/flink/flink-docs-release-2.3/fig/table-streaming/local_agg.png)

Local-Global은 **mini-batch가 켜져 있어야** 동작합니다.

```java
TableConfig configuration = tEnv.getConfig();
configuration.set("table.exec.mini-batch.enabled", "true"); // local-global은 mini-batch가 필요함
configuration.set("table.exec.mini-batch.allow-latency", "5 s");
configuration.set("table.exec.mini-batch.size", "5000");
configuration.set("table.optimizer.agg-phase-strategy", "TWO_PHASE"); // 2단계(local-global) 집계 활성화
```

> 6장과 연결: 책은 핫스팟 해결책으로 "집계 노드를 더 할당한다"를 제시합니다. Flink에서는 Local-Global로 쏠린 키를 업스트림 여러 태스크에서 미리 나눠 집계해서 같은 효과를 냅니다.

### Split Distinct Aggregation

**문제**: Local-Global은 SUM, COUNT, MAX, MIN, AVG 같은 일반 집계에는 잘 듣지만 **distinct 집계에는 효과가 작습니다.**

```sql
SELECT day, COUNT(DISTINCT user_id)
FROM T
GROUP BY day
```

`user_id`가 성기면(값이 거의 겹치지 않으면) 로컬 단계에서 합쳐도 accumulator에 원본 레코드가 거의 다 남습니다. 결국 글로벌 단계의 단일 태스크(하루당 하나)에 부하가 몰려 병목이 됩니다.

> **Q. distinct 집계에서는 왜 Local-Global 효과가 작을까?**
>
> 차이는 로컬 단계가 글로벌 단계로 보내는 **accumulator의 크기**에 있습니다.
>
> - **SUM, COUNT, MAX 같은 일반 집계**는 레코드가 1만 건이어도 accumulator가 숫자 하나입니다. 로컬 태스크는 1만 건을 `sum=12345` 한 줄로 압축해서 보냅니다. 글로벌 단계는 로컬 태스크들이 보낸 숫자를 더하기만 하면 됩니다.
> - **COUNT(DISTINCT user_id)** 는 로컬 단계가 숫자만 보낼 수 없습니다. 같은 `user_id`가 다른 로컬 태스크에도 나타날 수 있기 때문입니다. 로컬 A가 "3명", 로컬 B가 "3명"이라고만 보내면, 겹치는 사용자가 있는지 몰라서 6명인지 4명인지 알 수 없습니다. 그래서 로컬 단계는 **본 `user_id` 집합 자체**를 보내야 하고, 글로벌 단계가 집합을 합쳐 중복을 제거합니다.
> - 이때 `user_id`가 성기면(한 mini-batch 안에서 같은 사용자가 거의 반복되지 않으면) 집합이 거의 줄지 않습니다. 1만 건을 받으면 집합에도 `user_id`가 1만 개 가까이 남습니다. 로컬에서 압축되는 양이 거의 없으니, 셔플량과 글로벌 단계의 상태 크기가 Local-Global을 안 쓸 때와 비슷합니다.
> - 게다가 글로벌 단계는 여전히 `day` 하나당 태스크 하나입니다. 그날의 모든 `user_id` 집합이 그 태스크 하나에 몰리므로 핫스팟이 그대로 남습니다.
>
> 그래서 Split Distinct는 집합을 압축하는 대신 **집합 자체를 버킷으로 쪼갭니다.** `HASH_CODE(user_id) % 1024`로 나누면 같은 사용자는 항상 같은 버킷에 들어가므로, 버킷끼리는 사용자가 겹치지 않습니다. 따라서 각 버킷의 distinct 개수를 **숫자로** 보내고 2단계에서 `SUM`만 하면 됩니다. 결국 distinct 집계를 Local-Global이 잘 처리하는 일반 집계(SUM) 형태로 바꾸는 것입니다.

**해결**: distinct 집계를 두 단계로 자동 분할합니다.

1. 1단계: 그룹 키 + 버킷 키로 셔플합니다. 버킷 키는 `HASH_CODE(distinct_key) % BUCKET_NUM`입니다.
2. 2단계: 원래 그룹 키로 셔플하고 `SUM`으로 각 버킷의 `COUNT DISTINCT` 값을 합칩니다.

같은 distinct 키는 항상 같은 버킷에 들어가므로 결과는 원래 쿼리와 같습니다. `BUCKET_NUM`은 기본 **1024**이며 `table.optimizer.distinct-agg.split.bucket-num`으로 바꿉니다.

```sql
-- Flink가 자동으로 바꿔 주는 형태
SELECT day, SUM(cnt)
FROM (
    SELECT day, COUNT(DISTINCT user_id) as cnt
    FROM T
    GROUP BY day, MOD(HASH_CODE(user_id), 1024)
)
GROUP BY day
```

![Split Distinct Aggregation](https://nightlies.apache.org/flink/flink-docs-release-2.3/fig/table-streaming/distinct_split.png)

그림에서 색은 day, 글자는 user_id입니다. 분할하면 핫스팟 키의 부하가 여러 버킷으로 흩어집니다.

- 서로 다른 키에 대한 여러 distinct 집계(`COUNT(DISTINCT a), SUM(DISTINCT b)`)와 일반 집계(`SUM`, `MAX`, `MIN`, `COUNT`)가 섞인 쿼리도 분할할 수 있습니다.
- **제약**: 사용자 정의 AggregateFunction이 포함된 집계는 분할을 지원하지 않습니다.

```java
tEnv.getConfig().set("table.optimizer.distinct-agg.split.enabled", "true"); // distinct 집계 분할 활성화
```

```python
t_env.get_config().set("table.optimizer.distinct-agg.split.enabled", "true")
```

### Use FILTER Modifier on Distinct Aggregates

같은 컬럼을 조건만 바꿔 여러 번 `COUNT(DISTINCT ...)` 하는 경우가 있습니다. 예를 들어 전체 UV, 앱 UV, 웹 UV를 함께 구하는 쿼리입니다. 보통 `CASE WHEN`으로 씁니다.

```sql
SELECT
 day,
 COUNT(DISTINCT user_id) AS total_uv,
 COUNT(DISTINCT CASE WHEN flag IN ('android', 'iphone') THEN user_id ELSE NULL END) AS app_uv,
 COUNT(DISTINCT CASE WHEN flag IN ('wap', 'other') THEN user_id ELSE NULL END) AS web_uv
FROM T
GROUP BY day
```

이럴 때는 SQL 표준인 **`FILTER` 수식어**를 쓰는 편이 낫습니다.

```sql
SELECT
 day,
 COUNT(DISTINCT user_id) AS total_uv,
 COUNT(DISTINCT user_id) FILTER (WHERE flag IN ('android', 'iphone')) AS app_uv,
 COUNT(DISTINCT user_id) FILTER (WHERE flag IN ('wap', 'other')) AS web_uv
FROM T
GROUP BY day
```

옵티마이저가 "같은 distinct 키에 필터만 다르다"는 것을 알아보고, **세 개의 상태 대신 하나의 공유 상태**를 씁니다. 상태 접근과 상태 크기가 줄어듭니다. 별도 설정 없이 쿼리만 바꾸면 됩니다.

### MiniBatch Regular Joins

**문제**: 기본 Regular Join도 레코드를 하나씩 처리합니다. 상대편 상태를 조회하고, 자기 상태를 갱신하고, 결과를 내보냅니다. 상태 접근 부담이 크고, Join이 여러 단계로 이어지면 중간 결과가 불어나는(record amplification) 문제가 있습니다.

**해결**: Join 연산자 안에 입력을 버퍼링했다가 크기나 시간 임계값에 도달하면 처리합니다. 핵심 최적화는 두 가지입니다.

1. 버퍼링한 레코드를 미리 접어(fold) Join 전에 데이터 양을 줄입니다.
2. 버퍼링한 레코드를 처리할 때 불필요한 중간 결과 출력을 최대한 억제합니다.

```sql
SET 'table.exec.mini-batch.enabled' = 'true';
SET 'table.exec.mini-batch.allow-latency' = '5S';
SET 'table.exec.mini-batch.size' = '5000';

SELECT a.id as a_id, a.a_content, b.id as b_id, b.b_content
FROM a LEFT JOIN b
ON a.id = b.id
```

예시에서 두 입력 모두 Join 키(`id`)에 포함되는 unique key를 가집니다.

![MiniBatch Regular Joins](https://nightlies.apache.org/flink/flink-docs-release-2.3/fig/table-streaming/minibatch_join.png)

Regular Join의 mini-batch는 **기본 비활성화**이며, 위의 mini-batch 옵션 세 개로 켭니다.

### Multiple Regular Joins

**문제**: 시간 조건이 없는 Regular Join을 여러 개 이어 붙이면, 중간 결과를 담는 상태가 원본 입력보다 커질 수 있습니다. 이 때문에 잡이 불안정해지고 성능이 떨어집니다.

#### The MultiJoin Operator

Flink 2.1에서 여러 입력을 **한 번에** Join하는 MultiJoin 연산자가 들어왔습니다. 이진 Join을 줄줄이 잇는 대신 하나의 연산자가 모든 입력을 Join하므로 **중간 상태가 0**입니다. 저장 공간 대신 재계산을 택하는 방식입니다.

- 상태 크기가 크게 줄어듭니다.
- 레코드 증폭이 있는 체인 Join의 성능이 좋아집니다.
- 이진 Join은 처리량에 따라 상태가 다항식으로 늘지만, MultiJoin은 **선형으로** 늘어납니다. 그래서 초기화와 장애 복구도 빨라집니다.

#### When to enable the MultiJoin?

여러 Join이 **적어도 하나의 공통 Join 키**를 공유하고, 중간 상태가 원본보다 클 때 켭니다. 특히 다음 경우에 권장합니다.

- 공통 키의 선택도가 높을 때 (키 하나당 레코드가 적을 때)
- 중간 상태가 꽤 큰 체인 Join이 여러 개일 때
- 공통 키에 심한 쏠림이 없을 때
- Join 상태가 큰 경우 (50GB 이상)

키당 레코드가 많은(선택도가 낮은) 경우에는 기존 이진 Join이 낫습니다.

#### How to enable the MultiJoin?

전역 설정으로 켭니다.

```sql
SET 'table.optimizer.multi-join.enabled' = 'true';
```

또는 쿼리별 힌트로 켭니다.

```sql
SELECT /*+ MULTI_JOIN(t1, t2, t3) */ * FROM t1
JOIN t2 ON t1.id = t2.id
JOIN t3 ON t1.id = t3.id;
```

설정과 힌트가 둘 다 있으면 **설정이 우선**합니다.

현재 **실험적 기능**이며 스트리밍 INNER / LEFT Join만 지원합니다. 공통 키가 하나 이상 있어야 합니다.

```sql
-- 지원
A JOIN B ON A.key = B.key JOIN C ON A.key = C.key
-- 지원 (전이성: A.key = B.key = C.key)
A JOIN B ON A.key = B.key JOIN C ON B.key = C.key
-- 미지원 → MultiJoin 연산자 여러 개로 나뉨
A JOIN B ON A.key1 = B.key1 JOIN C ON B.key2 = C.key2
```

#### MultiJoin Operator Example - Benchmark

10개 테이블을 `tenant_id`로 LEFT JOIN하는 벤치마크입니다.

![MultiJoin 벤치마크](https://nightlies.apache.org/flink/flink-docs-release-2.3/fig/table-streaming/multijoin_operator.png)

- **처리 성능**: 둘 다 100% busy일 때 처리한 레코드 수가 2배에서 100배 이상까지 늘었습니다.
- **상태 크기**: 중간 상태가 커질수록 3배에서 1000배 이상까지 작아졌습니다.
- 실험 조건: tenant_id당 레코드 1개, upsert Kafka 토픽 10개, 병렬도 10, RocksDB(unaligned + incremental checkpoint), TaskManager 프로세스 메모리 8GB, blackhole 싱크.

```sql
INSERT INTO JoinResultsMJ
SELECT *all fields*
FROM TenantKafka t
         LEFT JOIN SuppliersKafka s ON t.tenant_id = s.tenant_id AND ...
         LEFT JOIN ProductsKafka p ON t.tenant_id = p.tenant_id AND ...
         -- CategoriesKafka, OrdersKafka, CustomersKafka, WarehousesKafka,
         -- ShippingKafka, PaymentKafka, InventoryKafka도 같은 패턴
```

### Delta Joins

**문제**: Regular Join은 양쪽 입력의 과거 데이터를 전부 상태에 들고 있어야 해서, 상태가 계속 커집니다.

**해결**: 상태 대신, 소스 테이블이 원래 저장된 **외부 저장소를 직접 인덱스로 조회**하는 양방향 lookup Join으로 바꿉니다. Flink 상태와 외부 저장소에 같은 데이터가 중복으로 쌓이지 않아 상태 크기와 자원 사용이 줄어듭니다.

**기본 활성화**입니다. 다만 자동 변환되려면 두 조건을 모두 만족해야 합니다.

1. SQL 패턴이 아래 지원 조건에 맞아야 합니다.
2. 소스 테이블의 외부 저장소가 인덱스 정보를 제공해야 합니다. 현재는 Apache Fluss가 테이블 단위 인덱스 정보를 제공합니다.

#### Working Principle

![Delta Join](https://nightlies.apache.org/flink/flink-docs-release-2.3/fig/table-streaming/delta_join.png)

상태를 조회하는 대신 외부 저장소에 인덱스 기반 쿼리를 날려 상대편 데이터를 찾습니다.

#### Important Configurations

끄려면 다음과 같이 설정합니다.

```sql
SET 'table.optimizer.delta-join.strategy' = 'NONE';
```

튜닝 옵션:

- `table.exec.delta-join.cache-enabled`
- `table.exec.delta-join.left.cache-size`
- `table.exec.delta-join.right.cache-size`

#### Supported Features and Limitations

지원:

1. INSERT만 있는 소스 테이블
2. DELETE가 없는 CDC 테이블
3. 소스와 Delta Join 사이의 projection, filter
4. Delta Join 연산자 내부 캐시
5. 연쇄 Delta Join (소스부터 싱크 방향으로 차례로 변환)
6. Delta Join 뒤의 lookup Join
7. Delta Join과 다운스트림 사이 projection/filter의 비결정적 함수

제약 (아래에 해당하면 최적화되지 않음):

1. 테이블의 인덱스 키가 Join 동등 조건에 포함돼야 합니다.
2. INNER JOIN만 지원합니다.
3. 다운스트림이 중복 변경을 처리할 수 있어야 합니다. 예: `upsertMaterialize` 없는 UPSERT 모드 싱크.
4. CDC 스트림이면 Join 키가 upsert 키에 포함돼야 합니다.
5. CDC 스트림이면 모든 필터가 upsert 키에 걸려야 합니다.
6. 소스와 Delta Join 사이의 projection/filter에는 비결정적 함수를 쓸 수 없습니다.

참고: 테이블 단위 "immutable columns" 제약으로 upsert 키를 보강할 수 있으며, Apache Fluss가 지원을 계획 중입니다.

---

## 5. 한눈에 보기

| 최적화 | 대상 | 기본값 | 켜는 방법 | 효과 |
|---|---|---|---|---|
| MiniBatch Aggregation | Group Agg | 꺼짐 (Window TVF는 항상 켜짐) | `table.exec.mini-batch.*` | 상태 접근 감소, 처리량 증가, 지연 증가 |
| Local-Global Aggregation | Group Agg (쏠림) | 꺼짐 | mini-batch + `agg-phase-strategy=TWO_PHASE` | 셔플·상태 접근 감소, 핫스팟 완화 |
| Split Distinct Aggregation | `COUNT(DISTINCT)` 쏠림 | 꺼짐 | `distinct-agg.split.enabled=true` | distinct 핫스팟을 버킷으로 분산 |
| FILTER 수식어 | 조건별 다중 distinct | 쿼리 작성법 | `CASE WHEN` 대신 `FILTER` | 상태 공유로 크기·접근 감소 |
| MiniBatch Regular Join | Regular Join | 꺼짐 | `table.exec.mini-batch.*` | 상태 접근·중복 출력 감소 |
| MultiJoin | 공통 키 다중 Join | 꺼짐 (실험적) | `multi-join.enabled=true` 또는 힌트 | 중간 상태 제거 |
| Delta Join | INNER Join + Fluss | 켜짐 | `delta-join.strategy=NONE`으로 끔 | Join 상태를 외부 저장소 조회로 대체 |

**한 줄 요약:** Flink는 Kafka 같은 스트림을 이벤트 시간·워터마크·체크포인트로 정확하게 실시간 집계하는 엔진이고, 튜닝 문서의 핵심은 "레코드 하나씩 상태를 건드리지 말고 묶어서 처리하라(mini-batch)", "쏠린 키는 나눠서 미리 집계하라(local-global, split distinct)", "Join 상태는 줄이거나 없애라(MultiJoin, Delta Join)"입니다.
