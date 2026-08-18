# OCP 4.20 SPIFFE/SPIRE + Service Mesh (Sidecar Mode) + NGINX IC Plus 연동 구성 가이드

## 1. 전체 아키텍처 개요

### 1.1 구성 환경 요약

| 항목 | 내용 |
|------|------|
| OCP 버전 | 4.20.23 |
| Ingress Controller | NGINX IC Plus + F5 CIS 연계 |
| Service Mesh 모드 | Sidecar Only (Gateway 미사용) |
| ID 프레임워크 | SPIFFE/SPIRE (Zero Trust Workload Identity Manager) |
| 목표 | Zero Trust mTLS 환경 + SPIRE 기반 Workload Identity |

### 1.2 아키텍처 다이어그램

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                         OCP 4.20.23 Cluster                                      │
│                                                                                  │
│  ┌──────────────────────────────────────────────────────────────────────────┐   │
│  │  External Traffic Flow (North-South)                                      │   │
│  │                                                                           │   │
│  │   Client → F5 BIG-IP → NGINX IC Plus (VirtualServer CR)                  │   │
│  │                              │                                            │   │
│  │                              ▼                                            │   │
│  │                    K8s Service (ClusterIP)                                 │   │
│  │                              │                                            │   │
│  │                              ▼                                            │   │
│  │              ┌──────────────────────────────┐                             │   │
│  │              │     Application Pod          │                             │   │
│  │              │  ┌────────┐  ┌────────────┐  │                             │   │
│  │              │  │  App   │  │ Envoy Side │  │                             │   │
│  │              │  │Container│  │   car      │  │                             │   │
│  │              │  └────────┘  └─────┬──────┘  │                             │   │
│  │              │                    │CSI Mount │                             │   │
│  │              └────────────────────┼─────────┘                             │   │
│  └───────────────────────────────────┼──────────────────────────────────────┘   │
│                                      │                                           │
│  ┌───────────────────────────────────┼──────────────────────────────────────┐   │
│  │  Service Mesh (East-West Traffic) │                                       │   │
│  │                                   │ SPIFFE UDS Socket                     │   │
│  │                                   ▼                                       │   │
│  │  ┌─────────────────────────────────────────────────────────────┐          │   │
│  │  │           Istio Control Plane (istiod)                       │          │   │
│  │  │   - VirtualService (라우팅)                                  │          │   │
│  │  │   - DestinationRule (mTLS: ISTIO_MUTUAL)                     │          │   │
│  │  │   - PeerAuthentication (STRICT)                              │          │   │
│  │  │   - AuthorizationPolicy (접근제어)                           │          │   │
│  │  └─────────────────────────────────────────────────────────────┘          │   │
│  └───────────────────────────────────────────────────────────────────────────┘   │
│                                                                                  │
│  ┌───────────────────────────────────────────────────────────────────────────┐   │
│  │  SPIFFE/SPIRE Layer (Zero Trust Workload Identity Manager)                 │   │
│  │                                                                            │   │
│  │  ┌────────────┐  ┌────────────┐  ┌──────────────┐  ┌─────────────────┐   │   │
│  │  │ SPIRE      │  │ SPIRE      │  │ SPIFFE CSI   │  │ OIDC Discovery  │   │   │
│  │  │ Server     │  │ Agent      │  │ Driver       │  │ Provider        │   │   │
│  │  │(StatefulSet)│  │(DaemonSet) │  │(DaemonSet)   │  │(Deployment)     │   │   │
│  │  └────────────┘  └────────────┘  └──────────────┘  └─────────────────┘   │   │
│  │                                                                            │   │
│  │  Namespace: zero-trust-workload-identity-manager                           │   │
│  └───────────────────────────────────────────────────────────────────────────┘   │
│                                                                                  │
└──────────────────────────────────────────────────────────────────────────────────┘
```

### 1.3 트래픽 흐름 상세

```
[North-South Traffic]
  Client
    ↓
  F5 BIG-IP (External LB)
    ↓ (CIS가 VirtualServer 감지)
  NGINX Ingress Controller Plus
    ↓ (VirtualServer CR에 의해 라우팅)
  K8s Service (ClusterIP)
    ↓
  Application Pod (Envoy Sidecar가 트래픽 인터셉트)
    ↓ (SPIRE 발급 인증서로 mTLS)
  Backend Service Pod

[East-West Traffic]
  Pod A (Envoy Sidecar)
    ↓ mTLS (SPIRE SVID)
  Pod B (Envoy Sidecar)
    - Istio VirtualService가 라우팅 제어
    - DestinationRule이 TLS 모드 결정
    - PeerAuthentication이 STRICT mTLS 강제
    - AuthorizationPolicy가 접근 허용/거부
```

---

## 2. 설치 절차 (순서 중요)

### 전체 설치 순서

| 단계 | 작업 | 네임스페이스 | 비고 |
|------|------|-------------|------|
| 1 | ZTWIM Operator 설치 | zero-trust-workload-identity-manager | OperatorHub에서 설치 |
| 2 | CreateOnly Mode 활성화 | zero-trust-workload-identity-manager | Istio 연동 시 필수 |
| 3 | ZeroTrustWorkloadIdentityManager CR | zero-trust-workload-identity-manager | Trust Domain 정의 |
| 4 | SpireServer CR | zero-trust-workload-identity-manager | CA 서버 역할 |
| 5 | SpireAgent CR | zero-trust-workload-identity-manager | 노드별 DaemonSet |
| 6 | SpiffeCSIDriver CR | zero-trust-workload-identity-manager | Pod에 소켓 마운트 |
| 7 | SpireOIDCDiscoveryProvider CR | zero-trust-workload-identity-manager | JWT 검증 endpoint |
| 8 | OpenShift Service Mesh Operator 설치 | - | OperatorHub에서 설치 |
| 9 | IstioCNI CR | istio-cni | CNI 네트워크 설정 |
| 10 | Istio CR (SPIRE 연동) | istio-system | 핵심 연동 설정 |
| 11 | 애플리케이션 네임스페이스 설정 | app namespaces | Injection 라벨 |
| 12 | PeerAuthentication 적용 | app namespaces | mTLS STRICT |
| 13 | AuthorizationPolicy 적용 | app namespaces | Zero Trust 접근제어 |
| 14 | 애플리케이션 배포 | app namespaces | Sidecar + SPIRE annotation |

---

## 3. Zero Trust Namespace Injection 전략

### 3.1 모든 네임스페이스에 Injection해야 하는가?

**결론: 아니오. 선택적으로 적용해야 합니다.**

#### 제외해야 하는 OCP 엔진 네임스페이스들

다음 네임스페이스는 Istio sidecar injection을 하면 **안 됩니다**:

| 네임스페이스 패턴 | 이유 |
|-------------------|------|
| `openshift-*` | OCP 핵심 인프라 컴포넌트 (API Server, etcd, SDN 등) |
| `kube-system` | Kubernetes 핵심 시스템 |
| `kube-public` | 공개 설정 |
| `default` | 기본 네임스페이스 (사용 자제) |
| `istio-system` | Istio Control Plane 자체 |
| `istio-cni` | CNI 네트워크 구성요소 |
| `zero-trust-workload-identity-manager` | SPIRE 인프라 구성요소 |

#### Injection 대상 네임스페이스 (애플리케이션 워크로드)

```bash
# 대상 네임스페이스에 라벨 적용
oc label namespace <app-namespace> istio-injection=enabled
```

#### 권장 전략

```
[ Mesh에 포함 ]                    [ Mesh에서 제외 ]
┌──────────────────────┐          ┌──────────────────────┐
│ - app-frontend       │          │ - openshift-*        │
│ - app-backend        │          │ - kube-system        │
│ - app-database       │          │ - istio-system       │
│ - app-messaging      │          │ - istio-cni          │
│ - dev-*              │          │ - zero-trust-*       │
│ - staging-*          │          │ - monitoring (선택)   │
│ - prod-*             │          │ - logging (선택)      │
└──────────────────────┘          └──────────────────────┘
```

### 3.2 점진적 적용 권장 절차

1. **Phase 1**: 하나의 비중요 애플리케이션 네임스페이스에 PERMISSIVE 모드로 적용
2. **Phase 2**: 정상 동작 확인 후 STRICT mTLS로 전환
3. **Phase 3**: 나머지 애플리케이션 네임스페이스로 확대 적용
4. **Phase 4**: AuthorizationPolicy로 세밀한 접근제어 적용

---

## 4. NGINX IC Plus VirtualServer → Service Mesh VirtualService 연동

### 4.1 트래픽 흐름 설계

```
NGINX IC Plus (VirtualServer CR)
        │
        │ upstream: service-name.namespace.svc.cluster.local:port
        │
        ▼
  K8s Service (ClusterIP, port 80/443)
        │
        │ (Envoy Sidecar가 인바운드 트래픽 인터셉트)
        │
        ▼
  Istio VirtualService (내부 라우팅 규칙)
        │
        │ - weight-based routing
        │ - header-based routing
        │ - fault injection
        │ - retry/timeout
        │
        ▼
  Backend Pod (SPIRE 인증서로 mTLS 통신)
```

### 4.2 핵심 포인트

1. **NGINX IC Plus VirtualServer**는 외부 트래픽을 클러스터 내부 서비스로 전달 (단일 서비스 라우팅)
2. **Istio VirtualService**는 서비스 메시 내부에서 세밀한 라우팅 (canary, A/B, retry 등)
3. NGINX IC 자체는 sidecar injection 대상이 아님 (별도 네임스페이스에서 운영)
4. 애플리케이션 Pod에 진입하는 시점에서 Envoy sidecar가 mTLS를 처리

---

## 5. 파일 구조

```
ocp-spiffe-spire-servicemesh/
├── 00-architecture-guide.md          # 이 문서
├── 01-ztwim-operator-config.yaml     # CreateOnly Mode 설정
├── 02-ztwim-cr.yaml                  # ZeroTrustWorkloadIdentityManager CR
├── 03-spire-server.yaml              # SpireServer CR
├── 04-spire-agent.yaml               # SpireAgent CR
├── 05-spiffe-csi-driver.yaml         # SpiffeCSIDriver CR
├── 06-spire-oidc-provider.yaml       # SpireOIDCDiscoveryProvider CR
├── 07-istio-cni.yaml                 # IstioCNI CR
├── 08-istio-cr-spire.yaml            # Istio CR (SPIRE 연동)
├── 09-namespace-setup.yaml           # 네임스페이스 라벨링
├── 10-nginx-virtualserver.yaml       # NGINX IC VirtualServer 예시
├── 11-istio-virtualservice.yaml      # Istio VirtualService 예시
├── 12-peer-authentication.yaml       # PeerAuthentication (STRICT mTLS)
├── 13-authorization-policy.yaml      # AuthorizationPolicy (Zero Trust)
├── 14-destination-rules.yaml         # DestinationRule (ISTIO_MUTUAL)
└── 15-sample-app-deployment.yaml     # 샘플 애플리케이션 (SPIRE annotation)
```

---

## 6. 참고 사항

### 주요 참조 문서

- [Red Hat OSSM 3.4 - SPIRE Integration](https://docs.redhat.com/en/documentation/red_hat_openshift_service_mesh/3.4/html/installing/ossm-spire_ossm-cert-manager)
- [OCP 4.21 - Zero Trust Workload Identity Manager](https://docs.redhat.com/en/documentation/openshift_container_platform/4.21/html/security_and_compliance/zero-trust-workload-identity-manager)
- [Istio SPIRE Integration](https://istio.io/latest/docs/ops/integrations/spire/)

### SPIRE + Service Mesh 연동 시 Technology Preview 주의

현재(ZTWIM 1.1.0, OSSM 3.4 기준) Service Mesh와 SPIRE 연동은 **Technology Preview** 상태입니다. 프로덕션에서는 Red Hat SLA 적용 범위를 확인하세요.

### 주요 환경 변수 (설치 전 확인)

```bash
# 클러스터 도메인 확인
export CLUSTER_DOMAIN=$(oc get ingresses.config/cluster -o jsonpath='{.spec.domain}')

# Trust Domain 설정 (클러스터 apps 도메인 권장)
export TRUST_DOMAIN=${CLUSTER_DOMAIN}

# ZTWIM 네임스페이스
export ZTWIM_NS=zero-trust-workload-identity-manager

# JWT Issuer (OIDC Discovery Provider URL)
export JWT_ISSUER="https://oidc-discovery.${CLUSTER_DOMAIN}"

# Istio 네임스페이스
export OSSM_NS=istio-system
export OSSM_CNI=istio-cni
```
