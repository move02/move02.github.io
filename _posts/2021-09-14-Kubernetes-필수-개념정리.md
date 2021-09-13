---
layout: post
title: \[Kubernetes\] 필수개념 정리
categories: [TIL-Kubernetes]
comments: true
---

쿠버네티스를 운영하기 위한 기본 지식 정리.
쿠버네티스 구성요소와 관련된 가벼운 지식 소개는 https://www.redhat.com/ko/topics/containers/kubernetes-architecture 참고

## 개념 및 구성요소 정리

### 클러스터 구조

클러스터 전체를 관리하는 컨트롤러로써 마스터가 존재하고, 컨테이너가 배포되는 머신인 노드(worker node)가 존재한다.

- 마스터 노드와 워커 노드
- 마스터와 노드
- control plane과 nodes

마스터는 control plane 이라는 이름으로도 불리우는데 워커 노드와 그 안의 pod 들을 관리하는 역할을 한다.
운영환경에서 마스터와 노드들은 fault-tolerance와 high availability를 위해 대부분 여러 서버에 걸쳐서 운영된다.

![](/img/kube/components-of-kubernetes.svg)

### 오브젝트

쿠버네티스를 이해하기 위한 가장 중요한 개념이다. 쿠버네티스가 상태를 관리하는 대상(리소스)을 통틀어 칭한는 단어. 여러가지 형태의 오브젝트를 제공하고, 새로운 오브젝트를 추가하기가 쉽기 때문에 확장성이 좋음.

> **공식 문서** 에서는.. 
> "쿠버네티스 시스템에서 영속성을 갖는 개체", "하나의 의도를 담은 레코드(a record of intent)" 라고 소개하고 있으며 쿠버네티스 시스템(컨트롤 플레인) 은 클러스터가 "의도한" 상태가 되도록 동작한다고 소개하고 있다. (무슨 의미인지는 아직 잘 모르겠음.)

#### 명세(Spec), 상태(Status)

오브젝트의 명세(spec)는 특성(설정 정보, 원하는 특징, 의도한 상태)을 기술한 설명서. yaml, json 등의 형태로 기술 가능.
오브젝트의 상태(status)는 쿠버네티스 시스템과 컴포넌트에 의해 제공되고 업데이트된 오브젝트의 현재 상태. 컨트롤 플레인은 모든 오브젝트의 상태를 사용자가 의도한 상태(desired state)와 일치시키기 위해 지속적이고 능동적으로 관리함.

#### Describe Object(오브젝트 기술)

쿠버네티스에 오브젝트를 생성하고 관리하기 위해서는 오브젝트에 대한 기본적인 정보와 함께 의도한 상태(desired state)를 기술한 오브젝트 spec을 제시해야 함.
대부분 `.yaml` 형태로 기술하여 kubectl에 제공하고, kubectl API는 이것을 json으로 변환하여 생성 요청을 한다.

아래는 nginx deployment 오브젝트의 예시와 필수 필드를 설명하는 것이다.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  selector:
    matchLabels:
      app: nginx
  replicas: 2 # tells deployment to run 2 pods matching the template
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.14.2
        ports:
        - containerPort: 80
```

생성된 (또는 제공되는) yaml 파일을 아래의 명령어를 입력해 실제 오브젝트 생성 요청을 할 수 있다.
`kubectl apply -f <object spec file>`

아래는 공통적으로 모든 오브젝트가 가져야하는 필수 속성
- apiVersion: 오브젝트를 생성하기 위해 사용하고 있는 쿠버네티스 API 버전
- kind: 생성하고자 하는 오브젝트의 종류
- metadata: 오브젝트의 이름, UID, namespace 등을 포함하여 오브젝트를 구분지어주는 데이터
- spec: 오브젝트에 대한 의도한 상태(desired state of an object)

### Pod

**Pod** 는 쿠버네티스에서 가장 작은 배포 단위의 컴퓨팅 자원이다. 

하나 이상의 컨테이너를 담고있으며, 스토리지와 네트워크 자원을 공유하고 컨테이너 실행방법에 대한 명세 등을 포함함.
아래는 nginx 컨테이너를 포함하는 Pod의 명세 예시

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
spec:
  containers:
  - name: nginx
    image: nginx:1.14.2
    ports:
    - containerPort: 80
```

IP와 스토리지(볼륨)을 공유하는 특성 때문에 같은 Pod 내의 컨테이너끼리 localhost를 통해 통신이 가능하고, 다른 컨테이너의 로그 등을 수집할 수 있다.

> 핵심 애플리케이션과 이를 뒷받침 또는 모니터링하기 위한 프로그램을 같이 배포하는 패턴을 MSA에서 사이드카 패턴이라고 한다. (고 한다. 이 외에도 Ambassador, Adaptor Container 등 다양한 패턴이 있음) 
> ex) 로그 수집기

### Volume

일반적으로 컨테이너마다 로컬 디스크를 생성되지만 이는 영구적이지 않기 때문에 컨테이너의 재기동, 삭제와 관계없이 데이터를 영속시켜야 하는 경우에는 **볼륨** 이라는 특별한 스토리지를 사용해야 한다.

도커의 볼륨은 단순히 호스트의 디스크를 마운트하거나 다른 컨테이너에 있는 반면, 쿠버네티스는 여러 유형의 볼륨을 갖는다.

볼륨은 하나의 Pod에 있는 컨테이너들이 공유할 수 있는 외장디스크처럼 생각하면 편하다. 서로 다른 유형의 볼륨도 하나의 Pod에 붙여서 사용 가능함. (Pod 또는 Deployment 스펙에 정의하기 - spec.volumes)

라이프사이클에 따라 임시 볼륨과 퍼시스턴트 볼륨(PV)로 나뉘기도 하고 연결할 수 있는 [볼륨의 종류](https://kubernetes.io/ko/docs/concepts/storage/volumes/#volume-types)(클라우드, 파일서버 종류 등)에 따라 나뉘기도 한다.

PV와 PVC에 대한 개념은 중요하므로 다음에 따로 정리할 예정.

### Service

애플리케이션이 실행되는 컨테이너는 Pod위에 올려서 사용하는데 이 Pod는 여러 개를 묶어서 클러스터처럼 이용할 수도 있고, 장애가 발생하거나 하면 노드를 옮겨다닐 수 있기 때문에 IP 동적으로 변한다.
이처럼 동적으로 변하는 IP를 지속적으로 추적해서 안정적인 연결을 제공하기 위해 쿠버네티스에서는 **서비스** 라는 오브젝트를 사용한다.

라벨(레이블)과 라벨 셀렉터 라는 개념을 이용해서 Pod를 계속 추적하게 된다. Pod를 정의할때 메타데이터 속성에 라벨을 정의하고 서비스를 정의할 때 Pod에 붙어있는 라벨을 골라서 하나의 서비스로 묶어 제공할 수 있도록 라벨 셀렉터를 이용한다. [라벨과 셀렉터](#라벨-셀렉터)

아래는 세 개의 Pod을 하나의 Service로 묶는 예시이다.

```yaml
kind: Pod
apiVersion: v1
metadata:
  app: pulse-api
spec:
  containers:
  - name: pulse1
    image: <my-image-name>:<tag>
    ports:
    - containerPort: 8080

---

kind: Pod
apiVersion: v1
metadata:
  app: pulse-api
spec:
  containers:
  - name: pulse2
    image: <my-image-name>:<tag>
    ports:
    - containerPort: 8080

---

kind: Pod
apiVersion: v1
metadata:
  app: pulse-api
spec:
  containers:
  - name: pulse3
    image: <my-image-name>:<tag>
    ports:
    - containerPort: 8080
```

```yaml
kind: Service
apiVersion: v1
metadata:
  name: pulse-service
spec:
  selector:
    app: pulse-api
  ports:
  - protocol: TCP
    port: 80
    targetPort: 8080
```

![](/img/kubernetes-service.svg)

서비스를 퍼블리싱 하는데에 있어 몇 가지 유형이 있다.

- ClusterIP: 서비스에 클러스터 내부 IP를 할당한다. 서비스 클러스터 내에서는 이 서비스에 접근이 가능하지만, 클러스터 외부에서는 외부 IP를 할당받지 못했기 때문에 클러스터 바깥쪽에서 이 서비스로 연결하고자 하자면 Ingress를 사용해야 한다.
- LoadBalancer: 로드밸런서가 제공되는 상황에서 사용할 수 있는 방법으로, 외부 IP를 가진 로드밸런서를 할당함. 대부분 IaaS 플랫폼(GCP, AWS, Azure) 등에서 서비스를 노출시킬 때 사용하는 방법이며 Bare-metal 클러스터의 경우에는 **[MetalLB](https://metallb.universe.tf/)** 같은 로드밸런서 구현체를 사용할 수 있지만 프로젝트가 아직 베타단계 수준이므로 고민해봐야 할 선택지이다. (레퍼런스도 아직까지 많지 않음.)
- NodePort: 클러스터 IP뿐 아니라, 클러스터를 구성하는 모든 노드의 IP와 포트를 통해 접근이 가능하게 됨. 예를 들어 `10.1.1.1` ~ `10.1.1.6` 의 여섯 개의 노드로 구성된 클러스터 위에 nodePort 를 31234로 지정한 서비스를 배포하면 `10.1.1.1:31234`, `10.1.1.2:31234`, ... , `10.1.1.6:31234` 모두에서 접근이 가능하다는 뜻. 
- ExternalName: 클러스터 내부에서 클러스터 외부에 있는 애플리케이션을 호출할 때 사용한다. (클러스터 외부에 있는 DB 호출 시) 일반적으로 exxternalName 속성에 DNS를 지정하는데 IP로 지정하고자 할 경우 Endpoints 리소스를 추가로 설정해야한다.

라벨과 라벨 셀렉터만 가지고도 마법처럼 Pod을 묶어 부하분산과 Fail over를 지원하는 점이 인상깊지만 내부 동작은 훨씬 더 복잡하니 **다음 기회에 더 자세히 공부해서 정리하도록 한다.** (+ 쿠버네티스 네트워크)

### Namespace

동일한 쿠버네티스 클러스터 내에서 구분지을 수 있는 논리적인 가상 클러스터를 네임스페이스라고 한다.

> 공식 문서에서는 사용자가 거의 없거나 수십명 정도 되는 경우에 네임스페이스를 고려할 필요가 없다고 하지만, 하나의 클러스터가 여러 사용목적을 가지면 논리적인 단위 구분을 통해 리소스를 구분지을 필요가 있다고 생각한다.

`kube-` 접두사로 시작하는 네임스페이스는 쿠버네티스 시스템용으로 예약되어 있으므로, 사용자가 사용하지 않도록 한다.

대부분의 리소스를 네임스페이스 별로 생성하여 관리할 수 있으며, 네임스페이스간 완전히 분리되는 것으로 서로 다른 네임스페이스의 Pod 간에도 통신이 가능하다.

다른 네임스페이스에 있는 서비스와 통신하고 싶을때는 빌트인 DNS 서비스를 이용하여 서비스 이름을 가르키면 된다.

```
<Service Name>.<Namespace Name>.svc.cluster.local
```

반대로 원하는 수준의 충분한 네임스페이스 간 네트워크 분리를 제공하지 않는다고 한다. 네트워크 정책 등을 이용하여 막을 수는 있지만, 결제나 보안과 같은 기능을 쿠버네티스에 올릴때는 높은 수준의 분리 정책을 원하는 경우에는 쿠버네티스 클러스터 자체를 분리하길 권장한다고 한다. ([쿠버네티스 공식 블로그 - 네임스페이스](https://kubernetes.io/blog/2016/08/kubernetes-namespaces-use-cases-insights/))

### 라벨, 셀렉터

라벨은 쿠버네티스의 리소스를 선택하는데 사용이 된다. 각 리소스(오브젝트)는 라벨을 가질 수 있고 선택조건을 지정하여 해당 라벨을 가진 리소스만 선택이 가능하다. 아래와 같이 metadata 속성 안에 지정이 가능하고 `key:value` 형태로 정의한다. 생성 이후에 얼마든지 수정이 가능하다.

```yaml
metadata: 
  labels: 
    release : stable,
    environment : production
    tier: backend
```

라벨 키는 슬래시(`/`)를 기준으로 prefix와 name으로 구분한다. `<prefix>/<name> : value`
아래는 prefix, name, value에 관한 규칙.

- prefix는 선택이며, 253자를 넘지 않아야 한다.
- prefix는 DNS의 하위 도메인으로 해야한다. (<- 개인적으로 이 말이 무슨 뜻인지 잘 이해가 안감.)
- name은 필수 값이고 (당연하지만) 시작과 끝은 알파벳과 숫자이며, `-`, `_`, `.`과 함께 사용할 수 있다.
- value는 비워둘 수도 있고 비워두지 않는다면 시작과 끝은 알파벳과 숫자이며, `-`, `_`, `.`과 함께 사용할 수 있다.

셀렉터는 이전의 [Service](#service)에서 잠깐 언급했지만, 사용자가 지정한 라벨을 이용해 오브젝트들을 선택할 수 있게 해준다. **일치성 기준**과 **집합성 기준** 이라느 두 가지 셀렉터를 지원하는데 `=`, `=!`로 질의를 하는지 `in`, `notin`로 질의하는지에 대한 차이이다. 조합해서 같이 사용할수도 있다.

> **어노테이션** 이라는 사용자정의 메타데이터도 존재하는데, 라벨과의 차이는 쿠버네티스 시스템에서 식별을 하느냐 하지 않느냐의 차이이다. 단순하게 생각하면 '셀렉터에 잡히지 않는 메타데이터' 라고 이해하면 될 듯 하다. 주로 다른 사용자에게 해당 리소스에 대해 알려주고싶은 유용한 정보(만든 사람, 참고url, 빌드번호, PR, commit id 등)를 작성. 

라벨은 규칙 안에서 원하는대로 작성이 가능하지만 쿠버네티스에서 공식적으로 [권장하는 레이블](https://kubernetes.io/ko/docs/concepts/overview/working-with-objects/common-labels/)이 있으니 참고하면 좋을듯. 

### 컨트롤러

본 문서의 [오브젝트](#오브젝트) 섹션에서 컨트롤 플레인은 모든 오브젝트의 상태를 사용자가 의도한 상태(desired state)와 일치시키기 위해 지속적이고 능동적으로 관리한다고 했다. 정확하게 얘기하자면 컨트롤 플레인의 kube-controller-manager 컴포넌트가 컨트롤러를 구동하여 컨트롤러가 추적하는 리소스를 의도된 상태가 되게끔 명령을 내리는 것이다.

공식 홈페이지는 컨트롤러 및 컨트롤 루프를 온도 조절기에 비유하는데, 사용자가 설정한 온도가 *의도한 상태*(Desired state)이고 조절기는 실내 온도가 설정온도에 맞게끔 지속적으로 *현재 상태*를 *의도한 상태*에 맞추려고 동작한다. 각 컨트롤러는 지속적으로 클러스터의 리소스를 추적하면서 클러스터의 상태를 의도한 상태에 가깝도록 움직인다.

리소스를 개발자가 직접 손대지 않고 리소스의 *의도한 상태*를 정의하기만 하면 해당 상태로 되기 위해 만들기 위해 대신 일을 해주는 것이 컨트롤러 라고 이해하면 될 것 같다. 컨트롤러를 직접 생성하기 보다는 필요한 리소스를 추상화한 오브젝트를 명세하면 내장된 컨트롤러가 일을 해주는 것.

컨트롤러를 생성하는 대표적인 워크로드 리소스의 종류는 아래와 같다. (컨트롤러의 종류)

- **ReplicaSet** : 여러 개의 Pod 집합을 안정적으로 유지하도록 하는 리소스. 명시된 파드 갯수에 대한 가용성 보증을 위해 정의하고 사용됨.
- **Deployment** : ReplicaSet과 ReplicationController 상위 추상화하여 사용되는 리소스. 상태가 없는(stateless) 애플리케이션을 배포하고 업데이트 할 때 일반적으로 많이 사용됨. 기본적으로 Pod의 레플리케이션의 관리를 하면서도 롤백을 위한 기존 컨트롤러 관리 등 여러 기능을 포함함.
- **StatefulSet** : 상태 유지가 필요한(stateful) Pod이 유지되도록 하는 리소스. Pod 집합의 디플로이먼트와 스케일링을 관리하고, 순서와 고유성을 보장한다. 같은 애플리케이션을 실행하는 Pod이더라도 각각의 역할이 다르다는 얘기. DB처럼 데이터를 영속시키거나 pod 간의 순서에 민감한 어플리케이션을 실행할 때 주로 사용된다.
- **Daemonset** : 모든 노드에 동일한 Pod을 실행시키고자 할 때 사용하는 리소스. 주로 메트릭, 로그, 스토리지 등을 활용하고자 할 때 사용한다. (클러스터 내 네트워킹 관리를 위해 kube-proxy가 데몬셋으로 기본적으로 존재한다.)
- **Job** : 배치성 프로세스를 위한 리소스. 스케줄링된 작업을 수행하기 위해 Pod을 만들고 그 안에서 작업이 완료될 때 까지 실행하는 Pod을 관리한다. (장애에 따라 pod 재 생성 또는 job 재 실행 등)
- **CronJob** : Job과 동일하지만 cron을 이용해서 반복적으로 작업을 수행한다. Job 보다 상위의 컨트롤러로 Job을 관리할 수도 있다.
- ReplicationController : 지정된 수의 Pod 레플리카가 계속 실행되도록 하는 리소스. replicaset을 유지하도록 조정하는 컨트롤러. 이 컨트롤러를 직접 생성하고 실행시키기 보다는 ReplicaSet을 구성하는 Deployment를 통해 Pod의 레플리케이션을 하는 방법이 권장된다. (직접 사용할 일 없다는 뜻)

각 리소스의 자세한 예시는 조금 더 시간이 지난 후에 컨트롤러에 대해 자세히 정리해보도록 하겠다.

## 참고

- [조대협님 블로그](https://bcho.tistory.com/1262)
- [핀다 기술블로그 - 쿠버네티스 네트워크 정리](https://medium.com/finda-tech/kubernetes-%EB%84%A4%ED%8A%B8%EC%9B%8C%ED%81%AC-%EC%A0%95%EB%A6%AC-fccd4fd0ae6)
- [구글 클라우드 블로그 - Kubernetes best practices](https://cloud.google.com/blog/products/containers-kubernetes/kubernetes-best-practices-organizing-with-namespaces)
