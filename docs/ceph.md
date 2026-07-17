# 🐙 Guia de Comandos Ceph

Anotações de diagnóstico, manutenção, OSDs, pools, monitores e RBD para um cluster Ceph — seja standalone, seja integrado como storage compartilhado de um cluster Proxmox (veja [proxmox.md](proxmox.md)).

## 📑 Índice

1. [Status e Diagnóstico do Cluster](#status-diagnostico)
2. [Manutenção e Flags Operacionais](#manutencao-flags)
3. [Gerenciamento de OSDs](#gerenciamento-osds)
4. [Gerenciamento de Pools](#gerenciamento-pools)
5. [Monitores e Managers (MON/MGR)](#monitores-managers)
6. [RBD (RADOS Block Device)](#rbd)
7. [Autenticação e Chaves (cephx)](#autenticacao-cephx)
8. [Boas Práticas](#boas-praticas)
---

## 1. <span id="status-diagnostico">🔍 Status e Diagnóstico do Cluster</span>

### 🔹 Ver o status geral do cluster
Resumo de saúde (`HEALTH_OK`/`WARN`/`ERR`), quórum de monitores, número de OSDs ativos e estado dos placement groups — o primeiro comando a rodar em qualquer investigação.
```bash
ceph -s
```

### 🔹 Ver detalhes de um problema de saúde
Quando o status geral reporta `HEALTH_WARN` ou `HEALTH_ERR`, este comando detalha exatamente o que está errado e, geralmente, sugere a ação corretiva.
```bash
ceph health detail
```

### 🔹 Acompanhar o log do cluster em tempo real
```bash
ceph -w
```

### 🔹 Ver a topologia dos OSDs
Mostra a árvore de hosts, OSDs e seus pesos/status (`up`/`down`, `in`/`out`) — útil para identificar rapidamente qual disco ou nó está com problema.
```bash
ceph osd tree
```

### 🔹 Ver o uso de espaço do cluster
```bash
ceph df
ceph df detail
```

### 🔹 Ver o status dos Placement Groups (PGs)
PGs em estados diferentes de `active+clean` costumam ser a causa raiz de um `HEALTH_WARN` — este comando resume a distribuição de estados.
```bash
ceph pg stat
```

## 2. <span id="manutencao-flags">🛠️ Manutenção e Flags Operacionais</span>

Flags que suspendem temporariamente comportamentos automáticos do Ceph durante uma janela de manutenção (troca de disco, atualização de nó, reboot). **Sempre desfaça a flag ao final da manutenção** — ver [Boas Práticas](#boas-praticas).

### 🔹 Impedir a re-replicação automática de um OSD/nó que vai sair do ar
Evita sobrecarga desnecessária na rede quando o cluster percebe OSDs temporariamente indisponíveis durante uma manutenção programada — é a flag mais usada antes de reiniciar um nó (veja o fluxo completo em [proxmox.md#ha-manutencao](proxmox.md#ha-manutencao)).
```bash
ceph osd set noout
ceph osd unset noout
```

### 🔹 Suspender o scrubbing
`scrub` e `deep-scrub` são verificações de integridade de dados que consomem I/O; suspendê-las durante uma manutenção evita competir por recursos com a evacuação/migração em andamento.
```bash
ceph osd set noscrub
ceph osd set nodeep-scrub
ceph osd unset noscrub
ceph osd unset nodeep-scrub
```

### 🔹 Impedir rebalanceamento e backfill
Usadas em manutenções mais longas, quando você não quer que o cluster comece a mover dados entre OSDs a cada pequena oscilação de disponibilidade.
```bash
ceph osd set nobackfill
ceph osd set norebalance
ceph osd set norecover
```

### 🔹 Pausar toda a I/O do cluster
⚠️ **Extremamente disruptivo** — congela leituras e escritas em todo o cluster. Reservado para cenários críticos de recuperação, nunca para manutenção de rotina.
```bash
ceph osd set pause
ceph osd unset pause
```

### 🔹 Ver quais flags de manutenção estão ativas
```bash
ceph osd dump | grep flags
```

## 3. <span id="gerenciamento-osds">💽 Gerenciamento de OSDs</span>

### 🔹 Listar os IDs de todos os OSDs
```bash
ceph osd ls
```

### 🔹 Tirar um OSD de operação (out) e devolvê-lo (in)
`out` faz o Ceph parar de alocar dados novos naquele OSD e iniciar a re-replicação dos dados existentes para os demais; `in` reverte esse estado.
```bash
ceph osd out osd.5
ceph osd in osd.5
```

### 🔹 Marcar um OSD como down
```bash
ceph osd down osd.5
```

### 🔹 Parar/iniciar o serviço de um OSD no host
```bash
systemctl stop ceph-osd@5
systemctl start ceph-osd@5
```

### 🔹 Remover permanentemente um OSD
Sequência completa para descomissionar um disco: primeiro tira o OSD de operação, espera a re-replicação terminar (acompanhe com `ceph -s`), depois purga a entrada do cluster.
```bash
ceph osd out osd.5
systemctl stop ceph-osd@5
ceph osd purge osd.5 --yes-i-really-mean-it
```

### 🔹 Criar um novo OSD (via Proxmox)
Em um cluster Proxmox, o `pveceph` automatiza a criação do OSD (particionamento, journal, registro no cluster) sobre um disco bruto.
```bash
pveceph osd create /dev/sdb
```

### 🔹 Listar os volumes lógicos usados pelos OSDs no host
```bash
ceph-volume lvm list
```

## 4. <span id="gerenciamento-pools">🗄️ Gerenciamento de Pools</span>

Pools são os "namespaces" lógicos do Ceph onde os dados são efetivamente armazenados — cada um com sua própria política de replicação e número de placement groups.

### 🔹 Criar um pool
Os dois números representam `pg_num` e `pgp_num` (quantidade de placement groups) — dimensione de acordo com o tamanho do cluster e o autoscaler.
```bash
ceph osd pool create nome-pool 128 128
```

### 🔹 Listar pools
```bash
ceph osd pool ls
ceph osd pool ls detail
```

### 🔹 Ver e ajustar o fator de replicação de um pool
`size` define quantas cópias de cada objeto existem no cluster — o padrão recomendado em produção é `3`.
```bash
ceph osd pool get nome-pool size
ceph osd pool set nome-pool size 3
```

### 🔹 Ativar o autoscaler de PGs
Deixa o próprio Ceph ajustar o número de placement groups do pool conforme o volume de dados cresce, evitando ajuste manual constante.
```bash
ceph osd pool set nome-pool pg_autoscale_mode on
```

### 🔹 Remover um pool
⚠️ **Irreversível.** Por segurança, o Ceph exige o nome do pool duplicado e uma flag de confirmação explícita.
```bash
ceph osd pool delete nome-pool nome-pool --yes-i-really-really-mean-it
```

## 5. <span id="monitores-managers">🧭 Monitores e Managers (MON/MGR)</span>

Monitores (MON) mantêm o mapa de estado do cluster; Managers (MGR) expõem métricas, dashboard e módulos de automação.

### 🔹 Ver o status dos monitores
```bash
ceph mon stat
ceph mon dump
```

### 🔹 Adicionar um monitor (via Proxmox)
```bash
pveceph mon create
```

### 🔹 Ver os serviços ativos nos managers
```bash
ceph mgr services
```

### 🔹 Habilitar o módulo de dashboard
Ativa a interface web nativa do Ceph, com métricas e visão gráfica de pools, OSDs e PGs.
```bash
ceph mgr module enable dashboard
```

## 6. <span id="rbd">📀 RBD (RADOS Block Device)</span>

RBD é o mecanismo de blocos do Ceph usado, por exemplo, pelo Proxmox para armazenar discos de VMs diretamente em um pool Ceph.

### 🔹 Listar imagens RBD de um pool
```bash
rbd ls nome-pool
```

### 🔹 Ver detalhes de uma imagem RBD
```bash
rbd info nome-pool/nome-imagem
```

### 🔹 Criar uma imagem RBD
```bash
rbd create nome-pool/novo-disco --size 10G
```

### 🔹 Redimensionar uma imagem RBD
```bash
rbd resize nome-pool/novo-disco --size 20G
```

### 🔹 Ver o uso de espaço de um pool RBD
```bash
rbd du nome-pool
```

### 🔹 Criar e listar snapshots de uma imagem RBD
```bash
rbd snap create nome-pool/nome-imagem@antes-da-migracao
rbd snap ls nome-pool/nome-imagem
```

### 🔹 Remover uma imagem RBD
```bash
rbd rm nome-pool/novo-disco
```

## 7. <span id="autenticacao-cephx">🔐 Autenticação e Chaves (cephx)</span>

O Ceph usa o protocolo `cephx` para autenticar clientes e serviços — cada cliente (usuário, VM, serviço) possui uma chave com capabilities específicas por recurso.

### 🔹 Listar todas as identidades e suas permissões
```bash
ceph auth list
```

### 🔹 Ver a chave de um cliente específico
```bash
ceph auth get client.admin
```

### 🔹 Criar um novo cliente com permissões restritas
Cria (ou reaproveita, se já existir) um cliente com leitura no monitor e leitura/escrita/execução restritas a um único pool — o princípio do menor privilégio aplicado ao Ceph.
```bash
ceph auth get-or-create client.novo-usuario mon 'allow r' osd 'allow rwx pool=nome-pool'
```

### 🔹 Remover um cliente
```bash
ceph auth del client.novo-usuario
```

## 8. <span id="boas-praticas">🧭 Boas Práticas</span>

### 🔹 Trate flags de manutenção como um par
Toda vez que você `set` uma flag (`noout`, `noscrub`, `nobackfill`, etc.), programe-se para `unset` assim que a manutenção terminar — flags esquecidas ativas mascaram problemas reais de saúde do cluster e atrasam a recuperação automática.

### 🔹 Nunca deixe o cluster operar por muito tempo em `HEALTH_WARN`
Um `WARN` ignorado tende a evoluir para `HEALTH_ERR` sob a próxima falha de disco. Trate o alerta como um chamado a ser investigado, não como ruído.

### 🔹 Mantenha `size=3` em pools de produção
Um fator de replicação menor que 3 reduz a tolerância a falhas simultâneas de disco/nó — reserve `size=2` apenas para ambientes de teste.

### 🔹 Acompanhe a re-replicação após remover ou trocar um OSD
Depois de um `osd out`/`osd purge`, monitore `ceph -s` até o cluster voltar para `active+clean` antes de iniciar qualquer outra manutenção — remover um segundo OSD nesse meio tempo pode colocar dados em risco.
