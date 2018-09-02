---
title: Istio
---
<script type="application/ld+json">
    {
        "@context": "http://schema.org",
        "@type": "Organization",
        "url": "https://istio.io",
        "logo": "https://istio.io/img/logo.png",
        "sameAs": [
            "https://twitter.com/IstioMesh",
            "https://istio.rocket.chat/"
        ]
    }
</script>
<script type="application/ld+json">
    {
        "@context": "http://schema.org",
        "@type": "WebSite",
        "url": "https://istio.io/",
        "potentialAction": {
            "@type": "SearchAction",
            "target": "https://istio.io/search.html?q={search_term_string}",
            "query-input": "required name=search_term_string"
        }
    }
</script>
<script type="application/ld+json">
    {
      "@context": "http://schema.org/",
      "@type": "Product",
      "name": "Istio",
      "image": [
          "https://istio.io/img/logo.png"
       ],
      "description": "Istio lets you connect, secure, control, and observe services."
    }
</script>
<script>
    document.addEventListener("DOMContentLoaded", function() {
        document.getElementById('card1').style.opacity = 1;

        window.setTimeout(function() {
            document.getElementById('card2').style.opacity = 1;
        }, 375);

        window.setTimeout(function() {
            document.getElementById('card3').style.opacity = 1;
        }, 750);

        window.setTimeout(function() {
            document.getElementById('card4').style.opacity = 1;
        }, 1125);

        window.setTimeout(function() {
            document.getElementById('buttons').style.opacity = 1;
        }, 1500);
    });
</script>

<main class="landing">
    <div class="container-fluid">
        <div class="row justify-content-center">
            {{< inline_image "landing/istio-logo.svg" >}}
            <div style="width: 20rem; margin-left: 3rem">
                <h1 class="hero-label">Istio</h1>
                <h1 class="hero-lead">서비스를 연결하고, 보호하고, 제어하고, 관찰하세요.
            </div>
        </div>
    </div>

    <div class="container-fluid">
        <div class="row justify-content-center">
            <div id="card1" class="card">
                <a href="/docs/concepts/traffic-management/">
                    <div class="card-img-top">
                        {{< inline_image "landing/routing-and-load-balancing.svg" >}}
                    </div>
                    <div class="card-body">
                        <hr class="card-line">
                        <h5 class="card-title text-center">연결</h5>
                        <hr class="card-line">
                        <p class="card-text">
                            지능적으로 트래픽의 흐름과 서비스 사이의 API 호출을 제어하고,
                            넓은 범위의 테스트를 수행하고, red/black 배포를 이용하여 점진적으로 업그레이드하세요.
                        </p>
                    </div>
                </a>
            </div>

            <div id="card2" class="card">
                <a href="/docs/concepts/security/">
                    <div class="card-img-top">
                        {{< inline_image "landing/resiliency.svg" >}}
                    </div>
                    <div class="card-body">
                        <hr class="card-line">
                        <h5 class="card-title text-center">보호</h5>
                        <hr class="card-line">
                        <p class="card-text">
                            당신의 서비스를 자동적으로 관리되는 인증, 인가, 서비스 사이의 암호화된 통신으로 보호하세요.
                        </p>
                    </div>
                </a>
            </div>

            <div id="card3" class="card">
                <a href="/docs/concepts/policies-and-telemetry/">
                    <div class="card-img-top">
                        {{< inline_image "landing/policy-enforcement.svg" >}}
                    </div>
                    <div class="card-body">
                        <hr class="card-line">
                        <h5 class="card-title text-center">제어</h5>
                        <hr class="card-line">
                        <p class="card-text">
                            정책을 적용해 잘 지켜지는지, 그리고 자원이 소비자에게 균등하게 나누어지는지 확인하세요.
                        </p>
                    </div>
                </a>
            </div>

            <div id="card4" class="card">
                <a href="/docs/concepts/policies-and-telemetry/">
                    <div class="card-img-top">
                        {{< inline_image "landing/telemetry-and-reporting.svg" >}}
                    </div>
                    <div class="card-body">
                        <hr class="card-line">
                        <h5 class="card-title text-center">관찰</h5>
                        <hr class="card-line">
                        <p class="card-text">
                            당신의 서비스에서 무엇이 일어나고 있는지 풍부한 자동 추적, 모니터링, 로깅을 활용하여 살펴보세요.
                       </p>
                    </div>
                </a>
            </div>
        </div>
    </div>

    <div id="buttons" class="buttons container-fluid">
        <div class="row justify-content-center">
            <a class="btn btn-istio" href="/docs/concepts/what-is-istio/">LEARN MORE</a>
            <a class="btn btn-istio" href="https://github.com/istio/istio/releases/">DOWNLOAD</a>
        </div>
    </div>
</main>
