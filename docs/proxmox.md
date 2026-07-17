# 🌐 Guia de Comandos Proxmox

Anotações de administração de cluster, máquinas virtuais, containers LXC, armazenamento, rede e rotinas de manutenção de um ambiente Proxmox VE.

> 📚 Comandos específicos de backend de armazenamento ficam em guias dedicados: [ceph.md](ceph.md) e [zfs.md](zfs.md).

## 📑 Índice

1. [Gerenciamento de Nós e Cluster](#cluster-nos)
2. [Alta Disponibilidade (HA) e Modo de Manutenção](#ha-manutencao)
3. [Gerenciamento de VMs (qm)](#gerenciamento-vms)
4. [Gerenciamento de Containers LXC (pct)](#gerenciamento-lxc)
5. [Armazenamento (pvesm)](#armazenamento)
6. [Backup e Restauração](#backup-restauracao)
7. [Rede (Networking)](#rede)
8. [Monitoramento e Logs](#monitoramento-logs)
9. [Usuários e Permissões (pveum)](#usuarios-permissoes)
10. [Boas Práticas](#boas-praticas)
---

## 1. <span id="cluster-nos">🖥️ Gerenciamento de Nós e Cluster</span>

### 🔹 Ver o status geral do cluster
Mostra quórum, número de nós ativos e a saúde geral da comunicação entre eles — o primeiro comando a rodar ao suspeitar de instabilidade no cluster.
```bash
pvecm status
```

### 🔹 Listar os nós do cluster
```bash
pvecm nodes
```

### 🔹 Adicionar um novo nó ao cluster
Executado a partir do nó novo, apontando para o IP de um nó já existente no cluster.
```bash
pvecm add ip-do-no-existente
```

### 🔹 Ajustar o número de votos esperados (quórum de emergência)
⚠️ **Use apenas em cenários de contingência**, quando nós ficaram indisponíveis permanentemente e o cluster perdeu quórum. Força o cluster a operar com menos votos que o normal.
```bash
pvecm expected 1
```

### 🔹 Atualizar os certificados SSL do cluster
```bash
pvecm updatecerts
```

## 2. <span id="ha-manutencao">🛠️ Alta Disponibilidade (HA) e Modo de Manutenção</span>

Fluxo padrão para tirar um nó de produção (atualização de kernel, troca de hardware, reboot) sem impactar as VMs/CTs sob gestão do HA.

### 🔹 Ativar o modo de manutenção em um nó
O Proxmox evacua automaticamente as VMs/CTs sob regras de Alta Disponibilidade (HA) para os demais nós saudáveis do cluster.
```bash
ha-manager crm-command node-maintenance enable pve1
```

> ℹ️ Se o cluster usa **Ceph** como storage compartilhado, também aplique as flags de manutenção do Ceph (`noout`, `noscrub`, `nodeep-scrub`) antes de evacuar o nó — ver [ceph.md#manutencao-flags](ceph.md#manutencao-flags).

### 🔹 Acompanhar a migração e o estado do HA em tempo real
```bash
watch -n 2 ha-manager status
```

### 🔹 Confirmar a "aterrissagem" das VMs no nó saudável
Lista as VMs rodando localmente no nó que recebeu a carga, confirmando que a migração foi concluída.
```bash
qm list
```

### 🔹 Desativar o modo de manutenção
Reintegra o nó ao HA do cluster, permitindo que volte a receber cargas de trabalho.
```bash
ha-manager crm-command node-maintenance disable pve1
```

> ℹ️ Lembre-se de reverter também as flags do Ceph (`unset noout`, `unset noscrub`, `unset nodeep-scrub`) — ver [ceph.md#manutencao-flags](ceph.md#manutencao-flags).

## 3. <span id="gerenciamento-vms">🖧 Gerenciamento de VMs (qm)</span>

`qm` é a CLI de gerenciamento de máquinas virtuais QEMU/KVM no Proxmox.

### 🔹 Listar VMs do nó
```bash
qm list
```

### 🔹 Ligar e desligar uma VM
`shutdown` envia um sinal ACPI, pedindo ao sistema operacional convidado para encerrar de forma limpa; `stop` força o desligamento imediato (equivalente a tirar o cabo de energia).
```bash
qm start 100
qm shutdown 100
qm stop 100
```

### 🔹 Reiniciar uma VM
```bash
qm reboot 100
```

### 🔹 Ver a configuração de uma VM
```bash
qm config 100
```

### 🔹 Alterar recursos de uma VM
```bash
qm set 100 --memory 4096 --cores 2
```

### 🔹 Redimensionar um disco virtual
Aumenta o tamanho do disco em tempo real; o sistema operacional convidado ainda precisa expandir a partição/filesystem internamente.
```bash
qm resize 100 scsi0 +10G
```

### 🔹 Clonar uma VM
`--full` faz uma cópia completa e independente dos discos; sem essa flag, o clone é *linked* e depende da VM original.
```bash
qm clone 100 101 --name minha-vm-clone --full
```

### 🔹 Migrar uma VM entre nós
`--online` realiza uma migração ao vivo (live migration), sem desligar a VM — requer storage compartilhado ou replicação configurada entre os nós.
```bash
qm migrate 100 pve2 --online
```

### 🔹 Criar e gerenciar snapshots
```bash
qm snapshot 100 antes-da-atualizacao
qm listsnapshot 100
qm rollback 100 antes-da-atualizacao
qm delsnapshot 100 antes-da-atualizacao
```

### 🔹 Acessar o console serial de uma VM
Útil para depurar uma VM que não responde via rede/RDP/SSH.
```bash
qm terminal 100
```

### 🔹 Remover uma VM permanentemente
```bash
qm destroy 100
```

## 4. <span id="gerenciamento-lxc">📦 Gerenciamento de Containers LXC (pct)</span>

`pct` é o equivalente do `qm` para containers LXC — mais leves que VMs por compartilharem o kernel do host.

### 🔹 Listar containers do nó
```bash
pct list
```

### 🔹 Ligar, desligar e reiniciar um container
```bash
pct start 200
pct shutdown 200
pct stop 200
pct reboot 200
```

### 🔹 Entrar diretamente no console de um container
Abre um shell dentro do container sem precisar de SSH — equivalente ao `docker exec -it` do mundo Docker.
```bash
pct enter 200
```

### 🔹 Executar um comando pontual dentro do container
```bash
pct exec 200 -- systemctl status nginx
```

### 🔹 Ver e alterar a configuração de um container
```bash
pct config 200
pct set 200 --memory 2048 --cores 2
```

### 🔹 Clonar um container
```bash
pct clone 200 201 --hostname container-clone
```

### 🔹 Criar e restaurar snapshots
```bash
pct snapshot 200 antes-do-deploy
pct rollback 200 antes-do-deploy
```

### 🔹 Remover um container permanentemente
```bash
pct destroy 200
```

## 5. <span id="armazenamento">💾 Armazenamento (pvesm)</span>

`pvesm` é a camada de abstração do Proxmox sobre os diferentes backends de armazenamento (LVM, diretórios locais, NFS, ZFS, Ceph/RBD, etc.). Para comandos específicos de cada backend, veja [zfs.md](zfs.md) e [ceph.md](ceph.md).

### 🔹 Ver o status de todos os storages configurados
```bash
pvesm status
```

### 🔹 Listar o conteúdo de um storage
Mostra os discos de VM, backups, ISOs e templates armazenados em um storage específico.
```bash
pvesm list local-lvm
```

### 🔹 Adicionar um novo storage ao cluster
```bash
pvesm add dir backup-local --path /mnt/backups --content backup
```

## 6. <span id="backup-restauracao">🗄️ Backup e Restauração</span>

### 🔹 Fazer backup de uma VM ou container
`--mode snapshot` realiza o backup sem desligar a VM (usando snapshot do storage); `--compress zstd` reduz o tamanho do arquivo final.
```bash
vzdump 100 --storage backup-local --mode snapshot --compress zstd
```

### 🔹 Fazer backup suspendendo a VM
Alternativa ao modo snapshot quando o storage não suporta snapshots — a VM é pausada brevemente durante o processo.
```bash
vzdump 100 --storage backup-local --mode suspend
```

### 🔹 Restaurar uma VM a partir de um backup
```bash
qmrestore /mnt/backups/vzdump-qemu-100.vma.zst 100 --storage local-lvm
```

### 🔹 Restaurar um container LXC a partir de um backup
```bash
pct restore 200 /mnt/backups/vzdump-lxc-200.tar.zst --storage local-lvm
```

## 7. <span id="rede">🌐 Rede (Networking)</span>

### 🔹 Editar a configuração de rede do nó
Bridges, bonds e VLANs são definidos neste arquivo, no formato do `ifupdown`.
```bash
nano /etc/network/interfaces
```

### 🔹 Aplicar mudanças de rede sem reiniciar o nó
Requer o pacote `ifupdown2` instalado — recarrega a configuração de rede a quente, evitando um reboot desnecessário.
```bash
ifreload -a
```

### 🔹 Ver as interfaces e bridges ativas
```bash
ip a
brctl show
```

### 🔹 Consultar a configuração de rede via API
```bash
pvesh get /nodes/pve1/network
```

## 8. <span id="monitoramento-logs">📊 Monitoramento e Logs</span>

### 🔹 Rodar um benchmark rápido do nó
Mede desempenho de CPU, disco e rede — útil como referência ao investigar lentidão ou comparar nós do cluster.
```bash
pveperf
```

### 🔹 Ver logs de serviços do Proxmox
```bash
journalctl -u pvedaemon -f
journalctl -u pve-cluster -f
```

### 🔹 Listar as tasks (tarefas) recentes do cluster
Mostra o histórico de operações executadas (migrações, backups, snapshots) com status de sucesso ou falha — útil para auditar o que aconteceu em uma janela de manutenção.
```bash
pvesh get /cluster/tasks
```

### 🔹 Acompanhar o log de uma task específica
```bash
pvenode task log UPID:pve1:00001234:...
```

## 9. <span id="usuarios-permissoes">🔐 Usuários e Permissões (pveum)</span>

### 🔹 Listar usuários
```bash
pveum user list
```

### 🔹 Criar um usuário local
```bash
pveum user add operador@pve --password
```

### 🔹 Listar papéis (roles) disponíveis
```bash
pveum role list
```

### 🔹 Conceder permissão a um usuário sobre um recurso
Associa um usuário/grupo a um papel em um caminho específico da árvore de recursos (VM, storage, pool, etc.).
```bash
pveum acl modify /vms/100 --users operador@pve --roles PVEVMAdmin
```

### 🔹 Listar realms de autenticação (LDAP/AD)
Mostra os domínios de autenticação configurados além do padrão `pve` — relevante em ambientes integrados a um diretório LDAP/Active Directory.
```bash
pveum realm list
```

## 10. <span id="boas-praticas">🧭 Boas Práticas</span>

### 🔹 Sempre tire um snapshot antes de mudanças arriscadas
Antes de atualizações de sistema operacional, migrações ou alterações estruturais em uma VM/CT, crie um snapshot (seções 3 e 4) — o rollback é ordens de magnitude mais rápido que uma restauração completa de backup.

### 🔹 Nunca reinicie um nó sem ativar o modo de manutenção
Pular a etapa `ha-manager crm-command node-maintenance enable` (seção 2) pode causar fencing e reinício abrupto de VMs em vez de uma migração controlada.

### 🔹 Monitore o quórum antes de intervenções em massa
Rodar `pvecm status` antes de desligar múltiplos nós evita perder quórum do cluster acidentalmente — sem quórum, o Proxmox bloqueia operações de gerenciamento por segurança.

### 🔹 Coordene a manutenção com o backend de armazenamento
Ao evacuar um nó (seção 2), verifique também o estado do storage compartilhado — flags de manutenção esquecidas no Ceph ou um scrub em andamento no ZFS podem mascarar problemas reais. Veja as boas práticas específicas em [ceph.md](ceph.md#boas-praticas) e [zfs.md](zfs.md#boas-praticas).
