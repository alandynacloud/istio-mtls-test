# istio-mtls-test

建立測試環境
該環境會建置以下項目
vpc:k8s-vpc
gke-private-cluster:demo-gke
cloud nat:demo-nat
![istio-mtls-test drawio](https://github.com/user-attachments/assets/76ea4156-d206-4ae1-b628-3b5d4c345eb8)

```
terraform apply
```

下載測試範例


git clone https://github.com/alandynacloud/istio-mtls-test.git

cd istio-mtls-test



建立nginx應用程式image
該nginx作為proxy，將流量轉至三方網站，該網站(ifconfig.me)確認來源ip

建立imag
docker build -t $IMAGE_NAME .


隨後請將該IMAGE推送到GKE可以Access的repo
這邊image目前推送到google artifact registry 

Push and pull images | Artifact Registry documentation | Google Cloud

範例
docker build -t asia-east1-docker.pkg.dev/dyncloud-lab-20221201/alan-istio-test/proxy-nginx:1.4 .

推送 Image
docker push


替換文件變數


export DOMAIN_NAME="YOUR_DOMAIN_NAME"
export IMAGE_NAME="YOUR_IMAGE_NAME"

透過envsubst替換tmpl檔裡面的變數$DOMAIN_NAME，並將tmpl檔轉為yaml檔
for template in $(find ./ -name '*.tmpl')
  do
    envsubst < ${template} > ${template%.*}
  done


佈署istio ingress gateway
參考文件:Installing and upgrading gateways with Istio APIs | Cloud Service Mesh

由於開啟mtls後需要透過istio-ingress-gateway才能access到應用程式，所以這邊我們需要建立 istio ingress gateway。
在目前下載的目錄istio-mtls-test/istio-ingressgateway底下，除了istio-ingress-gateway相關設定外還會建立ingress以及manage-cert的來串接google application loadbalancer與istio-ingress-gateway。如果不想使用ingress則可以下載Google範例檔[1]來進行佈署


[1]https://github.com/GoogleCloudPlatform/anthos-service-mesh-packages/tree/main/samples/gateways


為istio-ingress-gateway開啟sidecar auto injection
  kubectl label namespace istio-system \
      istio.io/rev- istio-injection=enabled --overwrite


佈署istio-ingress-gateway以及ingress與manage-cert
kubectl apply -n istio-system -f istio-ingressgateway/



確認ManagedCertificate以及ingress
kubectl get ManagedCertificate,ingress -n istio-system


佈署應用程式

為nginx應用程式開啟sidecar auto injection

kubectl label namespace default \
      istio.io/rev- istio-injection=enabled --overwrite

佈署應用程式

kubectl apply -f nginx/nginx-app.yaml
kubectl apply -f nginx/nginx-gateway.yaml 
kubectl get pod,svc
svervice/nginx-service type目前使用LoadBalancer 以方便確認開啟mtls後測試


這時我們可以透過測試剛剛svervice/nginx-service type開的LoadBalancer IP

kubectl get svc

測試svervice/nginx-service type開的LoadBalancer IP也可以通過
curl $LOADBALANCER_IP


開啟mtls
kubeclt apply -f nginx/mtls-namespace.yaml


測試svervice/nginx-service type開的LoadBalancer IP，由於clinet端無法跟應用程式進行雙向加密所以無法連線，但如果走ingress則可以正常access
curl $LOADBALANCER_IP


從叢集外測試
可透過cloud shell 
curl $DOMAIN_NAME;echo




