# ☸️ Guia de Comandos Kubernetes

Anotações de instalação, configuração de contexto, gerenciamento de workloads, redes, armazenamento e operação do dia a dia de um cluster Kubernetes via `kubectl`.

## 📑 Índice

1. [Instalação e Configuração do kubectl](#instalacao-configuracao)
2. [Contextos e Namespaces](#contextos-namespaces)
3. [Gerenciamento de Pods](#gerenciamento-pods)
4. [Deployments e Rollouts](#deployments-rollouts)
5. [Services e Exposição de Rede](#services-rede)
6. [ConfigMaps e Secrets](#configmaps-secrets)
7. [Volumes e Armazenamento Persistente](#volumes-armazenamento)
8. [Monitoramento, Logs e Debug](#monitoramento-debug)
9. [Aplicando e Gerenciando Manifests](#manifests)
10. [Nodes e Manutenção do Cluster](#nodes-cluster)
11. [RBAC e Segurança](#rbac-seguranca)
12. [Helm (Gerenciador de Pacotes)](#helm)
---

## 1. <span id="instalacao-configuracao">⚙️ Instalação e Configuração do kubectl</span>

Preparação do ambiente local para se comunicar com o cluster.

### 🔹 Instalar o client kubectl
Baixa a versão estável mais recente do binário e o instala no PATH do sistema.
```bash
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl
```

### 🔹 Verificar a versão instalada
Mostra a versão do client (kubectl) e, se conectado, também a versão do server (control plane).
```bash
kubectl version --client
```

### 🔹 Definir o arquivo de configuração (kubeconfig)
Aponta o kubectl para um arquivo de credenciais/cluster específico. Adicione essa linha ao `.bashrc`/`.zshrc` para persistir entre sessões.
```bash
export KUBECONFIG=$HOME/.kube/kubeconfig
```

### 🔹 Mesclar múltiplos arquivos kubeconfig
Quando você tem acesso a mais de um cluster, é possível combinar vários kubeconfigs em um só arquivo, listando-os separados por `:`.
```bash
KUBECONFIG=$HOME/.kube/config-prod:$HOME/.kube/config-homolog kubectl config view --flatten > $HOME/.kube/config
```

### 🔹 Criar um atalho para o comando kubectl
Reduz `kubectl` para `k` — um dos aliases mais usados no dia a dia de quem opera clusters.
```bash
echo "alias k=kubectl" >> ~/.bashrc
source ~/.bashrc
```

### 🔹 Ativar autocomplete no shell
Habilita autocompletar subcomandos, flags e nomes de recursos ao pressionar Tab.
```bash
echo 'source <(kubectl completion bash)' >> ~/.bashrc
```

## 2. <span id="contextos-namespaces">🗂️ Contextos e Namespaces</span>

Um *contexto* agrupa cluster, usuário e namespace padrão em uma única referência nomeada — essencial para transitar com segurança entre ambientes (produção, homologação, etc.).

### 🔹 Listar os contextos disponíveis
O contexto marcado com `*` é o atualmente ativo.
```bash
kubectl config get-contexts
```

### 🔹 Selecionar um contexto
Troca o cluster/usuário/namespace padrão usado em todos os comandos seguintes.
```bash
kubectl config use-context contexto-producao
```

### 🔹 Ver o contexto atual
```bash
kubectl config current-context
```

### 🔹 Definir o namespace padrão do contexto atual
Evita ter que repetir `-n nome-do-namespace` em todo comando.
```bash
kubectl config set-context --current --namespace=nome-do-namespace
```

### 🔹 Criar um namespace
```bash
kubectl create namespace argocd
```

### 🔹 Listar todos os namespaces
```bash
kubectl get namespaces
```

### 🔹 Remover um namespace
⚠️ **Atenção:** remove também todos os recursos contidos nele (pods, services, configmaps, etc.).
```bash
kubectl delete namespace nome-do-namespace
```

## 3. <span id="gerenciamento-pods">📦 Gerenciamento de Pods</span>

### 🔹 Listar pods
Sem `-n`, usa o namespace padrão do contexto atual; `-A` lista pods de todos os namespaces.
```bash
kubectl get pods -n nome-do-namespace
kubectl get pods -A
```

### 🔹 Listar pods com mais detalhes
Mostra IP, node onde está agendado e quantidade de reinicializações.
```bash
kubectl get pods -o wide -n nome-do-namespace
```

### 🔹 Descrever um pod
Exibe eventos, condições, variáveis de ambiente e o motivo de falhas (`CrashLoopBackOff`, `ImagePullBackOff`, etc.) — o primeiro comando para diagnosticar um pod com problema.
```bash
kubectl describe pod poddaaplicacao-59bf74f8b4-pfrxh -n nome-do-namespace
```

### 🔹 Executar um comando dentro de um pod
Abre um shell interativo dentro do container principal do pod.
```bash
kubectl exec -it poddaaplicacao-59bf74f8b4-pfrxh -n nome-do-namespace -- /bin/bash
```

### 🔹 Executar em um container específico (pod multi-container)
```bash
kubectl exec -it poddaaplicacao-59bf74f8b4-pfrxh -c nome-do-container -n nome-do-namespace -- /bin/sh
```

### 🔹 Copiar arquivos de/para um pod
```bash
kubectl cp ./arquivo.conf nome-do-namespace/poddaaplicacao-59bf74f8b4-pfrxh:/app/arquivo.conf
kubectl cp nome-do-namespace/poddaaplicacao-59bf74f8b4-pfrxh:/var/log/app.log ./app.log
```

### 🔹 Encaminhar uma porta local para o pod (port-forward)
Permite acessar uma aplicação que roda dentro do cluster diretamente pelo `localhost`, sem precisar expor um Service — ótimo para depuração pontual.
```bash
kubectl port-forward pod/poddaaplicacao-59bf74f8b4-pfrxh 8080:80 -n nome-do-namespace
```

### 🔹 Remover um pod
Se o pod pertence a um Deployment/ReplicaSet, ele será recriado automaticamente — útil para forçar a recriação de um pod travado.
```bash
kubectl delete pod poddaaplicacao-59bf74f8b4-pfrxh -n nome-do-namespace
```

## 4. <span id="deployments-rollouts">🚀 Deployments e Rollouts</span>

### 🔹 Listar deployments
```bash
kubectl get deployments -n nome-do-namespace
```

### 🔹 Descrever um deployment
```bash
kubectl describe deployment nome-do-deployment -n nome-do-namespace
```

### 🔹 Escalar o número de réplicas
```bash
kubectl scale deployment nome-do-deployment --replicas=5 -n nome-do-namespace
```

### 🔹 Atualizar a imagem de um container em execução
Dispara um rollout progressivo (rolling update) para a nova versão da imagem, sem downtime.
```bash
kubectl set image deployment/nome-do-deployment nome-do-container=minhaimagem:v2.0 -n nome-do-namespace
```

### 🔹 Acompanhar o progresso de um rollout
```bash
kubectl rollout status deployment/nome-do-deployment -n nome-do-namespace
```

### 🔹 Ver o histórico de revisões
```bash
kubectl rollout history deployment/nome-do-deployment -n nome-do-namespace
```

### 🔹 Reverter para a revisão anterior
Essencial em um rollback de emergência após uma release problemática.
```bash
kubectl rollout undo deployment/nome-do-deployment -n nome-do-namespace
```

### 🔹 Reverter para uma revisão específica
```bash
kubectl rollout undo deployment/nome-do-deployment --to-revision=3 -n nome-do-namespace
```

### 🔹 Forçar a recriação de todos os pods (restart)
Recria os pods do deployment sem alterar a imagem ou configuração — útil para aplicar uma mudança em ConfigMap/Secret que não é detectada automaticamente.
```bash
kubectl rollout restart deployment/nome-do-deployment -n nome-do-namespace
```

## 5. <span id="services-rede">🌐 Services e Exposição de Rede</span>

### 🔹 Listar services
```bash
kubectl get svc -n nome-do-namespace
```

### 🔹 Expor um deployment como um Service
Cria automaticamente um Service apontando para os pods do deployment, sem precisar escrever o manifesto YAML manualmente.
```bash
kubectl expose deployment nome-do-deployment --port=80 --target-port=8080 --type=ClusterIP -n nome-do-namespace
```

### 🔹 Descrever um service
Mostra os endpoints (IPs dos pods) associados e o seletor de labels usado.
```bash
kubectl describe svc nome-do-service -n nome-do-namespace
```

### 🔹 Encaminhar uma porta local para um Service
```bash
kubectl port-forward svc/nome-do-service 8080:80 -n nome-do-namespace
```

### 🔹 Listar Ingress
Mostra as regras de roteamento HTTP/HTTPS que expõem serviços para fora do cluster.
```bash
kubectl get ingress -n nome-do-namespace
```

### 🔹 Descrever um Ingress
```bash
kubectl describe ingress nome-do-ingress -n nome-do-namespace
```

## 6. <span id="configmaps-secrets">🔐 ConfigMaps e Secrets</span>

### 🔹 Criar um ConfigMap a partir de literais
```bash
kubectl create configmap app-config --from-literal=APP_ENV=producao -n nome-do-namespace
```

### 🔹 Criar um ConfigMap a partir de um arquivo
```bash
kubectl create configmap app-config --from-file=./config.yaml -n nome-do-namespace
```

### 🔹 Ver o conteúdo de um ConfigMap
```bash
kubectl get configmap app-config -o yaml -n nome-do-namespace
```

### 🔹 Criar um Secret genérico
Os valores são automaticamente codificados em base64 pelo próprio kubectl — não é criptografia, apenas ofuscação; trate secrets com o mesmo cuidado de uma senha em texto puro.
```bash
kubectl create secret generic db-credentials --from-literal=senha=SenhaForte123 -n nome-do-namespace
```

### 🔹 Ver um Secret (decodificado)
```bash
kubectl get secret db-credentials -o jsonpath='{.data.senha}' -n nome-do-namespace | base64 -d
```

### 🔹 Criar um Secret do tipo docker-registry
Usado para autenticar o cluster em um registry privado ao puxar imagens.
```bash
kubectl create secret docker-registry regcred \
  --docker-server=registry.gustbrito.br \
  --docker-username=usuario \
  --docker-password=senha \
  -n nome-do-namespace
```

## 7. <span id="volumes-armazenamento">💾 Volumes e Armazenamento Persistente</span>

### 🔹 Listar Persistent Volumes (PV)
Recurso de nível de cluster que representa o armazenamento físico/provisionado disponível.
```bash
kubectl get pv
```

### 🔹 Listar Persistent Volume Claims (PVC)
A "solicitação" de armazenamento feita em um namespace, vinculada a um PV compatível.
```bash
kubectl get pvc -n nome-do-namespace
```

### 🔹 Descrever um PVC
Mostra o status do binding, capacidade, modo de acesso e o PV associado — útil quando um pod fica preso em `Pending` por falta de armazenamento.
```bash
kubectl describe pvc nome-do-pvc -n nome-do-namespace
```

### 🔹 Listar StorageClasses
Define como o armazenamento é provisionado dinamicamente (tipo de disco, driver CSI, política de retenção).
```bash
kubectl get storageclass
```

## 8. <span id="monitoramento-debug">📊 Monitoramento, Logs e Debug</span>

### 🔹 Ver os logs de um pod
```bash
kubectl logs poddaaplicacao-59bf74f8b4-pfrxh -n nome-do-namespace
```

### 🔹 Acompanhar logs em tempo real, limitando às últimas N linhas
```bash
kubectl logs -f --tail=30 poddaaplicacao-59bf74f8b4-pfrxh -n nome-do-namespace
```

### 🔹 Ver logs de um container específico dentro do pod
```bash
kubectl logs poddaaplicacao-59bf74f8b4-pfrxh -c nome-do-container -n nome-do-namespace
```

### 🔹 Ver logs da execução anterior de um pod (crash)
Fundamental para investigar um `CrashLoopBackOff`: mostra os logs do container antes do último reinício.
```bash
kubectl logs --previous poddaaplicacao-59bf74f8b4-pfrxh -n nome-do-namespace
```

### 🔹 Ver o consumo de CPU e memória dos pods
Requer o metrics-server instalado no cluster.
```bash
kubectl top pods -n nome-do-namespace
```

### 🔹 Ver o consumo de CPU e memória dos nodes
```bash
kubectl top nodes
```

### 🔹 Ver os eventos recentes do namespace
Mostra, em ordem cronológica, tudo que o control plane registrou (agendamentos, falhas de pull de imagem, falta de recursos, etc.) — ótimo ponto de partida em qualquer investigação.
```bash
kubectl get events -n nome-do-namespace --sort-by='.lastTimestamp'
```

## 9. <span id="manifests">📄 Aplicando e Gerenciando Manifests</span>

### 🔹 Aplicar um manifesto YAML
Cria ou atualiza recursos no cluster de forma declarativa a partir de um arquivo.
```bash
kubectl apply -f deployment.yaml
```

### 🔹 Aplicar todos os manifestos de um diretório
```bash
kubectl apply -f ./manifests/
```

### 🔹 Ver o que seria alterado, sem aplicar (dry-run)
Simula a aplicação do manifesto e mostra o diff, sem alterar nada no cluster — essencial antes de aplicar mudanças em produção.
```bash
kubectl diff -f deployment.yaml
```

### 🔹 Validar o manifesto sem enviar ao cluster
```bash
kubectl apply -f deployment.yaml --dry-run=client
```

### 🔹 Editar um recurso diretamente no cluster
Abre o recurso no editor padrão; ao salvar, o kubectl aplica a alteração automaticamente.
```bash
kubectl edit deployment nome-do-deployment -n nome-do-namespace
```

### 🔹 Remover recursos a partir de um manifesto
```bash
kubectl delete -f deployment.yaml
```

### 🔹 Exportar a definição atual de um recurso
Útil para versionar o estado real de um recurso criado manualmente ou para clonar configuração entre ambientes.
```bash
kubectl get deployment nome-do-deployment -n nome-do-namespace -o yaml > deployment-export.yaml
```

## 10. <span id="nodes-cluster">🖥️ Nodes e Manutenção do Cluster</span>

### 🔹 Listar os nodes do cluster
```bash
kubectl get nodes
```

### 🔹 Descrever um node
Mostra capacidade, alocação de recursos, taints, condições (`Ready`, `MemoryPressure`, etc.) e os pods agendados nele.
```bash
kubectl describe node nome-do-node
```

### 🔹 Impedir novos agendamentos em um node (cordon)
Marca o node como `SchedulingDisabled`, sem afetar os pods já em execução nele — o primeiro passo antes de uma manutenção.
```bash
kubectl cordon nome-do-node
```

### 🔹 Esvaziar um node com segurança (drain)
Remove/reagenda com segurança todos os pods do node para outros nodes disponíveis, respeitando PodDisruptionBudgets — usado antes de desligar ou atualizar uma máquina.
```bash
kubectl drain nome-do-node --ignore-daemonsets --delete-emptydir-data
```

### 🔹 Reabilitar agendamentos em um node (uncordon)
```bash
kubectl uncordon nome-do-node
```

### 🔹 Aplicar um taint em um node
Impede que pods sejam agendados no node a menos que tolerem explicitamente o taint — usado para reservar nodes para cargas específicas (ex.: GPU, alta prioridade).
```bash
kubectl taint nodes nome-do-node chave=valor:NoSchedule
```

## 11. <span id="rbac-seguranca">🛡️ RBAC e Segurança</span>

Controle de acesso baseado em papéis (Role-Based Access Control) — define quem pode fazer o quê em quais recursos do cluster.

### 🔹 Listar Service Accounts
Identidade usada por pods/aplicações para se autenticar contra a API do Kubernetes.
```bash
kubectl get serviceaccounts -n nome-do-namespace
```

### 🔹 Criar uma Service Account
```bash
kubectl create serviceaccount minha-app-sa -n nome-do-namespace
```

### 🔹 Listar Roles e ClusterRoles
`Role` vale apenas dentro de um namespace; `ClusterRole` vale para o cluster inteiro.
```bash
kubectl get roles -n nome-do-namespace
kubectl get clusterroles
```

### 🔹 Vincular uma Role a um usuário ou Service Account
```bash
kubectl create rolebinding minha-app-binding \
  --role=nome-da-role \
  --serviceaccount=nome-do-namespace:minha-app-sa \
  -n nome-do-namespace
```

### 🔹 Verificar se você (ou outra identidade) tem permissão para uma ação
Simula a checagem de RBAC sem executar de fato a ação — extremamente útil para depurar erros `Forbidden`.
```bash
kubectl auth can-i delete pods -n nome-do-namespace
kubectl auth can-i list secrets --as=system:serviceaccount:nome-do-namespace:minha-app-sa
```

## 12. <span id="helm">📦 Helm (Gerenciador de Pacotes)</span>

Helm empacota manifests Kubernetes em *charts* reutilizáveis e parametrizáveis — o gerenciador de pacotes de facto do ecossistema Kubernetes, presente no dia a dia de quem opera clusters.

### 🔹 Adicionar um repositório de charts
```bash
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update
```

### 🔹 Buscar um chart em repositórios adicionados
```bash
helm search repo bitnami/postgresql
```

### 🔹 Instalar um chart
Cria um *release* (instância nomeada do chart) no namespace informado.
```bash
helm install minha-app bitnami/postgresql -n nome-do-namespace
```

### 🔹 Listar releases instalados
```bash
helm list -n nome-do-namespace
```

### 🔹 Atualizar um release (upgrade)
Aplica uma nova versão do chart ou novos valores a um release já existente.
```bash
helm upgrade minha-app bitnami/postgresql -f values.yaml -n nome-do-namespace
```

### 🔹 Reverter um release para uma revisão anterior
```bash
helm rollback minha-app 1 -n nome-do-namespace
```

### 🔹 Remover um release
```bash
helm uninstall minha-app -n nome-do-namespace
```
