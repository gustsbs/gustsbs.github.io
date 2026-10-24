# 📂 NFS - Comandos Rápidos

Guia prático de administração de servidores NFS no Linux: onde os exports são definidos, como identificar quem monta o quê (incluindo clusters Kubernetes), como avaliar e aplicar `root_squash` e como diagnosticar lentidão e travamentos. Complementa o caso real de migração de pool em [zfs.md](zfs.md), já que o servidor do exemplo exporta datasets ZFS.

## 📑 Índice

1. [Exports: Consultar e Aplicar](#exports)
2. [Exports Definidos pelo ZFS (`sharenfs`)](#sharenfs)
3. [Identificar os Clientes](#clientes)
4. [Clientes Kubernetes](#kubernetes)
5. [Avaliar e Aplicar `root_squash`](#root-squash)
6. [Diagnóstico de Lentidão e Travamentos](#diagnostico)
7. [Boas Práticas e Pegadinhas](#boas-praticas)

---

## 1. <span id="exports">📤 Exports: Consultar e Aplicar</span>

### 🔹 Ver o que está exportado agora

O `exportfs -v` mostra os exports **ativos**, com todas as opções efetivas, inclusive as padrão que não aparecem no arquivo de configuração.

```bash
exportfs -v
```

```text
/paginas      10.0.0.0/16(sync,wdelay,hide,no_subtree_check,sec=sys,rw,secure,no_root_squash,no_all_squash)
```

### 🔹 Descobrir onde os exports são definidos

Os exports podem vir do `/etc/exports`, de arquivos em `/etc/exports.d/` ou da propriedade `sharenfs` do ZFS. Se o `/etc/exports` estiver vazio e mesmo assim houver exports, a origem é o ZFS.

```bash
grep -v '^#' /etc/exports /etc/exports.d/*.exports 2>/dev/null
zfs get -r -t filesystem sharenfs                 # -t filesystem: ignora os snapshots na saída
```

### 🔹 Aplicar mudanças sem reiniciar o serviço

O `exportfs -ra` relê a configuração e reexporta. Os clientes já montados **não caem**, a menos que deixem de estar autorizados.

```bash
cp -a /etc/exports /etc/exports.bak-$(date +%F)
exportfs -ra
exportfs -v
```

### 🔹 Opções mais importantes

| Opção | Efeito |
| :--- | :--- |
| `rw` / `ro` | Leitura e escrita / somente leitura |
| `sync` / `async` | `sync` só confirma a escrita depois de gravada em disco. É mais seguro e é o padrão |
| `root_squash` | O root do cliente vira `nobody` no servidor. **É o padrão e o recomendado** |
| `no_root_squash` | O root do cliente é root no servidor. Qualquer cliente comprometido pode alterar tudo |
| `all_squash` | Todos os usuários viram `anonuid`/`anongid`. Útil para áreas públicas |
| `no_subtree_check` | Desliga a verificação de subárvore. Recomendado, evita erros `stale file handle` |
| `hide` / `crossmnt` | Se filesystems filhos ficam visíveis (ou não) através do export pai |

---

## 2. <span id="sharenfs">🗄️ Exports Definidos pelo ZFS (`sharenfs`)</span>

### 🔹 Sintaxe do `sharenfs` no Linux

No Linux, o ZFS traduz o `sharenfs` para o `exportfs`. Redes levam `@` e vários hosts são separados por `:`.

```bash
zfs get sharenfs pool/dataset
zfs set sharenfs="rw=@10.0.53.0/27:@10.0.0.3/32,no_root_squash" pool/dataset   # reexporta na hora
zfs set sharenfs=off pool/dataset                                               # deixa de exportar
```

### 🔹 Limitação: as opções valem para todos os hosts

No `sharenfs`, opções como `no_root_squash` se aplicam a **todos** os hosts da lista. Não dá para ter `root_squash` para o cluster e `no_root_squash` para o servidor de FTP no mesmo dataset. Para regras por cliente, use o `/etc/exports`.

### 🔹 Migrar do `sharenfs` para o `/etc/exports` sem derrubar clientes

Escreva o `/etc/exports` primeiro, carregue e só então desligue o `sharenfs`. Assim o export nunca deixa de existir.

```bash
# 1) Guardar o estado atual (é o rollback)
zfs get -H -o name,value -t filesystem -r sharenfs pool | tee /root/sharenfs-orig.txt

# 2) Escrever o /etc/exports com regras por cliente
cat > /etc/exports <<'EOF'
/paginas      10.0.53.0/27(rw,sync,no_subtree_check,root_squash) \
              10.0.10.202(rw,sync,no_subtree_check,no_root_squash)
/paginas/web  10.0.53.0/27(rw,sync,no_subtree_check,root_squash)
EOF

# 3) Carregar e desligar o sharenfs
exportfs -ra
zfs set sharenfs=off pool pool/web
exportfs -v
```

Rollback: reaplicar os valores de `/root/sharenfs-orig.txt` com `zfs set sharenfs=...` e rodar `exportfs -ra`.

---

## 3. <span id="clientes">🔍 Identificar os Clientes</span>

### 🔹 Quem está conectado agora

Lista os IPs com conexão TCP ativa na porta 2049. Funciona para NFSv3 e NFSv4.

```bash
ss -tn state established '( sport = :2049 )' | awk 'NR>1{split($4,a,":");print a[1]}' | sort -u
```

### 🔹 O que cada cliente monta

```bash
showmount -a --no-headers | sort -u                          # só NFSv3
grep -H -E 'address|name' /proc/fs/nfsd/clients/*/info       # NFSv4, kernel 5.3+
nfsstat -s | head -5                                         # proporção v3/v4
findmnt -t nfs,nfs4                                          # rodar no próprio cliente
```

### 🔹 Dar nome aos IPs

```bash
for ip in 10.0.0.3 10.0.10.202; do printf "%-15s %s\n" $ip "$(getent hosts $ip | awk '{print $2}')"; done
```

IP sem DNS reverso: busque pelo IP no vCenter/Proxmox (a busca aceita IP e encontra a VM) ou no inventário.

### 🔹 Registrar clientes por alguns dias antes de restringir

Uma foto do `ss` só mostra quem está conectado **agora**. Rotinas de backup, servidores de FTP e automounts podem aparecer só de vez em quando. Registre por uma semana antes de mexer nos exports:

```bash
cat > /etc/cron.d/nfs-clientes <<'EOF'
*/10 * * * * root ss -tn state established '( sport = :2049 )' | awk 'NR>1{split($4,a,":");print a[1]}' >> /var/log/nfs-clientes.log
EOF
# depois de ~7 dias:
sort /var/log/nfs-clientes.log | uniq -c | sort -rn
```

---

## 4. <span id="kubernetes">☸️ Clientes Kubernetes</span>

### 🔹 Quem monta NFS no cluster (volumes inline e PV/PVC)

Os pods podem montar NFS de duas formas: **direto no spec** (`volumes[].nfs`, comum em manifests legados) ou por **PV/PVC**. Um filtro que olhe só PVC deixa de fora os volumes inline.

```bash
# Volumes NFS inline, agrupados por servidor
kubectl get pods -A -o json | jq -r '
  .items[] | . as $p | .spec.volumes[]? | select(.nfs)
  | [.nfs.server, $p.metadata.namespace, ($p.metadata.name|sub("-[a-z0-9]+-[a-z0-9]+$";"")), .nfs.path] | @tsv' | sort -u | column -t

# PVs NFS (in-tree e csi-driver-nfs)
kubectl get pv -o json | jq -r '.items[] | select(.spec.nfs or .spec.csi.volumeAttributes.server)
  | [.metadata.name, .spec.claimRef.namespace, (.spec.nfs.server // .spec.csi.volumeAttributes.server), (.spec.nfs.path // .spec.csi.volumeAttributes.share)] | @tsv'
```

Bônus: a primeira listagem mostra **todos** os servidores NFS que o cluster usa. É um ótimo inventário para planejar manutenção e endurecimento.

### 🔹 Liberar a faixa dos nós, não IPs individuais

Quem monta o NFS é o **nó** onde o pod está rodando. Liberar só os IPs conectados no momento quebra o pod quando ele for reagendado em outro worker (fica em `ContainerCreating` com `access denied by server`). Workers recriados também podem vir com outro IP.

```bash
kubectl get nodes -o wide                                                       # IPs dos nós
kubectl get nodes -o custom-columns=NAME:.metadata.name,TAINTS:.spec.taints[*].key   # control-planes sem taint também rodam pods
```

Libere a sub-rede dos nós (ex.: `10.0.53.0/27`), reservada para o cluster.

---

## 5. <span id="root-squash">🛡️ Avaliar e Aplicar `root_squash`</span>

### 🔹 Por que importa

Com `no_root_squash`, **qualquer processo root em qualquer cliente autorizado** tem root no export. Num cluster, basta um container comprometido (ex.: WordPress desatualizado) rodando como root para alterar os arquivos de todos os sites daquele servidor.

### 🔹 1) Com qual UID os arquivos são gravados

No servidor, veja quem é o dono dos arquivos recentes. Se quase tudo for `www-data`, quem grava não é root.

```bash
find /paginas/sites/exemplo -newermt '-30 days' -printf '%u\n' 2>/dev/null | sort | uniq -c
find /paginas/sites/exemplo -newermt '-30 days' -user root -printf '%TY-%Tm-%Td %TH:%TM %p\n' | head
```

### 🔹 2) Com qual UID os processos do container rodam

O `runAsUser` vazio no spec não prova que o pod roda como root, porque a imagem pode definir outro usuário. Confira os processos reais. O `/proc` funciona mesmo em imagens sem `ps`:

```bash
kubectl -n NAMESPACE exec POD -- sh -c 'id; for p in /proc/[0-9]*; do printf "%s %s\n" $(stat -c %U $p) $(cat $p/comm); done 2>/dev/null | sort -u'
```

| Padrão encontrado | Compatível com `root_squash`? |
| :--- | :--- |
| Tudo como `www-data`/UID fixo (ex.: php-fpm UID 33) | ✅ Sim |
| Mestres nginx/php-fpm como root, workers como `www-data` (container único legado) | ✅ Sim. Os mestres só leem configuração fora do NFS |
| Processo que **grava** no volume como root | ❌ Precisa ser ajustado antes |

### 🔹 3) Pontos que quebram com `root_squash`

```bash
# Entrypoint que faz chown/chmod no volume (falha e pode causar CrashLoopBackOff)
kubectl -n NAMESPACE exec POD -- sh -c 'grep -rnE "chown|chmod" /entrypoint* /docker-entrypoint* /usr/local/bin/*entrypoint* 2>/dev/null'

# Arquivos sem leitura para "outros" (o root do cliente vira nobody e não consegue ler)
find /paginas/sites/exemplo ! -perm -o=r -printf '%m %u %p\n' | head
```

Efeito no dia a dia: editar arquivos via `kubectl exec` como root passa a dar `Permission denied`. Edite como o usuário da aplicação (`kubectl exec ... -- su -s /bin/sh www-data`) ou pelo fluxo de deploy.

### 🔹 4) Aplicar por etapas

Comece pelo export com menos clientes ou com os workloads já validados, teste upload e mídia na aplicação e depois estenda. Mantenha `no_root_squash` só para quem precisa de verdade (ex.: servidor de FTP ou backup), sempre por IP específico e nunca para a rede inteira.

---

## 6. <span id="diagnostico">🩺 Diagnóstico de Lentidão e Travamentos</span>

### 🔹 O problema é o NFS ou o disco por baixo?

Antes de reiniciar qualquer coisa, descubra onde está o travamento. Reiniciar o NFS com o filesystem travado pode prender o próprio restart.

```bash
timeout 5 ls /paginas >/dev/null && echo "FS LOCAL OK" || echo "FS LOCAL TRAVADO"
iostat -x 2 3                                          # %util e await do disco do export
ps -eo pid,stat,wchan:30,cmd | awk '$2 ~ /D/' | head   # processos presos em I/O (estado D)
dmesg -T | tail -40                                    # erros de SCSI/iSCSI
```

| Situação | Ação |
| :--- | :--- |
| Disco do export com `%util` ~100% | Achar e remover a carga (resilver, scrub, backup) antes de mexer no NFS |
| FS local OK, disco normal, clientes presos | `systemctl restart nfs-server` e aguardar ~90s (grace period do NFSv4) |
| FS local travado ou erros de storage no `dmesg` | **Não reinicie o NFS.** Trate primeiro o storage (iSCSI, pool) |

### 🔹 Depois do restart

Os clientes NFSv4 levam até ~90s para recuperar os locks (grace period). Não reinicie de novo nesse intervalo. Se algum servidor web continuar preso, reinicie o PHP-FPM/nginx **no cliente**, porque os workers podem ter ficado travados em I/O.

---

## 7. <span id="boas-praticas">🧭 Boas Práticas e Pegadinhas</span>

### 🔹 Nunca exporte com `no_root_squash` para uma rede inteira
`rw,no_root_squash` para uma `/16` dá root nos dados a centenas de máquinas. Restrinja aos clientes reais e use `root_squash` por padrão.

### 🔹 Mapeie os clientes antes de restringir
Combine `ss`, `showmount` e o inventário do Kubernetes, e registre as conexões por alguns dias. Um cliente esquecido (backup, FTP) só aparece quando falhar.

### 🔹 Em Kubernetes, pense em nós, não em pods
Libere a sub-rede dos workers. Os pods mudam de nó a todo momento e os nós podem ser recriados com outro IP.

### 🔹 `sharenfs` é prático, mas não faz regra por cliente
Para opções diferentes por host, migre para o `/etc/exports`, carregando antes e desligando o `sharenfs` depois, para não haver janela sem export.

### 🔹 Mudanças no export são online
Tanto o `exportfs -ra` quanto o `zfs set sharenfs` aplicam na hora, sem desmontar os clientes autorizados. Expandir o filesystem por baixo (ex.: crescer um pool ZFS) também não exige nenhuma ação nos exports nem nos clientes.

### 🔹 Banco de dados em NFS, só com muito cuidado
Bancos são sensíveis a latência e locks. Confira em qual storage os PVs de bancos estão antes de qualquer manutenção pesada no NFS.
