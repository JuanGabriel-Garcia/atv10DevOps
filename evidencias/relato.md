# Relato do grupo

Integrantes: Juan Gabriel Garcia e Gustavo Muller
Data: 16/10/2026
Distribuição Linux:
Python / Docker / Minikube:
Python 3.13.5
Docker version 29.2.1
minikube version: v1.39.0

## Etapas

| Etapa  | Comando e evidência | Resultado esperado | Resultado observado e explicação |
| ------ | ------------------- | ------------------ | -------------------------------- |
| Testes |                     | 8 testes OK        |                                  |

PS C:\Users\Juan\aulas\devops_kubernetes> python -m unittest discover -s tests -v
test_broken_alive (test_app.AppTests.test_broken_alive) ... pod=p version=quebrada "GET /healthz HTTP/1.1" 200 -
ok
test_broken_api (test_app.AppTests.test_broken_api) ... pod=p version=quebrada "GET /api/status HTTP/1.1" 503 -
ok
test_broken_readiness (test_app.AppTests.test_broken_readiness) ... pod=p version=quebrada "GET /readyz HTTP/1.1" 503 -
ok
test_html_escaping (test_app.AppTests.test_html_escaping) ... pod=<pod> version=v1 "GET / HTTP/1.1" 200 -
ok
test_readiness_healthy (test_app.AppTests.test_readiness_healthy) ... pod=p version=v1 "GET /readyz HTTP/1.1" 200 -
ok
test_status_v1 (test_app.AppTests.test_status_v1) ... pod=pod-1 version=v1 "GET /api/status HTTP/1.1" 200 -
ok
test_status_v2 (test_app.AppTests.test_status_v2) ... pod=pod-2 version=v2 "GET /api/status HTTP/1.1" 200 -
ok
test_unknown (test_app.AppTests.test_unknown) ... pod=p version=v1 "GET /inexistente HTTP/1.1" 404 -
ok

---

Ran 8 tests in 0.364s

OK

| Publicação v1 | | 2 Pods prontos; API v1 | |

2 pods prontos:
PS C:\Users\Juan\aulas\devops_kubernetes> .\scripts\kube.ps1 get pods -l app=entrega -o wide
NAME READY STATUS RESTARTS AGE IP NODE NOMINATED NODE READINESS GATES
entrega-5c8cd478b5-2w6kf 1/1 Running 0 10s 10.244.0.4 devops-k8s <none> <none>
entrega-5c8cd478b5-l58t4 1/1 Running 0 10s 10.244.0.3 devops-k8s <none> <none>

Construir a imagem com docker build não dispensa o minikube image load porque o Docker e o Minikube podem utilizar ambientes de imagens diferentes.

Publicação da v1:
PS C:\Users\Juan\aulas\devops_kubernetes> curl.exe -sS http://127.0.0.1:8080/api/status
{"version": "v1", "message": "Entrega v1 publicada pelo grupo.", "pod": "entrega-5c8cd478b5-2w6kf", "ready": true}
HTTP/1.0 200 OK
Server: BaseHTTP/0.6 Python/3.12.15
Date: Fri, 16 Oct 2026 18:32:57 GMT
Content-Type: application/json; charset=utf-8
Content-Length: 15
Cache-Control: no-store

{"ready": true}
PS C:\Users\Juan\aulas\devops_kubernetes> python scripts/smoke.py http://127.0.0.1:8080 --version v1 --output evidencias/v1-status.json
{"version": "v1", "message": "Entrega v1 publicada pelo grupo.", "pod": "entrega-5c8cd478b5-2w6kf", "ready": true}
SMOKE OK

[Evidência da publicação v1](v1-status.json)

| Escala | | 3 Pods prontos | |

3 Pods prontos:
PS C:\Users\Juan\aulas\devops_kubernetes> .\scripts\kube.ps1 get pods -l app=entrega
NAME READY STATUS RESTARTS AGE
entrega-5c8cd478b5-2w6kf 1/1 Running 0 20m
entrega-5c8cd478b5-96hz4 1/1 Running 0 13s
entrega-5c8cd478b5-l58t4 1/1 Running 0 20m

| Exclusão e recuperação | | Nome antigo removido; novo Pod criado | |

Exclusão:
PS C:\Users\Juan\aulas\devops_kubernetes> $lista = .\scripts\kube.ps1 get pods -l app=entrega -o json | ConvertFrom-Json
PS C:\Users\Juan\aulas\devops_kubernetes> $podAntigo = $lista.items[0].metadata.name
PS C:\Users\Juan\aulas\devops_kubernetes> $podAntigo | Set-Content evidencias/pod-excluido.txt -Encoding utf8
PS C:\Users\Juan\aulas\devops_kubernetes> .\scripts\kube.ps1 delete pod $podAntigo --wait=true
pod "entrega-5c8cd478b5-2w6kf" deleted from aula-k8s namespace

[Evidência do Pod excluído](pod-excluido.txt)

Recuperação:
PS C:\Users\Juan\aulas\devops_kubernetes> .\scripts\kube.ps1 get pods -l app=entrega -w  
NAME READY STATUS RESTARTS AGE
entrega-5c8cd478b5-2w6kf 1/1 Running 0 21m
entrega-5c8cd478b5-96hz4 1/1 Running 0 86s
entrega-5c8cd478b5-l58t4 1/1 Running 0 21m
entrega-5c8cd478b5-2w6kf 1/1 Terminating 0 22m
entrega-5c8cd478b5-2w6kf 1/1 Terminating 0 22m
entrega-5c8cd478b5-mqtns 0/1 Pending 0 0s
entrega-5c8cd478b5-mqtns 0/1 ContainerCreating 0 0s
entrega-5c8cd478b5-2w6kf 0/1 Completed 0 22m
entrega-5c8cd478b5-mqtns 0/1 ContainerCreating 0 1s
entrega-5c8cd478b5-mqtns 0/1 Running 0 1s
entrega-5c8cd478b5-2w6kf 0/1 Completed 0 22m
entrega-5c8cd478b5-mqtns 0/1 Running 0 3s
entrega-5c8cd478b5-mqtns 0/1 Running 0 3s
entrega-5c8cd478b5-mqtns 1/1 Running 0 3s

O controlador criou um novo Pod, com outro nome e outro identificador, para manter as três réplicas definidas no Deployment. Isso acontece porque os Pods são recursos descartáveis: o controlador busca manter o estado desejado, e não preservar o objeto excluído.
Entretanto, ter três Pods no mesmo nó não protege a aplicação contra a falha completa da máquina. Se o nó parar de funcionar, todos os Pods hospedados nele ficarão indisponíveis ao mesmo tempo. Para aumentar a disponibilidade, é necessário distribuir as réplicas entre diferentes nós, reduzindo o risco de uma única falha interromper toda a aplicação.

| ConfigMap | | Mensagem muda após recriar Pods | |

Mudança da mensagem sem alterar:
PS C:\Users\Juan\aulas\devops_kubernetes> .\scripts\kube.ps1 apply -f k8s/configmap.yaml
configmap/entrega-config configured
PS C:\Users\Juan\aulas\devops_kubernetes> curl.exe -sS http://127.0.0.1:8080/api/status
{"version": "v1", "message": "Entrega v1 publicada pelo grupo.", "pod": "entrega-5c8cd478b5-l58t4", "ready": true}

Alteração da mensagem:
PS C:\Users\Juan\aulas\devops_kubernetes> python scripts/smoke.py http://127.0.0.1:8080 --version v1 --message "Entrega configurada pelo grupo." --output evidencias/config-status.json
{"version": "v1", "message": "Entrega configurada pelo grupo.", "pod": "entrega-57647d75c5-cv67b", "ready": true}
SMOKE OK

[Evidência da configuração](config-status.json)

Não foi necessário construir outra imagem porque a alteração foi feita no ConfigMap do Kubernetes, que armazena configurações separadamente da imagem da aplicação. Ao atualizar o ConfigMap e reiniciar os Pods, a aplicação carregou a nova mensagem sem precisar reconstruir a imagem Docker.

| Atualização v2 | | 3 Pods v2 prontos | |

Atualização da v2:
PS C:\Users\Juan\aulas\devops_kubernetes> python scripts/smoke.py http://127.0.0.1:8080 --version v2 --message "Entrega configurada pelo grupo." --output evidencias/v2-status.json
{"version": "v2", "message": "Entrega configurada pelo grupo.", "pod": "entrega-fff5c98fd-jkd5w", "ready": true}
SMOKE OK

[Evidência da atualização v2](v2-status.json)

[revisao-ok.txt](../build/revisao-ok.txt)

| Incidente | | Novo Running 0/1; healthz 200, readyz 503 | |

Incidente:
PS C:\Users\Juan\aulas\devops_kubernetes> .\scripts\kube.ps1 rollout status deployment/entrega --timeout=30s
Waiting for deployment "entrega" rollout to finish: 1 out of 3 new replicas have been updated...
error: timed out waiting for the condition

PS C:\Users\Juan\aulas\devops_kubernetes> curl.exe -i http://127.0.0.1:8081/healthz
HTTP/1.0 200 OK
Server: BaseHTTP/0.6 Python/3.12.15
Date: Fri, 16 Oct 2026 22:57:13 GMT
Content-Type: application/json; charset=utf-8
Content-Length: 15
Cache-Control: no-store

{"alive": true}
PS C:\Users\Juan\aulas\devops_kubernetes> curl.exe -i http://127.0.0.1:8081/readyz
HTTP/1.0 503 Service Unavailable
Server: BaseHTTP/0.6 Python/3.12.15
Date: Fri, 16 Oct 2026 22:57:18 GMT
Content-Type: application/json; charset=utf-8
Content-Length: 16
Cache-Control: no-store

{"ready": false}

O sinal que justifica abandonar a entrega é a falha na verificação de prontidão (/readyz retornando HTTP 503), indicando que a aplicação está viva, mas não está pronta para atender requisições. Isso impede que o Pod receba tráfego normalmente e justifica reverter a implantação defeituosa.

[Evidência do diagnóstico do incidente](incidente-describe.txt)
[Evidência dos logs do incidente](incidente-logs.txt)

| Rollback | | v2 restaurada | |
PS C:\Users\Juan\aulas\devops_kubernetes> .\scripts\kube.ps1 rollout status deployment/entrega --timeout=120s
deployment "entrega" successfully rolled out
PS C:\Users\Juan\aulas\devops_kubernetes> .\scripts\kube.ps1 get pods -l app=entrega
NAME READY STATUS RESTARTS AGE
entrega-fff5c98fd-jkd5w 1/1 Running 0 56m
entrega-fff5c98fd-lpxth 1/1 Running 0 56m
entrega-fff5c98fd-xbntt 1/1 Running 0 56m
PS C:\Users\Juan\aulas\devops_kubernetes> $d = .\scripts\kube.ps1 get deployment entrega -o json | ConvertFrom-Json

> > $d.spec.template.spec.containers[0].image
> > entrega-local:v2

O rollback restaurou a revisão anterior do Deployment, recuperando o modelo dos Pods, incluindo a imagem e as configurações definidas nele. O ConfigMap não foi revertido, pois é um recurso independente, e os arquivos locais permaneceram inalterados. A quantidade de réplicas voltou ao valor definido na revisão restaurada.
O comando que iniciou a reversão foi kubectl rollout undo deployment/entrega --to-revision=$revOk. Já o timeout da etapa anterior apenas encerrou o tempo de espera, sem cancelar nem reverter a implantação.

| Pipeline v3 | | Smoke OK e estado final v3 | |

C:\Users\Juan\AppData\Local\Programs\Python\Python313\python.exe scripts/smoke.py http://127.0.0.1:18080 --version v3 --output evidencias/status-v3.json
{"version": "v3", "message": "Entrega configurada pelo grupo.", "pod": "entrega-59dff48b85-6vmd9", "ready": true}
SMOKE OK

- C:\Program Files\Kubernetes\Minikube\minikube.EXE -p devops-k8s kubectl -- --context=devops-k8s --namespace=aula-k8s get pods-l app=entrega -o wide
- C:\Program Files\Docker\Docker\resources\bin\docker.EXE image inspect entrega-local:v3
  Pipeline concluida: v3, replicas=3

## Respostas

1. Por que carregar a imagem no Minikube depois de construir no Docker?
   Porque a imagem construída no Docker local precisa estar disponível no ambiente do Minikube para que o Kubernetes consiga utilizá-la.
2. Qual a função das portas 8000, 80, 30080 e 8080?
   A porta 8000 é usada pela aplicação no contêiner; a 80, pelo Service; e a 8080, pelo acesso local via port-forward. A porta 30080 não é utilizada no fluxo descrito no texto.
3. Por que o Pod excluído foi substituído?
   Porque o Deployment mantém a quantidade desejada de réplicas, criando outro Pod para substituir o excluído. O novo Pod tem uma identidade diferente.
4. Por que três Pods no mesmo nó não protegem contra a perda da máquina?
   Porque, se o nó parar, os três Pods podem ficar indisponíveis ao mesmo tempo. Para aumentar a proteção, as réplicas devem estar distribuídas entre nós diferentes.
5. Por que a mensagem não mudou imediatamente após aplicar o ConfigMap?
   Porque a aplicação lê a mensagem pela variável de ambiente durante a inicialização. Foi necessário recriar os Pods para carregar o novo valor.
6. Tag de imagem e revisão do Deployment são a mesma coisa?
   Não. A tag identifica a versão da imagem, como v2; a revisão identifica uma versão do modelo de configuração dos Pods no Deployment.
7. Que sinais mostram que a versão quebrada está viva, mas não pronta?
   A rota /healthz retorna 200, enquanto /readyz retorna 503. O Pod aparece como Running, mas com READY 0/1.
8. O que o timeout fez? O que o rollback restaurou e o que não restaurou?
   O timeout apenas encerrou o tempo de espera do comando, sem desfazer a publicação. O rollback restaurou o modelo dos Pods da revisão escolhida, incluindo a imagem e as configurações nele definidas. Não reverteu o ConfigMap independente nem os arquivos locais.
9. Por que alinhar o arquivo gerado após a reversão?
   Porque o arquivo ainda descrevia a versão quebrada. Foi necessário gerá-lo novamente com v2 e três réplicas para evitar reaplicar a configuração defeituosa.
10. Quais verificações a pipeline realiza? Qual é seu limite?
    Executa os testes unitários, constrói e carrega a imagem, gera e aplica os manifestos, acompanha o rollout e realiza um teste HTTP. Seu limite é ser executada manualmente e localmente, sem integração contínua, além de não realizar rollback automático nem garantir que todos os problemas de produção sejam detectados.

## Divergências e correções

Descreva hipótese, comando de diagnóstico, sinal observado e correção.
