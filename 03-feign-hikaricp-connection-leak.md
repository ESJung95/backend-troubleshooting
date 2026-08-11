# 외부 MSA(Feign) 연동 시 DB 커넥션 풀 고갈 및 트랜잭션 전파 예외 해결

## 1. 개요

- **목적:** 통계 데이터 제출 완료 시, 이상치 탐지 분석 외부 API를 OpenFeign Client로 호출하여 비동기 시계열 분석 진행 중인 `job_id`를 발급받는 연동 기능 구현.

- **문제 흐름:** 커스텀 400 에러 ➔ 500 Read Timeout 및 Hikari Connection Leak ➔ Response DTO 직렬화 보완 ➔ TransactionTemplate 기반 트랜잭션 분리 ➔ 로그 저장 전파 예외 수정 ➔ 200 (성공)

## 2. 문제 발생 및 단계별 해결

### 1단계: 최초 연동 시 400 Bad Request

- **현상:** Feign 호출 시 상대방 서비스로부터 400 Bad Request 커스텀 에러 응답 수신.

```
feign.FeignException$BadRequest: [400 Bad Request] during [POST]
to [http://<internal-gateway>/statverify-service/api/v1/imr/outlier/detect]
[StatVerifyServiceClient#detectImrOutliers(String,ImrDetectRequestDto)]:
[{"timestamp":"2026-07-31T...","status":400,"error":"Bad Request","message":"..."}]
```

- **원인:** 상대방(분석 엔진) 측 환경에서 외부 연결 타임아웃 및 처리 서버 설정 문제 발생.
    - 상대방 게이트웨이 및 서버 측 처리 과정에서 타임아웃 설정 오류로 인해 400 Bad Request를 커스텀 에러 바디로 반환.
    - 반환된 JSON 메시지(검증서버 미연결, code: B001)는 게이트웨이나 웹서버(Nginx 등)가 만든 것이 아니라, 상대방 애플리케이션 내부 비즈니스 로직(Controller/GlobalExceptionHandler)이 직접 생성한 커스텀 에러 Response임을 확인.
- **해결:** 상대 개발자 측에서 해당 타임아웃 환경 설정 수정 후 재배포를 진행하여 400 에러 해결.

### 2단계: 500 Read Timeout 및 HikariCP Connection Leak 경고

- **현상:** 400 에러 해결 후, 곧바로 500 에러 및 HikariCP 커넥션 누수 경고 발생.

```
WARN  com.zaxxer.hikari.pool.ProxyLeakTask - Connection leak detection triggered for ... PgConnection ...
ERROR ...SupplyRateService - [500 Internal Server Error] during [POST] to [http://.../imr/outlier/detect]
```

- **원인:**
    - 메서드 상단에 `@Transactional`이 걸려 있어 메서드 진입 직후 PostgreSQL DB 커넥션을 획득함.
    - DB 커넥션을 쥔 채로 외부 Feign HTTP 통신을 수행하면서 DB 커넥션 반납이 지연됨.
    - HikariCP 커넥션 풀이 바닥나고, 대기 시간이 Read Timeout(15초)을 초과하면서 500 에러 발생.

### 3단계: Response DTO snake_case 적용 (근본 해결 실패)

- **현상:** Feign 수신 객체 필드들이 camelCase로만 되어 있어, Request DTO처럼 Response DTO에도 `@JsonProperty` 기반 스네이크 표기법 적용.
- **결과:** Response DTO 수정만으로는 DB 커넥션을 계속 쥐고 있는 근본 원인이 해결되지 않아 500 에러가 지속됨. ➔ **외부 I/O 통신과 DB 트랜잭션을 분리해야 함을 확인.**

### 4단계: TransactionTemplate 도입을 통한 DB 커넥션 분리 (500 에러 해결)

- **해결:**
    - Service 클래스/메서드 레벨의 `@Transactional` 어노테이션 제거.
    - `TransactionTemplate`을 사용하여 DB 저장, 검증, 상태 업데이트 영역만 트랜잭션으로 감싸 실행 후 즉시 커밋 및 DB 커넥션을 풀에 반납.
    - Feign HTTP 통신 영역은 DB 커넥션을 완전히 내려놓은 상태에서 실행하여 Hikari Leak 및 500 Read Timeout 에러 해결.

### 5단계: IllegalTransactionStateException (MANDATORY 전파 속성 예외)

- **현상:**

```
org.springframework.transaction.IllegalTransactionStateException:
No existing transaction found for transaction marked with propagation 'mandatory'
at ...service.log.StatsLogService.saveLog(...)
```

- **원인:** `@Transactional`을 제거하면서 트랜잭션이 없는 상태가 되었는데, 맨 마지막에 호출되는 로그 저장 내부 어노테이션이 `@Transactional(propagation = Propagation.MANDATORY)`(기존 트랜잭션 필수)로 설정되어 있어 예외 발생.
- **해결:** 로그 저장 호출부를 `transactionTemplate.executeWithoutResult(...)`로 감싸 로그 저장용 단기 트랜잭션을 새로 제공.

## 3. 결과 & 배운점

- **응답 결과:** `200 OK` 반환 및 분석 `Job ID` 정상 수신.
- **개선 효과:** 외부 연동 과정에서 DB 커넥션 풀을 잡아두지 않으므로 HikariCP 누수 경고 및 타임아웃 500 에러 차단.
- **배운점:**
    - 외부 네트워크 통신은 DB 커넥션을 잡지 않은 상태에서 실행하는 것이 Spring 백엔드 개발의 표준 원칙이다.
    - 메서드 전체에 `@Transactional`을 거는 대신, `TransactionTemplate`으로 DB 작업이 필요한 영역만 묶어서 트랜잭션 범위를 제어할 수 있다. (외부 통신 시에는 해야만 한다)
    - MSA 환경에서 외부 연동 실패 시 데이터가 자동으로 복구되지 않는다. 로컬 트랜잭션 범위를 넘는 정합성은 수동 보상(롤백) 설계가 필요하다.
    - Feign Response DTO는 `@JsonProperty("snake_case")`를 명시적으로 선언해야 응답 필드가 정상 매핑된다.