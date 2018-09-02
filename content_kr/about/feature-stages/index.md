---
title: Feature Status
description: List of features and their release stages.
weight: 10
aliases:
    - /docs/reference/release-roadmap.html
    - /docs/reference/feature-stages.html
    - /docs/welcome/feature-stages.html
    - /docs/home/roadmap.html
icon: /img/feature-status.svg
---

이 페이지에는 Istio의 모든 기능의 상대적인 성숙도와 지원 단계를 나열합니다. 각 단계(Alpha, Beta와 Stable)는 프로젝트 전체에 적용되는 것이 아니라 프로젝트의 기능 별로 적용된다는 것을 참고하세요. 여기에 각 라벨이 무엇을 의미하는지 대략적으로 설명합니다:

## 기능 단계의 정의

|            | Alpha      | Beta         | Stable
|-------------------|-------------------|-------------------|-------------------
|   **목적**         | 데모가 가능하고 end-to-end로 동작하지만 제약이 있음     | production에서 사용가능함, 이제 더이상 장난감이 아님         | 의존가능하며 production으로 굳어짐
|   **API**         | 하위 호환성이 보장되지 않음    | 버저닝된 API         | 의존가능하며 production에서 사용 가능함. 버저닝된 API와 하위 호환성을 위한 자동화된 버전 변환이 포함
|  **성능**         | 수치화되지 않았거나 보장되지 않음     | 수치화되지 않았거나 보장되지 않음         | 성능(지연/스케일)이 수치화되고 문서화되며 regression에 대한 보장
|   **Deprecation 정책**        | 없음     | 약함 - 3 달         | 의존 가능함,  견고함. 변경이 생기기 1 년 전에 안내를 제공함

## Istio 기능

아래는 존재하는 기능과 기능의 단계를 나열합니다. 이 정보는 매달 릴리즈 이후에 업데이트 됩니다.

### Traffic Management

| Feature           | Phase
|-------------------|-------------------
| Protocols: HTTP1.1 / HTTP2 / gRPC / TCP | Stable
| Protocols: Websockets / MongoDB  | Beta
| Traffic Control: label/content based routing, traffic shifting | Beta
| Resilience features: timeouts, retries, connection pools, outlier detection | Beta
| Gateway: 모든 프로토콜에 대한 Ingress와 Egress | Beta
| TLS termination and SNI Support in Gateways | Beta
| Envoy에서 custom filters 활성화  | Alpha

### Telemetry

| Feature           | Phase
|-------------------|-------------------
| [Prometheus Integration](/docs/tasks/telemetry/querying-metrics/) | Stable
| [Local Logging (STDIO)](/docs/examples/telemetry/) | Stable
| [Statsd Integration](/docs/reference/config/policy-and-telemetry/adapters/statsd/) | Stable
| [Client and Server Telemetry Reporting](/docs/concepts/policies-and-telemetry/) | Stable
| [Service Dashboard in Grafana](/docs/tasks/telemetry/using-istio-dashboard/) | Beta
| [Istio Component Dashboard in Grafana](/docs/tasks/telemetry/using-istio-dashboard/) | Beta
| [Stackdriver Integration](/docs/reference/config/policy-and-telemetry/adapters/stackdriver/) | Alpha
| [SolarWinds Integration](/docs/reference/config/policy-and-telemetry/adapters/solarwinds/) | Alpha
| [Service Graph](/docs/tasks/telemetry/servicegraph/) | Alpha
| [Distributed Tracing to Zipkin / Jaeger](/docs/tasks/telemetry/distributed-tracing/) | Alpha
| [Service Tracing](/docs/tasks/telemetry/distributed-tracing/) | Alpha
| [Logging with Fluentd](/docs/tasks/telemetry/fluentd/) | Alpha
| [Trace Sampling](/docs/tasks/telemetry/distributed-tracing/#trace-sampling) | Alpha

### Security and Policy Enforcement

| Feature           | Phase
|-------------------|-------------------
| [Deny Checker](/docs/reference/config/policy-and-telemetry/adapters/denier/)         | Stable
| [List Checker](/docs/reference/config/policy-and-telemetry/adapters/list/)        | Stable
| [Pluggable Key/Cert Support for Istio CA](/docs/tasks/security/plugin-ca-cert/)        | Stable
| [Service-to-service mutual TLS](/docs/concepts/security/#mutual-tls-authentication)         | Stable
| [Kubernetes: Service Credential Distribution](/docs/concepts/security/#mutual-tls-authentication)   | Stable
| [VM: Service Credential Distribution](/docs/concepts/security/#pki)         | Beta
| [Mutual TLS Migration](/docs/tasks/security/mtls-migration)    | Beta
| [Authentication policy](/docs/concepts/security/#authentication-policies)  | Alpha
| [End User (JWT) Authentication](/docs/concepts/security/#authentication)  | Alpha
| [OPA Checker](/docs/reference/config/policy-and-telemetry/adapters/opa/)    | Alpha
| [Authorization (RBAC)](/docs/concepts/security/#authorization)   | Alpha

### Core

| Feature           | Phase
|-------------------|-------------------
| [Kubernetes: Envoy Installation and Traffic Interception](/docs/setup/kubernetes/)        | Stable
| [Kubernetes: Istio Control Plane Installation](/docs/setup/kubernetes/) | Stable
| [Attribute Expression Language](/docs/reference/config/policy-and-telemetry/expression-language/)        | Stable
| [Mixer Adapter Authoring Model](/blog/2017/adapter-model/)        | Stable
| [Helm](/docs/setup/kubernetes/helm-install/) | Beta
| [Multicluster Mesh](/docs/setup/kubernetes/multicluster-install/) | Beta
| [Kubernetes: Istio Control Plane Upgrade](/docs/setup/kubernetes/) | Beta
| [Consul Integration](/docs/setup/consul/quick-start/) | Alpha
| [Cloud Foundry Integration](/docs/setup/consul/quick-start/)    | Alpha
| Basic Configuration Resource Validation | Alpha
| [Mixer Self Monitoring](/help/faq/mixer/#mixer-self-monitoring) | Alpha
| [Custom Mixer Build Model](https://github.com/istio/istio/wiki/Mixer-Compiled-In-Adapter-Dev-Guide) | Alpha
| [Out of Process Mixer Adapters (gRPC Adapters)](https://github.com/istio/istio/wiki/Mixer-Out-Of-Process-Adapter-Dev-Guide) | Alpha

> {{< idea_icon >}}
Please get in touch by joining our [community](/about/community/) if there are features you'd like to see in our future releases!
