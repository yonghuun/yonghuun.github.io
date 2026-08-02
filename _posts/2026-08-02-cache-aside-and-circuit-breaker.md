---
title: Cache-Aside와 Circuit Breaker로 장애에 강한 조회 구조 설계하기
date: 2026-08-02 00:00:00 +0900
categories: [Tech, Backend]
tags: [cache-aside, circuit-breaker, redis, resilience4j, spring-boot]
description: Cache-Aside의 조회·무효화 전략과 Circuit Breaker의 상태 전이·fallback 설계를 함께 정리한 학습 기록
---

캐시와 외부 API는 응답 성능과 기능 확장에 도움을 주지만, 새로운 장애 지점이 되기도 합니다. Redis가 느려졌을 때 모든 조회가 함께 느려지거나 외부 API 장애가 요청 스레드를 계속 점유하면, 일부 기능의 문제가 서비스 전체로 번질 수 있습니다.

이런 상황을 어떻게 다룰 수 있는지 알아보기 위해 **Cache-Aside**와 **Circuit Breaker**를 함께 정리했습니다.

> 이 글은 두 패턴의 원리와 적용 방법을 학습하고, 여행 서비스에 적용한다면 어떤 구조가 적합할지 검토한 기록입니다. 실제 운영 수치는 부하 테스트와 모니터링을 통해 조정해야 합니다.
{: .prompt-info }

## Cache-Aside란

Cache-Aside는 애플리케이션이 캐시와 원본 저장소를 직접 관리하는 읽기 전략입니다. Redis를 먼저 조회하고, 데이터가 없을 때만 DB에서 읽어 캐시에 저장합니다.

```text
요청
 └─ Redis 조회
     ├─ Cache Hit  → 바로 반환
     └─ Cache Miss → DB 조회 → Redis 저장 → 반환
```

중요한 점은 **Redis를 사용한다는 사실만으로 Cache-Aside가 되는 것은 아니라는 것**입니다. 예를 들어 점수 집계 자체를 Redis Sorted Set에 저장한다면 Redis는 집계 저장소입니다. Cache-Aside는 MySQL 같은 원본 저장소가 별도로 있고, 캐시 미스 때 그 원본을 읽어 캐시를 채우는 패턴입니다.

## 어떤 데이터부터 캐시할 것인가

모든 조회 결과를 캐시하면 키 조합과 무효화 대상이 빠르게 늘어납니다. 먼저 다음 조건에 가까운 데이터부터 적용하는 것이 단순합니다.

- 읽기 요청이 많다.
- 같은 데이터가 반복 조회된다.
- 데이터 변경 빈도가 낮다.
- 일시적으로 이전 값이 보여도 치명적이지 않다.
- 캐시 효과를 측정하기 쉽다.

여행 서비스라면 관광지 상세 정보가 한 가지 후보가 될 수 있습니다. 이름, 주소, 이미지와 같은 정보는 사용자별 검색 결과보다 키가 단순하고 변경 빈도도 낮기 때문입니다.

```text
Key   : attraction:detail:{attractionId}
Value : 관광지 상세 응답 JSON
TTL   : 30분 + 0~5분 지터
```

30분은 정답이 아니라 시작점입니다. 데이터의 변경 주기와 허용 가능한 신선도를 기준으로 정하고, 운영 중에는 cache hit ratio, DB 조회량, API p95·p99 응답 시간을 보고 조정해야 합니다.

## TTL만으로는 부족한 이유

TTL이 만료될 때까지 기다리면 DB에서 수정된 데이터와 캐시가 다른 상태로 남을 수 있습니다. 데이터 변경 경로가 있다면 명시적인 무효화도 필요합니다.

```text
1. DB 데이터 변경 및 트랜잭션 커밋
2. attraction:detail:{id} 캐시 삭제
3. 다음 조회가 DB에서 최신 값을 읽어 캐시 재생성
```

캐시 값을 직접 갱신하는 대신 삭제를 우선하면 MySQL을 진실의 원천으로 유지할 수 있습니다. 다만 DB 커밋 후 캐시 삭제가 실패하면 TTL 동안 이전 값이 남을 수 있습니다.

초기에는 짧은 TTL과 실패 로그로 위험을 제한할 수 있습니다. 더 강한 정합성이 필요해진다면 캐시 삭제 재시도, 메시지 큐, 트랜잭셔널 아웃박스 같은 구조를 검토할 수 있습니다.

## Redis가 장애 나면 어떻게 할까

캐시는 성능을 위한 보조 저장소이므로 캐시 장애가 핵심 조회 실패로 이어지지 않게 해야 합니다.

```text
Redis 정상 → 캐시 조회, 미스 시 DB 조회
Redis 장애 → 짧은 시간 안에 포기하고 DB로 우회
```

여기서 단순히 `try-catch`만 추가해서는 충분하지 않습니다. Redis 연결과 명령의 타임아웃이 길면 DB로 우회하기 전에 요청이 오래 대기하기 때문입니다. 캐시 접근 타임아웃을 짧게 두고, 장애 중 매 요청이 Redis를 다시 호출하지 않도록 Redis 접근에도 별도의 차단 전략을 검토할 수 있습니다.

DB 우회가 가능하더라도 모든 요청이 한꺼번에 DB로 몰리면 새로운 장애가 생길 수 있습니다. 따라서 DB 커넥션 풀과 처리 가능한 트래픽을 기준으로 rate limit이나 부하 차단 정책도 함께 살펴봐야 합니다.

## Cache Stampede와 Penetration

### 인기 키가 동시에 만료되는 경우

많은 요청이 들어오는 키가 만료되면 여러 요청이 동시에 DB를 조회할 수 있습니다. 이를 Cache Stampede라고 합니다.

대응은 단순한 방법부터 단계적으로 적용할 수 있습니다.

1. TTL에 지터를 추가해 여러 키의 동시 만료를 분산합니다.
2. 문제가 확인된 인기 키에는 짧은 키별 락을 적용합니다.
3. 락을 얻은 요청만 DB를 조회하고 나머지는 캐시를 다시 확인합니다.
4. 락에는 반드시 만료 시간을 둡니다.

모든 키에 처음부터 분산 락을 적용하면 구현 복잡성과 장애 지점이 늘어납니다. 실제 병목을 확인한 후 도입하는 편이 적절합니다.

### 존재하지 않는 ID를 반복 조회하는 경우

존재하지 않는 데이터를 계속 요청하면 매번 캐시 미스와 DB 조회가 발생합니다. 이를 Cache Penetration이라고 합니다.

존재하지 않는 결과를 30~60초 정도만 캐시하는 negative caching으로 반복 조회를 줄일 수 있습니다. 다만 곧 생성될 수 있는 데이터라면 긴 TTL을 사용하지 않아야 합니다.

## Circuit Breaker란

외부 API가 실패할 때마다 계속 호출하면 응답 대기 중인 스레드와 커넥션이 쌓입니다. Retry까지 무제한으로 수행하면 이미 장애가 난 외부 시스템에 더 큰 부하를 줄 수 있습니다.

Circuit Breaker는 최근 호출의 실패율이나 느린 호출 비율이 기준을 넘으면 외부 호출을 잠시 차단합니다. 목적은 외부 시스템을 복구하는 것이 아니라 **외부 장애가 우리 서비스 전체로 전파되는 것을 막는 것**입니다.

## CLOSED, OPEN, HALF_OPEN

Circuit Breaker는 일반적으로 세 상태를 가집니다.

```text
CLOSED
  정상 호출 및 성공·실패 기록
    │ 실패율 임계치 초과
    ▼
OPEN
  외부 호출 없이 즉시 fallback
    │ 대기 시간 경과
    ▼
HALF_OPEN
  제한된 시험 호출만 허용
    ├─ 성공 → CLOSED
    └─ 실패 → OPEN
```

- **CLOSED:** 정상적으로 외부 API를 호출합니다.
- **OPEN:** 외부 API를 호출하지 않고 즉시 fallback을 실행합니다.
- **HALF_OPEN:** 일부 시험 호출만 보내 복구 여부를 확인합니다.

OPEN 상태에서 네트워크 요청 자체를 보내지 않기 때문에 빠르게 응답하고 서버 자원을 보호할 수 있습니다.

## 외부 API별로 분리해야 하는 이유

여행 서비스에서 Naver Directions와 Gemini를 사용한다고 가정하면 Circuit Breaker도 각각 분리해야 합니다.

```text
naverDirections Circuit Breaker
gemini Circuit Breaker
```

두 API는 장애 원인과 대체할 수 있는 기능이 다릅니다. 하나의 Circuit Breaker를 공유하면 Naver의 장애 때문에 정상인 Gemini 호출까지 차단될 수 있습니다.

초기 설정은 다음과 같이 시작할 수 있습니다.

```yaml
resilience4j:
  circuitbreaker:
    instances:
      naverDirections:
        slidingWindowType: COUNT_BASED
        slidingWindowSize: 20
        minimumNumberOfCalls: 10
        failureRateThreshold: 50
        slowCallDurationThreshold: 2s
        slowCallRateThreshold: 50
        waitDurationInOpenState: 30s
        permittedNumberOfCallsInHalfOpenState: 3
      gemini:
        slidingWindowType: COUNT_BASED
        slidingWindowSize: 20
        minimumNumberOfCalls: 10
        failureRateThreshold: 50
        waitDurationInOpenState: 30s
        permittedNumberOfCallsInHalfOpenState: 3
```

호출이 한두 번 실패했다고 바로 차단하지 않도록 최소 호출 수를 두고, 실패뿐 아니라 지나치게 느린 호출도 집계합니다. 이 값 역시 트래픽과 외부 API의 평소 성공률을 측정한 뒤 조정해야 합니다.

## 모든 예외가 장애는 아니다

Circuit Breaker는 외부 시스템의 건강 상태를 판단해야 합니다. 모든 예외를 실패로 기록하면 사용자 입력 오류 때문에 정상인 외부 API의 회로가 열릴 수 있습니다.

| 상황 | 처리 방향 |
|---|---|
| 연결 실패, timeout, HTTP 5xx | 외부 장애로 기록 |
| HTTP 429 | `Retry-After`와 호출 제한 정책 검토 |
| HTTP 401·403 | 재시도보다 설정 오류 알림 |
| 잘못된 사용자 입력 | 보통 실패율 계산에서 제외 |

어떤 예외를 기록하고 제외할지는 API의 오류 계약을 확인해 명시적으로 정해야 합니다.

## fallback은 가짜 성공이 아니다

fallback은 무조건 `200 OK`를 만드는 기능이 아닙니다. 정확성을 해치지 않는 범위에서 기능 수준을 낮추는 전략입니다.

### Directions가 실패한 경우

동선 최적화라면 실제 도로 소요시간 대신 좌표 거리를 이용한 nearest-neighbor와 2-opt로 근사 결과를 제공할 수 있습니다.

```text
정상 → 실제 도로 소요시간 기반 최적화
장애 → 좌표 거리 기반 근사 최적화
```

이 경우 응답에 `APPROXIMATE` 같은 상태를 포함해 근사 결과임을 알리는 것이 좋습니다.

반면 실제 도로 경로는 로컬 알고리즘으로 동일하게 만들 수 없습니다. 이전 성공 결과를 제한적으로 사용할 수 없다면 빈 경로를 정상 결과처럼 반환하는 대신 `503 Service Unavailable`과 재시도 안내를 제공해야 합니다.

### AI API가 실패한 경우

AI API 장애 때 임의의 답변을 만들어 성공처럼 반환하면 사용자는 잘못된 정보를 신뢰할 수 있습니다. 질문을 저장한 뒤 현재 추천 기능을 사용할 수 없다는 명확한 상태와 재시도 방법을 제공하는 편이 안전합니다.

## Timeout, Retry, Circuit Breaker의 차이

세 기능은 비슷해 보이지만 역할이 다릅니다.

- **Timeout:** 한 번의 호출을 언제 포기할지 결정합니다.
- **Retry:** 일시적 오류에 한해 제한적으로 다시 호출합니다.
- **Circuit Breaker:** 여러 요청의 실패 이력을 보고 호출 자체를 차단합니다.

개념적인 호출 순서는 다음과 같습니다.

```text
사용자 요청
  → Circuit Breaker
    → Retry
      → Timeout
        → 외부 API
  → 실패 시 기능별 fallback
```

Retry는 503이나 일시적인 네트워크 오류처럼 다시 성공할 가능성이 있는 경우에만 제한해야 합니다. 낮은 최대 횟수와 exponential backoff, jitter를 사용하고 인증 오류나 잘못된 요청은 재시도하지 않습니다.

사용자 요청 한 건을 세 번 재시도하면 실제 외부 호출은 최대 네 번이 될 수 있습니다. Retry가 Circuit Breaker 통계와 외부 API 부하를 함께 증가시킨다는 점도 고려해야 합니다.

## 구현보다 먼저 정할 관측 지표

패턴을 적용했다는 사실보다 실제 효과와 장애 상태를 확인할 수 있어야 합니다.

### Cache-Aside

- cache hit·miss 비율
- DB 상세 조회 QPS
- Redis 오류율과 지연 시간
- API p95·p99 응답 시간
- 캐시 무효화 실패 횟수

### Circuit Breaker

- CLOSED·OPEN·HALF_OPEN 상태 변화
- 외부 API 성공률과 오류 코드 분포
- timeout과 slow call 비율
- fallback 실행 횟수
- Retry 횟수와 최종 성공률

Spring Boot Actuator와 Micrometer를 이용해 지표를 수집하고, OPEN 전환이나 인증 설정 오류는 운영 알림으로 연결할 수 있습니다.

## 테스트할 시나리오

정상 동작만 확인해서는 장애 대응 구조를 검증할 수 없습니다.

### Cache-Aside

1. 첫 조회에서 DB를 읽고 Redis에 저장되는지 확인합니다.
2. 두 번째 조회에서는 DB가 호출되지 않는지 확인합니다.
3. 데이터 변경 후 캐시가 삭제되고 최신 값으로 채워지는지 확인합니다.
4. Redis를 중단해도 DB 조회로 정상 응답하는지 확인합니다.
5. 인기 키가 동시에 만료될 때 DB 부하가 허용 범위인지 측정합니다.

### Circuit Breaker

1. 외부 API 연속 실패로 CLOSED에서 OPEN으로 전환되는지 확인합니다.
2. OPEN 상태에서 실제 외부 호출 없이 fallback이 실행되는지 확인합니다.
3. 대기 시간 후 HALF_OPEN에서 제한된 호출만 나가는지 확인합니다.
4. 시험 호출 성공 시 CLOSED, 실패 시 OPEN으로 돌아가는지 확인합니다.
5. 사용자 입력 오류가 외부 장애로 집계되지 않는지 확인합니다.

## 적용 순서

한 번에 모든 기능을 추가하면 어떤 변경이 효과를 냈는지 판단하기 어렵습니다.

1. 캐시 적용 전 DB QPS와 응답 시간을 측정합니다.
2. 변경 빈도가 낮은 단일 상세 조회에 Cache-Aside를 적용합니다.
3. 외부 API의 연결·읽기 timeout을 먼저 설정합니다.
4. API별 Circuit Breaker와 fallback을 분리합니다.
5. 재시도 가능한 오류에만 제한적 Retry를 추가합니다.
6. 장애 주입과 부하 테스트 결과로 TTL과 임계치를 조정합니다.

## 정리

Cache-Aside와 Circuit Breaker는 각각 성능과 장애 격리를 다루지만 공통점이 있습니다. 정상 상황의 빠른 응답만 보는 것이 아니라, 의존 시스템이 실패했을 때 서비스가 어떤 수준으로 계속 동작할지를 결정하는 패턴이라는 점입니다.

- Redis를 사용한다고 모두 Cache-Aside는 아닙니다.
- Cache-Aside의 원본은 DB이며 캐시 장애 시 우회 경로가 필요합니다.
- TTL뿐 아니라 무효화와 동시 만료 문제도 고려해야 합니다.
- Retry는 일시적 실패를 다시 시도하고 Circuit Breaker는 반복 장애 때 호출을 차단합니다.
- fallback은 결과의 정확성을 해치지 않는 범위에서 기능 수준을 낮춰야 합니다.
- TTL과 실패율 임계치는 고정된 정답이 아니라 측정으로 조정할 시작값입니다.

두 패턴을 도입할 때는 라이브러리 애노테이션을 붙이는 것보다 **원본 데이터의 위치, 실패 시 우회 경로, 허용할 데이터 신선도, 기능별 fallback과 관측 지표를 먼저 정의하는 것**이 중요합니다.
