# 📀 Guia de Comandos XFS — Partições e Redimensionamento

Anotações de administração de partições, discos e do sistema de arquivos XFS — expansão online de partições existentes, adição de novos discos e montagem persistente. Baseado no procedimento aplicado no servidor `template-26-04-xfs`, usado como base para o hardening dos serviços (OpenLDAP, Samba, Nginx, DNS Bind — veja também [proxmox.md](proxmox.md)).

## 📑 Índice

1. [Diagnóstico Inicial](#diagnostico-inicial)
2. [Expandindo uma Partição Existente (Online)](#expandir-particao)
3. [Adicionando um Disco Novo](#disco-novo)
4. [Formatando em XFS](#formatar-xfs)
5. [Montagem Persistente via `/etc/fstab`](#montagem-persistente)
6. [Verificação e Monitoramento](#verificacao-monitoramento)
7. [Boas Práticas e Pegadinhas](#boas-praticas)
---

## 1. <span id="diagnostico-inicial">🔍 Diagnóstico Inicial</span>

Antes de qualquer alteração em disco, é essencial mapear o estado atual das partições, do espaço livre e dos filesystems montados.

### 🔹 Listar discos e partições em formato de árvore
Mostra os block devices, tamanhos, pontos de montagem e tipo de filesystem de forma resumida.
```bash
lsblk -f
```

### 🔹 Ver uso de espaço dos filesystems montados
```bash
df -hT
```

### 🔹 Ver a tabela de partições de um disco
Exibe o layout de partições (GPT/MBR) e o espaço livre não particionado ao final do disco.
```bash
parted /dev/sda print free
```

### 🔹 Consultar UUID e tipo de filesystem de uma partição
```bash
blkid /dev/sda2
```

## 2. <span id="expandir-particao">📈 Expandindo uma Partição Existente (Online)</span>

Cenário: a partição raiz `/dev/sda2` foi expandida de 50G para 100G após aumentar o disco virtual no hypervisor (Proxmox), sem downtime — o disco virtual já havia sido redimensionado antes no lado do Proxmox.

### 🔹 Confirmar que o kernel já enxerga o novo tamanho do disco
Se o disco foi expandido no hypervisor com a VM ligada, força o kernel a reler o tamanho do dispositivo sem precisar reiniciar.
```bash
echo 1 > /sys/class/block/sda/device/rescan
lsblk
```

### 🔹 Expandir a partição para ocupar o espaço livre
`growpart` redimensiona a partição em si (a tabela de partições), sem tocar no filesystem — precisa do pacote `cloud-guest-utils` (Debian/Ubuntu) ou `cloud-utils-growpart` (RHEL).
```bash
growpart /dev/sda 2
```

### 🔹 Expandir o filesystem XFS para ocupar a partição
XFS só cresce — não existe `shrink`. O comando é aplicado no ponto de montagem, não no device.
```bash
xfs_growfs /
```

### 🔹 Conferir o resultado
```bash
df -hT /
```

## 3. <span id="disco-novo">💽 Adicionando um Disco Novo</span>

Cenário: um segundo disco virtual (`/dev/sdb`, 500G) foi anexado à VM para ser dedicado a `/opt`, mantendo a raiz enxuta.

### 🔹 Confirmar que o novo disco foi reconhecido
```bash
lsblk
```

### 🔹 Criar a tabela de partições GPT
GPT é o padrão recomendado para discos acima de 2TB e ambientes modernos (UEFI), mas também funciona normalmente em discos menores.
```bash
parted /dev/sdb mklabel gpt
```

### 🔹 Criar uma partição única ocupando 100% do disco
```bash
parted /dev/sdb mkpart primary xfs 0% 100%
```

### 🔹 Avisar o kernel sobre a nova tabela de partições
Necessário para que `/dev/sdb1` apareça sem precisar reiniciar o host.
```bash
partprobe /dev/sdb
lsblk
```

## 4. <span id="formatar-xfs">🧱 Formatando em XFS</span>

### 🔹 Criar o filesystem XFS na partição
```bash
mkfs.xfs /dev/sdb1
```

### 🔹 Obter o UUID real do filesystem recém-criado
Só existe **depois** do `mkfs.xfs` — antes disso, `blkid` de uma partição vazia retorna apenas `PARTUUID` (identificador da partição em si, não do filesystem). Ver nota na seção de Pegadinhas.
```bash
blkid /dev/sdb1
```

## 5. <span id="montagem-persistente">🔗 Montagem Persistente via `/etc/fstab`</span>

Montar manualmente com `mount` não sobrevive a um reboot — a entrada precisa ir para o `/etc/fstab`, referenciando o disco pelo `UUID` (mais estável que `/dev/sdb1`, que pode mudar de letra dependendo da ordem de detecção dos discos).

### 🔹 Criar o ponto de montagem
```bash
mkdir -p /opt
```

### 🔹 Adicionar a entrada no `/etc/fstab`
Substituir `<UUID>` pelo valor obtido no `blkid` do passo anterior.
```bash
echo 'UUID=<UUID>  /opt  xfs  defaults  0  2' >> /etc/fstab
```

### 🔹 Validar o `fstab` sem reiniciar
`mount -a` monta tudo que está no `fstab` e ainda não montado — se houver erro de sintaxe, ele aparece aqui, antes do próximo boot.
```bash
mount -a
```

### 🔹 Conferir a montagem
```bash
df -hT /opt
```

## 6. <span id="verificacao-monitoramento">📊 Verificação e Monitoramento</span>

### 🔹 Ver informações detalhadas do filesystem XFS
Tamanho de bloco, quantidade de AGs (allocation groups), tamanho do log — útil para diagnóstico de performance.
```bash
xfs_info /opt
```

### 🔹 Simular uma expansão sem aplicar (dry run)
Mostra o que `xfs_growfs` faria, sem alterar nada — útil para conferir antes de rodar de verdade em produção.
```bash
xfs_growfs -n /opt
```

### 🔹 Verificar a integridade do filesystem (somente com o disco desmontado)
`xfs_repair` não pode rodar em um filesystem montado. Em caso de dúvida sobre corrupção, desmonte primeiro.
```bash
umount /opt
xfs_repair /dev/sdb1
```

## 7. <span id="boas-praticas">🧭 Boas Práticas e Pegadinhas</span>

### 🔹 XFS não encolhe
Diferente do ext4, não existe `shrink` em XFS. Se uma partição precisar ficar menor, a única forma é recriar o filesystem do zero e restaurar os dados de um backup — planeje o tamanho das partições com folga.

### 🔹 `PARTUUID` não é `UUID`
Rodar `blkid` numa partição recém-criada, ainda sem filesystem, retorna só o `PARTUUID` (identifica a partição na tabela de partições). O `UUID` real do filesystem só aparece **depois** do `mkfs.xfs`. Usar o valor errado no `/etc/fstab` impede o boot de montar a partição corretamente.

### 🔹 Sempre valide o `fstab` com `mount -a` antes de reiniciar
Um erro de sintaxe no `fstab` pode deixar o sistema preso no modo de emergência (`emergency mode`) no próximo boot, exigindo acesso ao console.

### 🔹 Prefira `UUID` a `/dev/sdX` no `fstab`
Nomes como `/dev/sdb` dependem da ordem em que o kernel detecta os discos no boot e podem mudar (especialmente após adicionar/remover discos). `UUID` é fixo e não muda.

### 🔹 Separe dados que crescem em partições dedicadas
Reservar um disco/partição dedicado (ex.: `/opt`) para dados de serviços que crescem com o tempo (bases LDAP, compartilhamentos Samba, logs) evita que o crescimento desses dados comprometa o espaço da partição raiz `/`.

### 🔹 Confirme o redimensionamento do disco virtual no hypervisor antes de tudo
Em VMs (Proxmox, etc.), `growpart`/`xfs_growfs` só têm o que expandir se o disco virtual já foi aumentado no lado do hypervisor. Verifique isso primeiro caso `growpart` não encontre espaço livre.
