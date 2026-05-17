# 🔀 Git - Comandos Rápidos

Anotações de configuração, ciclo de vida de arquivos, gerenciamento de branches e estratégias de integração de código.

## 📑 Índice de Tópicos
1. [Configurações Iniciais](#config-inicial)
2. [Monitoramento e Status](#monitoramento-e-status)
3. [Preparação e Snapshot](#git-add-commit)
4. [Histórico de Commits (git log)](#git-log)
5. [Navegação e Desfazer Código](#git-checkout-reset)
6. [Gerenciamento de Branches (git branch)](#git-branch)
7. [Integração de Código: Merge vs Rebase](#git-merge-rebase)
8. [Repositórios Remotos e Sincronização (git remote & git push](#git-push-remote)
9. [Definição de Marcos e Versões](#git-tag)
10. [Desconsiderando Arquivos](#desconsiderar-ignore)
11. [Sincronização de Entrada (`git pull`)](#git-pull-buscar)
---

## 1. <span id="config-inicial"> ⚙️ Configurações Iniciais (`git config`)</span>
Para parametrizar o ambiente global do Git no host de desenvolvimento.

### Para configurar a identificação global do usuário
```bash
git config --global user.name "Gustavo de Brito dos Santos"
git config --global user.email "gustbrito@email.com.br"
```
### Para salvar as credenciais de acesso no cache temporário do host
```bash
git config --global credential.helper cache
```
### Para definir o Vim como editor de texto padrão das mensagens
```bash
git config --global core.editor vim
```
### Merge Padrão (Default): Caso haja divergências entre o seu código e o servidor, o Git criará automaticamente um commit de merge para unir os dois históricos.
```bash
git config pull.rebase false
```
### Rebase por Padrão: Força o Git a tentar fazer um rebase sempre que você der git pull, evitando a criação de commits de merge automáticos e mantendo o histórico linear.
```bash
git config pull.rebase true
```
### Fast-Forward Only: Trava o comando de pull. O Git só atualizará sua máquina se não houver conflitos ou commits locais divergentes. Se houver, ele aborta a operação e pede para você resolver manualmente.
```bash
git config pull.ff only
```

### Para alterar o nome padrão da branch de inicialização
```bash
git config --global init.defaultBranch teste
```

### Para listar e checar todas as configurações ativas
```bash
git config --list
```

## 2. <span id="monitoramento-e-status">🔍Monitoramento e Status</span>

O Git categoriza os arquivos locais em 4 estados principais:

**Untracked**: Arquivo novo que ainda não está a ser rastreado pelo Git.

**Unmodified**: Arquivo já rastreado que não sofreu alterações.

**Modified**: Arquivo rastreado modificado na pasta, mas fora da área de commit.

**Staged**: Arquivo pronto e selecionado para ser incluído no próximo commit.

### Para checar o status de rastreamento dos arquivos
```bash
git status
```

## <span id="git-add-commit"> 3. 📥 Preparação e Snapshot (git add & git commit)</span>

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

## <span id="git-log"> 4. 📜 Histórico de Commits (git log)</span>

Para inspecionar a linha do tempo e auditoria do repositório.

### Para mostrar o log geral de commits
```bash
git log
```

### Para mostrar o histórico exibindo as linhas exatas de código modificadas
```bash
git log -p
```
### Para listar os commits em uma única linha simplificada
```bash
git log --oneline
```
### Para mostrar apenas uma quantidade específica de commits recentes
```bash
git log -n 2
```

### Para filtrar o histórico por um autor específico
```bash
git log --author="Gustavo de Brito dos Santos"
```
### Para filtrar commits realizados a partir de um período de tempo
```bash
git log --after="1 week ago"
```
### Para exibir as estatísticas de arquivos modificados e linhas alteradas por commit
```bash
git log --stat
```

### Para renderizar o histórico de branches em formato de gráfico de linhas no terminal
```bash
git log --graph --oneline
```

### Para isolar e inspecionar o histórico de um arquivo específico
```bash
git log nomedoarquivo
```

## <span id="git-checkout-reset"> 5. ⏪ Navegação e Desfazer Código</span>

### Para selecionar ou alternar para uma branch existente
```bash
git branch master
```

### Para navegar e recuperar um snapshot específico do histórico
```bash
git checkout 11986923015fab393d4da32d8f8e4b2cc96d99b9
```

### Para retornar do ponto do último commit para a sua branch atual
```bash
git checkout main
```
### Para buscar um commit específico de outra branch e aplicá-lo na atual (Cherry-Pick)
```bash
git cherry-pick 505a880bc020864bc69ed44599852cd99b679789
```

### Para desfazer o último commit mantendo o estado dos arquivos em staging
```bash 
git reset --soft HEAD~1
```

### Para descartar commits e alterações locais apagando tudo até o commit especificado
```bash
git reset --hard
```

## <span id="git-branch"> 6. 🌿 Gerenciamento de Branches (git branch)</span>

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

## <span id="git-merge-rebase"> 7. 🚀 Integração de Código: Git Merge vs Git Rebase</span>

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

## <span id="git-push-remote"> 8. 🌐 Repositórios Remotos e Sincronização </span>

### Para clonar um repositório remoto para a sua máquina local
```bash
git clone git@github.com:MeuRepo/projetoxyz.git
```

### Para vincular um repositório remoto ao seu diretório local
```bash
git remote add origin git@github.com:gustbrito/asterisk.git
```

### Para enviar os dados pela primeira vez definindo a branch padrão no servidor
```bash
git push -u origin main
```

### Para criar e vincular uma nova branch local diretamente no servidor remoto
```bash
git push --set-upstream origin novabranch
```

### Para comparar as diferenças exatas entre uma branch local e a branch do servidor
```bash
git diff branch origin/novabranch
```

## <span id="git-tag"> 🔖 9. Definição de Marcos e Versões</span>

Tags funcionam como marcos permanentes no histórico (geralmente associados a Releases).

### Para criar uma tag anotada no commit atual (Marcar Versão)
```bash
git tag -a v2.0 -m "Versão 2.0"
```
### Para criar uma tag anotada retroativa apontando para um hash de commit específico
```bash
git tag -a v1.0 -m "Versao 1.0" 505a880bc020864bc69ed44599852cd99b679789
```

### Para listar todas as tags de versão existentes no repositório
```bash
git tag
```
### Para exibir os detalhes do commit e os metadados vinculados a uma tag
```bash
git show v2.0
```
### Para excluir uma tag do repositório local
```bash
git tag -d 'v2.0'
```

## <span id="gitignore">🙈 10. Desconsiderando Arquivos (`.gitignore`)</span>
Regras de sintaxe para impedir que arquivos locais específicos (como logs de servidores, diretórios de dependências ou arquivos temporários) entrem no rastreamento do Git.


| Padrão de Sintaxe | Comportamento e Impacto no Diretório |
| :--- | :--- |
| **`**/bin`** | **Coringa Duplo:** Ignora qualquer pasta ou arquivo chamado `bin`, independentemente da profundidade do diretório (Ex: `bin/exemplo.log`, `pasta/bin/log.txt`). |
| **`*/bin/debug.log`** | **Subpasta Relativa:** Ignora o arquivo `debug.log` desde que ele esteja exatamente um nível abaixo de qualquer diretório inicial (Ex: `build/bin/debug.log`). |
| **`*.log`** | **Extensão Global:** O asterisco funciona como coringa absoluto, ignorando qualquer arquivo do repositório que termine com a extensão `.log`. |
| **`!bin/*.log`** | **Negação/Exceção:** A exclamação inverte a regra. Significa que todos os arquivos `.log` dentro da pasta `bin` **devem ser rastreados**, agindo como uma exceção às travas globais. |
| **`teste`** | **Ocorrência Geral:** Sem a barra `/`, qualquer ocorrência com esse nome será considerada, seja um arquivo (`teste.log`) ou uma pasta (`teste/`). |
| **`node_modules/`** | **Diretório Restrito:** A barra `/` no final deixa explícito que se trata de uma pasta. Toda a árvore interna e subpastas desse diretório serão completamente ignoradas pelo Git. |

## <span id="git-pull-buscar">📥 11. Sincronização de Entrada (`git pull`)</span>

### Para atualizar a branch atual com o conteúdo do servidor remoto
```bash
git pull origin main

### Para puxar as alterações forçando um histórico linear (Rebase)
```bash
git pull --rebase origin feature/nome-projeto
```
O que faz: Em vez de criar um commit de merge para juntar o código do servidor ao seu, ele desfaz temporariamente seus commits locais, aplica os commits do servidor e depois recoloca os seus no topo, mantendo a linha do tempo limpa.

### Para baixar as atualizações de todas as branches remotas sem fundir o código (Fetch)
```bash
git fetch --all
```