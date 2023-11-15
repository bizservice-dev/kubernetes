# 3주차

## 1. Pod LifeCycle
![podlifecycle](/images/kubernetes-basic-inflearn/pod-lifecycle.png)

### Pod Status
- Phase : pod의 전체 상태
  - `Pending`, `Running`, `Succeeded`, `Failed`, `Unknown`

- Conditions : pod를 실행하는 단계와 상태
  - `Initialized`, `ContainerReady`, `PodScheduled`, `Ready`
  - reason
    - Conditions의 세부 내용을 알려주는 항목
    - 타입의 `status`가 False 인 경우, 원인을 알려주기 위해 존재한다.
    - `ContainersNotReady`, `PodCompleted`
- ContainerStatuses
  - 파드 내의 각 컨테이너들의 상태를 나타내는 속성
  - state
    - 컨테이너의 상태를 나타내는 속성
    - `Waiting`, `Running`, `Terminated`
  - reason
    - 세부 내용을 알려주는 항목
    - `ContainerCreating`, `CrashLoopBackOff`, `Error`, `Completed`

### LifeCycle 각 단계별 파드 상태
#### Pending
- Pod의 최초 상태
- Conditions
  - `PodScheduled` 
    - 파드가 노드에 정상적으로 올라가는지 체크
    - 정상이면 `PodScheduled` : `true`

  - `Initialized`
    - 보안, 볼륨 등 사전 세팅을 해야할 경우, 파드 생성 내용 안에 initContainers 라는 항목으로 초기화 스크립트를 넣을 수 있다. 
    - 이 스크립트가 본 컨테이너보다 먼저 실행이 되서 그 작업이 성공적으로 끝났거나, 아예 설정하지 않았을 경우에 `Initialized` : `true`, 실패인 경우에는 `Initialized` : `false`
    
- ContainerStatuses 
  - status : `Waiting`
    - 컨테이너의 이미지를 다운로드하고 있는 상태 
    - `reason` : `ContainerCreating`

#### Running
- 컨테이너가 기동되는 상태
- 정상적인 상황
  - Conditions 
    - `ContainerReady` : `true`
    - `Ready` : `true`
  - ContainerStatuses
    - `status` : `Running`
- 비정상적인 상황(하나 이상의 컨테이너가 기동 중에 문제가 발생한 경우 → 재시작됨)
  - Conditions
    - `ContainerReady` : `false`
    - `Ready` : `false`
  - ContainerStatuses
    - `status` : `Waiting`
    - `reason` : `CrashLoopBackOff`
          
#### Unknown
- Pending 이나 Running 중에 통신 장애가 발생한 상태, 통신 장애가 빨리 해결되면 다시 원래 상태로 돌아간다.
- 장애가 해결되지 않으면 Failed 상태가 된다.
          
#### Failed
- 파드가 작업을 처리하는 중에 내부 컨테이너가 하나라도 문제가 생긴 경우
- Pending 이나 Unknown 상태에서도 Failed 가 될 수 있다.
- Conditions
  - `ContainerReady` : `false`
  - `Ready` : `false`
- ContainerStatuses
  - `status` : `Terminated`
  - `reason` : `Error`

#### Succeeded
- 파드가 작업을 정상적으로 처리 완료한 상태
- Conditions
  - `ContainerReady` : `false`
  - `Ready` : `false`
- ContainerStatuses
  - `status` : `Terminated`
  - `reason` : `Completed`



## 2. Pod - ReadinessProbe, LivenessProbe

### ReadinessProbe
- Pod가 구동될 때 Pod와 Container 상태는 Running 이지만 App 이 Booting 중인 경우, 사용자가 접근하지 못하도록 서비스와 연결되지 않도록 해주는 기능
- 설정한 조건을 만족하는지 계속해서 App 의 상태를 체크하고 만족했을 경우 서비스와 파드를 연결시킨다.

### LivenessProbe
- Pod와 Container 상태는 Running 이지만 갑자기 App 에 장애가 나는 경우, 지속적인 실패를 막기 위해 장애 상황을 감지하고 재실행 시켜주는 기능
- 정상적으로 처리하고 있다가 갑자기 App에 문제가 생겨서 Probe 체크에 실패하게 되면 쿠버네티스는 이 파드에 문제가 있다고 판단하고 재실행시킨다.

### 설정하는 방식
- ReadinessProbe와 LivenessProbe는 사용 목적이 다를 뿐, 설정할 수 있는 내용은 같다.
- httpGet, Exec, tcpSocket 등 상태체크 방식을 설정
  - 3개중 하나는 꼭 설정해야 한다
- 상태 체크 성공 or 실패의 기준을 위한 옵션값들이 있다.
  - initialDelaySeconds : 최초 Probe 를 하기 전의 딜레이 시간 (default: 0초)
  - periodSeconds : Probe 를 체크하는 시간의 간격 (default: 10초)
  - timeoutSeconds : 결과가 오기까지 대기하는 시간 (default: 1초)
  - successThreshold : 성공으로 인정하기 위한 횟수 (default: 1회)
  - failureThreshold : 실패로 인정하기 위한 횟수 (default: 3회)

  
## 3. Pod - QoS(Quality of Service)
- 특정 Pod가 리소스가 필요한데 node에 자원이 없는 경우에 QoS를 통해 어떤 Pod의 자원을 가져올 지 결정한다.
  - Guaranteed, Burstable, BestEffort
  - 리소스 설정으로 memory, cpu를 어떻게 설정하는지에 따라 결정된다. 
  - BestEffort → Burstable 순으로 제거

### Guaranteed
- 모든 Container에 Request, Limit 설정
- Memory, CPU에 대해 모두 설정되어 있어야 함
- Memory, CPU의 request와 limit값이 일치해야 함

### Burstable
- Guaranteed와 BestEffort의 사이
- OOM Score에 따라 Burstable중에서는 어떤 것을 죽일지 결정(OOM Score가 더 큰 Pod가 먼저 제거)

### BestEffort
- 어떤 Container 내에도 Request와 Limit이 설정되지 않은 경우


## 4. Pod - Node Scheduling
- 파드를 특정 노드에 할당되도록 선택하기 위한 용도로 사용됨
### 노드를 직접 선택하는 방식
#### NodeName
- 명시적으로 할당할 수 있음
- 노드가 추가되고 삭제되면서 이름이 변경될 수 있어 잘 사용하지 않는다.

#### NodeSelector
- key-value label이 달린 노드에 할당
- 동일한 key-value label이 달려있다면, 리소스가 많은 노드에 할당
- 매칭되는 라벨이 없으면 pod가 할당되지 못한다.

#### NodeAffinity
- key만 설정하면 스케줄러가 자원이 많은 파드에 할당
- 조건에 맞지 않더라도 스케줄러가 판단해서 자원이 많은 노드에 할당하게끔 할 수도 있다.
- matchExpressions 활용
  - key, value, operator 를 사용해서 파드가 노드를 선택할 수 있도록 할 수 있다.
  - operator 종류
    - Exists : 키가 존재하는 노드에 할당
    - DoesNotExist : 키가 존재하지 않는 노드에 할당
    - In : 키가 같고 밸류에 포함되는 노드에 할당
    - NotIn : 키가 같고 밸류에 포함되지 않는 노드에 할당
    - Gt : 키는 같고 밸류값이 더 큰 노드에 할당
    - Lt : 키는 같고 밸류값이 더 작은 노드에 할당
- required, preferred
  - required: 키가 맞지 않으면 스케줄링 되지 않는다.
  - preferred: 키가 맞지 않아도 선호되지 않지만 할당은 된다.
    - preferred weight: 선호도에 대한 가중치
    - 스케줄러는 자원 + 가중치로 점수를 매겨서 노드를 결정

### Pod 간 집중/분산
- 여러 파드들을 한 노드에 집중하거나 파드들 간에 겹치는 노드가 없도록 분산해서 할당할 수 있음
#### Pod Affinity
- 두 파드가 같은 노드에 있어야하는 경우 사용할 수 있음
  - ex) PV 로 hostPath 를 사용해서 볼륨을 공유하는 경우
- Pod Affinity 설정을 통해 한 파드의 라벨을 다른 파드에도 지정해주면 해당 파드와 같은 노드에 생성이 됨
- topologyKey 를 설정해주므로써 해당 키를 가진 노드들로 범위를 지정할 수 있다.
- 최종적으로 topologyKey 를 통해 범위를 지정하고 그 범위 내에서 파드의 라벨을 보고 해당 파드가 있는 노드에 생성해주게 된다.
- matchExpressions 방식 활용 가능(노드가 아닌 파드의 라벨과 매칭되는 조건)
- 조건 만족할 때까지 노드 할당 X

#### Anti-Affinity
- 두 파드가 다른 노드에 있어야하는 경우 사용할 수 있음
  - ex) master - slave 관계의 파드
- Anti-Affinity 설정을 통해 한 파드의 라벨을 다른 파드에도 지정해주면 해당 파드와 다른 노드에 생성이 됨
- topologyKey 로 노드의 범위를 지정할 수 있다.
- Pod Affinity와 마찬가지로 matchExpressions 방식 활용 가능
- 위에서 말했던 required, preferred option 적용 가능하다.

### 특정 Node에는 할당 제한
- 특정 노드에는 아무 파드나 할당이 되지 않도록 제한하기 위해서 사용됨
#### Toleration/Taint
- 제한하려는 노드에 Taint 를 설정해두게 되면 아무 파드나 할당이 되지 않음
- 또한 직접 지정해서도 할당할 수 없음
- 이 노드에 할당해야하는 파드의 경우 Toleration 을 설정해줘야한다.
  - 내용으로는 key, operator, value, effect 가 있고 Taint 의 값들과 맞아야 한다.
- Taint 는 라벨인 키와 밸류가 있고, effect 라는 옵션이 있다.
  - effect 옵션
    - NoSchedule : 다른 파드들은 이 노드에 할당되지 않는다.
    - PreferNoSchedule : 가급적 스케줄링이 안되도록 한다.
    - NoExcute : 이미 해당 노드에 파드가 실행중일때 해당 파드를 삭제한다.
  - 파드가 삭제되지 않으려면 Toleration effect로 똑같이 NoExecute 를 설정해줘야함
  - 추가로 tolerationSeconds 라는 속성을 넣어주게 되면 해당 시간 이후에는 삭제가 됨
- 주의해야할 점
  - Toleration 설정을 했다고해서 Taint 설정을 한 노드에 무조건 할당되는 것이 아니라 다른 노드에 할당될 수도 있다.
  - 그렇기 때문에 추가적으로 nodeSelector 를 달아서 Taint 설정을 한 노드를 지정해줘야한다.

## 5. Service - Headless, Endpoint, ExternalName
- 사용자는 서비스와 파드에 접근하기 위해 clusterIP, NodePort, LoadBalancer를 활용한다.
- 쿠버네티스 클러스터에는 DNS Server가 존재한다.
  - 서비스의 도메인 이름과 IP가 저장되어있다.
  - Pod에서는 DNS를 통해 원하는 곳에 접근할 수 있다.
  
### Headless
- FQDN(Fully Qualified Domain Name)
  - <서비스이름>.<namespace 이름>.svc.<DNS 이름>
  - <파드의 Ip>.<namespace 이름>.pod.<DNS 이름>
- Pod가 Pod를 직접 연결하고 싶다면 Headless Service를 만들어야 한다.
  - clusterIP: None
  - pod에는 hostname에 도메인 이름을, subdomain에 서비스 이름을 넣어야 함.
  
### Endpoint
- 서비스와 파드 생성 시 selector의 label과 pod의 label를 연결 관리해주는 오브젝트
- 쿠버네티스는 서비스와 같은 이름으로 Endpoint설정, Endpoint 안에는 Pod의 접속정보를 넣어둔다.
- 사용자가 Endpoint에 직접 정보를 입력 가능

### ExternalName
- Service에 ExternalName에 도메인 이름을 넣어놓으면 DNS Cache가 IP를 알아낸다.
- Pod는 서비스만 가리키고 있으면, 서비스에서는 필요할 때마다 접속할 곳을 변경할 수 있음


## 6. Volume
### Dynamic Provisioning
- StorageClass → 동적으로 PV 생성

### Status & ReclaimPolicy
- PV는 available상태였다가 PVC와 연결되면 Bound가 된다.
- Pod와 연결되면 볼륨이 만들어진다.
- PVC를 삭제해야 PV가 released상태가 된다.
- 데이터 간 연결에 문제가 생기면 Failed가 된다.

#### Status
- Available : 최초 생성했을 때의 상태, 아직 연결 안된 상태, 연결 가능 상태
- Bound : 연결된 상태
- Released : PVC 삭제, 데이터와는 연결된 상태
- Failed : 실제 데이터 연결에 문제가 있는 상태
- ReclaimPolicy : PVC를 삭제했을 때 PV 상태에 대한 정책

#### ReclaimPolicy
- Retain
  - PVC가 삭제되면 Release
  - 기본 정책
  - 실제 볼륨은 유지되지만 다른 PVC에 연결할 수 없으므로 수동 삭제해야 함
- Delete
  - PVC를 지우면 PV도 같이 지워짐
  - StorageClass 기본정책
  - 볼륨의 종류에 따라 데이터가 삭제될 수 있음
- Recycle
  - PVC를 지우면 Available (재사용 가능)
  - deprecated
  - 실제 데이터가 삭제됨.
