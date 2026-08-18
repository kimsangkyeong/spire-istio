# SPIRE 제거 및 Istio 내장 CA 전환 가이드

## 1. 왜 수정 작업이 필요한가?

### 1.1 문제 상황

SPIFFE/SPIRE를 Service Mesh와 연동하여 운영 중이었으나, 의사결정에 의해 SPIRE Server를 제거하기로 했습니다. 이 경우 단순히 SPIRE 리소스만 삭제하면 **서비스 전체 장애**가 발생합니다.

### 1.2 단순 삭제 시 발생하는 문제

```
[현재 상태]
Envoy Sidecar → SPIRE Agent (SDS API) → SPIRE Server
                     ↑
              CSI Driver가 소켓 마운트

[SPIRE 삭제 직후]
Envoy Sidecar → SPIRE Agent (없음!) → ❌ 인증서 획득 불가
                     ↑
              CSI Driver (없음!) → ❌ 소켓 마운트 실패

결과:
- 모든 Pod의 Envoy가 인증서를 갱신할 수 없음
- STRICT mTLS 환경에서 인증서 만료 시 모든 서비스 간 통신 차단
- 새 Pod 배포 시 CSI 볼륨 마운트 실패로 Pod 시작 불가
- 기존 Pod도 인증서 TTL(기본 1시간) 만료 후 통신 불가
```

### 1.3 수정이 필요한 이유 상세

| 영역 | 변경 필요 이유 |
|------|----------------|
| **Istio CR** | SPIRE 연동 설정(spire 템플릿, WORKLOAD_IDENTITY_SOCKET_FILE, jwksResolverExtraRootCA) 제거 필요. Istio 내장 CA(citadel)로 전환해야 Envoy가 istiod에서 인증서 획득 |
| **Pod Annotations** | `inject.istio.io/templates: "sidecar,spire"` → `"sidecar"`로 변경. spire 템플릿이 참조하는 CSI 볼륨이 없으므로 Pod 시작 실패 |
| **CSI 볼륨 참조** | SPIFFE CSI Driver 삭제 후 `csi.spiffe.io` 드라이버가 존재하지 않으므로, 해당 볼륨을 사용하는 모든 Pod가 Pending 상태에 빠짐 |
| **PeerAuthentication** | mTLS 모드 자체는 유지 가능하나, 인증서 발급 주체가 SPIRE → Istio CA로 변경되므로 전환 기간 동안 PERMISSIVE 모드 필요 |
| **AuthorizationPolicy** | SPIFFE ID 기반 principals가 Istio SA 기반으로 변경됨. `cluster.local/ns/.../sa/...` 형태는 동일하나 Trust Domain이 달라질 수 있음 |
| **Trust Domain** | SPIRE: `apps.ocp4.example.com` → Istio 내장: `cluster.local` (기본값). principals 참조 수정 필요 |

### 1.4 인증서 발급 메커니즘 비교

```
[SPIRE 사용 시]
Pod → Envoy Sidecar → SPIRE Agent (UDS Socket via CSI)
                              ↓
                       SPIRE Server (CA)
                              ↓
                       X.509 SVID 발급
Trust Domain: apps.ocp4.example.com
SPIFFE ID: spiffe://apps.ocp4.example.com/ns/<ns>/sa/<sa>

[Istio 내장 CA 사용 시]
Pod → Envoy Sidecar → istiod (xDS/SDS API via gRPC)
                              ↓
                       Istio CA (citadel 내장)
                              ↓
                       X.509 인증서 발급
Trust Domain: cluster.local
Identity: spiffe://cluster.local/ns/<ns>/sa/<sa>
```

---

## 2. 전환 전략: 무중단 마이그레이션

### 2.1 전환 순서 개요

```
Phase 1: PERMISSIVE 전환 (평문 + mTLS 모두 허용)
    ↓
Phase 2: Istio CR에서 SPIRE 설정 제거 (Istio 내장 CA 활성화)
    ↓
Phase 3: 애플리케이션 Pod annotation 수정 + 롤링 재시작
    ↓
Phase 4: 새 인증서로 통신 정상 확인
    ↓
Phase 5: STRICT mTLS 복원
    ↓
Phase 6: AuthorizationPolicy Trust Domain 수정
    ↓
Phase 7: SPIRE 리소스 제거 (역순)
```

### 2.2 상세 전환 단계

| 단계 | 작업 | 파일 | 위험도 |
|------|------|------|--------|
| 1 | PeerAuthentication → PERMISSIVE | `01-permissive-transition.yaml` | 낮음 |
| 2 | Istio CR 수정 (SPIRE 연동 제거) | `02-istio-cr-without-spire.yaml` | 중간 |
| 3 | Pod annotation 수정 + 재배포 | `03-app-redeploy-no-spire.yaml` | 중간 |
| 4 | 정상 통신 확인 후 STRICT 복원 | `04-strict-restoration.yaml` | 낮음 |
| 5 | AuthorizationPolicy 수정 | `05-authz-policy-update.yaml` | 낮음 |
| 6 | SPIRE 리소스 제거 | `06-spire-cleanup.yaml` | 낮음 |

---

## 3. 주의 사항

### 3.1 다운타임 최소화

- **반드시 PERMISSIVE 먼저 적용** 후 Istio CR을 수정하세요
- PERMISSIVE 모드에서는 평문과 mTLS 모두 허용하므로, SPIRE 인증서가 만료되어도 통신 유지
- 새 Pod가 Istio 내장 CA에서 인증서를 받은 후에 STRICT로 복원

### 3.2 롤백 불가 포인트

- SPIRE CR 삭제 후에는 SPIRE 데이터(PVC)도 함께 삭제됨
- 재설치 시 새로운 CA 키가 생성되므로 기존 SVID와 호환 불가
- **Phase 7(SPIRE 삭제)은 모든 검증 완료 후 마지막에 실행**

### 3.3 NGINX IC Plus 영향

- NGINX IC Plus는 mesh 외부에서 운영되므로 SPIRE 제거에 직접적 영향 없음
- VirtualServer CR은 수정 불필요
- 단, Frontend Pod가 재시작되는 동안 일시적 503 가능 → rolling update로 최소화

---

## 4. 파일 구조

```
ocp-spire-removal-guide/
├── 00-spire-removal-guide.md              # 이 문서
├── 01-permissive-transition.yaml          # Phase 1: mTLS 완화
├── 02-istio-cr-without-spire.yaml         # Phase 2: Istio CR 수정
├── 03-app-redeploy-no-spire.yaml          # Phase 3: App 재배포
├── 04-strict-restoration.yaml             # Phase 4: STRICT 복원
├── 05-authz-policy-update.yaml            # Phase 5: 접근제어 수정
└── 06-spire-cleanup.yaml                  # Phase 6: SPIRE 리소스 삭제
```

---

## 5. 전환 전후 아키텍처 비교

### Before (SPIRE 연동)

```
Client → F5 → NGINX IC Plus → K8s Service → Pod
                                               │
                                          Envoy Sidecar
                                               │
                                          SPIRE Agent (CSI Mount)
                                               │
                                          SPIRE Server (CA)
                                               │
                                          X.509 SVID
Trust Domain: apps.ocp4.example.com
```

### After (Istio 내장 CA)

```
Client → F5 → NGINX IC Plus → K8s Service → Pod
                                               │
                                          Envoy Sidecar
                                               │
                                          istiod (gRPC SDS)
                                               │
                                          Istio CA (내장)
                                               │
                                          X.509 Certificate
Trust Domain: cluster.local
```

### 유지되는 것

- NGINX IC Plus VirtualServer (변경 없음)
- Istio VirtualService / DestinationRule (라우팅 로직 유지)
- Service Mesh sidecar injection (annotation만 변경)
- mTLS 통신 (인증서 발급 주체만 변경)

### 제거되는 것

- ZeroTrustWorkloadIdentityManager CR
- SpireServer, SpireAgent, SpiffeCSIDriver, SpireOIDCDiscoveryProvider
- SPIFFE CSI 볼륨 마운트
- SPIRE 전용 injection 템플릿
- `zero-trust-workload-identity-manager` 네임스페이스 (선택)
