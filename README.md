# Factory Tycoon K8s

Factory Tycoon 프로젝트의 Kubernetes 배포 매니페스트 및 Helm 차트

## 기술 스택

- Kubernetes
- Helm 3
- ArgoCD (GitOps)
- AWS ALB Ingress Controller
- ECR (Container Registry)

## 프로젝트 구조

```
├── helm/
│   ├── factory-tycoon-backend/     # 메인 백엔드 Helm 차트
│   ├── factory-tycoon-aws/         # AWS 백엔드 Helm 차트
│   └── factory-tycoon-websocket/   # WebSocket 서버 Helm 차트
├── be-factory/         # 메인 백엔드 K8s 매니페스트
├── be-aws/             # AWS 백엔드 K8s 매니페스트
├── be-websocket/       # WebSocket 서버 K8s 매니페스트
└── ingress/            # ALB Ingress 설정
```

## 배포 아키텍처

### Helm 차트 (권장)
- ArgoCD가 자동으로 감지 및 배포
- 환경별 values 파일 (`values-dev.yaml`, `values-prod.yaml`)
- ConfigMap/Secret은 Terraform으로 관리

### 매니페스트 (레거시)
- 직접 kubectl 배포용
- `be-factory/`, `be-aws/`, `be-websocket/`

## 배포 방법

### ArgoCD 자동 배포 (GitOps)

```bash
# ArgoCD가 자동으로 감지
# 1. GitHub에 커밋 푸시
# 2. ArgoCD가 변경사항 감지
# 3. 클러스터에 자동 배포
```

### 수동 배포 (Helm)

```bash
# 환경별 배포
$ helm upgrade --install factory-backend \
  ./helm/factory-tycoon-backend \
  -f ./helm/factory-tycoon-backend/values-prod.yaml \
  -n default

$ helm upgrade --install factory-aws \
  ./helm/factory-tycoon-aws \
  -f ./helm/factory-tycoon-aws/values-prod.yaml \
  -n default

$ helm upgrade --install factory-websocket \
  ./helm/factory-tycoon-websocket \
  -f ./helm/factory-tycoon-websocket/values-prod.yaml \
  -n default
```

### 수동 배포 (kubectl)

```bash
# 매니페스트 직접 배포
$ kubectl apply -f be-factory/
$ kubectl apply -f be-aws/
$ kubectl apply -f be-websocket/
$ kubectl apply -f ingress/
```

## 서비스 구성

### Backend (sf-backend)
- **포트**: 8080
- **레플리카**: 2
- **경로**: `/api`
- **이미지**: ECR `sf-backend`

### AWS Backend (sf-backend-aws)
- **포트**: 8081
- **레플리카**: 2
- **경로**: `/aws`
- **이미지**: ECR `sf-backend-aws`

### WebSocket (sf-backend-websocket)
- **포트**: 8000
- **레플리카**: 2
- **경로**: `/ws`
- **이미지**: ECR `sf-backend-websocket`

## Ingress

AWS ALB를 통한 라우팅:
- `/api/*` → Backend (8080)
- `/aws/*` → AWS Backend (8081)
- `/ws/*` → WebSocket (8000)

## 환경 변수

ConfigMap과 Secret은 Terraform이 관리 (`factory-tycoon-cloud/argocd/`):
- `sf-backend-config` - 일반 설정
- `sf-backend-secrets` - 민감 정보 (DB, JWT, Redis)

## 배포 확인

```bash
# Pod 상태 확인
$ kubectl get pods -n default

# 서비스 확인
$ kubectl get svc -n default

# Ingress 확인
$ kubectl get ingress -n default

# 로그 확인
$ kubectl logs -f deployment/sf-backend -n default
```

## 롤백

```bash
# Helm 롤백
$ helm rollback factory-backend -n default

# ArgoCD에서 이전 커밋으로 되돌리기
# values-prod.yaml의 image.tag를 이전 커밋 해시로 변경
```

## Node Selector

모든 백엔드 서비스는 `role: backend` 레이블이 있는 노드에 배포됩니다.
