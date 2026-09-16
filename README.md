# ex6-eks: EKS + ArgoCD 기반 GitOps CI/CD 파이프라인

## 개요
기존 ex1~ex5(EC2/S3/SSM/Docker/ASG 기반 CI/CD)에서 한 단계 더 나아가,
**EKS(Kubernetes) + ArgoCD를 이용한 GitOps 방식**으로 배포 파이프라인을 구성한 실습입니다.

GitHub Actions가 애플리케이션을 배포하는 것이 아니라,
**"Git 저장소의 상태를 갱신"**하고 ArgoCD가 그 변경을 감지해 클러스터에 반영하는
GitOps의 핵심 원칙을 구현합니다.

## 아키텍처 흐름
개발자 git push (main)
│
▼
GitHub Actions (.github/workflows/eks.yml)

소스 체크아웃
Docker Hub 로그인
Dockerfile 기반 이미지 빌드 & push (christian-shim/nginx:{git-sha})
sed로 k8s/deployment.yaml의 image 태그를 새 커밋 해시로 갱신
변경된 deployment.yaml을 git commit & push
│
▼
ArgoCD (EKS 클러스터 내 설치)
이 저장소(k8s/ 경로)를 지속적으로 감시(polling/webhook)
deployment.yaml 변경 감지 시 자동 동기화(sync)
│
▼
EKS 클러스터에 새 이미지로 Pod 재배포
Service(LoadBalancer, AWS ELB)를 통해 외부 접속

## 사전 인프라 (1회성 구축)
- EKS 클러스터
- AWS Load Balancer Controller
  - IAM 정책(`std12-AWSLoadBalancerControllerIAMPolicy`) 생성
  - OIDC 공급자 연동 (`eksctl utils associate-iam-oidc-provider`)
  - IRSA 설정 (`eksctl create iamserviceaccount`)
  - Helm으로 `kube-system`에 배포
- ArgoCD
  - `argocd` 네임스페이스에 공식 매니페스트로 설치
  - `argocd-server` 서비스를 `LoadBalancer` 타입으로 변경해 외부 접속 구성
  - HTTP 접속을 위해 `--insecure` 옵션 적용 (L4 LB 환경이라 TLS 종료를 서버가 직접 처리)

## 폴더 구조
ex6-eks/
├── .github/workflows/
│ └── eks.yml # CI/CD 파이프라인 정의
├── k8s/
│ └── deployment.yaml # Deployment + Service 매니페스트 (ArgoCD가 감시하는 대상)
├── nginx/
│ ├── Dockerfile # nginx 기반 커스텀 이미지 빌드 정의
│ └── html/ # 정적 웹 콘텐츠
└── README.md

## 필요한 GitHub Secrets
| Secret | 용도 |
|---|---|
| `DOCKER_USER` | Docker Hub 로그인 계정 |
| `DOCKER_PASSWORD` | Docker Hub 액세스 토큰 |

## 실행 방법
1. `main` 브랜치에 push하면 자동으로 파이프라인 실행
2. Actions 탭에서 빌드/푸시/커밋 단계 성공 여부 확인
3. ArgoCD UI 또는 CLI에서 Application sync 상태 확인

## 진행 상태 (TODO)
- [x] AWS Load Balancer Controller 설치
- [x] ArgoCD 설치 및 외부 접속 구성
- [x] Dockerfile / deployment.yaml / eks.yml 작성 및 검증
- [ ] ArgoCD Application 매니페스트 등록 (이 저장소 `k8s/` 경로를 감시하도록 연결)
- [ ] 실제 sync 및 배포 검증