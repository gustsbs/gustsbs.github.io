# 🔑 LDAP - Comandos Rápidos

Integração de VMs Linux (Ubuntu 24.04) à autenticação centralizada via OpenLDAP institucional — resolução de usuários/grupos via NSS e login via PAM, usando `nslcd` como daemon intermediário (projeto nss-pam-ldapd). Cobre desde o diagnóstico de rede até a criação automática de home directory no primeiro login.

## 📑 Índice
1. [Diagnóstico e Pré-requisitos](#diagnostico-pre-requisitos)
2. [Instalação de Pacotes](#instalacao-pacotes)
3. [Configuração do nslcd](#config-nslcd)
4. [NSS — Resolução de Usuários e Grupos](#nss-nsswitch)
5. [Configuração do Cliente LDAP (CLI)](#config-ldap-cli)
6. [PAM — Home Directory Automática](#pam-mkhomedir)
7. [Serviços e Validação](#servicos-validacao)
8. [SSH em Porta Alternativa](#ssh-porta-alternativa)
9. [Sudo via Grupo LDAP](#sudo-grupo-ldap)
10. [Boas Práticas e Pegadinhas](#boas-praticas)
11. [Referências](#referencias)
---

## 1. <span id="diagnostico-pre-requisitos">🔍 Diagnóstico e Pré-requisitos</span>

Antes de instalar qualquer pacote, vale confirmar que a VM alvo consegue resolver e alcançar o servidor LDAP — evita perder tempo depurando PAM quando o problema é de rede.

### 🔹 Nomes DNS do LDAP institucional
`ldap.teste.gustbrito.local` é **split-horizon**: resolve para um IP diferente dependendo de qual rede a VM está (ex.: um IP na rede interna, outro em redes externas) — sempre aponta para o nó/proxy **ativo** de um par de alta disponibilidade. Existe também um nome separado (`master.ldap.gustbrito.local`) usado **apenas por sistemas que precisam escrever** no LDAP (ex.: troca de senha via `rootpwmoddn`) — não usar esse nome como servidor principal em VMs que só fazem leitura (autenticação/NSS).

### 🔹 Confirmar resolução e alcance do servidor
```bash
getent hosts ldap.teste.gustbrito.local
nc -zv ldap.teste.gustbrito.local 389
```

⚠️ **Atenção:** bloqueios de VLAN/firewall entre segmentos de rede já causaram falhas silenciosas em outros serviços na infraestrutura (ex.: NFS). Se o `nc` não conectar, resolva a rede antes de seguir para os próximos passos.

## 2. <span id="instalacao-pacotes">📦 Instalação de Pacotes</span>

### 🔹 Não instale só o meta-pacote
Instalar apenas `apt install -y ldap-auth-client` **não garante** qual stack o `apt` vai resolver. Numa VM ele pode trazer a stack moderna (`nslcd` + `libnss-ldapd` + `libpam-ldapd`), e em outra a stack **legada** (`ldap-auth-config` + `libnss-ldap` + `libpam-ldap` — projeto `nss_ldap`/`pam_ldap`, sem daemon `nslcd`, sem `/etc/nslcd.conf`). É a stack legada que dispara aquele assistente debconf antigo com perguntas tipo "LDAP version to use" / "Does the LDAP database require login?". Para evitar a ambiguidade, peça os pacotes certos direto:
```bash
apt update
apt install -y libnss-ldapd libpam-ldapd nslcd nslcd-utils ldap-utils
```

### 🔹 Corrigir uma instalação que caiu na stack legada
Se a VM já ficou com `ldap-auth-config`/`libnss-ldap`/`libpam-ldap` instalados (sintoma: `/etc/nslcd.conf` não existe), remova antes de instalar a stack moderna:
```bash
apt purge -y ldap-auth-config libnss-ldap libpam-ldap
apt autoremove -y
apt install -y libnss-ldapd libpam-ldapd nslcd nslcd-utils ldap-utils
```

### 🔹 Pacotes dispensáveis
`nscd` não é necessário — o `nslcd` já cuida do cache. `libpam-cracklib`/`libpam-pwquality` também não são necessários para a autenticação em si (são só checagem de força de senha, opcional).

## 3. <span id="config-nslcd">⚙️ Configuração do nslcd</span>

A instalação da stack moderna abre um debconf simples com só duas perguntas — servidor e base de busca:

| Prompt | Resposta |
| :--- | :--- |
| LDAP server Uniform Resource Identifier | `ldap://ldap.teste.gustbrito.local` |
| LDAP search base | `dc=gustbrito,dc=local` |

⚠️ **Atenção:** o esquema `ldapi://` é para socket Unix **local** — não funciona apontando para um hostname remoto. Use `ldap://` (ou `ldaps://` se o proxy exigir TLS na porta 636).

### 🔹 Ajustes manuais em `/etc/nslcd.conf`
O wizard gera só um esqueleto básico. Para busca recursiva de grupos em sub-OUs, mapeamento explícito de `memberUid` e exclusão de contas de sistema da checagem LDAP, sobrescreva o arquivo:
```bash
cat > /etc/nslcd.conf << 'EOF'
uid nslcd
gid nslcd

uri ldap://ldap.teste.gustbrito.local

base dc=gustbrito,dc=local

# Garante que a busca de grupos olhe sub-OUs recursivamente
base group ou=groups,dc=gustbrito,dc=local
scope group sub

# Ignora contas de sistema na checagem LDAP
nss_initgroups_ignoreusers root,daemon,bin,sys

# Busca exata usando o login do usuário no memberUid
map group memberUid memberUid

tls_cacertfile /etc/ssl/certs/ca-certificates.crt
EOF

chmod 640 /etc/nslcd.conf
chown root:nslcd /etc/nslcd.conf
```

## 4. <span id="nss-nsswitch">🧩 NSS — Resolução de Usuários e Grupos</span>

O pacote `libnss-ldapd` pergunta, durante a instalação, quais bases do `/etc/nsswitch.conf` devem incluir `ldap` como fonte adicional. Marque **apenas**:

- [x] `passwd`
- [x] `group`
- [x] `shadow`

Deixe todo o resto desmarcado (`hosts`, `networks`, `ethers`, `protocols`, `services`, `rpc`, `netgroup`, `aliases`).

### 🔹 Resultado esperado
```
passwd:         compat systemd ldap
group:          compat systemd ldap
shadow:         compat ldap
gshadow:        files
```

Se não ficar exatamente assim, ajuste com `sed`/editor — **não sobrescreva o arquivo inteiro**, ele tem outras linhas (`hosts`, `networks` etc.) que devem continuar como estavam.

## 5. <span id="config-ldap-cli">🖥️ Configuração do Cliente LDAP (CLI)</span>

`/etc/ldap/ldap.conf` só precisa do certificado CA para as ferramentas de linha de comando (`ldapsearch`, `ldapadd` etc.):
```bash
cat > /etc/ldap/ldap.conf << 'EOF'
TLS_CACERT      /etc/ssl/certs/ca-certificates.crt
EOF
```

## 6. <span id="pam-mkhomedir">🏠 PAM — Home Directory Automática</span>

### 🔹 Criar o perfil customizado de mkhomedir
```bash
cat > /usr/share/pam-configs/my_mkhomedir << 'EOF'
Name: activate mkhomedir
Default: yes
Priority: 900
Session-Type: Additional
Session:
        required                        pam_mkhomedir.so umask=0022 skel=/etc/skel
EOF
```

### 🔹 Ativar os perfis PAM
```bash
pam-auth-update
```
Marque: `Unix authentication`, `LDAP Authentication`, `activate mkhomedir` (o perfil customizado acima), `Register user sessions in the systemd control group hierarchy`, `Inheritable Capabilities Management`. **Deixe desmarcado** `Create home directory on login` (o perfil padrão do sistema) — ver pegadinha na seção 10.

Isso gera `/etc/pam.d/common-{auth,account,password,session,session-noninteractive}` automaticamente — não editar esses arquivos na mão.

## 7. <span id="servicos-validacao">✅ Serviços e Validação</span>

```bash
systemctl restart nslcd
systemctl enable nslcd
systemctl status nslcd --no-pager
```

### 🔹 Validar resolução via NSS
```bash
getent passwd usuario.teste
getent group grupo-posix-existente
id usuario.teste
```

### 🔹 Testar login real (cria a home directory)
```bash
ssh usuario.teste@<ip-da-vm>
ls -la /home/usuario.teste
```

### 🔹 Depurar quando `getent` não retorna nada
Teste a busca direto no LDAP, sem depender do NSS, pra isolar se o problema é rede/servidor ou config local:
```bash
ldapsearch -x -H ldap://ldap.teste.gustbrito.local -b dc=gustbrito,dc=local "(uid=usuario.teste)"
```
Se isso funcionar mas `getent` não, o problema está em `nsswitch.conf`/`nslcd.conf`/PAM. Se nem isso funcionar, é rede/firewall — volte à seção 1.

## 8. <span id="ssh-porta-alternativa">🔌 SSH em Porta Alternativa</span>

Mudar a porta do SSH do padrão (22) pra outra, como parte do fechamento de hardening da VM.

### 🔹 Alterar a porta no sshd_config
```bash
cp /etc/ssh/sshd_config /etc/ssh/sshd_config.bak
sed -i -E 's/^#?Port .*/Port 1022/' /etc/ssh/sshd_config
sshd -t   # valida a sintaxe antes de reiniciar
```

### 🔹 Liberar a porta no firewall antes de reiniciar
```bash
ufw allow 1022/tcp
systemctl restart ssh
```

⚠️ **Atenção — risco de lockout:** não feche a sessão atual. Abra um segundo terminal e teste `ssh -p 1022 usuario@<ip-da-vm>` com a sessão original ainda aberta. Só depois de confirmar que a porta nova funciona é que vale remover a regra da porta 22 do firewall.

### 🔹 Pegadinha: Ubuntu 24.04 usa socket activation pro SSH
Mudar `Port` no `sshd_config` pode não ter efeito nenhum na prática — `sshd -T | grep port` mostra a porta "certa", mas quem realmente faz o bind é a unit `ssh.socket` (`systemctl status ssh` mostra `TriggeredBy: ● ssh.socket`). A diretiva `Port` do `sshd_config` é ignorada nesse modelo porque o socket já chega pronto via file descriptor, entregue pelo `systemd`.

Fix — reconfigurar o socket, não o `sshd_config`:
```bash
systemctl edit ssh.socket
```
Adicione:
```ini
[Socket]
ListenStream=
ListenStream=1022
```
(a linha vazia é necessária — `ListenStream` é cumulativo; sem ela o socket escuta na 22 *e* na 1022, em vez de trocar)
```bash
systemctl daemon-reload
systemctl restart ssh.socket
ss -tlnp | grep -E ':22|:1022'
```

Alternativa mais simples — abandonar o socket activation e deixar o `sshd.service` escutar direto, do jeito clássico:
```bash
systemctl disable --now ssh.socket
systemctl enable --now ssh.service
```

## 9. <span id="sudo-grupo-ldap">🔐 Sudo via Grupo LDAP</span>

Liberar sudo/root pra quem estiver num grupo LDAP específico, em vez de gerenciar usuário por usuário.

### 🔹 Criar a regra num arquivo dedicado
Evite editar `/etc/sudoers` direto — use um arquivo em `/etc/sudoers.d/`, mais seguro e não conflita em upgrades de pacote:
```bash
echo '%teste-admin ALL=(ALL) ALL' > /etc/sudoers.d/ldap-teste-admin
chmod 440 /etc/sudoers.d/ldap-teste-admin
visudo -c   # valida a sintaxe antes de sair da sessão root
```

### 🔹 Confirmar que o grupo resolve antes de confiar na regra
```bash
getent group teste-admin
```
Se não retornar nada, o `%teste-admin` no sudoers é **silenciosamente ignorado** — ninguém do grupo ganha sudo, sem erro nenhum. O grupo precisa estar dentro do escopo configurado na seção 3 (`base group`/`scope group sub`).

### 🔹 Testar antes de encerrar a sessão root
```bash
ssh <usuario-do-grupo>@<ip-da-vm>
sudo -l   # deve listar "(ALL) ALL"
```

### 🔹 Pegadinha grande: `nscd` quebra `id`/`sudo` sem deixar rastro óbvio
Se `getent group teste-admin` mostra o usuário certo, mas `id <usuario>` e `sudo -l -U <usuario>` só mostram o grupo primário (sem nenhum grupo suplementar) — e o log do `nslcd` (`journalctl -u nslcd`) nem registra uma consulta de grupo quando você roda `id` — o suspeito é o `nscd`. Ele intercepta `initgroups()` no nível da libc, **antes** da chamada chegar no `nsswitch.conf`/`nslcd`, e pode estar servindo um cache vazio/antigo de antes do LDAP estar bem configurado. `getent group` (enumeração completa) não passa pelo mesmo cache — por isso só ele funciona certo enquanto `id`/`sudo` continuam errados.

Diagnóstico:
```bash
dpkg -l | grep nscd
systemctl status nscd 2>/dev/null
```
Fix:
```bash
systemctl stop nscd
systemctl disable nscd
apt purge -y nscd
```
Não precisa reiniciar mais nada depois disso — teste `id <usuario>` numa sessão nova.

**O que NÃO resolve esse sintoma específico** (pra não perder tempo tentando de novo): reiniciar `nslcd`/`sshd`; adicionar uma linha `+` em `/etc/group`/`/etc/passwd` (é uma pegadinha clássica de NSS `compat` + NIS, mas não é a causa aqui — dá pra ter `compat` funcionando perfeitamente sem nenhum `+`); comparar `nsswitch.conf` ou a versão dos pacotes `nslcd`/`libnss-ldapd`/`libpam-ldapd` entre máquinas, se já estiverem idênticos.

## 10. <span id="boas-praticas">🧭 Boas Práticas e Pegadinhas</span>

### 🔹 Prefira o FQDN do proxy à IP fixo
Um servidor LDAP institucional normalmente roda em par de alta disponibilidade atrás de um nome com DNS split-horizon. Hardcodar um IP fixo no `uri` do `nslcd.conf` corre o risco de apontar para um nó específico que saia do ar num failover — prefira sempre o FQDN.

### 🔹 Nome de leitura vs. nome de escrita
Nem todo nome LDAP institucional serve para tudo — um pode ser só leitura (uso em `nslcd.conf`/autenticação) e outro reservado para operações de escrita (troca de senha via `rootpwmoddn`, por exemplo). Confirme com o time responsável pelo LDAP qual nome usar para cada finalidade antes de configurar.

### 🔹 Evite duplicar o perfil PAM de mkhomedir
Criar mais de um arquivo customizado em `/usr/share/pam-configs/` com o mesmo `pam_mkhomedir.so`, e ainda deixar marcado o perfil padrão `Create home directory on login`, faz o módulo rodar mais de uma vez em `common-session` (inofensivo — a segunda chamada é um no-op — mas redundante e confuso de depurar depois). Crie um único perfil customizado e deixe o padrão desmarcado.

### 🔹 `minimum_uid` evita lookups desnecessários pra contas de sistema
Os perfis PAM/NSS gerados normalmente incluem `minimum_uid=1000` — isso faz contas locais/de sistema (uid < 1000) pularem a consulta LDAP inteiramente, evitando login lento ou travado quando o LDAP está fora do ar.

### 🔹 Bind anônimo é aceitável para leitura
Se o servidor permitir bind anônimo ("Does the LDAP database require login? No"), não é necessário configurar `binddn`/`bindpw` no `nslcd.conf` para autenticação/NSS — só seria necessário para operações de escrita.

### 🔹 `nscd` instalado por engano não basta ignorar — tem que remover
Mesmo sem querer instalar de propósito, se `nscd` ficou presente de alguma tentativa anterior, ele quebra `initgroups()` (`id`/`sudo`) de um jeito que não aparece nem no log do `nslcd` nem no `getent group`. Ver diagnóstico e fix completos na seção 9.

## 11. <span id="referencias">📚 Referências</span>

- [nss-pam-ldapd — nslcd.conf(5) man page](https://arthurdejong.org/nss-pam-ldapd/nslcd.conf.5)
- [Debian Wiki — LDAP Client Authentication](https://wiki.debian.org/LDAP/NSS)
