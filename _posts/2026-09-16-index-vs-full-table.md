---
title: "인덱스 스캔 50만 vs. 풀 테이블 스캔 100만"
date: 2026-09-16 10:00:00 +0900
---

## 서론
[Feed App](https://github.com/taeyoung-no/feed-app) 프로젝트에서 offset, limit 방식의 페이지네이션을 구현했습니다. 게시글을 100만 개 작성하고 중간과 마지막 페이지 조회의 지연을 측정하고 결과를 분석하며 겪은 시행착오를 정리했습니다.

---

## 본론
### EXPLAIN
`EXPLAIN`으로 중간 페이지 조회와 마지막 페이지 조회의 계획을 확인했습니다. 중간 페이지의 경우 `id`와 `created_at`을 기준으로 **인덱스 스캔**을 사용합니다. 마지막 페이지의 경우 **풀 테이블 스캔**을 사용합니다. 

### 첫 번째 가설: 인덱스 스캔 50만이 풀 테이블 스캔 100만보다 빠르다
중간 페이지 조회(인덱스 스캔)는 게시글 50만 개, 마지막 페이지 조회(풀 테이블 스캔)는 100만 개를 읽습니다. 따라서 인덱스 스캔 50만이 풀 테이블 스캔 100만보다 빠를 것입니다.

### 두 번째 가설: 인덱스 스캔 50만이 풀 테이블 스캔 100만보다 느리다
> If InnoDB can determine there is a high probability that data might be needed soon, it performs read-ahead operations to bring that data into the buffer pool so that it is available in memory. Making a few large read requests for contiguous data can be more efficient than making several small, spread-out requests. [MySQL 9.7 Reference Manual](https://dev.mysql.com/doc/refman/9.7/en/innodb-disk-io.html)

즉, 연속된 데이터를 읽으면 다음에 읽을 데이터를 미리 메모리에 올립니다(**read-ahead**). 풀 테이블 스캔 100만이 인덱스 스캔보다 read-ahead가 많아서 인덱스 스캔 50만이 더 느릴 것입니다.

### 첫 번째 실험 방법
스캔 방식, virtual user(VU)의 수, 콜드 스타트 여부에 따라 8가지 경우의 수에 대해 read-ahead와 지연을 측정합니다.

```sql
SHOW GLOBAL STATUS WHERE Variable_name IN (
  'Innodb_buffer_pool_reads',
  'Innodb_buffer_pool_read_ahead'
);
```

위 명령을 실행해서 디스크 I/O와 read-ahead 횟수를 측정할 수 있습니다. 지연은 부하 테스트로 측정합니다. 10번 측정해서 평균을 기록합니다. 가설과 비교합니다.

### 첫 번째 실험 결과 및 분석

| | VU | cold | disk I/O | read-ahead | p95 |
| --- | --- | --- | --- | --- | --- |
| 인덱스 스캔 50만 | 1 | true | 198 | 3072 | 0.595s |
| 인덱스 스캔 50만 | 1 | false | 0 | 0 | 0.601s |
| 인덱스 스캔 50만 | 10 | true | 76 | 2802 | 3.602s |
| 인덱스 스캔 50만 | 10 | false | 0 | 0 | 3.620s |
| 풀 테이블 스캔 100만 | 1 | true | 97 | 4634 | 0.640s |
| 풀 테이블 스캔 100만 | 1 | false | 0 | 0 | 0.652s |
| 풀 테이블 스캔 100만 | 10 | true | 37 | 4200 | 1.344s |
| 풀 테이블 스캔 100만 | 10 | false | 0 | 0 | 1.326s |

1. 한 번의 요청에 대해 인덱스 스캔 50만이 더 빠릅니다. 여러 번의 요청에 대해서는 더 느립니다. 
2. 풀 테이블 스캔의 경우 read-ahead가 더 많습니다. 
3. read-ahead와 p95가 반비례하지 않습니다. 
4. 콜드 스타트가 아니어서 디스크 I/O가 없어도 지연이 감소하지 않습니다.

두 번째 가설처럼 풀 테이블 스캔에서 read-ahead가 더 많지만 지연에는 유의미한 영향을 미치지 않는 것으로 분석합니다.

### 세 번째 가설
> You might encounter bottlenecks from multiple threads trying to access the buffer pool at once. [MySQL 8.0 Reference Manual](https://dev.mysql.com/doc/refman/8.0/en/innodb-multiple-buffer-pools.html)

한 번에 많은 스레드가 메모리에 접근하면 병목이 발생할 수 있습니다. 인덱스 스캔 50만이 메모리에 더 많이 접근하기 때문에 스캔 횟수, 즉 VU가 증가할수록 p95가 큰 폭으로 상승하는 것으로 가설을 설정했습니다.

### 두 번째 실험 방법
```sql
SHOW GLOBAL STATUS LIKE 'Innodb_buffer_pool_read_requests';
```

위 명령어로 메모리 접근 횟수를 측정할 수 있습니다. p95는 이전에 측정한 것 그대로 사용했습니다.


### 두 번째 실험 결과 및 분석

| | VU | cold | requests | p95 |
| --- | --- | --- | --- | --- |
| 인덱스 스캔 50만 | 1 | true | 4.00e7 | 0.595s |
| 인덱스 스캔 50만 | 1 | false | 4.00e7 | 0.601s |
| 인덱스 스캔 50만 | 10 | true | 4.00e8 | 3.602s |
| 인덱스 스캔 50만 | 10 | false | 4.00e8 | 3.620s |
| 풀 테이블 스캔 100만 | 1 | true | 2.01e7 | 0.640s |
| 풀 테이블 스캔 100만 | 1 | false | 2.01e7 | 0.652s |
| 풀 테이블 스캔 100만 | 10 | true | 2.01e8 | 1.344s |
| 풀 테이블 스캔 100만 | 10 | false | 2.01e8 | 1.326s |

다음과 같이 분석합니다. VU가 1일 때는 인덱스 스캔의 메모리 동시 접근 병목보다 풀 테이블 스캔의 정렬 비용이 더 커서 인덱스 스캔이 더 빠릅니다. 10일 때는 메모리 동시 접근 병목이 커져서 정렬 비용을 역전합니다.

---

## 결론
offset, limit 페이지네이션에서 뒤 페이지일수록 지연이 더 크다고 알려져 있습니다. 이 프로젝트 기준 한 번의 요청에 대해서는 사실이었으나 동시에 여러 번의 요청을 받으면 마지막 페이지 조회가 중간 페이지보다 더 빨랐습니다.

중간 페이지 조회, 즉 인덱스 스캔은 메모리에 더 많이 접근합니다. 한편 마지막 페이지 조회, 즉 풀 테이블 스캔은 메모리에 덜 접근하지만 정렬 비용이 있습니다. 동시에 여러 번 조회하면 인덱스 스캔의 메모리 접근 비용이 풀 테이블 스캔의 정렬 비용을 뛰어넘는 순간이 존재합니다.

---

## 한계
추가 실험을 통해 `index_tree_rw_lock`과 `hash_table_locks`이 병목임을 알게 됐습니다. 메모리를 동시에 읽기만 하더라도 병목이 발생하는 원인일 것으로 예상합니다. 하지만 인덱스 트리와 해시 테이블에 대한 이해가 부족해서 이 부분은 본론에 작성하지 않았습니다. 나중에 데이터베이스를 더 잘 이해하게 되면 추가할 수도 있습니다.
