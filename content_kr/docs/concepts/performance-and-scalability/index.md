---
title: 성능과 확장성
description: Istio 컴포넌트의 성능과 확장성에 대한 방법론과 결과, 그리고 모범 사례를 소개합니다.
weight: 50
aliases:
    - /docs/performance-and-scalability/overview
    - /docs/performance-and-scalability/microbenchmarks
    - /docs/performance-and-scalability/performance-testing-automation
    - /docs/performance-and-scalability/realistic-app-benchmark
    - /docs/performance-and-scalability/scalability
    - /docs/performance-and-scalability/scenarios
    - /docs/performance-and-scalability/synthetic-benchmarks
keywords: [performance,scalability,scale,benchmarks]
---

우리는 Istio의 성능 평가, 성능 추적, 그리고 성능 향상에 대해 4갈래의 접근법을 따르고 있습니다:

* 코드 레벨의 마이크로 벤치마크 <sub>micro-benchmarks</sub>

* 여러 시나리오에 대한 합성 end-to-end 벤치마크

* 여러 설정에 걸쳐 앱에 대한 복잡하고 현실적인 end-to-end 벤치마크

* 성능이 다시 떨어지지 않도록 하기 위한 자동화

## 마이크로 벤치마크 <sub>Micro benchmarks</sub>

우리는 성능이 중요한 영역에 대해 Go의 도구를 사용해 마이크로 벤치마크를 작성합니다.
이러한 접근 방법을 통안 주요 목표는 개발자가 사용해 자신의 변경점에 대해 빠르게 전후 성능 비교를 할 수 있는 사용하기 쉬운 마이크로 벤치마크를 제공하기 위함입니다.

속성을 처리하는 코드의 성능을 측정하는 Mixer의 [샘플 마이크로 벤치마크]({{< github_file >}}/mixer/test/perf/singlecheck_test.go)를 보세요.

개발자는 성능 관리와 참고 목적으로 자신의 벤치마크 결과를 소스 트리에 저장하는 방법도 사용할 수 있습니다.
GitHub는 [baseline 파일]({{< github_file >}}/mixer/test/perf/bench.baseline)을 가지고 있습니다.

이러한 테스팅 종류의 성격 때문에 기계에 따라 높은 지연 시간의 편차를 보입니다.
그러므로 마이크로 벤치마크의 수치는 같은 기계에서 해당 벤치마크를 수행했을 때 얻었던 수치와만 비교하는 것을 추천합니다.

[`perfcheck.sh` 스크립트]({{< github_file >}}/bin/perfcheck.sh)는 빠르게 하위 폴더의 벤치마크를 수행하고 같은 위치에 있는 baseline 파일의 결과와 비교하는 데 사용할 수 있습니다.

## 테스트 시나리오

{{< image width="80%" ratio="75%"
    link="https://raw.githubusercontent.com/istio/istio/master/tools/perf_setup.svg?sanitize=true"
    alt="Performance scenarios diagram"
    caption="Performance scenarios diagram"
    >}}

합성된 벤치마크 시나리오와 수행된 테스트의 소스 코드는 [GitHub]({{< github_blob >}}/tools#istio-load-testing-user-guide)에 있습니다.

<!-- add blueperf and more details -->

## 합성된 말단간 벤치마크

Fortio를 Istio의 합성된 말단간 로드 테스팅 도구로 사용합니다.
Fortio는 지정된 query per second(qps)로 수행하며 수행 시간을 히스토그램으로 기록하며 백분위를 계산합니다(예로 p99는 99%의 요청이 그 시간보다 적게 걸리는 숫자(SI 계의 초 단위)).
Fortio는 고정된 QPS나 연결이나 쓰레드 별 최고 속도 또는 최고 부하를 지정할 수 있으며, 특정 기간동안 수행되거나 특정 횟수 만큼 수행되거나 종료될 때 까지 수행될 수 있습니다.

Fortio는 빠르고 작고 재사용 가능하고 내장 가능한 Go 라이브러리이며 명령줄 도구이며 간단한 웹 UI와 결과의 지연 시간 그래프와 여러 비교 가능한 최소, 최대, 평균, 백분위 그래프를 포함한 서버 프로세스 입니다.

Fortio는 100% 오픈 소스이며 Go와 gRPC를 제외한 다른 의존성이 없어 쉽게 결과물을 재현할 수 있고 탐구하고 싶은 시나리오나 변형을 쉽게 추가할 수 있습니다.

여기에 (우리가 매 빌드마다 수행하는 8가지 중 1개인) istio-0.7.1 버전에서 메쉬 안의 2 서비스 사이에 400 Query-Per-Second(qps)로 요청을 보내는 (상호 TLS와 Mixer 정책 검사, 원격 측정 값 수집이 포함된) 시나리오의 지연 시간 분포 예제가 있습니다:

<iframe src="https://fortio.istio.io/browse?url=qps_400-s1_to_s2-0.7.1-2018-04-05-22-06.json&xMax=105&yLog=true" width="100%" height="1024" scrolling="no" frameborder="0"></iframe>

같은 시나리오에 대한 0.6.0과 0.7.1의 히스토그램/응답 시간 분포의 비교가 0.7의 성능 향상을 보여줍니다:

<iframe src="https://fortio.istio.io/?xMin=2&xMax=110&xLog=true&sel=qps_400-s1_to_s2-0.7.1-2018-04-05-22-06&sel=qps_400-s1_to_s2-0.6.0-2018-04-05-22-33" width="100%" height="1024" scrolling="no" frameborder="0"></iframe>

그리고 해당 시나리오에 대한 모든 버전의 테스트 결과를 추적하면:

<iframe src="https://fortio.istio.io/?s=qps_400-s1_to_s2" width="100%" height="1024" scrolling="no" frameborder="0"></iframe>

[Fortio](https://github.com/istio/fortio/blob/master/README.md#fortio)에 대해 GitHub에서 더 알아볼 수 있고 결과는 [https://fortio.istio.io](https://fortio.istio.io)에서 더 볼 수 있습니다.

## 현실적인 어플리케이션 벤치마크

BluePerf로도 알려진 Acmeair는 Java로 구현된 Istio의 사용자 같은 마이크로서비스 어플리케이션입니다.
이 어플리케이션은 WebSphere Liberty에서 실행되며 가상의 항공사를 시뮬레이션합니다.

Acmeair는 다음과 같은 마이크로서비스로 구성되어 있습니다:

* **Flight 서비스**는 항공기 경로 데이터를 검색합니다.
이 서비스는 Booking 서비스가 보상 프로그램(Acmeair 사용자 충실 프로그램)을 위해 마일을 조회할 때 사용합니다.

* **Customer 서비스**는 고객 정보를 저장하고 갱신하고 검색합니다.
이 서비스는 Auth 서비스가 로그인을 위해 사용하고 Booking 서비스가 보상 프로그램을 위해 사용합니다.

* **Booking 서비스**는 예약 데이터를 저장하고 갱신하고 검색합니다.

* **Auth 서비스**는 사용자 정보와 비밀번호가 맞다면 JWT를 생성합니다.

* **Main 서비스**는 다른 서비스와 상호 작용을 하는 외형 계층(웹페이지)으로 이루어져 있습니다.
이 서비스는 사용자가 브라우저를 통해 어플리케이션과 상호 작용을 할 수 있게 해 줍니다.
하지만 이 서비스는 로드 테스팅 동안 사용되지 않습니다.

아래의 그림은 Kubernetes/Istio 환경에서 어플리케이션의 pod/container를 나타냅니다:

{{< image width="100%" ratio="80%"
    link="https://ibmcloud-perf.istio.io/regpatrol/istio_regpatrol_readme_files/image004.png" alt="Acmeair microservices overview"
    >}}

아래의 표는 리그레션 테스트에서 스크립트가 사용하는 트랜잭션과 요청의 분포를 나타냅니다:

{{< image width="100%" ratio="20%"
    link="https://ibmcloud-perf.istio.io/regpatrol/istio_regpatrol_readme_files/image006.png" alt="Acmeair request types and distribution"
    >}}

Acmeair 벤치마크 어플리케이션은 이 곳에서 찾을 수 있습니다: [IBM's BluePerf](https://github.com/blueperf).

## 자동화

Fortio 기반의 합성적인 벤치마크와 현실적인 어플리케이션(BluePerf) 모두 밤마다 수행되는 배포 파이프라인의 일부이며 그 결과를 여기에서 볼 수 있습니다:

* [https://fortio-daily.istio.io/](https://fortio-daily.istio.io/)
* [https://ibmcloud-perf.istio.io/regpatrol/](https://ibmcloud-perf.istio.io/regpatrol/)

이런 자동화가 리그레션을 조기에 잡을 수 있고 시간이 지나가며 생기는 개선을 추적할 수 있게 해 줍니다.

## 확장성과 크기에 따른 가이드

* 제어 평면 컴포넌트의 여러 복제를 설정하세요.

* [Horizontal Pod Autoscaling](https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale/)을 설정하세요

* Mixer 검사와 보고 pod를 분리하세요.

* 고 가용성(HA).

* [Istio의 성능 관련 FAQ](https://github.com/istio/istio/wiki/Istio-Performance-oriented-setup-FAQ)을 보세요.

* [성능과 확장성 Working Group](https://github.com/istio/community/blob/master/WORKING-GROUPS.md#performance-and-scalability)에서 하는 일을 보세요.

현재 Istio의 모든 기능을 사용할 때의 추천 사항:

* 접근 로깅이 활성화된 sidecar의 초당 1000 요청 마다 1 vCPU를 필요로 합니다.
로깅이 비활성화 되면 0.5 vCPU를 필요로 합니다.
sidecar의 `fluentd`가 로그를 수집하고 전송하며 해당 비용에 가장 큰 지분을 가지고 있습니다.

* Mixer 검사의 일반적인 캐시 적중 비율(>80%)을 가정하면: Mixer pod의 초당 1000 요청 마다 0.5 vCPU를 필요로 합니다.

* 0.7.1 버전 기준으로 서비스 간 요청의 지연 비용/오버헤드는 대략 [10 밀리초](https://fortio.istio.io/browse?url=qps_400-s1_to_s2-0.7.1-2018-04-05-22-06.json)입니다.
이 값을 수 밀리초 이내로 줄일 수 있을 것으로 기대하고 있습니다.

* 상호 TLS 비용은 CPU나 지연 시간 모두 AES-NI를 지원하는 하드웨어에서 무시가 가능한 정도입니다.

Istio를 부분적으로 도입하는 사용자를 위한 보다 세부적인 가이드를 준비하고 있습니다.

진행중인 목표로 Istio를 도입했을 때 생기는 CPU 부하와 지연 시간을 줄이고자 합니다.
만약 당신의 어플리케이션이 원격 측정과 정책, 보안, 네트워크 라우팅, A/B 테스팅 등을 이미 관리하고 있다면 그 코드를 제거했을 때 Istio의 비용의 전부는 아닐 지라도 대부분의 비용을 제거해 줄 것입니다.
