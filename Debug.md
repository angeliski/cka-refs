
Para todo os recursos, `describe` é um bom ponto de partida, pois vai te dar muitas informações

## Debug Pods
refs https://kubernetes.io/docs/tasks/debug/debug-application/debug-pods/
refs https://kubernetes.io/docs/tasks/debug/debug-application/debug-running-pod/

O caminho mais simples pra entender o que esta acontencendo com seu pod é usando o `describe`:

`kubectl describe pods ${POD_NAME}`

Exemplo de output:
```
Name:         liveness-exec
Namespace:    default
Priority:     0
Node:         node01/172.30.2.2
Start Time:   Sat, 17 Sep 2022 21:21:44 +0000
Labels:       test=liveness
Annotations:  cni.projectcalico.org/containerID: 226fa050ae71b04eae18f47f7673f74026e236065378ae6c059070105621a6d0
              cni.projectcalico.org/podIP: 192.168.1.3/32
              cni.projectcalico.org/podIPs: 192.168.1.3/32
Status:       Running
IP:           192.168.1.3
IPs:
  IP:  192.168.1.3
Containers:
  liveness:
    Container ID:  containerd://408c25fbad16bc7350678ef178442ccb5aa39c78cde520e4e6b0fe727bc175d4
    Image:         registry.k8s.io/busybox
    Image ID:      sha256:36a4dca0fe6fb2a5133dc11a6c8907a97aea122613fa3e98be033959a0821a1f
    Port:          <none>
    Host Port:     <none>
    Args:
      /bin/sh
      -c
      touch /tmp/healthy; sleep 30; rm -f /tmp/healthy; sleep 600
    State:          Running
      Started:      Sat, 17 Sep 2022 21:21:45 +0000
    Ready:          True
    Restart Count:  0
    Liveness:       exec [cat /tmp/healthy] delay=5s timeout=1s period=5s #success=1 #failure=3
    Environment:    <none>
    Mounts:
      /var/run/secrets/kubernetes.io/serviceaccount from kube-api-access-xqsdh (ro)
Conditions:
  Type              Status
  Initialized       True
  Ready             True
  ContainersReady   True
  PodScheduled      True
Volumes:
  kube-api-access-xqsdh:
    Type:                    Projected (a volume that contains injected data from multiple sources)
    TokenExpirationSeconds:  3607
    ConfigMapName:           kube-root-ca.crt
    ConfigMapOptional:       <nil>
    DownwardAPI:             true
QoS Class:                   BestEffort
Node-Selectors:              <none>
Tolerations:                 node.kubernetes.io/not-ready:NoExecute op=Exists for 300s
                             node.kubernetes.io/unreachable:NoExecute op=Exists for 300s
Events:
  Type    Reason     Age   From               Message
  ----    ------     ----  ----               -------
  Normal  Scheduled  28s   default-scheduler  Successfully assigned default/liveness-exec to node01
  Normal  Pulling    27s   kubelet            Pulling image "registry.k8s.io/busybox"
  Normal  Pulled     27s   kubelet            Successfully pulled image "registry.k8s.io/busybox" in 565.845349ms
  Normal  Created    27s   kubelet            Created container liveness
  Normal  Started    27s   kubelet            Started container liveness
```

 ### Pod preso em pending
 Ou você não tem mais recursos no k8s, ou vc usou `hostPort` e seus nodes já estao usando isso.

 ### Pod preso em waiting
 Describe vai te ajudar aqui. É comum ser algum problema em dar pull na imagem.


 ### Meu pod não ta fazendo o que eu pedi
 Você pode ter digitado alguma chave do arquivo de maneira errada (command vs commnd), conferir a chave e usar o `--validate` pode te ajudar


 Você pode investigar os logs do seu pod tambem:
 `kubectl logs ${POD_NAME} ${CONTAINER_NAME}`

 Ou se ele estiver executando, pode acessar ele:
 `kubectl exec ${POD_NAME} -c ${CONTAINER_NAME} -- ${CMD} ${ARG1} ${ARG2} ... ${ARGN}`



 ## Debug Services
 ref https://kubernetes.io/docs/tasks/debug/debug-application/debug-pods/#debugging-services
 ref https://kubernetes.io/docs/tasks/debug/debug-application/debug-service/

Serviços criam um acesso aos seus pods, então uma primeira coisa que se pode verificar é o numero de endpoints que ele tem acesso:
`kubectl get endpoints ${SERVICE_NAME}`, isso deve retornar o mesmo numero de pods que seu serviço deveria estar servindo.

### Tem endpoints faltando
Verifique o seletor do seu serviço, se as labels estiverem diferentes, você não vai conseguir o resultado desejado


Um caminho para debug problemas no cluster é rodar um pod para realizar verificações dentro do cluster:
```shell
kubectl run -it --rm --restart=Never busybox --image=gcr.io/google-containers/busybox sh
```
e para criar um service de um deployment rapidamente: (`port` se refere a porta de acesso do service e `target-port` a porta alvo do container)
`kubectl expose deployment hostnames --port=80 --target-port=9376`


### Consigo acessar o Service pelo dns?

De dentro de um pod você pode tentar:
`nslookup svc-name` ou `nslookup svc-name.namespace` para ver como será resolvido esse acesso.

Em casos que isso não é acessível, podemos ver o arquivo de configuração:
`cat /etc/resolv.conf` .

Se nada disso estiver dando certo, o `nslookup kubernetes.default` deve sempre funcionar. Isso pode direcionar se é um problema local ou geral no servidor de DNS do seu cluster.

Se tudo estiver certo, convém confirmar que o Service funciona pela chamada do ip:
`wget -qO- 10.109.41.112:80` (o ip pode ser obtido através de `k get svc service-name`)

Se tudo estiver certo, você pode conferir algumas coisas:
- O service esta definido corretamente? (As portas podem estar erradas por exemplo)
- O service tem endpoints sendo fornecidos
- Os pods estão funcionando? (o service pode estar certo, mas o pod recusa a conexão)
- O kube-proxy ta funcionando? (`ps auxw | grep kube-proxy` e se tem erros no log `sudo journalctl -u kube-proxy`)
