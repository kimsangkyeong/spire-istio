내가 질문하는 것들을 reademe.txt파일에 작업자를 식별할 수 있도록 "--------------" 구분자를 해서 추가해줘.
지금 질문은 첫 질문이기 때문에 그냥 유지해줘
너는 OCP 전문가이며, service mesh 전문가 역할을 수행해줘. 내가 ocp 4.20.23 version을 구성해 놓았어.
나는 Nginx Ingress Controller Plus로 F5와 CIS로 연계하여 IC를 구성했어. 그래서 service mesh에서는 Gateway는 사용하지 않고,
sidecar mode만 사용할 꺼야. 그래서 NGINX IC Plus의 IC의 virtualserver 로 단일 서비스로 전달하고,
service mesh 에서는 Application에서 Virtual Service로 라우팅하도록 하고 싶어 처리 방법을 작성해줘.

이제 SPIFFE/SPIRE and ServiceMesh 연동을 하는 환경을 만들려고 해. 내가 OPERATOR들은 설치를 해 놓았는데, 어떻게 CR등 환경을 구성해야 하는지? 모르겠어.
각 영역을 설치하는 절차와 연동 아키텍처 구성도를 그리고, 상세 설치 방법과 설명, yaml 파일 등 세부 내용 정리해줘.
그리고 OCP의 엔진 Project들이 많이 있는데, zero trust 환경을 구성해야 한다고 하면, 모든 namespace를 istio injection 해야 하는지? 등도 포함해서 통합해서
전체 구성에 대해서 작업 절차, 순서를 고려해서 정리해줘.
아키텍처와 절차 설명은 md 파일로 만들어 주고, 리소스 들은 yaml 파일로 만들어서 따라하기 쉽게 정리 부탁해
--------------
[작업자: Kiro AI] [날짜: 2026-08-18]
[질문] SPIRE 제거 후 NGINX IC Plus + Service Mesh만 유지하는 전환 절차 작성 요청
[결과] ocp-spire-removal-guide/ 폴더에 6단계 무중단 마이그레이션 절차 작성 완료
       - Phase 1: PERMISSIVE 전환
       - Phase 2: Istio CR SPIRE 연동 제거
       - Phase 3: App Pod annotation 수정
       - Phase 4: STRICT mTLS 복원
       - Phase 5: AuthorizationPolicy Trust Domain 수정
       - Phase 6: SPIRE 리소스 완전 제거
--------------
--------------
[작업자: Kiro AI] [날짜: 2026-08-20]
[질문] OCP SPIFFE/SPIRE + Service Mesh 구성 시 cert-manager도 필수로 연계해야 하는가?

[답변]

cert-manager는 필수가 아닙니다. 선택 사항(Optional)입니다.

[근거 - Red Hat 공식 문서 (OSSM 3.4)]

Red Hat OpenShift Service Mesh 3.4 Installing 문서 Chapter 7에 명시:
"You can deploy SPIRE as a standalone solution or integrate it with cert-manager.
 When used together, cert-manager acts as the root Certificate Authority (CA),
 automatically issuing intermediate signing certificates to SPIRE, while SPIRE
 handles workload identity for mesh communication."

출처: https://docs.redhat.com/en/documentation/red_hat_openshift_service_mesh/3.4/html/installing/ossm-spire_ossm-cert-manager

[두 가지 구성 방식 비교]

┌─────────────────────────────────────────────────────────────────────────┐
│ 방식 1: SPIRE Standalone (cert-manager 없이)                             │
├─────────────────────────────────────────────────────────────────────────┤
│ - SPIRE Server가 자체적으로 Self-Signed Root CA를 생성/관리              │
│ - SPIRE가 직접 워크로드에 X.509 SVID(인증서) 발급                        │
│ - 추가 Operator 설치 불필요                                              │
│ - 구성이 단순                                                            │
│ - UpstreamAuthority 플러그인: disk (기본 자체서명) 또는 외부 CA 연동 가능 │
│                                                                         │
│ 적합한 경우:                                                             │
│ - 빠른 PoC/검증 환경                                                     │
│ - 자체서명 CA로 충분한 경우                                               │
│ - 외부 PKI 인프라가 없는 경우                                             │
│ - cert-manager 운영 부담을 줄이고 싶은 경우                               │
└─────────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────────┐
│ 방식 2: SPIRE + cert-manager 연동                                        │
├─────────────────────────────────────────────────────────────────────────┤
│ - cert-manager가 Root CA 역할 수행                                       │
│ - cert-manager가 SPIRE에게 Intermediate Signing Certificate 자동 발급   │
│ - SPIRE는 해당 Intermediate CA로 워크로드 SVID 발급                      │
│ - 인증서 갱신/교체가 cert-manager에 의해 자동화                           │
│ - 외부 CA(Vault, AWS PCA 등)와 cert-manager Issuer로 연동 가능          │
│                                                                         │
│ 적합한 경우:                                                             │
│ - 기업 PKI 인프라(Vault, AWS PCA, EJBCA 등)와 연동 필요                  │
│ - Root CA를 중앙에서 관리해야 하는 보안 정책                              │
│ - 인증서 수명주기 자동화가 중요한 프로덕션 환경                            │
│ - 감사(Audit) 요구사항이 있는 경우                                        │
└─────────────────────────────────────────────────────────────────────────┘

[현재 구성(00-architecture-guide.md)과의 관계]

현재 작성된 구성 가이드는 "방식 1: SPIRE Standalone"으로 구성되어 있습니다.
- SpireServer CR에서 자체 CA를 생성
- cert-manager Operator 설치 없이 동작
- SPIRE가 독립적으로 SVID 발급 및 갱신 처리

따라서 cert-manager 설치 없이 현재 구성대로 진행해도 문제없습니다.

[SPIRE의 자체 CA 동작 방식]

SPIRE Standalone 모드에서의 인증서 체인:
  SPIRE Server (Self-Signed Root CA)
      │
      ├── Intermediate CA (SPIRE Server가 자동 생성)
      │       │
      │       ├── Workload SVID (Pod A)
      │       ├── Workload SVID (Pod B)
      │       └── Workload SVID (Pod C)
      │
      └── OIDC Discovery Provider (JWT 검증)

- SPIRE Server는 ca_ttl 설정에 따라 CA 인증서 수명을 관리 (기본 24시간)
- 워크로드 SVID는 짧은 수명(기본 1시간)으로 자동 갱신
- Root CA 교체도 SPIRE가 자동으로 처리 (graceful rotation)

[향후 cert-manager 연동이 필요한 시점]

다음 상황이 발생하면 cert-manager 연동을 검토:
1. 기업 PKI 정책상 외부 Root CA를 반드시 사용해야 하는 경우
2. HashiCorp Vault, AWS Private CA 등 외부 CA를 사용해야 하는 경우
3. 멀티 클러스터 Federation에서 공통 Root CA를 공유해야 하는 경우
4. 보안 감사에서 Self-Signed CA가 허용되지 않는 경우

연동 시 추가 필요 구성요소:
- cert-manager Operator for Red Hat OpenShift (v1.15+)
- cert-manager Issuer/ClusterIssuer CR
- SPIRE Server의 UpstreamAuthority를 cert-manager 방식으로 변경

[요약]

cert-manager 필수 여부: 아니오 (Optional)
현재 구성(Standalone): 문제 없음, cert-manager 없이 동작
프로덕션 권장: 보안 정책에 따라 판단
- 자체서명 CA 허용 → Standalone으로 충분
- 외부 CA/PKI 정책 필수 → cert-manager 연동 필요

참고 문서:
- https://docs.redhat.com/en/documentation/red_hat_openshift_service_mesh/3.4/html/installing/ossm-spire_ossm-cert-manager
- https://github.com/spiffe/spire/blob/main/doc/plugin_server_upstreamauthority_disk.md
--------------
[작업자: Kiro AI] [날짜: 2026-08-20]
[질문] Red Hat 기술 미팅을 위한 SPIRE + Service Mesh 구성/운영 관련 질문 목록 작성 요청
[결과] redhat-meeting-questions.txt 파일로 별도 작성 완료
       - 7개 카테고리, 20개 질문
       - TP/GA 지원범위, 아키텍처, 장애 시나리오, Day-2 운영,
         멀티클러스터 확장, 롤백, 레퍼런스 사례
--------------
[작업자: Kiro AI] [날짜: 2026-08-20]
[질문] OSSM이 무엇인가?

[답변]

OSSM = OpenShift Service Mesh

Red Hat이 제공하는 Istio 기반 서비스 메시 제품으로,
OpenShift Container Platform(OCP) 위에서 운영되는 상용 배포판입니다.

정식 명칭: Red Hat OpenShift Service Mesh
약어: OSSM (OpenShift Service Mesh)
현재 버전: 3.4 (Istio Sail Operator 기반)

핵심 구성요소:
- Istio (데이터 플레인 + 컨트롤 플레인)
- Envoy Proxy (사이드카)
- Kiali (시각화/대시보드)
- Jaeger/Tempo (분산 트레이싱)

OSSM 2.x → 3.x 주요 변경:
- 2.x: Maistra Operator 기반 (Red Hat 자체 포크)
- 3.x: Istio Sail Operator 기반 (업스트림 Istio에 더 가까움)
