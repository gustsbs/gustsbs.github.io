# ☸️ Kubernetes


### Instalação do client kubectl

```bash
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"

sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl
```
### Exportar variável de ambiente (adicionar no bashrc do usuário)

```bash
export KUBECONFIG=$HOME/.kube/kubeconfig
```

### Comandos de Verificação de Pods
Para checar os pods em execução
```bash
kubectl get pods -n namespace
```

### Analisa as 30 últimas linhas de um pod, acompanhando em tempo real
```bash
sudo kubectl logs -f --tail=30 -n nomedonamespace poddaaplicacao-59bf74f8b4-pfrxh 
```

