# cluster/hardening — 클러스터 하드닝 레이어

Flux `cluster-hardening` Kustomization(`../flux-system/layers.yaml`)이 이 디렉터리를 적용한다.
스캐너(kube-bench, kubescape)와 어드미션 강제 정책(ValidatingAdmissionPolicy)을 **조건별로 선택 적용**한다.

## 구성

```
hardening/
├── kustomization.yaml               # 스캐너·상시정책 + components 토글
├── scanning/
│   ├── namespace.yaml               # security-scanning (PSA privileged)
│   ├── kube-bench.yaml              # CIS 벤치마크 CronJob (+ 전용 SA)
│   └── kubescape-release.yaml       # kubescape-operator (posture 스캔)
├── admission/                       # 상시 강제 VAP (토글과 무관)
│   ├── no-latest.yaml               # :latest/무태그 이미지 차단 (Deny)
│   └── no-default-sa.yaml           # default SA 사용 금지 (Warn/Audit → 승격)
└── components/                      # Kustomize Component = 조건 프로필
    ├── baseline/                    # PSS baseline VAP (전역 관측, opt-out)
    ├── restricted/                  # PSS restricted VAP (켜면 baseline 까지 전역 Deny)
    ├── fsi/                         # 금융보안(FSI) 커스텀 VAP (opt-in, ConfigMap 파라미터)
    └── inject-securitycontext/      # raw 워크로드 securityContext 주입 (재사용)
```

## 상시 강제 정책 (`admission/`)

baseline/restricted 토글과 무관하게 항상 적용된다.

| 정책 | 동작 | 내용 | opt-out 라벨 |
|------|------|------|-------------|
| `hardening-no-latest` | **Deny** | `:latest` 태그 및 무태그(암묵적 latest) 이미지 차단 | `hardening.opencsp.io/no-latest=ignore` |
| `hardening-no-default-sa` | **관측(Warn+Audit)** | 파드가 `default` SA 사용 금지 (배포 전용 SA 요구) | `hardening.opencsp.io/default-sa=ignore` |

> `no-default-sa` 는 우선 관측만 한다. 위반 워크로드를 전용 SA 로 옮긴 뒤 `admission/no-default-sa.yaml`
> 바인딩의 `validationActions` 를 `[Warn, Audit]` → `[Deny]` 로 승격한다.
> `no-latest` 를 켜기 전, `latest` 를 쓰던 워크로드(예: tofu-runner)는 버전 태그로 고정해야 한다.

## 조건(프로필) 선택 적용

`kustomization.yaml`의 `components:` 목록에서 원하는 프로필만 주석 해제한다.

```yaml
components:
  - components/baseline      # 전역 관측 (기본)
  # - components/restricted  # 켜면 baseline 까지 전역 Deny 로 승격
  # - components/fsi         # 금융보안 커스텀 (opt-in ns)
```

각 정책의 **적용 방식/대상**은 아래와 같다. baseline 은 관측 전용, restricted·fsi 는 실제 차단이다.

| 프로필 | 컴포넌트 로드 | 동작 | 범위 | 네임스페이스 라벨 |
|--------|--------------|------|------|-------------------|
| baseline | `components/baseline` | **관측(Warn+Audit)** — 차단 안 함 | 전역 (시스템/opt-out 제외) | opt-out: `hardening.opencsp.io/baseline=ignore` |
| restricted | `components/restricted` | **강제(Deny)** — 켜면 baseline 도 Deny 로 오버라이드 | 전역 (시스템/opt-out 제외) | opt-out: `hardening.opencsp.io/restricted=ignore` |
| fsi | `components/fsi` | **강제(Deny)** | opt-in | `hardening.opencsp.io/fsi=enforce` |

> **baseline 만 켠 상태**는 위반을 관측만 한다(전역). 위반이 없음을 확인한 뒤 그대로 `components/restricted`
> 를 주석 해제하면, restricted 의 kustomize patch 가 baseline 바인딩을 `[Warn, Audit]` → `[Deny]` 로
> 오버라이드하고 restricted 규칙도 전역 Deny 로 적용된다. 즉 **restricted 를 켜는 순간 전역 강제로 전환**된다.
> ⚠ restricted 는 baseline 바인딩을 patch 하므로 `components/baseline` 이 함께 켜져 있어야 한다.

특정 네임스페이스만 강제에서 빼려면 opt-out 라벨을 붙인다:

```bash
kubectl label ns my-namespace hardening.opencsp.io/restricted=ignore
```

예) 특정 네임스페이스에 FSI(opt-in) 추가 강제:

```bash
kubectl label ns my-namespace hardening.opencsp.io/fsi=enforce
```

### FSI 커스텀 조건 편집

신뢰 레지스트리 allowlist 는 `components/fsi/vap-fsi.yaml`의 ConfigMap `hardening-fsi-params`에서 편집한다.

```yaml
data:
  allowedRegistries: "ghcr.io/h001-lab/,registry.k8s.io/,..."
```

## securityContext 일괄 주입 (VAP 는 "차단", 주입은 별도)

VAP 는 비준수 파드를 **거부(Deny)**할 뿐 값을 채워주지 않는다. 주입 경로는 워크로드 종류에 따라 다르다.

### 1) HelmRelease 앱 → `postRenderers`

현재 앱들은 대부분 HelmRelease 이므로 kustomize 패치가 닿지 않는다. HelmRelease 에 postRenderer 를 붙여 렌더링된 파드에 주입한다.

```yaml
spec:
  postRenderers:
    - kustomize:
        patches:
          - target:
              kind: Deployment
            patch: |
              - op: add
                path: /spec/template/spec/securityContext
                value:
                  runAsNonRoot: true
                  seccompProfile:
                    type: RuntimeDefault
```

### 2) raw 매니페스트 → `inject-securitycontext` Component

git 으로 직접 관리하는 Deployment/StatefulSet 등이 있는 레이어의 `kustomization.yaml`에 추가한다.

```yaml
components:
  - ../hardening/components/inject-securitycontext
```

> 파드 레벨(`runAsNonRoot`, `seccompProfile`)만 와일드카드 주입된다. 컨테이너 전용 필드
> (`allowPrivilegeEscalation`, `readOnlyRootFilesystem`, `capabilities`)는 컨테이너별로 명시하거나
> restricted/FSI VAP 로 강제한다.

## 점진 롤아웃 (Deny 전 dry-run)

강한 정책을 바로 Deny 하기 전에, 각 `ValidatingAdmissionPolicyBinding`의
`validationActions` 를 `[Deny]` → `[Warn, Audit]` 로 바꿔 위반만 관측한 뒤 전환한다.

## 스캔 결과 확인

```bash
kubectl -n security-scanning logs job/<kube-bench-job>   # CIS 결과
kubectl -n security-scanning get pods                    # kubescape operator
```

kube-bench 로그는 otel-collector agent(daemonset)가 수집하므로 관측 파이프라인에서도 확인 가능하다.

## 주의

- `cluster-hardening` 은 `cluster-system-configs` 에만 의존하며 `cluster-apps` 를 막지 않는다.
  앱 배포 전 정책 강제를 보장하려면 `cluster-apps.spec.dependsOn` 에 `cluster-hardening` 을 추가한다.
  (단, 기존 앱이 정책을 위반하면 배포가 막히므로 baseline 부터 단계적으로 적용할 것)
