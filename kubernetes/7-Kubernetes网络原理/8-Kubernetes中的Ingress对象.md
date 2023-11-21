## **知识点**
- Ingress实际上就是Kubernetes对反向代理的抽象
- Ingress只能工作在七层，Service只能工作在四层。所以在Kubernetes中为应用进行TLS配置等HTTP相关操作，必须通过Ingress进行。

## **Ingress**
全局的、为了代理不同后端Service而设置的负载均衡服务，就是Kubernetes中的Ingress对象。Ingress就是Service的Service。

假如有一个站点：http://cafe.example.com，其中http://cafe.example.com/coffee对应咖啡点餐系统，http://cafe.example.com/tea对应茶水点餐系统。这两个系统分别由名为coffee和tea的两个Deployment来提供服务。

```yaml
apiVersion: extensions/v1
kind: Ingress
metadata:
  name: cafe-ingress
spec:
  tls:
  - hosts:
    - cafe.example.com
    secretName: cafe-secret
  rules:
  - host: cafe.example.com
    http:
      paths:
      - path: /tea
        backend:
          serviceName: tea-svc
          servicePort: 80
      - path: /coffee
        backend:
          serviceName: coffee-svc
          servicePort: 80
```
Ingress对象的每一个path对应一个后端Service。

一个Ingress对象的主要内容，就是一个反向代理服务（例如nginx）的配置文件的描述。这个代理服务对应的转发规则，就是IngressRule。

假设使用Nginx Ingress Controller，用户创建一个新的Ingress对象后，nginx-ingress-controller会根据Ingress对象的定义生成一份Nginx配置文件，并使用该配置文件启动一个Nginx服务。

为了让用户能够使用这个Nginx，需要创建一个Service对外暴露Nginx Ingress Controller管理的Nginx服务。使用NodePort类型的Service或者LoadBalancer（公有云）。

```yaml
apiVersion: v1
kind: Service
metadata:
  name: ingress-nginx
  namespace: ingress-nginx
  labels:
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/part-of: ingress-nginx
spec:
  type: NodePort
  ports:
    - name: http
      port: 80
      targetPort: 80
      protocol: TCP
    - name: https
      port: 443
      targetPort: 443
      protocol: TCP
  selector:
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/part-of: ingress-nginx
```

这个Service做的工作就是将所有携带ingress-nginx标签的Pod的80和443端口向外暴露。



