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
