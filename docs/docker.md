# 🐳 Guia de Comandos Docker

Este guia centraliza os principais comandos do Docker utilizados no dia a dia para gerenciamento de containers, imagens, redes, volumes, monitoramento e builds.

## 📑 Índice

1. [Execução de Containers (docker run)](#execucao-de-containers)
2. [Gerenciamento e Ciclo de Vida](#gerenciamento-de-containers)
3. [Administração e Inspeção](#administracao-e-inspecao)
4. [Monitoramento e Recursos](#monitoramento-e-recursos)
5. [Logs de Containers](#logs-de-containers)
6. [Transferência de Arquivos](#transferencia-de-arquivos)
7. [Gerenciamento e Inspeção de Imagens](#gerenciamento-e-inspecao-de-imagens)
8. [Build de Imagens (Dockerfile)](#build-de-imagens)
9. [Persistência de Dados (Volumes & Mounts)](#persistencia-de-dados)
10. [Redes (Networking)](#redes-docker)
11. [Docker Compose](#docker-compose)
12. [Limpeza e Manutenção do Sistema](#limpeza-sistema)
---

## 1. <span id="execucao-de-containers">🚀 Execução de Containers (docker run)</span>

`docker run` é o comando mais usado do Docker: cria e inicia um container a partir de uma imagem em um único passo. As flags abaixo cobrem os cenários mais comuns do dia a dia.

### 🔹 Executar um container básico
Cria e inicia um container a partir de uma imagem. Se a imagem não existir localmente, o Docker faz o download (`pull`) automaticamente antes de rodar.
```bash
docker run nginx
```

### 🔹 Executar em segundo plano (detached)
Mantém o container rodando em background, devolvendo o terminal imediatamente. É o modo padrão para serviços de longa duração (bancos, APIs, proxies).
```bash
docker run -d nginx
```

### 🔹 Nomear o container
Sem `--name`, o Docker gera um nome aleatório. Nomear facilita referenciar o container em outros comandos (`stop`, `logs`, `exec`).
```bash
docker run -d --name meu-nginx nginx
```

### 🔹 Mapear portas (host:container)
Expõe uma porta do container para a rede do host, permitindo acesso externo ao serviço.
```bash
docker run -d --name meu-nginx -p 8080:80 nginx
```

### 🔹 Definir variáveis de ambiente
Injeta variáveis de ambiente no processo do container — comum para configurar credenciais, URLs de conexão, flags de comportamento etc.
```bash
docker run -d --name mysql -e MYSQL_ROOT_PASSWORD=senha123 mysql
```

### 🔹 Executar em modo interativo
`-i` mantém o STDIN aberto e `-t` aloca um pseudo-terminal — combinação usada para abrir um shell interativo assim que o container sobe.
```bash
docker run -it ubuntu /bin/bash
```

### 🔹 Remover o container automaticamente ao parar
Útil para execuções pontuais/descartáveis (testes, scripts), evitando acumular containers parados no host.
```bash
docker run --rm -it ubuntu /bin/bash
```

### 🔹 Definir política de reinício
Determina o comportamento do container em caso de falha ou reboot do host. `unless-stopped` reinicia sempre, exceto quando parado manualmente — ideal para serviços em produção.
```bash
docker run -d --name mysql --restart unless-stopped mysql
```

### 🔹 Limitar recursos de CPU e memória
Evita que um único container consuma todos os recursos do host — essencial em ambientes compartilhados.
```bash
docker run -d --name app --memory="512m" --cpus="1.5" minhaimagem
```

## 2. <span id="gerenciamento-de-containers">📦 Gerenciamento e Ciclo de Vida</span>

Comandos para controlar o estado de containers já existentes: iniciar, parar, reiniciar, pausar e remover.

### 🔹 Iniciar um container parado
```bash
docker start mysql
```

### 🔹 Parar um container em execução
Envia o sinal `SIGTERM`, dando tempo para o processo encerrar de forma limpa antes de ser finalizado.
```bash
docker stop mysql
```

### 🔹 Forçar o encerramento imediato
Envia `SIGKILL` diretamente, sem aguardar o encerramento gracioso — use quando o container está travado e `stop` não responde.
```bash
docker kill mysql
```

### 🔹 Reiniciar um container
Equivale a um `stop` seguido de `start` em um único comando — útil após alterar variáveis de ambiente ou aplicar correções pontuais.
```bash
docker restart mysql
```

### 🔹 Pausar e retomar um container
`pause` congela todos os processos do container (via cgroups freezer) sem finalizá-los; `unpause` retoma de onde parou.
```bash
docker pause mysql
docker unpause mysql
```

### 🔹 Renomear um container
```bash
docker rename mysql mysql-producao
```

### 🔹 Remover um container
Exclui permanentemente um container parado do sistema. Use `-f` para forçar a remoção mesmo que ele ainda esteja em execução.
```bash
docker rm mysql
docker rm -f mysql
```

### 🔹 Remover todos os containers do host
Lista os IDs de todos os containers (`-qa`) e os remove forçadamente (`-f`), limpando o ambiente por completo.
```bash
docker container rm -f $(docker container ls -qa)
```

## 3. <span id="administracao-e-inspecao">🔎 Administração e Inspeção</span>

### 🔹 Listar containers
Sem flags, mostra apenas os containers em execução; `-a` inclui também os parados.
```bash
docker ps
docker ps -a
```

### 🔹 Executar comandos em um container ativo
Abre um terminal interativo dentro de um container que já está rodando — o jeito mais comum de "entrar" em um container para depuração.
```bash
docker exec -it nginx /bin/bash
```

### 🔹 Inspecionar detalhes de um container
Retorna um JSON completo com toda a configuração do container: IP, volumes montados, variáveis de ambiente, políticas de restart, etc.
```bash
docker inspect nomedocontainer
```

### 🔹 Ver as diferenças de arquivos em relação à imagem original
Lista arquivos criados, modificados ou removidos no sistema de arquivos do container desde que ele foi iniciado.
```bash
docker diff nomedocontainer
```

### 🔹 Ver os processos rodando dentro do container
Equivalente a um `ps` executado de fora, sem precisar entrar no container.
```bash
docker top nomedocontainer
```

## 4. <span id="monitoramento-e-recursos">📊 Monitoramento e Recursos</span>

### 🔹 Uso de CPU, memória e rede em tempo real
Exibe um painel ao vivo (similar ao `top`) com o consumo de recursos de todos os containers em execução — o primeiro comando a rodar ao investigar lentidão.
```bash
docker stats
```

### 🔹 Uso de recursos de um container específico, sem atualização contínua
```bash
docker stats --no-stream nomedocontainer
```

### 🔹 Acompanhar eventos do Docker em tempo real
Transmite em tempo real eventos do daemon (`start`, `stop`, `die`, `kill`, `health_status`, etc.) — útil para correlacionar incidentes com ações tomadas no host.
```bash
docker events
```

### 🔹 Verificar o espaço em disco usado pelo Docker
Mostra quanto espaço containers, imagens, volumes e cache de build estão consumindo no host.
```bash
docker system df
```

### 🔹 Checar o status de saúde de um container (healthcheck)
Se a imagem define um `HEALTHCHECK`, este comando mostra o status atual (`healthy`, `unhealthy`, `starting`).
```bash
docker inspect --format='{{.State.Health.Status}}' nomedocontainer
```

## 5. <span id="logs-de-containers">📜 Logs de Containers</span>

### 🔹 Ver os logs de um container
Exibe toda a saída padrão (stdout/stderr) já registrada pelo container desde que ele foi iniciado.
```bash
docker logs nomedocontainer
```

### 🔹 Acompanhar os logs em tempo real
Equivalente ao `tail -f`, mantém o terminal exibindo novas linhas conforme são geradas — o comando mais usado para depurar um serviço rodando.
```bash
docker logs -f nomedocontainer
```

### 🔹 Ver apenas as últimas N linhas
```bash
docker logs --tail 100 nomedocontainer
```

### 🔹 Filtrar logs por período
Mostra apenas logs gerados após um determinado horário — útil para isolar o intervalo exato de um incidente.
```bash
docker logs --since 30m nomedocontainer
docker logs --since "2026-07-17T10:00:00" nomedocontainer
```

### 🔹 Exibir timestamps em cada linha de log
Por padrão o Docker não mostra o horário de cada linha; essa flag adiciona o timestamp, essencial para correlacionar com outros sistemas de observabilidade.
```bash
docker logs -t nomedocontainer
```

## 6. <span id="transferencia-de-arquivos">📁 Transferência de Arquivos</span>

### 🔹 Copiar arquivos do host para o container
```bash
docker cp ./arquivo.conf nomedocontainer:/etc/app/arquivo.conf
```

### 🔹 Copiar arquivos do container para o host
```bash
docker cp nomedocontainer:/var/log/app.log ./app.log
```

### 🔹 Exportar o sistema de arquivos de um container
Gera um tarball com o filesystem completo do container (sem histórico de camadas nem metadados de imagem) — usado para migrar o estado de um container para outro host.
```bash
docker export nomedocontainer -o container.tar
```

### 🔹 Importar um sistema de arquivos como nova imagem
Reconstrói uma imagem a partir de um tarball gerado pelo `export`.
```bash
docker import container.tar novaimagem:latest
```

### 🔹 Salvar uma imagem em arquivo (com histórico de camadas)
Diferente do `export`, o `save` preserva as camadas e metadados da imagem — ideal para transportar imagens entre hosts sem precisar de um registry.
```bash
docker save -o nomedaimagem.tar nomedaimagem:tag
```

### 🔹 Carregar uma imagem a partir de um arquivo
```bash
docker load -i nomedaimagem.tar
```

## 7. <span id="gerenciamento-e-inspecao-de-imagens">🖼️ Gerenciamento e Inspeção de Imagens</span>

### 🔹 Listar imagens locais
Exibe todas as imagens Docker baixadas ou construídas que estão disponíveis na máquina.
```bash
docker image ls
```

### 🔹 Baixar uma imagem de um registry
```bash
docker pull nginx:latest
```

### 🔹 Autenticar e enviar uma imagem para um registry
Requer login prévio e que a imagem esteja marcada (`tag`) com o repositório de destino.
```bash
docker login registry.gustbrito.br
docker push registry.gustbrito.br/sistemas/minhaimagem:v1.0
```

### 🔹 Marcar (tag) uma imagem
Cria um novo nome/referência para uma imagem existente — necessário antes de enviá-la a um registry específico.
```bash
docker tag minhaimagem:latest registry.gustbrito.br/sistemas/minhaimagem:v1.0
```

### 🔹 Inspecionar detalhes técnicos da imagem
Retorna um JSON detalhado com as especificações técnicas da imagem: camadas, variáveis, portas expostas, etc.
```bash
docker image inspect nomedaimagem
```

### 🔹 Ver o histórico de construção da imagem
Mostra as camadas estruturais da imagem e os comandos que foram executados para criá-la.
```bash
docker image history nomedaimagem
```

### 🔹 Criar uma imagem a partir de um container modificado
Gera uma nova imagem local a partir de alterações feitas manualmente dentro de um container em execução — útil para depuração pontual, mas evite depender disso em produção (prefira sempre um Dockerfile versionado).
```bash
docker commit id_do_container nome_da_imagem
docker commit c3f279d17e0a novaimagem/testimage:version3
```

### 🔹 Remover uma imagem
```bash
docker rmi nomedaimagem
```

### 🔹 Remover imagens não utilizadas
Remove do host todas as imagens que não estão associadas a nenhum container ativo ou parado, liberando espaço em disco.
```bash
docker image prune
```

## 8. <span id="build-de-imagens">🛠️ Build de Imagens (Dockerfile)</span>

### 🔹 Compilar uma imagem a partir de um Dockerfile
Compila uma nova imagem apontando explicitamente para um arquivo (`-f`). O ponto (`.`) no final indica o contexto de build (diretório enviado ao daemon).
```bash
docker build -t nome_da_imagem -f Dockerfile .
```

### 🔹 Build sem cache
Força o Docker a reexecutar todas as etapas do Dockerfile do zero, sem reaproveitar o cache de builds anteriores — útil para garantir que uma dependência realmente foi atualizada.
```bash
docker build -t nome_da_imagem . --no-cache
```

### 🔹 Injetar variáveis em tempo de build
Passa valores para instruções `ARG` definidas no Dockerfile — diferente das variáveis de ambiente do `run`, essas só existem durante o build.
```bash
docker build -t nome_da_imagem --build-arg VAR_TEXTO=teste .
```

### 🔹 Gerar múltiplas tags no mesmo build
```bash
docker build -t nome_da_imagem:latest -t nome_da_imagem:v1.0 .
```

### 🔹 Buildar para uma plataforma específica
Relevante ao construir imagens em máquinas Apple Silicon (ARM) destinadas a servidores x86, ou vice-versa.
```bash
docker build --platform linux/amd64 -t nome_da_imagem .
```

## 9. <span id="persistencia-de-dados">💾 Persistência de Dados (Volumes & Mounts)</span>

### 🔹 Criar um volume
Cria um volume gerenciado pelo Docker para persistência de dados isolada do ciclo de vida do container.
```bash
docker volume create nomedovolume
```

### 🔹 Listar e inspecionar volumes
```bash
docker volume ls
docker volume inspect nomedovolume
```

### 🔹 Montar um volume em um container
```bash
docker run -d --name mysql -v nomedovolume:/var/lib/mysql mysql
```

### 🔹 Montar um diretório do host (bind mount)
Diferente do volume gerenciado, o bind mount vincula diretamente um caminho do host ao container — útil em desenvolvimento, para refletir mudanças de código em tempo real.
```bash
docker run -d --name app -v /caminho/no/host:/app minhaimagem
```

### 🔹 Remover um volume
Exclui permanentemente um volume nomeado do host (só funciona se o volume não estiver em uso).
```bash
docker volume rm nomedovolume
```

### 🔹 Remover volumes não utilizados
```bash
docker volume prune
```

## 10. <span id="redes-docker">🌐 Redes (Networking)</span>

Por padrão, containers na mesma rede Docker conseguem se comunicar entre si pelo nome — a base de como serviços conversam em um ambiente com múltiplos containers.

### 🔹 Listar redes
```bash
docker network ls
```

### 🔹 Criar uma rede
Cria uma rede isolada do tipo `bridge`, permitindo que containers conectados a ela se resolvam pelo nome via DNS interno do Docker.
```bash
docker network create minha-rede
```

### 🔹 Conectar um container a uma rede na criação
```bash
docker run -d --name app --network minha-rede minhaimagem
```

### 🔹 Conectar/desconectar um container já em execução
```bash
docker network connect minha-rede nomedocontainer
docker network disconnect minha-rede nomedocontainer
```

### 🔹 Inspecionar uma rede
Mostra quais containers estão conectados, faixa de IPs, gateway e driver utilizado.
```bash
docker network inspect minha-rede
```

### 🔹 Remover redes não utilizadas
```bash
docker network prune
```

## 11. <span id="docker-compose">🧱 Docker Compose</span>

Orquestra múltiplos containers (serviços, redes e volumes) descritos em um único arquivo `docker-compose.yml` — o padrão para rodar aplicações com mais de um container em desenvolvimento e em hosts únicos de produção.

### 🔹 Subir todos os serviços
Cria e inicia todos os containers definidos no `docker-compose.yml` em segundo plano.
```bash
docker compose up -d
```

### 🔹 Reconstruir as imagens antes de subir
```bash
docker compose up -d --build
```

### 🔹 Parar e remover os serviços
Remove containers e redes criados pelo `up`; `-v` remove também os volumes associados.
```bash
docker compose down
docker compose down -v
```

### 🔹 Ver logs de todos os serviços
```bash
docker compose logs -f
```

### 🔹 Listar os serviços em execução
```bash
docker compose ps
```

### 🔹 Executar um comando em um serviço específico
```bash
docker compose exec app /bin/bash
```

### 🔹 Reiniciar um único serviço
```bash
docker compose restart app
```

## 12. <span id="limpeza-sistema">🧹 Limpeza e Manutenção do Sistema</span>

### 🔹 Limpeza geral (containers, redes e imagens não utilizadas)
Remove containers parados, redes não utilizadas e imagens *dangling* — não afeta volumes por padrão.
```bash
docker system prune
```

### 🔹 Limpeza completa, incluindo imagens não referenciadas e volumes
⚠️ **Atenção:** `-a` remove também imagens sem container associado (não apenas as *dangling*), e `--volumes` apaga volumes não utilizados — dados não referenciados serão perdidos permanentemente.
```bash
docker system prune -a --volumes
```

### 🔹 Ver informações gerais do host Docker
Exibe versão, driver de storage, número de containers/imagens e recursos totais do daemon — útil para um diagnóstico inicial do host.
```bash
docker info
```
