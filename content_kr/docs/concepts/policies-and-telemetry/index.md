---
title: 정책과 원격 측정
description: 정책 적용과 원격 측정 메커니즘을 설명합니다.
weight: 40
keywords: [policies,telemetry,control,config]
aliases:
    - /docs/concepts/policy-and-control/mixer.html
    - /docs/concepts/policy-and-control/mixer-config.html
    - /docs/concepts/policy-and-control/attributes.html
    - /docs/concepts/policies-and-telemetry/overview/
    - /docs/concepts/policies-and-telemetry/config/
---

Istio는 메쉬에 있는 서비스에 대해 인가 정책을 적용하고 원격 측정 값을 수집하기 위한 유연한 모델을 제공합니다.

인프라스트럭처 백엔드는 서비스를 만드는데 필요한 지원 기능을 제공하도록 설계되어 있습니다.
이러한 지원 기능에는 접근 제어 시스템, 원격 측정 수집 시스템, 할당량 강제 시스템, 요금 고지 시스템 등이 있습니다.
전통적으로는 서비스가 직접 이런 백엔드 시스템과 통합해 강한 결합을 만들고 특수한 문법과 옵션을 적용해야 했습니다.

Istio는 균등한 추상화를 제공해 확장 가능한 인프라스트럭처의 집합과 Istio가 연결되는 것을 가능하게 했습니다.
이런 기능은 서비스 개발자에게 부담이 되지 않으면서 운영자가 깊은 제어를 가질 수 있는 방법으로 되었습니다.
Istio는 계층 사이의 경계를 바꿔 시스템의 복잡도를 줄이고 정책 논리를 서비스 코드에서 제거하고 운영자에게 제어권을 주도록 설계되었습니다.

Mixer는 정책 제어와 원격 측정 값 수집을 담당하는 Istio의 컴포넌트입니다:

{{< image width="55%" ratio="69.79%"
    link="./topology-without-cache.svg"
    caption="Mixer Topology"
    >}}

Envoy sidecar는 논리적으로 각 요청 전 선조건 검사와 요청 후 원격 측정 치 보고를 위해 Mixer를 부릅니다.
sidecar는 대부분의 선조건 검사가 캐시에서 이루어질 수 있도록 로컬 캐시를 가지고 있습니다.
추가적으로 sidecar는 내보내는 원격 측정 치를 버퍼링하여 Mixer를 자주 부르지 않도록 되어있습니다.

Mixer는 다음과 같은 기능을 제공합니다:

* **Backend 추상화**. Mixer는 Istio의 나머지 부분을 각 인프라스트럭처 백엔드의 구현으로부터 격리합니다.

* **중재**. Mixer는 운영자가 메쉬와 인프라스트럭처 백엔드 사이의 상호작용에 대한 세부적인 제어를 할 수 있게 해줍니다.

이런 기능적인 부분에 더해 Mixer는 아래에 소개된 [신뢰성과 확장성](#reliability-and-latency)도 가지고 있습닏.

정책 적용과 원격 측정 치 수집은 설정으로 관리됩니다.
Istio 배포에서 Mixer를 실행하는 것을 피하기 위해 [기능을 완전히 비활성화](/docs/setup/kubernetes/helm-install/#customization-example-traffic-management-minimal-set) 시키는 것도 가능합니다.

## 어댑터<sub>Adapter</sub>

Mixer는 고도로 모듈화되고 확장 가능한 컴포넌트입니다.
핵심 기능 중 하나는 다른 정책과 원격 측정 백엔드 시스템의 세부 사항을 추상화 시켜 Istio의 나머지 요소들이 이런 백엔드에 관계 없이 실행될 수 있도록 하는 것입니다.

Mixer의 여러 인프라스트럭처 백엔드에 대한 유연성은 Mixer의 다용도 플러그인 모델으로부터 나옵니다.
이런 플러그인은 *어댑터*로 알려져 있으며 어댑터는 Mixer가 로깅, 모니터링, 할당량, ACL 검사 등과 같은 핵심 기능을 제공하는 여러 인프라스트럭처 백엔드와 교류할 수 있게 해 줍니다.
실행 시간에 사용될 정확한 어댑터의 집합은 설정에 의해 결졍되고 쉽게 새로운 인프라스트럭처 백엔드나 사용자 지정 인프라스트럭처 백엔드를 지원하기 위해 확장될 수 있습니다.

{{< image width="80%" ratio="69.79%"
    link="./adapters.svg"
    alt="Showing Mixer with adapters."
    caption="Mixer and its Adapters"
    >}}

[지원되는 어댑터 집합](/docs/reference/config/policy-and-telemetry/adapters/)에 대해 더 알아보세요.

## 신뢰성<sub>Reliability</sub>과 지연<sub>latency</sub>

Mixer는 메쉬에 있는 서비스의 가용성을 높이고 평균 지연 시간을 줄이는데 도움을 주는 설계를 가진 높은 가용성을 지닌 컴포넌트입니다.
Mixer의 설계가 주는 핵심 기능은 다음과 같습니다:

* **무상태<sub>Statelessness</sub>**. Mixer는 자신이 관리하는 영속적인 저장소가 없으므로 상태가 없습니다.

* **Hardening**. Mixer proper는 높은 신뢰성을 가지도록 설계되었습니다. 설계 목적은 모든 Mixer 인스턴스가 > 99.999%의 가동 시간을 가질 수 있게 하는 것이었습니다.

* **캐싱과 버퍼링**. Mixer는 대용량의 일시적인 수명이 짧은 상태를 축적할 수 있도록 설계되었습니다.

메쉬 안에 있는 서비스의 옆에 있는 sidecar proxy는 적은 메모리를 사용해야 해서 로컬 캐시와 버퍼링의 양을 제한합니다.
하지만 Mixer는 독자적으로 살아 보다 큰 캐시와 출력 버퍼를 가질 수 있습니다
그러므로 Mixer는 고 확장성을 지니고 고 가용성을 지닌 sidecar의 2단계 캐시로서 동작할 수 있습니다.

{{< image width="65%" ratio="65.89%"
    link="./topology-with-cache.svg"
    caption="Mixer Topology"
    >}}

Mixer의 기대 가용성이 종종 99.9%의 가용성을 가지기도 하는 대부분의 인프라스트럭처 백엔드보다 꽤 높기 때문에, Mixer의 로컬 캐시와 버퍼는 지연 시간을 줄일 뿐만 아니라 응답을 하지않는 인프라스트럭처 백엔드의 실패를 캐시와 버퍼를 이용해 지속적으로 동작하며 감추는 데에도 사용됩니다.

마지막으로, Mixer의 캐시와 버퍼링은 백엔드에 대한 요청의 빈도를 줄이고 때때로는 로컬 집계를 통해 백엔드에 전달되는 데이터를 줄입니다.
몇몇 상황에서 이런 기능들은 운영적인 비용을 줄일 수 있습니다.

## 속성<sub>Attribute</sub>

속성은 Istio의 정책과 원격 측정 기능에 있어 핵심이 되는 개념입니다.
속성은 서비스 요청이나 요청을 위한 환경을 설명하는 작은 데이터 입니다
예를 들면 속성은 어떤 요청의 크기나 어떠한 작업의 응답 코드, 요청이 시작된 곳의 IP 주소 등일 수 있습니다.

각 속성은 이름과 타입을 가지고 있습니다.
타입은 속성이 가지고 있는 데이터의 종류를 정의합니다.
예를 들면 속성이 `STRING` 타입을 가지는 것은 문자열 값을 가진다는 것을 의미하며 `INT64` 타입을 가지는 것은 64 비트 정수 값을 가진다는 것을 의미합니다.

여기 몇가지 속성과 알맞은 값의 예가 있습니다:

{{< text plain >}}
request.path: xyz/abc
request.size: 234
request.time: 12:34:56.789 04/17/2017
source.ip: 192.168.0.1
destination.service: example
{{< /text >}}

Mixer는 기본적으로 속성 처리 기계입니다.
Envoy sidecar는 각 요청마다 Mixer를 불러 요청과 요청의 환경을 설명하는 속성의 집합을 전달합니다.
Mixer의 설정과 주어진 속성 집합에 따라 Mixer는 여러 인프라스트럭처 백엔드에 요청을 보냅니다.

{{< image width="60%" ratio="42.60%"
    link="./machine.svg"
    caption="Attribute Machine"
    >}}

### 속성 어휘

Istio 배포는 고정된 어휘의 속성만을 이해할 수 있습니다.
이 특수한 어휘는 배포된 환경에서 사용하는 속성 생산자의 집합에 따라 결정됩니다.
특화된 Mixer 어댑터도 속성을 생산할 수 있지만, Istio의 기본 속성 생산자는 Envoy입니다.

[대부분의 Istio 환경에서 사용할 수 있는 일반적인 기본 속성 집합](/docs/reference/config/policy-and-telemetry/attribute-vocabulary/)에 대해 여기에서 더 알아보세요.

### 속성 표현

속성 표현은 [인스턴스](#인스턴스s)를 설정할 때 사용됩니다.
여기 속성 표현의 예제가 있습니다:

{{< text yaml >}}
destination_service: destination.service
response_code: response.code
destination_version: destination.labels["version"] | "unknown"
{{< /text >}}

콜론의 오른쪽에 있는 문자열이 가장 간단한 속성 표현들입니다.
처음 두 개의 표현은 속성 이름만으로 이루어져 있습니다.
`response_code` 라벨은 `response.code` 속성의 값을 가져옵니다.

여기 조건 표현의 예제가 있습니다:

{{< text yaml >}}
destination_version: destination.labels["version"] | "unknown"
{{< /text >}}

위의 예제에서 `destination_version` 라벨은 `destination.labels["version"]`의 값을 가져옵니다.
하지만 그 속성이 없다면 `"unknown"` 문자열이 사용됩니다.

더 자세한 정보는 [속성 표현](/docs/reference/config/policy-and-telemetry/expression-language/)를 참고하세요.

## 설정 모델

Istio의 정책과 원격 측정 기능은 운영자가 인가 정책과 원격 측정 값 수집의 모든 면에 대한 제어를 할 수 있게 하는 일반적인 모델을 통해 설정됩니다.
Istio의 많은 기능을 제어할 수 있을 만큼 강력하면서도 모델을 간단히 만들기 위해 노력하였습니다.

정책 제어와 원격 측정 기능은 세 가지의 자원을 설정하는 것으로 이루어집니다:

* 어떤 어댑터의 집합이 사용되고 어떻게 동작하는 지를 결정하는 *핸들러*의 집합을 설정합니다.
Statsd 백엔드의 IP 주소를 명시한 `statsd` 어댑터는 핸들러 설정의 한 예입니다.

* 요청의 속성을 어떻게 어댑터의 입력으로 변경할 지를 결정하는 *인스턴스*의 집합을 설정합니다.
인스턴스는 하나 또는 그 이상의 어댑터가 사용하는 데이터의 덩어리를 나타냅니다.
예를 들면 운영자가 `destination.service`나 `response.code`와 같은 속성으로부터 `requestcount` 메트릭 인스턴스를 만들어내고자 할 수 있습니다.

* 어떤 인스턴스가 주어지고 어떤 어댑터가 불릴지를 결정하는 *규칙*의 집합을 설정합니다.
규칙은 *match* 표현과 *actions*으로 구성되어 있습니다. match 표현은 어댑터를 언제 부를 지 결정하고 actions은 어떤 인스턴스를 어댑터에게 전달할 지 결정합니다.
예를 들면 어떤 규칙은 생성된 `requestcount` 메트릭 인스턴스를 `statsd` 어댑터에게 전달하도록 설정될 수 있습니다.

설정은 *어댑터*와 *템플릿*으로 이루어져 있습니다:

* **어댑터**은 Mixer와 특정한 인프라스트럭처 백엔드가 교류하는데 필요한 논리를 캡슐화 합니다.

* **템플릿**은 요청의 속성에서 어댑터의 입력으로 변환하는 스키마를 정의합니다.
한 어댑터는 여러 템플릿을 지원할 수 있습니다.

### 핸들러

어댑터는 [Prometheus](https://prometheus.io)나 [Stackdriver](https://cloud.google.com/logging)와 같은 특정한 인프라스트럭처 백엔드와 Mixer 교류하는데 필요한 논리를 캡슐화 합니다.
각 어댑터는 일반적으로 운영적인 인자를 요구합니다.
예를 들면 로깅 어댑터는 로그 수집 백엔드의 IP 주소와 포트를 필요로할 수 있습니다.

여기에 `listchecker` 종류의 어댑터를 설정하는 방법을 나타내는 예제가 있습니다.
`listchecker` 어댑터는 입력 값을 특정 목록에 대해 확인합니다.
만약 어댑터가 화이트리스트로 설정되었다면, 값이 리스트에서 발견되면 성공을 반환합니다.

{{< text yaml >}}
apiVersion: config.istio.io/v1alpha2
kind: listchecker
metadata:
  name: staticversion
  namespace: istio-system
spec:
  providerUrl: http://white_list_registry/
  blacklist: false
{{< /text >}}

`spec` 스탠자에 있는 데이터의 스키마는 설정되는 어댑터의 종류에 따라 달라집니다.

어떤 어댑터는 Mixer와 backend를 연결하는 것 이상의 기능을 구현하기도 합니다.
예를 들면 `prometheus` 어댑터는 메트릭을 소비하고 집계하여 분포나 개수를 셀 수 있도록 설정할 수 있습니다.

{{< text yaml >}}
apiVersion: config.istio.io/v1alpha2
kind: prometheus
metadata:
  name: handler
  namespace: istio-system
spec:
  metrics:
  - name: request_count
    instance_name: requestcount.metric.istio-system
    kind: COUNTER
    label_names:
    - destination_service
    - destination_version
    - response_code
  - name: request_duration
    instance_name: requestduration.metric.istio-system
    kind: DISTRIBUTION
    label_names:
    - destination_service
    - destination_version
    - response_code
    buckets:
      explicit_buckets:
        bounds: [0.005, 0.01, 0.025, 0.05, 0.1, 0.25, 0.5, 1, 2.5, 5, 10]
{{< /text >}}

각 어댑터는 자신을 설정하는 데이터의 형식을 정의합니다
[어댑터의 종류와 어댑터의 설정 형식](/docs/reference/config/policy-and-telemetry/adapters/)을 살펴보세요.

### 인스턴스

인스턴스 설정은 요청의 속성을 어댑터의 입력 값으로 변환하는 작업을 정의합니다.
다음은 `requestduration`을 만드는 메트릭 인스턴스 설정의 예입니다.

{{< text yaml >}}
apiVersion: config.istio.io/v1alpha2
kind: metric
metadata:
  name: requestduration
  namespace: istio-system
spec:
  value: response.duration | "0ms"
  dimensions:
    destination_service: destination.service | "unknown"
    destination_version: destination.labels["version"] | "unknown"
    response_code: response.code | 200
  monitored_resource_type: '"UNSPECIFIED"'
{{< /text >}}

핸들러의 설정에서 기대하는 모든 차원은 매핑에 지정된 것을 살펴보세요.
템플릿은 각 인스턴스가 요구하는 필수 내용을 정의합니다.
[템플릿의 종류와 종류별 설정 형식](/docs/reference/config/policy-and-telemetry/templates/)에 대해 여기에서 더 자세히 알아보세요.

### 규칙

규칙은 어떤 핸들러가 어떤 인스턴스와 함께 불리는지 정의합니다.
목적지 서비스가 `service1`이고 `x-user` 요청 헤더가 특별한 값일 때  `requestduration` 메트릭을 `prometheus` 핸들러에 전달하고자 하는 예제를 생각해 봅시다.

{{< text yaml >}}
apiVersion: config.istio.io/v1alpha2
kind: rule
metadata:
  name: promhttp
  namespace: istio-system
spec:
  match: destination.service == "service1.ns.svc.cluster.local" && request.headers["x-user"] == "user1"
  actions:
  - handler: handler.prometheus
    instances:
    - requestduration.metric.istio-system
{{< /text >}}

규칙이 `match`에 조건식 표현을 가지고 있고 그 조건식이 참일 때 수행할 action의 목록을 정의하고 있습니다.
action은 핸들러에 전달될 인스턴스의 목록을 지정합니다.
규칙은 핸들러와 인스턴스의 전체 이름을 사용해야만 합니다.
규칙, 핸들러, 인스턴스가 모두 같은 namespace에 있다면, namespace 접미사는 `handler.prometheus`에서 처럼 생략될 수 있습니다.
