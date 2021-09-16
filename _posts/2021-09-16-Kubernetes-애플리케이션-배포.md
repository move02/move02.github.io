---
layout: post
title: Kuberenetes 애플리케이션 배포(Manifest vs Helm)
categories: [TIL-Kubernetes]
comments: true
---

필수개념편에서 쿠버네티스 오브젝트를 정의하여 클러스터의 의도된 상태를 표현한다고 했다.
Pod, Service, Volume 등 각기 다른 종류의 오브젝트를 정의하여 우리가 쿠버네티스 클러스터 안에서 운영하고자 하는 애플리케이션을 만들고 서비스할 수 있다.

이번 편에서는 쿠버네티스 오브젝트를 통해 애플리케이션을 배포하는 방법을 비교해보고 구축중인 데이터 서버에서는 어떤 방법을 선택했는지에 대해 적어보려 한다.

## Manifest

이전 편까지 살펴본 쿠버네티스 오브젝트는 yaml 형태로 정의하여 사용하고는 했다. 대체로 공식 홈페이지를 보면 yaml 파일과 `kubectl apply -f <path>` 명령어를 이용하는 예제가 많다.

아래는 쿠버네티스 클러스터 구성 후 백엔드 애플리케이션을 쿠버네티스에 배포하는 manifest이다.

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: myapp
  labels:
    app: myapp
---
apiVersion: apps/v1
kind: Deployment
metadata:
  creationTimestamp: null
  labels:
    app: myapp-backend
  name: myapp-backend
  namespace: myapp
spec:
  replicas: 1
  selector:
    matchLabels:
      app: myapp-backend
  strategy: {}
  template:
    metadata:
      creationTimestamp: null
      labels:
        app: myapp-backend
    spec:
      containers:
      - image: move02/myapp-backend:beta.v1
        name: myapp-backend
        ports:
          - containerPort: 8080
        resources: {}
      imagePullSecrets:
        - name: docker-auth
status: {}
---
apiVersion: v1
kind: Service
metadata:
  creationTimestamp: null
  labels:
    app: myapp-backend
  name: myapp-backend
  namespace: myapp
spec:
  ports:
  - name: http
    port: 80
    protocol: TCP
    targetPort: 8080
  selector:
    app: myapp-backend
  type: ClusterIP
status:
  loadBalancer: {}
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  annotations:
    kubernetes.io/ingress.class: "nginx"
    ingress.kubernetes.io/ssl-passthrough: "true"
    kubernetes.io/ingress.allow-http: "true"
    nginx.ingress.kubernetes.io/rewrite-target: /$2
  name: backend-ingress
  namespace: myapp
spec:
  rules:
    - http:
        paths:
        - path: /app(/|$)(.*)
          pathType: Prefix
          backend:
            service:
              name: myapp-backend
              port: 
                number: 80
```

myapp 라는 namespace를 만들고 Deployment, Service, Ingress의 스펙을 정의하였다. Deployment에 컨테이너 레지스트리를 따로 설정하지 않으면 dockerhub에서 가져오며 프라이빗 이미지일 경우 인증용 Secret이 따로 필요하다. (Secret을 만드는 방법은 [공식문서 참고](https://kubernetes.io/ko/docs/tasks/configure-pod-container/pull-image-private-registry/)) 

이렇게 manifest를 직접 작성해서 애플리케이션 배포에 필요한 오브젝트를 생성할 수도 있지만, 배포할 애플리케이션의 복잡도가 높고 클러스터 운영 환경이 다양해질 경우(ex. 개발, 테스트, 프로덕션 등) 환경에 맞게 변경해야 할 설정이 다소 존재한다.(포트, ingress 룰, 레지스트리 등) 이럴때, 긴 파일을 다 찾아가며 값을 변경하기에는 번거롭다는 생각이 들었다.

## Helm

Helm 쿠버네티스 애플리케이션을 설치하고 관리하게 해주는 툴이다. `kubectl`로도 가능한 일들이지만 조금 더 관리가 쉬워지며 운영체제의 yum, apt 등과 비슷한 포지션이라고 한다.

기본적으로 Helm Chart 라는 것을 이용해서 설치할 애플리케이션을 설치하고 업그레이드 하는데, 원격 repository를 등록하여 다양한 커뮤니티에서 만든 패키징된 애플리케이션을 손쉽게 설치할 수도 있다. 

최신 버전을 이용하다 보니 Helm v3를 설치해서 사용하게 되었고, 이전까지는 쿠버네티스 클러스터에 Tiller 라는 서버사이드 애플리케이션을 배포하여 클라이언트/서버 구조로 운영하였지만 v3부터는 완전한 클라이언트/라이브러리 구조로 바뀌었다고 한다. 

### Helm chart

Helm을 이용할 때 가장 핵심이 되는 개념이다. 쿠버네티스 리소스와 관련된 셋을 설명하는 파일의 모음이며, 단순하게 Pod 하나만 배포할 수도 있고 복잡한 웹앱 형태로 애플리케이션을 배포할 수도 있다.

차트는 특정한 디렉터리 구조를 가진 파일들로 생성되며, 버전이 지정된 아카이브로 묶어서 배포할 수도 있고 배포된 차트를 끌어와서 바로 설치할 수도 있다.
Helm이 패키지 관리에 더 용이하다는 점이 바로 이 점이다. 한 번 차트를 만들어 Helm repository에 배포해두면 배포된 차트를 설치할 때 단 한 줄의 명령어만 입력하면 된다.

아래는 쿠버네티스 클러스터를 모니터링하는 de-facto 표준인 프로메테우스를 Helm으로 설치하는 명령어와 설치 결과이다.  

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
# 여러가지 차트가 있지만, kube-state-metrics나 node-exporter, grafana 같은 모듈을 모두 한 번에 설치해주기 때문에 처음 설치한다면 이게 가장 편할 듯 함.
# 모든 차트 종류를 보려면 https://github.com/prometheus-community/helm-charts/tree/main/charts 방문
helm install prometheus prometheus-community/kube-prometheus-stack
```

![prometheus-stack-install-through-helm](/img/kube/helm-installed-prometheus-stack.png)

단 세 줄의 명령어로 저렇게 많은 오브젝트가 설치되었고 모니터링 대시보드 툴인 Grafana도 같이 설치되어 모니터링이 제대로 동작하는걸 확인할 수 있었다. 만약 설치 전에 기본 설정을 변경하고 싶다면 아무 이름으로 yaml파일을 만들고 차트의 values.yaml 중 수정하고싶은 속성만 지정하여 다시 작성하면 된다.

[예시 차트](https://github.com/prometheus-community/helm-charts/blob/main/charts/kube-prometheus-stack)를 통해 함께 설치되는 grafana의 관리자 비밀번호를 변경하고 싶다면 grafana.yaml 파일을 만들고 

```yaml
# grafana.yaml
grafana:
  adminPassword: mypassword
``` 

이런식으로 내용을 작성한 후

```bash
helm install -f </path/to/grafana.yaml> prometheus prometheus-community/kube-prometheus-stack
```

위의 명령어를 통해 커스텀한 설정을 반영하여 설치하면 된다. 

> 모니터링 툴에 대한 자세한 이야기는 다른 일지에서 다룰 예정입니다.

### Helm vs Manifest

Manifest에 비해 Helm이 다른 사람들이 배포한 애플리케이션을 설치하는 데에는 훨씬 간단하다는 것은 prometheus 설치 과정 예시를 통해 와닿았을 것이다. 그렇다면 직접 만든 애플리케이션을 배포하는 것은 어떨까?

[Manifest](#manifest) 섹션에서 작성했던 myapp 백엔드 애플리케이션 배포를 위해 작성한 오브젝트 들을(네임스페이스 제외) Helm chart로 만들어 배포해보자.

먼저 helm 차트를 만들기 위한 구조를 만들어야 한다.

```sh
# 기존에 설치해둔 애플리케이션과 이름이 겹치지 않도록 h를 붙였다
helm create myapp-h
```

위 명령어를 실행하면 다음과 같은 파일들이 생성된다.
![helm-chart-structure](/img/kube/helm-create.png)

아래는 helm 공식문서에 나와있는 차트파일 구조에 대한 설명이다
```
Chart.yaml          # 차트에 대한 정보를 가진 YAML 파일
LICENSE             # 옵션: 차트의 라이센스 정보를 가진 텍스트 파일
README.md           # 옵션: README 파일
values.yaml         # 차트에 대한 기본 환경설정 값들
values.schema.json  # 옵션: values.yaml 파일의 구조를 제약하는 JSON 파일
charts/             # 이 차트에 종속된 차트들을 포함하는 디렉터리
crds/               # 커스텀 자원에 대한 정의
templates/          # values와 결합될 때, 유효한 쿠버네티스 manifest 파일들이 생성될 템플릿들의 디렉터리
templates/NOTES.txt # 옵션: 간단한 사용법을 포함하는 텍스트 파일
```

설명과 완전히 동일하지 않지만 얼추 비슷하게 생성이 되었다. (crds는 쿠버네티스의 커스텀 리소스를 정의하는 부분이라고 한다)

생성된 차트파일의 내용을 바꾸어서 SpringBoot 애플리케이션을 실행하는 컨테이너가 배포되고 이를 클러스터에서 서비스할 수 있도록 만들어야한다. 

templates 안의 파일들은 Jinja 템플릿엔진을 사용하여 values.yaml이나 차트 release name 같은 외부 값을 참조하여 동적으로 manifest가 만들어지도록 되어있다. 
myapp 백엔드 같은 경우에는 templates/deployment.yaml 파일의 `spec.template.spec.containers` 속성 중 ports, livenessProbe, readinessProbe를 커스텀하기 위해 아래와 같이 변경하였다.

```yaml
# templates/deployment.yaml
# ...생략...
containers:
  - name: {{ .Chart.Name }}
    securityContext:
      {{- toYaml .Values.securityContext | nindent 12 }}
    image: "{{ .Values.image.repository }}:{{ .Values.image.tag | default .Chart.AppVersion }}"
    imagePullPolicy: {{ .Values.image.pullPolicy }}
    # 커스텀 시작 지점
    {{- with .Values.containerPorts }}
    ports:
      {{- toYaml . | nindent 10 }}
    {{- end }}
    livenessProbe: 
      httpGet: 
        path: {{ .Values.livenessProbe.path }}
        port: {{ .Values.livenessProbe.port }}
      initialDelaySeconds: {{ .Values.livenessProbe.initialDelaySeconds }}
    readinessProbe:
      httpGet:
        path: {{ .Values.livenessProbe.path }}
        port: {{ .Values.livenessProbe.port }}
      initialDelaySeconds: {{ .Values.livenessProbe.initialDelaySeconds }}
    # 커스텀 끝 지점
    resources:
      {{- toYaml .Values.resources | nindent 12 }}
# ...생략...
```

해당 설정은 [Spring boot helm starter](https://github.com/alexandreroman/spring-boot-helm-starter) 깃허브 레포를 참고하여 변경하였다. (deployment.yaml 템플릿 외에도 ConfigMap 같은 다른 오브젝트도 추가하는 것이 운영 환경에서는 좋아보임)

이렇게 커스텀이 완료된 템플릿에 value를 씌워야하니 values.yaml 파일에 커스터마이징이 필요한 설정값을 채워넣는다.

```yaml

# ... 생략 ...

replicaCount: 1

image:
  repository: move02/myapp-backend
  pullPolicy: IfNotPresent
  # Overrides the image tag whose default is the chart appVersion.
  tag: beta.v1

imagePullSecrets: 
  - name: docker-auth

# ... 생략 ...

containerPorts: 
  - name: http
    containerPort: 8080
    protocol: TCP
livenessProbe: 
  path: /actuator/health
  port: http
  initialDelaySeconds: 60
readinessProbe: 
  path: /actuator/health
  port: http
  initialDelaySeconds: 20

service:
  type: ClusterIP
  port: 80

# ... 생략 ...
```

중간중간 변경하지 않은 부분은 생략하였다. 당장 보기에는 기존에 작성했던 manifest보다 복잡해보이기는 하지만 한 번 만들어두면 여러 환경에 배포를 할 때 prometheus 설치할 때처럼 환경마다 변경이 필요한 설정만 바꾸면 쉽게 변경이 가능하다.

staging 환경이 아니라 production 환경에 맞는 이미지로 빌드하는 경우를 예로 들어 설정을 변경해보자. 
안정성을 위해 replica의 갯수를 3으로 늘리고 image의 태그를 latest로 변경하고자 할 때 아래와 같이 파일을 작성한 후 명령어를 입력하면 된다.

```yaml
# myapp-production-settings.yaml
replicaCount: 3
image:
  tag: latest
```


```sh
helm install -f myapp-production-settings.yaml myapp-api-production ./myapp-h 
```

실제로 한 번 차트를 만들어두면 다른 환경에 배포하거나 업그레이드를 할 때는

1. 변경할 속성과 값만 정의하여 파일 만듦
2. 해당 파일을 반영하여 install 또는 upgrade (파일을 만들지 않고도 커맨드 라인의 --set 옵션을 이용해 간단하게 반영도 가능) 

이 두 가지 과정만 거쳐도 된다. helm install 또는 upgrade의 자세한 명령어는 Helm 공식 문서 참고.

> 많은 설정들이 생략되었고 실제 운영환경에 서비스를 배포하는 단계는 아니기에 예제를 최대한 가볍게 만들었다.

## myapp 데이터 서버 애플리케이션 배포 방침

1. 외부 애플리케이션의 설치는 가능한 Helm을 통해 설치할 것
   1. values의 변경이 필요한 경우 반드시 변경 되는 값을 파일로 저장하여 반영  
   2. 지원하는 repository가 없을 경우에만 manifest를 만들고 note.txt나 readme를 꼭 남길 것.
2. 직접 애플리케이션을 빌드하여 설치 또는 배포하는 경우
   1. helm 차트를 생성하여 파일의 형태로 관리자에게 공유
   2. 어렵다면 빌드된 docker image와 애플리케이션 실행 환경(서비스 포트, 프레임워크 정보, health 체크 경로 등) 관리자에게 공유


## 참고

- [Helm 공식문서](https://helm.sh/ko/docs/)
- [bitnami 문서](https://docs.bitnami.com/tutorials/create-your-first-helm-chart/)