# 🐳 Guia de Comandos Docker

Este guia centraliza os principais comandos do Docker utilizados no dia a dia para gerenciamento de containers, imagens, volumes e builds.

1. [Gerenciamento de Containers](#gerenciamento-de-containers)
2. [Gerenciamento e Inspeção de Imagens](gerencimento-e-expansao-de-imagens)
3. [Build de Imagens (Dockerfile)](#build-de-imagens)



## 1. <span id=#gerenciamento-de-containers> 📦 Gerenciamento de Containers</span>

### Listar todos os containers
Exibe uma lista de todos os containers no host, incluindo os que estão em execução e os que já foram parados.
```bash
docker container ls -a
```
### Iniciar um container que já foi criado anteriormente mas encontra-se parado.
```bash
docker start mysql
```
### Parar um container em execução
Envia um sinal amigável para interromper a execução de um container de forma limpa.
```bash
docker stop mysql
```
### Remover um container
Exclui permanentemente um container parado do sistema.
```bash
docker rm mysql
```
### Executar comandos em um container ativo
Abre um terminal interativo dentro de um container que já está rodando.
Exemplo: Executando o shell bash dentro do container nginx
```shell
docker exec -it nginx /bin/bash
```

### Forçar a remoção de todos os containers do host
Listar apenas os IDs de todos os containers (-qa) e os remove forçadamente (-f), limpando o ambiente.
```bash
docker container rm -f $(docker container ls -qa)
```

## 2. <span id='gerenciamento-e-inspecao-de-imagens'> 🖼️ Gerenciamento e Inspeção de Imagens</span>

### Exibe todas as imagens Docker baixadas ou buildadas que estão disponíveis na máquina.
```bash
docker image ls
```
### Retorna um JSON detalhado com as especificações técnicas da imagem (camadas, variáveis, portas expostas, etc.).
```bash
docker image inspect nomedaimagem
```
### Mostra as camadas estruturais da imagem e os comandos que foram executados para criá-la.
```bash
docker image history nomedaimagem
```

### Gera uma nova imagem local a partir das alterações feitas manualmente dentro de um container em execução.
```bash
docker commit id_da_imagem nome_da_imagem
```
### Similar ao comando acima, mas especificando tag e versão:
```bash
docker commit c3f279d17e0a novaimagem/testimage:version3
```
### Salva uma imagem local em um arquivo compactado (tarball), facilitando o transporte manual para outros servidores.
```bash
docker image save nomedaimagem
```
### Remove do host todas as imagens que não estão associadas a nenhum container ativo ou parado, liberando espaço em disco.
```bash
docker image prune
```

## 3. <span id="build-de-imagens"> 🛠️ Build de Imagens (Dockerfile)</span>

### Compila uma nova imagem apontando explicitamente para um arquivo (-f). O ponto (.) no final indica o contexto atual.
```bash
docker build -t nome_da_imagem -f Dockerfile .
```
### Força o Docker a reexecutar todas as etapas do Dockerfile do zero, sem reaproveitar o cache de builds anteriores. 
```bash
docker build -t nome_da_imagem . --no-cache
```

### Permite injetar variáveis em tempo de build (--build-arg) que serão consumidas pelas instruções ARG dentro do seu Dockerfile.
```bash
docker build -t nome_da_imagem --build-arg VAR_TEXTO=teste .
```
### Permite injetar variáveis em tempo de build (--build-arg) que serão consumidas pelas instruções ARG dentro do seu Dockerfile.
docker build -t nome_da_imagem --build-arg VAR_TEXTO=teste .



