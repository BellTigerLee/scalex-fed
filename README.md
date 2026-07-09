# scalex-fed

`scalex-fed`는 ScaleX federation layer 저장소입니다. 여러 ScaleX child
cluster를 하나의 federation 의도로 묶기 위한 workload source, Karmada policy,
Tower-facing GitOps entrypoint를 보관합니다.

이 변경은 `scalex-fed` 내부 구조만 준비합니다. `tower-k8s` 자체 설정은 이
저장소 변경에서 수정하지 않습니다.

## 모델

`tower-k8s` Argo CD가 이 저장소의 `apps/federation` chart를 sync합니다.
`apps/federation`은 직접 workload나 Karmada policy를 render하지 않는 얇은
app-of-apps entrypoint입니다. Helm chart는 chart root 밖의 파일을 include할 수
없으므로, top-level `workloads/`나 `policies/` 파일을 직접 template처럼 다루지
않습니다.

대신 `apps/federation`은 명시적으로 enable된 경우에만 child Argo CD
Application을 만들 수 있습니다.

- `workloads.enabled: true`이면 child Application이 `workloads/examples`를 source로 가리킵니다.
- `policies.enabled: true`이면 child Application이 `policies/examples`를 source로 가리킵니다.
- 기본값은 둘 다 `false`입니다. 따라서 기본 render는 child Application을 만들지 않습니다.

`policies/`의 Karmada policy는 tower control plane에 적용되어
`workloads/`의 resource를 selector로 고르고, 선택된 workload를 Karmada member
cluster `b` 및 `c` 같은 대상 cluster로 전파합니다.

## 저장소 경계

| 저장소 | 역할 |
| --- | --- |
| `scalex-fed` | ScaleX federation workload examples, Karmada policy examples/templates, Tower-facing federation entrypoint |
| `b-k8s` | `b` child cluster preset |
| `c-k8s` | `c` child cluster preset |
| `eecs-k8s` | 공통 base/framework repo |
| `tower-k8s` | Argo CD/Karmada control-plane cluster preset. 이 저장소의 `apps/federation` path를 Argo Application source로 참조할 수 있음 |

`scalex-fed`는 child cluster 자체의 preset이나 control-plane bootstrap을
소유하지 않습니다.

## 현재 구성

```text
README.md
apps/
`-- federation/
    |-- Chart.yaml
    |-- README.md
    |-- values.yaml
    `-- templates/
        `-- applications.yaml
workloads/
|-- README.md
`-- examples/
    |-- README.md
    |-- namespace.yaml
    `-- scalex-federation-example-placeholder.configmap.yaml
policies/
|-- README.md
|-- examples/
|   |-- cluster-override-policy.child-baremetal.example.yaml
|   `-- cluster-propagation-policy.child-baremetal.example.yaml
`-- templates/
    `-- cluster-policies.example.helm.yaml
```

## 역할별 디렉터리

### `workloads/`

Karmada policy가 선택하고 전파할 source workload resource를 둡니다. 현재는
production app이 아니라 `scalex-federation-example-placeholder` ConfigMap만
있습니다. 이 placeholder는 policy selector 검증을 위한 안전한 예시이며 실제
서비스, secret, kubeconfig, bearer token, credential을 포함하지 않습니다.

### `policies/`

Karmada `ClusterPropagationPolicy` 및 `ClusterOverridePolicy` 예시와 template
source를 둡니다. 예시는 기존 Karmada member label인
`scalex.io/role=child`, `scalex.io/topology=baremetal`, `scalex.io/site=b|c`를
사용합니다.

### `apps/`

`tower-k8s`가 Argo CD Application source로 바라볼 Helm entrypoint를 둡니다.
`apps/federation` chart는 policy dumping ground가 아니며, top-level source
paths로 향하는 child Application만 조건부로 생성합니다.

## 안전 기본값

`apps/federation/values.yaml`의 기본값은 아무 child Application도 생성하지
않습니다.

```yaml
workloads:
  enabled: false
policies:
  enabled: false
```

child Application sync automation도 기본적으로 꺼져 있고, automation을 켜더라도
`prune` 기본값은 `false`입니다.

```yaml
syncPolicy:
  automated:
    enabled: false
    prune: false
    selfHeal: false
```

## tower-k8s에서 필요한 설정

`scalex-fed`를 실제로 tower control plane에 붙이려면 `scalex-fed`가 아니라
`tower-k8s` 쪽 설정이 필요합니다. 지금 단계에서는 `tower-k8s`를 수정하지
않았으므로 아래 항목은 후속 작업입니다.

### 1. Argo CD가 scalex-fed repo를 읽을 수 있게 하기

`scalex-fed`가 public repository이면 별도 credential 없이 접근할 수 있습니다.
private repository이면 Tower Argo CD에 repository credential을 등록해야 합니다.

예상 source repo는 다음 형태입니다.

```yaml
repoURL: https://github.com/BellTigerLee/scalex-fed.git
path: apps/federation
targetRevision: main
```

### 2. Tower Argo CD Application 추가

`tower-k8s`는 `apps/values.yaml`의 `apps:` 목록을 읽어 Argo CD Application을
생성합니다. Karmada 설치가 sync wave 20, member join이 sync wave 30이라면
federation entrypoint app은 그 뒤인 sync wave 40 정도가 적절합니다.

초기 연결 검증용 예시는 다음과 같습니다.

```yaml
apps:
  scalex-federation:
    enabled: true
    name: tower-scalex-federation
    namespace: argo
    project: tower-ops
    destination:
      name: tower
    source:
      repoURL: https://github.com/BellTigerLee/scalex-fed.git
      path: apps/federation
      targetRevision: main
      releaseName: scalex-federation
      helm:
        valuesObject:
          workloads:
            enabled: false
          policies:
            enabled: false
    annotations:
      argocd.argoproj.io/sync-wave: "40"
    syncPolicy:
      automated:
        enabled: true
        selfHeal: true
        prune: false
      syncOptions:
        - CreateNamespace=true
        - ServerSideApply=true
        - RespectIgnoreDifferences=true
        - SkipDryRunOnMissingResource=true
```

처음에는 `workloads.enabled: false` 및 `policies.enabled: false`로 연결만
확인합니다. 이 상태에서는 `apps/federation` chart가 child Application을 만들지
않으므로 안전합니다.

## 예시를 켜기 전에 필요한 것

현재 policy 예시는 placeholder `ConfigMap`만 선택합니다. 실제 공통 앱을 `b`,
`c` cluster에 전파하려면 다음을 먼저 정해야 합니다.

1. `scalex-fed`가 보관할 공통 app manifest 또는 chart 위치
2. Karmada가 선택할 resource label/name/namespace
3. `b`, `c`에 동일하게 보낼지, cluster별 override가 필요한지
4. `ClusterPropagationPolicy`를 쓸지, namespace-scoped `PropagationPolicy`를 쓸지
5. `ClusterOverridePolicy`가 필요한 경우 어떤 field/image/annotation을 바꿀지

그 다음에 `workloads/examples` 및 `policies/examples`를 참고해 production용
source와 policy를 별도 path에 추가하고, `apps/federation` child Application을
명시적으로 enable합니다.

## 현재 member cluster label

현재 `tower-k8s`의 Karmada member join 설정은 `b`, `c` cluster에 다음 label을
붙이는 구조입니다.

```yaml
scalex.io/role: child
scalex.io/topology: baremetal
scalex.io/site: b
```

```yaml
scalex.io/role: child
scalex.io/topology: baremetal
scalex.io/site: c
```

`policies/examples`의 기본 selector는 이 label을 기준으로 `b`, `c`를 대상
cluster 후보로 잡습니다.
