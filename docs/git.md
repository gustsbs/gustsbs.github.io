# 🔀 Git - Comandos Rápidos

Anotações de configuração, ciclo de vida de arquivos, gerenciamento de branches e estratégias de integração de código.

## 📑 Índice
1. [Configurações Iniciais](#config-inicial)
2. [Monitoramento e Status](#monitoramento-e-status)
3. [Preparação e Snapshot](#git-add-commit)
4. [Histórico de Commits (git log)](#git-log)
5. [Navegação e Desfazer Código](#git-checkout-reset)
6. [Gerenciamento de Branches (git branch)](#git-branch)
7. [Guardando Trabalho Temporário (git stash)](#git-stash)
8. [Integração de Código: Merge vs Rebase](#git-merge-rebase)
9. [Repositórios Remotos e Sincronização (git remote & git push)](#git-push-remote)
10. [Definição de Marcos e Versões (git tag)](#git-tag)
11. [Desconsiderando Arquivos (.gitignore)](#desconsiderar-ignore)
12. [Sincronização de Entrada (git pull)](#git-pull-buscar)
13. [Submódulos (git submodule)](#git-submodule)
14. [Recuperação de Emergência (git reflog)](#rollback)
15. [Boas Práticas, Git Flow e Commits Semânticos](#boas-praticas)
---

## 1. <span id="config-inicial">⚙️ Configurações Iniciais</span>

Parametrização do ambiente global do Git no host de desenvolvimento. É o primeiro passo antes de qualquer trabalho sério com o repositório.

### 🔹 Inicializar um repositório local
Cria um novo repositório Git na pasta atual, já forçando a criação da branch principal com o nome padrão moderno `main` (evitando o termo legado `master`).
```bash
git init --initial-branch=main
```

### 🔹 Identificação global do usuário
Define nome e e-mail que serão gravados em todo commit feito nesta máquina. É obrigatório antes do primeiro commit.
```bash
git config --global user.name "Gustavo de Brito dos Santos"
git config --global user.email "gustbrito@email.com.br"
```

### 🔹 Editor de texto padrão
Define qual editor abrirá para escrever mensagens de commit, rebases interativos etc.
```bash
git config --global core.editor vim
```

### 🔹 Nome padrão de branch em novos repositórios
Configura globalmente qual será o nome da branch inicial toda vez que você rodar `git init` sem a flag `--initial-branch`.
```bash
git config --global init.defaultBranch main
```

### 🔹 Cache de credenciais
Salva as credenciais de acesso HTTPS em cache temporário na memória do host, evitando digitar usuário/senha (ou token) a cada `push`/`pull`.
```bash
git config --global credential.helper cache
```

### 🔹 Registrar e trocar o remoto `origin`
`remote add` mapeia um novo servidor remoto (por exemplo, sua instância do GitLab) sob o codinome `origin`, criando o canal de comunicação para futuros envios de código. `remote set-url` atualiza a URL de um remoto que já existe.
```bash
git remote add origin https://gitlab.gustbrito.br/sistemas/glpi.git
git remote set-url origin https://gitlab.gustbrito.br/sistemas/glpi.git
```

### 🔹 Gerando uma chave SSH para autenticação
Alternativa mais segura e prática ao HTTPS com token: gera um par de chaves para cadastrar a pública no GitLab/GitHub e nunca mais digitar senha.
```bash
ssh-keygen -t ed25519 -C "gustbrito@email.com.br"
cat ~/.ssh/id_ed25519.pub
```

### 🔹 Comportamento padrão do `git pull`
Essas três opções são mutuamente exclusivas e definem como o Git reconcilia seu histórico local com o remoto sempre que houver divergência. Sem `--global`, valem apenas para o repositório atual.

**Merge (padrão do Git):** cria automaticamente um commit de merge para unir os dois históricos.
```bash
git config --global pull.rebase false
```
**Rebase:** reaplica seus commits locais por cima dos commits recebidos, evitando merge commits e mantendo o histórico linear.
```bash
git config --global pull.rebase true
```
**Fast-forward only:** trava o `pull`. Só atualiza a máquina local se não houver divergência; se houver, aborta e exige resolução manual.
```bash
git config --global pull.ff only
```

### 🔹 Listar todas as configurações ativas
Mostra a soma das configurações de sistema, global e local, útil para depurar qual valor está realmente valendo.
```bash
git config --list --show-origin
```

### 🔹 Criando atalhos (aliases)
Reduz comandos longos e repetitivos do dia a dia a poucas letras.
```bash
git config --global alias.st status
git config --global alias.co checkout
git config --global alias.lg "log --oneline --graph --all"
```

## 2. <span id="monitoramento-e-status">🔍 Monitoramento e Status</span>

O Git categoriza os arquivos locais em 4 estados principais:

- **Untracked** — arquivo novo que ainda não está sendo rastreado pelo Git.
- **Unmodified** — arquivo já rastreado que não sofreu alterações.
- **Modified** — arquivo rastreado que foi modificado, mas ainda está fora da área de commit.
- **Staged** — arquivo pronto e selecionado para entrar no próximo commit.

### 🔹 Checar o status de rastreamento dos arquivos
```bash
git status
```

### 🔹 Status resumido
Versão compacta, uma linha por arquivo — ótima quando há muitas mudanças e você só quer uma visão rápida.
```bash
git status -s
```

### 🔹 Ver as diferenças exatas (diff)
Mostra linha a linha o que mudou. Sem argumentos, compara o *working directory* com a área de staging; com `--staged`, compara o que já está preparado com o último commit.
```bash
git diff
git diff --staged
```

## 3. <span id="git-add-commit">📥 Preparação e Snapshot (git add & git commit)</span>

Fluxo para mover arquivos modificados para a área de preparação (staging) e salvar a versão no histórico.

### 🔹 Adicionar arquivos à área de staging
Aceita um arquivo específico ou `.` para adicionar todas as modificações do diretório atual.
```bash
git add nomedoarquivo
git add .
```

### 🔹 Staging seletivo (patch mode)
Permite revisar e escolher, trecho por trecho (*hunk*), o que vai para o commit — útil para não misturar mudanças não relacionadas num mesmo commit.
```bash
git add -p
```

### 🔹 Remover arquivos da área de staging
Tira os arquivos da fila de commit sem descartar as alterações no diretório de trabalho.
```bash
git restore --staged nomedoarquivo
```

### 🔹 Criar um commit
Sem `-m`, abre o editor padrão para escrever a mensagem; com `-m`, aplica a mensagem direto no comando.
```bash
git commit
git commit -m "Adiciona validação de login no módulo de autenticação"
```

### 🔹 Commitar arquivos já rastreados sem passar pelo `git add`
Adiciona automaticamente ao staging apenas os arquivos que **já eram rastreados** e foram modificados (não pega arquivos novos/untracked).
```bash
git commit -a -m "Corrige tratamento de erro na API de faturamento"
```

### 🔹 Corrigir o último commit (amend)
Reaproveita o commit mais recente para incluir mudanças esquecidas ou corrigir a mensagem, em vez de criar um novo commit. **Nunca use em commits já enviados a uma branch compartilhada.**
```bash
git commit --amend -m "Nova mensagem corrigida"
```

### 🔹 Remover ou mover arquivos rastreados
`git rm` remove o arquivo do disco e do rastreamento; `git mv` renomeia/move mantendo o histórico associado.
```bash
git rm nomedoarquivo
git mv nomeantigo.txt nomenovo.txt
```

## 4. <span id="git-log">📜 Histórico de Commits (git log)</span>

Para inspecionar a linha do tempo e auditoria do repositório.

### 🔹 Log geral de commits
```bash
git log
```

### 🔹 Histórico com as linhas exatas de código modificadas
```bash
git log -p
```

### 🔹 Commits em uma única linha simplificada
```bash
git log --oneline
```

### 🔹 Limitar a quantidade de commits exibidos
```bash
git log -n 2
```

### 🔹 Filtrar por autor
```bash
git log --author="Gustavo de Brito dos Santos"
```

### 🔹 Filtrar por período de tempo
```bash
git log --after="1 week ago"
git log --before="2026-01-01"
```

### 🔹 Estatísticas de arquivos e linhas alteradas por commit
```bash
git log --stat
```

### 🔹 Histórico de branches em formato de gráfico
```bash
git log --graph --oneline --all
```

### 🔹 Isolar o histórico de um arquivo específico
```bash
git log nomedoarquivo
```

### 🔹 Ranking de contribuições por autor
Útil para times: mostra quantos commits cada pessoa fez no repositório.
```bash
git shortlog -sn
```

### 🔹 Inspecionar um commit específico
Exibe a mensagem, o autor e o diff completo de um commit pelo hash.
```bash
git show 505a880bc020864bc69ed44599852cd99b679789
```

### 🔹 Descobrir quem alterou cada linha (blame)
Essencial em investigação de incidentes: mostra o autor e o commit responsável por cada linha de um arquivo.
```bash
git blame nomedoarquivo
```

## 5. <span id="git-checkout-reset">⏪ Navegação e Desfazer Código</span>

### 🔹 Trocar de branch
`checkout` é o comando clássico; `switch` é a alternativa moderna e mais explícita, recomendada a partir do Git 2.23.
```bash
git checkout main
git switch main
```

### 🔹 Criar e trocar para uma nova branch em um único comando
```bash
git checkout -b nomedabranch/nomeprojeto
git switch -c nomedabranch/nomeprojeto
```

### 🔹 Navegar até um snapshot específico do histórico
Coloca o repositório em estado *detached HEAD* (fora de qualquer branch), útil para inspecionar um ponto exato do passado.
```bash
git checkout 11986923015fab393d4da32d8f8e4b2cc96d99b9
```

### 🔹 Trazer um commit específico de outra branch (cherry-pick)
Aplica na branch atual apenas um commit pontual de outra branch, sem trazer todo o histórico dela.
```bash
git cherry-pick 505a880bc020864bc69ed44599852cd99b679789
```

### 🔹 Desfazer commits (git reset)
As três variantes diferem no quanto desfazem:

| Modo | Efeito |
| :--- | :--- |
| `--soft` | Desfaz o commit, mas mantém tudo em staging (pronto para recommitar). |
| `--mixed` (padrão) | Desfaz o commit e tira do staging, mas preserva as alterações no diretório de trabalho. |
| `--hard` | Descarta o commit **e** todas as alterações locais. Irreversível pela interface normal do Git. |

```bash
git reset --soft HEAD~1
git reset --hard HEAD~1
```

### 🔹 Desfazer um commit já enviado ao remoto (revert)
Diferente do `reset`, o `revert` **não reescreve o histórico** — ele cria um novo commit que anula as alterações de um commit anterior. É a forma segura de desfazer algo em branches compartilhadas.
```bash
git revert 505a880bc020864bc69ed44599852cd99b679789
```

## 6. <span id="git-branch">🌿 Gerenciamento de Branches (git branch)</span>

Isolamento e controle do fluxo de ramificações do repositório.

### 🔹 Listar branches
Sem flag, mostra apenas as locais; com `-a`, inclui também as remotas (prefixadas com `remotes/origin/`).
```bash
git branch
git branch -a
```

### 🔹 Criar uma nova branch a partir do ponto atual
Apenas cria — não troca automaticamente para ela (para isso, use `checkout -b` ou `switch -c`, vistos na seção anterior).
```bash
git branch nomedabranch
```

### 🔹 Renomear a branch atual
```bash
git branch -m nome-novo
```

### 🔹 Excluir uma branch local
`-d` é seguro e recusa apagar branches com commits não mesclados; `-D` força a exclusão mesmo assim.
```bash
git branch -d feature/novoarquivo
git branch -D feature/novoarquivo
```

### 🔹 Ver branches já mescladas ou pendentes
Ajuda a identificar branches "mortas" que já podem ser limpas com segurança.
```bash
git branch --merged
git branch --no-merged
```

## 7. <span id="git-stash">🎒 Guardando Trabalho Temporário (git stash)</span>

Guarda mudanças ainda não commitadas numa "gaveta" temporária, deixando o diretório de trabalho limpo — ideal para trocar de branch rapidamente sem precisar commitar algo incompleto.

### 🔹 Guardar as alterações atuais
```bash
git stash
```

### 🔹 Guardar incluindo arquivos novos (untracked)
Por padrão, o stash ignora arquivos que ainda não foram rastreados. A flag `-u` inclui esses arquivos também.
```bash
git stash push -u -m "wip: ajuste no middleware de auth"
```

### 🔹 Listar tudo o que está guardado
```bash
git stash list
```

### 🔹 Recuperar o último stash
`pop` aplica e remove da lista; `apply` aplica mas mantém o stash guardado (útil para reaplicar em mais de uma branch).
```bash
git stash pop
git stash apply
```

### 🔹 Ver o conteúdo de um stash sem aplicá-lo
```bash
git stash show -p stash@{0}
```

### 🔹 Descartar um stash
```bash
git stash drop stash@{0}
```

## 8. <span id="git-merge-rebase">🚀 Integração de Código: Git Merge vs Git Rebase</span>

### 1️⃣ git merge — Combina o histórico de dois ramos criando um novo commit de merge.
```bash
git fetch origin
git merge nomedabranch
# ou
git merge origin/main
```
**O que faz:** cria um commit de junção na branch atual, interligando os dois históricos de forma paralela (não linear).

**Prós:** preserva o histórico real e completo de cada ramificação — ideal para auditorias em projetos de equipes grandes.

**Contras:** o log visual do histórico pode ficar poluído com commits automáticos de merge.

### 2️⃣ git rebase — Reaplica os commits da branch atual sobre o topo de outra branch.
```bash
git rebase nomedabranch
```
**O que faz:** remove temporariamente seus commits locais, traz as atualizações da outra branch e reaplica suas alterações por cima, gerando uma linha do tempo única e linear.

**Prós:** produz um histórico extremamente limpo e fácil de ler.

**Contras:** reescreve o histórico. Nunca deve ser usado em branches públicas/compartilhadas (risco de quebrar os clones locais de outras pessoas do time).

### 🔹 Rebase interativo (squash/reorganização de commits)
Permite reescrever, combinar (`squash`), reordenar ou editar mensagens dos últimos N commits antes de abrir um Merge Request — prática comum para limpar o histórico de uma feature branch.
```bash
git rebase -i HEAD~3
```

### 🔹 Resolvendo conflitos
Quando o Git não consegue unir as mudanças automaticamente (em `merge`, `rebase` ou `cherry-pick`), ele marca os trechos conflitantes no arquivo com `<<<<<<<`, `=======` e `>>>>>>>`. O fluxo de resolução é:
```bash
# 1. Edite os arquivos conflitantes e remova os marcadores
# 2. Marque como resolvido
git add nomedoarquivo

# 3a. Se estava em um merge, finalize com:
git commit

# 3b. Se estava em um rebase, continue com:
git rebase --continue
```

### 🔹 Abortando uma operação em andamento
Cancela o processo e retorna o repositório ao estado anterior — útil quando o conflito é grande demais para resolver naquele momento.
```bash
git merge --abort
git rebase --abort
git cherry-pick --abort
```

## 9. <span id="git-push-remote">🌐 Repositórios Remotos e Sincronização</span>

### 🔹 Clonar um repositório remoto
```bash
git clone git@github.com:MeuRepo/projetoxyz.git
```

### 🔹 Vincular um repositório remoto ao diretório local
```bash
git remote add origin git@github.com:gustbrito/asterisk.git
```

### 🔹 Listar e remover remotos
```bash
git remote -v
git remote remove origin
```

### 🔹 Enviar pela primeira vez, definindo a branch padrão no servidor
```bash
git push -u origin main
```

### 🔹 Criar e vincular uma nova branch local diretamente no servidor remoto
```bash
git push --set-upstream origin novabranch
```

### 🔹 Excluir uma branch remota
```bash
git push origin --delete nomedabranch
```

### 🔹 Forçar push com segurança
`--force` sobrescreve o remoto às cegas — arriscado se alguém mais fez push nesse meio tempo. `--force-with-lease` só sobrescreve se ninguém mais alterou a branch remota desde o seu último `fetch`, evitando perder trabalho alheio.
```bash
git push --force-with-lease origin nomedabranch
```

### 🔹 Abrir um Merge Request direto pelo push (GitLab Push Options)
O GitLab permite criar e configurar uma Merge Request sem sair do terminal, usando `-o`/`--push-option` no próprio push.
```bash
git push -o merge_request.create -o merge_request.target=main origin nomedabranch
```

### 🔹 Comparar diferenças entre branch local e remota
```bash
git diff main origin/main
```

### 🔹 Limpar referências de branches remotas já apagadas
Sincroniza a lista local de branches remotas, removendo referências a branches que já foram deletadas no servidor.
```bash
git fetch --prune
```

## 10. <span id="git-tag">🔖 Definição de Marcos e Versões (git tag)</span>

Tags funcionam como marcos permanentes no histórico, geralmente associados a Releases.

### 🔹 Criar uma tag anotada no commit atual
```bash
git tag -a v2.0 -m "Versão 2.0"
```

### 🔹 Criar uma tag retroativa apontando para um commit específico
```bash
git tag -a v1.0 -m "Versão 1.0" 505a880bc020864bc69ed44599852cd99b679789
```

### 🔹 Listar todas as tags do repositório
```bash
git tag
```

### 🔹 Exibir os detalhes de uma tag
```bash
git show v2.0
```

### 🔹 Enviar tags para o servidor remoto
Por padrão, `git push` **não** envia tags — é preciso enviá-las explicitamente.
```bash
git push origin v2.0
git push origin --tags
```

### 🔹 Voltar para o código de uma tag específica
Como uma tag aponta para um commit fixo, o checkout direto entra em *detached HEAD*. Para trabalhar em cima dela, crie uma branch a partir dela.
```bash
git checkout tags/v1.0 -b hotfix/v1.0.1
```

### 🔹 Excluir uma tag
```bash
git tag -d v2.0
```

## 11. <span id="desconsiderar-ignore">🙈 Desconsiderando Arquivos (`.gitignore`)</span>
Regras de sintaxe para impedir que arquivos locais específicos (como logs de servidores, diretórios de dependências ou arquivos temporários) entrem no rastreamento do Git.

| Padrão de Sintaxe | Comportamento e Impacto no Diretório |
| :--- | :--- |
| **`**/bin`** | **Coringa duplo:** ignora qualquer pasta ou arquivo chamado `bin`, independentemente da profundidade do diretório (ex.: `bin/exemplo.log`, `pasta/bin/log.txt`). |
| **`*/bin/debug.log`** | **Subpasta relativa:** ignora o arquivo `debug.log` desde que ele esteja exatamente um nível abaixo de qualquer diretório inicial (ex.: `build/bin/debug.log`). |
| **`*.log`** | **Extensão global:** o asterisco funciona como coringa absoluto, ignorando qualquer arquivo do repositório que termine com a extensão `.log`. |
| **`!bin/*.log`** | **Negação/exceção:** a exclamação inverte a regra. Significa que todos os arquivos `.log` dentro da pasta `bin` **devem ser rastreados**, agindo como exceção às travas globais. |
| **`teste`** | **Ocorrência geral:** sem a barra `/`, qualquer ocorrência com esse nome será ignorada, seja um arquivo (`teste.log`) ou uma pasta (`teste/`). |
| **`node_modules/`** | **Diretório restrito:** a barra `/` no final deixa explícito que se trata de uma pasta. Toda a árvore interna e subpastas desse diretório serão completamente ignoradas pelo Git. |

### 🔹 Aplicar o `.gitignore` retroativamente
Criar ou editar o `.gitignore` não afeta arquivos que **já estão sendo rastreados**. Para "esquecê-los" sem apagá-los do disco:
```bash
git rm -r --cached .
git add .
git commit -m "Aplica novas regras do .gitignore"
```

## 12. <span id="git-pull-buscar">📥 Sincronização de Entrada (`git pull`)</span>

### 🔹 Atualizar a branch atual com o conteúdo do servidor remoto
```bash
git pull origin main
```

### 🔹 Puxar alterações forçando um histórico linear (rebase)
Em vez de criar um commit de merge para juntar o código do servidor ao seu, o Git desfaz temporariamente seus commits locais, aplica os commits do servidor e depois recoloca os seus por cima, mantendo a linha do tempo limpa.
```bash
git pull --rebase origin feature/nome-projeto
```

### 🔹 Baixar atualizações sem fundir o código (fetch)
Traz as referências de todas as branches remotas para o repositório local, sem alterar seus arquivos de trabalho — seguro para rodar a qualquer momento.
```bash
git fetch --all
```

### 🔹 Descartar tudo o que é local e espelhar fielmente o servidor remoto

**1.** Busque as referências mais atuais do servidor:
```bash
git fetch origin
```

**2.** Confirme a branch local — neste exemplo, pretendemos espelhar a `main` do servidor:
```bash
git status
```

**3.** Force o reset da branch local para coincidir exatamente com o remoto:
```bash
git reset --hard origin/main
```

**4.** Remova arquivos e diretórios não rastreados.
O passo anterior reseta apenas os arquivos que o Git já conhece. Se você criou arquivos novos (logs, backups, scripts de teste) que nunca foram commitados, eles permanecem no disco. Para removê-los também:

⚠️ **Atenção: o comando abaixo apaga permanentemente arquivos e pastas locais que não estão no repositório remoto.**
```bash
git clean -fd
```

## 13. <span id="git-submodule">🧩 Submódulos (git submodule)</span>

Permite incluir um repositório Git dentro de outro, mantendo o histórico e versionamento de cada um independentes — comum quando um projeto depende de uma biblioteca interna versionada à parte.

### 🔹 Adicionar um submódulo
```bash
git submodule add https://gitlab.gustbrito.br/libs/shared-utils.git libs/shared-utils
```

### 🔹 Clonar um repositório que já contém submódulos
```bash
git clone --recurse-submodules git@github.com:MeuRepo/projetoxyz.git
```

### 🔹 Inicializar/atualizar submódulos em um clone já existente
```bash
git submodule update --init --recursive
```

## 14. <span id="rollback">🪽 Recuperação de Emergência</span>

### 🔹 Reflog — a rede de segurança do Git
O `reflog` registra **todo** movimento do `HEAD` na sua máquina (commits, resets, checkouts, rebases), mesmo aqueles que já não aparecem mais no `git log`. É o primeiro comando a rodar quando algo parece "perdido" após um `reset --hard` ou rebase malfeito.
```bash
git reflog
git reset --hard HEAD@{2}
```

### 🔹 Resetar o repositório local para um commit específico
```bash
git reset --hard 47ce0f9d9c04a5f8b72be653b908c781d450561f
```

### 🔹 Sincronizar o servidor remoto com o estado local corrigido
Se você já tinha dado push de commits incorretos e, após corrigir localmente, precisa forçar o servidor (GitLab/GitHub) a refletir exatamente o seu histórico local:

⚠️ **Cuidado: isso reescreve o histórico remoto e pode impactar quem já deu pull da branch anterior.** Prefira `--force-with-lease` (seção 9) sempre que possível.
```bash
git push origin <nome-da-sua-branch> --force
```

## 15. <span id="boas-praticas">🧭 Boas Práticas, Git Flow e Commits Semânticos</span>

Enquanto as seções anteriores cobrem *o que* cada comando faz, esta seção documenta *como e quando* usá-los em conjunto: a estratégia de branches adotada nos projetos (um Git Flow simplificado) e o padrão de mensagens de commit (Conventional Commits). Ter esse combinado documentado evita branches soltas, mensagens de commit inúteis e facilita automações (changelog, versionamento semântico, CI/CD).

### 🔹 Git Flow simplificado
A estratégia se baseia em duas branches permanentes e um conjunto de branches temporárias e descartáveis, criadas para uma tarefa específica e apagadas após o merge.

| Branch | Papel |
| :--- | :--- |
| **`main`** | Branch estável. Representa a versão oficial, publicável, do projeto. Só recebe código já revisado e consolidado (via merge de `release/` ou `hotfix/`). |
| **`develop`** | Branch de integração. Concentra o trabalho já revisado das branches temporárias antes de seguir para uma release. |

### 🔹 Tipos de branch temporária
Cada branch temporária nasce de `develop` (ou de `main`, no caso de `hotfix/`) e existe só até ser mesclada de volta.

| Prefixo | Quando usar |
| :--- | :--- |
| `feature/` | Novas funcionalidades ou novos documentos. |
| `fix/` | Correção de erro que não é urgente a ponto de ir direto para `main`. |
| `docs/` | Alterações exclusivamente documentais. |
| `refactor/` | Reorganização de código ou estrutura, sem alterar comportamento. |
| `test/` | Criação ou ajuste de testes. |
| `chore/` | Tarefas auxiliares: infraestrutura, build, organização de repositório. |
| `hotfix/` | Correção urgente, criada diretamente a partir de `main` (produção). |
| `release/` | Preparação de uma versão/entrega, criada a partir de `develop`. |

### 🔹 Padrão de nomenclatura de branches
Nomes curtos, descritivos, em letras minúsculas, sem espaços e sem acentos, seguindo o formato:
```text
<tipo>/<descricao-curta>
```
Exemplos:
```text
feature/pop-configuracao-dns-proxmox
fix/corrigir-comando-zfs
docs/atualizar-guia-instalacao-proxmox
refactor/reorganizar-estrutura-pops
chore/ajustar-readme
release/v1.0.0
hotfix/corrigir-instrucao-producao
```

Quando existir uma demanda, chamado, issue ou identificador formal associado ao trabalho, inclua-o no nome da branch — isso cria rastreabilidade direta entre o código e o sistema de gestão (Jira, GLPI, etc.):
```text
feature/CSI-1234-pop-configuracao-dns-proxmox
fix/INC-2026-077-corrigir-timeout-ldap
docs/RDM-2026-014-atualizar-procedimento-deploy
```

### 🔹 Fluxo básico de trabalho

**1.** Atualize a branch base antes de começar, garantindo que você parte do ponto mais recente:
```bash
git checkout develop
git pull
```

**2.** Crie a branch de trabalho a partir dela:
```bash
git checkout -b docs/pop-configuracao-dns-proxmox
```

**3.** Faça alterações pequenas e focadas — uma branch, um objetivo. Evite misturar assuntos não relacionados no mesmo trabalho.

**4.** Registre commits semânticos (ver convenção abaixo) descrevendo cada mudança de forma clara:
```bash
git commit -m "docs(pop): adicionar POP para configuração de DNS no Proxmox"
```

**5.** Envie a branch ao repositório remoto:
```bash
git push -u origin docs/pop-configuracao-dns-proxmox
```

**6.** Abra um Merge Request/Pull Request para revisão por outra pessoa do time.

**7.** Após aprovação, faça o merge para `develop`.

### 🔹 Fechando uma release
Quando o conjunto de mudanças em `develop` estiver pronto para ser entregue, formalize uma branch de release para estabilização final:
```bash
git checkout develop
git pull
git checkout -b release/v1.0.0
```

Após validar a release (testes, revisão, ajustes finais), integre-a em `main` e marque a versão com uma tag:
```bash
git checkout main
git merge release/v1.0.0
git tag v1.0.0
git push origin main --tags
```
> ℹ️ `--tags` envia **todas** as tags locais ainda não publicadas, não apenas a `v1.0.0` — veja mais detalhes na [seção de Tags](#git-tag).

Por fim, integre a release de volta em `develop` também, para garantir que qualquer ajuste feito durante a estabilização não se perca:
```bash
git checkout develop
git merge release/v1.0.0
git push origin develop
```

Depois do merge, apague a branch de release e demais branches temporárias já incorporadas — mantém o repositório limpo (ver comandos de exclusão de branch na [seção 6](#git-branch)).

### 🔹 Commits Semânticos (Conventional Commits)
Um padrão de prefixos torna o histórico legível por humanos e por máquinas — habilita geração automática de changelog, versionamento semântico (SemVer) e regras de CI baseadas no tipo de mudança.

| Prefixo | Significado |
| :--- | :--- |
| `feat` | Nova funcionalidade. |
| `fix` | Correção de bug. |
| `docs` | Alteração apenas na documentação. |
| `refactor` | Refatoração de código sem alterar funcionalidade. |
| `chore` | Tarefas auxiliares: infraestrutura, build, dependências, etc. |
| `test` | Adição ou ajuste de testes. |
| `style` | Ajustes visuais ou de formatação, sem alteração funcional. |
| `ci` | Ajustes em integração contínua, pipelines ou automações. |
| `build` | Alterações no processo de build, empacotamento ou dependências. |
| `perf` | Melhoria de desempenho. |
| `revert` | Reversão de uma alteração anterior. |

Formato geral da mensagem:
```text
<tipo>(<escopo opcional>): <descrição curta no imperativo>
```
Exemplo básico:
```bash
git commit -m "docs(pop): adicionar POP para configuração de DNS no Proxmox"
```

O escopo (entre parênteses) é opcional, mas ajuda a indicar rapidamente qual área do projeto foi afetada:
```bash
git commit -m "fix(proxmox): corrigir instrução de configuração de bridge"
git commit -m "docs(zfs): atualizar procedimento de criação de pool"
git commit -m "chore(repo): reorganizar estrutura de diretórios"
```

### 🔹 Boas práticas para commits e revisão final
- ✅ Prefira commits pequenos e frequentes — cada um representando uma unidade lógica de mudança, mais fácil de revisar e de reverter isoladamente (`git revert`, seção 5) se necessário.
- ✅ Cada branch deve tratar um objetivo claro e delimitado; evite misturar alterações não relacionadas na mesma branch ou no mesmo commit.
- ✅ Sempre atualize o campo **"Última atualização"** do documento, quando aplicável, antes de abrir o Merge Request.
- ✅ Antes de abrir o Merge Request/Pull Request, revise ortografia, comandos, caminhos e datas — e confira se os exemplos ainda são consistentes com o restante do repositório.
- ✅ Ao concluir uma entrega estável, prefira criar uma tag de versão (seção 10) a deixar apenas o commit "solto" no histórico — isso facilita rollback e auditoria futura.
- ✅ Apague branches temporárias após o merge; branches vivas e esquecidas divergem do código real e confundem o time.
