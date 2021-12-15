# science
poc using istio+flagger

I choose the option to get the image from argoproj/rollouts-demo because it is interesting to check the requests because the app print the collor in the screen regarding weigth configured on istio. 
I tried used the RedHat's free CodeReady, but i could not install istio and Tekton, i never used before redhat  codeready, so i followed the way to deliver it in the generic environment.
I created 2 types the first one just istio and changing the weigth directly on Kiali another way(my favourite) using flagger(https://docs.flagger.app/tutorials/istio-progressive-delivery#traffic-mirroring).
I did the deploy using the kustomization. 


#I Could not installed istio in the by RedHat's free CodeReady Containers, i did it in my env GCP cluster and it works fine, just setup istio on cluster and run the kustomization.
#install Istio
curl -L https://istio.io/downloadIstio | sh -
cd istio-1.12.1

istioctl manifest generate —set profile=demo > istio.yaml

kubectl apply -f istio.yaml

OR

istioctl install --set profile=default -y --set meshConfig.accessLogFile=/dev/stdout

#to preserve the original ip form cliente
kubectl patch svc istio-ingressgateway -n istio-system -p '{"spec":{"externalTrafficPolicy":"Local"}}'

#check logs envoy/ingress
kubectl get pods -n istio-system -o name -l istio=ingressgateway | sed 's|pod/||' | while read -r pod; do kubectl logs "$pod" -n istio-system; done

#auto-inject Istio ns
kubectl label namespace default istio-injection=enabled
kubectl label namespace flagger istio-injection=enabled
#get ingress gw
kg svc -n istio-system

#Install integrations
#Prometheus
kubectl apply -f https://raw.githubusercontent.com/istio/istio/release-1.12.1/samples/addons/prometheus.yaml
#Grafana
kubectl apply -f https://raw.githubusercontent.com/istio/istio/release-1.12.1/samples/addons/grafana.yaml
#Kiali
kubectl apply -f https://raw.githubusercontent.com/istio/istio/release-1.12.1/samples/addons/kiali.yaml

#Deploy Flagger in the istio-system namespace with Slack notifications enabled:
helm upgrade -i flagger flagger/flagger \
--namespace=istio-system \
--set crd.create=false \
--set metricsServer=http://prometheus.istio-system:9090 \
#--set slack.url=https://hooks.slack.com/services/xxxxxxxxxxxxxxx \
#--set slack.channel=istio-canary-deploy \
#--set slack.user=istio-flagger

##OPTION INSTALL JUSTIO ISTIO E BG MANUAL USING KIALI
cd application
ka -k .
kubectl get pods,hpa,svc,dr,vs -n science
while true; do curl <url>; sleep 5 && echo "\n next"; done //Test connections
istioctl d kiali


##OPTION INTALL FLAGGER
#Install Canary with flagger
cd flagger
ka -k .
kubectl get pods,hpa,canary,svc,dr,vs -n flagger
kubectl describe canary/helloworld-flagger
while true; do curl https://flagger-staging.petlove.com.br/uuid; sleep 5 && echo "\n next"; done //Test connections
istioctl d kiali

#test deploy
kubectl describe canary/helloworld
watch kubectl get canaries --all-namespaces
kubectl logs deployment/flagger -f  -n istio-system | jq .msg


