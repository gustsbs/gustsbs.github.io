# 🔀 Git - Comandos Rápidos

Anotações de configuração, ciclo de vida de arquivos, gerenciamento de branches e estratégias de integração de código.

## 📑 Índice de Tópicos
* [⚙️ Configurações Iniciais](#-configuracoes-iniciais-git-config)
* [🔎 Monitoramento e Status](#-monitoramento-e-status-git-status)
* [📥 Preparação e Snapshot](#-preparacao-e-snapshot-git-add--git-commit)
* [⏪ Navegação e Desfazer Código](#-navegacao-e-desfazer-codigo-git-checkout--git-reset)
* [🚀 Integração de Código: Merge vs Rebase](#-integracao-de-codigo-git-merge-vs-git-rebase)


---

## ⚙️ Configurações Iniciais (`git config`)
Para parametrizar o ambiente global do Git no host de desenvolvimento.

### Para configurar a identificação global do usuário
```bash
git config --global user.name "Gustavo de Brito dos Santos"
git config --global user.email "gustbrito@email.com.br"
```

### Para definir o Vim como editor de texto padrão das mensagens
```bash
git config --global core.editor vim
```

### Para alterar o nome padrão da branch de inicialização
```bash
git config --global init.defaultBranch teste
```

### Para listar e checar todas as configurações ativas
```bash
git config --list
```

## 🔎 Monitoramento e Status (git status)

O Git categoriza os arquivos locais em 4 estados principais:

**Untracked**: Arquivo novo que ainda não está a ser rastreado pelo Git.

**Unmodified**: Arquivo já rastreado que não sofreu alterações.

**Modified**: Arquivo rastreado modificado na pasta, mas fora da área de commit.

**Staged**: Arquivo pronto e selecionado para ser incluído no próximo commit.

### Para checar o status de rastreamento dos arquivos
```bash
git status
```

## 📥 Preparação e Snapshot (git add & git commit)

Fluxo para mover arquivos modificados para a área de preparação e salvar a versão.
### Para adicionar um arquivo específico à área de Staged
```bash
git add nomedoarquivo
```

### Para adicionar todas as modificações do diretório atual à área de Staged
```bash
git add .
```

### Para limpar a área de Staged (remover da fila de commit)
```bash
git reset
```

### Para criar um commit abrindo o editor de texto para a mensagem
```bash
git commit
```

### Para aplicar o commit inserindo a mensagem direto no comando
```bash
git commit -m "Aplica o commit e adiciona a informação sobre alterações"
```

### Para commitar arquivos modificados diretamente (ignora o git add manual)
```bash
git commit -a -m "Aplica o commit e adiciona a informação sobre alterações"
```

## 📜 Histórico de Commits (git log)

Para inspecionar a linha do tempo e auditoria do repositório.

### Para mostrar o log geral de commits
```bash
git log
```

### Para mostrar o histórico exibindo as linhas exatas de código modificadas
```bash
git log -p
```

## ⏪ Navegação e Desfazer Código (git checkout & git reset)

Ações para restaurar pontos antigos do histórico ou reverter commits com erros.

### Para navegar e recuperar um snapshot específico do histórico
```bash
git checkout 11986923015fab393d4da32d8f8e4b2cc96d99b9
```

### Para retornar do ponto do último commit para a sua branch atual
```bash
git checkout main
```

### Para desfazer o último commit mantendo o estado dos arquivos em staging
```bash 
git reset --soft HEAD~1
```

### Para descartar commits e alterações locais apagando tudo até o commit especificado
```bash
git reset --hard
```

## 🌿 Gerenciamento de Branches (git branch)

Isolamento e controle do fluxo de ramificações do repositório.

### Para listar todas as branches locais existentes
```bash
git branch
```

### Para criar uma nova branch a partir do ponto atual
```bash
git branch nomedabranch
```

### Para criar uma nova branch e mudar para ela imediatamente
```bash
git checkout -b nomedabranch/nomeprojeto
```

### Para excluir uma branch local de feature com segurança
```bash
git branch -d feature/novoarquivo
```

## 🚀 Integração de Código: Git Merge vs Git Rebase

### 1. git merge - Combina o histórico de dois ramos criando um novo commit de merge.
```bash
git merge nomedabranch
```
O que faz: Cria um commit de junção na branch atual, interligando os dois históricos de forma paralela (não-linear).

Prós: Preserva o histórico real completo de cada ramificação, sendo ideal para auditorias em projetos de equipas grandes.

Contras: O log visual do histórico pode ficar poluído com commits automáticos de merge.

### 2. git rebase
Reaplica os commits da branch atual diretamente sobre o topo da outra branch indicada.
```bash
git rebase nomedabranch
```
O que faz: Remove temporariamente os seus commits locais, puxa as atualizações da outra branch e reaplica as suas alterações por cima, gerando uma linha do tempo única e linear.

Prós: Produz um histórico extremamente limpo e fácil de ler.

Contras: Modifica o histórico original do repositório. Nunca deve ser usado em branches públicas compartilhadas no servidor local da instituição (risco de quebrar os clones locais dos outros programadores).