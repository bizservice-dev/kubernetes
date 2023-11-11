# 2주차

## Controller

### Controller 의 기능(역할)
- auto healing
    - 노드 위의 파드나, 노드가 다운되었을 때 컨트롤러가 즉각적으로 인지하고 다른 노드에 해당 파드를 다시 띄운다.
- auto scaling
    - 파드 리소스가 limit 상태가 되었을 때 이를 감지하고 파드를 하나 더 생성하여 트래픽을 분산시키고 파드가 죽지 않도록 만든다.
- software update
    - 여러 파드의 버전을 업그레이드 해야하는 경우, 컨트롤러를 통해 쉽게 할 수 있으며, 롤백기능도 제공한다.
- job
    - 일시적인 작업을 해야 할 경우, 컨트롤러가 필요한 순간에만 Pod를 만들어서 이행하고 삭제한다.
    - 그 순간에만 자원이 활용되고, 이후 반환된다.
### Controller 의 설정 요소
- Template
    - Controller와 Pod는 Service와 Pod처럼 label과 selector로 매핑된다.
    - 컨트롤러를 만들 때 template으로 pod의 내용을 넣어두고, Pod가 다운되면 Template에 있는 pod로 새로 만들어준다.
    - template의 버전을 변경함으로써 모든 파드의 버전을 업데이트 할 수 있는 형태이다.
- Replicas
    - replicas만큼 Pod의 갯수가 관리된다.
    - 갯수를 늘려주면 scale out, 줄이면 scale in
    - 파드에 문제가 생겼을 때 Replicas 수치만큼의 파드 갯수를 유지하려고 한다.
    - 보통 replicas와 template의 내용을 가지고 파드를 자동으로 생성한다.(파드를 직접 만들지 않고 컨트롤러를 관리하는 형태로 많이 사용한다고 한다.)
- Selector
    - 파드를 연결하기 위한 key-value 값
    - ReplicaSet은 matchLabels와 matchExpressions가 있다.
        - matchLabels : key-value가 모두 같으면 연결
        - matchExpressions : key-value를 디테일하게 조정 가능
            - 존재여부 등으로 조건을 걸 수 있음
                - Exists: 키에 맞는 값을 가지고 있는 Pod 연결
                - DoesNotExist: 해당 키가 없는 Pod 연결
                - In: 키에 해당하는 value 값들을 만족하는 경우 연결
                - NotIn: 키에 해당하는데 value값들을 만족하지 않는 경우


## Controller 의 종류
### 1. Replication Controller
- 현재 Deprecated된 object이며 ReplicaSet으로 대체되었다.
- Template, Replicas, Selector 등을 이용해서 파드를 관리할 수 있다.
- Template, Replicas는 공통적인 기능, Selector는 ReplicaSet에만 확장된 기능이 있다.
### 2. ReplicaSet
- Replication Controller 와 동일한 기능을 제공한다.
- 추가적으로 selector 가 좀 더 확장된 기능을 가지고 있다.(위에 Selector 부분 참고)
### 3. Deployment
- 현재 운영중인 서비스를 업그레이드하려고 할 때, 재배포를 도와주는 컨트롤러
- Deployment에는 selector, replicas, template이 들어가는데 이 값은 파드를 직접 생성하는게 아닌 ReplicaSet을 생성하는 용도로 사용된다
- 4가지 배포 방식
    - ReCreate
        - Deployment의 replicas를 0으로 만들고 변경된 ReplicaSet을 새로 생성한다.
        - downtime이 발생하므로, 일시정지가 가능한 서비스에 대해서만 사용하는 것을 권장한다.
        - revisonHistoryLimit은 relicas가 0인 ReplicaSet의 갯수(기본은 10개)
    - Rolling Update
        - v2 pod를 하나씩 생성하며 v1 pod를 하나씩 삭제
        - 중간에 v1, v2를 전부 다 쓰는 상황 존재 → 추가적인 자원이 필요함
        - Deployment의 기본값
        - minReadySeconds: v1과 v2를 추가하고 삭제하는데 걸리는 시간
    - Blue/Green
        - Controller를 하나 더 만든다(v2버전) → 서비스에 있는 라벨을 v1에서 v2로 변경한다.
        - 순간적으로 변경되어 downtime은 없다.
        - v2에 문제가 생기면 v1으로 라벨을 바꿔주며 롤백한다.
        - 안정적인 배포방식이지만, 자원이 역시나 2배로 필요하다(v1, v2가 공존하므로)
    - Canary
        - 동일한 라벨에 대해서 v1, v2 pod가 생성 → v1, v2 pod를 둘 다 활용
            - 새 버전에 대한 테스트 및 불특정 다수에 대한 테스팅에 활용
        - 서비스를 2개 만들어서 할 수도 있다.
            - 각각의 서비스를 만들고, Ingress Controller를 활용해서 특정 path에 대해서 특정 controller를 쓰게끔 한다. 이후 기존에 쓰던 Path에 전부 다 합치는 방식
        - downtime은 없지만, 자원이 필요하다.
### 4. DaemonSet
- 노드의 자원과 상관없이 모든 노드에 pod가 하나씩 만든다.
    - 각 노드의 성능 수집(prometheus 등)
    - 로그 수집(fluentd 등)
    - 스토리지 (GlusterFS)
- 쿠버네티스도 프록시 역할을 하는 pod를 각각 만든다.
- nodeSelector를 이용해서 특정 노드에만 pod 생성 가능
### 5. Job
- 특정 작업만 처리하고 종료시킬 파드를 관리하는 컨트롤러
- Controller에 의해 만들어진 pod들은 노드에 장애가 감지되면 다른 노드에 재생성된다.
### 6. CronJob
- Job들을 주기적인 시간에 따라 생성한다.
    - jobTemplate의 내용에 따라 job을 생성
    - cron format의 schedule에 따라 job을 생성( * * * * *)
- concurrencyPolicy
    - allow(default)
        - 설정된 스케줄 간격마다 job과 pod가 생성됨
        - 사전에 만들어진 pod와 무관하게 스케줄대로 생성
    - forbid
        - 기존에 만들어진 pod가 실행중이면 다음 스케줄이 와도 생성하지 않음, 해당 pod가 종료되면 즉시 다음 pod가 만들어짐
    - replace
        - 이전 Job이 실행중이면 현재 주기의 새로운 Job을 생성하지 않고 기존에 생성된 Pod만 그대로 사용하는 방식
- suspend를 통해 일시정지가 가능하다.