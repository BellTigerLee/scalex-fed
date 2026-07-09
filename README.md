# scalex-fed

`scalex-fed`는 ScaleX federation layer 저장소입니다. 여러 ScaleX child
cluster를 하나의 federation 의도로 묶기 위한 Karmada/GitOps 소스를
보관합니다.

이 저장소는 현재 `tower-k8s`에서 자동으로 동작하지 않습니다. `tower-k8s`가
나중에 Argo CD Application source로 이 저장소의 chart path를 바라볼 수 있도록
구조를 준비한 상태입니다.

## 역할

`scalex-fed`가 소유하는 범위는 federation intent입니다.

- Karmada `ClusterPropagationPolicy` 및 `ClusterOverridePolicy`
- 여러 child cluster에 공통으로 적용할 multi-cluster app grouping
- `b-k8s`, `c-k8s` 같은 member cluster를 대상으로 하는 placement/override rule
- 향후 공통 앱을 어떤 cluster group에 전파할지에 대한 GitOps 선언

`scalex-fed`는 child cluster 자체의 preset이나 control-plane bootstrap을
소유하지 않습니다.

## 저장소 경계

| 저장소 | 역할 |
| --- | --- |
| `scalex-fed` | ScaleX federation intent, Karmada policy, shared multi-cluster grouping |
| `b-k8s` | `b` child cluster preset |
| `c-k8s` | `c` child cluster preset |
| `eecs-k8s` | 공통 base/framework repo |
| `tower-k8s` | Argo CD/Karmada control-plane cluster preset. 향후 이 저장소의 chart path를 Argo Application source로 참조 |

## 현재 구성

```text
apps/
`-- scalex-federation/
    |-- Chart.yaml
    |-- README.md
    |-- values.yaml
    |-- examples/
    `-- templates/
```

`apps/scalex-federation` chart는 `tower-k8s`의 future Argo Application에서
`source.path: apps/scalex-federation`로 가리킬 수 있도록 준비한 최소 Helm
application chart입니다.

## 안전 기본값

기본값으로 example policy rendering은 꺼져 있습니다.

```yaml
examplePolicies:
  enabled: false
```

따라서 chart를 그대로 render해도 Karmada policy 리소스가 생성되지 않습니다.
예시는 기존 Karmada member label인 `scalex.io/role=child`,
`scalex.io/topology=baremetal`, `scalex.io/site=b|c`를 문서화하지만, 실제
적용은 명시적으로 enable해야 합니다.

실제 workload, secret, kubeconfig, bearer token, credential은 이 저장소에
두지 않습니다.

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
path: apps/scalex-federation
targetRevision: main
```

### 2. Argo CD가 Karmada API를 destination으로 알게 하기

Karmada policy 리소스는 Tower host cluster가 아니라 Karmada API server에
적용되어야 합니다. 따라서 Tower Argo CD에 Karmada API를 cluster destination으로
등록해야 합니다.

권장 destination 이름은 다음처럼 둡니다.

```yaml
destination:
  name: karmada
```

이 이름은 예시일 뿐이며, 실제 Argo CD cluster secret의 이름과 맞아야 합니다.
Karmada API가 Argo CD cluster로 등록되어 있지 않으면 `scalex-fed` Application은
정상 sync되지 않습니다.

### 3. AppProject 허용 범위 확인

`tower-k8s`의 Application이 `project: tower-ops`를 사용한다면, 해당 AppProject가
다음을 허용해야 합니다.

- source repo: `https://github.com/BellTigerLee/scalex-fed.git`
- destination cluster: Karmada API destination 이름
- Karmada policy resource: `policy.karmada.io/*`

AppProject가 source/destination/resource를 제한하지 않는다면 별도 변경이 필요
없을 수 있습니다.

### 4. tower-k8s/apps/values.yaml에 Application 추가

`tower-k8s`는 `apps/values.yaml`의 `apps:` 목록을 읽어 Argo CD Application을
생성합니다. Karmada 설치가 sync wave 20, member join이 sync wave 30이므로
federation policy app은 그 뒤인 sync wave 40 정도가 적절합니다.

초기 연결 검증용 예시는 다음과 같습니다.

```yaml
apps:
  scalex-federation:
    enabled: true
    name: tower-scalex-federation
    namespace: scalex-federation-system
    project: tower-ops
    destination:
      name: karmada
    source:
      repoURL: https://github.com/BellTigerLee/scalex-fed.git
      path: apps/scalex-federation
      targetRevision: main
      releaseName: scalex-federation
      helm:
        valuesObject:
          examplePolicies:
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

처음에는 `examplePolicies.enabled: false`로 연결만 확인합니다. 이 상태에서는
chart가 policy 리소스를 만들지 않으므로 안전합니다.

## 실제 policy를 켜기 전에 필요한 것

지금 chart의 policy는 placeholder `ConfigMap`만 선택합니다. 실제 공통 앱을
`b`, `c` cluster에 전파하려면 다음을 먼저 정해야 합니다.

1. `scalex-fed`가 보관할 공통 앱 manifest 또는 chart 위치
2. Karmada가 선택할 resource label/name/namespace
3. `b`, `c`에 동일하게 보낼지, cluster별 override가 필요한지
4. `ClusterPropagationPolicy`를 쓸지, namespace-scoped `PropagationPolicy`를 쓸지
5. `ClusterOverridePolicy`가 필요한 경우 어떤 field/image/annotation을 바꿀지

그 다음에 `values.yaml`의 selector를 실제 resource에 맞게 바꾸고
`examplePolicies.enabled: true` 또는 별도 production policy 값을 추가합니다.

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

`apps/scalex-federation`의 기본 selector는 이 label을 기준으로 `b`, `c`를
대상 cluster 후보로 잡습니다.
