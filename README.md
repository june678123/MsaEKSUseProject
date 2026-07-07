# MSA 커머스 플랫폼 — 인프라 + 백엔드

Kotlin/Go 기반 마이크로서비스 아키텍처로 설계한 커머스 백엔드 시스템입니다.
서비스 개발부터 AWS 인프라 설계, Kubernetes 운영, 로그 파이프라인까지 구축되게 하였습니다.

[![Highlight](https://img.shields.io/badge/▶_하이라이트_영상-YouTube-red?style=for-the-badge&logo=youtube)](https://youtu.be/wotoO3KQBxQ?si=Y5XQOrLAwjbeYobA)
[![Video](https://img.shields.io/badge/▶_기능_시연_영상-YouTube-red?style=for-the-badge&logo=youtube)](https://youtu.be/t9DFECeHMd8?si=OPeJg1i5BChrNKEs)

---

## 기술 스택

| 영역 | 기술 |
|---|---|
| 언어 | Kotlin, Go |
| 프레임워크 | Spring Boot 4, Spring WebFlux, Spring Security, Spring Data JPA/R2DBC/MongoDB Reactive |
| 통신 | gRPC (Protobuf), REST, Kafka |
| DB | MySQL (RDS), MongoDB (DocumentDB), Redis (ElastiCache) |
| 인프라 | AWS EKS (2 Cluster), Terraform (IaC), Istio (Service Mesh), Docker |
| 시크릿 | AWS Secrets Manager + External Secrets Operator (IRSA 기반) |
| 모니터링 | Fluent Bit, Logstash, Elasticsearch, Kibana, Kafka 스택들 사용해 로그 파이프라인 구축 |
| 안정성 | Circuit Breaker, Retry, Graceful Shutdown, HPA, Pod Anti-Affinity |

---

## 아키텍처

```
                    ┌──────────────────────────────────────────────────────────┐
  Client ──HTTPS──▶ │                 Business Cluster (EKS)                   │
                    │                                                          │
                    │  BFF ─────────────────────────────────────────────────── │
                    │  ┌──────────┐    ┌──────────┐    ┌───────────────────┐   │
                    │  │ demoGate │    │adminGate │    │ product-grpc-gw   │   │
                    │  │  (BFF)   │    │  (BFF)   │    │ (Go, REST↔gRPC)  │    │
                    │  └──┬─┬──┬─┘    └────┬─────┘    └─────────┬─────────┘    │
                    │     │ │  │      gRPC │                gRPC │             │
                    │     │ │  │  ┌───────┘                     │              │
                    │     │ │  │  │                              │             │
                    │  서비스 ──┼──┼─────────────────────────────┼───────────  │
                    │     │ │  │  │                              │             │
                    │     │ │  └──┼──────────┐                  │              │
                    │     │ │gRPC │     gRPC │                  │              │
                    │  ┌──▼─▼────▼──┐  ┌────▼─────┐  ┌────────▼──────────┐     │
                    │  │  demoUser  │  │demoOrder │  │   demoProduct     │     │
                    │  └──────┬─────┘  └────┬─────┘  └────────┬──────────┘     │
                    │         │              │                  │              │
                    │  DB ────┼──────────────┼──────────────────┼────────────  │
                    │         │              │                  │              │
                    │  ┌──────▼─────┐  ┌────▼─────┐  ┌────────▼──────────┐     │
                    │  │ DocumentDB │  │   RDS    │  │       RDS         │     │
                    │  │ (MongoDB)  │  │ (MySQL)  │  │     (MySQL)       │     │
                    │  └────────────┘  └──────────┘  └────────────────────┘    │
                    │                                                          │
                    │  ┌──────────┐  ┌───────────────────────┐                 │
                    │  │  Redis   │  │     Kafka (KRaft)     │                 │
                    │  │(ElastiC) │  │   Controller+Broker   │                 │
                    │  └──────────┘  └────────────┬──────────┘                 │
                    └──────────────────────────────┼────────────────────────── ┘
                                                   │
                    ┌──────────────────────────────▼────────────────────────── ┐
                    │              Monitoring Cluster (EKS)                    │
                    │                                                          │
                    │  Fluent Bit, Logstash, Elasticsearch, Kibana, Kafka      │
                    │  스택들 사용                                             │
                    └────────────────────────────────────────────────────────  ┘

  demoGate  ──gRPC──▶ demoUser, demoProduct, demoOrder
  adminGate ──gRPC──▶ demoUser
```

- 비즈니스 클러스터와 모니터링 클러스터 분리 — 로그 수집이 서비스 성능에 영향을 주지 않음
- 서비스별 전용 DB — 서비스 간 DB 의존성 없이 독립 배포 가능
- 전체 인프라를 Terraform으로 코드화 

---

## 핵심 설계 판단

### 인증/보안

- **AT+RT+RTR 구조**: Refresh Token 사용 시 즉시 폐기 + 재발급하여 탈취 시 재사용 차단
- **탈취 감지**: 이미 폐기된 RT로 접근 시도 시 해당 계정의 전체 세션 강제 종료
- **RT 저장**: httpOnly 쿠키
- **Istio mTLS**: 서비스 간 통신 전구간 암호화
- **관리자 게이트웨이 분리**: 사용자/관리자 진입점을 분리하여 공격 표면 축소
- **세션 공유**: 두 게이트웨이가 동일 Redis를 사용하여 관리자 강제 로그아웃이 즉시 반영

### 데이터 정합성

- **동시성 제어**: 상황별로 적합한 락 전략을 선택하여 성능과 안정성 사이의 트레이드오프를 조정
- **Kafka 멱등성 보장**: 메시지 재전달 시 중복 처리 방지를 DB 레벨에서 보장
- **서비스별 DB 분리 + Kafka 이벤트 기반 통신**: 서비스 간 트랜잭션 경계 분리

### 성능 최적화

- **CQRS**: 특정 서비스에 조회 부하가 쓰기에 영향을 덜 줄수도 있게하는 구조 적용되게 함
- **BFF 병렬 호출**: WebFlux 기반 비동기 병렬 조회로 응답 시간 단축 
- **N+1 해소**: JOIN 기반 단일 쿼리로 변환 (100건 기준 쿼리 101회 → 1회)
- **인덱스 설계**: 조회 패턴 분석 후 복합 인덱스 컬럼 순서 최적화, 미사용 인덱스 제거
- **동적 검색**: 입력된 조건만 쿼리에 반영되는 동적 쿼리 구현
- **gRPC**: 내부 서비스 간 Protobuf 기반 통신, 대량 결과는 Server Streaming 적용
- **grpc-gateway (Go)**: 외부 REST 요청을 내부 gRPC로 변환하는 프록시를 별도 서비스로 분리

### 인프라/운영

- **Terraform IaC**: VPC, EKS 2클러스터, RDS, DocumentDB, ElastiCache, Kafka, ELK, IAM, IRSA 등 전체 인프라를 코드로 관리
- **dev/prod 환경 분기**: Terraform locals에서 환경별 스펙/설정 자동 분기 (DB 인스턴스, NAT HA, Kafka 보존 등)
- **시크릿 관리**: AWS Secrets Manager + ESO + IRSA — 서비스 코드에 시크릿 미포함, Pod별 최소 권한 IAM 역할
- **Resilience 외부화**: CB/Retry 설정을 Terraform → ConfigMap → 환경변수로 관리하여 이미지 재빌드 없이 변경 가능
- **로그 파이프라인**: Fluent Bit, Logstash, Elasticsearch, Kibana, Kafka 스택들 사용해 로그 파이프라인 구축
- **무중단 운영**: Graceful Shutdown, HPA 오토스케일링, Pod Anti-Affinity(AZ 분산 배치)
- **카나리 배포**: Istio를 활용한 트래픽 비율 기반 점진적 배포 + 롤백
- **AZ 장애 대응**: 노드 drain 시 다른 AZ에서 Pod 자동 복구 — 영상에서 검증

---

## 시연 영상 내용

**하이라이트 영상:**
- Kafka 이벤트 기반 주문 처리 + 데이터 정합성 검증
- AZ 장애 시뮬레이션 (노드 drain → 다른 AZ에서 무중단 복구)
- HPA 오토스케일링 (부하 인가 → Pod 자동 확장)
- Kibana 로그 모니터링

**기능 시연 영상:**
- 전체 사용자/구매 플로우 + 어드민 기능
- JWT 보안 (RTR 도용 감지, 권한 분리)
- TLS + Istio mTLS 확인
- Kafka 이벤트 정합성, Pod 자동 복구
- 카나리 배포 (트래픽 비율 조절, 롤백)
- gRPC 성능 + BFF 병렬 호출 + 동적 검색

---

## 요약

| 항목 | 내용 |
|---|---|
| 서비스 구성 | 6개 마이크로서비스 (BFF 2개 + 도메인 서비스 3개 + gRPC 프록시 1개) |
| 배포 환경 | AWS EKS 2클러스터 (비즈니스 + 모니터링) |
| 보안 | AT+RT+RTR + 탈취 감지 + httpOnly 쿠키 + IRSA 최소 권한 |
| 데이터 | 서비스별 전용 DB + 상황별 락 전략 + Kafka 멱등성 보장 |
| 성능 | CQRS + BFF 병렬 호출 + gRPC + 동적 쿼리 + 인덱스 최적화 |
| 안정성 | Graceful Shutdown + HPA + Pod Anti-Affinity(AZ 분산) + AZ 장애 대응 검증 |
| 인프라 | Terraform IaC (전체 인프라) + dev/prod 환경 분기 + Resilience 설정 외부화 |
| 모니터링 | Fluent Bit, Logstash, Elasticsearch, Kibana, Kafka 스택들 사용해 로그 파이프라인 구축 |
