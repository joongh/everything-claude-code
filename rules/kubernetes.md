# Kubernetes 클러스터 접근

`kubectl`, `helm` 등으로 Kubernetes 클러스터에 명령을 실행할 때는 **항상 context를 명시적으로 지정**한다. 동시에 여러 클러스터(alpha/beta/real 등)에 작업해야 하는 경우가 있어, 현재 기본 context에 의존하면 의도하지 않은 클러스터에 명령이 나갈 수 있다.

## 지정 방법

`~/.kube` 디렉토리의 `.kubeconfig` 파일을 `--kubeconfig` 플래그로 지정한다.

```bash
# kubectl
kubectl --kubeconfig ~/.kube/alpha.kubeconfig get pods -n my-ns
kubectl --kubeconfig ~/.kube/beta.kubeconfig get nodes
kubectl --kubeconfig ~/.kube/real.kubeconfig logs my-pod

# helm
helm --kubeconfig ~/.kube/alpha.kubeconfig list -n my-ns
helm --kubeconfig ~/.kube/real.kubeconfig upgrade my-release ./chart
```

## 규칙

- `kubectl` / `helm` 명령에는 반드시 `--kubeconfig <path>`를 포함한다
- `~/.kube/config`(기본 파일)에 의존하지 않는다. 어떤 클러스터인지 명령어만 봐도 알 수 있어야 한다
- 어느 클러스터에 실행할지 명확하지 않으면, 실행 전 사용자에게 어느 환경(alpha/beta/real 등)인지 확인한다
- 사용 가능한 kubeconfig 목록은 `ls ~/.kube/*.kubeconfig`로 확인 가능

## 주의

- 변경 작업(`apply`, `delete`, `helm upgrade/install/uninstall` 등)은 `--kubeconfig`가 의도한 환경을 가리키는지 한 번 더 확인한 후 실행한다
- `KUBECONFIG` 환경변수를 export해서 세션에 박아두는 방식은 사용하지 않는다 (다음 명령이 어떤 클러스터에 가는지 명령어만으로 알기 어려워짐)
