# deploy-infra

> 포트폴리오 공개용 스냅샷입니다. 실제 IP·도메인·조직명은 예시값(`10.0.0.x`, `example.ac.kr`, `myorg`)으로 바꿨고 비밀값은 포함하지 않습니다. 전체 소개는 [k8s-manager](https://github.com/milk024/k8s-manager) 를 보세요.

`infra` 네임스페이스 서비스의 **배포 상태 기준 저장소**. ArgoCD 가 이 저장소를 읽어 클러스터에 동기화한다.

| 앱 | 경로 | 소스 저장소 | 네임스페이스 |
|---|---|---|---|
| k8s-manager | `apps/k8s-manager` | `myorg/k8s-manager` | infra |
| monitoring-web | `apps/monitoring-web` | `myorg/monitoring-web` | infra |

## 흐름

```
소스 저장소 push(main) → CI 빌드·이미지 push → 이 저장소 apps/<앱>/values.yaml 의 image.tag 갱신 커밋(infra-ci)
                                                → ArgoCD 가 감지해 동기화
```

## 규칙

- **main 직접 push**: CI 계정 `infra-ci`(image.tag 갱신), `infra-maintainers`(담당자 E), `Owners`(양진형). 인프라 담당이 1~2명이라 승인 절차 없음.
  rag·vkey 배포 저장소는 PR + 각 팀 maintainers(`rag-maintainers`/`vkey-maintainers`) 1명 승인.
- **비밀값을 커밋하지 않는다.** 아래 Secret 은 클러스터에서 별도 관리한다.
  - `k8s-manager-secret` (`GITEA_WEBHOOK_SECRET`)
  - `k8s-manager-gitea-token` (`GITEA_TOKEN`, Gitea 계정 `k8s-manager-svc`)
  - `gitea-registry-cred`, `k8s-manager-registry-secret` (이미지 pull)
- 클러스터에서 `kubectl edit` / `helm upgrade` 로 직접 바꾸지 않는다. ArgoCD 가 이 저장소 기준으로 되돌린다.
- 이미지 레지스트리는 Gitea 컨테이너 레지스트리(`gpu.example.ac.kr/myorg/...`). Harbor 전환 계획 없음.

## ArgoCD

- `argocd/project-infra.yaml` — AppProject `infra` (이 저장소만 읽고 `infra` 네임스페이스에만 배포)
- `argocd/app-*.yaml` — Application. **자동 동기화(prune + selfHeal)**: 클러스터를 직접 고치면 수 초 안에 되돌아간다.
- 이 매니페스트 자체는 아직 수동 적용(`kubectl apply -f argocd/`). 앱이 늘면 app-of-apps 로 전환.
- 접속: `kubectl port-forward -n argocd svc/argocd-server 8080:443` → https://localhost:8080
- Helm 이력: 이전 helm 릴리스 Secret 은 삭제하지 않고 `owner=helm-archived` 로 라벨만 바꿔 보관 (helm 이 더 이상 이 앱을 관리하지 않음)

## 로컬 확인

```bash
helm template k8s-manager apps/k8s-manager -n infra | kubectl diff -f -
```

## 이력

- 2026-11-12: ArgoCD Application 등록 → 수동 동기화(spec 차이 0, Pod 재시작 없음) → helm 이력 보관 → 자동 동기화 활성화
- 2026-11-12: 각 소스 저장소의 `k8s/` 차트와 실제 클러스터 값(helm release)에서 이전. 이전 시점 `kubectl diff` = 0.
