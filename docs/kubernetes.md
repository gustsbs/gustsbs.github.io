# ☸️ Kubernetes

### Comandos de Verificação de Pods
Para checar os pods em execução
```bash
kubectl get pods -n namespace
```

### Analisa as 30 últimas linhas de um pod, acompanhando em tempo real
```bash
sudo kubectl logs -f --tail=30 -n nomedonamespace poddaaplicacao-59bf74f8b4-pfrxh 
```

