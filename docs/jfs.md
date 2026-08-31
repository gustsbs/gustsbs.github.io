# 🗃️ Guia de Comandos JFS — Partições e Redimensionamento

Anotações de administração de partições, discos e do sistema de arquivos JFS (Journaled File System, originalmente da IBM) — expansão online de partições existentes, adição de novos discos e montagem persistente. Estrutura espelhada do procedimento equivalente em [xfs.md](xfs.md), adaptada para as particularidades do JFS.

## 📑 Índice

1. [Diagnóstico Inicial](#diagnostico-inicial)
2. [Expandindo uma Partição Existente (Online)](#expandir-particao)
3. [Adicionando um Disco Novo](#disco-novo)
4. [Formatando em JFS](#formatar-jfs)
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

Cenário: a partição `/dev/sda2`, formatada em JFS, foi expandida de 50G para 100G após aumentar o disco virtual no hypervisor (Proxmox), sem downtime — o disco virtual já havia sido redimensionado antes no lado do Proxmox.

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

### 🔹 Expandir o filesystem JFS para ocupar a partição
Diferente do XFS (que tem o binário dedicado `xfs_growfs`), o JFS não possui uma ferramenta separada de crescimento — o redimensionamento é feito remontando o filesystem com a opção `resize`, e só funciona com o filesystem **montado**. `resize=0` cresce até preencher todo o espaço disponível no dispositivo/partição.
```bash
mount -o remount,resize=0 /
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
parted /dev/sdb mkpart primary 0% 100%
```

### 🔹 Avisar o kernel sobre a nova tabela de partições
Necessário para que `/dev/sdb1` apareça sem precisar reiniciar o host.
```bash
partprobe /dev/sdb
lsblk
```

## 4. <span id="formatar-jfs">🧱 Formatando em JFS</span>

### 🔹 Instalar as ferramentas de administração do JFS
O utilitário `mkfs.jfs` não vem por padrão na maioria das distribuições — faz parte do pacote `jfsutils`.
```bash
apt install jfsutils        # Debian/Ubuntu
dnf install jfsutils        # RHEL/Rocky/Alma (pode exigir o repositório EPEL)
```

### 🔹 Criar o filesystem JFS na partição
A flag `-q` (quiet) evita o prompt interativo de confirmação, útil para automação.
```bash
mkfs.jfs -q /dev/sdb1
```

### 🔹 Obter o UUID real do filesystem recém-criado
Só existe **depois** do `mkfs.jfs` — antes disso, `blkid` de uma partição vazia retorna apenas `PARTUUID` (identificador da partição em si, não do filesystem). Ver nota na seção de Pegadinhas.
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
echo 'UUID=<UUID>  /opt  jfs  defaults  0  2' >> /etc/fstab
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

### 🔹 Ver informações do superbloco do filesystem JFS
`jfs_tune -l` (equivalente ao `tune2fs -l` do ext4 ou `xfs_info` do XFS) lista os metadados gravados no superbloco.
```bash
jfs_tune -l /dev/sdb1
```

### 🔹 Sem dry run nativo para o resize
Diferente do `xfs_growfs -n`, o JFS não oferece uma simulação do `remount,resize` — confira o espaço livre disponível com `parted ... print free` ou `lsblk` antes de aplicar o resize de verdade.

### 🔹 Verificar a integridade do filesystem (somente com o disco desmontado)
`fsck.jfs` não deve rodar em um filesystem montado. Em caso de dúvida sobre corrupção, desmonte primeiro.
```bash
umount /opt
fsck.jfs /dev/sdb1
```

## 7. <span id="boas-praticas">🧭 Boas Práticas e Pegadinhas</span>

### 🔹 JFS também não encolhe
Assim como o XFS, não existe `shrink` em JFS — o `resize` só cresce o filesystem. Se uma partição precisar ficar menor, a única forma é recriar o filesystem do zero e restaurar os dados de um backup.

### 🔹 O resize exige o filesystem montado
Ao contrário do `xfs_growfs`, que é um binário dedicado, o crescimento do JFS é feito via `mount -o remount,resize=0 <ponto-de-montagem>` — se o filesystem estiver desmontado, monte-o normalmente primeiro e só então remonte com `resize`.

### 🔹 `PARTUUID` não é `UUID`
Rodar `blkid` numa partição recém-criada, ainda sem filesystem, retorna só o `PARTUUID` (identifica a partição na tabela de partições). O `UUID` real do filesystem só aparece **depois** do `mkfs.jfs`. Usar o valor errado no `/etc/fstab` impede o boot de montar a partição corretamente.

### 🔹 Sempre valide o `fstab` com `mount -a` antes de reiniciar
Um erro de sintaxe no `fstab` pode deixar o sistema preso no modo de emergência (`emergency mode`) no próximo boot, exigindo acesso ao console.

### 🔹 Prefira `UUID` a `/dev/sdX` no `fstab`
Nomes como `/dev/sdb` dependem da ordem em que o kernel detecta os discos no boot e podem mudar (especialmente após adicionar/remover discos). `UUID` é fixo e não muda.

### 🔹 `jfsutils` costuma ser menos mantido que as ferramentas de XFS/ext4
O pacote `jfsutils` tem atualizações mais espaçadas que `xfsprogs` ou `e2fsprogs`. Para novas implantações sem exigência específica de JFS, avalie se XFS ou ext4 não atendem melhor — mantenha JFS para os casos em que já é o filesystem legado em produção.

### 🔹 Confirme o redimensionamento do disco virtual no hypervisor antes de tudo
Em VMs (Proxmox, etc.), `growpart`/`resize` só têm o que expandir se o disco virtual já foi aumentado no lado do hypervisor. Verifique isso primeiro caso `growpart` não encontre espaço livre.
