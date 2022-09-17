## Quality of Service for Pods
ref: https://kubernetes.io/docs/tasks/configure-pod-container/quality-service-pod/

- Guaranteed - limit e request são identicos (para todos os containers do pod)
- Burstable - Se ele não for Guaranteed e pelo menos um container do pod tiver definição de recursos (limit/request)
- BestEffort - Se o container não tiver definição de recursos (limit/request)

## Configure a Pod to Use a Volume for Storage
ref: https://kubernetes.io/docs/tasks/configure-pod-container/configure-volume-storage/


Volumes permitem um container (no pod) acessar dados. O mais básico é compartilhar dados entre containers no mesmo pod.

Definição do `spec.volumes` e `spec.containers.volumeMounts`
- `emptyDir` é um tipo de volume útil pra compartilhar dados no pod https://kubernetes.io/docs/concepts/storage/volumes/#emptydir
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: redis
spec:
  containers:
  - name: redis
    image: redis
    volumeMounts:
    - name: redis-storage
      mountPath: /data/redis
  volumes:
  - name: redis-storage
    emptyDir: {}
```

Em alguns casos, pode-se querer usar um `PersistentVolume`(PV) que é um volume persistente. Ele precisa de algumas coisas para funcionar

- `PersistentVolumeClaim` (PVC) - Esse é o cara que diz informações de tamanho, acesso e funcionamento dessa "pasta" e vai ser usado pelo pod
- `StorageClass` - Define informações sobre como funciona a criação do PV (exemplo: Cria um bucket S3 e limpa ele toda as vezes que for usado). O PVC faz uso dessa classe para dizer como/onde serão registrados esses dados
- `PersistentVolume` - É a pasta de fato, aqui por exemplo é definido o tamanho (quase como se fosse um HD)


O exemplo é: Imagine que o `PersistentVolume` é como um HD. O `StorageClass` é alguém que sabe adicionar HDs na sua maquina, e o `PersistentVolumeClaim` é usado para criar pastas nesses HDs e vincular ela nos nossos pods através do `spec.volumes`


StorageClass

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: local-storage
provisioner: kubernetes.io/no-provisioner
volumeBindingMode: WaitForFirstConsumer
```

PersistentVolume
```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: task-pv-volume
  labels:
    type: local
spec:
  storageClassName: local-storage
  capacity:
    storage: 10Gi
  accessModes:
    - ReadWriteOnce
  hostPath:
    path: "/mnt/data"
```
PersistentVolumeClaim
```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: task-pv-claim
spec:
  storageClassName: local-storage
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 3Gi
```


## Configure Service Accounts for Pods
ref https://kubernetes.io/docs/tasks/configure-pod-container/configure-service-account/

Quando nao especificado, o pod recebe por padrão a Service Account (SA) `default` no mesmo namespace.
A SA serve para permitir (ou não) iterações com as APIs do k8s.
Você pode definir a SA que o pod vai usar:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-pod
spec:
  serviceAccountName: build-robot
```


## Configure Liveness, Readiness and Startup Probes
ref https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/

O pod tem alguns recursos para verificar a saúde dele, conhecidos como [`probes`](https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/#types-of-probe)
- liveness - Verifica se o container ta vivo. Se falha, o k8s mata o pod e sobe outro
- readiness - Verifica se o container ta pronto pra receber request. **Enquanto** estiver falhando, o k8s não entrega trafego pro pod
- startup - Ele é uma verificação de start. Se ele existe, os outros dois não são executados ate o sucesso. Se ele falha, o pod é morto pra outro nascer

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: goproxy
  labels:
    app: goproxy
spec:
  containers:
  - name: goproxy
    image: registry.k8s.io/goproxy:0.1
    ports:
    - containerPort: 8080
    readinessProbe:
      tcpSocket:
        port: 8080
      initialDelaySeconds: 5
      periodSeconds: 10
    livenessProbe:
      tcpSocket:
        port: 8080
      initialDelaySeconds: 15
      periodSeconds: 20

```

## Assign Pods to Nodes
ref https://kubernetes.io/docs/tasks/configure-pod-container/assign-pods-nodes/
ref https://kubernetes.io/docs/concepts/scheduling-eviction/assign-pod-node/

É possível limitar um pod a um node com algumas tecnicas:


nodeSelector -> limita pela label presente no node
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
  labels:
    env: test
spec:
  containers:
  - name: nginx
    image: nginx
    imagePullPolicy: IfNotPresent
  nodeSelector:
    disktype: ssd
```


nodeName -> limita pelo nome do node
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
spec:
  nodeName: foo-node
  containers:
  - name: nginx
    image: nginx
    imagePullPolicy: IfNotPresent
```

## Assign Pods to Nodes using Node Affinity
ref https://kubernetes.io/docs/tasks/configure-pod-container/assign-pods-nodes-using-node-affinity/
ref https://kubernetes.io/docs/concepts/scheduling-eviction/assign-pod-node/

A afinidade é uma maneira mais avançada de selecionar onde seu pod pode ir parar, enquanto o nodeSelector é um caminho mais simples.
Ele traz uma flexibilidade:
- requiredDuringSchedulingIgnoredDuringExecution - Obriga a regras a serem seguidas para o agendamento do pod (igual o `nodeSelector` mas com uma sintaxe mais expressiva)
- preferredDuringSchedulingIgnoredDuringExecution - Ele tenta seguir a regra, mas se não conseguir ele segue com o agendamento do pod

Esse tipo de estrategia é útil quando vc quer ter maior controle de quão perto (ou longe) as coisas vão estar dentro do seu cluster.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: with-node-affinity
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: topology.kubernetes.io/zone
            operator: In
            values:
            - antarctica-east1
            - antarctica-west1
      preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 1
        preference:
          matchExpressions:
          - key: another-node-label-key
            operator: In
            values:
            - another-node-label-value
  containers:
  - name: with-node-affinity
    image: registry.k8s.io/pause:2.0
```

## Create a Pod that has an Init Container
ref https://kubernetes.io/docs/tasks/configure-pod-container/configure-pod-initialization/#create-a-pod-that-has-an-init-container

Em alguns cenários você pode querer fazer algo antes do seu container principal rodar (Exemplo: Registrar o novo serviço em um service discovery), ou até gerar arquivos de configuração para o pod principal

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: init-demo
spec:
  containers:
  - name: nginx
    image: nginx
    ports:
    - containerPort: 80
    volumeMounts:
    - name: workdir
      mountPath: /usr/share/nginx/html
  # These containers are run during pod initialization
  initContainers:
  - name: install
    image: busybox:1.28
    command:
    - wget
    - "-O"
    - "/work-dir/index.html"
    - http://info.cern.ch
    volumeMounts:
    - name: workdir
      mountPath: "/work-dir"
  dnsPolicy: Default
  volumes:
  - name: workdir
    emptyDir: {}
```
