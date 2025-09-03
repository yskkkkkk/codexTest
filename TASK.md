# TASKS for AI Codegen — K8s 첫 경험 프로젝트

## 컨텍스트
- 목적: 쿠버네티스를 처음 경험하는 개발자가, 로컬에서 컨테이너ized 샘플 앱을 **빌드 → 배포 → 노출 → 모니터링 → 롤백**까지 전 과정을 체험한다.
- 철학: "작게, 빨리, 안전하게" — 단일 서비스부터 시작, 단계별로 가시적인 성공 기준을 둔다.
- 표준 도구: Docker, kubectl, kind(기본) / (옵션) minikube

## 제약
- OS: Windows 10/11 또는 macOS (로컬 개발자 환경 가정)
- k8s: 로컬 단일 노드(클러스터) — 고가용성/보안 하드닝은 범위 밖
- 앱: HTTP 기반 간단한 REST API(헬스체크, Echo 엔드포인트 포함), 언어는 자유(예: Java/Spring 또는 Node/Express)

## 산출물(파일/디렉터리)
- `app/` 샘플 애플리케이션 소스 + `Dockerfile`
- `k8s/base/` 매니페스트: `namespace.yaml`, `deployment.yaml`, `service.yaml`, `configmap.yaml`, `secret.example.yaml`
- `k8s/ingress/` `ingress.yaml`(옵션: NGINX Ingress Controller)
- `k8s/overlays/dev/`(옵션: Kustomize) 또는 `helm/`(옵션: Helm chart)
- `scripts/` 자동화 스크립트: `kind-up.sh`, `kind-down.sh`, `kubectl-wait-ready.sh`, `port-forward.sh` 등
- `Makefile` 표준 타깃: `make up`, `make down`, `make build`, `make push`, `make deploy`, `make logs`, `make test`
- `README.md`, `TASKS.md`(본 문서), `OPERATIONS_NOTES.md`(문제해결/롤백 가이드)

## 품질 규칙(정량)
- 모든 스크립트는 **idempotent**: 재실행해도 장애 없음
- `Deployment`는 **liveness/readiness** 프로브 설정 필수, `resources.requests/limits` 명시
- `make up` 실행 후 **120초 이내** 모든 파드 `Ready=1/1`
- `make test` 시 `HTTP 200` 및 예시 응답 확인
- 모든 매니페스트는 `kubectl apply --server-side` 또는 `apply -f`로 적용 가능, `kubectl delete`로 깨끗이 제거 가능
- 시나리오 롤백: **1분 이내** 이전 버전으로 `kubectl rollout undo` 성공

---

# STEP BY STEP

## Step 0. 로컬 도구 설치 및 버전 고정
### 해야 할 일
1) Docker Desktop 설치 확인, 2) kubectl 설치, 3) kind 설치, 4) 버전 출력 스크립트 `scripts/print-versions.sh` 생성.  
Windows는 Git Bash 또는 WSL 환경 가정.

### 산출물
- `scripts/print-versions.sh` : `docker version`, `kubectl version --client`, `kind version` 출력
- `README.md`에 요구 버전 범위 명시

### 완료 기준
- `bash scripts/print-versions.sh` 결과가 정상 출력

---

## Step 1. 샘플 앱 골격 + Dockerfile
### 해야 할 일
- `app/`에 단일 HTTP 서버 구현(예: `/healthz`, `/echo?msg=`).  
- `Dockerfile`은 멀티스테이지 빌드로 **이미지 크기 최소화**, 포트 `8080` 노출.

### 산출물
- `app/main.(java|js|ts|go 등)`, `Dockerfile`
- `Makefile`: `make build` → `docker build -t k8s101/app:0.1 .`

### 완료 기준
- `docker run -p 8080:8080 k8s101/app:0.1` 후 `curl localhost:8080/healthz` 가 `200 OK`

---

## Step 2. 로컬 레지스트리와 kind 클러스터
### 해야 할 일
- kind용 로컬 레지스트리(예: `registry:5000`) 컨테이너 구성, kind 클러스터에서 해당 레지스트리를 신뢰하도록 설정.
- `scripts/kind-up.sh`는 (1) 레지스트리 생성, (2) kind 클러스터 생성, (3) 레지스트리와 네트워크 연결을 자동화.

### 산출물
- `scripts/kind-up.sh`, `scripts/kind-down.sh` (삭제/정리)
- `kind-config.yaml` (containerd mirror/registry 설정 포함)
- `Makefile`: `make up`, `make down`

### 완료 기준
- `kubectl get nodes`에 `Ready` 노드 1개 이상
- `docker tag k8s101/app:0.1 localhost:5000/k8s101/app:0.1 && docker push localhost:5000/k8s101/app:0.1` 성공
- (옵션) minikube 대체 시 `minikube start` 및 `eval $(minikube docker-env)` 설명 추가

---

## Step 3. K8s 기본 객체(네임스페이스/서비스/디플로이)
### 해야 할 일
- `k8s/base/namespace.yaml`(예: `ns: demo`)
- `k8s/base/deployment.yaml`: 레플리카 2, 컨테이너 이미지 `localhost:5000/k8s101/app:0.1`
- `k8s/base/service.yaml`: `ClusterIP`, 포트 80 → 타겟 8080

### 산출물
- `k8s/base/*.yaml`
- `Makefile`: `make deploy` → `kubectl apply -n demo -f k8s/base`

### 완료 기준
- `kubectl -n demo get deploy,svc,pod`에서 파드 2개 모두 `Running` + `Ready=1/1`

---

## Step 4. 프로브/리소스/롤링업데이트 전략
### 해야 할 일
- `deployment.yaml`에 `readinessProbe`(HTTP GET `/healthz`, initialDelay 5s, period 5s), `livenessProbe` 추가.
- `resources`(requests/limits) 작성: 예) CPU `100m/300m`, 메모리 `128Mi/256Mi`
- `strategy`는 `RollingUpdate`로 설정(최대 surged 25%, 최대 unavailable 0~25% 범위)

### 완료 기준
- `kubectl -n demo rollout status deploy/<name>` 성공
- `kubectl describe pod`에 프로브 설정 확인

---

## Step 5. 설정 주입(ConfigMap/Secret)
### 해야 할 일
- `configmap.yaml`에 `APP_MESSAGE="hello-k8s"` 등 환경 변수
- `secret.example.yaml` 제공(민감값은 Base64 인코딩 예시만, 실제 파일은 커밋 금지)
- `deployment.yaml`에 `envFrom`으로 주입

### 완료 기준
- `kubectl -n demo exec deploy/<name> -- printenv | grep APP_MESSAGE` 로 값 확인

---

## Step 6. 접근 노출(Port-Forward → Ingress)
### 해야 할 일
- 초보자는 먼저 `scripts/port-forward.sh`로 `Service` 80 → 로컬 8081 포워딩.  
- 이후 Ingress Controller(NGINX) 설치(옵션), `k8s/ingress/ingress.yaml`로 `host: app.localtest.me` 라우팅.

### 완료 기준
- `curl http://localhost:8081/echo?msg=hi` 가 200
- (옵션 Ingress) `curl http://app.localtest.me/healthz` 가 200

---

## Step 7. 관측성(로그/메트릭)과 헬스체크 스크립트
### 해야 할 일
- `make logs`로 `kubectl -n demo logs deploy/<name> -f`
- 간단한 `scripts/kubectl-wait-ready.sh`로 `Ready` 대기
- (옵션) `/metrics` 엔드포인트를 앱에 추가하고, 추후 Prometheus로 확장 가능하도록 주석

### 완료 기준
- `make logs` 실행 시 애플리케이션 로그 스트림 확인
- `scripts/kubectl-wait-ready.sh demo deploy <name> 120` 내 완료

---

## Step 8. 오토스케일(HPA) 체험(옵션)
### 해야 할 일
- `kubectl -n demo autoscale deploy/<name> --cpu-percent=50 --min=2 --max=5`
- 부하 생성(예: `hey`/`ab`), 파드 수 증가 확인

### 완료 기준
- `kubectl -n demo get hpa`에서 타깃 리소스와 현재/목표 CPU 비율 확인
- 부하 중 파드가 자동으로 스케일 아웃되는 것을 관찰

---

## Step 9. 배포 전략/롤백 시나리오
### 해야 할 일
- 앱 버전 `0.2` 생성(간단 수정), 이미지 푸시, `Deployment` 이미지 태그 업데이트
- 배포 후 오류 유도(예: 헬스체크 실패), `kubectl -n demo rollout undo deploy/<name>` 로 즉시 롤백

### 완료 기준
- `kubectl -n demo rollout history deploy/<name>`에 `REVISION` 목록 존재
- `undo` 이후 `/healthz` 정상

---

## Step 10. 정리와 청소
### 해야 할 일
- `make down` 으로 리소스 삭제, `scripts/kind-down.sh`로 클러스터/레지스트리 제거
- `OPERATIONS_NOTES.md`에 문제해결 TIPS 기록(파드 CrashLoopBackOff, ImagePullBackOff, Probe 실패, Events 조회 등)

### 완료 기준
- `kubectl config get-contexts`에서 kind 컨텍스트 제거 또는 비활성
- `docker ps -a`에 잔여 컨테이너/네트워크 없음

---

# Makefile 타깃 예시(명세)
- `make build` : `docker build -t k8s101/app:$(VER) app/`
- `make push`  : `docker tag k8s101/app:$(VER) localhost:5000/k8s101/app:$(VER) && docker push localhost:5000/k8s101/app:$(VER)`
- `make up`    : `scripts/kind-up.sh && scripts/print-versions.sh`
- `make deploy`: `kubectl apply -f k8s/base -n demo && scripts/kubectl-wait-ready.sh demo deploy app 120`
- `make logs`  : `kubectl -n demo logs deploy/app -f`
- `make test`  : `curl -sf http://localhost:8081/healthz || (scripts/port-forward.sh & sleep 2; curl -sf http://localhost:8081/healthz)`
- `make down`  : `kubectl delete ns demo --ignore-not-found=true && scripts/kind-down.sh`

# 실패 지점 디버깅 체크리스트
1) 파드가 안 뜬다 → `kubectl -n demo get events --sort-by=.lastTimestamp`, `describe pod`, 이미지 태그/레지스트리 경로 확인  
2) Probe 실패 → 컨테이너 실제 포트, path, 초기 지연 시간 조정  
3) CrashLoopBackOff → 컨테이너 엔트리포인트/환경 변수/권한  
4) ImagePullBackOff → 로컬 레지스트리 주소/태그 푸시 여부, kind의 registry mirror 설정  
5) Service/Ingress 접근 불가 → Service Port/TargetPort, Ingress host, DNS(localtest.me) 확인

# 참고(선택)
- Kustomize 또는 Helm 중 **하나**만 채택(초보는 Kustomize가 단순)
- 다음 단계: Prometheus/Grafana, Loki, NetworkPolicy, RBAC, PodSecurity 등 보안/운영 심화
