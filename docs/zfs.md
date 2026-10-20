# 🗄️ Guia de Comandos ZFS

Anotações de administração de pools, datasets, snapshots, clones e replicação ZFS — comumente usado como backend de armazenamento local em nós Proxmox (veja [proxmox.md](proxmox.md)).

## 📑 Índice

1. [Gerenciamento de Pools](#gerenciamento-pools)
2. [Gerenciamento de Datasets](#gerenciamento-datasets)
3. [Snapshots](#snapshots)
4. [Clones](#clones)
5. [Envio e Recepção (send/receive)](#send-receive)
6. [Propriedades e Ajustes](#propriedades)
7. [Monitoramento e Manutenção](#monitoramento-manutencao)
8. [Expandindo um Pool Após Crescer o Disco Virtual (Caso Real)](#expandir-pool-disco-virtual)
9. [Migração Online do Pool para Outro Disco — iSCSI → vSAN (Caso Real)](#migracao-online-pool)
10. [Boas Práticas](#boas-praticas)
11. [Referências](#referencias)
---

## 1. <span id="gerenciamento-pools">🧱 Gerenciamento de Pools</span>

Um *pool* é o agrupamento de discos físicos sobre o qual todos os datasets são criados — a unidade fundamental de armazenamento e redundância no ZFS.

### 🔹 Criar um pool espelhado (mirror)
Cria um pool com redundância 1:1 entre dois discos, similar a um RAID 1.
```bash
zpool create nome-pool mirror /dev/sdb /dev/sdc
```

### 🔹 Ver o status de todos os pools
Exibe o estado de saúde (`ONLINE`, `DEGRADED`, `FAULTED`) de cada pool e disco associado — essencial para detectar falhas de disco precocemente.
```bash
zpool status
```

### 🔹 Listar pools e seu uso de espaço
```bash
zpool list
```

### 🔹 Adicionar um disco a um pool existente
Expande a capacidade do pool adicionando um novo vdev — não redistribui os dados já existentes entre os discos antigos e o novo.
```bash
zpool add nome-pool /dev/sdd
```

### 🔹 Substituir um disco com falha
Inicia a resilvering (reconstrução dos dados) no disco novo a partir da redundância disponível no pool.
```bash
zpool replace nome-pool /dev/sdb-antigo /dev/sdb-novo
```

### 🔹 Rodar uma verificação de integridade (scrub)
Lê todos os dados do pool e corrige inconsistências encontradas usando a redundância disponível — o equivalente ZFS de um "check-up" preventivo.
```bash
zpool scrub nome-pool
```

### 🔹 Colocar um disco offline/online
Útil para tirar um disco fisicamente com segurança (troca a quente) sem derrubar o pool inteiro.
```bash
zpool offline nome-pool /dev/sdb
zpool online nome-pool /dev/sdb
```

### 🔹 Exportar e importar um pool
`export` desmonta o pool e o torna "portátil"; `import` o reconecta — usado ao migrar discos físicos para outro servidor.
```bash
zpool export nome-pool
zpool import nome-pool
```

### 🔹 Ver o histórico de comandos executados em um pool
```bash
zpool history nome-pool
```

### 🔹 Destruir um pool
⚠️ **Irreversível** — apaga todos os datasets e dados contidos no pool.
```bash
zpool destroy nome-pool
```

## 2. <span id="gerenciamento-datasets">📁 Gerenciamento de Datasets</span>

Datasets são sistemas de arquivos independentes dentro de um pool, cada um com suas próprias propriedades (quota, compressão, permissões).

### 🔹 Criar um dataset
```bash
zfs create nome-pool/dataset
```

### 🔹 Listar datasets
```bash
zfs list
```

### 🔹 Renomear um dataset
```bash
zfs rename nome-pool/dataset nome-pool/dataset-novo
```

### 🔹 Montar e desmontar um dataset manualmente
```bash
zfs mount nome-pool/dataset
zfs umount nome-pool/dataset
```

### 🔹 Remover um dataset
⚠️ **Irreversível** — falha se existirem snapshots ou clones dependentes, a menos que use `-r` (recursivo).
```bash
zfs destroy nome-pool/dataset
```

## 3. <span id="snapshots">📸 Snapshots</span>

Snapshots são cópias read-only e instantâneas do estado de um dataset em um ponto no tempo — extremamente baratos em espaço, pois armazenam apenas os blocos alterados.

### 🔹 Criar um snapshot
```bash
zfs snapshot nome-pool/dataset@antes-da-migracao
```

### 🔹 Listar snapshots
```bash
zfs list -t snapshot
```

### 🔹 Reverter um dataset para um snapshot
⚠️ **Descarta todas as alterações feitas após o snapshot escolhido.**
```bash
zfs rollback nome-pool/dataset@antes-da-migracao
```

### 🔹 Ver as diferenças entre o snapshot e o estado atual
Lista arquivos criados, modificados ou removidos desde que o snapshot foi tirado — útil para auditar o que mudou antes de decidir por um rollback.
```bash
zfs diff nome-pool/dataset@antes-da-migracao
```

### 🔹 Remover um snapshot
```bash
zfs destroy nome-pool/dataset@antes-da-migracao
```

## 4. <span id="clones">🌱 Clones</span>

Um clone é um dataset gravável criado a partir de um snapshot — inicialmente compartilha os blocos com o snapshot de origem, divergindo apenas conforme é modificado.

### 🔹 Criar um clone a partir de um snapshot
```bash
zfs clone nome-pool/dataset@snapshot nome-pool/dataset-clone
```

### 🔹 Promover um clone a dataset independente
Inverte a relação de dependência entre o clone e o snapshot original — necessário caso você queira remover o dataset original mantendo o clone.
```bash
zfs promote nome-pool/dataset-clone
```

## 5. <span id="send-receive">📡 Envio e Recepção (send/receive)</span>

`send`/`receive` transmitem snapshots inteiros (ou incrementos entre snapshots) entre pools — a base de estratégias de backup e replicação eficientes no ZFS.

### 🔹 Enviar um snapshot para outro pool local
```bash
zfs send nome-pool/dataset@snapshot | zfs receive outro-pool/dataset
```

### 🔹 Replicar um snapshot para um servidor remoto via SSH
```bash
zfs send nome-pool/dataset@snapshot | ssh usuario@servidor-destino zfs receive pool-destino/dataset
```

### 🔹 Enviar apenas as diferenças entre dois snapshots (incremental)
Transmite só os blocos alterados entre `snap1` e `snap2` — muito mais rápido e leve que reenviar o dataset inteiro a cada replicação.
```bash
zfs send -i nome-pool/dataset@snap1 nome-pool/dataset@snap2 | zfs receive outro-pool/dataset
```

## 6. <span id="propriedades">⚙️ Propriedades e Ajustes</span>

### 🔹 Ver todas as propriedades de um dataset
```bash
zfs get all nome-pool/dataset
```

### 🔹 Ativar compressão
`lz4` é o algoritmo recomendado na maioria dos casos: baixo overhead de CPU e boa taxa de compressão.
```bash
zfs set compression=lz4 nome-pool/dataset
```

### 🔹 Definir uma quota de espaço
```bash
zfs set quota=100G nome-pool/dataset
```

### 🔹 Desativar atualização de atime
Reduz escritas desnecessárias em disco causadas pela atualização do horário de último acesso a cada leitura — recomendado para a maioria das cargas de trabalho.
```bash
zfs set atime=off nome-pool/dataset
```

### 🔹 Ajustar o recordsize
Tamanho de bloco usado pelo dataset. Valores menores (ex.: `16K`) favorecem bancos de dados; valores maiores (ex.: `1M`) favorecem arquivos grandes e sequenciais (backups, mídia).
```bash
zfs set recordsize=16K nome-pool/dataset
```

## 7. <span id="monitoramento-manutencao">📊 Monitoramento e Manutenção</span>

### 🔹 Acompanhar I/O do pool em tempo real
```bash
zpool iostat -v 2
```

### 🔹 Ver o progresso de um scrub em andamento
```bash
zpool status nome-pool
```

### 🔹 Rodar TRIM em discos SSD/NVMe
Libera para o firmware do disco os blocos não utilizados pelo ZFS, mantendo a performance de escrita ao longo do tempo em SSDs.
```bash
zpool trim nome-pool
```

## 8. <span id="expandir-pool-disco-virtual">📈 Expandindo um Pool Após Crescer o Disco Virtual (Caso Real)</span>

Cenário: um pool de disco único (`geonode`, sobre `/dev/sdb`) tinha seu `/opt` sem espaço. O disco virtual foi expandido no hypervisor (VxRail) — mas o pool não cresceu sozinho mesmo com `autoexpand=on`, porque ele estava desimportado no momento em que o disco foi redimensionado. Passo a passo do diagnóstico e da correção.

### 🔹 Confirmar que o kernel já enxerga o disco maior
Se o disco foi expandido no hypervisor com a VM ligada, força o kernel a reler o tamanho do dispositivo sem precisar reiniciar.
```bash
echo 1 > /sys/class/block/sdb/device/rescan
lsblk
```

### 🔹 Corrigir a tabela GPT antes de mexer em qualquer coisa
Quando o disco cresce "por baixo" (no hypervisor), a cópia de backup da GPT continua referenciando o fim do disco *antigo*. Ferramentas como `growpart`/`zpool` podem não enxergar o espaço novo até isso ser corrigido.
```bash
parted /dev/sdb print free
```
Ao ser perguntado `Arrumar/Fix/Ignorar/Ignore?`, responder **Fix** — realoca a cópia de backup da GPT para o fim real do disco, sem tocar nos dados das partições existentes.

### 🔹 Confirmar o pool e qual device ele realmente usa
`zpool import` (sem argumentos) lista pools importáveis sem importar nada — é seguro rodar mesmo sem saber o estado atual. Se a GPT estiver inconsistente, ele pode não encontrar nada mesmo havendo um pool real ali (ver nota em Boas Práticas). `zpool status -v` mostra o device exato usado pelo vdev — pode aparecer como o disco inteiro (`sdb`) mesmo que o ZFS tenha particionado ele por baixo (`sdb1` + `sdb9` reservada).
```bash
zpool import
zpool status -v nome-pool
```

### 🔹 Checar e forçar o autoexpand
`autoexpand=on` só reage a um device que cresce **enquanto o pool está importado**. Se o pool foi reimportado depois do disco já ter crescido, é preciso forçar o reconhecimento manualmente com `zpool online -e`, usando exatamente o nome do device que apareceu no `zpool status` (não necessariamente a partição).
```bash
zpool get autoexpand nome-pool
zpool set autoexpand=on nome-pool
zpool online -e nome-pool sdb
```

### 🔹 Confirmar o crescimento
```bash
zpool list nome-pool
df -h /caminho-do-dataset
zfs list -o name,used,avail,mountpoint nome-pool
```

## 9. <span id="migracao-online-pool">🚚 Migração Online do Pool para Outro Disco — iSCSI → vSAN (Caso Real)</span>

Procedimento real usado no servidor NFS `nfs-paginas-novo`: o pool `paginas` estava com ~91% de ocupação em um LUN iSCSI de um storage EMC legado, acessado por caminho único (sem multipath). Ele foi migrado **sem parar o NFS** para um VMDK novo de 3 TB no vSAN do VxRail, usando espelhamento (`attach`), resilver e `split`. Pool, datasets, `fsid` e exports não mudam, e os clientes NFS não precisam remontar.

### 🔹 Ver para onde vai o espaço (dados × snapshots × filhos)

O `-o space` separa o consumo de cada dataset em dados próprios (`USEDDS`), snapshots (`USEDSNAP`) e datasets filhos (`USEDCHILD`). Assim você descobre se apagar snapshots resolve antes de pensar em expandir.

```bash
zfs list -o space -r paginas
zfs list -t snapshot -o name,used,refer,creation -s creation
```

| Coluna | Significado |
| :--- | :--- |
| `USEDSNAP` | Espaço que seria liberado se **todos** os snapshots daquele dataset fossem apagados |
| `USEDDS` | Dados "vivos" do próprio dataset |
| `USEDCHILD` | Soma dos datasets filhos |
| `USED` do snapshot | Só o espaço **exclusivo** dele. Blocos compartilhados entre vários snapshots não aparecem aqui |

Um `REFER` que salta muito de um snapshot para o seguinte indica uma gravação grande naquele intervalo (cópia, restauração ou incidente).

### 🔹 Simular quanto um `destroy` de snapshots libera

O `-n` faz um dry-run e o `-v` lista o que seria apagado e o total recuperado. O `%` define um intervalo **inclusivo**, em ordem de criação.

```bash
zfs destroy -nv paginas/sie@backup-diario-20231101%zfs-auto-snap_daily-2025-12-31-0925
```

⚠️ **Atenção:** sem o `-n`, o comando apaga de fato e não tem volta. Sempre rode o dry-run antes.

### 🔹 Proteger snapshots importantes com `hold` (ex.: evidência de incidente)

Um snapshot com *hold* não pode ser destruído, nem manualmente, nem pelas rotações automáticas (`zfs-auto-snapshot`, scripts de backup), nem por um `destroy` com intervalo `%`. Use para preservar os snapshots do dia de um incidente de segurança.

```bash
TAG="incidente-20261009"
zfs list -H -t snapshot -o name -r paginas | grep -E '2026-?10-?(0[89]|10)' | xargs -n1 zfs hold "$TAG"
zfs list -H -t snapshot -o name -r paginas | grep -E '2026-?10-?(0[89]|10)' | xargs zfs holds   # conferir
```

Para liberar quando o caso for encerrado:

```bash
zfs list -H -t snapshot -o name -r paginas | grep -E '2026-?10-?(0[89]|10)' | xargs -n1 zfs release incidente-20261009
```

### 🔹 Descobrir em qual disco o pool está de verdade

Nunca presuma que o pool está no disco que aparece no vCenter. O `-P` mostra o caminho completo do vdev.

```bash
zpool status -P paginas
lsblk -o NAME,SIZE,TYPE,TRAN,MODEL,SERIAL
blkid /dev/sd*                      # discos com TYPE="zfs_member" e LABEL do pool
ls -l /dev/disk/by-id/ | grep -v part
```

| Pista | Interpretação |
| :--- | :--- |
| `TRAN=spi`, `MODEL=Virtual disk` | VMDK entregue pelo hypervisor (aparece no vCenter) |
| `TRAN=iscsi`, vários discos com o **mesmo SERIAL/WWN** | Um único LUN visto por vários caminhos (o vCenter **não** enxerga LUN montado dentro do guest) |
| `zfs_member` com o mesmo `UUID` do pool, mas fora do `zpool status` | Label órfão de um vdev antigo (sobra de migração anterior) |

### 🔹 Verificar iSCSI e multipath

Se o pool estiver em LUN iSCSI, confirme se está usando o device do multipath (`/dev/mapper/...` ou `dm-uuid-mpath-...`) ou um caminho cru (`/dev/sdX`).

```bash
iscsiadm -m session                 # sessões/caminhos ativos
multipath -ll; ls /dev/mapper/      # só "control" = multipath inativo
```

⚠️ **Cuidado:** pool sobre um `/dev/sdX` de um LUN com vários caminhos significa **caminho único**. Uma falha de porta ou de controladora do storage derruba o pool e todo o serviço NFS.

### 🔹 Ler o histórico administrativo do pool

O `zpool history` registra todo `create`, `attach`, `detach`, `replace` e `online`, o que permite reconstruir migrações anteriores. O `-i` inclui eventos internos e o `-l` mostra usuário e host.

```bash
zpool history paginas | grep -Ei 'create|attach|detach|replace|split|add|online'
zpool history -il paginas | awk '$1 >= "2026-10-05"' | grep -Ev 'snapshot|destroy paginas.*@'
```

### 🔹 Medir a carga de I/O antes de começar

O resilver disputa I/O com a produção. Meça a carga atual para escolher o horário certo.

```bash
zpool iostat -v paginas 2 5         # I/O por vdev (confirma qual disco está ativo)
iostat -x sdd 2 3                   # %util e await do disco de origem
```

### 🔹 Adicionar o disco novo e detectá-lo sem reboot

Depois de adicionar o disco na VM pelo vCenter (com a VM **sem snapshots**), force o rescan dos controladores SCSI.

```bash
for h in /sys/class/scsi_host/host*/scan; do echo "- - -" > $h; done
lsblk -o NAME,SIZE,TRAN,MODEL /dev/sdg
blkid /dev/sdg                      # esperado: vazio
wipefs -n /dev/sdg                  # esperado: nenhuma assinatura
ls -l /dev/disk/by-path/ | grep -w sdg
```

Use o caminho `by-path` no ZFS. O nome `sdX` pode mudar a cada boot. O VMDK só aparece em `/dev/disk/by-id` se a VM tiver `disk.EnableUUID=TRUE`.

### 🔹 `attach`, `replace` ou `add`?

| Comando | O que faz | Para migrar disco |
| :--- | :--- | :--- |
| `zpool attach` | Transforma o vdev em **espelho** e copia tudo para o disco novo | ✅ Recomendado: depois você escolhe quando separar |
| `zpool replace` | Espelho temporário que **remove o disco antigo sozinho** ao final | ✅ Válido, mas não deixa cópia de rollback |
| `zpool add` | Cria um **stripe** (soma capacidade, sem redundância) | ❌ Errado: perder qualquer disco perde o pool, e o vdev não sai de forma limpa |

### 🔹 Espelhar o pool no disco novo (`attach`)

O comando retorna em segundos. O resilver roda **no kernel**, então dá para fechar o terminal e não é preciso `nohup`/`tmux`. Se o servidor reiniciar, o resilver retoma sozinho.

```bash
zpool attach paginas /dev/sdd /dev/disk/by-path/pci-0000:00:10.0-scsi-0:0:2:0
zpool status paginas
```

Durante o resilver, as escritas novas já vão para os dois discos. O espelho fica com o tamanho do **menor** disco até o disco antigo sair.

### 🔹 Entender o progresso do resilver

```text
scan: resilver in progress since Mon Oct 19 23:14:04 2026
      82.7G scanned at 2.85G/s, 111M issued at 3.82M/s, 1.75T total
      110M resilvered, 0.01% done, no estimated completion time
```

| Campo | Significado |
| :--- | :--- |
| `scanned` | Varredura de metadados para mapear o que copiar (rápida, mas pesada em leitura) |
| `issued` | Cópia efetiva, feita em ordem sequencial depois do mapeamento |
| `resilvered` | Volume já gravado no disco novo |
| Taxas `at X/s` | **Média desde o início**, não a velocidade atual |
| `no estimated completion time` | Normal no começo, enquanto ainda não há taxa de cópia |

A linha `since` mostra quando o resilver **atual** começou. Use-a para confirmar que um resilver foi reiniciado.

### 🔹 Reduzir a prioridade ou pausar o resilver (tunables)

Se a produção sofrer, ajuste os parâmetros do módulo em tempo real. Eles valem até o próximo reboot. **Sempre salve os valores originais antes.**

```bash
cd /sys/module/zfs/parameters
for p in zfs_resilver_min_time_ms zfs_vdev_scrub_max_active zfs_scan_vdev_limit; do echo "$p=$(cat $p)"; done | tee /root/zfs-tunables-orig.txt

echo 500 > zfs_resilver_min_time_ms      # padrão 3000: menos tempo de resilver por txg
echo 1048576 > zfs_scan_vdev_limit       # menos I/O em voo por vdev
echo 1 > zfs_scan_suspend_progress       # PAUSA o scan/resilver (progresso preservado)
```

| Parâmetro | Efeito |
| :--- | :--- |
| `zfs_resilver_min_time_ms` | Tempo mínimo de resilver por txg. Menor = mais gentil com a produção; maior (ex.: 5000) = mais rápido à noite |
| `zfs_vdev_scrub_max_active` | I/Os simultâneos de scrub/resilver por vdev |
| `zfs_scan_vdev_limit` | Bytes de scan em voo por vdev |
| `zfs_scan_suspend_progress` | `1` congela o scan; `0` retoma |
| `zfs_resilver_disable_defer` | `1` faz um novo `attach` **reiniciar** o resilver em vez de enfileirá-lo |

Para restaurar e retomar:

```bash
cd /sys/module/zfs/parameters
while IFS='=' read -r p v; do echo "$v" > "$p"; done < /root/zfs-tunables-orig.txt
echo 0 > zfs_scan_suspend_progress
```

⚠️ **Atenção:** com o scan suspenso, o `zpool status` continua mostrando "resilver in progress". Para confirmar que parou, compare o `scanned` entre duas leituras.

### 🔹 Cancelar a migração e o "resilver fantasma"

O `detach` do disco novo cancela o espelho e o pool volta a ser só o disco original.

```bash
zpool detach paginas /dev/disk/by-path/pci-0000:00:10.0-scsi-0:0:2:0
```

⚠️ **Pegadinha real:** quando o `detach` acontece **durante a fase de varredura**, o scan continua rodando sozinho, mesmo sem disco de destino, e segue saturando o disco de origem (os sites pararam de responder). Congele na hora:

```bash
echo 1 > /sys/module/zfs/parameters/zfs_scan_suspend_progress
```

Se o pool tiver `feature@resilver_defer` ativo, um novo `attach` **não** substitui esse scan pendente: o resilver novo fica na fila. Para retomar corretamente:

```bash
zpool get feature@resilver_defer paginas
cd /sys/module/zfs/parameters
echo 1 > zfs_resilver_disable_defer
echo 0 > zfs_scan_suspend_progress
zpool attach paginas /dev/sdd /dev/disk/by-path/pci-0000:00:10.0-scsi-0:0:2:0   # logo em seguida
zpool status paginas | grep since                                                # deve mostrar o horário de agora
```

Ao terminar a migração, volte o `zfs_resilver_disable_defer` para `0`.

### 🔹 Limpar um disco que já teve label ZFS

Num `attach` em disco inteiro, o ZFS cria partições (`sdg1` com os dados e `sdg9` reservada de 8M). O label fica na **partição**, então o `labelclear` no disco cru falha com `failed to clear label`.

```bash
zpool status -P paginas | grep -q 'scsi-0:0:2:0' && echo "PARE: disco no pool" || echo "ok"
zpool labelclear -f /dev/sdg1
wipefs -a /dev/sdg1
wipefs -a /dev/sdg                  # remove GPT primária, de backup e PMBR
partprobe /dev/sdg; udevadm settle
lsblk -o NAME,SIZE,FSTYPE,LABEL /dev/sdg   # esperado: só o disco, sem partições
blkid /dev/sdg*                            # esperado: vazio
```

Se ainda sobrar partição: `sgdisk --zap-all /dev/sdg`.

⚠️ **Cuidado:** confirme sempre que o disco **não** está no pool antes de qualquer `labelclear`/`wipefs`.

### 🔹 Finalizar: separar o disco antigo (`split`) e crescer o pool

Só depois de `resilvered ... with 0 errors`. O `split` tira o disco antigo do pool e o transforma em um **pool separado, consistente e importável** (não importado). Ele serve de rollback e de cópia de evidência. O `detach` só removeria, sem deixar essa cópia.

```bash
zpool split paginas paginas_emc_old /dev/sdd
zpool online -e paginas /dev/disk/by-path/pci-0000:00:10.0-scsi-0:0:2:0   # usa os 3T
zpool list paginas
zfs list -r paginas
```

Para voltar atrás em caso de problema: `zpool import paginas_emc_old`.

### 🔹 Validar o NFS após a migração

```bash
exportfs -v                         # exports inalterados
zfs get sharenfs,quota,refquota -r paginas
df -h /paginas /paginas/web         # e também num cliente NFS, sem remontar
```

Se houver `quota`/`refquota`, o cliente continua vendo o limite da quota, não o espaço novo do pool.

### 🔹 Desligar o iSCSI antigo (após o período de segurança)

Depois de alguns dias estáveis e com backup validado:

```bash
iscsiadm -m node -u                 # logout de todas as sessões
iscsiadm -m node -o delete          # não reconectar no boot
```

Em seguida, peça o desmapeamento do LUN no storage e remova a NIC iSCSI da VM, se ela não tiver outro uso.

## 10. <span id="boas-praticas">🧭 Boas Práticas</span>

### 🔹 Nunca use RAID de hardware sob o ZFS
O ZFS precisa de acesso direto aos discos para gerenciar redundância e integridade — controladoras RAID de hardware escondem o estado real dos discos e quebram a autocorreção do ZFS. Configure a controladora em modo HBA/passthrough.

### 🔹 Agende scrubs regulares
Rode `zpool scrub` periodicamente (ex.: semanal, via cron) — é o mecanismo que detecta e corrige *bit rot* antes que ele vire perda de dados real.

### 🔹 Mantenha espaço livre no pool
Pools ZFS acima de ~80–90% de ocupação sofrem degradação de performance perceptível, já que o alocador de blocos passa a ter cada vez menos opções livres contíguas.

### 🔹 Prefira `send`/`receive` a cópias de arquivo para backup
Além de mais rápido (transmite só blocos alterados em modo incremental), preserva metadados e é a forma nativa de replicar dados entre hosts ZFS com consistência de snapshot.

### 🔹 `autoexpand=on` não é retroativo
Ele só expande o vdev em reação a um evento de crescimento do device *enquanto o pool está importado*. Se o disco cresceu com o pool exportado (ou antes de o pool ficar visível por algum outro motivo, como uma GPT inconsistente), o `autoexpand` sozinho não resolve — force com `zpool online -e nome-pool <device>` depois de confirmar o import.

### 🔹 ZFS não usa `/etc/fstab`
Datasets montam sozinhos pela propriedade `mountpoint`, via `zfs-mount.service`/`zfs-import.target` no boot. Não estranhar a ausência de uma entrada de `/opt` (ou qualquer outro ponto de montagem ZFS) no `fstab` — isso é o comportamento normal, não sinal de que alguém desmontou manualmente ou trocou o filesystem.

### 🔹 GPT inconsistente pode esconder um pool do `zpool import`
Se `blkid` mostra `TYPE="zfs_member"` numa partição mas `zpool import` não acha nenhum pool, desconfie da tabela de partições antes de considerar o disco "livre" para reformatar — rode `parted <disco> print free`, corrija (`Fix`) se necessário, e só então reavalie. Formatar um disco por achar que o pool "não existe mais" sem checar isso primeiro pode destruir dados que só pareciam inacessíveis.

### 🔹 Confirme onde o pool está antes de expandir
O disco visível no hypervisor pode não ser o que o pool usa (LUN iSCSI montado no guest não aparece no vCenter, e labels órfãos de migrações antigas confundem). `zpool status -P` é a fonte da verdade.

### 🔹 Migrar disco: `attach` + `split`, nunca `add`
`attach` + `split` deixa o disco antigo como pool importável para rollback; `replace` é válido mas não deixa cópia; `add` num pool de vdev único cria stripe sem redundância e o vdev não sai mais de forma limpa.

### 🔹 Resilver em storage lento: fora do expediente e com a pausa pronta
Tenha à mão `zfs_resilver_min_time_ms` e `zfs_scan_suspend_progress` (salvando os valores originais) antes de começar — o resilver pode derrubar a produção em minutos.

### 🔹 `detach` no meio da varredura deixa um scan fantasma
O scan continua lendo o disco de origem mesmo sem destino. Congele com `zfs_scan_suspend_progress=1`; e, se `feature@resilver_defer` estiver ativo, use `zfs_resilver_disable_defer=1` para o próximo `attach` reiniciar o resilver em vez de enfileirá-lo.

### 🔹 Use nomes de dispositivo estáveis
Prefira `/dev/disk/by-path` ou `/dev/disk/by-id` a `/dev/sdX` (em VMware, habilite `disk.EnableUUID=TRUE`). Em LUN com vários caminhos, use o device do multipath — nunca um caminho cru.

### 🔹 `zfs hold` em snapshots de incidente
Aplique o hold antes que a rotação automática apague a evidência; ele também sobrevive a `destroy` com intervalo `%`.

## 11. <span id="referencias">📚 Referências</span>

- [OpenZFS Documentation — Documentação oficial](https://openzfs.github.io/openzfs-docs/)
