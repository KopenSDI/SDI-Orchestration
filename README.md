# ☁️ KETI Orchestration 환경 설정 가이드

## 📋 목차

1. [리포지터리 구조](#-1-리포지터리-구조)
2. [환경 설정 및 버전](#-2-환경-설정-및-버전)
3. [컴포넌트 역할](#3-컴포넌트-역할)
4. [오케스트레이션 구성](#4-오케스트레이션-구성)
   1. [K3s 설치(필수)](#k3s-설치필수-)
   2. [K9s 설치(선택)](#k9s-설치선택-)
5. [컴포넌트 설치](#5-컴포넌트-설치)
   1. [프로파일링 컴포넌트](#프로파일링-컴포넌트-)
   2. [스케줄러 컴포넌트](#스케줄러-컴포넌트-)
   3. [오케스트레이션 엔진](#오케스트레이션-엔진-)
6. [로그 확인](#6-로그-확인-)
7. [기타 커맨드 레퍼런스](#7-기타-커맨드-레퍼런스)
8. [미션 구성](#8-미션-구성)
9. [참고 자료](#-9-참고-자료)

---

## 🗂️ 1. 리포지터리 구조

```text
SDI-Orchestration/
├── README.md                      # ← 현재 문서
├── orchestration-engines/         # Analysis/Policy‑Engine 배포 YAML 보관
│   └── orchestration-engines-deploy.yaml
├── profiling/                     # 메트릭 수집 · 적재 스택 매니페스트
│   └── tbot-monitoring.yaml       # 메트릭콜렉터 필요 모듈 및 · InfluxDB 포함
├── scheduler/                     # 커스텀 스케줄러 코드 & 배포 파일
│   ├── sdi-scheduler-deploy.yaml  # SDI‑Scheduler Deployment · RBAC
│__ └── check-sdi-scheduler.yaml   # 스케줄러 동작 검증용 워크로드

```

| 경로                                      | 설명                                                                                                               |
| --------------------------------------- | ---------------------------------------------------------------------------------------------------------------- |
| **orchestration-engines/**              | **analysis‑engine**·**policy‑engine** 두 Deployment 를 정의한다.                                                       |
| **profiling/tbot-monitoring.yaml**      | metric-collector + InfluxDB + metrics‑ingester Deployment 및 관련 Secret·Service 를 일괄 정의한다.                         |
| **scheduler/sdi‑scheduler‑deploy.yaml** | ServiceAccount·ClusterRole·Binding + Deployment 로 구성된 **SDI Scheduler** 매니페스트다. 터틀봇 배터리 및 위치정보를 기반하여 스케줄링을 진행한다. |
| **scheduler/sdi‑scheduler‑test.yaml**   | `schedulerName: sdi-scheduler` 스케줄러 동작 여부를 즉시 확인할 수 있는 간단한 워크로드 이다.                                              |

---

## 🛠️ 2. 환경 설정 및 버전

Control-plane(터틀봇 원격 PC) 의 주요 소프트웨어 및 버전 정보 Control-plane이 터틀봇 원격 PC 가 아닐경우, ROS는 관련 없는 점 참고바랍니다.

| 항목                    | 버전 / 세부 정보                 |
| --------------------- | -------------------------- |
| **ROS 2**             | ros2-jazzy                 |
| **Kernel**            | Linux 6.11.0-26-generic    |
| **Architecture**      | x86-64                     |
| **Operating System**  | Ubuntu 24.04.2 LTS         |
| **k3s**               | v1.32.5+k3s1               |
| **Container Runtime** | containerd://2.0.5-k3s1.32 |

## 3. 컴포넌트 역할

| 컴포넌트                    | 설명                                               |
| ----------------------- | ------------------------------------------------ |
| 🗓️ **sdi-scheduler**   | turtlebot 배터리(Wh) 및 위치정보 기반 커스텀 스케줄러             |
| 🔄 **metric-collector** | TurtleBot 및 시스템 메트릭을 Metric-Collector 큐에 적재하는 모듈 |
| 🗄️ **influxdb**        | 수집된 시계열 메트릭을 영구 저장하는 InfluxDB 데이터베이스             |
| 📥 **metrics-ingester** | Metric-Collector 큐에서 메트릭 메시지를 소비해 InfluxDB에 기록   |
| 🔎 **analysis-engine**  | 다양한 로그 및 메트릭 분석 엔진                               |
| ⚖️ **policy-engine**    | 분석 결과 기반으로 MALE 정책을 적용해 시스템 동작을 제어하는 엔진          |

## 4. 오케스트레이션 구성

### K3s 설치(필수)&#x20;

```bash
# Control‑Plane에서 실행
curl -sfL https://get.k3s.io | sh -
sudo cat /var/lib/rancher/k3s/server/node-token  # 토큰 복사

# Worker(Node/TurtleBot)에서 실행, CONTROL-PLANE_IP, CONTROL-PLANE_TOKEN 값 명령어에 넣기
curl -sfL https://get.k3s.io | K3S_URL=https://<CONTROL‑PLANE_IP>:6443 K3S_TOKEN=<CONTROL‑PLANE_TOKEN> sh -
```

### K9s 설치(선택)&#x20;

파드들의 정보를 손쉽게 확인하기 위해 KETI에서는 설치했습니다. 필수가 아닌 선택입니다.

```bash
mkdir k9s && cd k9s
wget https://github.com/derailed/k9s/releases/download/v0.26.7/k9s_Linux_x86_64.tar.gz
tar zxvf k9s_Linux_x86_64.tar.gz
sudo mv k9s /usr/local/bin/
cp /etc/rancher/k3s/k3s.yaml ~/.kube/config  # k9s에서 k3s 클러스터 조회 가능
```

`k9s` 명령어를 입력해 UI가 실행되면 정상 설치된 것입니다.

---

## 5. 컴포넌트 설치

### 프로파일링 컴포넌트&#x20;

```bash
git clone https://github.com/sungmin306/SDI-Orchestration.git
cd SDI-Orchestration/profiling/

# 주석 “직접 설정” 적힌 부분(12·13·21·22행) id,pw 설정
vi tbot-monitoring.yaml
```

#### 배포

```bash
kubectl apply -f tbot-monitoring.yaml
```

컴포넌트 상태는 다음과 같이 확인할 수 있습니다.

```bash
kubectl get pods -n tbot-monitoring # 또는 k9s
```

<img src="https://github.com/user-attachments/assets/47e675a8-86f8-44a7-be2f-a90d9e9d2f97" width="600" height="84"/>

### 스케줄러 컴포넌트&#x20;

#### `scheduler/sdi-scheduler-deploy.yaml`

- **ServiceAccount/RBAC** : `kube-system` 내부에서 PodBinding 권한만 최소 부여.
- **Deployment** : ENV 로 Influx Endpoint / Token 주입하며 `schedulerName: sdi-scheduler` 로 호출.

#### InfluxDB 토큰 확인 절차

브라우저에서 `http://<CONTROL‑PLANE_IP>:32086` 접속 이후 아래 이미지 대로 따라가시면 됩니다.

 <img src="https://github.com/user-attachments/assets/deab7d73-dbdc-4378-bd6b-adbfc63aedae" width="600" height="993"/>


#### 배포 파일 수정

```bash
cd ../../scheduler
vi sdi-scheduler-deploy.yaml   # 43행 주석에“직접 설정” 적혀있는 부분에 복사한 토큰 값 넣기
```

#### 배포

```bash
kubectl apply -f sdi-scheduler-deploy.yaml
```

상태 확인:

```bash
kubectl get pod -n kube-system # 또는 k9s
```

<img src="https://github.com/user-attachments/assets/ff5c0f17-8394-4487-932e-1e90a319f122" width="600" height="197"/>


스케줄링시 로그(로그 확인 방법 하단 기술)

<img src="https://github.com/user-attachments/assets/56a93d54-bfac-4b7a-ba47-7acfd95b2c1d" width="600" height="183"/>


#### 스케줄러 사용법

sdi-scheduler-test.yaml 파일 6번째줄 처럼 schedulerName: `schedulerName: sdi-scheduler`를 적고 사용하면됩니다.(주석 확인)

### 오케스트레이션 엔진&#x20;

#### `orchestration-engines/orchestration-engines-deploy.yaml`

`analysis-engine` 과 `policy-engine` 두 Deployment 가 포함되어있으며, 스케줄링 후 워크로드 분석과 MALE 기반 정책 설정하는 워크로드이다.

#### 배포

```bash
cd ../orchestration-engines
kubectl apply -f orchestration-engines-deploy.yaml

```

상태 확인:

```bash
kubectl get pod -n orchestration-engines   # 또는 k9s
```

<img src="https://github.com/user-attachments/assets/06315652-55db-4e3f-abb3-a73b3b4c914b" width="600" height="70"/>
---

analysis-engine로그(로그 확인 방법 하단 기술)

<img src="https://github.com/user-attachments/assets/783d2214-f865-4791-ba4a-de03d26dbc30" width="600" height="161"/>

policy-engine로그(로그 확인 방법 하단 기술)

<img src="https://github.com/user-attachments/assets/434c8d1a-6c3d-4836-ac3e-984078a40a0a" width="600" height="137"/>

## 6. 로그 확인&#x20;

> **Tip** 모든 컴포넌트가 Deployment 형태로 실행되므로 자동 생성된 파드 이름 추정할 수 없어 로그 확인 방법을 추가로 작성합니다.

| 방법      | 명령                                   |
| ------- | ------------------------------------ |
| kubectl | `kubectl logs -f <파드이름> -n <네임스페이스>` |
| k9s     | 파드 선택 후 **Space** → **L**            |

---

## 7. 기타 커맨드 레퍼런스

| 목적             | 명령                                                                   | 비고                |                    |
| -------------- | -------------------------------------------------------------------- | ----------------- | ------------------ |
| 리소스 적용         | `kubectl apply -f <file>`                                            | 선언적 관리            |                    |
| 리소스 삭제         | `kubectl delete -f <file>`                                           | —                 |                    |
| 네임스페이스 전환      | `kubectl config set-context --current --namespace=<ns>`              | 편의 설정             |                    |
| 스케줄러 검증        | \`kubectl describe pod <파드이름>                                             | grep Scheduler:\` | `sdi-scheduler` 확인 |

## 8. 워크로 구성 (2개)

### 1. YOLO Backbone Service (`yolo-backbone`)

- **역할**: TurtleBot에서 이미지를 추출하여 Backbone 모델(초기 컨볼루션 레이어)을 실행합니다. 추출된 특징 맵(feature map)을 Neck-Head 서비스로 전송합니다.
- **구성 파일**: `yolo-backbone-move.yaml`
- **주요 환경 변수**:
  - `TURTLEBOT_IMAGE_SOURCE_URL`: TurtleBot 이미지 스트림 URL
  - `PROCESS_URL`: Neck-Head 서비스의 입력 URL (예: `http://yolo-neck-head-service.default.svc.cluster.local:8080/process_feature`)
  - `UPDATE_PERIOD_SEC`: 이미지 추출 및 처리 주기(초)
- **아키텍처 제약**: 이 파드는 **ARM64** 환경에서만 동작하도록 설정되어 있습니다.
- **동작 방식**:
  1. `UPDATE_PERIOD_SEC` 주기에 따라 TurtleBot에서 이미지 GET
  2. Backbone 모델로 특징 맵(feature map) 추출
  3. 추출된 특징 맵을 JSON으로 직렬화하여 Neck-Head 서비스에 POST
  4. 처리 로그를 stdout으로 출력(쿠버네티스가 자동 수집)

### 2. YOLO Neck-Head Service (`yolo-neck-head`)

- **역할**: Backbone 서비스에서 전송된 특징 맵을 받아 Neck 레이어와 Head 레이어를 순차적으로 실행한 뒤, 최종 예측 결과(바운딩 박스가 그려진 이미지 또는 JSON)를 FastAPI 서버로 전달하여 사용자에게 이미지로 표시합니다.
- **구성 파일**: `yolo-neck-head.yaml`
- **주요 환경 변수**:
  - `BACKBONE_SERVICE_URL`: Backbone 서비스의 처리 URL (예: `http://yolo-backbone-service.default.svc.cluster.local:8080/process_feature`)
  - `FASTAPI_SERVER_URL`: 결과 이미지를 표시할 FastAPI 서버 URL (예: `http://fastapi-service.default.svc.cluster.local:8000/display`)
  - `SEND_INTERVAL_SEC`: 처리 주기(초)
- **동작 방식**:
  1. Backbone 서비스로부터 특징 맵 JSON GET
  2. Neck 모델을 통해 레이어 간 스케일 조정 및 합성
  3. Head 모델을 통해 객체 감지 실행(바운딩 박스 생성)
  4. 결과 이미지를 FastAPI 서버에 POST하여 웹 UI에 표시

> **모델 분할 안내**: 전체 YOLO 모델을 Backbone(초기 특징 추출)과 Neck-Head(후처리 및 객체 예측) 레이어로 분할함으로써 엣지 디바이스(TurtleBot)와 클러스터 환경 간에 연산 부하를 분산시킵니다.

각 워크로드별 YAML 파일을 참고하여 아래 명령으로 배포합니다:

```bash
# Mission 디렉토리 이동
cd ../Mission
kubectl apply -f fastapi_image_server.yaml
kubectl apply -f yolo-neck-head.yaml
kubectl apply -f yolo-backbone-move.yaml
```

## 📚 9. 참고 자료

- Kubernetes 공식: [https://kubernetes.io/docs/](https://kubernetes.io/docs/)
- InfluxDB v2 Docs: [https://docs.influxdata.com/influxdb/v2/](https://docs.influxdata.com/influxdb/v2/)
- Scheduler Framework: [https://kubernetes.io/docs/concepts/scheduling-eviction/scheduling-framework/](https://kubernetes.io/docs/concepts/scheduling-eviction/scheduling-framework/)



## 설치 완료 후 터틀봇 설치 진행
- 터틀봇 필수 프로세스 설치: [https://github.com/sungmin306/SDI-Turtlebot-Setting](https://github.com/sungmin306/SDI-Turtlebot-Setting)
