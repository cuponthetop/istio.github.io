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

## 결함 주입<sub>Fault injection</sub>

Envoy sidecar/proxy가 Istio에서 실행되는 서비스에 대해 여러 [실패 복구 메커니즘](#handling-failures)을 제공하지만 어플리케이션 전체의 end-to-end 실패 복구 능력에 대한 테스트를 하는 것도 필수이다.
잘못 설정된 실패 복구 정책(예를 들면, 서비스 요청 사이의 호환되지 않거나 제한적인 타임아웃과 같은)은 어플리케이션의 핵심 서비스에 대한 지속된 비가용성을 초래하여 나쁜 사용자 경험을 제공하게 될 수 있다.

Istio는 pod를 죽이거나 TCP 단의 패킷을 지연시키거나 오염시키지 않고 프로토콜 특화된 네트워크 결함 주입을 사용할 수 있게 합니다.
이렇게 하는 이유는 네트워크 수준 실패의 종류에 관계없이 어플리케이션 단에서 관측되는 실패는 같고, (HTTP 에러 코드와 같은) 더 의미있는 실패가 어플리케이션 단에서 주입되어 어플리케이션의 회복을 연습해 볼 수 있기 때문입니다.

특정한 조건에 맞는 요청에 대해 결함이 주입되도록 설정할 수 있습니다.
결함이 주입되는 요청의 비율 또한 지정할 수 있습니다.
두 가지 종류의 결함이 주입될 수 있습니다: 지연과 중단.

지연은 타이밍 실패로, 증가된 네트워크 지연 시간을 흉내내거나 과부하된 상류 서비스를 흉내냅니다.
중단은 상류 서비스의 실패를 흉내내는 크래시 실패<sub>crash failure</sub>입니다.
중단은 주로 HTTP 에러 코드나 TCP 연결 실패의 형태로 나타납니다.

## 규칙 설정<sub>Rule configuration</sub>

Istio는 어플리케이션 배포에서 서비스 사이의 API 요청과 전송 계층의 트래픽 흐름을 제어하기 위한 간편한 설정 모델을 제공합니다.
설정 모델은 회로 차단, 타임아웃, 재시도와 같은 서비스 수준의 속성 설정과 canary rollout, A/B 테스팅, % 기반으로 트래픽이 분리된 staged rollout과 같은 일반적인 지속적 배포<sub>continuous deployment</sub> 작업을 할 수 있게 해줍니다.

Istio에는 네 가지의 트래픽 관리 설정 자원<sub>resource</sub>이 있습니다:
`VirtualService`, `DestinationRule`, `ServiceEntry`, `Gateway`:

* [`VirtualService`](/docs/reference/config/istio.networking.v1alpha3/#VirtualService)는 Istio 서비스 메쉬 내부에서 서비스에 대한 요청이 라우팅 되는 지를 제어하는 규칙을 정의합니다.

* [`DestinationRule`](/docs/reference/config/istio.networking.v1alpha3/#DestinationRule)은 요청에 대해 `VirtualService` 라우팅이 이루어진 뒤에 적용될 정책들을 설정합니다.

* [`ServiceEntry`](/docs/reference/config/istio.networking.v1alpha3/#ServiceEntry)는 Istio 서비스 메쉬의 외부에 있는 서비스에 대한 요청을 가능하게 하는데 주로 쓰입니다.

* [`Gateway`](/docs/reference/config/istio.networking.v1alpha3/#Gateway)는 메쉬의 가장자리에서 동작하여 어플리케이션에 트래픽 유입이 가능하게 하는 HTTP/TCP 트래픽에 대한 로드 밸런서를 설정합니다.

예를 들면, *reviews* 서비스에 유입되는 트래픽의 100%를 "v1" 버전으로 보내는 간단한 규칙을 아래의 `VirtualService` 설정을 사용해 만들 수 있다:

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

이 설정은 (`hosts` 필드에 지정된) *reviews* 서비스로 보내진 트래픽이 *reviews* 서비스 인스턴스 중 v1 subset으로 전달되어야 한다고 표현하고 있습니다.
route 항목의 `subset` 필드가 destination 규칙에서 목적지로 정의된 subset 을 지정합니다.

subset은 버전 특화된 인스턴스를 나타내는 하나 또는 하나 이상의 라벨을 지정합니다.
예로, Istio의 Kubernetes 배포에서, "version: v1"은 "version: v1" 라벨을 가지고 있는 pod만 트래픽을 받는 것은 나타냅니다.

`DestinationRule`으로 정책을 더 추가할 수 있습니다.
예로, 아래의 정의는 랜덤 로드 밸런싱<sub>random load balancing</sub> 모드를 사용하도록 지정합니다:

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

규칙은 `kubectl` 명령어를 사용하여 설정할 수 있습니다.
[요청 라우팅 설정 작업](/docs/tasks/traffic-management/request-routing/)에서 더 많은 예를 볼 수 있습니다.

이어지는 절은 트래픽 관리 설정 자원에 대한 기초적인 개요를 제공합니다.
보다 자세한 정보는 [네트워킹 레퍼런스](/docs/reference/config/istio.networking.v1alpha3/)에서 볼 수 있습니다.

### 가상 서비스<sub>Virtual Service</sub>

[`VirtualService`](/docs/reference/config/istio.networking.v1alpha3/#VirtualService)는 Istio 서비스 메쉬 안의 서비스에 대한 요청이 라우팅되는 규칙을 정의합니다.
예를 들면, 가상 서비스는 요청을 요청한 서비스와 다른 버전의 서비스나 완전히 다른 서비스로 라우팅할 수 있습니다.
요청은 출발지, 도착지, HTTP 경로와 헤더 필드, 그리고 각  서비스 버전에 따른 가중치에 기반하여 라우팅되게 됩니다.

#### 규칙 도착지<sub>Rule destination</sub>

라우팅 규칙은 `VirtualService` 설정에서 지정된 하나 또는 그 이상의 목적지 호스트에 대응됩니다.
이 호스트는 실제 목적지의 작업량과 같을 수도, 같지 않을 수도 있고 메쉬에 있는 실제 라우팅 가능한 서비스에 대응하지 않을 수 조차 있습니다.
예로, *reviews* 서비스의 내부 메쉬 이름인 `reviews`나 호스트인 `bookinfo.com`을 사용하여 요청 라우팅 규칙을 정의하기 위해서 `VirtualService`는  자신의 `hosts` 필드를 아래와 같이 설정할 수 있습니다:

{{< text yaml >}}
hosts:
  - reviews
  - bookinfo.com
{{< /text >}}

`hosts` 필드는 암시적이나 명시적으로 하나 또는 그 이상의 도메인 이름의 절대 표기(FQDN)를 지정합니다.
짧은 이름인 `reviews`는 암시적으로 구현 특화적인 FQDN으로 확장됩니다.
예로, Kubernetes 환경에서는 확장된 이름은 클러스터와 `VirtualService`의 네임스페이스로부터 만들어집니다(`reviews.default.svc.cluster.local`와 같이).

#### 여러 버전에 트래픽 나누기

각각의 라우팅 규칙은 규칙이 적용될 때 부를 하나 또는 그 이상의 가중치를 가진 백엔드를 지정합니다.
각각의 백엔드는 _labels_ 로 표현할 수 있는 목적지 서비스의 한 버전에 대응합니다.
지정된 라벨에 해당하는 인스턴스가 여러 개가 등록되어 있다면, 서비스에 설정된 로드 밸런싱 정책이나 기본 정책인 round-robin 방식으로 라우팅 될 것입니다.

예로, 아래의 규칙은 *reviews* 서비스에 대한 트래픽의 25%를 "v2" 라벨을 지닌 인스턴스에 보내고 나머지 75%의 트래픽을 "v1" 라벨을 지닌 인스턴스에 보냅니다:

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

#### 타임아웃과 재시도

기본으로, HTTP 요청의 타임아웃은 15초이지만 라우팅 규칙에서 다음과 같은 방법으로 덮어쓸 수 있다:

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

HTTP 요청에 대한 재시도 횟수 또한 라우팅 규칙에서 지정할 수 있다.
재시도의 최대 시도 횟수나, 기본 혹은 덮어씌워진 타임아웃 시간 동안 시도 가능한 재시도 횟수는 아래와 같이 설정할 수 있다:

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

요청의 타임아웃과 재시도는 [요청 별로 덮어씌워질](#fine-tuning) 수 있다는 것을 기억하세요.

타임아웃 제어의 예를 보려면 [요청 타임아웃 작업](/docs/tasks/traffic-management/request-timeouts)을 보세요.

#### 결함 주입<sub>Injecting fault</sub>

라우팅 규칙은 하나 또는 그 이상의 HTTP 요청을 규칙에 따른 목적지에 전달하는 과정에 주입할 결함을 지정할 수 있습니다.
결함은 지연이나 중단 중에 하나가 됩니다.

아래의 예제는 *ratings* 마이크로서비스의 "v1" 버전에 대한 요청의 10%에 5초의 지연을 주입합니다:

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

다른 유형의 결함인 중단을 사용해 요청을 (예를 들면, 모의로 실패를 하기 위해) 미성숙한 상태에서 종료할 수 있습니다.

아래의 예제는 *ratings* 서비스의 "v1"에 대한 요청의 10%에 대해 HTTP 400 에러 코드를 반환합니다:

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

종종 지연과 중단이 함께 사용될 수 있습니다.
예로, 아래의 규칙은 the following rule delays by 5 seconds all requests from the *reviews* 서비스의 "v2"로부터 *ratings* 서비스의 "v1"으로 가는 모든 요청을 5초 만큼 지연시키고 그 중 10%의 요청을 중단합니다:

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

결함 주입이 동작하는 것을 보고 싶다면 [결함 주입 작업](/docs/tasks/traffic-management/fault-injection/)을 보세요.

#### 조건부 규칙<sub>Conditional rule</sub>

규칙은 다음과 같은 조건에 맞는 요청에만 적용되도록 선택적으로 적용될 수 있습니다:

_1. 작업량 라벨<sub>workload label</sub>을 사용하여 특정한 클라이언트 작업량을 제한_.
예로, 규칙이 *reviews* 서비스를 구현하는 작업량(pod)으로부터 오는 요청에 대해 적용되는 것을 나타낼 수 있습니다:

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

`sourceLabels`의 값은 서비스의 구현에 따라 달라집니다.
예를 들어 Kubernetes에서는, `sourceLabels`는 일반적으로 해당 Kuberneted 서비스의 pod selector에 사용된 라벨과 같을 것 입니다.

위의 예제는 또한 *reviews* 서비스의 "v2" 버전에만 적용되도록 수정할 수 있습니다:

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

_2. HTTP 헤더에 기반하여 규칙 선택_.
예로, 다음 규칙은 들어오는 요청 중 사용자 지정 "end-user" 헤더의 값이 문자열 "jason"일 때에만 적용됩니다:

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

만약 규칙에 한 개 이상의 헤더가 지정되어 있다면 모든 헤더가 조건을 만족해야 규칙이 적용됩니다.

_3. 요청 URI에 기반하여 규칙 선택_.
예로, 다음 규칙은 `/api/v1`으로 시작하는 URI 경로를 가진 요청에 대해서만 적용됩니다:

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

#### 다중 매치 조건

여러 매치 조건은 동시에 설정될 수 있습니다.
그런 경우에, 중첩 단계에 따라 AND나 OR이 적용됩니다.

만약 여러 조건이 한 개의 매치 절에 중첩되어 있다면 그 조건들은 AND가 적용됩니다.
예를 들어, 다음 규칙은 클라이언트 작업량이 "reviews:v2"이고 요청의 사용자 정의 헤더 "end-user" 헤더의 값이 "jason"일 때에만 적용됩니다:

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

만약 조건들이 별개의 매치 절에 나타난다면, 하나의 조건만 만족하면 됩니다(OR):

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

이 규칙은 클라이언트 작업량이 "reviews:v2"이거나 요청의 사용자 정의 헤더 "end-user" 헤더의 값이 "jason"일 경우에 적용됩니다.

#### 우선순위

주어진 목적지에 대해 여러 규칙이 있다면, `VirtualService`에 나타나는 순서대로 처리되어 가장 처음 나타나는 규칙이 가장 높은 우선 순위를 가집니다.

**우선순위가 왜 중요한가요?**
특정 서비스의 라우팅 시나리오가 순수하게 가중치에만 기반한다면 하나의 규칙으로 설정할 수 있습니다.
하지만 다른 조건(특정 유저로부터 오는 요청과 같은)이 트래픽을 라우팅 하는데 쓰여진다면, 그 라우팅을 표현하기 위해 한 개 이상의 규칙이 필요하게 됩니다.
이 때, 규칙들이 올바른 순서로 처리될 수 있도록 규칙의 우선순위를 세심하게 고려하여야 합니다.

일반화된 라우트 정의에 사용되는 패턴은 하나 또는 그 이상의 높은 우선순위를 가진 여러 조건을 포함하는 규칙을 제공하고, 마지막으로 매치 조건이 없는 한 개의 가중치 기반의 규칙을 추가하여 다른 모든 경우에 대해 처리하는 것입니다.

예로, 다음의 `VirtualService`는 종합하여 *reviews* 서비스에 대한 요청의 "Foo"라는 헤더의 값이 "bar"인 모든 요청을 "v2" 인스턴스로 보내도록 하는 규칙 2개를 지정하고 있습니다.
다른 모든 요청은 "v1"으로 보내어집니다:

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

헤더 기반의 규칙이 더 높은 우선순위를 가지고 있는 것에 주목하세요.
헤더 기반 규칙의 우선순위가 가중치 기반의 규칙의 우선순위 보다 더 낮다면, 이 규칙은 위에서 설명한 대로 동작하지 않을 것입니다.
"Foo" 헤더의 값이 알맞은 요청이 들어와도 가중치 기반의 규칙이 먼저 처리되어 모든 트래픽을 "v1"으로 보내버릴 것이기 때문입니다.
들어오는 요청에 대해 적용되는 규칙이 발견되면, 즉시 규칙이 적용되고 규칙 평가 과정은 종료됩니다.
그렇기 때문에 규칙이 한 개 이상일 때 우선순위를 잘 정하는 것이 아주 중요합니다.

### 목적지 규칙<sub>Destination rule</sub>

[`DestinationRule`](/docs/reference/config/istio.networking.v1alpha3/#DestinationRule)은 `VirtualService`의 라우팅이 이루어진 뒤에 적용될 정책을 설정합니다.
이 규칙은 서비스의 소유자가 관리하여 회로 차단기를 정의하고, 로드 밸런서 설정을 정의하고, TLS 설정을 하고 다른 설정 또한 할 수 있게 의도되어 있습니다.

`DestinationRule`은 또한 목적지 호스트의 주소 지정이 가능한 `subset`(이름 있는 버전<sub>named version</sub>)을 정의합니다.
이 서브셋은 특정 버전의 서비스에 트래픽을 보낼 때 `VirtualService`의 라우트 정의에서 사용됩니다.

다음의 `DestinationRule`은 reviews 서비스에 대한 정책과 서브셋을 설정합니다:

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

하나의 `DestinationRule` 설정에 여러 개의 정책을 (이 예제에서는 기본 정책과 v2 특화 정책) 지정할 수 있는 것을 알아두세요.

#### 회로 차단기

연결 제한과 요청 제한과 같은 조건에 기반한 간단한 회로 차단기도 설정할 수 있습니다.

예로, 다음의 `DestinationRule`은 *reviews* 서비스의 "v1" 버전 백엔드에 100개의 연결 제한을 설정합니다:

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

회로 차단기의 제어를 살펴보기 위해서는 [회로 차단 작업](/docs/tasks/traffic-management/circuit-breaking/)을 보세요.

#### 규칙 처리

라우트 규칙과 유사하게 `DestinationRule`에 정의된 정책도 특정한 *호스트*와 연관되어 있습니다.
하지만 subset 특화된 규칙이라면, 그 규칙의 활성화는 라우트 규칙의 처리 결과에 따라 달라집니다.

규칙 처리 과정의 첫 단계는 요청하는 *호스트*에 대응되는 `VirtualService`의 라우트 규칙을 평가하고, 목적지 서비스의 요청이 전달될 subset(특정된 버전을 의미함)을 결정하는 것입니다.
그 다음으로는, 선택된 subset에 적용되는 정책이 있다면, 요청에 그 정책이 적용되어야 하는 지 평가됩니다.

**참고:** 명심해야할 알고리즘의 미묘한 점은 특정한 subset에 대해 정의된 정책은 해당 subset으로 명시적으로 라우팅 되었을 때에만 적용된다는 것입니다.
예로, 다음과 같은 설정이 *reviews* 서비스에 대해 정의된 유일한 규칙이라 생각해봅시다.
이 말인 즉슨 *reviews* `VirtualService`의 정의에는 다른 라우트 규칙이 없다는 것입니다:

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

*reviews* 서비스에 대해 다른 라우트 규칙이 정의된 것이 없기 때문에, 기본 규칙인 round-robin 라우팅 동작이 적용되어 때떄로 "v1" 인스턴스를 부르거나 "v1"이 실행 중인 유일한 버전이라면 항상 "v1"을 부를 것입니다.
그렇지만 위에 정의된 연결 제한 정책은 기본 라우팅 동작이 더 낮은 수준에서 동작하고 있기 때문에 "v1" 버전에도 적용되지 않을 것입니다.
규칙 처리 엔진이 최종 목적지를 알 수 없어 요청을 subset 정책에 짝 짓지 못합니다.

위 예제는 두 가지 방법으로 고칠 수 있습니다.
트래픽 정책을 `DestinationRule`의 정의에서 한 수준 위로 올려 모든 버전에 적용되도록 할 수 있습니다:

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

또는, 더 나은 방법으로는 `VirtualService`의 정의에서 서비스에 알맞은 라우트 규칙을 정의하는 것이 있습니다.
예를 들면, "reviews:v1"에 대한 라우트 규칙을 추가하세요:

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

Istio의 기본 동작이 아무런 규칙 설정 없이 편리하게 어떤 곳에서던 오는 트래픽을 모든 버전의 목적지 서비스로 보내주지만, 버전에 대한 다른 정책이 필요해지는 순간 규칙이 필요해집니다.
그러므로, 시작부터 모든 서비스에 대해 기본 규칙을 설정하는 것이 Istio에서 모범 사례입니다.

### 서비스 항목<sub>Service entry</sub>

[`ServiceEntry`](/docs/reference/config/istio.networking.v1alpha3/#ServiceEntry)는 추가적인 항목을 Istio가 내부적으로 관리하는 서비스 레지스트리에 등록하기 위해 사용됩니다.
`ServiceEntry`는 서비스 외부에 있는 서비스에 요청을 보낼 수 있게 하기 위해서 가장 많이 사용됩니다.
예로, 다음의 `ServiceEntry`는 `*.foo.com` 도메인에서 관리되는 서비스에 대한 외부 요청을 허용하는데 사용됩니다:

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

`ServiceEntry`의 목적지는 `hosts` 필드를 통해 지정됩니다.
`hosts` 필드의 값은 절대 표기된 이름이거나 와일드카드가 포함된 이름입니다.
`hosts` 필드의 값은 하나 또는 그 이상의, 메쉬 안의 서비스가 요청을 보낼 수 있는 서비스를 나타냅니다.

`ServiceEntry`는 외부 서비스 설정에만 국한되지 않습니다.
`ServiceEntry`는 두 종류 중 하나가 될 수 있습니다: 메쉬 내부적<sub>mesh-internal</sub>, 메쉬 외부적<sub>mesh-external</sub>.
메쉬 내부적인 항목은 내부 서비스와 비슷하지며 메쉬에 서비스를 명시적으로 추가하기 위해 사용됩니다.
메쉬 내부적인 항목은 관리되지 않은 인프라스트럭쳐를 포함하여 서비스 메쉬를 확장하기 위해 서비스를 추가하는 데 사용됩니다(예로, Kubernetes 기반의 서비스 메쉬에 추가된 VM).
메쉬 외부적인 항목은 메쉬의 외부에 존재하는 서비스를 나타냅니다.
메쉬 외부적인 항목에게는 상호 TLS 인증<sub>mutual TLS authentication</sub>은 비활성화 되어 있으며 내부 서비스 요청과 다르게 정책 적용을 서버 단이 아닌 클라이언트 단에서 수행하게 됩니다.

서비스 항목은 제대로 짝지어진 `hosts`를 사용하는 서비스를 가르킨다면 virtual service와 destination rule과 함께 잘 동작합니다.
예로, 다음의 규칙은 위의 `ServiceEntry` 규칙과 함께 사용되어 `bar.foo.com`에 있는 외부 서비스에 대한 요청의 타임아웃을 10s로 설정할 수 있습니다.:

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

트래픽을 리다이렉트하고 전달하거나 재시도, 타임아웃, 결함 주입을 하는 정책과 규칙은 모두 외부 목적지에 대해서도 동작합니다.
하지만 가중치(버전 기반) 기반 라우팅은 외부 서비스에는 다중 버전이라는 개념이 없기 때문에 불가능합니다.

외부 서비스에 접근하는 것에 대해 더 알고 싶다면 [egress 작업](/docs/tasks/traffic-management/egress/)을 보세요.

### 게이트웨이<sub>Gateway</sub>

[Gateway](/docs/reference/config/istio.networking.v1alpha3/#Gateway)는 일반적으로 메쉬의 가장자리에서 동작하여 어플리케이션에 유입되는 HTTP/TCP 트래픽에 대한 로드 밸런서를 설정합니다.

Kubernetes의 Ingress와는 다르게 Istio의 `Gateway`는 L4-L6 기능만을 설정합니다(예로, 노출할 포트, TLS 설정).
사용자는 `VirtualService`을 `Gateway`에 묶음으로서 표준 Istio 규칙을 사용하여 `Gateway`에 유입되는 HTTP 요청과 TCP 트래픽을 제어할 수 있습니다.

예로, 다음의 간단한 `Gateway`는 `bookinfo.com` 호스트에 대한 외부로부터의 HTTPS 요청을 메쉬로 들여보내는 설정을 합니다:

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

대응되는 경로를 설정하기 위해서 반드시 `Gateway`에서 사용한 것과 같은 호스트로 `VirtualService`의 `gateways` 필드를 지정하여 묶어 설정해 주어야 합니다:

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

완전한 ingress gateway 예제를 보려면 [ingress 작업](/docs/tasks/traffic-management/ingress/)을 보세요.

기본적으로 ingress 트래픽을 관리하기 위해 사용되지만 `Gateway`는 완전히 내부용이나 egress proxy를 만들기 위해 사용할 수 있습니다.
장소에 관계없이 모든 게이트웨이는 같은 방법으로 설정되고 제어됩니다.
더 자세한 내용은 [게이트웨이](/docs/reference/config/istio.networking.v1alpha3/#Gateway)를 보세요.
