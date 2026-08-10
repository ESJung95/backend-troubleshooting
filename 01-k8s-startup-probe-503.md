# 쿠버네티스 환경 503 Service Unavailable 원인 분석 및 해결

## 1. 개요

- **발생 일시:** 2026-07-30
- **영향 범위:** 사내 CMP 기반 K8s 클러스터 내 애플리케이션 서비스
- **증상:**
    1. 외부 호출 시 `503 Service Unavailable` 에러 지속 발생.
    2. 소스 코드 변경 사항 없음.
    3. 애플리케이션 내부 로그(STDOUT/Log 파일)가 전혀 출력되지 않음.
    4. 파드 내부에서 `8080` 포트로 `curl` 요청 시 `Connection refused` 발생. → 이벤트 탭에서 확인했음

## 2. 원인 추적 과정

이슈 해결을 위해 아래의 순서로 단계별 원인을 찾아가 봄

### ① 1단계: 인프라 및 컨테이너 자원 확인

- **점검:** 파드 터미널 접속 후 디스크 용량(`df -h`) 및 프로세스(`ps aux`) 점검.
- **결과:** 디스크 용량 여유 있음. Java 프로세스(`java -Dspring.profiles.active=production...`)는 정상 실행(PID 1) 중.

### ② 2단계: 애플리케이션 포트 바인딩 점검

- **점검:** 파드 내부에서 `curl -v http://localhost:8080/health` 실행.
- **결과:** **`Connection refused` 발생.**
- **분석:** Java 프로세스는 살아있으나 내장 웹 서버(Tomcat)가 8080 포트를 Open하지 못하고 부팅 단계에 갇혀 있음.

### ③ 3단계: 외부 연동(DB) 통신 상태 검증

- **점검:** DB 포트로 TCP 소켓 통신 테스트 (`curl -v http://<DB_IP>:<DB_PORT>`).
- **결과:** 정상 파드 및 문제 파드 모두 `Empty reply from server` 반환 (네트워크 및 DB 접속 이상 없음).

### ④ 4단계: K8s 이벤트(Events) 및 헬스체크 분석 (최종 원인 발견)

- **점검:** CMP 콘솔 ➔ 파드 상세 ➔ **[Events] 탭** 조회.
- **발견된 경고 메시지:**

    > `Warning` | `Unhealthy` | **`Startup probe failed: get "http://...:8080/..." : dial tcp ...:8080: connect: connection refused`**

## 3. 문제 발생 원인

**`Startup Probe` 타임아웃으로 인한 K8s의 트래픽 차단 및 파드 재시작**

1. **상황:** 애플리케이션 부팅 시 DB 커넥션 풀(HikariCP) 생성 및 프레임워크 초기화 과정으로 인해 **8080 포트가 열리기까지 시간이 소요**됨.
2. **문제점:** K8s 매니페스트 설정상 `startupProbe`의 `failureThreshold` 값이 `3`으로 지나치게 작게 설정되어 있음.
3. **결과:** 체크 주기가 10초라 가정할 때 **단 30초** 만에 K8s가 부팅 실패로 판정함. 포트가 열리기도 전에 K8s가 트래픽을 차단(Endpoint 제외)하여 외부 요청에 **503**을 뱉었으며, 애플리케이션이 완전히 부팅되지 못해 에러 로그조차 남기지 못함.

## 4. 해결 방법

배포 매니페스트(`deployment.yaml`)를 수정하여 CI/CD 파이프라인으로 재배포를 진행

### ① Deployment YAML 파일 수정

`startupProbe` 항목의 `failureThreshold` 수치를 늘려 애플리케이션이 완전히 부팅될 때까지 충분한 대기 시간(예: 5분 이상)을 확보하도록 수정.

```yaml
spec:
  template:
    spec:
      containers:
      - name: app-container
        # ... 기존 설정 ...
        startupProbe:
          httpGet:
            path: /actuator/health  # 또는 설정된 헬스체크 경로
            port: 8080
          initialDelaySeconds: 20
          periodSeconds: 10
          failureThreshold: 30     # 기존 3 -> 30으로 수정 (10s * 30회 = 최대 300초/5분 대기)
```

### ② Git Commit & CI/CD 파이프라인 실행

## 5. 결과

1. **Startup Probe 통과:** 파드 재구동 시 K8s가 최대 5분간 기다려주면서 `Startup probe failed` 경고가 사라짐.
2. **포트 정상 개방:** Spring Boot 부팅 완료 후 8080 포트 Listen 확인 (`curl -v http://localhost:8080` 통과).
3. **로그 정상 출력:** 부팅 완료 및 API 요청 로그가 정상적으로 출력 됨.
4. **서비스 정상화:** 외부 API 호출 시 503 에러가 해결되고 `200 OK` 응답 확인.

## 6. 배운점

- **배포 매니페스트 튜닝:** Spring Boot와 같이 초기화 및 DB 연동 시간이 필요한 Heavy 애플리케이션은 `startupProbe` 대기 시간을 넉넉하게 설정해야 함
- **트러블슈팅 가이드:** 소스 변경 없이 갑자기 503이 뜨고 로그가 찍히지 않는다면, 앱 문제 이전에 K8s Probe(Startup/Readiness) 이벤트 실패로 인한 Endpoint 탈락 가능성을 1순위로 확인해보자
