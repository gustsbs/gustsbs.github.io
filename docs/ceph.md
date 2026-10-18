# 🐙 Guia de Comandos Ceph

Anotações de diagnóstico, manutenção, OSDs, pools, monitores e RBD para um cluster Ceph — seja standalone, seja integrado como storage compartilhado de um cluster Proxmox (veja [proxmox.md](proxmox.md)).

## 📑 Índice

1. [Status e Diagnóstico do Cluster](#status-diagnostico)
2. [Manutenção e Flags Operacionais](#manutencao-flags)
3. [Gerenciamento de OSDs](#gerenciamento-osds)
4. [Detecção e Troca de Disco com Falha](#disco-com-falha)
5. [Gerenciamento de Pools](#gerenciamento-pools)
6. [Monitores e Managers (MON/MGR)](#monitores-managers)
7. [RBD (RADOS Block Device)](#rbd)
8. [Autenticação e Chaves (cephx)](#autenticacao-cephx)
9. [Integração com Kubernetes (ceph-csi)](#ceph-csi)
10. [Boas Práticas e Pegadinhas](#boas-praticas)
11. [Referências](#referencias)
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

### 🔹 Identificar o cluster (fsid)
Cada cluster Ceph tem um identificador único. É o mesmo valor usado como `clusterID` pelo ceph-csi no Kubernetes — compare os dois para ter certeza de que uma StorageClass aponta para o cluster que você imagina.
```bash
ceph fsid
```

### 🔹 Ver a latência de cada OSD
Lista a latência de *commit* e *apply* (ms) de cada OSD. Ordenando pela coluna de latência, os discos mais lentos aparecem no topo — se forem sempre do mesmo host, é ele que está ditando a velocidade do cluster.
```bash
ceph osd perf | sort -k3 -n -r | head -15
```

### 🔹 Ver a vazão atual de um pool
Mostra operações e MB/s de leitura e escrita dos clientes de um pool específico — útil para separar a carga de um teste da carga de produção.
```bash
ceph osd pool stats kubernetes
ceph osd pool stats proxmox
```

### 🔹 Registrar o lado do servidor durante um benchmark
Durante um teste de desempenho (ex.: `fio` num pod), grave a cada 10 s, com horário, a vazão dos pools e as OSDs mais lentas. Rode com `nohup` no próprio nó do Ceph para não depender da sessão SSH.
```bash
cat > monitor-ceph.sh <<'EOF'
#!/usr/bin/env bash
OUT="ceph-monitor-$(date +%Y%m%d-%H%M).txt"
{ echo "### inicio $(date -Is)"; ceph -s; ceph df
  for p in "$@"; do echo "## pool $p"; ceph osd pool get "$p" size; ceph osd pool get "$p" crush_rule; done; } > "$OUT"
while :; do
  { echo "### $(date -Is)"
    for p in "$@"; do ceph osd pool stats "$p" | grep -E "pool|client io|recovery"; done
    ceph osd perf | sort -k3 -n -r | head -15
  } >> "$OUT"
  sleep 10
done
EOF
chmod +x monitor-ceph.sh
nohup ./monitor-ceph.sh kubernetes proxmox > monitor.log 2>&1 & echo $! > monitor.pid
kill $(cat monitor.pid)        # ao final do teste
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

## 4. <span id="disco-com-falha">🩺 Detecção e Troca de Disco com Falha</span>

Passo a passo para sair de um alerta genérico até a troca física de um disco, separando **disco com defeito** de **disco apenas lento** (classe inadequada). Registre a detecção como incidente e a troca física como mudança.

### 🔹 Passo 1 — Verificar a saúde do cluster
Qualquer `HEALTH_WARN` merece investigação. Alertas como `BLUESTORE_SLOW_OP_ALERT`, `SLOW_OPS` ou `DEVICE_HEALTH` são os que apontam para discos.
```bash
ceph -s
```

### 🔹 Passo 2 — Descobrir quais OSDs geraram o alerta
```bash
ceph health detail
```
```text
[WRN] BLUESTORE_SLOW_OP_ALERT: 6 OSD(s) experiencing slow operations in BlueStore
     osd.17 observed slow operation indications in BlueStore
     osd.42 observed slow operation indications in BlueStore
```

### 🔹 Passo 3 — Confirmar a latência e localizar o host
O `ceph osd perf` mostra a latência atual (ms); o `ceph osd find` diz em que servidor a OSD está. Se todas as OSDs lentas forem do mesmo host, suspeite do host (tipo de disco, controladora) antes de suspeitar de discos isolados.
```bash
ceph osd perf | sort -k3 -n -r | head -15
ceph osd find 42 | grep -E '"host"|"osd"'
```

### 🔹 Passo 4 — Identificar o disco físico de cada OSD
Mostra o modelo/número de série (`DEVICE`) e o caminho no sistema (`DEV`) — é o caminho que o `smartctl` usa.
```bash
ceph device ls-by-host ceph-node0
ceph device ls-by-daemon osd.42
```
```text
DEVICE                          DEV  DAEMONS  EXPECTED FAILURE
SEAGATE_ST4000NM0295_ZC1XXXXX   sdv  osd.42
```

### 🔹 Passo 5 — Ler o SMART do disco
Discos SAS mostram status e lista de defeitos; discos SATA mostram atributos. Atrás de controladora RAID (ex.: Dell PERC), use `-d megaraid,N`.
```bash
# SAS
smartctl -a /dev/sdv | grep -iE "health|grown defect|uncorrected|non-medium"
# SATA (vários de uma vez)
for d in sdc sdd sde; do echo "=== /dev/$d"; smartctl -H -A /dev/$d | grep -E "overall-health|Reallocated_Sector_Ct|Current_Pending_Sector|Offline_Uncorrectable|UDMA_CRC_Error_Count|Power_On_Hours"; done
# atrás da controladora
smartctl -a -d megaraid,0 /dev/sdv | head -30
```

| Indicador | Saudável | Problema |
| :--- | :--- | :--- |
| `SMART Health Status` (SAS) / `overall-health` (SATA) | `OK` / `PASSED` | `impending failure`, `FAILED` |
| `Elements in grown defect list` (SAS) | perto de 0 | milhares → disco falhando |
| `Reallocated_Sector_Ct` / `Current_Pending_Sector` (SATA) | 0 | > 0 e crescendo |
| `UDMA_CRC_Error_Count` (SATA) | 0 | alto → cabo/controladora, não o disco |

### 🔹 Passo 6 — Decidir: defeito ou classe inadequada
| Situação | Diagnóstico | Ação |
| :--- | :--- | :--- |
| SMART com falha/defeitos | Disco com defeito | Retirar a OSD (passo 8) e trocar o disco |
| SMART saudável, mas sempre lento | Disco de classe inadequada (ex.: discos de vigilância, 5400 rpm) | Mitigar (passo 7) e planejar substituição |

### 🔹 Passo 7 — Mitigar discos lentos sem mover dados
Com `primary-affinity 0`, a OSD deixa de ser a primária das leituras — os clientes leem das réplicas em outros hosts. Não move dados e é reversível (`1`).
```bash
for o in 17 18 19 22 25; do ceph osd primary-affinity osd.$o 0; done
ceph osd tree | head -30        # coluna PRI-AFF
```

### 🔹 Passo 8 — Retirar do cluster a OSD com defeito
O `out` faz o Ceph copiar os dados para as outras OSDs. Opcionalmente, priorize o tráfego dos clientes durante a redistribuição.
⚠️ **Atenção:** só retire uma OSD por vez e aguarde `active+clean` antes de mexer em outra.
```bash
ceph osd out osd.42
ceph config set osd osd_mclock_profile high_client_ops
watch -n 30 ceph -s                      # aguardar todos os PGs active+clean
ceph config rm osd osd_mclock_profile
ceph osd safe-to-destroy osd.42          # confirma que nenhum dado depende mais dela
ceph orch osd rm 42 --replace            # cephadm: remove e reserva o ID para o disco novo
```

### 🔹 Passo 9 — Localizar o disco na gaveta do servidor
Tente nesta ordem. **Antes de puxar o disco, confira o número de série na etiqueta.**
```bash
# a) pelo Ceph (depende de suporte da gaveta/controladora)
ceph device light on SEAGATE_ST4000NM0295_ZC1XXXXX ident
ceph device light off SEAGATE_ST4000NM0295_ZC1XXXXX ident
# b) pela controladora Dell PERC
perccli /c0 show
perccli /c0/e32/s7 start locate
perccli /c0/e32/s7 stop locate
# c) último recurso: leitura contínua acende só o LED de atividade desse disco (somente leitura)
dd if=/dev/sdv of=/dev/null bs=1M status=progress
```
Em servidores Dell, o iDRAC também oferece **Storage → Physical Disks → Blink**.

### 🔹 Passo 10 — Conferir depois da troca
Com `--replace`, o cephadm recria a OSD com o mesmo ID no disco novo.
```bash
ceph orch ps --daemon-type osd | grep osd.42
ceph osd tree | grep -w osd.42
ceph -s                                  # aguardar active+clean e o fim do alerta
```

### 🔹 Prevenção — ligar a previsão de falha e os alertas
O Ceph coleta o SMART de todos os discos e pode prever falhas (coluna `EXPECTED FAILURE`). Sem alerta configurado, porém, o `HEALTH_WARN` passa despercebido.
```bash
ceph device monitoring on
ceph mgr module enable diskprediction_local
ceph config set global device_failure_prediction_mode local
ceph device get-health-metrics SEAGATE_ST4000NM0295_ZC1XXXXX | head -40
ceph orch ls | grep -E "prometheus|alertmanager|grafana"    # pilha de alertas do cephadm
```

## 5. <span id="gerenciamento-pools">🗄️ Gerenciamento de Pools</span>

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

### 🔹 Descobrir em que tipo de disco um pool grava (regra CRUSH)
A regra CRUSH decide onde ficam as réplicas. Se o `item_name` for só `default` (sem `~hdd` ou `~ssd`), a regra **mistura HDD e SSD** — num cluster majoritariamente HDD, os dados caem quase todos em disco mecânico.
```bash
ceph osd pool get proxmox crush_rule
ceph osd crush rule dump replicated_rule | grep item_name
```

### 🔹 Criar regras por classe de disco e aplicar a um pool
Permite separar o que precisa de latência baixa (ex.: discos das VMs do control plane/etcd) em SSD.
⚠️ **Atenção:** trocar a regra de um pool existente faz o Ceph **mover todos os dados** daquele pool — planeje para fora do horário de pico e acompanhe com `ceph -s`.
```bash
ceph osd crush rule create-replicated replicated-hdd default host hdd
ceph osd crush rule create-replicated replicated-ssd default host ssd
ceph osd pool set kubernetes-ssd crush_rule replicated-ssd
```

### 🔹 Conferir/habilitar a aplicação de um pool
O Ceph avisa (`HEALTH_WARN`) quando um pool não tem aplicação definida. Para pools de blocos, o `rbd pool init` define a aplicação `rbd`.
```bash
ceph osd pool application get kubernetes
rbd pool init kubernetes
```

### 🔹 Limitar o espaço de um pool (cota)
Protege o cluster de um pool que cresce sem controle — importante com imagens *thin*, em que a soma dos volumes pode passar do espaço real.
```bash
ceph osd pool set-quota kubernetes max_bytes $((2*1024**4))   # 2 TiB
ceph osd pool get-quota kubernetes
```

### 🔹 Remover um pool
⚠️ **Irreversível.** Por segurança, o Ceph exige o nome do pool duplicado e uma flag de confirmação explícita.
```bash
ceph osd pool delete nome-pool nome-pool --yes-i-really-really-mean-it
```

## 6. <span id="monitores-managers">🧭 Monitores e Managers (MON/MGR)</span>

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

## 7. <span id="rbd">📀 RBD (RADOS Block Device)</span>

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

### 🔹 Testar um usuário cephx criando uma imagem
Valida chave e permissões de um cliente sem passar pelo Kubernetes. Se o erro for `(2) No such file or directory` ao abrir o pool, o pool não existe **neste** cluster — confira o `ceph fsid`.
```bash
ceph auth get client.k8s-rbd > /tmp/k8s-rbd.keyring
rbd ls -p kubernetes --id k8s-rbd --keyring /tmp/k8s-rbd.keyring
rbd create kubernetes/teste-k8s-rbd --size 1G --id k8s-rbd --keyring /tmp/k8s-rbd.keyring
rbd rm kubernetes/teste-k8s-rbd --id k8s-rbd --keyring /tmp/k8s-rbd.keyring
rm /tmp/k8s-rbd.keyring
```

### 🔹 Montar uma imagem para perícia ou backup (somente leitura)
⚠️ **Cuidado:** nunca monte uma imagem que está em uso por uma VM ou pod — dois sistemas montando o mesmo ext4 corrompem os dados. Use sempre um snapshot em modo somente leitura.
```bash
rbd snap create kubernetes/csi-vol-1234@pericia
rbd map --read-only kubernetes/csi-vol-1234@pericia     # retorna /dev/rbdX
mount -o ro,noload /dev/rbd0 /mnt/pericia
umount /mnt/pericia && rbd unmap /dev/rbd0
```

### 🔹 Remover uma imagem RBD
```bash
rbd rm nome-pool/novo-disco
```

## 8. <span id="autenticacao-cephx">🔐 Autenticação e Chaves (cephx)</span>

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

### 🔹 Criar um cliente para RBD restrito a um pool (recomendado)
Os perfis `profile rbd` já trazem as permissões certas para criar, mapear e remover imagens. É o padrão para o ceph-csi e para clientes de virtualização.
```bash
ceph auth get-or-create client.k8s-rbd \
  mon 'profile rbd' \
  osd 'profile rbd pool=kubernetes' \
  mgr 'profile rbd pool=kubernetes'
```

### 🔹 Alterar as permissões de um cliente
⚠️ **Atenção:** `ceph auth caps` **substitui todas** as permissões do cliente. Repita as que já existem e acrescente as novas na mesma linha.
```bash
ceph auth caps client.k8s-rbd \
  mon 'profile rbd' \
  osd 'profile rbd pool=kubernetes, profile rbd pool=kubernetes-ssd' \
  mgr 'profile rbd pool=kubernetes, profile rbd pool=kubernetes-ssd'
```

### 🔹 Obter só a chave de um cliente
Imprime apenas a chave (40 caracteres, terminando em `=`). Evite colar a saída em chats, tickets ou repositórios.
```bash
ceph auth get-key client.k8s-rbd; echo
```

### 🔹 Trocar a chave de um cliente (rotação)
Se a chave vazou e o cliente ainda não está em uso, o caminho mais simples é remover e recriar o cliente — uma chave nova é gerada.
```bash
ceph auth del client.k8s-rbd
ceph auth get-or-create client.k8s-rbd mon 'profile rbd' osd 'profile rbd pool=kubernetes' mgr 'profile rbd pool=kubernetes'
```

### 🔹 Remover um cliente
```bash
ceph auth del client.novo-usuario
```

## 9. <span id="ceph-csi">☸️ Integração com Kubernetes (ceph-csi)</span>

O ceph-csi é o driver que cria imagens RBD (ou volumes CephFS) quando um PVC é pedido e as monta nos nós. A chave do Ceph pertence ao **driver**, não aos sites: normalmente uma por cluster Kubernetes (ou por StorageClass). O isolamento entre namespaces é feito pelo próprio Kubernetes.

### 🔹 Ver quais clusters Ceph o ceph-csi conhece
O `config.json` lista cada `clusterID` (= `ceph fsid`) e seus monitores.
```bash
kubectl -n ceph-csi get cm ceph-csi-config -o jsonpath='{.data.config\.json}'; echo
```

### 🔹 Acrescentar um cluster ao ceph-csi
Use um *merge patch* com o `config.json` completo (cluster existente + novo). Guarde uma cópia antes.
```bash
kubectl -n ceph-csi get cm ceph-csi-config -o yaml > ~/ceph-csi-config.bkp.yaml
kubectl -n ceph-csi patch cm ceph-csi-config --type merge --patch-file ceph-csi-config-patch.yaml
```
```yaml
data:
  config.json: |
    [
      {"clusterID": "<FSID-CLUSTER-ATUAL>", "monitors": ["10.0.60.1:6789", "10.0.60.2:6789", "10.0.60.3:6789"]},
      {"clusterID": "<FSID-CLUSTER-NOVO>",  "monitors": ["10.0.70.1:6789", "10.0.70.2:6789", "10.0.70.3:6789"]}
    ]
```

### 🔹 Criar o Secret do ceph-csi com a chave em texto
Use `--from-literal` (o kubectl codifica em base64 corretamente). Valide contando os caracteres, sem exibir a chave: deve dar **40**.
```bash
kubectl -n ceph-csi create secret generic csi-rbd-secret-v2 \
  --from-literal=userID=k8s-rbd \
  --from-literal=userKey='<CHAVE-DO-CEPH-AUTH-GET-KEY>'
kubectl -n ceph-csi get secret csi-rbd-secret-v2 -o jsonpath='{.data.userKey}' | base64 -d | wc -c
```

### 🔹 Descobrir qual Secret e qual cluster uma StorageClass usa
```bash
kubectl get sc csi-rbd-sc-hdd -o jsonpath='{.parameters.clusterID}{"  "}{.parameters.pool}{"  "}{.parameters.csi\.storage\.k8s\.io/provisioner-secret-name}{"\n"}'
kubectl get pv <NOME-DO-PV> -o jsonpath='{.spec.csi.nodeStageSecretRef}{"\n"}'
```

### 🔹 Criar uma StorageClass nova a partir de uma existente
Os parâmetros de uma StorageClass **não podem ser editados**. Para trocar Secret, cluster ou pool, crie uma classe nova (os PVs antigos continuam apontando para a configuração com que foram criados).
```bash
kubectl get sc csi-rbd-sc-hdd -o yaml \
  | grep -vE "uid:|resourceVersion:|creationTimestamp:|last-applied-configuration|is-default-class|allowVolumeExpansion\":true" \
  | sed -e "s/name: csi-rbd-sc-hdd/name: csi-rbd-sc-hdd-v2/" -e "s/csi-rbd-secret\b/csi-rbd-secret-v2/g" \
  > sc-rbd-hdd-v2.yaml
grep -E "name:|secret|pool|clusterID" sc-rbd-hdd-v2.yaml      # conferir antes de aplicar
kubectl apply -f sc-rbd-hdd-v2.yaml
```

### 🔹 Achar o erro real de um PVC em Pending
Os eventos do PVC mostram o resumo; o log do contêiner `csi-rbdplugin` do provisionador mostra a chamada `CreateVolume` e o erro do Ceph.
```bash
kubectl -n <NAMESPACE> describe pvc <PVC> | tail -6
for p in $(kubectl -n ceph-csi get pods -o name | grep rbdplugin-provisioner); do
  kubectl -n ceph-csi logs $p -c csi-rbdplugin --tail=300 | grep -A4 CreateVolume | tail -8
done
```

### 🔹 Ver quais pods usam volumes do Ceph
```bash
kubectl get pv -o custom-columns=NS:.spec.claimRef.namespace,PVC:.spec.claimRef.name,DRIVER:.spec.csi.driver,CLASSE:.spec.storageClassName | grep -E "ceph|s3"
kubectl get volumeattachment -o custom-columns=DRIVER:.spec.attacher,PV:.spec.source.persistentVolumeName,NO:.spec.nodeName
```

### 🔹 Limitar o espaço por namespace
O tamanho do PVC define o tamanho da imagem RBD; a `ResourceQuota` limita quanto cada site/setor pode pedir.
```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: armazenamento
  namespace: site-exemplo
spec:
  hard:
    requests.storage: 50Gi
    persistentvolumeclaims: "3"
```

## 10. <span id="boas-praticas">🧭 Boas Práticas e Pegadinhas</span>

### 🔹 Trate flags de manutenção como um par
Toda vez que você `set` uma flag (`noout`, `noscrub`, `nobackfill`, etc.), programe-se para `unset` assim que a manutenção terminar — flags esquecidas ativas mascaram problemas reais de saúde do cluster e atrasam a recuperação automática.

### 🔹 Nunca deixe o cluster operar por muito tempo em `HEALTH_WARN`
Um `WARN` ignorado tende a evoluir para `HEALTH_ERR` sob a próxima falha de disco. Trate o alerta como um chamado a ser investigado, não como ruído.

### 🔹 Mantenha `size=3` em pools de produção
Um fator de replicação menor que 3 reduz a tolerância a falhas simultâneas de disco/nó — reserve `size=2` apenas para ambientes de teste.

### 🔹 Acompanhe a re-replicação após remover ou trocar um OSD
Depois de um `osd out`/`osd purge`, monitore `ceph -s` até o cluster voltar para `active+clean` antes de iniciar qualquer outra manutenção — remover um segundo OSD nesse meio tempo pode colocar dados em risco.

### 🔹 "invalid UTF-8" ao provisionar um PVC = chave corrompida no Secret
Chaves do Ceph parecem base64. Se forem coladas direto no campo `data:` de um Secret, o Kubernetes as decodifica e o resultado vira binário — o provisionador nem consegue montar a mensagem gRPC (`grpc: error while marshaling: string field contains invalid UTF-8`). Recrie o Secret com `--from-literal` ou `stringData:`.

### 🔹 Cuidado com `$(ssh host comando)` dentro do `screen`/`tmux`
O shell remoto pode imprimir sequências de título do terminal que são capturadas junto com a saída (ex.: uma chave com 71 caracteres em vez de 40). Para segredos, prefira copiar o valor manualmente.

### 🔹 Confira o fsid antes de culpar permissões
Mais de um cluster Ceph no mesmo ambiente é comum. Um pool "inexistente" ou um provisionamento que fica esperando sem erro pode ser só a StorageClass apontando para outro cluster.

### 🔹 O host mais lento dita a latência de escrita
Com réplica 3 e uma regra que espalha os dados por todos os hosts, quase toda escrita espera uma OSD do servidor mais lento. Acompanhe `ceph osd perf`: se o topo for sempre o mesmo host, investigue os discos dele (`ceph health detail`, SMART) ou reduza o peso dele no CRUSH.

### 🔹 Não use discos de vigilância em OSDs
Discos de vigilância (ex.: linha "Purple") foram feitos para gravação sequencial contínua de vídeo. Mesmo novos e com SMART perfeito, ficam lentos sob a escrita aleatória com réplicas do Ceph e aparecem em `BLUESTORE_SLOW_OP_ALERT`. Especifique discos de linha empresarial/NAS nas compras.

### 🔹 Sinal sem alerta é o mesmo que não ter sinal
Um disco com falha iminente pode passar semanas em `HEALTH_WARN` sem ninguém ver. Ligue a previsão de falha do Ceph e encaminhe os alertas (Alertmanager, iDRAC, `smartd`) para a equipe.

### 🔹 Regra padrão não separa HDD de SSD
A `replicated_rule` criada na instalação usa a raiz inteira. Crie regras por classe antes de colocar cargas sensíveis a latência (etcd, bancos) num pool.

### 🔹 Apagar arquivos não devolve espaço sem `discard`
Imagens RBD são *thin*. Arquivos apagados dentro do ext4/xfs só liberam espaço no Ceph se o volume for montado com `discard` (ou com `fstrim` periódico) — a StorageClass pode definir `mountOptions: [discard]`.

## 11. <span id="referencias">📚 Referências</span>

- [Ceph CSI — documentação do driver para Kubernetes](https://github.com/ceph/ceph-csi/tree/devel/docs)
- [Ceph — User Management (cephx, profiles)](https://docs.ceph.com/en/latest/rados/operations/user-management/)
- [Ceph — CRUSH Maps e device classes](https://docs.ceph.com/en/latest/rados/operations/crush-map/)
- [Ceph Documentation — Documentação oficial](https://docs.ceph.com/)
- [Deploy Hyper-Converged Ceph Cluster (Proxmox Wiki)](https://pve.proxmox.com/wiki/Deploy_Hyper-Converged_Ceph_Cluster)
- [pveceph(1) — manual do utilitário de integração Proxmox+Ceph](https://pve.proxmox.com/pve-docs/pveceph.1.html)
