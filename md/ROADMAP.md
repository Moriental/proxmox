# 홈랩 로드맵: AI 워크로드를 위한 셀프서비스 플랫폼

> 방향성 목표: 리소스가 제한된 온프렘 k8s 클러스터 위에, GitOps로 배포되는 GPU 기반 RAG 서빙 플랫폼을 운영하며, 그 과정의 아키텍처 결정과 장애 대응을 문서화한다.

하드웨어: i5-8400 / GTX 1060 3GB / DDR4 16GB (단일 물리 호스트, Proxmox)

---

## Phase 0 — 클러스터 기반: **완료**

- [x] Proxmox 설치, 네트워크(vmbr0) 구성
- [x] Ubuntu 24.04 cloud-init 템플릿 제작
- [x] kubeadm 3-node(마스터1 + 워커2, v1.35) 클러스터 구성, **Calico** CNI (Flannel에서 변경 — NetworkPolicy 지원 필요해서 처음부터 Calico로 결정)
- **완료 기준**: `kubectl get nodes` 3대 Ready — 확인됨
- 트러블슈팅 기록: `proxmox-homelab-troubleshooting.md`, 상세 진행 로그: `PROGRESS.md`

## Phase 1 — 플랫폼 뼈대 (GitOps + CI)

- [ ] ArgoCD 설치 (pull 기반 GitOps)
- [ ] Git 레포에 매니페스트/Helm 차트 구조 설계
- [ ] Jenkins: 이미지 빌드 → 레지스트리 푸시까지만 담당 (배포는 ArgoCD가 git 상태 감지해서 sync)
- **산출물**: 빌드→레지스트리→GitOps sync→배포 아키텍처 다이어그램, "git push 한 번으로 배포" 데모
- **인터뷰 포인트**: 전통적 push형 CD와 GitOps(pull형)의 차이를 왜 선택했는지

## Phase 2 — 첫 대표 워크로드: llama.cpp 서빙

- [ ] GPU 노드에 taint/toleration + resource limit으로 llama.cpp 파드 스케줄링
- [ ] Helm 차트화 후 ArgoCD Application으로 등록 (Phase 1 파이프라인에 온보딩)
- **산출물**: GPU 워크로드 스케줄링 설명
- **인터뷰 포인트**: 3GB VRAM 제약 안에서 어떻게 리소스를 관리했는가 (모델 성능이 아니라 운영 관점)

## Phase 3 — RAG로 확장 (마이크로서비스화)

- [ ] 임베딩 서비스 + 벡터DB(Chroma/Qdrant) + llama.cpp = 3개 서비스로 분리
- [ ] 서비스 메시 도입 검토: Linkerd 우선 추천 (사이드카 경량, 리소스 제약 환경에 적합), Istio는 이름값은 높지만 리소스 여유 생긴 후로 보류
- **산출물**: RAG 파이프라인 아키텍처, (메시 도입 시) 서비스 간 트래픽 관측
- **인터뷰 포인트**: Istio 대신 Linkerd를 고른 이유 (또는 그 반대 결정을 내렸다면 그 이유)

## Phase 4 — 관측성 & 복원력

- [ ] Prometheus + Grafana + Loki로 GPU/파드/로그 관측
- [ ] 의도적 장애 주입(노드 다운, 디스크 풀, pod crash loop) → 포스트모템 문서화
- **산출물**: 장애 대응 로그, 간단한 SLI/SLO 정의
- **인터뷰 포인트**: 장애를 어떻게 재현하고 어떤 지표로 감지·대응했는가

## Phase 5 — 여유 시 확장 (우선순위 낮음)

- [ ] Tailscale로 AWS VPC 연결 → 서버리스 게임 프로젝트와 "서버리스 vs k8s 운영 비용/복잡도" ADR 비교
- [ ] kagent 등 최신 AI-on-k8s 도구 실험

---

## 리소스 운용 원칙

16GB RAM으로 모든 스택을 동시에 못 켠다. 데모/작업 중인 Phase의 스택만 활성화하고, 나머지는 `kubectl scale --replicas=0`으로 내려둔다. 이 우선순위 판단 자체도 인터뷰에서 쓸 수 있는 이야기다.

---

*작성: 2026-08-22*
