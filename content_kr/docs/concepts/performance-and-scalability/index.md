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

We use Go’s native tools to write targeted micro-benchmarks in performance sensitive areas. Our main goal with this approach is to provide easy-to-use micro-benchmarks that developers can use to perform quick before/after performance comparisons for their changes.

See the [sample micro-benchmark]({{< github_file >}}/mixer/test/perf/singlecheck_test.go) for Mixer that measures the performance of attribute processing code.

Developers can also utilize a golden-files approach to capture the state of their benchmark results in the source tree for keeping track and  referencing purposes. GitHub has this [baseline file]({{< github_file >}}/mixer/test/perf/bench.baseline).

Due to the nature of this testing type, there is a high-variance in latency numbers across machines. It is recommended that micro-benchmark numbers captured in this way are compared only against the previous runs on the same machine.

The [`perfcheck.sh` script]({{< github_file >}}/bin/perfcheck.sh) can be used to quickly run benchmarks in a sub-folder and compare its results against the co-located baseline files.

## Testing scenarios

{{< image width="80%" ratio="75%"
    link="https://raw.githubusercontent.com/istio/istio/master/tools/perf_setup.svg?sanitize=true"
    alt="Performance scenarios diagram"
    caption="Performance scenarios diagram"
    >}}

The synthetic benchmark scenarios and the source code of the tests are described
on [GitHub]({{< github_blob >}}/tools#istio-load-testing-user-guide)

<!-- add blueperf and more details -->

## Synthetic end to end benchmarks

We use Fortio as Istio's synthetic end to end load testing tool. Fortio runs at a specified query per second (qps) and records an histogram of execution time and calculates percentiles (e.g. p99 i.e. the response time such as 99% of the requests take less than that number (in seconds, SI unit)). It can run for a set duration, for a fixed number of calls, or until interrupted (at a constant target QPS, or max speed/load per connection/thread).

Fortio is a fast, small, reusable, embeddable go library as well as a command line tool and server process, the server includes a simple web UI and graphical representation of the results (both a single latency graph and a multiple results comparative min, max, average and percentiles graphs).

Fortio is also 100% open-source and with no external dependencies beside go and gRPC so you can reproduce all our results easily and add your own variants or scenarios you are interested in exploring.

Here is an example of scenario (one out of the 8 scenarios we run for every build) result graphing the latency distribution for istio-0.7.1 at 400 Query-Per-Second (qps) between 2 services inside the mesh (with mutual TLS, Mixer policy checks and telemetry collection):

<iframe src="https://fortio.istio.io/browse?url=qps_400-s1_to_s2-0.7.1-2018-04-05-22-06.json&xMax=105&yLog=true" width="100%" height="1024" scrolling="no" frameborder="0"></iframe>

Comparing 0.6.0 and 0.7.1 histograms/response time distribution for the same scenario, clearly showing 0.7 improvements:

<iframe src="https://fortio.istio.io/?xMin=2&xMax=110&xLog=true&sel=qps_400-s1_to_s2-0.7.1-2018-04-05-22-06&sel=qps_400-s1_to_s2-0.6.0-2018-04-05-22-33" width="100%" height="1024" scrolling="no" frameborder="0"></iframe>

And tracking the progress across all the tested releases for that scenario:

<iframe src="https://fortio.istio.io/?s=qps_400-s1_to_s2" width="100%" height="1024" scrolling="no" frameborder="0"></iframe>

You can learn more about [Fortio](https://github.com/istio/fortio/blob/master/README.md#fortio) on GitHub and see results on [https://fortio.istio.io](https://fortio.istio.io).

## Realistic application benchmark

Acmeair (a.k.a, BluePerf) is a customer-like microservices application implemented in Java. This application runs on WebSphere Liberty and simulates the operations of a fictitious airline.

Acmeair is composed by the following microservices:

* **Flight Service** retrieves flight route data. It is called by the Booking service to check miles for the rewards operations (Acmeair customer fidelity program).

* **Customer Service** stores, updates, and retrieves customer data. It is invoked by the Auth service for login and by the Booking service for the rewards operations.

* **Booking Service** stores, updates, and retrieves booking data.

* **Auth Service** generates JWT if the user/password is valid.

* **Main Service** primarily consists of the presentation layer (web pages) that interact with the other services. This allows the user to interact directly with the application via browser, but it is not exercised during the load test.

The diagram below represents the different pods/containers of the application in the Kubernetes/Istio environment:

{{< image width="100%" ratio="80%"
    link="https://ibmcloud-perf.istio.io/regpatrol/istio_regpatrol_readme_files/image004.png" alt="Acmeair microservices overview"
    >}}

The following table shows the transactions that are driven by the script during the regression test and the approximate distribution of the requests:

{{< image width="100%" ratio="20%"
    link="https://ibmcloud-perf.istio.io/regpatrol/istio_regpatrol_readme_files/image006.png" alt="Acmeair request types and distribution"
    >}}

The Acmeair benchmark application can be found here: [IBM's BluePerf](https://github.com/blueperf).

## Automation

Both the synthetic benchmarks (fortio based) and the realistic application (BluePerf)
are part of the nightly release pipeline and you can see the results on:

* [https://fortio-daily.istio.io/](https://fortio-daily.istio.io/)
* [https://ibmcloud-perf.istio.io/regpatrol/](https://ibmcloud-perf.istio.io/regpatrol/)

This enables us to catch regression early and track improvements over time.

## Scalability and sizing guide

* Setup multiple replicas of the control plane components.

* Setup [Horizontal Pod Autoscaling](https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale/)

* Split mixer check and report pods.

* High availability (HA).

* See also [Istio's Performance oriented FAQ](https://github.com/istio/istio/wiki/Istio-Performance-oriented-setup-FAQ)

* And the [Performance and Scalability Working Group](https://github.com/istio/community/blob/master/WORKING-GROUPS.md#performance-and-scalability) work.

Current recommendations (when using all Istio features):

* 1 vCPU per peak thousand requests per second for the sidecar(s) with access logging (which is on by default) and 0.5 without, `fluentd` on the node is a big contributor to that cost as it captures and uploads logs.

* Assuming typical cache hit ratio (>80%) for mixer checks: 0.5 vCPU per peak thousand requests per second for the mixer pods.

* Latency cost/overhead is approximately [10 millisecond](https://fortio.istio.io/browse?url=qps_400-s1_to_s2-0.7.1-2018-04-05-22-06.json) for service-to-service (2 proxies involved, mixer telemetry and checks) as of 0.7.1, we expect to bring this down to a low single digit ms.

* Mutual TLS costs are negligible on AES-NI capable hardware in terms of both CPU and latency.

We plan on providing more granular guidance for customers adopting Istio "A la carte".

We have an ongoing goal to reduce both the CPU overhead and latency of adding Istio to your application. Please note however that if you application is
handling its own telemetry, policy, security, network routing, a/b testing, etc... all that code and cost can be removed and that should offset most if
not all of the Istio overhead.
