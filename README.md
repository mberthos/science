# science
poc using istio+flagger+tekton

#install Istio
curl -L https://istio.io/downloadIstio | sh -
cd istio-1.12.1
istioctl manifest generate — set profile=demo > istio.yaml
kubectl apply -f istio.yaml
OR
istioctl install --set profile=default -y --set meshConfig.accessLogFile=/dev/stdout

#GCP GKE	TCP/UDP Network Load Balancer	Istio is Network LB
#to preserve the original ip form cliente
kubectl patch svc istio-ingressgateway -n istio-system -p '{"spec":{"externalTrafficPolicy":"Local"}}'
#check logs envoy/ingress
kubectl get pods -n istio-system -o name -l istio=ingressgateway | sed 's|pod/||' | while read -r pod; do kubectl logs "$pod" -n istio-system; done

#auto-inject Istio ns
kubectl label namespace default istio-injection=enabled
#get ingress gw
kg svc -n istio-system

#Install integrations
#Prometheus
kubectl apply -f https://raw.githubusercontent.com/istio/istio/release-1.12.1/samples/addons/prometheus.yaml
#Grafana
kubectl apply -f https://raw.githubusercontent.com/istio/istio/release-1.12.1/samples/addons/grafana.yaml
#Kiali
kubectl apply -f https://raw.githubusercontent.com/istio/istio/release-1.12.1/samples/addons/kiali.yaml

#HelloWorld
#Install application and loadtester
cd application
ka -k . 
ka -f gateway-istio.yaml
ka -f application.yaml
ka -f application_loadtester.yaml
ka -f grafana-flagger.yaml

#Install flagger
helm repo add flagger https://flagger.app
#Deploy Flagger in the istio-system namespace with Slack notifications enabled:
helm upgrade -i flagger flagger/flagger \
--namespace=istio-system \
--set crd.create=false \
--set metricsServer=http://prometheus.istio-system:9090 \
--set slack.url=https://hooks.slack.com/services/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 \
--set slack.channel=istio-canary-deploy \
--set slack.user=istio-flagger

#Install Canary with flagger
kubectl apply -f canary_deploy.yaml

kubectl get pods,hpa,canary,svc,dr,vs -n flagger

kubectl describe canary/helloworld
while true; do curl https://flagger-staging.petlove.com.br/uuid; sleep 5 && echo "\n next"; done //Test connections

#test deploy
kubectl set image deployment/helloworld helloworld=testcontainers/helloworld:1.1.0 | latest | 1.0.0

kubectl describe canary/helloworld
watch kubectl get canaries --all-namespaces
kubectl logs deployment/flagger -f  -n istio-system | jq .msg