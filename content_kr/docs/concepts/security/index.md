---
title: 보안
description: Istio의 인증과 인가 기능을 설명합니다.
weight: 30
keywords: [security,authentication,authorization,rbac,access-control]
aliases:
    - /docs/concepts/network-and-auth/auth.html
    - /docs/concepts/security/authn-policy/
    - /docs/concepts/security/mutual-tls/
    - /docs/concepts/security/rbac/
---

모노리식 어플리케이션을 원자적인 서비스로 나누는 것은 더 나은 어질리티와 확장성과 재사용성과 같은 여러 이득을 제공합니다.
하지만 마이크로서비스에는 특수한 보안 요구사항이 있습니다:

- 중간자<sub>man-in-the-middle</sub> 공격에 대한 보호를 위해 트래픽 암호화가 필요합니다.

- 유연한 서비스 접근 제어를 위해 상호 TLS와 세심한 접근 정책이 필요합니다.

- 누가 무엇을 언제 했는지 감사하기 위해 감사 도구가 필요합니다.

Istio의 보안은 이러한 모든 이슈에 대해 솔루션을 제공하고자 시도합니다.

이 페이지는 Istio의 보안 기능을 사용하여 서비스를 어디에서 실행하던 상관없이 보호하는 방법의 개요를 제공합니다.
특히 Istio의 보안은 데이터와 엔드포인트, 커뮤니케이션과 플랫폼에 대한 내부자와 외부 위협을 완화시켜줍니다.

{{< image width="80%" ratio="56.25%"
    link="./overview.svg"
    caption="Istio Security Overview"
    >}}

Istio의 보안 기능은 당신의 서비스와 데이터를 보호하기 위한 강한 신원과 강력한 정책, 투명한 TLS 암호화와 인증, 인가, 감사(AAA) 도구를 제공합니다.
Istio 보안의 목표는 다음과 같습니다:

- **기본적으로 안전함**: 어플리케이션 코드와 인프라스트럭처에 변경이 필요하지 않습니다

- **깊은 방어**: 존재하는 보안 시스템과 통합하여 여러 단계의 방어를 제공합니다

- **신뢰하지 않는<sub>Zero-trust</sub> 네트워크**: 신뢰되지 않은 네트워크에 보안 솔루션을 생성합니다

Istio 보안 기능을 배포된 서비스에서 사용하기 위해서는 [상호 TLS 이전 문서](/docs/tasks/security/mtls-migration/)를 보세요.
보다 자세한 보안 기능 사용에 대한 설명을 위해서는 [보안 작업](/docs/tasks/security/)를 보세요.

## 고수준 구조

Istio의 보안에는 여러 컴포넌트가 사용됩니다:

- **Citadel**은 키와 인증서 관리를 합니다

- **Sidecar와 주위 proxy**는 클라이언트와 서버 사이의 안전한 커뮤니케이션을 구현합니다

- **Pilot**은 [인증 정책](/docs/concepts/security/#authentication-policies)과 [안전한 이름 정보](/docs/concepts/security/#secure-naming)를 proxy에 배포합니다

- **Mixer**는 인가와 감사를 관리합니다

{{< image width="80%" ratio="56.25%"
    link="./architecture.svg"
    caption="Istio Security Architecture"
    >}}

아래의 절에서 Istio의 보안 기능을 자세히 설명합니다.

## Istio 신원<sub>identity</sub>

신원은 어떤 보안 인프라스트럭처에서도 중요한 개념입니다.
서비스 사이의 커뮤니케이션이 시작될 때, 두 서비스는 상호 인증을 위한 신원 정보와 자격 증명을 교환해야만 합니다.
클라이언트 측에서는 서버의 신원을 [안전한 이름](/docs/concepts/security/#secure-naming) 정보에 서버가 서비스의 인가된 실행자인지 확인합니다.
서버 측에서는 [인가 정책](/docs/concepts/security/#authorization-policy)에 따라 클라이언트가 접근할 수 있는 정보를 결정하고 누가 언제 무엇에 접근했는지 감사하며 클라이언트가 사용한 것에 기반하여 요금을 물리고 접근 비용을 내지 못하는 클라이언트를 거부할 수 있습니다.

Istio 신원 모델에서, Istio는 일급 서비스 신원을 사용하여 서비스의 신원을 결정합니다.
일급 서비스 신원은 아주 유연하고 세분화되어 인간 사용자나 서비스, 서비스의 그룹을 나타낼 수 있습니다.
그런 신원이 존재하지 않는 플랫폼에 대해 Istio는 서비스 이름과 같은 다른 신원을 사용해 서비스 인스턴스를 묶을 수 있습니다.

플랫폼에 따른 Istio의 서비스 신원:

- **Kubernetes**: Kubernetes service account

- **GKE/GCE**: GCP service account를 사용할 수 있습니다

- **GCP**: GCP service account

- **AWS**: AWS IAM user/role account

- **On-premise (non-Kubernetes)**: 사용자 계정, 사용자 지정 서비스 계정, 서비스 이름, istio의 서비스 계정 service account, GCP service account.
  사용자 지정 서비스 계정은 사용자의 Identity Directory가 관리하는 신원과 같은 존재하고 있던 서비스 계정을 말합니다.

### Istio 보안 vs SPIFFE

[SPIFFE](https://spiffe.io/) 표준은 자신의 신원을 설정할 수 있고 서로 다른 종류로 이루어진 환경에 걸쳐 서비스 신원을 발급할 수 있는 프레임워크에 대한 사양을 제공합니다.

Istio와 SPIFFE는 같은 신원 문서를 공유합니다: [SVID](https://github.com/spiffe/spiffe/blob/master/standards/SPIFFE-ID.md) (SPIFFE Verifiable Identity Document).
예를 들어, Kubernetes에서 X.509 인증서는 "spiffe://\<domain\>/ns/\<namespace\>/sa/\<serviceaccount\>"와 같은 형식의 URI 필드를 가지고 있습니다.
이 점은 Istio 서비스가 다른 SPIFFE를 만족하는 시스템과 연결을 맺고 허용할 수 있다는 것을 말합니다.

Istio의 보안과 SPIFFE의 구현인 [SPIRE](https://spiffe.io/spire/)는 PKI 구현 세부 사항에서 다릅니다.
Istio는 인증, 인가, 감사를 포함한 더 종합적인 보안 솔루션을 제공합니다.

## PKI

Istio의 PKI는 Istio Citadel 위에 구현되어 있고 안전하게 강력한 워크로드 신원을 모든 워크로드에 공급합니다.
Istio는 신원을 [SPIFFE](https://spiffe.io/) 형식에 맞게 전달하기 위해 X.509 인증서를 사용합니다.
PKI는 대규모 키와 인증서 회전을 자동화 합니다.

Istio는 Kubernetes pod와 on-premise 기계에서 실행되고 있는 서비스를 모두 지원합니다.
지금은 각 시나리오에 마다 다른 인증서 키 발금 메커니즘을 사용하고 있습니다.

### Kubernetes 시나리오

1. Citadel은 Kubernetes의 `apiserver`를 감시하여 존재하던 서비스 계정과 새로운 서비스 계정의 SPIFFE 인증서와 키 페어를 생성합니다.
   Citadel은 인증서와 키 페어를 [Kubernetes secrets](https://kubernetes.io/docs/concepts/configuration/secret/)로 저장합니다.

1. pod를 생성할 때, Kubernetes는 인증서와 서비스 계정에 맞는 키 페어를 [Kubernetes secret volume](https://kubernetes.io/docs/concepts/storage/volumes/#secret)를 통해 마운트 합니다.

1. Citadel은 각 인증서의 일생을 감시하고 Kubernetes secrets을 다시 씀으로서 인증서를 자동으로 회전 시킵니다.

1. Pilot은 어떤 서비스 계정이나 계정의 그룹이 특정 서비스를 실행할 수 있는지를 정의하는 [안전한 이름](/docs/concepts/security/#secure-naming) 정보를 생성합니다.
   Pilot은 그 다름 안전한 이름 정보를 sidecar Envoy로 전달합니다.

### on-premises 기계 시나리오

1. Citadel은 [Certificate Signing Requests](https://en.wikipedia.org/wiki/Certificate_signing_request) (CSRs)를 받는 gRPC 서비스를 생성합니다.

1. 노드 에이전트는 개인 키와 CSR을 생성하고 CSR을 자격 증명과 함께 서명을 위해 Citadel에 보냅니다.

1. Citadel은 CSR과 함께 전달된 자격 증명을 검증하고 CSR에 서명하여 인증서를 생성합니다.

1. 노드 에이전트는 Citadel로부터 받은 인증서와 개인 키를 Envoy에 전달합니다.

1. 위의 CSR 과정이 주기적으로 반복되어 인증서와 키를 회전합니다.

### Kubernetes의 노드 에이전트 (개발 중)

근미래에 Istio는 아래와 같이 노드 에이전트를 사용하여 인증서와 키 발급을 할 것입니다.
Kubernetes 시나리오에 대해서만 설명하므로 on-premise 기계에 대한 신원 발급 흐름은 같은 것을 유념하세요.

{{< image width="80%" ratio="56.25%"
    link="./node_agent.svg"
    caption="PKI with node agents in Kubernetes"
    >}}

흐름은 다음과 같습니다:

1. Citadel은 CSR 요청을 받는 gRPC 서비스를 생성합니다.

1. Envoy는 인증서와 키 요청을 Envoy secret discovery service (SDS) API를 통해 보냅니다.

1. SDS 요청을 받으면 노드 에이전트는 개인 키와 CSR을 생성해 Citadel에 서명을 위해 자격 증명과 함께 CSR을 보냅니다.

1. Citadel은 CSR과 자격 증명을 검증하고 CSR에 서명하여 인증서를 생성합니다.

1. 노드 에이전트는 Citadel로부터 받은 인증서와 개인 키를 Envoy로 Envoy SDS API를 통해 보냅니다.

1. 위의 CSR 과정은 주기적으로 반복되어 인증서와 키를 회전시킵니다.

## 모범 사례

이 절에서는 몇몇 배포 가이드라인을 제공하고 실제 시나리오를 하나 다룹니다.

### 배포 가이드라인

중간규모나 대규모 클러스터에 다른 서비스를 배포하는 서비스 운영자가 여러 명이라면([SRE](https://en.wikipedia.org/wiki/Site_reliability_engineering)), 각 SRE 팀이 접근 권한을 격리하기 위해 팀마다 [Kubernetes namespace](https://kubernetes.io/docs/tasks/administer-cluster/namespaces-walkthrough/)를 만드는 것을 추천합니다.
예를 들어 `team1`에 대해 `team1-ns` namespace를 만들고 `team2`를 위해  `team2-ns` namespace를 만들어 두 팀이 각자 서로의 서비스에 접근할 수 없게 할 수 이습니다.

> {{< warning_icon >}} Citadel이 위험에 노출되었다면, 클러스터 안의 Citadel이 관리하는 모든 키와 인증서는 노출되었을 수 있습니다.
 Citadel을 독립된 namespace(예를 들면, `istio-citadel-ns`)에서 실행하여 클러스터의 관리자만 접근할 수 있게 하는 것을 **강력히** 추천합니다.

### 예제

3 서비스를 가지고 있는 3단계 어플리케이션을 생각해봅시다: `photo-frontend`, `photo-backend`, `datastore`.
photo SRE팀은 `photo-frontend`와 `photo-backend` 서비스를 관리하고 datastore SRE팀은 `datastore` 서비스를 관리합니다.
`photo-frontend` 서비스는 `photo-backend`에 접근할 수 있고, `photo-backend` 서비스는 `datastore` 서비스에 접근할 수 있습니다.
아지만 `photo-frontend` 서비스는 `datastore`에 접근할 수 없습니다.

이 시나리오에서 클러스터 관리자는 3개의 namespace를 만듭니다: `istio-citadel-ns`, `photo-ns`, `datastore-ns`.
관리자는 모든 namespaces에 대한 접근 권한을 가지고 있고 각 팀은 팀의 namespace에만 접근할 수 있습니다.
photo SRE팀은 `photo-frontend`와 `photo-backend`를 실행하기 위한 2개의 서비스 계정을 `photo-ns` namespace에 만듭니다.
datastore SRE팀은 `datastore` 서비스를 실행하기 위한 서비스 계정을 `datastore-ns` namespace에 만듭니다.
거기에 더해 [Istio Mixer](/docs/concepts/policies-and-telemetry/)의 서비스 접근 제어를 사용해 `photo-frontend`가 datastore에 접근하지 못하게 합니다.

이 설정으로 Kubernetes는 서비스를 관리하는 운영자 권한을 격리할 수 있습니다.
Istio는 모든 namespace에서 인증서와 키를 관리하고 서비스에 서로 다른 접근 제어 규칙을 적용합니다.

## 인증<sub>Authentication</sub>

Istio는 두 가지의 인증을 제공합니다:

- **전송<sub>Transport</sub> 인증**, **서비스 간<sub>service-to-service</sub> 인증**:
  연결을 맺는 클라이언트를 검증합니다.
  Istio는 전송 인증 솔루션으로 [상호 TLS](https://en.wikipedia.org/wiki/Mutual_authentication)를 제공합니다.
  이 기능을 어떠한 서비스 코드 수정 없이 활성화할 수 있습니다.
  이 기능은:

    - 클러스터와 클라우드 간 상호 운용을 가능하도록 역할을 설명하는 강한 신원을 각 서비스에 제공합니다.
    - 서비스 간 커뮤니케이션과 서비스와 말단 사용자 사이의 커뮤니케이션을 보호합니다.
    - 키와 인증서 생성, 배포, 회전을 자동화 하는 키 관리 시스템을 제공합니다.

- **근원지<sub>Origin</sub> 인증**, **말단<sub>end-user</sub> 인증**:
  요청을 하는 근본적인 클라이언트를 말단 사용자나 기기로서 검증합니다.
  Istio는 요청 수준 인증을 JSON Web Token(JWT) 검증과 [Auth0](https://auth0.com/), [Firebase Auth](https://firebase.google.com/docs/auth/), [Google Auth](https://developers.google.com/identity/protocols/OpenIDConnect), 사용자 지정 인증 등으로 제공합니다.

두 경우 모두 Istio는 인증 정책을 `Istio config store`에 사용자 지정 Kubernetes API를 통해 저장합니다.
Pilot은 각 proxy에 대해 알맞은 키와 함께 인증 정책을 최신 상태로 유지합니다.
추가로, Istio는 변경할 정책의 효과를 적용하기 전에 이해할 수 있게 도움을 주는 관대한 인증 모드<sub>permissive authentication mode</sub>를 지원합니다.

### 상호 TLS 인증<sub>Mutual TLS authentication</sub>

Istio는 서비스 간 커뮤니케이션을 클라이언트 측과 서버 측의 [Envoy proxies](https://envoyproxy.github.io/envoy/)를 이용해 터널링 합니다.
클라이언트가 서버를 부르기 위한 단계는 다음과 같습니다:

1. Istio는 클라이언트의 outbound 트래픽을 클라이언트의 sidecar Envoy로 리라우팅합니다.

1. 클라이언트 측의 Envoy가 서버 측의 Envoy와 상호 TLS handshake를 시작합니다.
   handshake가 일어나는 동안, 클라이언트 측의 Envoy는 서버 인증서의 서비스 계정이 해당 서비스를 실행할 수 있게 인가되었는지 확인하지 위해 [안전한 이름](/docs/concepts/security/#secure-naming) 확인을 함께 합니다.

1. 클라이언트 측의 Envoy와 서버 측의 Envoy는 상호 TLS 연결을 수립하고 Istio가 트래픽을 클라이언트 측 Envoy에서 서버 측 Envoy로 전달합니다.

1. 인가 뒤에는 서버 측 Envoy는 서버로 들어오는 트래픽을 TCP 연결을 통해 전달합니다.

#### 안전한 이름<sub>Secure naming</sub>

안전한 이름 정보는 인증서 안에 암호화되어 저장된 서버 신원과 서비스 디스커버리 서비스나 DNS가 사용하는 서비스 이름 사이의 *N-to-N* 매핑을 담고 있습니다.
신원 `A`와 서비스 이름 `B`의 매핑은 "`A`가 허용되었고 서비스 `B`를 실행할 수 있게 인가받았다"는 의미입니다.
Pilot은 Kubernetes의 `apiserver`를 감시하여 안전한 이름 정보를 생성하고 sidecar Envoy로 안전하게 배포합니다.
아래의 예제는 왜 안전한 이름이 인증에서 중요한지를 설명합니다.

`datastore`를 실행하며 `infra-team` 신원만을 사용하는 정당한 서버를 가정합시다.
한 악의적인 사용자가 `test-team` 신원에 대한 키와 인증서를 가지고 있습니다.
악의적인 사용자는 서비스로 가장하여 클라이언트가 보낸 데이터를 사찰하고자 합니다.
악의적인 사용자는 `test-team` 신원에 대한 키와 인증서를 가진 위조 서버를 배포합니다.
악의적인 사용자가 성공적으로 서비스 디스커버리 서비스나 DNS를 해킹하여 서비스 이름 `datastore`를 위조 서버로 연결했다고 가정합시다.

클라이언트가 `datastore`에 요청을 하면, `test-team` 신원을 서버의 인증서로부터 추출하고 `test-team`이 `datastore` 서비스를 실행할 수 있는 인가가 있는지를 안전한 이름 정보를 가지고 확인합니다.
클라이언트는 `test-team`이 `datastore` 서비스를 실행하도록 인가되지  **않**았다는 것을 발견하고 인증이 실패합니다.

### 인증 구조

Istio의 메쉬 안의 요청을 받는 서비스의 인증 요구 사항을 인증 정책을 사용하여 지정할 수 있습니다.
메쉬 운영자는 `.yaml` 파일을 사용하여 정책을 지정합니다.
정책은 배포되고 나면 Istio의 설정 저장소에 저장됩니다.
Istio의 컨트롤러인 Pilot는 설정 저장소를 감시합니다.
어떤 정책이 변화하면 Pilot이 새 정책을 Envoy sidecar proxy에게 어떻게 요구된 인증 메커니즘을 수행해야 하는지 알려주는 알맞은 설정으로 변환합니다.
Pilot은 JWT 검증을 위해 공개 키를 가져와서 설정에 첨부할 수 있습니다.
또는, Pilot은 상호 TLS를 위해 Istio 시스템이 관리하는 키와 인증서의 경로를 제공하고 어플리케이션 pod에 설치할 수 있습니다.
더 많은 정보를 [PKI 절](/docs/concepts/security/#pki)에서 확인할 수 있습니다.
Istio는 비동기적으로 목표 엔드포인트에 설정을 전달합니다.
proxy가 설정을 받으면 그 pod에 새 인증 요구 사항이 즉시 적용됩니다.

일반적으로는 요청을 보내는 클라이언트 서비스가 필요한 인증 메커니즘을 따를 책임이 있습니다.
근원지 인증(JWT)의 경우에 어플리케이션이 JWT 자격 증명을 얻고 요청에 첨부할 책임을 가지고 있습니다.
상호 TLS의 경우에 Istio는 [목적지 규칙](/docs/concepts/traffic-management/#destination-rules)을 제공합니다.
운영자는 목적지 규칙을 사용하여 클라이언트 proxy에게 서버 측에서 기대하는 인증서를 사용해 TLS를 사용한 초기 연결을 만들도록 할 수 있습니다.
[PKI와 신원 절](/docs/concepts/security/mutual-tls/)에서 Istio의 상호 TLS가 어떻게 동작하는지 확인할 수 있습니다.

{{< image width="60%" ratio="67.12%"
    link="./authn.svg"
    caption="Authentication Architecture"
    >}}

Istio는 신원과 두 가지 종류의 인증과 자격 증명에 들어 있는 다른 요구를 다음 계층으로 전달합니다: [인가](/docs/concepts/security/#authorization).
추가로 운영자는 Istio가 전송이나 근원지 인증 중 어떤 신원을 주로 사용할 지 정할 수 있습니다.

### 인증 정책

이 절은 어떻게 Istio의 인증 정책이 어떻게 동작하는지에 대한 세부 내용을 제공합니다.
[구조 절](/docs/concepts/security/#authentication-architecture)의 내용으로부터 기억하겠지만, 인증 정책은 서비스가 **받는** 요청에 대해 적용됩니다
상호 TLS에서 클라이언트 측의 인증 규칙을 지정하기 위해서는 `DestinationRule`에서 `TLSSettings`를 지정해 주어야 합니다.
더 많은 정보는 [TLS 설정 레퍼런스 문서](/docs/reference/config/istio.networking.v1alpha3/#TLSSettings)에서 찾을 수 있습니다.
다른 Istio 설정과 마찬가지로, 인증 정책은 `.yaml` 파일에 지정합니다.
`kubectl`을 사용해서 정책을 배포합니다.

아래의 예제 인증 정책은 `reviews` 서비스에 대한 전송 인증은 상호 TLS를 사용해야만 한다고 지정하고 있습니다:

{{< text yaml >}}
apiVersion: "authentication.istio.io/v1alpha1"
kind: "Policy"
metadata:
  name: "reviews"
spec:
  targets:
  - name: reviews
  peers:
  - mtls: {}
{{< /text >}}

#### 정책 저장 범위

Istio는 인증 정책을 namespace 범위나 메쉬 범위로 저장할 수 있습니다:

- `"default"`라는 이름을 가진 메쉬 범위 정책은 `kind` 필드에 `"MeshPolicy"` 값을 설정해 지정됩니다:

    {{< text yaml >}}
    apiVersion: "authentication.istio.io/v1alpha1"
    kind: "MeshPolicy"
    metadata:
      name: "default"
    spec:
      peers:
      - mtls: {}
    {{< /text >}}

- Namespace 범위 정책은 `kind` 필드에 `"Policy"` 값을 설정하고 특정할 namespace를 설정해 지정됩니다.
  만약 namespace가 지정되어 있지 않으면 기본 값이 사용됩니다.
  예를 들어 namespace `ns1`을 위한 정책은 아래와 같습니다:

    {{< text yaml >}}
    apiVersion: "authentication.istio.io/v1alpha1"
    kind: "Policy"
    metadata:
      name: "default"
      namespace: "ns1"
    spec:
      peers:
      - mtls: {}
    {{< /text >}}

namespace 범위의 저장소에 있는 정책은 같은 namespace안에 있는 서비스에만 적용됩니다.
메쉬 범위의 정책은 메쉬 안의 모든 서비스에 적용됩니다.
충돌과 오용을 막기 위해서 메쉬 범위 저장소에는 한 개의 정책만 정의할 수 있습니다.
그 정책은 `default`로 이름지어져야 하며 `targets:` 항목이 비어있어야만 합니다.
[목표 선택자 절](/docs/concepts/security/#target-selectors)에서 더 많은 정보를 찾을 수 있습니다.

Kubernetes에서는 Istio 설정을 Custom Resource Definitions(CRDs)으로 구현하였습니다.
이러한 CRD는 namespace 범위와 클러스터 범위의 `CRD`에 대응되며 Kubernetes RBAC로부터 자동으로 접근 보호를 받게 됩니다.
[Kubernetes CRD 문서](https://kubernetes.io/docs/concepts/extend-kubernetes/api-extension/custom-resources/#customresourcedefinitions)에서 더 읽어볼 수 있습니다.

#### 목표 선택자<sub>Target selector</sub>

인증 정책의 목표는 정책이 적용되는 서비스나 서비스 그룹을 지정합니다.
아래의 예제는 `targets:` 항목이 해당 정책이 아래의 서비스에 적용되도록 지정한 것을 보여줍니다:

- 모든 포트를 사용하고 있는 `product-page` 서비스.
- `9000`를 사용하는 reviews 서비스 .

{{< text yaml >}}
targets:
 - name: product-page
 - name: reviews
   ports:
   - number: 9000
{{< /text >}}

`targets:` 항목을 제공하지 않으면 Istio는 정책을 정책의 저장 범위 안에 있는 모든 서비스에 적용시킵니다
따라서 `targets:` 항목은 정책의 점위를 지정할 수 있게 해줍니다:

- 메쉬 정책: 목표 선택자 섹션이 비어있는 채로 정의된 메쉬 범위의 정책.
  **메쉬 안**에 최대 **한** 개의 메쉬 정책만이 있을 수 있습니다.

- Namespace 정책: `default`를 이름으로 하고 목표 선택자가 없이 namespace 범위에 저장된 정책.
  **namespace 당** 최대 **한** 개의 namespace 정책만이 있을 수 있습니다.

- 서비스 특화 정책: 목표 선택자가 비어있지 않고 namespace 범위에 저장된 정책.
  namespace는 **0개, 1개, 또는 많은** 서비스 특화 정책을 가질 수 있습니다.

각 서비스마다 Istio는 가장 좁게 매칭된 정책을 적용합니다.
그 순서는: **서비스 특화 > namespace > mesh**입니다.
만약 하나 이상의 서비스 특화 정책이 매치된다면, Istio는 그 중 하나를 임의로 선택합니다.
운영자는 정책 설정을 할 때 이러한 충돌을 피해야만 합니다.

메쉬 정책과 namespace 정책에 유일성을 적용하기 위해 Istio는 메쉬 당 단 하나의 인증 정책과 namespace 당 하나의 인증 정책만을 받아들입니다. Istio는 또 메쉬 정책과 namespace 정책이 `default`라는 특수한 이름을 가지도록 강제합니다.

#### Transport 인증

`peers:` 항목은 인증 방법과 정책의 전송 인증을 위한 인자를 정의합니다.
이 항목은 여러 인증 방법을 전달할 수 있고 그 중 단 하나만 만족해도 인증을 통과할 수 있습니다.
하지만 Istio 0.7 버전에서, 지원되는 유일한 전송 인증 방법은 상호 TLS입니다.
전송 인증이 필요하지 않다면, 이 절을 건너뛰세요.

다음의 예제는 `peers:` 항목으로 상호 TLS를 사용한 전송 인증을 활성화하는 것을 보여줍니다.

{{< text yaml >}}
peers:
  - mtls: {}
{{< /text >}}

현재는 상호 TLS 설정은 다른 인자를 필요로 하지 않습니다.
그러므로 `-mtls: {}`, `- mtls:` 또는 `- mtls: null` 선언은 모두 같은 것으로 취급됩니다.
미래에는 상호 TLS 설정이 서로 다른 상호 TLS 구현에 전달할 인자를 가질 수도 있습니다.

#### 근원지 인증

`origins:` 항목은 근원지 인증을 위한 인자와 인증 방법을 정의합니다.
Istio는 JWT 근원지 인증만을 지원합니다.
하지만 정책은 다른 발행인의 여러 JWT를 나열할 수 있습니다.
peer 인증과 비슷하게 나열된 방법 중 하나 만 만족되면 인증을 통과합니다.

다음의 예제는 Google이 발급한 JWT를 받는 근원지 인증을 위해 `origins:` 항목을 지정하는 것을 보여줍니다:

{{< text yaml >}}
origins:
- jwt:
    issuer: "https://accounts.google.com"
    jwksUri: "https://www.googleapis.com/oauth2/v3/certs"
{{< /text >}}

#### Principal binding

principal binding 키-값 쌍은 정책의 인증 원칙을 지정합니다.
기본적으로 Istio는 `peers:` 항목에 설정된 인증을 사용합니다.
`peers:` 항목에 인증이 설정되어 있지 않다면 Istio는 인증이 설정되지 않은 채로 둡니다.
정책 설정을 하는 사람은 이 기본 동작은 `USE_ORIGIN` 값으로 덮어쓸 수 있습니다.
이 값은 Istio가 근원지 인증을 인증 원칙으로 사용하도록 합니다.
미래에는 peer가 X일 때 `USE_PEER`를 사용하고 기본으로 `USE_ORIGIN`을 사용 하는 것과 같은 조건부 인증을 지원할 예정입니다.

아래의 예제는 `principalBinding` 키의 값이 `USE_ORIGIN`인 경우를 보여줍니다:

{{< text yaml >}}
principalBinding: USE_ORIGIN
{{< /text >}}

### 인증 정책 갱신

언제나 인증 정책을 변경할 수 있고 Istio가 그러한 변경점을 실시간에 가깝게 엔드포인트로 전파하여 줍니다.
하지만 Istio는 모든 엔드포인트가 새 정책을 동시에 받는 것을 보장할 수는 없습니다.
아래는 인증 정책을 갱신할 때 혼선을 피하기위해 추천하는 방법입니다:

- 상호 TLS를 활성화 또는 비활성화 하기 위해: `mode:` 키와 `PERMISSIVE` 값을 사용한 임시 정책을 사용하세요.
  이 설정은 요청을 받는 서비스가 일반 텍스트와 TLS 두 종류의 트래픽을 모두 허용하게 해줍니다.
  따라서 어떤 요청도 제거되지 않습니다. 모든 클라이언트가 새로운 프로토콜을 사용하게 되었을 때, `PERMISSIVE` 정책을 목표하는 정책으로 변경하면 됩니다. 더 많은 정보를 보려면 [상호 TLS 이전 튜토리얼](/docs/tasks/security/mtls-migration)를 보세요.

{{< text yaml >}}
peers:
- mTLS:
    mode: PERMISSIVE
{{< /text >}}

- JWT 인증의 이전: 정책을 변경하기 전에 요청은 새로운 JWT를 포함해야 합니다.
  이전에 사용하던 JWT가 있다면, 서버 측에서 새 정책으로 완전히 전환된 이후에 제거되어야 합니다.
  클라이언트 어플리케이션은 이러한 변경에 대비해 수정할 필요가 없습니다.

## 인가<sub>Authorization</sub>

Istio의 Role-based Access Control(RBAC)로도 알려진 인가 기능은
- Istio 메쉬에 있는 서비스에 대해 namespace 수준, 서비스 수준, 메소드 수준의 접근 제어를 제공합니다.
  다음과 같은 기능을 제공합니다:

- 간단하고 사용하기 쉬운 **Role 기반 의미 체계**.
- **서비스 간과 말단 사용자와 서비스 간 인가**.
- **사용자 지정 속성을 지원해 유연함**. 예를 들면, role과 role-binding에서의 조건문.
- Istio 인가는 Envoy에서 기본적으로 적용되기 떄문에 **고성능**을 보임.

### 인가 구조

{{< image width="90%" ratio="56.25%"
    link="./authz.svg"
    alt="Istio Authorization"
    caption="Istio Authorization Architecture"
    >}}

위 그림은 기본적인 Istio 인가 구조를 보여줍니다.
운영자는 `.yaml` 파일을 사용하여 Istio의 인가 정책을 지정합니다.
배포된 뒤에는 Istio가 정책을 `Istio Config Store`에 저장합니다.

Pilot은 Istio의 인가 정책의 변화를 감시합니다.
Pilot은 갱신된 인가 정책을 가져와 서비스 인스턴스와 함꼐 있는 Envoy proxy에 배포합니다.

각 Envoy proxy는 실행 중에 요청을 인가하는 인가 엔진을 실행합니다
요청이 proxy에 도착하면 인가 엔진이 현재 인가 정책으로 요청의 컨텍스트를 평가하고 `ALLOW`나 `DENY`의 인가 결과를 반환합니다.

### 인가 활성화

Istio의 인가를 `RbacConfig` 오브젝트를 통해 활성화 할 수 있습니다.
`RbacConfig` 오브젝트는 `default`를 고정된 이름으로 가지고 있는 메쉬 싱글톤입니다.
메쉬 안에서는 하나의 `RbacConfig` 인스턴스만 사용할 수 있습니다.
다른 Istio 설정 오브젝트와 마찬가지로 `RbacConfig`는 Kubernetes `CustomResourceDefinition`[(CRD)](https://kubernetes.io/docs/concepts/extend-kubernetes/api-extension/custom-resources/) 오브젝트로 정의됩니다.

운영자는 `RbacConfig` 오브젝트 안에 다음과 같은 값을 가질 수 있는 `mode` 값을 설정할 수 있습니다:

- **`OFF`**: Istio의 인가는 비활성화 됩니다.
- **`ON`**: Istio의 인가는 메쉬 안의 모든 서비스에 대해 활성화 됩니다.
- **`ON_WITH_INCLUSION`**: Istio의 인가는 `inclusion` 항목에 지정된 namespace와 서비스에 대해서만 활성화 됩니다.
- **`ON_WITH_EXCLUSION`**: Istio의 인가는 `exclusion` 항목에 지정된 namespace와 서비스를 제외하고 메쉬 내의 모든 서비스에 대해 활성화됩니다.

다음 예제에서 Istio의 인가는 `default` namespace에 대해 활성화 됩니다.

{{< text yaml >}}
apiVersion: "rbac.istio.io/v1alpha1"
kind: RbacConfig
metadata:
  name: default
spec:
  mode: 'ON_WITH_INCLUSION'
  inclusion:
    namespaces: ["default"]
{{< /text >}}

### 인가 정책

Istio의 인가 정책을 설정하기 위해서는 `ServiceRole`과 `ServiceRoleBinding`을 지정해야 합니다.
다른 Istio 설정 오브젝트와 마찬가지로 `ServiceRole`과 `ServiceRoleBinding`은 Kubernetes `CustomResourceDefinition`[(CRD)](https://kubernetes.io/docs/concepts/extend-kubernetes/api-extension/custom-resources/) 오브젝트로 정의됩니다.

- **`ServiceRole`**은 서비스에 접근하기 위한 허가 그룹을 정의합니다.
- **`ServiceRoleBinding`**는 `ServiceRole`를 사용자나 그룹, 서비스와 같은 특정 대상에게 인가합니다.

`ServiceRole`과 `ServiceRoleBinding`의 조합이 **누가** **어떤 조건에서** **무엇을** 할 수 있는지를 허가합니다.
자세하게는:

- **누구** `ServiceRoleBinding`의 `subjects` 항목을 가리킵니다.
- **무엇** `ServiceRole`의 `permissions` 항목을 가리킵니다.
- **어떤 조건에서**  `ServiceRole`나 `ServiceRoleBinding`에서 [Istio 속성](/docs/reference/config/policy-and-telemetry/attribute-vocabulary/)을 사용해 지정할 수 있는 `conditions` 항목을 가리킵니다.

#### `ServiceRole`

`ServiceRole` 사양은 허가로도 불리는 `rules`의 목록을 포함합니다.
각 rule은 다음과 같은 표준 항목을 가지고 있습니다:

- **`services`**: 서비스 이름의 목록. 값을 `*`으로 지정해 지정된 namespace 안의 모든 서비스를 포함할 수 있습니다.

- **`methods`**: HTTP 메소드 이름의 목록.
  gRPC 요청의 허가를 위한 HTTP 동사는 항상 `POST`입니다.
  값을 `*`로 지정해 모든 HTTP 메소드를 포함할 수 있습니다.

- **`paths`**: HTTP 경로나 gRPC 메소드.
  gRPC 메소드는 `/packageName.serviceName/methodName`과 같은 형식이어야만 하며 대소문자를 구분합니다.

`ServiceRole` 사양은 `metadata` 항목에 지정된 namespace에만 적용됩니다.
`services`와 `methods` 필드는 필수 필드입니다. `paths`는 선택 필드입니다. rule이 지정되지 않았거나 `*`로 지정되어 있다면 모든 인스턴스에 적용됩니다.

아래의 예제는 간단한 `default` namespace에 있는 서비스에 대한 모든 접근 권한을 가지는 role인 `service-admin`을 보여줍니다.

{{< text yaml >}}
apiVersion: "rbac.istio.io/v1alpha1"
kind: ServiceRole
metadata:
  name: service-admin
  namespace: default
spec:
  rules:
  - services: ["*"]
    methods: ["*"]
{{< /text >}}

또 다른 `default` namespace에 있는 `products.default.svc.cluster.local` 서비스에 대한 `"GET"`과 `"HEAD"` 읽기 권한을 가지고 있는 role인 `products-viewer`이 있습니다.

{{< text yaml >}}
apiVersion: "rbac.istio.io/v1alpha1"
kind: ServiceRole
metadata:
  name: products-viewer
  namespace: default
spec:
  rules:
  - services: ["products.default.svc.cluster.local"]
    methods: ["GET", "HEAD"]
{{< /text >}}

추가로, Istio는 rule의 모든 필드에 대해 prefix matching과 suffix matching을 지원합니다.
예를 들면 `tester` role을 `default` namespace에 대해 다음과 같은 허가를 주도록 정의할 수 있습니다:

- 다음과 같은 접두사 `"test-*"`를 가지고 있는 모든 서비스에 대한 완전한 접근 권한:
   `test-bookstore`, `test-performance`, `test-api.default.svc.cluster.local`.
- 다음과 같은 접미사  `"*/reviews"`를 가지고 있는 모든 경로에 대한 읽기(`"GET"`)권한:
   `bookstore.default.svc.cluster.local` 서비스의 `/books/reviews`, `/events/booksale/reviews`, `/reviews`.

{{< text yaml >}}
apiVersion: "rbac.istio.io/v1alpha1"
kind: ServiceRole
metadata:
  name: tester
  namespace: default
spec:
  rules:
  - services: ["test-*"]
    methods: ["*"]
  - services: ["bookstore.default.svc.cluster.local"]
    paths: ["*/reviews"]
    methods: ["GET"]
{{< /text >}}

`ServiceRole`에서 `namespace` + `services` + `paths` + `methods`의 조합이 **서비스나 서비스 그룹이 어떻게 접근되는 지**를 정의합니다.
몇몇 상황에서 rule을 위해 추가적인 조건을 지정해야 할 필요가 있을 수 있습니다.
예를 들면 rule은 서비스의 특정 **버전**에만 적용될 수 있거나특정 `"foo"`와 같은 **라벨**을 가지고 있는 서비스에만 적용될 수 있습니다.
`constraints`를 사용해서 이런 조건을 쉽게 지정할 수 있습니다.

예를 들면 아래의 `ServiceRole` 정의는 기존의 `products-viewer` role에 `request.headers[version]`이 `"v1"`나 `"v2"`어야 하는 제약을 추가합니다.
제약이 지원하는 `key` 값은 [제약과 속성 페이지](/docs/reference/config/authorization/constraints-and-properties/)에 정리되어 있습니다.
이 경우에 속성은 `request.headers`과 같은 `map`이고 `key`는 `request.headers[version]`과 같이 map 안에 있는 항목입니다.

{{< text yaml >}}
apiVersion: "rbac.istio.io/v1alpha1"
kind: ServiceRole
metadata:
  name: products-viewer-version
  namespace: default
spec:
  rules:
  - services: ["products.default.svc.cluster.local"]
    methods: ["GET", "HEAD"]
    constraints:
    - key: request.headers[version]
      values: ["v1", "v2"]
{{< /text >}}

#### `ServiceRoleBinding`

`ServiceRoleBinding` 사양은 두 부분을 포함합니다:

-  **`roleRef`**는 같은 namespace에 있는 `ServiceRole` 자원을 가리킵니다.
-  role에 배정된 **`대상`<sub>subjects</sub>**의 목록.

*subject*를 `user`나 `properties`의 집합으로 명시적으로 지정할 수 있습니다.
`ServiceRoleBinding` *subject* 안에 있는 *property*는 `ServiceRole` 사양의 *constraint*와 비슷합니다.
*property*는 이 role에 배정된 계정을 지정하기 위해 조건을 사용할 수 있게 해줍니다.
*property*는 `key`와 허용된 *values*을 가집니다. 지원되는 제약의 `key`의 값은 [제약과 속성 페이지](/docs/reference/config/authorization/constraints-and-properties/)에 나열되어 있습니다.

아래의 예제는 `test-binding-products`라는, `"product-viewer"`이라는 `ServiceRole`에 두 개의 대상을 묶고 다음과 같은 `subjects`를 가지고 있는 `ServiceRoleBinding`을 보여줍니다.

- 서비스 **a** `"service-account-a"`를 나타내는 서비스 계정.
- Ingress 서비스 `"istio-ingress-service-account"`**이고** JWT `email` 요청이 `"a@foo.com"`인 서비스를 나타내는 서비스 계정.

{{< text yaml >}}
apiVersion: "rbac.istio.io/v1alpha1"
kind: ServiceRoleBinding
metadata:
  name: test-binding-products
  namespace: default
spec:
  subjects:
  - user: "service-account-a"
  - user: "istio-ingress-service-account"
    properties:
      request.auth.claims[email]: "a@foo.com"
  roleRef:
    kind: ServiceRole
    name: "products-viewer"
{{< /text >}}

서비스를 외부에 공개하고 싶다면 `subject`를 `user: "*"`로 지정하면 됩니다.
이 값은 `ServiceRole`을 **모든 (인증 받지 않고 인가 받지 않은)** 사용자와 서비스에게 배정합니다.
예들 들면:

{{< text yaml >}}
apiVersion: "rbac.istio.io/v1alpha1"
kind: ServiceRoleBinding
metadata:
  name: binding-products-allusers
  namespace: default
spec:
  subjects:
  - user: "*"
  roleRef:
    kind: ServiceRole
    name: "products-viewer"
{{< /text >}}

**인증된** 사용자와 서비스에게만 `ServiceRole`을 배정하고 싶다면 `source.principal: "*"`을 사용하세요.
예를 들면:

{{< text yaml >}}
apiVersion: "rbac.istio.io/v1alpha1"
kind: ServiceRoleBinding
metadata:
  name: binding-products-all-authenticated-users
  namespace: default
spec:
  subjects:
  - properties:
      source.principal: "*"
  roleRef:
    kind: ServiceRole
    name: "products-viewer"
{{< /text >}}

### 다른 인가 메커니즘 사용하기

Istio의 인가 메커니즘을 사용하길 강하게 추천하지만, Istio는 Mixer 컴포넌트를 통해 당신의 인증과 인가 메커니즘을 사용할 수 있을 정도로 유연합니다.
Mixer를 설정하고 사용하고 싶다면 [정책과 원격 측정 어댑터 문서](/docs/concepts/policies-and-telemetry/#adapters)을 보세요.
