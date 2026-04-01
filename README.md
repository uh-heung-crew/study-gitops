# study-gitops 프로젝트 분석 보고서

## 1. 이 프로젝트는 뭐야?

**GitOps 방식으로 Kubernetes 클러스터를 관리하는 학습용 프로젝트**이다.

핵심 아이디어는 간단하다:
> "Git 저장소에 있는 YAML 파일을 수정 → Push하면 → 쿠버네티스 클러스터에 자동 반영"

수동으로 서버에 접속해서 명령어를 치는 대신, **Git이 곧 배포의 진실의 원천(Single Source of Truth)** 이 되는 방식이다.

---

## 2. 핵심 개념 정리 (완전 처음부터)

### 2-1. Kubernetes (쿠버네티스, K8s)

컨테이너(Docker 등)를 **자동으로 배포하고, 관리하고, 확장**해주는 플랫폼이다.

| 개념 | 쉬운 설명 |
|---|---|
| **Pod** | 컨테이너 1개(또는 여러개)를 감싸는 최소 실행 단위. "컨테이너를 담는 봉투" |
| **Deployment** | Pod를 몇 개 띄울지, 어떤 이미지를 쓸지 정의하는 설정. "Pod 공장 설계도" |
| **Service** | Pod에 접근할 수 있게 네트워크를 열어주는 것. "Pod의 전화번호" |
| **Namespace** | 리소스를 논리적으로 그룹핑하는 폴더 같은 개념 |
| **Cluster** | 쿠버네티스가 돌아가는 서버 전체 묶음 |
| **Node** | 클러스터를 구성하는 개별 서버(머신) 1대 |

### 2-2. ArgoCD

쿠버네티스 전용 **GitOps 배포 도구**이다.

- Git 저장소를 감시하다가 변경이 생기면 → 클러스터에 자동 반영
- 웹 UI가 있어서 현재 배포 상태를 시각적으로 확인 가능
- 이 프로젝트에서는 ArgoCD가 `uh-heung-crew/study-gitops` 저장소를 감시하고 있음

### 2-3. Prometheus & Grafana (모니터링 스택)

| 도구 | 역할 |
|---|---|
| **Prometheus** | 클러스터의 메트릭(CPU, 메모리, 요청수 등)을 수집하는 모니터링 시스템 |
| **Grafana** | Prometheus가 수집한 데이터를 예쁜 대시보드로 보여주는 시각화 도구 |

### 2-4. Helm

쿠버네티스 앱의 **패키지 매니저**이다. (npm이나 brew 같은 것)
- 복잡한 YAML 파일 수십 개를 직접 작성하는 대신, `helm install` 한 방으로 설치
- 이 프로젝트에서는 `kube-prometheus-stack` Helm 차트를 ArgoCD를 통해 설치하고 있음

---

## 3. 프로젝트 디렉토리 구조

```
study-gitops/
├── root-application.yaml              # App of Apps 루트 (이것만 수동 적용)
├── apps/                              # 실제 배포할 앱들의 K8s 매니페스트
│   ├── nginx/
│   │   ├── namespace.yaml             # "nginx"라는 이름공간 생성
│   │   ├── deployment.yaml            # nginx 웹서버를 4개 복제본으로 배포
│   │   └── service.yaml               # nginx Pod에 접근할 수 있는 네트워크 서비스
│   └── spring-app/
│       ├── namespace.yaml             # "spring-app" 이름공간 생성
│       ├── deployment.yaml            # Spring Boot CRUD 앱 배포
│       ├── service.yaml               # 8080 포트 서비스
│       └── servicemonitor.yaml        # Prometheus가 JVM 메트릭을 수집하도록 설정
│
├── argocd/                            # ArgoCD Application 정의 (root-app이 자동 관리)
│   ├── nginx-application.yaml         # apps/nginx 폴더를 감시 → 자동 배포
│   ├── monitoring-application.yaml    # Prometheus+Grafana Helm 차트를 자동 배포
│   └── spring-app.yaml               # apps/spring-app 폴더를 감시 → 자동 배포
│
└── .gitignore
```

### App of Apps 패턴

이 프로젝트는 **App of Apps 패턴**을 사용한다.
`root-application.yaml` 하나만 수동으로 `kubectl apply`하면, ArgoCD가 `argocd/` 폴더를 감시하면서 안에 있는 Application들을 자동으로 생성/관리한다.

```
[root-app] ──감시──▶ argocd/
                      ├── monitoring-application.yaml ──▶ Prometheus+Grafana 설치
                      ├── nginx-application.yaml ──▶ apps/nginx/ 배포
                      └── spring-app.yaml ──▶ apps/spring-app/ 배포
```

새로운 앱을 추가하고 싶으면 `argocd/` 폴더에 Application yaml을 추가하고 push하면 끝.

### 각 파일이 하는 일

| 파일 | 하는 일 |
|---|---|
| `root-application.yaml` | ArgoCD에게 `argocd/` 폴더를 감시하라고 지시. **최초 1회만 수동 적용** |
| `apps/nginx/namespace.yaml` | `nginx`라는 Namespace 생성 |
| `apps/nginx/deployment.yaml` | nginx 컨테이너를 **4개 복제본(replicas)**으로 실행. 헬스체크(liveness/readiness probe) 포함 |
| `apps/nginx/service.yaml` | nginx Pod들에 80번 포트로 접근할 수 있는 ClusterIP 서비스 생성 |
| `apps/spring-app/deployment.yaml` | Spring Boot CRUD 앱을 2개 복제본으로 실행. Actuator 헬스체크 포함 |
| `apps/spring-app/service.yaml` | Spring 앱에 8080 포트로 접근할 수 있는 서비스 |
| `apps/spring-app/servicemonitor.yaml` | Prometheus가 `/actuator/prometheus` 엔드포인트에서 JVM 메트릭을 15초마다 수집 |
| `argocd/nginx-application.yaml` | ArgoCD가 이 Git 저장소의 `apps/nginx` 경로를 감시하고 자동 동기화 |
| `argocd/monitoring-application.yaml` | ArgoCD가 `kube-prometheus-stack` Helm 차트(v72.6.2)를 `monitoring` 네임스페이스에 자동 배포 |
| `argocd/spring-app.yaml` | ArgoCD가 이 Git 저장소의 `apps/spring-app` 경로를 감시하고 자동 동기화 |

---

## 4. 전체 동작 흐름

```
[개발자가 YAML 수정 후 Git Push]
         │
         ▼
[GitHub 저장소: uh-heung-crew/study-gitops]
         │
         ▼
[ArgoCD가 변경 감지]  ← ArgoCD는 K8s 클러스터 안에서 돌고 있음
         │
         ▼
[K8s 클러스터에 자동 반영]
  ├── nginx 4개 Pod 배포 (apps/nginx)
  └── Prometheus + Grafana 배포 (Helm 차트)
         │
         ▼
[Prometheus가 메트릭 수집 → Grafana 대시보드에서 확인]
```

**예시:** `deployment.yaml`에서 `replicas: 4`를 `replicas: 2`로 바꿔서 push하면 → ArgoCD가 감지 → 자동으로 Pod 2개를 줄여줌.

---

## 5. 맥북(로컬)에 필요한 것들

이 프로젝트를 로컬에서 실습하려면 아래 도구들이 필요하다.

### 5-1. 필수 설치 목록

| 도구 | 용도 | 설치 명령어 |
|---|---|---|
| **Docker Desktop** | 컨테이너 런타임 + 로컬 K8s 클러스터 제공 | [docker.com](https://www.docker.com/products/docker-desktop/)에서 다운로드 |
| **kubectl** | 쿠버네티스 CLI (클러스터에 명령을 보내는 도구) | `brew install kubectl` |
| **Helm** | K8s 패키지 매니저 | `brew install helm` |
| **ArgoCD CLI** | ArgoCD를 CLI로 조작 (선택사항, 웹UI로도 가능) | `brew install argocd` |
| **kind** | 로컬에 K8s 클러스터를 만드는 도구 (Docker 컨테이너 기반). 대안으로 k3d(경량 k3s 기반)도 있음 | `brew install kind` |

### 5-2. 현재 로컬 설치 상태 확인법

터미널에서 아래 명령어를 실행해보면 된다:

```bash
# 각 도구가 설치되어 있는지 확인
docker --version
kubectl version --client
helm version
argocd version --client
kind version
```

### 5-3. 처음부터 세팅하는 순서

---

#### 공통 (어떤 방식이든 무조건 필요)

도구 설치와 클러스터 생성은 수동이든 자동이든 반드시 해야 하는 기본 준비 단계입니다.

```bash
# 1. Docker Desktop 설치 → 실행
# 2. CLI 도구 설치
brew install kubectl helm kind argocd

# 3. 로컬 K8s 클러스터 생성 (최초 1회만)
kind create cluster
# → 실행하면 Docker Desktop의 Containers에 "kind-control-plane" 컨테이너가 생깁니다.
#   kind는 Docker 컨테이너 1개를 K8s 노드(서버) 1대처럼 사용하는 구조입니다.
#   실제 운영환경에서는 별도의 서버나 VM을 만들어서 K8s를 설치하지만,
#   kind는 이미 깔려있는 Docker 위에 컨테이너 하나만 띄워서 K8s 클러스터를 바로 만들어줍니다.
```

---

아래 A와 B는 **같은 결과를 내는 두 가지 방법**입니다.
학습할 때는 A(수동)를 먼저 해보고, B(자동/GitOps)로 전환하는 순서로 진행했습니다.

---

#### 방법 A: 수동 배포 (GitOps 없이 직접 명령어로)

kubectl과 helm 명령어를 직접 쳐서 배포하는 방식입니다.
"쿠버네티스가 어떻게 동작하는지" 체감하기 위한 학습 단계입니다.

```bash
# --- nginx 수동 배포 ---
kubectl create namespace apps
docker pull nginx:latest
kubectl create deployment nginx --image=nginx:latest -n apps
kubectl expose deployment nginx --type=ClusterIP --port=80 -n apps
kubectl get pods -n apps -w                          # Pod 뜨는지 확인
kubectl port-forward svc/nginx 8080:80 -n apps       # localhost:8080에서 확인

# --- 모니터링(Prometheus+Grafana) 수동 설치 ---
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
kubectl create namespace monitoring
helm install monitoring prometheus-community/kube-prometheus-stack -n monitoring
kubectl get pods -n monitoring                       # Pod 뜨는지 확인
kubectl port-forward svc/monitoring-grafana -n monitoring 3000:80
# 브라우저에서 localhost:3000 접속 (기본 계정: admin / prom-operator)
```

> **단점:** 매번 명령어를 쳐야 하고, 누가 뭘 바꿨는지 추적이 안 됩니다.

---

#### 방법 B: GitOps 자동 배포 (ArgoCD) ← 이 레포에서 사용하는 방식

Git 저장소의 YAML 파일을 수정하고 push하면 ArgoCD가 자동으로 클러스터에 반영하는 방식입니다.
**이 study-gitops 레포는 방법 B로 진행합니다.**

```bash
# --- Step 1: ArgoCD 설치 (최초 1회만) ---
kubectl create namespace argocd
kubectl apply -n argocd -f \
  https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml \
  --server-side

# --- Step 2: ArgoCD 웹 UI 접속 ---
kubectl port-forward svc/argocd-server -n argocd 9090:443
argocd admin initial-password -n argocd              # 초기 비밀번호 확인
# 브라우저에서 https://localhost:9090 접속 (admin / 위 비밀번호)

# --- Step 3: App of Apps 루트 등록 (이것만 하면 됨) ---
kubectl apply -f root-application.yaml
# → ArgoCD가 argocd/ 폴더를 감시 → 안에 있는 Application들이 자동 생성됨
# → nginx, monitoring, spring-app 전부 자동 배포
```

> **이후 흐름:** Git에서 YAML 수정 → push → ArgoCD가 감지 → 클러스터에 자동 반영
> ArgoCD 웹 UI에서 root, nginx, monitoring, spring-app이 Synced 상태인지 확인하면 됩니다.
> 새로운 앱을 추가하려면 `argocd/` 폴더에 Application yaml을 추가하고 push하면 끝.

---

## 6. 용어 요약 (치트시트)

| 용어 | 한 줄 설명 |
|---|---|
| **GitOps** | Git을 기준으로 인프라/배포를 관리하는 방법론 |
| **매니페스트(Manifest)** | K8s에게 "이걸 만들어줘"라고 알려주는 YAML 파일 |
| **Sync** | Git 상태와 클러스터 상태를 일치시키는 동작 |
| **Self-Heal** | 누가 수동으로 클러스터를 바꿔도 Git 상태로 자동 복구 |
| **Prune** | Git에서 삭제한 리소스를 클러스터에서도 자동 삭제 |
| **Replica** | 동일한 Pod를 몇 개 복제해서 실행할지의 수 |
| **Probe (Liveness/Readiness)** | Pod가 살아있는지(liveness), 트래픽 받을 준비가 됐는지(readiness) 체크 |
| **NodePort** | 클러스터 외부에서 접근할 수 있게 포트를 열어주는 서비스 타입 |
| **ClusterIP** | 클러스터 내부에서만 접근 가능한 서비스 타입 (기본값) |
| **App of Apps** | ArgoCD Application이 다른 Application들을 관리하는 패턴. 루트 1개만 수동 적용하면 나머지는 자동 |
| **ServiceMonitor** | Prometheus에게 "이 서비스의 메트릭을 수집해라"라고 알려주는 설정 |
| **Actuator** | Spring Boot에서 헬스체크, 메트릭 등 운영 정보를 HTTP로 노출하는 기능 |
| **Micrometer** | JVM 메트릭(메모리, GC, 스레드 등)을 Prometheus 형식으로 변환해주는 라이브러리 |
