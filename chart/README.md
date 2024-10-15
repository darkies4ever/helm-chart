### 설치방법

<aside>
✅ 튜토리얼 설치 방법은 연습을 위해 설정이 되어 있으므로 운영에서 사용할 때는 설정을 검토해야 합니다. 그리고 운영환경은 안전성을 위해 고가용 설정이 필요합니다.

</aside>

argocd namespace를 생성하고 공식문서 가이드(https://argo-cd.readthedocs.io/en/stable/#getting-started)에서 제공하는 yaml파일을 kubectl apply해주면 됩니다.z

```bash
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

![1.webp](https://prod-files-secure.s3.us-west-2.amazonaws.com/76eb865f-1e9a-440c-9d79-e386fea4df4f/a59c8d74-d01e-44e2-aa39-0d5846335b42/1.webp)

### 설치확인

```bash
kubectl -n argocd get po,service,configmap,secret
```

![2.webp](https://prod-files-secure.s3.us-west-2.amazonaws.com/76eb865f-1e9a-440c-9d79-e386fea4df4f/05a1da27-c4ca-45b8-839d-d366abd1b830/2.webp)

### WEB UI 접속

WEB UI에 접속하려면 service에서 argocd-server를 포트포워딩 하거나 NodePort 또는 LoadBalancer변경해야 합니다. 포트포워딩 명령어는 아래와 같습니다.

```bash
kubectl port-forward svc/argocd-server -n  argocd --address=0.0.0.0 30001:443
```

### 초기 패스워드

```bash
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d; echo
```

### ArgoCD CLI 설치

```bash
# argocd cli 설치 (Linux)
curl -LO https://github.com/argoproj/argo-cd/releases/download/[VERSION]/argocd-linux-amd64
chmod u+x argocd-linux-amd64

mv argocd-linux-amd64 /usr/local/bin/argocd
# argocd cli 설치 (windows)
https://argo-cd.readthedocs.io/en/stable/cli_installation/#windows 참고

```