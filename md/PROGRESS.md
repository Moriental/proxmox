# 홈랩 진행 상황 + 학습 정리

> 컨텍스트가 길어져도 여기부터 다시 읽으면 이어갈 수 있게 정리한 문서.
> 관련 문서: 트러블슈팅은 `proxmox-homelab-troubleshooting.md`, 방향성/로드맵은 `ROADMAP.md`, 논의만 하고 미실행인 항목은 `CONSIDERATIONS.md`.

---

## 1. 하드웨어 / 네트워크 현황

- **하드웨어**: i5-8400 / GTX 1060 3GB / DDR4 16GB (기존 유휴 데스크탑 재활용, "미니PC" 아님)
- **디스크**: Samsung 970 EVO NVMe 250GB
  - `pve-root` 67.7G (Proxmox OS)
  - `pve-swap` 8G
  - `pve-data`(local-lvm, thin pool) 136.5G — **VM 디스크는 전부 여기**
- **네트워크**: 유선 연결 필수(Wi-Fi로는 브리지 불가 확인함). 현재 **이중 NAT** 상태:
  - 메인 공유기(192.168.0.x) 아래에 ipTIME을 무선 확장으로 연결했는데, ipTIME이 라우터 모드로 동작하며 자기 내부 대역을 `192.168.1.x`로 자동 변경함
  - **Proxmox 호스트(pve)는 `192.168.1.2`, 게이트웨이 `192.168.1.1`** — 이후 만드는 모든 VM도 이 대역(`192.168.1.x`)으로 맞춰야 함 (과거 안내에서 `192.168.0.x`로 잘못 얘기한 적 있으니 주의)
  - 향후 개선 옵션: ipTIME을 AP(브리지) 모드로 바꿔서 단일 대역으로 통합 가능

## 2. Proxmox 설치 단계에서 배운 것

- Rufus로 USB 구울 때 **DD 모드가 정상** (ISO 모드 아님), USB가 드라이브 2개로 보이는 것도 정상(하이브리드 이미지)
- 설치 직후 `pve-enterprise` 유료 저장소 때문에 401 에러 남 → `.sources` 파일을 `.disabled`로 바꾸고 `pve-no-subscription` 무료 저장소 추가해야 `apt update` 정상 동작. **Ceph 저장소도 별도로 유료라 같이 비활성화** 필요(단일 노드라 Ceph 자체가 불필요)
- **`local` vs `local-lvm` 헷갈리지 말 것**: `local`은 ISO/템플릿/백업 저장용(디렉토리, pve-root 위), `local-lvm`은 VM/CT 디스크 전용(thin pool). ISO 업로드는 `local`에 함

## 3. 가상화 개념 정리

- **`qm`은 QEMU 명령어가 아니라 Proxmox 자체 CLI 래퍼.** 내부적으로 QEMU 프로세스를 직접 관리(libvirt 안 씀). Proxmox의 "VM"은 **QEMU/KVM 기반**이고, "LXC"(컨테이너)는 커널을 호스트와 공유하는 별개 기술 — 지금 하는 건 전부 VM(QEMU/KVM) 쪽
- **소켓(socket) vs 코어(core)**: 소켓 = 물리 CPU 개수. i5-8400은 1소켓짜리라 VM도 소켓 1 / 코어 2로 설정 (2소켓으로 하면 존재하지 않는 CPU를 에뮬레이션하려다 손해)
- **그래픽카드(Display) 설정은 물리 GPU와 무관** — 그냥 콘솔 화면 보여주는 가상 어댑터. GTX 1060을 VM이 실제로 쓰게 하려면 **PCI Passthrough**(IOMMU 활성화, vfio-pci 바인딩, PCI 장치 추가)라는 완전히 별도 작업 필요 → 이건 로드맵 **Phase 2**(llama.cpp GPU 서빙)에서 다룰 예정

## 4. ISO 설치 vs Cloud-init 템플릿

- **ISO**(`ubuntu-24.04.3-live-server-amd64.iso`) = 설치 프로그램. 부팅 후 마법사를 매번 손으로 진행해야 완성된 OS가 됨
- **Cloud image**(`noble-server-cloudimg-amd64.img`) = 이미 설치 끝난 디스크 이미지. `qm importdisk`로 붙이고 cloud-init 드라이브(ide2)를 추가하면, 첫 부팅 때 IP/계정/SSH키가 자동 주입됨 → 이후엔 `qm clone` 한 줄로 VM 복제 가능
- 둘은 완전히 다른 파일이라 서로 재사용 불가. ISO는 `local` 스토리지에 그대로 둬도 무방(다른 스토리지 차지, 용량 문제 없음, 나중에 수동 설치/rescue용으로 재사용 가능)
- 설치 마법사에서 겪은 디테일:
  - 체크섬(해시)은 선택 사항, 생략 가능
  - Search domain은 사설 DNS 안 쓰면 불필요
  - "Import SSH identity"(GitHub/Launchpad 연동)는 스킵해도 됨 — 스킵하면 그냥 비밀번호 로그인만 가능한 상태가 됨
- **GUI로 마스터를 수동 설치하다가 비밀번호를 깜빡한 이슈 발생** → 이미 설치 과정 자체는 한 번 학습했으니, 이후 마스터/워커 **3대 전부 cloud-init 템플릿 방식으로 통일**하기로 결정함

## 5. 방향성 / 로드맵 (`ROADMAP.md` 참고)

여러 방향(플랫폼 엔지니어링 / SRE 복원력 / 하이브리드 비교 / AI 인프라) 중 **"AI 인프라 + 플랫폼 엔지니어링"을 하나로 묶기로 결정**:

> "리소스가 제한된 온프렘 k8s 클러스터 위에, GitOps로 배포되는 GPU 기반 RAG 서빙 플랫폼을 운영하며, 그 과정의 아키텍처 결정과 장애 대응을 문서화한다."

- Phase 0: 클러스터 기반 (진행 중)
- Phase 1: GitOps(ArgoCD) + CI(Jenkins)
- Phase 2: llama.cpp GPU 서빙 (PCI passthrough 필요)
- Phase 3: RAG 마이크로서비스화 + 서비스 메시(Linkerd 우선 추천, 리소스 이유)
- Phase 4: 관측성(Prometheus/Grafana/Loki) + 의도적 장애 주입/포스트모템
- Phase 5(여유 시): Tailscale로 AWS 연결, kagent 등

리소스 원칙: 16GB RAM으로 전부 동시 구동 불가 → 필요한 스택만 켜고 나머지는 스케일다운.

## 6. 현재 진행 상태

### Phase 0 — 클러스터 기반: **완료**

- [x] Proxmox 설치 + 저장소(enterprise/ceph) 비활성화, no-subscription 추가
- [x] 네트워크 확인 (`192.168.1.x` 대역, 이중 NAT 상태 파악)
- [x] Ubuntu 24.04 ISO 업로드 (`local` 스토리지, 보관용으로 유지)
- [x] cloud-init 템플릿(VM ID 9000) 제작
- [x] 마스터(101, `192.168.1.101`) + 워커1(102, `.102`) + 워커2(103, `.103`) 클론
- [x] kubeadm 사전작업(swap off, containerd, kubeadm/kubelet/kubctl v1.35 설치)
- [x] `kubeadm init`(마스터, pod-network-cidr=10.244.0.0/16) + **Calico**(Flannel 대신 — NetworkPolicy 필요해서 변경) + `kubeadm join`(워커 2대)
- [x] `kubectl get nodes` 3대 Ready 확인
- [x] kubectl bash-completion + `k` alias 설정 (`source ~/.bashrc` 필요했음)

### 부수적으로 해결한 이슈
- 홈 디렉토리(`C:\Users\admin`) 전체가 실수로 git 저장소에 물려있던 것 발견 → 삭제, `proxmox` 폴더를 독립 저장소로 재`init`, `.gitignore` 추가
- 윈도우에서 VM SSH 접속용 키를 PVE에서 복사하다 파일이 손상(CRLF + 내용 손상)되는 문제 → 윈도우에서 별도 키페어(`id_ed25519_homelab`) 새로 생성 후 공개키만 각 VM `authorized_keys`에 등록하는 방식으로 전환

## 7. Phase 1 — 플랫폼 뼈대 (GitOps + CI): 진행 중

**완료된 것 (ArgoCD)**:
- [x] Helm 설치 (curl 공식 스크립트 방식 — apt 저장소 방식도 검토했으나 번거로워서 제외)
- [x] ArgoCD를 Helm으로 설치 (`helm repo add argo`, `argocd` 네임스페이스, `helm install argocd argo/argo-cd -n argocd`)
- [x] 초기 admin 비밀번호 확인 (`argocd-initial-admin-secret` 시크릿, base64 디코딩)
- [x] `kubectl port-forward`로 1차 접속 → NodePort로 전환 → Helm values로 선언적 관리
- [ ] Git 레포에 매니페스트/Helm 차트 구조 설계 — 미착수
- [ ] Jenkins(CI 전용) 설치 — 미착수

**과정 요약**:
1. `kubectl port-forward svc/argocd-server -n argocd 8080:443`으로 첫 접속 시도
2. NodePort로 영구 전환 결정: `helm show values argo/argo-cd`로 `server.service.type`/`nodePortHttps` 키 확인 → `helm pull --untar` 후 `grep -rl nodePortHttps`로 실제 템플릿 위치 검증
3. `argocd-values.yaml` 작성(`server.service.type: NodePort`, `nodePortHttps: 30443`) → `helm upgrade argocd argo/argo-cd -n argocd -f argocd-values.yaml`
4. `kubectl get svc argocd-server -n argocd`로 `443:30443/TCP` 확인, `https://192.168.1.101:30443` 접속 성공

**트러블슈팅 (겪은 순서대로)**:
- `kubectl port-forward`는 기본적으로 `127.0.0.1`에만 바인딩 → 외부(윈도우)에서 접근하려면 `--address 0.0.0.0` 필요했음
- port-forward 세션이 끊기면서 브라우저에 `Request has been terminated` (Angular 범용 네트워크 에러, CORS 아니었음) — SSH/터미널이 살아있어야 유지되는 임시 터널이라 발생. → NodePort 전환의 직접적 계기
- `helm upgrade argo/argo-cd -n argocd -f argocd-values.yaml` 실행 시 `Error: "helm upgrade" requires 2 arguments` — 릴리스 이름(`argocd`)을 빠뜨림. `helm upgrade <릴리스이름> <차트경로> ...` 순서 필요
- kubeadm 설치 가이드에 처음 v1.31을 썼다가, 실제로는 EOL된 버전이라는 걸 확인 후 지원 버전(1.34/1.35/1.36) 중 **1.35**로 정정

**개념 정리**:
- Helm values override의 원리: 차트(템플릿) 자체를 바꾸는 게 아니라, 차트 개발자가 이미 만들어둔 파라미터(`{{ .Values.xxx }}`로 템플릿에 이미 박혀있는 자리)에 값을 채워넣는 것. `-f values.yaml`이 표준 패턴(git으로 diff 추적 가능), `--set`은 한두 개 급할 때만, 차트 전체 다운로드(`helm pull --untar`)는 기본값 탐색/오프라인용
- `helm show values <chart>`로 전체 기본값 확인, `helm template`으로 적용 전 렌더링 결과 미리보기 가능

## 8. 원격 접근 — Cloudflare Tunnel: 완료

**목적**: 노트북 등 외부 네트워크에서 Proxmox 웹 UI / ArgoCD에 접근. Tailscale은 이미 많이 써봐서, 이번엔 Cloudflare Tunnel로 시도 (Tailscale은 로드맵 Phase 5의 AWS 하이브리드 연결용으로 별도 유지 — 용도가 다름, 대체 관계 아님).

**사전 확인된 것**:
- 보유 중인 가비아 도메인 `moriental.store` 사용 — 예전에 AWS Route53으로 네임서버를 옮겼던 적 있었으나, 해당 Route53 호스팅 영역은 이미 삭제된 상태라 안전하게 재사용 가능 (걸려있던 레코드 없음)
- Cloudflare Tunnel은 아웃바운드 전용 연결이라 인바운드 포트포워딩 불필요 → 이중 NAT 환경 자동 우회
- 접근 제어(ACL)는 Tunnel 자체엔 없고 **Cloudflare Access**(무료 티어, 이메일 허용목록)를 별도로 걸어야 함
- 서브도메인(`proxmox.moriental.store`, `argocd.moriental.store`)만 쓰므로 루트 도메인은 향후 별도 웹사이트 용도로 그대로 사용 가능

**진행 상태**:
- [x] Cloudflare 계정 생성, 도메인 추가("Add a domain")
- [x] DNS 레코드 스캔 (0개 — 미사용 도메인이라 정상)
- [x] 임시 A 레코드(`@` → `192.0.2.1`) 추가 — Cloudflare가 레코드 0개 상태로는 사이트 활성화를 안 시켜줘서 더미값으로 통과
- [x] 네임서버 발급받아 가비아에 등록, 전파 확인
- [x] cloudflared 설치 (Proxmox 호스트)
- [x] 터널 생성 + ingress 설정(`config.yml`) + DNS 라우팅
- [x] Cloudflare Access 정책 설정 (이메일 허용목록: 본인 이메일 1개)
- [x] systemd 서비스 등록 + 외부망에서 검증 — `proxmox.moriental.store`, `argocd.moriental.store` 둘 다 접속 확인 완료

**트러블슈팅 / 주의사항**:
- Cloudflare UI가 "Add a Site"에서 "Add a domain"/"Onboard a domain"으로 명칭 변경됨 (검색으로 확인)
- 온보딩 중 "Configure AI training" 같은 AI 크롤러 차단 설정 단계가 새로 추가됨 — 이번 작업과 무관, 기본값 두고 통과
- "Without DNS records, Cloudflare is unable to activate your site" — 빈 상태로는 활성화 불가, 더미 A 레코드(`192.0.2.1`, 예약된 테스트용 IP)로 해결
- "발원지에서 Cloudflare IP 주소만 허용하세요" 안내 — **Tunnel 방식에는 해당 없음**. 이 경고는 원본 서버가 공인 IP로 직접 노출된 전통적 프록시 구성 대상이고, Tunnel은 애초에 인바운드 포트가 없어 원본 직접 접근 자체가 불가능하므로 자동으로 충족됨
- DNS 라우팅 직후 `nslookup`(8.8.8.8)이 계속 NXDOMAIN — 레코드는 정상 생성되어 있었고, 8.8.8.8의 네거티브 캐싱(이전 조회 결과를 잠깐 기억)이 원인. `nslookup ... 1.1.1.1`로 우회 조회해서 실제로는 정상임을 확인
- 브라우저(Chrome/Edge)는 OS DNS 캐시와 별개로 자체 Secure DNS(DoH) 캐시를 가짐 — `ipconfig /flushdns`로는 안 풀렸고, 시스템 DNS 서버를 1.1.1.1로 변경하고 나서야 해결
- Access 로그인 시 이메일 인증코드 입력 화면 없이 바로 통과되는 현상 발생 → Zero Trust의 로그인 방법(Login Methods)에 "Cloudflare 계정으로 로그인"이 켜져 있었고, 이미 로그인된 Cloudflare 세션으로 통과된 것이었음. 정책의 `Include: Emails` 조건 자체는 정상 작동 중이며(다른 이메일 소유자는 여전히 차단됨), 보안 문제가 아님을 정책 화면 확인으로 검증함

## 9. 다음 단계

- Git 레포에 매니페스트/Helm 차트 구조 설계
- Jenkins 설치 (CI 전용) — 가이드는 전달됐으나 실행 확인 전 (`md/CONSIDERATIONS.md` 참고)
- 상세 가이드는 다음 대화에서 요청하면 됨

---

*마지막 업데이트: 2026-08-22*
