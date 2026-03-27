PREREQUISITES
  • Docker Desktop running
  • kind installed (kind create cluster --name testing)
  • kubectl installed
  • Helm installed

── STEP 1: Create kind cluster ──────────────────────────────────────────

  kind create cluster --name testing
  kubectl config use-context kind-testing

  Verify:
    kubectl get nodes

── STEP 2: Install Envoy Gateway via Helm ───────────────────────────────

  helm install eg oci://docker.io/envoyproxy/gateway-helm \
    --version v1.3.2 \
    -n envoy-gateway-system --create-namespace

  Wait for readiness:
    kubectl wait --timeout=5m \
      -n envoy-gateway-system \
      deployment/envoy-gateway \
      --for=condition=Available

  Verify:
    kubectl get pods -n envoy-gateway-system

── STEP 3: Create observability namespace ───────────────────────────────

  kubectl apply -f manifests/01-namespace.yaml

── STEP 4: Deploy demo app (nginx) ──────────────────────────────────────

  kubectl apply -f manifests/02-demo-app.yaml
  kubectl rollout status deployment/demo-app -n default --timeout=120s

  Verify:
    kubectl get pods -n default

── STEP 5: Deploy Jaeger ────────────────────────────────────────────────

  kubectl apply -f manifests/04-jaeger.yaml
  kubectl rollout status deployment/jaeger -n observability --timeout=180s

  Verify:
    kubectl get pods -n observability

── STEP 6: Apply EnvoyProxy telemetry config ────────────────────────────

  This is the key config that enables both access logging and tracing.

  kubectl apply -f manifests/05-envoyproxy-telemetry.yaml

  What it does:
    • Enables JSON access logs → stdout
    • Enables OTLP tracing → jaeger:4317
    • 100% sampling rate

── STEP 7: Apply Gateway resources ──────────────────────────────────────

  kubectl apply -f manifests/03-gateway.yaml

  kubectl rollout restart deployment/envoy-gateway -n envoy-gateway-system
  kubectl rollout status deployment/envoy-gateway -n envoy-gateway-system

── STEP 8: Verify everything is running ─────────────────────────────────

  kubectl get pods -A | grep -E "Running|envoy|jaeger"

  Expected:
    envoy-gateway-system   envoy-gateway-*              1/1   Running
    envoy-gateway-system   envoy-default-demo-gateway-* 2/2   Running
    observability          jaeger-*                     1/1   Running
    default                demo-app-*                   1/1   Running


━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
CHAPTER 6 — LOGGING DEMO (STEP BY STEP)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━


── DEMO STEP 1: Port-forward the Envoy proxy ────────────────────────────

  Run this in one terminal and keep it open:

    kubectl -n envoy-gateway-system \
      port-forward svc/envoy-default-demo-gateway-88752a31 10080:80


── DEMO STEP 2: Open another terminal and tail the logs ─────────────────

    kubectl logs -n envoy-gateway-system \
      deploy/envoy-default-demo-gateway-88752a31 -f

── DEMO STEP 3: Send requests and watch logs appear ─────────────────────

  In a third terminal:

    for i in {1..5}; do   curl http://127.0.0.1:8080/;   sleep 1; done


  Watch log lines appear in the second terminal.

── DEMO STEP 4: Explain a log line ──────────────────────────────────────


  {
    "method":         "GET",
    "path":           "/",
    "protocol":       "HTTP/1.1",
    "response_code":  200,
    "response_flags": "-",
    "bytes_sent":     615,
    "duration_ms":    5,
    "upstream_host":  "10.244.0.13:80",
    "request_id":     "f1cc8cb5-5e0a-9b93-b609-c5beea1df038",
    "trace_id":       "2894e1351d37b08227eb95249fc8590d"
  }

  Talk through
    method         → client sent a GET request
    path           → to the / (root) path
    response_code  → Envoy returned 200 success
    response_flags → '-' means no Envoy error, clean pass-through
    bytes_sent     → 615 bytes were returned to client
    duration_ms    → entire round-trip was 5 milliseconds
    upstream_host  → backend pod at 10.244.0.13 port 80 served it
    request_id     → unique ID to identify just this request
    trace_id       → links this log to a Jaeger trace span

── DEMO STEP 5: Demonstrate a response flag ─────────────────────────────

  Scale demo-app to 0 (simulate down):
    kubectl scale deploy demo-app -n default --replicas=0

  Send a request:
    Invoke-WebRequest -UseBasicParsing http://127.0.0.1:10080/

  Look at log line:
    "response_code":503, "response_flags":"UH"
    UH = Upstream Unhealthy — no backend pods available

  Restore:
    kubectl scale deploy demo-app -n default --replicas=1

── DEMO KEY POINTS TO MENTION ───────────────────────────────────────────

  1. Logs go to stdout — viewable with kubectl logs
  2. JSON format is better than text for searching and filtering
  3. Every request is logged, even failures
  4. request_id links the log to what happened on the client side
  5. trace_id links the log to the Jaeger distributed trace


━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
CHAPTER 7 — TRACING DEMO (STEP BY STEP)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

── DEMO STEP 1: Port-forward Jaeger UI ──────────────────────────────────

  kubectl -n observability port-forward svc/jaeger 16686:16686

  Open browser: http://localhost:16686


── DEMO STEP 2: Generate traffic ────────────────────────────────────────

  1..20 | ForEach-Object {
      Invoke-WebRequest -UseBasicParsing http://127.0.0.1:10080/ | Out-Null
      Start-Sleep -Milliseconds 300
  }


── DEMO STEP 3: Open Jaeger Search page ─────────────────────────────────

  In Jaeger UI:
    Service   → demo-gateway.default
    Operation → all
    Lookback  → Last 1 hour
    Limit     → 20

  Click Find Traces.

── DEMO STEP 4: Open one trace ──────────────────────────────────────────

  Walk through the top bar:
    Trace Start  → timestamp of when request arrived
    Duration     → total time from entry to response
    Services     → which services were involved (1 in our case)
    Total Spans  → 2 (ingress + egress)

── DEMO STEP 5: Explain the span tree ───────────────────────────────────

  Span tree:
    demo-gateway.default   ingress
      demo-gateway.default   router httproute/.../demo-route/rule/0 egress


── DEMO STEP 6: Click the egress span ───────────────────────────────────

  Click the span row. Tags expand below.

  Walk through tags:
    component        = proxy         → generated by Envoy proxy
    http.protocol    = HTTP/1.1      → request used HTTP/1.1
    http.status_code = 200           → backend returned success
    peer.address     = 10.244.0.13:80 → exact backend pod IP and port
    response_flags   = -             → no Envoy errors
    span.kind        = client        → Envoy was the client calling the backend
    upstream_address = 10.244.0.13:80 → same backend target
    upstream_cluster = httproute/default/demo-route/rule/0
                                     → the route name that matched

  Process section:
    otel.library.name    = envoy     → spans generated by Envoy itself
    otel.library.version = 1f24...   → Envoy build version



── DEMO STEP 7: Click the ingress span ──────────────────────────────────

  Tags include:
    http.method    = GET
    http.url       = http://127.0.0.1:10080/
    http.status_code = 200
    peer.address   = 127.0.0.1       → the client IP
    span.kind      = server          → Envoy received this request
    guid:x-request-id               → same as request_id in access log
    user_agent     = WindowsPowerShell/5.1.x
    request_size   = 0
    response_size  = 615


── DEMO STEP 8: Correlate log and trace ─────────────────────────────────

  Show side by side (use scripts/correlate.ps1):

  ACCESS LOG line:
    "request_id": "6e7cc93f-c7c3-99bb-9579-026d80a5e799"
    "trace_id"  : "bea633c740c792f0290f74ad556fef6d"

  JAEGER trace:
    Trace ID : bea633c740c792f0290f74ad556fef6d
    Span tag : guid:x-request-id = 6e7cc93f-c7c3-99bb-9579-026d80a5e799




