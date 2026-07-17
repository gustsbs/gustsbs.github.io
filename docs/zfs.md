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
8. [Boas Práticas](#boas-praticas)
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

## 8. <span id="boas-praticas">🧭 Boas Práticas</span>

### 🔹 Nunca use RAID de hardware sob o ZFS
O ZFS precisa de acesso direto aos discos para gerenciar redundância e integridade — controladoras RAID de hardware escondem o estado real dos discos e quebram a autocorreção do ZFS. Configure a controladora em modo HBA/passthrough.

### 🔹 Agende scrubs regulares
Rode `zpool scrub` periodicamente (ex.: semanal, via cron) — é o mecanismo que detecta e corrige *bit rot* antes que ele vire perda de dados real.

### 🔹 Mantenha espaço livre no pool
Pools ZFS acima de ~80–90% de ocupação sofrem degradação de performance perceptível, já que o alocador de blocos passa a ter cada vez menos opções livres contíguas.

### 🔹 Prefira `send`/`receive` a cópias de arquivo para backup
Além de mais rápido (transmite só blocos alterados em modo incremental), preserva metadados e é a forma nativa de replicar dados entre hosts ZFS com consistência de snapshot.
