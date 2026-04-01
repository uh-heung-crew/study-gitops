# study-gitops

> ArgoCD 기반 GitOps로 Kubernetes 애플리케이션을 자동 배포하는 학습 프로젝트

Git 저장소의 YAML을 수정하고 Push하면, ArgoCD가 자동으로 클러스터에 반영합니다.

## 프로젝트 구조

```
study-gitops/
├── root-application.yaml          # App of Apps 루트 (최초 1회 수동 적용)
├── apps/
│   ├── nginx/                     # nginx 웹서버 (replicas: 8)
│   ├── spring-app/                # Spring Boot Article CRUD API
│   └── monitoring/                # kube-prometheus-stack 로컬 Helm 차트
├── argocd/                        # ArgoCD Application 정의
│   ├── nginx-application.yaml
│   ├── spring-app.yaml
│   └── monitoring-application.yaml
└── .gitignore
```

## 배포 흐름

```
root-application.yaml (수동 1회)
    └── argocd/ 폴더 감시
         ├── nginx-application.yaml      → apps/nginx/ 배포
         ├── spring-app.yaml             → apps/spring-app/ 배포
         └── monitoring-application.yaml → apps/monitoring/ 배포 (Prometheus + Grafana)
```

> 새로운 앱을 추가하려면 `argocd/` 폴더에 Application YAML을 추가하고 Push하면 끝입니다.

## Spring Boot Article API

별도 레포 [article-sample](https://github.com/uh-heung-crew/article-sample)에서 관리합니다.

| 항목 | 설명 |
|---|---|
| API | CRUD + 검색 (`/api/articles`) |
| 메트릭 | Actuator + Micrometer (`/actuator/prometheus`) |
| CI/CD | GitHub Actions → Docker Hub Push → 이 레포의 `deployment.yaml`에서 참조 |

## 모니터링

`apps/monitoring/`에서 kube-prometheus-stack을 로컬 Helm 차트로 커스텀합니다.

| 구성 요소 | 설정 |
|---|---|
| Prometheus | 모든 네임스페이스의 ServiceMonitor 수집 |
| Grafana | JVM (Micrometer), Spring Boot Statistics 대시보드 자동 등록 |
| ServiceMonitor | Spring 앱의 `/actuator/prometheus`를 15초마다 수집 |

## 시작하기

### 필수 설치 목록

| 도구 | 용도 | 설치 명령어 |
|---|---|---|
| **Docker Desktop** | 컨테이너 런타임 + 로컬 K8s 클러스터 제공 | [docker.com](https://www.docker.com/products/docker-desktop/)에서 다운로드 |
| **kubectl** | 쿠버네티스 CLI (클러스터에 명령을 보내는 도구) | `brew install kubectl` |
| **Helm** | K8s 패키지 매니저 | `brew install helm` |
| **ArgoCD CLI** | ArgoCD를 CLI로 조작 (선택사항, 웹UI로도 가능) | `brew install argocd` |
| **kind** | 로컬에 K8s 클러스터를 만드는 도구 (Docker 컨테이너 기반) | `brew install kind` |

### 설치 확인

```bash
docker --version
kubectl version --client
helm version
argocd version --client
kind version
```

### 클러스터 생성

```bash
brew install kubectl helm kind argocd
kind create cluster
```

### ArgoCD 설치 및 배포

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

## 접속 방법

| 서비스 | 명령어 | URL |
|---|---|---|
| ArgoCD | `kubectl port-forward svc/argocd-server -n argocd 9090:443` | https://localhost:9090 |
| Grafana | `kubectl port-forward svc/monitoring-grafana -n monitoring 3000:80` | http://localhost:3000 |
| Spring App | `kubectl port-forward svc/spring-app -n spring-app 8080:8080` | http://localhost:8080 |

## 기술 스택

| 도구 | 역할 |
|---|---|
| ArgoCD | GitOps 자동 배포 |
| Prometheus | 메트릭 수집 |
| Grafana | 대시보드 시각화 |
| kube-prometheus-stack | 모니터링 Helm 차트 |
| kind | 로컬 K8s 클러스터 |
| Spring Boot Actuator + Micrometer | JVM 메트릭 노출 |

---

## 참고 자료

<details>
<summary><b>핵심 개념 정리</b></summary>

### Kubernetes (K8s)

컨테이너를 **자동으로 배포, 관리, 확장**해주는 플랫폼

| 개념 | 설명 |
|---|---|
| **Pod** | 컨테이너를 감싸는 최소 실행 단위 |
| **Deployment** | Pod를 몇 개 띄울지, 어떤 이미지를 쓸지 정의하는 설정 |
| **Service** | Pod에 접근할 수 있게 네트워크를 열어주는 것 |
| **Namespace** | 리소스를 논리적으로 격리하는 공간 |
| **Cluster** | K8s가 돌아가는 서버 전체 묶음 |
| **Node** | 클러스터를 구성하는 개별 서버 1대 |

### ArgoCD

K8s 전용 **GitOps 배포 도구**

- Git 저장소를 감시하다가 변경이 생기면 클러스터에 자동 반영
- 웹 UI에서 배포 상태를 시각적으로 확인 가능

### Prometheus & Grafana

| 도구 | 역할 |
|---|---|
| **Prometheus** | 메트릭(CPU, 메모리, 요청수 등)을 수집하는 모니터링 시스템 |
| **Grafana** | 수집한 데이터를 대시보드로 시각화하는 도구 |

### Helm

K8s의 **패키지 매니저** (npm, brew 같은 것)
- `helm install` 한 줄로 복잡한 K8s 리소스를 한번에 설치
- 이 프로젝트에서는 `kube-prometheus-stack` 차트를 사용

</details>

<details>
<summary><b>수동 배포 vs GitOps 배포 비교</b></summary>

### 방법 A: 수동 배포

```bash
# --- nginx 수동 배포 ---
kubectl create namespace apps
docker pull nginx:latest
kubectl create deployment nginx --image=nginx:latest -n apps
kubectl expose deployment nginx --type=ClusterIP --port=80 -n apps
kubectl get pods -n apps -w
kubectl port-forward svc/nginx 8080:80 -n apps

# --- 모니터링(Prometheus+Grafana) 수동 설치 ---
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
kubectl create namespace monitoring
helm install monitoring prometheus-community/kube-prometheus-stack -n monitoring
kubectl get pods -n monitoring
kubectl port-forward svc/monitoring-grafana -n monitoring 3000:80
# 브라우저에서 localhost:3000 접속 (기본 계정: admin / prom-operator)
```

> 단점: 매번 명령어를 쳐야 하고, 누가 뭘 바꿨는지 추적이 안 됨

### 방법 B: GitOps 자동 배포 (이 레포 방식)

```bash
kubectl apply -f root-application.yaml
# → 이후로는 Git Push만으로 자동 배포
```

> YAML 수정 → Push → ArgoCD가 감지 → 클러스터에 자동 반영

</details>

<details>
<summary><b>용어 사전</b></summary>

| 용어 | 설명 |
|---|---|
| **GitOps** | Git을 기준으로 인프라/배포를 관리하는 방법론 |
| **매니페스트** | K8s에게 "이걸 만들어줘"라고 알려주는 YAML 파일 |
| **Sync** | Git 상태와 클러스터 상태를 일치시키는 동작 |
| **Self-Heal** | 수동 변경이 발생해도 Git 상태로 자동 복구 |
| **Prune** | Git에서 삭제한 리소스를 클러스터에서도 자동 삭제 |
| **Replica** | 동일한 Pod를 몇 개 복제할지의 수 |
| **Probe** | Pod가 살아있는지(Liveness), 트래픽 받을 준비가 됐는지(Readiness) 체크 |
| **NodePort** | 클러스터 외부에서 접근 가능한 서비스 타입 |
| **ClusterIP** | 클러스터 내부에서만 접근 가능한 서비스 타입 (기본값) |
| **App of Apps** | ArgoCD Application이 다른 Application들을 관리하는 패턴 |
| **ServiceMonitor** | Prometheus에게 메트릭 수집 대상을 알려주는 설정 |
| **Actuator** | Spring Boot에서 운영 정보를 HTTP로 노출하는 기능 |
| **Micrometer** | JVM 메트릭을 Prometheus 형식으로 변환해주는 라이브러리 |

</details>
