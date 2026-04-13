# Playbook: New Kubernetes Controller

Пошаговый процесс создания нового K8s controller/operator.
Подходит для задач, где нужно расширить поведение Kubernetes через custom controller.

---

## Context

**Когда использовать:** Нужна автоматизация поведения в кластере, которую нельзя решить стандартными ресурсами (Deployment, Job, CronJob).
**Prerequisite:** Go >= 1.21, kubebuilder или operator-sdk, доступ к dev-кластеру, базовое понимание controller-runtime.

---

## Steps

### 1. Define the problem

- Какое поведение автоматизируем?
- Нужен ли CRD (Custom Resource Definition), или достаточно watch за стандартными ресурсами?
- Записать Goal Statement + Constraints (см. `docs/ai-workflow.md`)

### 2. Scaffold project

```bash
# С kubebuilder
kubebuilder init --domain example.com --repo github.com/org/controller-name
kubebuilder create api --group apps --version v1alpha1 --kind MyResource

# Или с operator-sdk
operator-sdk init --domain example.com --repo github.com/org/controller-name
operator-sdk create api --group apps --version v1alpha1 --kind MyResource
```

### 3. Define CRD spec

В `api/v1alpha1/myresource_types.go`:

- `Spec` — что пользователь хочет (desired state)
- `Status` — текущее состояние (observed state)
- Добавить `kubebuilder` маркеры для validation
- Добавить `+kubebuilder:printcolumn` для `kubectl get` output

```go
// MyResourceSpec defines the desired state
type MyResourceSpec struct {
    // Replicas is the desired number of instances
    // +kubebuilder:validation:Minimum=1
    // +kubebuilder:validation:Maximum=10
    Replicas int32 `json:"replicas"`
}

// MyResourceStatus defines the observed state
type MyResourceStatus struct {
    // ReadyReplicas is the number of ready instances
    ReadyReplicas int32 `json:"readyReplicas,omitempty"`
    // Conditions represent the latest observations
    Conditions []metav1.Condition `json:"conditions,omitempty"`
}
```

### 4. Implement reconcile loop

В `internal/controller/myresource_controller.go`:

- Идемпотентность — reconcile можно вызывать повторно без side effects
- Error handling — return `ctrl.Result{RequeueAfter: ...}` для retry
- Status updates — обновлять status conditions после каждого reconcile
- Logging — structured logging через `log.FromContext(ctx)`

```go
func (r *MyResourceReconciler) Reconcile(ctx context.Context, req ctrl.Request) (ctrl.Result, error) {
    log := log.FromContext(ctx)

    // 1. Fetch the resource
    // 2. Check if deleted (handle finalizers)
    // 3. Reconcile desired state
    // 4. Update status
    // 5. Return result
}
```

### 5. Add RBAC markers

```go
//+kubebuilder:rbac:groups=apps.example.com,resources=myresources,verbs=get;list;watch;create;update;patch;delete
//+kubebuilder:rbac:groups=apps.example.com,resources=myresources/status,verbs=get;update;patch
//+kubebuilder:rbac:groups=apps.example.com,resources=myresources/finalizers,verbs=update
```

Только минимальные permissions (Rule 13).

### 6. Write tests

```bash
# Unit tests для reconcile логики
go test ./internal/controller/... -v

# Integration tests через envtest
# (kubebuilder scaffold включает setup для envtest)
go test ./... -v -count=1
```

### 7. Build container image

```dockerfile
# Multi-stage build
FROM golang:1.21 AS builder
WORKDIR /workspace
COPY go.mod go.sum ./
RUN go mod download
COPY . .
RUN CGO_ENABLED=0 GOOS=linux go build -o manager cmd/main.go

FROM gcr.io/distroless/static:nonroot
COPY --from=builder /workspace/manager .
USER 65532:65532
ENTRYPOINT ["/manager"]
```

Не `latest` — конкретный тег (Rule 14 — security).

### 8. Deploy manifests

- Helm chart или kustomize overlays
- RBAC: ClusterRole/Role + Binding с минимальными permissions
- SecurityContext: `runAsNonRoot`, `readOnlyRootFilesystem` (Rule 14)
- Resource requests и limits
- Liveness/readiness probes на `/healthz` (Rule 8)

### 9. Validate in dev cluster

```bash
# Deploy CRDs
make install

# Run locally against dev cluster
make run

# Create sample CR
kubectl apply -f config/samples/

# Check status
kubectl get myresources -o wide
kubectl describe myresource sample

# Check controller logs
kubectl logs -l control-plane=controller-manager -n system
```

### 10. PR & Review

- Пройти PR checklist (`docs/pr-checklist.md`)
- Убедиться, что CRD проходит `kubeconform`
- Проверить RBAC — нет лишних permissions

---

## Verification

- [ ] CRD установлен и принимает CR
- [ ] Controller запускается без ошибок
- [ ] Reconcile loop идемпотентный
- [ ] Status обновляется корректно
- [ ] RBAC минимальный
- [ ] Unit tests проходят
- [ ] Integration tests (envtest) проходят
- [ ] Container image собирается
- [ ] SecurityContext restricted

---

## Common Pitfalls

- **Не-идемпотентный reconcile** — controller будет создавать дубликаты ресурсов при каждом reconcile
- **Забытые finalizers** — ресурс не удалится корректно, если нужен cleanup
- **Слишком широкий RBAC** — `*` в verbs или resources — security risk
- **Отсутствие rate limiting** — без `RequeueAfter` controller может DDoS'ить API server
- **Status update без retry** — конфликт при concurrent update Status subresource
- **No owner references** — дочерние ресурсы не удалятся при удалении родительского CR
