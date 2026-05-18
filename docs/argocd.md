# 🐙Guia de Comandos ARGOCD


1. [Instalação e utilização] 

### Instalando o cliente de administração
```bash
curl -sSL -o argocd-linux-amd64 https://github.com/argoproj/argo-cd/releases/latest/download/argocd-linux-amd64

sudo install -m 555 argocd-linux-amd64 /usr/local/bin/argocd

rm argocd-linux-amd64
```
Referência: https://argo-cd.readthedocs.io/en/stable/cli_installation/


### Efetuando o Login no console de administração
```bash
argocd login argocd.homologa.gustbrito.br
```




Link da documentação oficial: https://argo-cd.readthedocs.io/en/stable/getting_started/