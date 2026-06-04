# 🐙Guia de Comandos ARGOCD

## 📑 Índice

1. [Instalação - 04/06/20226](#instalacao)
2. [Operação](#operacao)


## 1. <span id=#instalacao> 📦 Instalação do ARGOCD em cluster Kubernetes</span>
Referência: https://argo-cd.readthedocs.io/en/stable/cli_installation/

Link da documentação oficial: https://argo-cd.readthedocs.io/en/stable/getting_started/

### baixar o manifesto
```bash
curl -o argocd-install.yaml https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

### instalar no namespace argocd (suponha que este já tenha sido criado)
Aplica-se o manifesto local com Server-Side
```bash
kubectl apply -n argocd -f argocd-install.yaml --server-side
```

## 2. <span id=#operacao> Operação do ARGOCD

### Adicionar os contextos existentes ao KUBECONFIG
Aqui tenho dois contextos, um kubernetes de teste e outro produção. Veja em kubernetes os comandos para exibição dos contextos
```bash
export KUBECONFIG=/home/gustavo/.kube/kubernetes_producao.conf:/home/gustavo/.kube/kubernetes_teste.conf
```

 ```bash
argocd cluster add contexto-teste --name kubernetes-teste --grpc-web
```

 ```bash
argocd cluster add contexto-prod --name kubernetes-prod --grpc-web
``

### Capturar a primeira senha de admin
```bash
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d; echo
```

### Efetuar o login 
O --grpc-web é uma camada de tradução: Ele instrui a CLI do ArgoCD a usar um protocolo chamado gRPC-Web.
Como funciona: Ele encapsula as mensagens gRPC dentro de requisições HTTP comuns que navegadores e Proxies (como o seu Nginx) conseguem entender e encaminhar sem problemas.
```bash
argocd login argocd.servidor.dominio.br --username admin --grpc-web
```

### Atualização da senha de admin ( recomendado após o primeiro login)
```bash
argocd account update-password --account admin --current-password <SENHA_ANTIGA> --new-password <NOVA_SENHA>
```

### Adicinando um cluster kubernetes
```bash
argocd cluster add contexto-teste --name kubernetes-teste --grpc-web
```

