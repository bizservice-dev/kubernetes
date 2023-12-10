# 5주차

## Component
- 각 component 들은 파드형태로 띄워져 있다.
- 각 component 생성 yaml 파일 위치 : 마스터 노드의 /etc/kubernetes/manifests
  - kube-apiserver.yaml, etcd.yaml, kube-scheduler.yaml, kube-controller-manager.yaml, kube-proxy.yaml
- 쿠버네티스 기동 시 파일들을 읽어서 파드를 띄운다.

<br/>

### Master Node
- etcd : 쿠버네티스의 여러 데이터들을 저장하는 저장소
- kube-scheduler : 각 노드의 자원 상태를 체크 + watch 기능 ( kube-apiserver를 체크해서 etcd에 pod 생성 명령이 있는지 체크한다)
  - etcd에 존재하는 노드가 할당되지 않은 파드들을 찾아 자원 상황에 맞게 노드를 할당해주는 역할
- kube-apiserver
- controller manager : DaemonSet, StatefulSet, Deployment, ReplicaSet 각각의 기능을 담당하는 스레드들이 존재하는 파드
  - kube-apiserver에서 watch기능을 통해서 해당 컨트롤러 관련 요청이 들어오면 해당 스레드가 동작하여 명령을 수행한다.

### Worker Node
- kubelet : watch 기능 ( kube-apiserver를 확인하여 pod에 자기 자신의 노드 정보가 붙어있는 파드를 찾은 경우에 etcd에 존재하는 정보로 해당 노드에 파드를 생성한다)
  - container runtime에 컨테이너 생성 요청
  - kube-proxy에 network 생성을 요청
- container runtime (ex docker)
- kube-proxy : container runtime에 의해 생성된 파드와 통신할 수 있도록 네트워크 연결

### Pod 생성 과정
1. 사용자가 pod 생성을 요청한다.(kubectl create)
2. kube-apiserver로 요청이 전달된다.
3. Etcd에 pod에 대한 입력정보를 저장한다.
4. kube-scheduler가 watch 기능으로 etcd에 들어온 pod 생성요청을 확인한다.
5. 노드 자원 확인 & 어느 노드로 가는게 좋을 지 확인 후 etcd에 있는 pod에 노드 정보를 추가한다.
6. 노드에 있는 kubelet이 watch로 etcd에 있는 pod에 자신의 노드 정보가 있음을 확인한다.
7. 정보를 가져와 pod를 만들기 시작한다.
8. 도커에게 컨테이너 생성을 요청허고, 도커가 컨테이너를 생성한다.
9. kubelet이 kube-proxy에 네트워크 생성을 요청한다.
10. kube-proxy가 컨테이너가 통신이 되도록 도와준다.

### Deployment 생성 과정
1. 사용자가 deployment 생성을 요청한다.(kubectl create, deployment replicas: 2)
2. kube-apiserver로 요청을 전달한다.
3. Etcd에 deployment에 대한 입력정보를 저장한다.
4. kube-apiserver에서 Controller Manager에 정보를 전달한다.
5. Controller Manager(deployment)가 replicaSet 생성을 요청한다.
6. Controller Manager(replicaSet)이 etcd내의 replicaSet에 대한 정보 확인 후 Pod 생성을 요청한다.
7. kube-scheduler가 할당될 노드를 지정한다.

<br/>

---
<br/>

## Networking

<br/>

### Pod Network

- 클러스터를 설치할 때 파드 네트워크 CIDR로 이 네트워크 대역에 대해 설정한다.
- 파드 내 컨테이너로 가는 네트워크
- 네트워크 플러그인을 통한 파드 간 통신
  - CNI를 통해 오픈소스 네트워크 인터페이스 설치 가능

#### 구성
- Pause Container : 파드의 네트워크를 담당한다.
  - 파드를 만들 때 자동으로 생성되는 컨테이너
  - 파드 생성시 퍼즈 컨테이너를 통해 네트워크 네임스페이스가 생성되고 파드 내의 컨테이너 간 통신은 포트를 통해 이루어지게 된다.
  - 각 파드는 워커노드의 host network namespace의 가상 인터페이스와 통신한다.
- Network Plugin : 클러스터의 네트워크를 담당한다.
  - kubenet 기준
    - 호스트 네트워크에서 각 파드와 통신하는 인터페이스들을 묶어 브릿지를 생성, 한 노드 위에 255개의 파드와 통신할 수 있도록 대역이 세팅된다.
    - router를 담당하는 NAT도 생성된다.
  - calico 기준
    - 호스트 인터페이스가 바로 연결된 라우터를 생성 : 큐브넷보다 더 많은 파드를 할당 가능
    - 타 노드에 있는 파드와 통신 가능한 오버레이 층이 생성된다.

### Service Network(iptables, IPVS)

- 서비스의 이름과 IP에 대한 정보가 master node의 kube-dns에 등록된다.
- 파드가 서비스를 호출하면 kube-dns를 통해 IP주소를 얻고, 이를 NAT에 호출하면 파드 매핑 정보를 얻어 트래픽이 네트워크 플러그인을 통해 해당 파드로 가게 된다.

#### proxy mode

- 유저 스페이스 모드
  - 리눅스 워커노드에 기본으로 설치되어 있는 iptables에 서비스 CIDR로 들어오는 트래픽이 모두 kube-proxy로 전달하도록 설정되어 있다.
  - kube-proxy는 자신이 가진 매핑정보를 보고 트래픽을 파드 네트워크로 넘긴다.
  - 모든 트래픽이 큐브 프록시를 지나야 해서 성능, 안정성 문제가 존재한다.
- Iptables/IPVS mode
  - default
  - 매핑정보가 직접 iptables에 등록된다.
  - IPVS는 iptables와 같은 역할을 하는데 부하가 클 때 더 성능이 좋다.


#### service type

- clusterIP
  - 파드들이 클러스터 내에서 부여받은 IP를 이용해 파드 간의 통신을 한다.
- NodePort
  - 노드들이 부여받은 포트로 외부 트래픽이 들어오면 iptables가 트래픽을 네트워크 플러그인으로 보내주고, NAT를 통해 파드 IP로 변환되어 파드 네트워크로 트래픽이 넘어간다.


<br/>

---
<br/>

## Storage

<br/>

### PV 를 만드는 방법
- Common
  - PV를 먼저 만들고 PVC를 연결하는 방법
- StorageClass - Grouping
  - 그룹의 개념으로 StorageClass 들을 만들고, PV들을 특정 StorageClass에 속하도록 만들어준댜.
  - PVC를 만들때 StorageClass 를 지정하면, 속해있는 PV 들 중에 적절한 하나와 연결이 된다.
- StorageClass - Dynamic Provisioning
  - StorageClass 를 만들 때 Provisioner 를 지정하고, PVC를 만들 때 StorageClass 를 지정한다.
  - 자동으로 Provisioner가 PV를 만들어주고 PVC에 연결된다.

### PV
- capacity : storage 용량 설정
- accessMode : ReadWriteOnce, ReadOnlyMany, ReadWriteMany 등 접근 방식
- Volume Plugin : PV의 실체를 결정하는 옵션
  - hostPath : 워커노드에 지정된 path 를 볼륨으로 사용
  - NFS : 외부 Storage에 NFS Server(Volume) 가 구성되어있을 때 PV가 이를 사용할 수 있도록 도와주는 클라이언트 역할을 해줌
  - Cloud Service : 클라우드 서비스 볼륨을 사용
  - 3rd Party Vendors : 외부 Storage 에 Volume Solution 가 설치되어있는 경우 이를 사용할 수 있도록 해주는 Vendor
- CSI (Container Storage Interface)
  - Volume Plugin 을 미리 반영하지 않았거나 Volume의 최신버전이 나왔을 때 바로 이를 지원할 수 있는 Volume Plugin 을 적용할 수 있도록 해준다.
  - 쿠버네티스에 CSI를 만드는 방법에 대한 가이드가 있고 기업들이 가이드에 따라 csi-plugin 이나 Provisioner 를 만들어놓고 설치방법만 사용자에게 안내해서 사용할 수 있도록 해줌
  - 기존 볼륨 시스템과 연결하는 방식이 아닌 새롭게 컨테이너로 볼륨 시스템을 띄울 수 있도록 해둔 기업도 많음

### StorageType
- 각 볼륨마다 지원하는 StorageType 이 있다.
  - NFS : FileStorage 지원
  - Cloud Service : BlockStorage, FileStorage, ObjectStorage 모두 지원
  - 3rd Party Vendors : FileStorage 지원
  - CSI : 각 솔루션마다 상이
- StorageType 에 따라 지원하는 AccessMode가 다르기 때문에 가이드 확인이 필요하다.
  - FileStoragem, ObjectStorage
    - RWO, RWM, ROM 모두 지원
    - 한 노드, 여러 노드에서 읽고 쓰기가 가능하다.
    - 공유 데이터용, 백업용, 이미지 저장용으로 주로 사용된다.
    - Deployment 에 PVC를 붙일 때 주로 사용된다.
- BlockStorage
  - RWO 만 가능
  - 한 노드에 스토리지를 마운팅하는 개념
  - DB 데이터용으로 주로 사용된다.
  - StatefulSet 에서 주로 사용된다.


<br/>

---
<br/>

## Logging

### Core Pipeline
- 쿠버네티스 자체에서 제공하는 파이프라인 기능
- Monitoring
  - 각 워커 노드마다 리소스 예측기인 cAdvisor가 존재하고 kubelet이 이를 통해서 cpu, 메모리 정보를 얻는다.
  - 각 노드의 kubelet에 수집된 모니터링 정보는 마스터 노드의 metircs-server로 노드 별 모니터링 정보가 모아진다.
  - kubectl top 명령으로 api-server를 통해 모니터링 정보를 조회할 수 있다.
- Logging
  - kubectl log 기능으로 api-server에 요청, kubelet을 통해 컨테이너 로그 파일을 확인할 수 있다.

### Service Pipeline 
- 별도의 플러그인을 설치해서 사용하는 파이프라인 기능
- Agent
  - kubelet과 비슷하게 daemonSet을 이용해 모든 노드에 설치되어 리소스 자원을 수집하는 역할
  - 노드 내에 존재하는 cAdvisor, docker, 워커노드 자체를 통해서 로그 정보와 리소스 자원을 수집한다.
- Metric Aggregator
  - 별도의 워커노드에 존재하여 매트릭스 서버와 같이 각 노드의 매트릭들을 수집하고 분석한다.
  - 많은 데이터가 저장되기 때문에 별도의 저장소가 필요하다.
- Web ui
  - Metric Aggregator에 존재하는 로그 정보를 쿼리하여 사용자에게 원하는 로그 정보를 보여줄 수 있다.
  - 대표적으로 ES, Loki, Prometheus가 있다.

### Logging Architectures

#### Node-level logging
- 단일 노드 안에서 일어나는 로깅 구조
- 파드가 살아있는 동안 유지된다.
- kubelet log 명령으로 파드의 현재 실시간 로그를 조회할 수 있는데, 파드가 죽게되면 더 이상 그 파드의 로그를 볼 수 없다. (만약 이 로그를 보고 싶다면 클러스터 레벨 로깅이 필요함.)

#### Cluster-level Logging
- 쿠버네티스 클러스터 내의 모든 노드들에 있는 노드들을 포괄하는 로깅
- 쿠버네티스에서 기능을 제공해주는 것은 아니고, 아키텍처만 제시해준다.
- Node Logging Agent 구조
  - Node Logging Agent : 각 노드에 데몬셋으로 에이전트를 둬서 이 경로의 로그들을 조회할 수 있다.
  - 이렇게 읽은 로그를 수집 서버로 전송하는 패턴
- Sidecar Container Streaming
  - 한 컨테이너에서 access.log와 app.log 라는 두 종류의 로그 파일이 쌓인다고 가정했을 때, 이 두 로그파일을 stdout으로 쏘게되면 워커노드의 파일이 쌓일 때, 로그가 한 파일에 섞이게 된다.
  - 이때, 메인 컨테이너에서 로그를 stdout으로 바로 쏘지 않고, 그냥 파일로만 저장하고 별도의 컨테이너를 하나 더 만들어서 파일을 읽어서 stdout으로 출력하게 만든다. (로그 파일을 분리할 수 있다.)
- Sidecar Container with a logging agent
  - 로깅 에이전트를 사이드카 형태로 파드 안에 집어넣는 경우
  - app 컨테이너는 stdout으로 출력하지 않아도 이 에이전트 컨테이너에 의해서 로그가 수집되고, 그 로그는 수집서버로 바로 보내주는 구조

### Logging/Monitoring Stack
- Web UI - Aggregator/Analytics - Agent 구조를 가지고 있다.
- ELK Stack : Kibana - Elasticsearch - Logstash
- EFK Stack : Kibana - Elasticsearch - Fluentd, Fluent-bit
- Prometheus : Dashboard - Server - Exportor
- PLG Stack : Grafana - Loki - Promtail
  - 파드 별 로그 파일이 쌓인다.
  - Promtail이 모든 노드에 생성하고 configMap이 파드의 log path를 읽도록 설정한다.
  - StatefulSet으로 Loki 생성 - Loki로 들어오는 서비스를 통해 모든 Promtail이 로그 파일을 서비스로 push
  - Deployment로 Grafana 생성 - 외부 접속을 위해 Nodeport로 변경 & Loki쪽 서비스로 쿼리를 날려 원하는 로그 조회
