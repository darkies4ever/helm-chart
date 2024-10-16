### yckim README devops guide

<aside>
✅ devops init setting guide

</aside>

### **Install k9s**

```
curl -LO https://github.com/derailed/k9s/releases/download/v0.32.5/k9s_Linux_amd64.tar.gz
tar -xzvf k9s_Linux_amd64.tar.gz
rm k9s_Linux_amd64.tar.gz, LICENSE README.md
sudo mv k9s /usr/local/bin

```

### **Install Helm**

```
curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3
chmod 700 get_helm.sh
./get_helm.sh
rm get_helm.sh

```

### istio install

```markdown
helm repo add istio https://istio-release.storage.googleapis.com/charts
helm repo update
```

```bash
# CRD install
helm upgrade --install istio-base istio/base -n istio-system --set defaultRevision=default --create-namespace

# daemon install
helm upgrade --install istiod istio/istiod -n istio-system --wait --create-namespace

# ingress install
helm upgrade --install istio-ingress istio/gateway -n istio-system --set labels.istio=ingressgateway --create-namespace
```

### Metal Load Balancer Install

```markdown
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.11.0/manifests/namespace.yaml
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.11.0/manifests/metallb.yaml
```

```bash
## metallb-config.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  namespace: metallb-system  
  name: config
data:
  config: |
    address-pools:
    - name: default
      protocol: layer2       
      addresses:
      - 34.64.55.185/32      
      - 34.64.203.245/32     
      - 34.22.109.91/32
```

### Domain Setting

```bash
sudo vi /etc/hosts

...
127.0.0.1       localhost argocd.localhost kiali.localhost prom.localhost grafana.localhost bookinfo.localhost kibana.localhost
...

192.168.49.2 bookinfo.yckim.co.kr argocd.yckim.co.kr
```

**domain settings modify**

```bash
minikube ip
--> 192.168.49.2

*local*
sudo vi /etc/hosts
192.168.49.2       localhost argocd.localhost kiali.localhost prom.localhost grafana.localhost bookinfo.localhost kibana.localhost

*ssh*
minikube ssh
sudo vi /etc/hosts
192.168.49.2       localhost argocd.localhost kiali.localhost prom.localhost grafana.localhost bookinfo.localhost kibana.localhost

```

### **Istio Sample**

```
kubectl create ns bookinfo
kubectl label namespace bookinfo istio-injection=enabled
kubectl apply -f https://raw.githubusercontent.com/istio/istio/release-1.23/samples/bookinfo/platform/kube/bookinfo.yaml -n bookinfo
```

### **Istio VirtualService and Gateway**

Create `istio-values.yaml`(helm)

```yaml
gatewayset:
  - name: bookinfo-gateway
    namespace: bookinfo
    hosts:
      - "bookinfo.localhost"

# gatewayservers:
      - port:
          number: 80
          name: http
          protocol: HTTP

# virtualservicehttp:
      service: productpage
      port:
        number: 9080
      match:
        - uri:
            exact: /productpage
        - uri:
            prefix: /static
        - uri:
            exact: /login
        - uri:
            exact: /logout
        - uri:
            prefix: /api/v1/products
```

apply로 할꺼면 이걸로

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: Gateway
metadata:
  name: bookinfo-gateway
  namespace: bookinfo
spec:
  # The selector matches the ingress gateway pod labels.
  # If you installed Istio using Helm following the standard documentation, this would be "istio=ingress"
  selector:
    istio: ingressgateway # use istio default controller
  servers:
  - port:
      number: 80
      name: http
      protocol: HTTP
    hosts:
    - "bookinfo.yckim.co.kr"
---
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: bookinfo
  namespace: bookinfo
spec:
  hosts:
  - "bookinfo.yckim.co.kr"
  gateways:
  - bookinfo-gateway
  http:
  - match:
    - uri:
        exact: /productpage
    - uri:
        prefix: /static
    - uri:
        exact: /login
    - uri:
        exact: /logout
    - uri:
        prefix: /api/v1/products
    route:
    - destination:
        host: productpage
        port:
          number: 9080
```

api/v1/namespaces/kubernetes-dashboard/services/http:kubernetes-dashboard:/proxy/

### **helm chart**

```
helm repo add ggorockee https://ggorockee.github.io/helm-repository/chart
helm repo update

```

```
helm upgrade --install istio-gateway-stack ggorockee/istio-gateway-stack --namespace istio-system --create-namespace -f istio-values.yaml

```

# **Install argocd**

argocd의 경우 namespace에 `istio-injection: true`가 설정되어 있으면 올라가지 않는 문제가 있기 때문에 최초에는 설정을 하지 않음

```
helm repo add argocd https://argoproj.github.io/argo-helm
helm repo update
helm upgrade --install argocd \
               --namespace argocd \
               --create-namespace \
               --version 7.5.2 \
               --set global.domain="argocd.localhost" \
               --set configs.params.server\\.insecure=true \
               argocd/argo-cd
```

`istio-values.yaml` 파일 수정(helm)

```yaml
gatewayset:
  - name: bookinfo-gateway
    namespace: bookinfo
    hosts:
      - "bookinfo.localhost"

# gatewayservers:
      - port:
          number: 80
          name: http
          protocol: HTTP

# virtualservicehttp:
      service: productpage
      port:
        number: 9080
      match:
        - uri:
            exact: /productpage
        - uri:
            prefix: /static
        - uri:
            exact: /login
        - uri:
            exact: /logout
        - uri:
            prefix: /api/v1/products

# argocd- name: argocd
    namespace: argocd
    hosts:
      - "argocd.localhost"

# gatewayservers:
      - port:
          number: 80
          name: http
          protocol: HTTP

# virtualservicehttp:
      service: argocd-server
      port:
        number: 80
      match:
        - uri:
            prefix: /

```

**gateway / virtualservice(yaml) - argocd**

```bash
apiVersion: networking.istio.io/v1alpha3
kind: Gateway
metadata:
  name: argocd
  namespace: argocd
spec:
  # The selector matches the ingress gateway pod labels.
  # If you installed Istio using Helm following the standard documentation, this would be "istio=ingress"
  selector:
    istio: ingressgateway # use istio default controller
  servers:
  - port:
      number: 80
      name: http
      protocol: HTTP
    hosts:
    - "argocd.yckim.co.kr"
---
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: argocd-server
  namespace: argocd
spec:
  hosts:
  - "argocd.yckim.co.kr"
  gateways:
  - argocd
  http:
  - match:
    - uri:
        prefix: /
    route:
    - destination:
        host: argocd-server
        port:
          number: 80

```

이후,

```
helm upgrade --install istio-gateway-stack ggorockee/istio-gateway-stack --namespace istio-system --create-namespace -f istio-values.yaml

# istio-injection setting
kubectl label namespace argocd istio-injection=enabled

# restart argocd server
kubectl rollout restart deployments -n argocd
kubectl rollout restart statefulsets -n argocd

```

# **install prometheus**

```
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update

helm upgrade --install promethes-operator  \
             --set prometheus.prometheusSpec.serviceMonitorSelectorNilUsesHelmValues=false \
       	     --version 62.7.0 \
       	     --namespace=monitoring \
       	     --create-namespace \
       	     prometheus-community/kube-prometheus-stack

####       	     
helm upgrade --install promethes-operator  \
             --set prometheus.prometheusSpec.serviceMonitorSelectorNilUsesHelmValues=false \
             --set prometheus.prometheusSpec.podMonitorSelectorNilUsesHelmValues=false \
             --version 62.7.0 \
             --namespace=monitoring \
             --create-namespace \
             prometheus-community/kube-prometheus-stack

```

add virtual service & gateway `istio-values.yaml`

```yaml
gatewayset:
  ...
# prometheus- name: prometheus-operated
    namespace: monitoring
    hosts:
      - "prom.localhost"

# gatewayservers:
      - port:
          number: 80
          name: http
          protocol: HTTP

# virtualservicehttp:
      service: prometheus-operated
      port:
        number: 9090
      match:
        - uri:
            prefix: /

# grafana- name: promethes-operator-grafana
    namespace: monitoring
    hosts:
      - "grafana.localhost"

# gatewayservers:
      - port:
          number: 80
          name: http
          protocol: HTTP

# virtualservicehttp:
      service: promethes-operator-grafana
      port:
        number: 80
      match:
        - uri:
            prefix: /

```

upgrade istio-gateway-stack

```
$ helm upgrade --install istio-gateway-stack ggorockee/istio-gateway-stack --namespace istio-system --create-namespace -f istio-values.yaml

```

# **Install kiali**

```
helm upgrade --install \
    --set cr.create=true \
    --set cr.namespace=istio-system \
    --set cr.spec.auth.strategy="anonymous" \
    --namespace kiali-operator \
    --create-namespace \
    kiali-operator \
    kiali/kiali-operator

```

```yaml
gatewayset:
  ...
# kiali- name: kiali
    namespace: istio-system
    hosts:
      - "kiali.localhost"

# gatewayservers:
      - port:
          number: 80
          name: http
          protocol: HTTP

# virtualservicehttp:
      service: kiali
      port:
        number: 20001
      match:
        - uri:
            prefix: /

```

## **how to connect kiali with prometheus previous installed**

create `kube-monitor.yaml`

```yaml
# Please edit the object below. Lines beginning with a '#' will be ignored,# and an empty file will abort the edit. If an error occurs while saving this file will be# reopened with the relevant failures.#apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  annotations:
    meta.helm.sh/release-name: promethes-operator
    meta.helm.sh/release-namespace: monitoring
  labels:
    app.kubernetes.io/instance: promethes-operator
    app: istiod
  name: promethes-operator-istiod
  namespace: monitoring
spec:
  endpoints:
  - path: /metrics
    port: http-monitoring
  namespaceSelector:
    any: true
  selector:
    matchLabels:
      app: istiod
---
apiVersion: monitoring.coreos.com/v1
kind: PodMonitor
metadata:
  name: envoy-stats-monitor
  namespace: monitoring
  labels:
    meta.helm.sh/release-name: promethes-operator
spec:
  selector:
    matchExpressions:
    - {key: istio-prometheus-ignore, operator: DoesNotExist}
  namespaceSelector:
    any: true
  jobLabel: envoy-stats
  podMetricsEndpoints:
  - path: /stats/prometheus
    interval: 15s
    relabelings:
    - action: keep
      sourceLabels: [__meta_kubernetes_pod_container_name]
      regex: "istio-proxy"
    - action: keep
      sourceLabels: [__meta_kubernetes_pod_annotationpresent_prometheus_io_scrape]
    - action: replace
      regex: (\d+);(([A-Fa-f0-9]{1,4}::?){1,7}[A-Fa-f0-9]{1,4})
      replacement: '[$2]:$1'
      sourceLabels:
      - __meta_kubernetes_pod_annotation_prometheus_io_port
      - __meta_kubernetes_pod_ip
      targetLabel: __address__
    - action: replace
      regex: (\d+);((([0-9]+?)(\.|$)){4})
      replacement: $2:$1
      sourceLabels:
      - __meta_kubernetes_pod_annotation_prometheus_io_port
      - __meta_kubernetes_pod_ip
      targetLabel: __address__
    - action: labeldrop
      regex: "__meta_kubernetes_pod_label_(.+)"
    - sourceLabels: [__meta_kubernetes_namespace]
      action: replace
      targetLabel: namespace
    - sourceLabels: [__meta_kubernetes_pod_name]
      action: replace
      targetLabel: pod_name

```

check prom.localhost > status > targets

> podMonitor/monitoring/envoy-stats-monitor/0 (14/14 up)serviceMonitor/monitoring/promethes-operator-istiod/0 (1/1 up)
> 

prometheus와 grafana service이름 확인

```
kubectl get svc -n monitoring
# NAME                                          TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)                      AGE# alertmanager-operated                         ClusterIP   None             <none>        9093/TCP,9094/TCP,9094/UDP   22h# promethes-operator-grafana                    ClusterIP   10.100.219.172   <none>        80/TCP                       22h# promethes-operator-kube-pr-alertmanager       ClusterIP   10.111.81.218    <none>        9093/TCP,8080/TCP            22h# promethes-operator-kube-pr-operator           ClusterIP   10.96.143.111    <none>        443/TCP                      22h# promethes-operator-kube-pr-prometheus         ClusterIP   10.106.17.71     <none>        9090/TCP,8080/TCP            22h# promethes-operator-kube-state-metrics         ClusterIP   10.107.219.2     <none>        8080/TCP                     22h# promethes-operator-prometheus-node-exporter   ClusterIP   10.106.109.227   <none>        9100/TCP                     22h# prometheus-operated                           ClusterIP   None             <none>        9090/TCP                     22h
```

sevice에 해당하는 FQDN은 아래와 같음

```
# [service name].[namespace].svc.cluster.local
promethes-operator-grafana -> promethes-operator-grafana.monitoring.svc.cluster.local:80
prometheus-operated        -> prometheus-operated.monitoring.svc.cluster.local:9090

```

이제 프로메테우스와 kiali를 연결할 준비가 되었으니 kiali의 configmap을 수정한다.

```
$ kubectl get cm kiali -n istio-system -o yaml > kiali-configmap.yaml
$ vi kiali-configmap.yaml

# configmap 내용 수정
...

deployment:
  external_services:
    grafana:
      in_cluster_url: http://promethes-operator-grafana.monitoring.svc.cluster.local
      url: http://promethes-operator-grafana.monitoring.svc.cluster.local

    prometheus:
      url: http://prometheus-operated.monitoring.svc.cluster.local:9090
...

$ kubectl delete -f kiali-configmap.yaml
$ kubectl apply -f kiali-configmap.yaml
$ kubectl rollout restart deployments kiali -n istio-system
```