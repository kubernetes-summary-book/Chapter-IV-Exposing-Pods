# Chapter-IV-Exposing-Pods

# Servicios

![](./assets/service.png)

un servicio se describe como una forma abstarcta de exponer una aplicación ejecutándose en un Pod. Es decir, es una capa de indirección entre un Pod y otra entidad.

- **K8s tiene dos redes: Una para los Pods y otra para los servicios.**
- Ips del los servicios no contest a `ping` => No enruta => Son Ips virtuales (privadas).
- IPs de los servicios **NO cambia una vez creada**
- IP de Pod es efimeras.  
- Los servicios **TIENEN UNA VIDA MÁS LARGA que un Pod**.
- Un servicio == Actúa como un Loadbalancer

=> Los endpoints no son más que un **recurso de K8s que contiene uno o más direcciones IPsy puertos**.

Cuando creas un servicio en Kubernetes, el servicio utiliza un selector para identificar los pods que deberían recibir el tráfico. Kubernetes automáticamente crea y gestiona un objeto de tipo Endpoints que contiene la lista de IPs y puertos de los pods seleccionados.

Estos endpoints son utilizados internamente por Kubernetes para enrutar el tráfico desde el servicio a los pods correspondientes. Esto permite la abstracción y hace que los pods puedan ser reemplazados, actualizados o escalados sin que los consumidores del servicio necesiten conocer los detalles específicos de los pods.

`Los clientes que utilizan el servicio solo necesitan conocer el nombre del servicio y su puerto`, **no las direcciones IP específicas de los pods** que están detrás del servicio. Kubernetes gestiona dinámicamente la relación entre el servicio y los pods, actualizando los endpoints conforme los pods cambian, sin que esto sea visible o relevante para los consumidores del servicio.

## ClusterIp

![](./assets/clusterIP.png)

ClusterIP expone el servicio de forma interna, es decir, **solo es accesible desde dentro del clúster.
`Ventajas o diferenica es que los servicios son permanentes y sus IPs no cambian`

```bash
kubectl run myapp --image tuxotron/demoapp:v1 --labels app=myapp,ver=1
```

**Ver los labels de un Pod:**
```bash
kubectl get pods --show-labels
````


**Ver los servicios: **
```bash
kubectl get services
````


```bash
youneskabiri@Youness-MacBook-Pro Chapter-IV-Exposing-Pods % kubectl get pods --show-labels
NAME                                READY   STATUS             RESTARTS        AGE   LABELS
myapp                               0/1     CrashLoopBackOff   8 (3m45s ago)   19m   app=myapp,ver=1
```

Este comando crear un Pod con la imagen `tuxotron/demoapp:v1`y con dos labels `app=myapp,ver=1`.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-service
spec:
  type: ClusterIP
  selector:
    app: myapp
    ver: "1"
  ports:
    - port: 80
      targetPort: 8080
```

K8s al recibir la petición, el controlador de servicios buscará el espacio de nombres donde se despliegue el servicio los Pods que contengan esas dos etiquetas (app=myapp,ver=1). Y por cada Pod que encuentra crea un `endpoint` con IP del Pod y el puerto especifico en `targetPort`, y lo añade al servicio.  Y por último, define el puerto (80) que recibe las peticiones.

```bash
kubectl apply -f files/service-clusterIP.yaml 
```

```bash
kubectl describe service my-service
```

***Ver los endpoints :***
```bash
kubectl get endpoints
kubectl describe endpoints my-service
***

youneskabiri@Youness-MacBook-Pro Chapter-IV-Exposing-Pods % kubectl apply -f files/service-clusterIP.yaml 
    service/my-service created
youneskabiri@Youness-MacBook-Pro Chapter-IV-Exposing-Pods % kubectl describe service my-service
    Name:              my-service
    Namespace:         default
    Labels:            <none>
    Annotations:       <none>
    Selector:          app=myapp,ver=1
    Type:              ClusterIP
    IP Family Policy:  SingleStack
    IP Families:       IPv4
    IP:                10.110.128.211
    IPs:               10.110.128.211
    Port:              <unset>  80/TCP
    TargetPort:        8080/TCP
    Endpoints:         
    Session Affinity:  None
    Events:            <none>
```

**Al crear diferentes Pods con las misma labels, se añaden automaticamnete al mismo servicio que hará de LoadBalancer.**

## NodePort

![](./assets/nodeport.png)

NodePort expone en un puerto estático de cada nodo del clúster. Este es una extensión del tipo ClusterIP, es decir, `es un servicio tipo ClusterIP con el mapeo del puerto del servicio a un puerto estático en todos los nodos del clúster`

**Lo interesante, es que se puede hacer una petición a cualquier de los nodos en el clúster, sin importar si el Pod al que apunta se encuentra ejecutándose en el dicho nodo.**

Para ver un ejemplo, debemos crear un clúster con `minikube con multiples nodos`:
```bash
minikube delete
minikube start --driver=vmware --cni=cilium --memory=4gb --nodes=2
```

**Si estamos en macOs:**
```bsh
minikube start --cni=cilium --memory=4gb --nodes=2
```

**Con `--cni=cilium` añadimos un Container Network Interface** para poder comunicar diferentes nodos.

**Ver los nodos del clúster:**
```bash
kubectl get nodes -o wide
```

```bash
kubectl run myapp-np --image nginx:latest --labels app=myapp,ver=nodeport
```
Ahora vamos a crear un servicio `nodeport-servie`

```bash
kubectl apply -f files/service-nodePort.yaml
kubectl describe nodeport-service
```
```bash
youneskabiri@Youness-MacBook-Pro Chapter-IV-Exposing-Pods % kubectl apply -f files/ service-nodePort.yaml

  service/nodeport-service created

youneskabiri@Youness-MacBook-Pro Chapter-IV-Exposing-Pods % kubectl get services             
  NAME               TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)        AGE
  kubernetes         ClusterIP   10.96.0.1       <none>        443/TCP        14m
  nodeport-service   NodePort    10.101.73.255   <none>        80:30852/TCP   20s

youneskabiri@Youness-MacBook-Pro Chapter-IV-Exposing-Pods % kubectl describe service nodeport-service
  Name:                     nodeport-service
  Namespace:                default
  Labels:                   <none>
  Annotations:              <none>
  Selector:                 app=myapp,ver=nodeport
  Type:                     NodePort
  IP Family Policy:         SingleStack
  IP Families:              IPv4
  IP:                       10.101.73.255
  IPs:                      10.101.73.255
  Port:                     <unset>  80/TCP
  TargetPort:               8080/TCP
  NodePort:                 <unset>  30852/TCP<--------
  Endpoints:                10.244.1.98:8080<-----------
  Session Affinity:         None
  External Traffic Policy:  Cluster
  Events:                   <none>
```
```bash
youneskabiri@Youness-MacBook-Pro ~ % kubectl get nodes
  NAME           STATUS   ROLES           AGE   VERSION
  minikube       Ready    control-plane   22m   v1.28.3
  minikube-m02   Ready    <none>          22m   v1.28.3
youneskabiri@Youness-MacBook-Pro ~ % kubectl get nodes -o wide
  NAME           STATUS   ROLES           AGE   VERSION   INTERNAL-IP    EXTERNAL-IP   OS-IMAGE             KERNEL-VERSION    CONTAINER-RUNTIME
  minikube       Ready    control-plane   22m   v1.28.3   192.168.49.2   <none>        Ubuntu 22.04.3 LTS   6.6.22-linuxkit   docker://24.0.7
  minikube-m02   Ready    <none>          22m   v1.28.3   192.168.49.3   <none>        Ubuntu 22.04.3 LTS   6.6.22-linuxkit   docker://24.0.7
youneskabiri@Youness-MacBook-Pro ~ % kubectl describe service nodeport-service
  Name:                     nodeport-service
  Namespace:                default
  Labels:                   <none>
  Annotations:              <none>
  Selector:                 app=myapp,ver=nodeport
  Type:                     NodePort
  IP Family Policy:         SingleStack
  IP Families:              IPv4
  IP:                       10.101.73.255
  IPs:                      10.101.73.255
  Port:                     <unset>  80/TCP
  TargetPort:               8080/TCP
  NodePort:                 <unset>  30852/TCP
  Endpoints:                10.244.1.98:8080
  Session Affinity:         None
  External Traffic Policy:  Cluster
  Events:                   <none>
youneskabiri@Youness-MacBook-Pro ~ % curl 192.168.49.3:30852
```














## LoadBalancer

![](./assets/loadbalancer.png)

## Servicios sin selector de etiquetas



## ExternalName


## Service Descovery
## Headless

# Ingress




## Authors

- [@Younes Kabiri Farah](https://github.com/younesKabiriFarah)


## Acknowledgements

 - [Kubernetes](https://kubernetes.io/docs/home/)
 - [Book: Kubernetes para profesionales: Desde cero al despliegue de aplicaciones seguras y resilientes](https://0xword.com/es/libros/213-kubernetes-para-profesionales-desde-cero-al-despliegue-de-aplicaciones-seguras-y-resilientes.html)


## Features

- Workloads
- Configuración de aplicaciones y secretos
- Selección de nodos
- Volúmenes Persistentes
- Autorización Basada en Roles
- Politicas de Red
- Contexto y Políticas de Seguridad
