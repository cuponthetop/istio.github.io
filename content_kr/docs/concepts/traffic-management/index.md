---
title: 트래픽 관리
description: 트래픽 라우팅과 컨트롤에 연관된 Istio의 여러 기능을 설명합니다.
weight: 20
keywords: [traffic-management]
aliases:
    - /docs/concepts/traffic-management/overview
    - /docs/concepts/traffic-management/pilot
    - /docs/concepts/traffic-management/rules-configuration
    - /docs/concepts/traffic-management/fault-injection
    - /docs/concepts/traffic-management/handling-failures
    - /docs/concepts/traffic-management/load-balancing
    - /docs/concepts/traffic-management/request-routing
---

이 페이지에서는 Istio가 어떻게 트래픽을 관리하고 그러한 트래픽 관리 규칙으로 얻는 이득에 대한 개요를 제공합니다.
이 페이지는 당신이 [Istio가 뭔가요?](/docs/concepts/what-is-istio/)를 이미 읽었고 Istio의 상위 단계 아키텍처에 익숙한 것으로 가정합니다.

Istio의 트래픽 관리 모델을 사용하여 트래픽 흐름과 인프라스트럭쳐의 확장을 분리하고, 어떤 pod/VM이 트래픽을 받을 지 지정하는 것이 아닌 트래픽이 흐르는 규칙을 Pilot을 사용하여 지정하면 Pilot과 지능적인 Envoy proxy가 그 뒤를 맡습니다.
예를 들어, 당신은 어떤 서비스에 대한 5%의 트래픽이 canary 버전의 배포 크기에 관계없이 canary 버전으로 향하도록 하거나, 트래픽을 요청의 내용에 따라 특정 버전으로 보내도록 하는 작업을 Pilot을 통해 할 수 있습니다.

{{< image width="85%" ratio="75%"
    link="./TrafficManagementOverview.svg"
    caption="Traffic Management with Istio"
    >}}

트래픽 흐름을 인프라스트럭쳐 확장으로 부터 분리하는 것은 Istio가 어플리케이션 코드 바깥에 존재하는 다양한 트래픽 관리 기능을 제공할 수 있게 해줍니다.
A/B 테스트를 위한 동적 [요청 라우팅](#request-routing)<sub>request routing</sub>, 점진적 롤아웃<sub>gradual rollouts</sub>, canary 릴리즈<sub>canary release</sub> 뿐만 아니라 타임아웃, 재시도, 회로 차단<sub>circuit breaker</sub>기를 사용한 [실패 복구](#handling-failures)<sub>failure recovery</sub>와 서비스간의 실패 복구 정책을 테스트 하기 위한 [결함 주입](#fault-injection)<sub>fault injection</sub>도 다룹니다.
이런 기능은 서비스 메쉬<sub>service mesh</sub>에 걸쳐 배포되어있는 Envoy sidecar/proxy로 실현되었습니다.

## Pilot과 Envoy

Istio에서 트래픽 관리를 위해 사용하는 핵심 컴포넌트는 **Pilot**입니다.
Pilot은 특정 Istio 서비스 메쉬에 배포된 모든 Envoy proxy 인스턴스를 설정하고 관리합니다.
Pilot은 Envoy proxy 사이에 오가는 트래픽의 경로를 변경하기 위해 사용할 규칙을 정할 수 있게 해주고 타임아웃, 재시도, 회로 차단기와 같은 실패 복구 기능을 설정할 수 있게 해 줍니다.
Pilot은 또한 메쉬에 있는 모든 서비스의 원형<sub>canonical model</sub>을 관리하고 이 원형을 사용하여 Envoy 인스턴스가 Pilot의 디스커버리 서비스를 활용하여 다른 Envoy 인스턴스에 대해 알 수 있게 해 줍니다.

각각의 Envoy 인스턴스는 Pilot으로부터 얻는 정보와 인스턴스가 속한 로드밸런싱 풀에 있는 다른 인스턴스에 대한 주기적인 health-check를 바탕으로 [로드밸런싱 정보](#discovery-and-load-balancing)<sub>load balancing information</sub>를 관리합니다.
로드밸런싱 정보를 관리함으로서 각각의 Envoy 인스턴스는 지정된 라우팅 규칙을 준수함과 동시에 지능적으로 트래픽을 목표 인스턴스들에 분산시킬 수 있습니다.

Pilot은 서비스 메쉬에 배포된 Envoy 인스턴스의 생명주기를 관리합니다.

{{< image width="60%" ratio="70%"
    link="./PilotAdapters.svg"
    caption="Pilot Architecture"
    >}}

위 그림에서 볼 수 있듯이, Pilot은 메쉬에 있는 서비스의 원형을 기반이 되는 플랫폼으로부터 독립적으로 관리합니다.
Pilot의 플랫폼 특화된 어댑터는 이 원형을 알맞게 만들어내는 역할을 합니다.
예를 들어, Pilot의 Kubernetes 어댑터는 Kubernetes API 서버의 pod 등록 정보, ingress 자원, 트래픽 관리 규칙을 저장하는 third-party 자원 변화를 감시하는데 필요한 controller를 구현합니다.
이 데이터는 원형 표현<sub>canonical representation</sub>으로 바뀌어집니다.
그런 다음, Envoy 특화적인 설정이 원형 표현으로부터 생성됩니다.

Pilot은 [서비스 디스커버리](https://www.envoyproxy.io/docs/envoy/latest/api-v1/cluster_manager/sds)<sub>service discovery</sub>와 [로드밸런싱 풀](https://www.envoyproxy.io/docs/envoy/latest/configuration/cluster_manager/cds)<sub>load balancing pool</sub>과 [라우팅 테이블](https://www.envoyproxy.io/docs/envoy/latest/configuration/http_conn_man/rds)<sub>routing table</sub>에 대한 동적인 변경을 가능케 합니다.

당신은 [Pilot의 규칙 설정](/docs/reference/config/istio.networking.v1alpha3/)을 통해 고수준의 트래픽 관리 규칙을 지정할 수 있습니다.
이 규칙은 저수준의 설정으로 바뀌어 Envoy 인스턴스로 분배됩니다.

## 요청 라우팅<sub>Request routing</sub>

위에서 설명한 것 처럼, 메쉬에 있는 서비스의 원형 표현은 Pilot이 관리합니다.
Istio의 서비스 모델은 기반이 되는 플랫폼(Kubernetes, Mesos, Cloud Foundry, etc.)에 관계없이 독립적으로 표현됩니다.
플랫폼 특화된 어댑터는 플랫폼의 메타 데이터의 여러 필드를 사용하여 Istio의 내부적 모델 표현을 만들어내는 역할을 합니다.

Istio는 서비스 인스턴스를 버전(`v1`, `v2`)이나 환경(`staging`, `prod`)에 따라 나누는 아주 세분화된 방법으로 _서비스 버전<sub>service version</sub>_ 을 소개합니다.
이런 분화된 버전은 꼭 다른 API 버전일 필요는 없습니다: 이 버전들은 다른 환경(prod, staging, dev, etc.)에 배포된 같은 서비스의 반복적인 변경일 수 있습니다.
서비스 버전의 일반적인 적용 시나리오는 A/B 테스팅이나 canary rollout입니다.
Istio의[트래픽 라우팅 규칙](#rule-configuration)은 서비스 간의 트래픽에 대한 추가적인 제어권을 제공하기 위해 서비스 버전을 사용할 수 있습니다.

### 서비스 간 커뮤니케이션

{{< image width="60%" ratio="100%"
    link="./ServiceModel_Versions.svg"
    alt="Showing how service versions are handled."
    caption="Service Versions"
    >}}

위 그림에서 볼 수 있듯이, 서비스의 클라이언트는 서비스의 다른 버전에 대한 것을 모릅니다.
클라이언트는 계속해서 서비스의 hostname이나 IP 주소를 사용해서 서비스에 접근할 수 있습니다.
Envoy sidecar/proxy가 클라이언트와 서비스 사이의 모든 요청과 응답을 가로채어 전달해줍니다.

Envoy는 당신이 Pilot을 사용하여 지정한 라우팅 규칙에 기반해 동적으로 사용할 서비스 버전을 결정합니다.
이러한 방식은 어플리케이션 코드가 다른 이득은 그대로 두면서([Mixer](/docs/concepts/policies-and-telemetry/)를 보세요) 자손 서비스의 발전으로부터 분리될 수 있게 합니다.
라우팅 규칙은 Envoy가 헤더, 요청의 시작점/도착점에 관련있는 태그, 그리고/또는 각 버전에 지정된 가중치 등의 조건에 따라 버전을 선택할 수 있게 해줍니다.

Istio는 같은 서비스 버전을 가진 다중 인스턴스에 대해 트래픽을 로드 밸런싱 하는 기능 또한 제공합니다.
더 알고 싶다면 [디스커버리과 로드 밸런싱](/docs/concepts/traffic-management/#discovery-and-load-balancing)을 보세요.

Istio는 DNS를 제공하지 않습니다.
어플리케이션은 기반이 되는 플랫폼(`kube-dns`, `mesos-dns`, etc.)이 제공하는 DNS 서비스를 활용하여 FQDN을 풀수 있습니다.

### Ingress와 egress

Istio는 서비스 메쉬의 모든 트래픽이 Envoy proxies 들어오고 나간다고 가정합니다.
서비스 앞에 Envoy proxy를 배포함으로서, 사용자가 직접 사용하는 서비스에 대한 A/B 테스팅을 할 수 있고, canary service를 배포하는 등의 작업을 할 수 있습니다.
유사하게, Envoy sidecar를 통해 트래픽을 외부 웹 서비스(예를 들어, 지도 API나 비디오 API에 접근하기)로 라우팅함으로써, 타임아웃, 재시도, 회로 차단과 같은 실패 복구 기능을 추가할 수 있고 외부 서비스에 대한 연결의 보다 상세한 측정치를 얻을 수 있습니다.

{{< image width="85%" ratio="35.51%"
    link="./ServiceModel_RequestFlow.svg"
    alt="Ingress and Egress through Envoy."
    caption="Request Flow"
    >}}

## 디스커버리<sub>Discovery</sub>과 로드 밸런싱<sub>load balancing</sub>

Istio는 서비스 메쉬에 있는 서비스의 인스턴스들에 걸쳐 트래픽의 부하 균형을 맞춥니다.

Istio는 어플리케이션의 pod/VM을 기록하는 서비스 레지스트리가 존재하는 것을 가정합니다.
Istio는 또한 서비스의 새 인스턴스가 자동으로 그 서비스 레지스트리에 등록되고 건강하지 않은 인스턴스는 자동으로 서비스 레지스트리에서 제거되는 것을 가정합니다.

컨테이너 기반의 어플리케이션에 대해서는 이미 Kubernetes나 Mesos와 같은 플랫폼이 이런 기능을 제공하며, VM 기반의 서비스에 대해서는 많은 솔루션이 있습니다.

Pilot은 서비스 레지스트리로부터 정보를 받아 플랫폼 독립적인 서비스 디스커버리 인터페이스를 제공합니다.
메쉬에 있는 Envoy 인스턴스는 서비스 디스커버리를 행하며 그에 맞게 동적으로 자신의 로드 밸런싱 풀을 갱신합니다.

{{< image width="55%" ratio="80%"
    link="./LoadBalancing.svg"
    caption="Discovery and Load Balancing"
    >}}

위 그림에서 보였듯이, 메쉬에 있는 서비스는 서로를 각각의 DNS 이름으로 접근할 수 있습니다.
서비스를 향하는 모든 HTTP 트래픽은 Envoy를 통해 자동으로 리라우팅됩니다.
Envoy는 로드 밸런싱 풀에 존재하는 인스턴스에 걸쳐 트래픽을 분배합니다.
Envoy가 몇몇 [복잡한 로드 밸런싱 알고리즘](https://www.envoyproxy.io/docs/envoy/latest/intro/arch_overview/load_balancing)을 지원하지만, Istio는 현재 세가지의 로드 밸런싱 모드를 허용합니다:
라운드 로빈<sub>round robin</sub>, 랜덤<sub>random</sub>, 치우친 최소 요청<sub>weighted least request</sub>.

Envoy는 로드 밸런싱에 더해 주기적으로 로드 밸런싱 풀 안에 있는 인스턴스의 상태를 확인합니다.
Envoy는 상태 검사 API 요청에 대한 실패율에 기반해 어떤 인스턴스가 건강한지 아닌지를 판단하기 위해 회로 차단 패턴을 사용합니다.
달리 말해, 어떤 인스턴스에 대한 상태 검사가 미리 지정된 한계점을 넘어서는 만큼 실패하게 되면, 그 인스턴스는 로드 밸런싱 풀에서 내보내지게 될 것입니다.
유사하게, 어떤 인스턴스에 대한 상태 검사가 미리 지정된 한계점을 넘어서는 만큼 성공하게 되면, 그 인스턴스는 로드 밸런싱 풀에 다시 추가될 것입니다.
Envoy의 실패 처리 기능에 대해서는 [실패 처리하기](#handling-failures)에서 더 찾아볼 수 있습니다.

서비스는 상태 검사 요청에 HTTP 503으로 응답하는 방법으로 자발적으로 로드를 흘려낼 수 있습니다.
그런 일이 일어나면 HTTP 503 응답을 보낸 인스턴스는 즉시 요청자의 로드 밸런싱 풀에서 제거될 것입니다.

## 실패 처리하기

Envoy는 어플리케이션의 서비스에서 사용할 수 있는 _사전 정의된<sub>opt-in</sub>_ 특별한 실패 복구 기능을 제공합니다.
기능은 다음과 같습니다:

1. 타임아웃

1. 타임아웃 예산과 재시도 요청간의 유동적인 산발적 간격 조정<sub>jitter</sub>을 포함한 제한된 재시도<sub>Bounded retry</sub>

1. 동시에 발생하는 상류<sub>upstream</sub> 서비스에 대한 연결과 요청의 수 제한

1. 로드 밸런싱 풀의 각 구성원에 대한 적극적(주기적)인 상태 검사

1. 로드 밸런싱 풀의 각 구성원의 인스턴스 마다 적용된 세분화된 회로 차단기(수동적 상태 검사)

이러한 기능은 [Istio의 트래픽 관리 규칙](#rule-configuration)을 사용하여 실행 중에 동적으로 설정 가능합니다.

재시도 사이의 산발적 간격 조정은 재시도가 상류 서비스에 줄 수 있는 부하를 줄여주고, 타임아웃 예산은 서비스에 대한 요청이 예측 가능한 시간 간격 안에 응답(성공/실패)을 받는 것을 보장합니다.

능동적인 상태 검사와 수동적인 상태 검사의 조합(4번과 5번)은 로드 밸런싱 풀에서 건강하지 않은 상태의 인스턴스에 접근할 확률을 최소화 합니다.
Kubernetes나 Mesos와 같은 플랫폼 수준의 상태 검사와 함께 사용되었을 때, 어플리케이션은 건강하지 않은 pod/container/VM가 서비스 메쉬에서 빠르게 내보내져 요청 실패와 그에 따른 지연을 최소화 할 수 있게 보장합니다.

합쳐, 이 기능들이 서비스 메쉬가 실패하는 노드를 견딜 수 있게 하고 지엽적인 실패가 전체 노드의 불안정으로 연결되지 않게 합니다.

### 세부 조정<sub>Fine tuning</sub>

Istio의 트래픽 관리 규칙은 서비스와 버전 별로 다르게 실패 복구 방법의 기본 값을 지정할 수 있게 해 줍니다.
하지만 서비스의 사용자도 [타임아웃](/docs/reference/config/istio.networking.v1alpha3/#HTTPRoute-timeout)과 [재시도](/docs/reference/config/istio.networking.v1alpha3/#HTTPRoute-retries)의 기본값을 특별한 HTTP 헤더를 사용하여 요청 수준의 값을 제공하여 덮어쓸 수 있습니다.
Envoy proxy 구현에서 그 헤더는 각각 `x-envoy-upstream-rq-timeout-ms`과 `x-envoy-max-retries`입니다.

### 실패 처리 FAQ

Q: *Istio에서 실행해도 어플리케이션은 여전히 실패를 처리해야 하나요?*

네. Istio는 메쉬에 있는 서비스의 신뢰성와 가용성을 향상시킵니다.
하지만, **어플리케이션은 실패(에러)를 처리하고 그에 알맞는 대책을 수행하여야 합니다**.
예를 들어, 로드 밸런싱 풀의 모든 인스턴스가 실패한다면 Envoy는 HTTP 503을 반환할 것입니다.
상류 서비스로부터 받은 HTTP 503을 처리하는 대비 로직을 구현하는 것은 어플리케이션의 역할입니다.

Q: *Envoy의 실패 복구 기능이 이미 실패 내성 라이브러리<sub>fault tolerance libraries</sub>(예를 들면 [Hystrix](https://github.com/Netflix/Hystrix))를 사용하고 있는 어플리케이션이 동작하지 않게 하나요?*

아니오. 어플리케이션은 Envoy를 볼 수 없습니다.
Envoy가 반환하는 실패 응답은 요청한 상류 서비스의 실패 응답과 같아 구분할 수 없을 것입니다.

Q: *어플리케이션 수준의 라이브러리와 Envoy가 함께 사용되면 실패는 어떻게 처리되나요?*

목표 서비스에 대해 두 가지의 실패 복구 정책이 있다면, **실패가 일어날 때 더 제한적인 정책이 발동되게 됩니다**.
예를 들어 Envoy에서 설정된 타임아웃과 어플리케이션의 라이브러리에서 설정된 타임아웃, 두 개의 타임아웃이 있다고 합시다.
이 경우에 어플리케이션이 서비스에 대한 API 요청의 타임아웃을 5초로 지정하고 Envoy는 10초로 지정되어 있다면, 어플리케이션의 타임아웃이 먼저 발동됩니다.
유사하게, Envoy의 회로 차단기가 어플리케이션의 회로 차단기보다 먼저 발동하면, 그 서비스에 대한 API 요청은 Envoy로부터 503을 받게 될 것입니다.

## 실패 주입<sub>Fault injection</sub>

While the Envoy sidecar/proxy provides a host of [failure recovery mechanisms](#handling-failures) to services running on Istio, it is still imperative to test the end-to-end failure recovery capability of the
application as a whole.
Misconfigured failure recovery policies (for example, incompatible/restrictive timeouts across service calls) could result in continued unavailability of critical services in the application, resulting in poor user experience.

Istio enables protocol-specific fault injection into the network, instead of killing pods or delaying or corrupting packets at the TCP layer.
The rationale is that the failures observed by the application layer are the same regardless of network level failures, and that more meaningful failures can be injected at the application layer (for example, HTTP error codes) to exercise the resilience of an application.

You can configure faults to be injected into requests that match specific conditions.
You can further restrict the percentage of requests that should be subjected to faults.
Two types of faults can be injected: delays and aborts.
Delays are timing failures, mimicking increased network latency, or an overloaded upstream service.
Aborts are crash failures that mimic failures in upstream services.
Aborts usually manifest in the form of HTTP error codes or TCP connection failures.

## Rule configuration

Istio provides a simple configuration model to control how API calls and layer-4 traffic flow across various services in an application deployment.
The configuration model allows you to configure service-level properties such as circuit breakers, timeouts, and retries, as well as set up common continuous deployment tasks such as canary rollouts, A/B testing, staged rollouts with %-based traffic splits, etc.

There are four traffic management configuration resources in Istio:
`VirtualService`, `DestinationRule`, `ServiceEntry`, and `Gateway`:

* A [`VirtualService`](/docs/reference/config/istio.networking.v1alpha3/#VirtualService)
defines the rules that control how requests for a service are routed within an Istio service mesh.

* A [`DestinationRule`](/docs/reference/config/istio.networking.v1alpha3/#DestinationRule)
configures the set of policies to be applied to a request after `VirtualService` routing has occurred.

* A [`ServiceEntry`](/docs/reference/config/istio.networking.v1alpha3/#ServiceEntry) is commonly used to enable requests to services outside of an Istio service mesh.

* A [`Gateway`](/docs/reference/config/istio.networking.v1alpha3/#Gateway)
configures a load balancer for HTTP/TCP traffic, most commonly operating at the edge of the mesh to enable ingress traffic for an application.

For example, you can implement a simple rule to send 100% of incoming traffic for a *reviews* service to version "v1" by using a `VirtualService` configuration as follows:

{{< text yaml >}}
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: reviews
spec:
  hosts:
  - reviews
  http:
  - route:
    - destination:
        host: reviews
        subset: v1
{{< /text >}}

This configuration says that traffic sent to the *reviews* service (specified in the `hosts` field) should be routed to the v1 subset of the underlying *reviews* service instances.
The route `subset` specifies the name of a defined subset in a corresponding destination rule configuration.

A subset specifies one or more labels that identify version-specific instances.
For example, in a Kubernetes deployment of Istio, "version: v1" indicates that only pods containing the label "version: v1" will receive traffic.

In a `DestinationRule`, you can then add additional policies. For example, the following definition specifies to use the random load balancing mode:

{{< text yaml >}}
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: reviews
spec:
  host: reviews
  trafficPolicy:
    loadBalancer:
      simple: RANDOM
  subsets:
  - name: v1
    labels:
      version: v1
  - name: v2
    labels:
      version: v2
{{< /text >}}

Rules can be configured using the `kubectl` command.
See the [configuring request routing task](/docs/tasks/traffic-management/request-routing/) for examples.

The following sections provide a basic overview of the traffic management configuration resources.
See [networking reference](/docs/reference/config/istio.networking.v1alpha3/) for detailed information.

### Virtual Services

A [`VirtualService`](/docs/reference/config/istio.networking.v1alpha3/#VirtualService) defines the rules that control how requests for a service are routed within an Istio service mesh.
For example, a virtual service could route requests to different versions of a service or to a completely different service than was requested.
Requests can be routed based on the request source and destination, HTTP paths and header fields, and weights associated with individual service versions.

#### Rule destinations

Routing rules correspond to one or more request destination hosts that are specified in a `VirtualService` configuration.
These hosts may or may not be the same as the actual destination workload and may not even correspond to an actual routable service in the mesh.
For example, to define routing rules for requests to the *reviews* service using its internal mesh name `reviews` or via host `bookinfo.com`, a `VirtualService` could set the `hosts` field as:

{{< text yaml >}}
hosts:
  - reviews
  - bookinfo.com
{{< /text >}}

The `hosts` field specifies, implicitly or explicitly, one or more fully qualified domain names (FQDN).
The short name `reviews`, above, would implicitly expand to an implementation specific FQDN.
For example, in a Kubernetes environment the full name is derived from the cluster and namespace of the `VirtualSevice` (for example, `reviews.default.svc.cluster.local`).

#### Splitting traffic between versions

Each route rule identifies one or more weighted backends to call when the rule is activated.
Each backend corresponds to a specific version of the destination service, where versions can be expressed using _labels_.
If there are multiple registered instances with the specified label(s), they will be routed to based on the load balancing policy configured for the service, or round-robin by default.

For example, the following rule will route 25% of traffic for the *reviews* service to instances with the "v2" label and the remaining 75% of traffic to "v1":

{{< text yaml >}}
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: reviews
spec:
  hosts:
    - reviews
  http:
  - route:
    - destination:
        host: reviews
        subset: v1
      weight: 75
    - destination:
        host: reviews
        subset: v2
      weight: 25
{{< /text >}}

#### Timeouts and retries

By default, the timeout for HTTP requests is 15 seconds, but it can be overridden in a route rule as follows:

{{< text yaml >}}
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: ratings
spec:
  hosts:
    - ratings
  http:
  - route:
    - destination:
        host: ratings
        subset: v1
    timeout: 10s
{{< /text >}}

You can also specify the number of retry attempts for an HTTP request in a route rule.
The maximum number of retry attempts, or the number of attempts possible within the default or overridden timeout period, can be set as follows:

{{< text yaml >}}
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: ratings
spec:
  hosts:
    - ratings
  http:
  - route:
    - destination:
        host: ratings
        subset: v1
    retries:
      attempts: 3
      perTryTimeout: 2s
{{< /text >}}

Note that request timeouts and retries can also be [overridden on a per-request basis](#fine-tuning).

See the [request timeouts task](/docs/tasks/traffic-management/request-timeouts) for an example of timeout control.

#### Injecting faults

A route rule can specify one or more faults to inject while forwarding HTTP requests to the rule's corresponding request destination.
The faults can be either delays or aborts.

The following example introduces a 5 second delay in 10% of the requests to the "v1" version of the *ratings* microservice:

{{< text yaml >}}
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: ratings
spec:
  hosts:
  - ratings
  http:
  - fault:
      delay:
        percent: 10
        fixedDelay: 5s
    route:
    - destination:
        host: ratings
        subset: v1
{{< /text >}}

You can use the other kind of fault, an abort, to prematurely terminate a request. For example, to simulate a failure.

The following example returns an HTTP 400 error code for 10% of the requests to the *ratings* service "v1":

{{< text yaml >}}
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: ratings
spec:
  hosts:
  - ratings
  http:
  - fault:
      abort:
        percent: 10
        httpStatus: 400
    route:
    - destination:
        host: ratings
        subset: v1
{{< /text >}}

Sometimes delay and abort faults are used together. For example, the following rule delays by 5 seconds all requests from the *reviews* service "v2" to the *ratings* service "v1" and then aborts 10% of them:

{{< text yaml >}}
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: ratings
spec:
  hosts:
  - ratings
  http:
  - match:
    - sourceLabels:
        app: reviews
        version: v2
    fault:
      delay:
        fixedDelay: 5s
      abort:
        percent: 10
        httpStatus: 400
    route:
    - destination:
        host: ratings
        subset: v1
{{< /text >}}

To see fault injection in action, see the [fault injection task](/docs/tasks/traffic-management/fault-injection/).

#### Conditional rules

Rules can optionally be qualified to only apply to requests that match some specific condition such as the following:

_1. Restrict to specific client workloads using workload labels_.  For example, a rule can indicate that it only applies to calls from workloads (pods) implementing the *reviews* service:

{{< text yaml >}}
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: ratings
spec:
  hosts:
  - ratings
  http:
  - match:
      sourceLabels:
        app: reviews
    ...
{{< /text >}}

The value of `sourceLabels` depends on the implementation of the service.
In Kubernetes, for example, it would probably be the same labels that are used in the pod selector of the corresponding Kubernetes service.

The above example can also be further refined to only apply to calls from version "v2" of the *reviews* service:

{{< text yaml >}}
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: ratings
spec:
  hosts:
  - ratings
  http:
  - match:
    - sourceLabels:
        app: reviews
        version: v2
    ...
{{< /text >}}

_2. Select rule based on HTTP headers_. For example, the following rule only applies to an incoming request if it includes a custom "end-user" header that contains the string "jason":

{{< text yaml >}}
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: reviews
spec:
  hosts:
    - reviews
  http:
  - match:
    - headers:
        end-user:
          exact: jason
    ...
{{< /text >}}

If more than one header is specified in the rule, then all of the corresponding headers must match for the rule to apply.

_3. Select rule based on request URI_. For example, the following rule only applies to a request if the URI path starts with `/api/v1`:

{{< text yaml >}}
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: productpage
spec:
  hosts:
    - productpage
  http:
  - match:
    - uri:
        prefix: /api/v1
    ...
{{< /text >}}

#### Multiple match conditions

Multiple match conditions can be set simultaneously.
In such a case, AND or OR semantics apply, depending on the nesting.

If multiple conditions are nested in a single match clause, then the conditions are ANDed.
For example, the following rule only applies if the client workload is "reviews:v2" AND the custom "end-user" header containing "jason" is present in the request:

{{< text yaml >}}
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: ratings
spec:
  hosts:
  - ratings
  http:
  - match:
    - sourceLabels:
        app: reviews
        version: v2
      headers:
        end-user:
          exact: jason
    ...
{{< /text >}}

If instead, the condition appear in separate match clauses, then only one of the conditions applies (OR semantics):

{{< text yaml >}}
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: ratings
spec:
  hosts:
  - ratings
  http:
  - match:
    - sourceLabels:
        app: reviews
        version: v2
    - headers:
        end-user:
          exact: jason
    ...
{{< /text >}}

This rule applies if either the client workload is "reviews:v2" OR the custom "end-user" header containing "jason" is present in the request.

#### Precedence

When there are multiple rules for a given destination, they are evaluated in the order they appear in the `VirtualService`, meaning the first rule in the list has the highest priority.

**Why is priority important?** Whenever the routing story for a particular service is purely weight based, it can be specified in a single rule.
On the other hand, when other conditions (such as requests from a specific user) are being used to route traffic, more than one rule will be needed to specify the routing.
This is where the rule priority must be carefully considered to make sure that the rules are evaluated in the right order.

A common pattern for generalized route specification is to provide one or more higher priority rules that match various conditions,
and then provide a single weight-based rule with no match condition last to provide the weighted distribution of traffic for all other cases.

For example, the following `VirtualService` contains two rules that, together, specify that all requests for the *reviews* service that includes a header named "Foo" with the value "bar" will be sent to the "v2" instances.
All remaining requests will be sent to "v1":

{{< text yaml >}}
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: reviews
spec:
  hosts:
  - reviews
  http:
  - match:
    - headers:
        Foo:
          exact: bar
    route:
    - destination:
        host: reviews
        subset: v2
  - route:
    - destination:
        host: reviews
        subset: v1
{{< /text >}}

Notice that the header-based rule has the higher priority.
If it was lower, these rules wouldn't work as expected because the weight-based rule, with no specific match condition, would be evaluated first to route all traffic to "v1", even requests that include the matching "Foo" header.
Once a rule is found that applies to the incoming request, it is executed and the rule-evaluation process terminates.
That's why it's very important to carefully consider the priorities of each rule when there is more than one.

### Destination rules

A [`DestinationRule`](/docs/reference/config/istio.networking.v1alpha3/#DestinationRule) configures the set of policies to be applied to a request after `VirtualService` routing has occurred.
They are intended to be authored by service owners, describing the circuit breakers, load balancer settings, TLS settings, and other settings.

A `DestinationRule` also defines addressable `subsets`, meaning named versions, of the corresponding destination host.
These subsets are used in `VirtualService` route specifications when sending traffic to specific versions of the service.

The following `DestinationRule` configures policies and subsets for the reviews service:

{{< text yaml >}}
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: reviews
spec:
  host: reviews
  trafficPolicy:
    loadBalancer:
      simple: RANDOM
  subsets:
  - name: v1
    labels:
      version: v1
  - name: v2
    labels:
      version: v2
    trafficPolicy:
      loadBalancer:
        simple: ROUND_ROBIN
  - name: v3
    labels:
      version: v3
{{< /text >}}

Notice that multiple policies, default and v2-specific in this example, can be specified in a single `DestinationRule` configuration.

#### Circuit breakers

A simple circuit breaker can be set based on a number of conditions such as connection and request limits.

For example, the following `DestinationRule` sets a limit of 100 connections to *reviews* service version "v1" backends:

{{< text yaml >}}
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: reviews
spec:
  host: reviews
  subsets:
  - name: v1
    labels:
      version: v1
    trafficPolicy:
      connectionPool:
        tcp:
          maxConnections: 100
{{< /text >}}

See the [circuit-breaking task](/docs/tasks/traffic-management/circuit-breaking/) for a demonstration of circuit breaker control.

#### Rule evaluation

Similar to route rules, policies defined in a `DestinationRule` are associated with a particular *host*.
However if they are subset specific, activation depends on route rule evaluation results.

The first step in the rule evaluation process evaluates the route rules in the `VirtualService` corresponding to the requested *host*, if there are any, to determine the subset (meaning specific version) of the destination service that the current request will be routed
to.
Next, the set of policies corresponding to the selected subset, if any, are evaluated to determine if they apply.

**NOTE:** One subtlety of the algorithm to keep in mind is that policies that are defined for specific subsets will only be applied if
the corresponding subset is explicitly routed to.
For example, consider the following configuration as the one and only rule defined for the *reviews* service, meaning there are no route rules in the corresponding `VirtualService` definition:

{{< text yaml >}}
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: reviews
spec:
  host: reviews
  subsets:
  - name: v1
    labels:
      version: v1
    trafficPolicy:
      connectionPool:
        tcp:
          maxConnections: 100
{{< /text >}}

Since there is no specific route rule defined for the *reviews* service, default round-robin routing behavior will apply, which will
presumably call "v1" instances on occasion, maybe even always if "v1" is the only running version.
Nevertheless, the above policy will never be invoked since the default routing is done at a lower level.
The rule evaluation engine will be unaware of the final destination and therefore unable to match the subset policy to the request.

You can fix the above example in one of two ways.
You can either move the traffic policy up a level in the `DestinationRule` to make it apply to any version:

{{< text yaml >}}
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: reviews
spec:
  host: reviews
  trafficPolicy:
    connectionPool:
      tcp:
        maxConnections: 100
  subsets:
  - name: v1
    labels:
      version: v1
{{< /text >}}

Or, better yet, define proper route rules for the service in the `VirtualService` definition.
For example, add a simple route rule for "reviews:v1":

{{< text yaml >}}
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: reviews
spec:
  hosts:
  - reviews
  http:
  - route:
    - destination:
        host: reviews
        subset: v1
{{< /text >}}

Although the default Istio behavior conveniently sends traffic from any source to all versions of a destination service without any rules being set, as soon as version discrimination is desired rules are going to be needed.
Therefore, setting a default rule for every service, right from the start, is generally considered a best practice in Istio.

### Service entries

A [`ServiceEntry`](/docs/reference/config/istio.networking.v1alpha3/#ServiceEntry) is used to add additional entries into the service registry that Istio maintains internally.
It is most commonly used to enable requests to services outside of an Istio service mesh.
For example, the following `ServiceEntry` can be used to allow external calls to services hosted under the `*.foo.com` domain:

{{< text yaml >}}
apiVersion: networking.istio.io/v1alpha3
kind: ServiceEntry
metadata:
  name: foo-ext-svc
spec:
  hosts:
  - *.foo.com
  ports:
  - number: 80
    name: http
    protocol: HTTP
  - number: 443
    name: https
    protocol: HTTPS
{{< /text >}}

The destination of a `ServiceEntry` is specified using the `hosts` field, which can be either a fully qualified or wildcard domain name.
It represents a white listed set of one or more services that services in the mesh are allowed to access.

A `ServiceEntry` is not limited to external service configuration.
It can be of two types: mesh-internal or mesh-external.
Mesh-internal entries are like all other internal services but are used to explicitly add services to the mesh.
They can be used to add services as part of expanding the service mesh to include unmanaged infrastructure (for example, VMs added to a Kubernetes-based service mesh).
Mesh-external entries represent services external to the mesh.
For them, mutual TLS authentication is disabled and policy enforcement is performed on the client-side, instead of on the server-side as it is for internal service requests.

Service entries work well in conjunction with virtual services and destination rules as long as they refer to the services using matching `hosts`.
For example, the following rule can be used in conjunction with the above `ServiceEntry` rule to set a 10s timeout for calls to the external service at `bar.foo.com`:

{{< text yaml >}}
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: bar-foo-ext-svc
spec:
  hosts:
    - bar.foo.com
  http:
  - route:
    - destination:
        host: bar.foo.com
    timeout: 10s
{{< /text >}}

Rules to redirect and forward traffic, to define retry, timeout, and fault injection policies are all supported for external destinations.
Weighted (version-based) routing is not possible, however, since there is no notion of multiple versions of an external service.

See the [egress task](/docs/tasks/traffic-management/egress/) for a more about accessing external services.

### Gateways

A [Gateway](/docs/reference/config/istio.networking.v1alpha3/#Gateway) configures a load balancer for HTTP/TCP traffic, most commonly operating at the edge of the mesh to enable ingress traffic for an application.

Unlike Kubernetes Ingress, Istio `Gateway` only configures the L4-L6 functions (for example, ports to  expose, TLS configuration).
Users can then use standard Istio rules to control HTTP requests as well as TCP traffic entering a `Gateway` by binding a `VirtualService` to it.

For example, the following simple `Gateway` configures a load balancer to allow external HTTPS traffic for host `bookinfo.com` into the mesh:

{{< text yaml >}}
apiVersion: networking.istio.io/v1alpha3
kind: Gateway
metadata:
  name: bookinfo-gateway
spec:
  servers:
  - port:
      number: 443
      name: https
      protocol: HTTPS
    hosts:
    - bookinfo.com
    tls:
      mode: SIMPLE
      serverCertificate: /tmp/tls.crt
      privateKey: /tmp/tls.key
{{< /text >}}

To configure the corresponding routes, you must define a `VirtualService` for the same host and bound to the `Gateway` using the `gateways` field in the configuration:

{{< text yaml >}}
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: bookinfo
spec:
  hosts:
    - bookinfo.com
  gateways:
  - bookinfo-gateway # <---- bind to gateway
  http:
  - match:
    - uri:
        prefix: /reviews
    route:
    ...
{{< /text >}}

See the [ingress task](/docs/tasks/traffic-management/ingress/) for a complete ingress gateway example.

Although primarily used to manage ingress traffic, a `Gateway` can also be used to model a purely internal or egress proxy.
Irrespective of the location, all gateways can be configured and controlled in the same way.
See [gateway reference](/docs/reference/config/istio.networking.v1alpha3/#Gateway) for details.
