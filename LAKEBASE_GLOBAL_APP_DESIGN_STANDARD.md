# Lakebase 기반 글로벌 앱 설계 표준 (v1.0)

## 1) 목적과 적용 범위
- 본 문서는 **Lakebase(데이터 레이크 + 데이터베이스 통합 아키텍처)** 를 기반으로, 다국가/다언어 서비스에서 일관된 품질을 확보하기 위한 설계 표준을 정의한다.
- 적용 대상: 웹, 모바일, 백오피스, 파트너 API, 데이터 파이프라인, 분석/AI 워크로드.
- 우선순위: **보안/법규 준수 > 가용성 > 성능 > 기능 확장성**.

---

## 2) 핵심 설계 원칙
1. **Region-first 설계**: 사용자 데이터의 저장/처리는 지역(리전) 단위 정책을 우선 적용한다.
2. **Local experience, global consistency**: 로컬 UX는 현지화하되, 핵심 도메인 규칙과 데이터 계약은 글로벌 표준을 유지한다.
3. **Schema contract 우선**: API/이벤트/데이터셋은 스키마 버전 관리와 하위호환을 기본으로 한다.
4. **Zero-trust 보안**: 네트워크 경계 대신 워크로드/아이덴티티 단위 접근 통제.
5. **Observability by default**: 로그/메트릭/트레이스/감사를 기능 정의 단계에서 포함.

---

## 3) 참조 아키텍처 (Lakebase)

### 3.1 계층 구조
- **Experience Layer**: Web/App/BFF(GraphQL 또는 REST Gateway)
- **Domain Service Layer**: 인증, 결제, 주문, 콘텐츠 등 도메인 마이크로서비스
- **Data Product Layer**: 도메인별 정제 데이터셋, 피처 스토어, 리포팅 뷰
- **Lakebase Layer**
  - Lake Zone: Raw / Bronze / Silver / Gold
  - Base DB Zone: OLTP(트랜잭션) + OLAP(분석) + Search
- **Governance Layer**: 카탈로그, 계보(Lineage), 품질 규칙, 접근 정책, 감사 로그

### 3.2 데이터 흐름 표준
- 온라인 트랜잭션 데이터는 OLTP 저장 후 이벤트로 비동기 발행.
- 이벤트는 Lake Raw에 적재되고, 표준 변환 파이프라인으로 Silver/Gold 승격.
- 분석/AI/리포팅은 Gold 또는 인증된 Data Product만 사용.
- 운영 API는 Gold 직접 조회를 지양하고, 목적별 Serving DB/Cache를 둔다.

---

## 4) 글로벌/멀티리전 설계

### 4.1 리전 전략
- 기본 원칙: **Data residency(데이터 거주성)** 요구가 있으면 해당 국가/권역 내 저장.
- 권장 토폴로지
  - `Global Control Plane`: 계정, 설정, 배포 메타데이터
  - `Regional Data Plane`: PII/트랜잭션/로그 원본
- 리전 장애 시
  - Tier-1 서비스: RTO 30분 이하, RPO 5분 이하 목표
  - Tier-2 서비스: RTO 4시간 이하, RPO 1시간 이하 목표

### 4.2 데이터 분류 및 위치 정책
- **P0 (규제 민감/PII/결제)**: 리전 고정 저장, 크로스리전 복제 금지(예외 승인 필수)
- **P1 (업무 핵심)**: 익명화 후 제한적 복제 가능
- **P2 (일반 운영)**: 글로벌 복제 가능
- **P3 (공개/비민감)**: CDN/글로벌 캐시 활용

---

## 5) 데이터 모델링/스키마 표준

### 5.1 공통 컬럼 규약
모든 테이블/이벤트에 아래 메타 컬럼을 권장:
- `id` (ULID/UUID)
- `tenant_id`
- `region_code` (ISO 3166-1 alpha-2)
- `locale` (BCP 47, 예: `en-US`, `ko-KR`)
- `event_time` (UTC)
- `ingested_at` (UTC)
- `schema_version`
- `is_deleted` (soft delete)

### 5.2 시간/통화/단위 표준
- 저장 시각: UTC, 화면 표시만 로컬 타임존 변환.
- 통화 금액: `amount_minor`(정수) + `currency`(ISO 4217) 조합.
- 길이/무게 등 단위는 SI 기준 저장, UI에서 변환.

### 5.3 버전 관리
- 스키마 변경은 **Backward-compatible** 을 기본 원칙으로 한다.
- 필드 제거는 2개 릴리스 이상 deprecation 기간 후 수행.
- 파이프라인/소비자 영향도는 계보(Lineage) 기반으로 사전 검증.

---

## 6) API/BFF 설계 표준
- 외부 공개 API는 버전 prefix(`/v1`)를 명시.
- 멱등성 필요한 쓰기 API는 `Idempotency-Key`를 지원.
- 페이지네이션은 cursor 기반 기본.
- 오류 모델 표준화: `code`, `message`, `details`, `trace_id`.
- 지역화 가능한 메시지는 서버 코드와 번역 키를 분리.

---

## 7) 현지화(i18n/l10n) 표준
- 기본 언어 fallback 체인: `user locale -> region default -> en-US`.
- 번역 리소스는 키 기반 관리(문장 하드코딩 금지).
- 날짜/숫자/통화 표기는 CLDR/ICU 규칙 사용.
- 문화권별 민감 요소(이름 표기, 주소 포맷, 색상/아이콘 의미) 체크리스트 운영.

---

## 8) 보안/컴플라이언스 표준
- 인증: OIDC/OAuth2.1, 서비스 간 통신은 mTLS 권장.
- 권한: RBAC + ABAC 혼합 모델, 최소권한 원칙.
- 암호화
  - At-rest: KMS 기반 키 관리
  - In-transit: TLS 1.2+
- 비밀정보는 시크릿 매니저에서 관리(코드/CI 로그 노출 금지).
- 감사 로그는 위·변조 방지 저장소에 1년 이상 보관(국가 규정 우선).

---

## 9) 데이터 품질/거버넌스 표준
- 필수 품질 지표: 완전성, 유효성, 유일성, 최신성, 일관성.
- 도메인별 Data Owner 지정, SLA/SLO 명문화.
- Gold 승격 조건
  - 품질 규칙 통과율 99%+
  - 스키마 검증 통과
  - PII 마스킹/토큰화 완료
- 메타데이터 카탈로그에 데이터 정의/계보/사용 예시 등록 필수.

---

## 10) 성능/비용 최적화 표준
- 저장소 티어링: 핫/웜/콜드 분리, 수명주기 정책 자동화.
- 쿼리 최적화: 파티셔닝(`event_date`, `region_code`)과 클러스터링 기준 정의.
- 캐시 정책: TTL + 무효화 이벤트 기반 하이브리드.
- FinOps KPI: TB당 저장비, 쿼리당 비용, DAU당 인프라 비용을 월별 추적.

---

## 11) 신뢰성/운영 표준
- SLO 예시
  - 사용자 API 가용성 99.9%
  - P95 응답시간 300ms 이하(핵심 읽기 API)
- 배포 전략: 블루/그린 또는 카나리, 자동 롤백 기준 명시.
- 런북 필수 항목: 장애 시나리오, 커뮤니케이션 채널, 책임자(on-call), 복구 절차.
- 대규모 이벤트(세일/런칭) 전 부하 테스트와 게임데이 시행.

---

## 12) 분석/AI 활용 표준
- Feature/Training/Serving 데이터셋 분리.
- 학습 데이터는 시점 정합성(time-travel) 보장.
- 모델 성능 외 공정성/편향 모니터링 지표 포함.
- AI 결과 저장 시 설명가능성 메타데이터(`model_version`, `feature_set`, `decision_reason`) 기록.

---

## 13) 개발 프로세스/품질 게이트
- Definition of Done
  - 보안 점검 통과(SAST/DAST/Dependency)
  - 데이터 계약 테스트 통과
  - 관측성(로그/메트릭/트레이스) 계측 완료
  - 문서(ADR/API/스키마) 업데이트
- PR 체크리스트
  - 리전 정책 영향 검토
  - 개인/민감정보 처리 검토
  - 역호환성 검토

---


## 14) 역할 분리(Responsibility Segregation) 표준

### 14.1 기본 원칙
- **설계(Design) / 구현(Build) / 승인(Approve) / 운영(Operate) / 감사(Audit)** 는 동일인 독점 금지.
- 프로덕션 반영(배포/권한 변경/정책 예외)은 최소 2인 승인(4-eyes principle) 적용.
- 데이터 소유권(Data Ownership)과 플랫폼 운영권한(Platform Admin)을 분리.

### 14.2 권장 역할 정의
- **Product Owner(PO)**: 비즈니스 우선순위, 릴리스 범위, KPI 책임.
- **Solution Architect(SA)**: 아키텍처 적합성, 비기능 요구사항, ADR 승인.
- **Domain Engineer(DE)**: 도메인 서비스/API/이벤트 구현 및 테스트.
- **Data Engineer(DaE)**: Lakebase 파이프라인, 데이터 품질 규칙, 스키마 진화.
- **SRE/Platform Engineer(SRE)**: 배포 자동화, 관측성, SLO/에러버짓 운영.
- **Security/Compliance Officer(Sec/Comp)**: 보안 통제, 규제 준수, 예외 승인.
- **Data Steward(DS)**: 데이터 정의, 카탈로그/계보, 품질 지표 관리.
- **QA/Release Manager(QA/RM)**: 품질 게이트, 테스트 커버리지, 배포 승인 절차.

### 14.3 RACI 매트릭스 (핵심 활동)
| 활동 | PO | SA | DE | DaE | SRE | Sec/Comp | DS | QA/RM |
|---|---|---|---|---|---|---|---|---|
| 글로벌 아키텍처/ADR 승인 | C | A/R | C | C | C | C | I | I |
| API 계약/버전 정책 수립 | C | A | R | C | C | C | I | C |
| 데이터셋 스키마 변경 | I | C | C | A/R | I | C | R | C |
| PII 분류/마스킹 정책 | I | C | C | R | I | A | R | I |
| 배포 승인(프로덕션) | I | C | R | C | A/R | C | I | R |
| 접근권한/키 정책 변경 | I | C | I | I | R | A | I | C |
| 장애 대응 및 사후분석 | I | C | R | C | A/R | C | I | C |
| 데이터 품질 SLA/SLO 관리 | C | C | I | R | C | I | A/R | C |

> R=Responsible, A=Accountable, C=Consulted, I=Informed

### 14.4 승인 게이트(필수)
- **Schema 변경**: DaE + DS 승인, 하위호환성 체크 리포트 첨부.
- **보안정책/권한 변경**: Sec/Comp 승인, 만료일 있는 예외 토큰만 허용.
- **프로덕션 배포**: QA/RM + SRE 이중 승인.
- **리전 간 데이터 이동**: Sec/Comp + DS 승인, 법무 검토 기록 필수.

---

## 15) 표준 산출물 템플릿
프로젝트 시작 시 다음 문서를 필수 작성:
1. **Global Architecture Decision Record (ADR)**
2. **Regional Data Residency Matrix**
3. **API Contract & Versioning Plan**
4. **Data Product Spec (owner/SLA/schema/quality rule)**
5. **Security & Compliance Control Matrix**
6. **SLO/Error Budget 운영안**

---

## 16) 단계별 도입 로드맵
- **Phase 1 (0~2개월)**: 핵심 도메인 스키마 표준화, 공통 메타컬럼 도입, 기본 관측성 구축
- **Phase 2 (2~4개월)**: 멀티리전 데이터 정책 적용, 품질 게이트 자동화
- **Phase 3 (4~6개월)**: Data Product 운영체계 정착, FinOps/SLO 고도화
- **Phase 4 (6개월+)**: AI/고급 분석 워크로드 표준 통합

---

## 17) 최소 준수 체크리스트 (요약)
- [ ] 리전/거주성 정책 문서화 및 승인
- [ ] 스키마 버전/역호환 전략 확정
- [ ] PII 분류·암호화·마스킹 적용
- [ ] 로그/메트릭/트레이스/감사 로그 계측
- [ ] 품질 규칙 + Gold 승격 기준 자동화
- [ ] SLO/런북/DR 훈련 계획 수립

> 본 표준은 조직의 법무/보안 정책 및 국가별 규제를 우선하며, 분기별 1회 이상 업데이트를 권장한다.
