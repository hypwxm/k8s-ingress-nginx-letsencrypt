### k8s配置ingress-nginx并且使用letsencrypt安装https


1: 安装helm
 下载对应版本的helm（可能需要其他工具下载）
 ```
 wget https://storage.googleapis.com/kubernetes-helm/helm-v2.11.0-linux-amd64.tar.gz
 ```
 解压
 ```
 tar -zxvf helm-v2.11.0-linux-amd64.tar.gz
 ```
 解压后将文件夹里面的helm可执行文件放入```/usr/local/bin```目录下
 ```
 helm help
 ```
 add the Helm chart repository
 ```
 helm repo add rancher-stable https://releases.rancher.com/server-charts/stable

 ```

2: 安装tiller
 最简单的方式是执行```helm init```
 tiller镜像可能会拉取失败
 为了tiller顺利启动，先
 ```
 docker pull hypwxm/tiller:v2.11.0
 ```
 然后打tag
 ```
 docker tag hypwxm/tiller:v2.11.0 gcr.io/kubernetes-helm/tiller:v2.11.0
 ```
 在执行
 ```
 helm init
 ```

3: 安装cert-manager
 拉取构建镜像的github文件到/home目录
 ```
 cd /home
 ```
 ```
 git clone git@github.com:jetstack/cert-manager.git
 ```
 这里稍花点时间
 下载完后
 ```
 cd /home/cert-manager
 ```
 可以自己查看最新版，然后checkout到最新版，文档编写时最新版为v0.5.2
 ```
 git checkout v0.5.2
 ```
 ```
 cd /home/cert-manager/contrib/charts/cert-manager
 ```
 ```
 helm dependency update
 ```
 ```
 helm init
 ```
 ```
 cd /home/cert-manager
 ```
 
 ```
 helm install \
  --name cert-manager \
  --namespace kube-system \
  --set ingressShim.defaultIssuerName=letsencrypt-production \
  --set ingressShim.defaultIssuerKind=ClusterIssuer \
  contrib/charts/cert-manager
 ```
 查看日志
 ```
 kubectl logs deployment/cert-manager --namespace kube-system -f
 ```
 

 4: 创建ClusterIssuer的yaml启动文件
 在kube-system命名空间中
 ```
apiVersion: certmanager.k8s.io/v1alpha1
kind: ClusterIssuer
metadata:
  name: letsencrypt-production
spec:
  acme:
    # 用来获取https的证书的服务机构
    server: https://acme-v02.api.letsencrypt.org/directory
    email: 1825909531@qq.com
    privateKeySecretRef:
      name: letsencrypt-production
    http01: {}
 ```

5: 安装ingress-nginx（如果不需要https，这一步请单独安装，最新版的貌似去掉了）
预先下载defaulbackend
```
docker pull hypwxm/defaultbackend:1.4
```
打tag
```
docker tag hypwxm/defaultbackend:1.4 k8s.gcr.io/defaultbackend:1.4
```
通过helm安装（网络问题，估计会失败，可以使用“官方配置文件安装”）
```
helm install stable/nginx-ingress   --name ingress-nginx   --set rbac.create=true --namespace ingress-nginx
```
官方配置文件安装
```
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/master/deploy/mandatory.yaml
```
安装之后需要在nginx-ingress-controller的deployment的template的spec中加上```hostNetwork: true```，然后更新这个deployment


6: 创建ingress

启动ingress
```
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: zhh-ingress
  namespace: default
  annotations:  # 对单独ingress有效
    # 限制并发请求输
    nginx.ingress.kubernetes.io/limit-connections: "10"   
    # 限制一个ip的每秒请求输
    nginx.ingress.kubernetes.io/limit-rps: "20"  
    # client_max_body_size 
    nginx.ingress.kubernetes.io/proxy-body-size: "8m"  
    kubernetes.io/ingress.class: nginx
    certmanager.k8s.io/cluster-issuer: letsencrypt-production
    # 配置完证书后再打开
    #kubernetes.io/tls-acme: "true"
# data:  写在data里面的规则将会覆盖全局的默认配置
spec:
  rules:
  - host: zhh.msudn.com
    http:
      paths:
      - path: /
        backend:
          # 对应的Service中metadata的name
          serviceName: zhh-gateway
          # 对应Service中spec.ports.port
          servicePort: 3000
  tls:
  # 证书名字对应secret，随便取，用来存放https的证书
  - secretName: nginx-ingress-certs 
    hosts:
    - zhh.msudn.com


```


#### 注意
1: 重新安装cert-manager可能需要删除crd的内容，可以
```
kubectl get crd
```
```
kubectl delete crd xxx
```