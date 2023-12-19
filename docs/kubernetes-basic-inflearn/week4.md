# 1. 쿠버네티스 API에 접근하기

- 마스터 노드에 있는 쿠버네티스 서버 API를 통해 쿠버네티스의 자원을 생성하거나, 조회할 수 있다.
- 쿠버네티스 API에 외부에서 접근하기 위해서는 인증서가 필요하다.
  - 외부 PC에서 kubectl 명령을 통해 config 기능을 활용하여 접근 가능한 클러스터에 연결
    - kubectl은 마스터 노드 외부에 설치해서 여러 클러스터를 관리하는 것도 가능함
  - 내부 관리자가 프록시를 열어줬다면 인증서 없이도 접근이 가능함
- 구분
  - User Account : 유저들이 API 서버에 접근하는 방법
  - Service Account : 파드들이 API 서버에 접근하는 방법

## 1.1. Authentication
- 쿠버네티스의 api서버로 접근하기 위한 인증 방법
- X509 Certs, kubectl, ServiceAccount

### 1.1.1. X509 Certs
- X509 Certs는 공개 키 인증서를 정의하는 ITU-T의 국제 표준인증서를 갖고 쿠버네티스 API에 접근하는 방식임
- 클러스터는 kube-config 파일안에 인증 정보들을 보관함
  - 인증서의 종류
    - CA crt (발급기관 인증서)
    - Client csr (클라이언트 인증요청서)
    - Client crt (클라이언트 인증서)
  - 클라이언트 키와 인증서 정보를 복사하면 https 요청을 통해서 접근할 수 있음
  - kubectl 명령도 kube-config 파일을 복사하여 사용하기 때문에 클러스터의 리소스들을 조회할 수 있는 것
- 외부에서 http로 접근 가능하게 하려면 kubectl 의 accept-hosts 옵션을 통해 특정 포트로 프록시를 열어두면 됨
  - kubectl이 인증서를 가지고 있기 때문에 사용자는 아무런 인증 없이 접근할 수 있음

## 1.1.2. 외부 서버에 kubectl을 설치해서 멀티 클러스터에 접근하기
- 사전에 각 클러스터에 있는 kube-config 파일이 외부 서버의 kubectl에 있어야 한다.
- contexts에 클러스터와 유저 정보를 설정
- clusters 항목으로 클러스터 등록(name, url, CA crt)
- users 항목으로 사용자 등록(name, Client crt, Client key)
- kubectl config 명령을 통해 현재 사용하고 싶은 컨텍스트를 설정할 수 있다.
  - `kubectl config user-context context-A`
  - clusterA에 연결하여 노드 정보 조회

### 1.1.3. Service Account
- 네임스페이스를 생성하면 기본적으로 default 라는 이름의 서비스 어카운트가 생성된다.
- 해당 서비스 어카운트에는 Secret이 하나 있고, CA crt 정보와 토큰값이 있다.
- 파드를 생성하면 이 서비스 어카운트와 연결되고, 파드는 이 토큰값을 통해서 API 서버에 연결할 수 있다.
  - 해당 토큰값만 알면 외부 사용자도 이 방식으로 쿠버네티스 API 서버에 접근할 수 있음

## 1.2. Authorization
쿠버네티스 자원의 종류
- 클러스터 단위
  - 노드, PV, 네임스페이스
- 네임스페이스 단위
  - 파드, 서비스
  - ServiceAccount
    - 자동으로 만들어지기도 하고, 추가할 수도 있음

자원에 대한 권한을 지정하는 방법
- RBAC (가장 많이 사용함)
- ABAC
- Webhook
- Node

### 1.2.1. RBAC
- 역할 기반으로 권한을 부여하는 방법
  - Role-Based Access Control : 사용자에게 부여되는 권한을 역할에 기반하여 관리하는 접근 제어 모델
  - RBAC의 구성 요소
    - 역할(Role): 일련의 관련된 권한 집합으로 정의되는 사용자의 역할. 예) "관리자", "사용자", "게스트" 등
    - 권한(Permission): 특정 작업이나 리소스에 대한 접근 권한. 각 역할은 하나 이상의 권한을 가지며, 사용자가 특정 역할을 할당받으면 해당 권한도 부여됨
    - 사용자(User): 시스템에 접근하는 개별 사용자. 사용자는 하나 이상의 역할을 가질 수 있음.
    - 할당(Assignment): 사용자에 대한 특정 역할이나 권한을 부여하는 프로세스.
- 쿠버네티스에서는 서비스 어카운트에 Role과 Role Binding을 설정하여 네임스페이스 자원이나 클러스터 자원에 대한 접근권한을 부여함.
- 네임스페이스 단위의 자원에 접근하기
  - Role, RoleBinding을 이용하여 네임 스페이스 자원에 접근하는 권한을 관리한다
  - Role
    - 네임 스페이스 내에 있는 자원에 대하여 조회, 생성 등의 권한
    - 여러 개 생성 가능
    - Role을 통해 각 자원에 대한 권한을 설정할 수 있음(생성 및 조회 등)
    - Role은 하나만 지정할 수 있고, 여러개의 ServiceAccount가 동일한 role을 가질 수 있음
  - Role Binding
    - Role과 서비스 어카운트를 연결해주는 역할
    - Role Binding과 Role은 1:1 관계고, Role Binding과 서비스 어카운트는 1:N 관계임
- 클러스터 단위의 자원에 접근하기
  - 클러스터 Role, 클러스터 Role Binding을 이용하여 클러스터 자원에 접근하는 권한을 관리한다
  - 모든 네임스페이스에 같은 권한을 만들어서 관리해야 할 때 유용함
    - 네임스페이스마다 동일한 Role 을 만들어서 관리하게 되면 Role에 대한 변경이 필요할 때 모든 Role을 수정해야 함
    - 클러스터 Role 하나를 만들어서 모든 네임스페이스에 있는 Role Binding을 클러스터 Role을 지정하게 된다면 권한에 대한 변경사항이 있을 때 하나만 수정하면 된다

# 2. 컨트롤러
## 2.1. StatefulSet
### 2.1.1. 어플리케이션의 종류

- Stateless Application
  - App이 여러 개 배포되더라도 모두 같은 서비스 역할을 함 (단순 복제)
  - App이 죽으면 같은 서비스 역할을 하는 App을 복제해주면 된다. (이름은 중요하지 않음.)
  - 볼륨 하나에 모든 서비스의 로그 저장 가능
  - 사용자들의 접속 트래픽은 여러 App에 분산되는 형태가 일반적
  - 쿠버네티스의 ReplicaSet 컨트롤러
  - 예 : 웹 서버 (Apache, nginx, IIS 등)
- Stateful Application
  - 각각의 App 마다 자신의 역할이 있음 (Primary, Secondary, Arbiter)
  - App이 죽으면 해당 App의 역할을 하는 App이 생성되어야 한다. 이름이 고유 identity로 식별요소이기 때문에 변경되면 안된다.
  - 각 App 마다 볼륨을 따로 써야 함
  - 내부 시스템들이 역할에 따라 연결되어 있어야 하고, 트래픽은 각 App의 역할에 맞게 들어옴
  - 쿠버네티스의 StatefulSet 컨트롤러
  - 예 : 데이터 베이스

### 2.1.2. StatefulSet Controller
- replicas 만큼 파드가 순차적으로 생성됨 (ReplicaSet은 동시에 생성됨)
- 파드의 이름은 index로 설정됨 (ReplicaSet은 랜덤으로 설정됨)
- 파드 삭제 후 재생성될 때, 기존 이름으로 생성됨
- replicas를 0으로 설정하면, 인덱스가 높은 순서대로 파드가 순차적으로 삭제됨 (ReplicaSet은 동시에 삭제됨)

### 2.1.3. 볼륨 설정
- 템플릿을 통해 파드가 생성되고, volumeClaimTemplates를 통해 PVC가 동적으로 생성되어 파드와 연결된다. (ReplicaSet은 PVC를 직접 생성해야 됨.)
- 파드가 생성될 때 새로운 PVC와 연결되어 파드마다 각자의 역할에 맞는 데이터를 연결할 수 있으며 파드가 삭제되고 재생성되어도 전에 연결되어 사용되었던 PVC와 연결된다. (ReplicaSet은 하나의 PVC에 모든 파드가 연결됨)
- PVC가 동적으로 생성되기 때문에 PVC와 해당 PVC와 연결되어야 할 파드는 동일한 노드에 생성된다. (ReplicaSet은 nodeSelector를 통해 PVC가 있는 노드에 파드가 생성되도록 지정해야 한다.)
- replicas를 0으로 설정하여 파드가 삭제되어도, PVC는 삭제되지 않는다.

## 2.2. Ingress Controller

기능
- url path에 따라 특정 서비스로 연결하도록 설정해 줌
- Service LoadBalancing
  - L4/L7스위치의 역할을 함
    - 각 path로 들어오는 요청을 Pod로 분산해줌
    - 각 path에 맞는 ingress rule을 설정해서 외부 접근 시 path에 맞는 서비스로 연결
- Canary Upgrade
  - 새로운 버전을 배포할 때 설정해준 비율에 따라서 일부 트래픽은 새로운 파드로 접근하도록 해준다.
  - 두 버전의 서비스를 연결해서 일부 트래픽을 새 버전으로 돌릴 수 있음
  - 어노테이션을 이용해 트래픽 조절 가능
    - @weight : 파드별 트래픽 비율 조절
    - @header : 특정 헤더값을 가진 경우 특정 파드로 매핑 예) 언어별 트래픽 지정
- Https
  - tls 옵션을 이용해 인증서가 담긴 secret을 연결하여 유저가 https로만 요청을 보내도록 할 수 있음
  - Pod 자체에서 인증서 기능을 제공하기 힘든 경우에 사용 가능

작동 원리
- Ingress object
  - 연결 정보 설정 : Host, path, serviceName을 설정함
  - path와 serviceName을 매핑해서 각 서비스로 접근하도록 설정
- Ingress controller
  - 실제 연결을 해주는 구현체
  - Nginx, kong 등
    - Nginx라는 ingress controller를 사용하게 되면 nginx라는 namespace가 생기고 해당 네임스페이스 안에 deployment, replicaset이 만들어지며 여기에 맞춰서 nginx pod가 생성됨
  - 외부에서 service로 요청이 오면 연결된 pod가 ingress rule에 따른 연결을 해 줌
  - Ingress 기능을 수행하는 네임스페이스와 파드가 생성되고 Ingress에 설정한 룰에 따라 서비스에 접근할 수 있다.
  - 외부에서 들어오는 트래픽이 해당 파드를 지나도록 외부에서 접근할 수 있는 서비스를 생성하여(NodePort, LoadBalancer) 파드와 연결시켜줘야 한다.

## 2.3. Autoscaler

종류
- HPA (Horizontal Pod Autoscaler)
  - 파드 개수 늘리기 (Scale In/Out)
  - 파드의 리소스(메모리, CPU) 상태를 감지하고 있다가 필요시 컨트롤러의 replicas를 조정한다.
  - 권장되는 조건
    - 기동이 빠르게 되는 App
    - Stateless App
- VPA (Vertical Pod Autoscaler)
  - 파드 리소스 늘리기 (Scale Up/Down)
  - VPA가 파드의 리소스 상태를 감지하고 있다가 필요시 파드의 리소스를 증가시켜서 재시동한다.
  - 권장되는 조건
    - Stateful App
  - 하나의 컨트롤러에 HPA와 함께 사용하면 안된다.
- CA (Cluster Autoscaler)
  - 클러스터에 동적으로 워커노드 추가하기
  - CA에 노드 생성 요청을 하면, 사전에 CA와 연결된 Cloud Provider에 노드를 생성한다.
  - 이후에 로컬 노드의 자원에 여유가 생기면, CA에 노드 삭제 요청을 하고, Cloud Provider에 있는 노드의 파드가 로컬 노드로 옮겨진다.

HPA 상세

- target : HPA를 적용할 deployment나 controller 지정
- maxReplicas, minReplicas : HPA에 의해 scale out/in 될 최대, 최소 수치
- metrics : scale out/in의 기준을 설정할 수 있는 옵션
- type : Resource, Pods, Object 설정 가능
  - Resource : Pod의 resource 값을 기준으로 결정
  - Pods : custom api로 파드와 관련된 정보를 통해서 결정
  - Object : Pods 외에 ingress나 다른 오브젝트의 정보로 결정
- name : cpu, memory 설정 가능
  - target.type : Utilization, AverageValue, Value
  - 스케일할 조건 기준 : 파드 사용률 기준, 평균 값 기준, 특정 값 기준
  - 설정한 target.type의 값에 따라서 averageUtilization, averageValue, value 옵션을 추가로 설정할 수 있다.
- 파드 갯수를 늘리는 방법
  - Target 리소스의 averageUtilization 비율에 리소스 limit을 곱한 값으로 총 사용량을 나눠 scale in/out 진행
  - 위 그림에서는 10m * 0.50 = 5m로 limit인 20m을 나누면 4가 나오기 때문에 파드를 4개 늘려 준다.
