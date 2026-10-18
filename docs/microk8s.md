# 🔬 MicroK8s - Comandos Rápidos

Instalação de um cluster Kubernetes local com MicroK8s no WSL 2, para laboratório e testes antes de aplicar manifestos no cluster de produção, incluindo os componentes adicionais usados no fluxo GitOps (Sealed Secrets e Argo CD). Os comandos do dia a dia com `kubectl` estão em [kubernetes.md](kubernetes.md).

## 📑 Índice

1. [Preparação do WSL 2](#preparacao-wsl)
2. [Instalação do MicroK8s](#instalacao)
3. [Addons Essenciais](#addons)
4. [Acesso ao Cluster sem Misturar Contextos](#acesso-contexto)
5. [Descobrindo as Versões da Produção](#versoes-producao)
6. [Sealed Secrets via Helm](#sealed-secrets)
7. [Argo CD](#argocd)
8. [Acesso pelo Navegador do Windows](#acesso-navegador)
9. [Imagens de Container no MicroK8s](#imagens)
10. [Operação do Dia a Dia](#operacao)
11. [Boas Práticas e Pegadinhas](#boas-praticas)
---

## 1. <span id="preparacao-wsl">🪟 Preparação do WSL 2</span>

O MicroK8s é distribuído via **snap**, e o snap depende do **systemd** — por isso o WSL precisa estar na versão 2 e com systemd habilitado.

### 🔹 Conferir a versão do WSL (PowerShell)
A distribuição precisa aparecer com `VERSION 2`.
```powershell
wsl -l -v
wsl --set-version Ubuntu 2    # apenas se estiver na versão 1
```

### 🔹 Reservar recursos para o WSL
Arquivo `C:\Users\<USUARIO>\.wslconfig` no Windows. O modo `mirrored` faz o WSL usar a rede do Windows (inclusive VPN), o que ajuda a alcançar serviços internos como banco de dados e S3.
```ini
[wsl2]
memory=8GB
processors=4
networkingMode=mirrored
```

### 🔹 Habilitar o systemd no Ubuntu do WSL
Depois de criar o arquivo, reinicie o WSL pelo PowerShell com `wsl --shutdown` e abra o Ubuntu de novo.
```bash
sudo tee /etc/wsl.conf >/dev/null <<'CONF'
[boot]
systemd=true
CONF
```

### 🔹 Verificar se o systemd está ativo
`running` ou `degraded` indicam que o systemd está funcionando.
```bash
systemctl is-system-running
```

## 2. <span id="instalacao">⚙️ Instalação do MicroK8s</span>

### 🔹 Instalar na mesma versão do cluster de produção
O `--channel` fixa a versão minor do Kubernetes — use a mesma da produção para que o laboratório se comporte igual.
```bash
sudo snap install microk8s --classic --channel=1.32/stable
```

### 🔹 Liberar o uso sem sudo
Adiciona o usuário ao grupo `microk8s` e ajusta a permissão da pasta `~/.kube`.
```bash
sudo usermod -aG microk8s $USER
mkdir -p ~/.kube && sudo chown -f -R $USER ~/.kube
newgrp microk8s
```

### 🔹 Aguardar o cluster ficar pronto
```bash
microk8s status --wait-ready
microk8s kubectl get nodes -o wide
```

### 🔹 Trocar o canal (versão) de uma instalação existente
```bash
sudo snap refresh microk8s --channel=1.32/stable
```

## 3. <span id="addons">🧩 Addons Essenciais</span>

### 🔹 Habilitar DNS, ingress, storage local e Helm
O addon `ingress` instala o ingress-nginx, o mesmo controller usado em produção; o `hostpath-storage` cria uma StorageClass local para PVCs de teste.
```bash
microk8s enable dns ingress hostpath-storage helm3
microk8s kubectl get pods -A
```

| Addon | Para que serve |
| :--- | :--- |
| `dns` | CoreDNS — resolução de nomes de Services dentro do cluster |
| `ingress` | ingress-nginx escutando nas portas 80/443 do host |
| `hostpath-storage` | StorageClass que grava em disco local (só para laboratório) |
| `helm3` | Cliente Helm embutido (`microk8s helm3`) |

## 4. <span id="acesso-contexto">🔐 Acesso ao Cluster sem Misturar Contextos</span>

### 🔹 Exportar um kubeconfig separado para o laboratório
Mantém o `kubectl` padrão apontando para a produção e cria um arquivo exclusivo para o MicroK8s.
```bash
microk8s config > ~/.kube/microk8s.config
```

### 🔹 Criar aliases para o laboratório
`klab` e `hlab` sempre apontam para o MicroK8s; `kubectl` e `helm` continuam apontando para a produção.
```bash
cat >> ~/.bashrc <<'ALIAS'
alias klab='kubectl --kubeconfig ~/.kube/microk8s.config'
alias hlab='microk8s helm3 --kubeconfig ~/.kube/microk8s.config'
ALIAS
source ~/.bashrc
klab get nodes
```

## 5. <span id="versoes-producao">🔎 Descobrindo as Versões da Produção</span>

Para o laboratório reproduzir a produção, instale os componentes na mesma versão e com o mesmo nome/namespace.

### 🔹 Listar namespace, nome e imagem de todos os deployments
Busca pela imagem, sem depender de labels (que mudam conforme o método de instalação).
```bash
kubectl get deploy -A -o jsonpath='{range .items[*]}{.metadata.namespace}{"\t"}{.metadata.name}{"\t"}{.spec.template.spec.containers[0].image}{"\n"}{end}' | grep -iE 'sealed|argocd'
```

### 🔹 Descobrir se foi instalado via Helm e com qual chart
Os labels `app.kubernetes.io/managed-by: Helm` e `helm.sh/chart` mostram a versão do chart, mesmo quando o `helm list` da sua máquina não lista a release (instalação feita de outra máquina).
```bash
kubectl -n kube-system get deploy sealed-secrets -o jsonpath='{.metadata.labels}' | tr ',' '\n'
```

Exemplo de saída:
```text
"app.kubernetes.io/managed-by":"Helm"
"app.kubernetes.io/version":"0.37.0"
"helm.sh/chart":"sealed-secrets-2.18.6"
```

## 6. <span id="sealed-secrets">🔏 Sealed Secrets via Helm</span>

Instalação idêntica à produção: chart `2.18.6` (app `0.37.0`), release `sealed-secrets` no `kube-system`. Conceitos e uso do `kubeseal` em [kubernetes.md](kubernetes.md#sealed-secrets).

### 🔹 Baixar o chart da release e conferir o hash
O repositório mudou de `bitnami-labs` para `bitnami` e o índice Helm antigo retorna 404 — baixe o `.tgz` direto da release no GitHub e confira o SHA-256 publicado nela.
```bash
cd ~ && curl -fsSLO https://github.com/bitnami/sealed-secrets/releases/download/helm-v2.18.6/sealed-secrets-2.18.6.tgz
echo "4af8cb2b150b83460f15242e36706159cb77f6ba834f79a7bd172cb2f5debc72  sealed-secrets-2.18.6.tgz" | sha256sum -c -
```

### 🔹 Instalar o chart no laboratório
Usando a release `sealed-secrets`, o deployment recebe o mesmo nome da produção.
```bash
hlab install sealed-secrets ./sealed-secrets-2.18.6.tgz -n kube-system
klab -n kube-system rollout status deploy/sealed-secrets --timeout=180s
klab -n kube-system get deploy sealed-secrets -o jsonpath='{..image}'; echo
```

### 🔹 Exportar o certificado público do laboratório
Instalado via Helm, o controller se chama `sealed-secrets` (não `sealed-secrets-controller`), por isso os parâmetros `--controller-*` são obrigatórios.
```bash
kubeseal --kubeconfig ~/.kube/microk8s.config \
  --controller-name sealed-secrets --controller-namespace kube-system \
  --fetch-cert > ~/.kube/lab-sealed.pem
```

## 7. <span id="argocd">🐙 Argo CD</span>

### 🔹 Instalar o Argo CD na versão da produção
O `--server-side` evita o erro `metadata.annotations: Too long` nos CRDs. Substitua a versão pela que roda em produção (seção [Descobrindo as Versões da Produção](#versoes-producao)).
```bash
ARGO_VER=v3.4.0
klab create namespace argocd
klab apply -n argocd --server-side -f https://raw.githubusercontent.com/argoproj/argo-cd/${ARGO_VER}/manifests/install.yaml
klab -n argocd rollout status deploy/argocd-server --timeout=300s
```

### 🔹 Obter a senha inicial do admin
```bash
klab -n argocd get secret argocd-initial-admin-secret -o jsonpath='{.data.password}' | base64 -d; echo
```

### 🔹 Acessar a interface web
Depois, abra `https://localhost:8443` no navegador do Windows (certificado autoassinado).
```bash
klab -n argocd port-forward svc/argocd-server 8443:443
```

## 8. <span id="acesso-navegador">🌐 Acesso pelo Navegador do Windows</span>

### 🔹 Apontar um domínio de teste para o ingress
O ingress do MicroK8s escuta nas portas 80/443 do WSL, que o WSL 2 repassa para o Windows. Edite como administrador `C:\Windows\System32\drivers\etc\hosts`:
```text
127.0.0.1   site-base.lab
```

### 🔹 Testar conectividade com serviços externos ao cluster
Antes de configurar uma aplicação, confirme que o WSL alcança o banco de dados e o S3 da rede interna.
```bash
curl -s -o /dev/null -w '%{http_code}\n' http://<IP_DO_S3>
nc -zv <IP_DO_BANCO> 3306
```

## 9. <span id="imagens">📦 Imagens de Container no MicroK8s</span>

O MicroK8s tem o seu próprio containerd e **não enxerga as imagens do Docker** instalado no WSL.

### 🔹 Usar o registry corporativo (recomendado)
Testa o mesmo caminho da produção: faça push para o registry e crie um `imagePullSecret` no namespace.
```bash
klab -n wp-site-base-01 create secret docker-registry registry-ufpr \
  --docker-server=registry.gustbrito.br --docker-username=<USUARIO> --docker-password=<TOKEN>
```

### 🔹 Importar uma imagem local direto no containerd
Alternativa rápida para testes sem push.
```bash
docker save registry.gustbrito.br/csoft/wordpress:6.8.3-v1.0 | microk8s ctr image import -
microk8s ctr images ls | grep wordpress
```

## 10. <span id="operacao">🔧 Operação do Dia a Dia</span>

### 🔹 Parar e iniciar o cluster
Libera CPU e memória do WSL quando o laboratório não estiver em uso.
```bash
microk8s stop
microk8s start && microk8s status --wait-ready
```

### 🔹 Gerar relatório de diagnóstico
Quando o `status` não fica pronto ou um addon não sobe.
```bash
microk8s inspect
```

### 🔹 Recriar o laboratório do zero
⚠️ **Atenção:** remove todos os recursos e dados do cluster local (inclusive a chave do Sealed Secrets do laboratório).
```bash
microk8s reset
```

## 11. <span id="boas-praticas">💡 Boas Práticas e Pegadinhas</span>

### 🔹 `helm repo add` do Sealed Secrets retorna 404
O projeto mudou de `bitnami-labs/sealed-secrets` para `bitnami/sealed-secrets`. Os links de *releases* antigos redirecionam, mas o `index.yaml` do repositório Helm não — instale a partir do `.tgz` da release, conferindo o SHA-256.

### 🔹 `helm list` vazio, mas o componente foi instalado via Helm
A release fica registrada no cluster, mas o `helm` local pode não estar instalado ou apontar para outro contexto. Os labels `helm.sh/chart` e `app.kubernetes.io/version` do deployment mostram a versão real.

### 🔹 Nome do controller do Sealed Secrets
`controller.yaml` oficial cria `sealed-secrets-controller`; o chart Helm cria `sealed-secrets`. O `kubeseal` assume o primeiro por padrão — informe sempre `--controller-name` e `--controller-namespace`.

### 🔹 Chaves do Sealed Secrets separadas por ambiente
O controller do laboratório gera um par de chaves próprio: um `SealedSecret` selado para o laboratório não abre na produção, e vice-versa. Isso é o comportamento correto — nunca copie a chave da produção para o laboratório.

### 🔹 Contexto errado
- ✅ Use `klab`/`hlab` para o laboratório e confira `kubectl config current-context` antes de um `apply` na produção.
- ✅ Fixe a mesma versão do Kubernetes (`--channel`), dos charts e das imagens da produção.
- ✅ Mantenha banco de dados e S3 fora do cluster local, como na produção — o overlay muda só endereços e credenciais.
- ✅ Prefira fixar charts e imagens por hash (SHA-256/digest), não apenas por tag.
