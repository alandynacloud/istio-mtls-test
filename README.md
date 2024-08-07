# istio-mtls-test

建立測試環境

該環境會建置以下項目

vpc:k8s-vpc

gke-private-cluster:demo-gke(開啟service mesh)

cloud nat:demo-nat
![istio-mtls-test drawio](https://github.com/user-attachments/assets/76ea4156-d206-4ae1-b628-3b5d4c345eb8)

```
terraform apply
```


下載測試範例

```
git clone https://github.com/alandynacloud/istio-mtls-test.git

cd istio-mtls-test
```


建立nginx應用程式image
該nginx作為proxy，將流量轉至三方網站，該網站(ifconfig.me)確認來源ip

# 建立imag
```
docker build -t $IMAGE_NAME .
```

隨後請將該IMAGE推送到GKE可以Access的repo
這邊image目前推送到google artifact registry 
參考文件:https://cloud.google.com/artifact-registry/docs/docker/pushing-and-pulling

範例
```
docker build -t asia-east1-docker.pkg.dev/dyncloud-lab-20221201/alan-istio-test/proxy-nginx:1.4 .
```

推送 Image
```
docker push
```

# 替換文件變數

```
export DOMAIN_NAME="YOUR_DOMAIN_NAME"
export IMAGE_NAME="YOUR_IMAGE_NAME"
```
透過envsubst替換tmpl檔裡面的變數$DOMAIN_NAME，並將tmpl檔轉為yaml檔
```
for template in $(find ./ -name '*.tmpl')
  do
    envsubst < ${template} > ${template%.*}
  done
```

# 佈署istio ingress gateway
參考文件:https://cloud.google.com/service-mesh/docs/operate-and-maintain/gateways

由於開啟mtls後需要透過istio-ingress-gateway才能access到應用程式，所以這邊我們需要建立 istio ingress gateway。
在目前下載的目錄istio-mtls-test/istio-ingressgateway底下，除了istio-ingress-gateway相關設定外還會建立ingress以及manage-cert的來串接google application loadbalancer與istio-ingress-gateway。如果不想使用ingress則可以下載Google範例檔[1]來進行佈署


[1]https://github.com/GoogleCloudPlatform/anthos-service-mesh-packages/tree/main/samples/gateways


為istio-ingress-gateway開啟sidecar auto injection
```
  kubectl label namespace istio-system \
      istio.io/rev- istio-injection=enabled --overwrite
```
![image](https://github.com/user-attachments/assets/f89aaffb-3704-4a9e-a76a-855ed85b819a)




佈署istio-ingress-gateway以及ingress與manage-cert
```
kubectl apply -n istio-system -f istio-ingressgateway/
```
![image](https://github.com/user-attachments/assets/d9e21eb7-ff7e-463d-a094-0732764601fe)



確認ManagedCertificate以及ingress
```
kubectl get ManagedCertificate,ingress -n istio-system
```
![image](https://github.com/user-attachments/assets/272b089a-97c2-4a7c-915e-a4757e02210c)



# 佈署應用程式

為nginx應用程式開啟sidecar auto injection
```
kubectl label namespace default \
      istio.io/rev- istio-injection=enabled --overwrite
```

佈署應用程式
```
kubectl apply -f nginx/nginx-app.yaml
kubectl apply -f nginx/nginx-gateway.yaml 
kubectl get pod,svc
```
![image](https://github.com/user-attachments/assets/63102fc7-9cc8-496a-8732-9f7a35548dd1)


svervice/nginx-service type目前使用LoadBalancer 以方便確認開啟mtls後測試


這時我們可以透過測試剛剛svervice/nginx-service type開的LoadBalancer IP
```
kubectl get svc
```
![image](https://github.com/user-attachments/assets/884f450e-adc6-4ea4-ba4b-f2f645d7f43b)






測試svervice/nginx-service type開的LoadBalancer IP也可以通過
```
curl $LOADBALANCER_IP
```

開啟mtls
```
kubeclt apply -f nginx/mtls-namespace.yaml
```
![image](https://github.com/user-attachments/assets/81126361-61f3-4ad3-aeb7-c981d5c7d858)


測試svervice/nginx-service type開的LoadBalancer IP，由於clinet端無法跟應用程式進行雙向加密所以無法連線，但如果走ingress則可以正常access
```
curl $LOADBALANCER_IP
```
![image](https://github.com/user-attachments/assets/5e3199a4-deed-472e-9548-02c6d7083d9f)



從叢集外browser測試
![image](https://github.com/user-attachments/assets/e8e86c4b-3edf-456b-bd64-d4cd853f1701)



可透過cloud shell 
```
curl $DOMAIN_NAME;echo
```

![image](https://github.com/user-attachments/assets/4064b503-c256-41a8-8028-1fad7c22b3d6)




