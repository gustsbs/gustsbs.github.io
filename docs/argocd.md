# 🐙 Guia de Comandos ArgoCD

Anotações de instalação, configuração de acesso e operação do dia a dia do ArgoCD para entrega contínua (GitOps) em clusters Kubernetes.

## 📑 Índice

1. [Instalação (Cluster & CLI)](#instalacao)
2. [Configuração Inicial e Login](#login-configuracao)
3. [Gerenciamento de Clusters e Contextos](#clusters-contextos)
4. [Gerenciamento de Aplicações](#gerenciamento-aplicacoes)
5. [Sincronização e Rollback](#sincronizacao-rollback)
6. [Repositórios Git](#repositorios-git)
7. [Projetos (AppProjects)](#projetos)
8. [Contas, RBAC e Segurança](#contas-rbac)
9. [ApplicationSets](#applicationsets)
10. [Boas Práticas](#boas-praticas)
---

## 1. <span id="instalacao">📦 Instalação (Cluster & CLI)</span>

Referência oficial: https://argo-cd.readthedocs.io/en/stable/getting_started/

### 🔹 Criar o namespace do ArgoCD
```bash
kubectl create namespace argocd
```

### 🔹 Baixar e aplicar o manifesto de instalação
Baixa o manifesto oficial localmente antes de aplicar — permite revisar (ou versionar) o conteúdo antes de subir ao cluster.
```bash
curl -o argocd-install.yaml https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
kubectl apply -n argocd -f argocd-install.yaml --server-side
```
> ℹ️ Para ambientes de produção com alta disponibilidade, use o manifesto `ha/install.yaml` do mesmo repositório em vez do `install.yaml` padrão.

### 🔹 Acompanhar a subida dos pods
```bash
kubectl get pods -n argocd -w
```

### 🔹 Instalar o CLI do ArgoCD (Linux)
Referência: https://argo-cd.readthedocs.io/en/stable/cli_installation/
```bash
curl -sSL -o argocd-linux-amd64 https://github.com/argoproj/argo-cd/releases/latest/download/argocd-linux-amd64
sudo install -m 555 argocd-linux-amd64 /usr/local/bin/argocd
rm argocd-linux-amd64
```

### 🔹 Verificar a versão instalada (client e server)
```bash
argocd version
```

## 2. <span id="login-configuracao">🔑 Configuração Inicial e Login</span>

### 🔹 Capturar a senha inicial do usuário admin
O ArgoCD gera automaticamente uma senha de admin no primeiro deploy, armazenada em um Secret.
```bash
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d; echo
```

### 🔹 Acessar a interface via port-forward
Alternativa rápida para acessar a UI/API sem depender de um Ingress já configurado.
```bash
kubectl port-forward svc/argocd-server -n argocd 8080:443
```

### 🔹 Efetuar login via CLI
A flag `--grpc-web` instrui a CLI a encapsular as mensagens gRPC dentro de requisições HTTP comuns — necessário quando o servidor ArgoCD está atrás de um proxy/Ingress (como Nginx) que não repassa gRPC nativo.
```bash
argocd login argocd.servidor.dominio.br --username admin --grpc-web
```

### 🔹 Atualizar a senha do admin
Recomendado logo após o primeiro login, substituindo a senha gerada automaticamente.
```bash
argocd account update-password --account admin --current-password <SENHA_ANTIGA> --new-password <NOVA_SENHA>
```

### 🔹 Encerrar a sessão
```bash
argocd logout argocd.servidor.dominio.br
```

## 3. <span id="clusters-contextos">🗂️ Gerenciamento de Clusters e Contextos</span>

O ArgoCD pode gerenciar aplicações tanto no próprio cluster onde está instalado quanto em clusters externos registrados.

### 🔹 Combinar múltiplos kubeconfigs em uma única variável
Necessário para que a CLI enxergue todos os contextos que serão registrados no ArgoCD (veja também a seção de contextos em [kubernetes.md](kubernetes.md#contextos-namespaces)).
```bash
export KUBECONFIG=$HOME/.kube/kubernetes_producao.conf:$HOME/.kube/kubernetes_teste.conf
```

### 🔹 Registrar um cluster no ArgoCD
Associa um contexto do kubeconfig local a um cluster gerenciado pelo ArgoCD, permitindo implantar aplicações nele.
```bash
argocd cluster add contexto-teste --name kubernetes-teste --grpc-web
argocd cluster add contexto-prod --name kubernetes-prod --grpc-web
```

### 🔹 Listar os clusters registrados
```bash
argocd cluster list
```

### 🔹 Remover um cluster registrado
```bash
argocd cluster rm kubernetes-teste
```

## 4. <span id="gerenciamento-aplicacoes">🚀 Gerenciamento de Aplicações</span>

Uma *Application* no ArgoCD é a unidade que vincula um repositório Git (fonte da verdade) a um destino (cluster + namespace), mantendo-os sincronizados.

### 🔹 Criar uma aplicação via CLI
```bash
argocd app create minha-app \
  --repo https://gitlab.gustbrito.br/sistemas/minha-app.git \
  --path k8s/overlays/producao \
  --dest-server https://kubernetes.default.svc \
  --dest-namespace producao
```

### 🔹 Listar aplicações
```bash
argocd app list
```

### 🔹 Ver detalhes de uma aplicação
Mostra status de sincronização (sync), saúde (health), histórico e os recursos gerenciados.
```bash
argocd app get minha-app
```

### 🔹 Ver a árvore de recursos de uma aplicação
Exibe hierarquicamente todos os recursos Kubernetes criados/gerenciados pela aplicação (deployments, pods, services, etc.).
```bash
argocd app resources minha-app
```

### 🔹 Editar parâmetros de uma aplicação existente
```bash
argocd app set minha-app --dest-namespace homologacao
```

### 🔹 Remover uma aplicação
Por padrão, remove apenas o registro no ArgoCD; `--cascade` remove também os recursos que ela gerenciava no cluster.
```bash
argocd app delete minha-app
argocd app delete minha-app --cascade
```

## 5. <span id="sincronizacao-rollback">🔄 Sincronização e Rollback</span>

### 🔹 Ver o diff entre o Git e o estado atual do cluster
Mostra exatamente o que seria alterado antes de sincronizar — o equivalente ao `kubectl diff` do ArgoCD.
```bash
argocd app diff minha-app
```

### 🔹 Sincronizar manualmente uma aplicação
Aplica no cluster o estado definido no repositório Git, mesmo com o sync automático desativado.
```bash
argocd app sync minha-app
```

### 🔹 Forçar a sincronização, ignorando diferenças já aplicadas
Recria/reaplica os recursos mesmo que o ArgoCD acredite que já estão sincronizados — útil quando alguém alterou algo manualmente no cluster.
```bash
argocd app sync minha-app --force
```

### 🔹 Habilitar sync automático
Faz o ArgoCD aplicar mudanças do Git automaticamente, sem intervenção manual, sempre que detectar divergência.
```bash
argocd app set minha-app --sync-policy automated
```

### 🔹 Habilitar self-heal e prune no sync automático
`self-heal` corrige automaticamente alterações manuais feitas fora do Git; `auto-prune` remove recursos que foram excluídos do repositório.
```bash
argocd app set minha-app --sync-policy automated --self-heal --auto-prune
```

### 🔹 Ver o histórico de sincronizações
```bash
argocd app history minha-app
```

### 🔹 Reverter para uma revisão anterior
```bash
argocd app rollback minha-app <ID_DA_REVISAO>
```

### 🔹 Aguardar a aplicação ficar saudável e sincronizada
Bloqueia o terminal até que a aplicação atinja o status `Healthy` e `Synced` — muito usado em pipelines de CI/CD logo após disparar um deploy.
```bash
argocd app wait minha-app
```

## 6. <span id="repositorios-git">📚 Repositórios Git</span>

### 🔹 Adicionar um repositório Git privado (usuário/token)
```bash
argocd repo add https://gitlab.gustbrito.br/sistemas/minha-app.git --username usuario --password token
```

### 🔹 Adicionar um repositório via chave SSH
```bash
argocd repo add git@gitlab.gustbrito.br:sistemas/minha-app.git --ssh-private-key-path ~/.ssh/id_ed25519
```

### 🔹 Listar repositórios registrados
```bash
argocd repo list
```

### 🔹 Remover um repositório registrado
```bash
argocd repo rm https://gitlab.gustbrito.br/sistemas/minha-app.git
```

## 7. <span id="projetos">🗃️ Projetos (AppProjects)</span>

*Projects* agrupam aplicações e restringem o que elas podem fazer: quais repositórios, clusters e tipos de recurso são permitidos — a base do controle multi-tenant no ArgoCD.

### 🔹 Criar um projeto
```bash
argocd proj create producao --dest https://kubernetes.default.svc,producao --src https://gitlab.gustbrito.br/sistemas/*
```

### 🔹 Listar projetos
```bash
argocd proj list
```

### 🔹 Ver detalhes de um projeto
```bash
argocd proj get producao
```

### 🔹 Permitir um tipo de recurso de cluster em um projeto
Por padrão, projetos restringem quais tipos de recurso (CRDs, ClusterRoles, etc.) suas aplicações podem criar — esse comando abre exceção para um tipo específico.
```bash
argocd proj allow-cluster-resource producao apiextensions.k8s.io CustomResourceDefinition
```

## 8. <span id="contas-rbac">🛡️ Contas, RBAC e Segurança</span>

### 🔹 Listar contas locais
```bash
argocd account list
```

### 🔹 Ver detalhes e permissões de uma conta
```bash
argocd account get --account admin
```

### 🔹 Gerar um token de API para automações
Usado para autenticar pipelines de CI/CD contra a API do ArgoCD sem depender de login interativo.
```bash
argocd account generate-token --account ci-cd-bot
```

### 🔹 Editar a política RBAC do ArgoCD
Abre o ConfigMap `argocd-rbac-cm`, onde são definidas as permissões por papel e por grupo.
```bash
kubectl edit configmap argocd-rbac-cm -n argocd
```

## 9. <span id="applicationsets">🧬 ApplicationSets</span>

`ApplicationSet` gera e gerencia múltiplas Applications a partir de um único template — usado para deploys multi-cluster ou multi-ambiente (ex.: a mesma aplicação em `dev`, `homologação` e `produção`) sem duplicar manifestos manualmente.

### 🔹 Listar ApplicationSets
```bash
argocd appset list
```

### 🔹 Ver detalhes de um ApplicationSet
```bash
argocd appset get meu-applicationset
```

### 🔹 Criar um ApplicationSet a partir de um manifesto
ApplicationSets normalmente são declarados em YAML e aplicados via `kubectl`, e não criados diretamente pela CLI do ArgoCD.
```bash
kubectl apply -f applicationset.yaml -n argocd
```

## 10. <span id="boas-praticas">🧭 Boas Práticas</span>

### 🔹 Padrão App of Apps
Uma Application "raiz" que, em vez de apontar para manifestos de infraestrutura, aponta para um diretório contendo outras definições de Application — permite gerenciar dezenas de aplicações a partir de um único ponto de entrada, com todo o controle de acesso já garantido pelo próprio Git.

### 🔹 Sync Waves
Anotação (`argocd.argoproj.io/sync-wave`) que define a ordem de aplicação dos recursos dentro de uma mesma sincronização — útil para garantir que um CRD ou Secret exista antes do Deployment que depende dele.
```yaml
metadata:
  annotations:
    argocd.argoproj.io/sync-wave: "-1"
```

### 🔹 Nunca edite recursos gerenciados diretamente no cluster
Qualquer alteração manual feita fora do Git será revertida pelo ArgoCD caso `self-heal` esteja ativo — e, mesmo sem ele, o recurso aparecerá como `OutOfSync` até a próxima sincronização. Trate o Git como única fonte da verdade (o próprio princípio de GitOps).
