# 💠 PowerShell - Comandos Rápidos

Instalação e uso de ferramentas de linha de comando no Windows pelo PowerShell: gerenciador de pacotes winget, AWS CLI v2 (inclusive contra um Ceph RGW/S3 on-premises) e WSL2. Para os comandos de Git, veja [git.md](git.md); para o Ceph em si, [ceph.md](ceph.md).

## 📑 Índice
1. [Primeiros Passos no PowerShell](#primeiros-passos)
2. [Gerenciador de Pacotes (winget)](#winget)
3. [Instalação da AWS CLI v2](#awscli-instalacao)
4. [AWS CLI com Ceph RGW (S3 on-premises)](#awscli-ceph)
5. [WSL2 (Linux no Windows)](#wsl2)
6. [Sintaxe: PowerShell x Bash](#sintaxe)
7. [Boas Práticas e Pegadinhas](#boas-praticas)
8. [Referências](#referencias)
---

## 1. <span id="primeiros-passos">🖥️ Primeiros Passos no PowerShell</span>

Conferências básicas antes de instalar qualquer ferramenta: versão do PowerShell, se a sessão tem privilégio de administrador e onde um comando está instalado.

### 🔹 Verificar a versão do PowerShell
O Windows já vem com o Windows PowerShell 5.1; o PowerShell 7 (`pwsh`) é instalado à parte e convive com ele.
```powershell
$PSVersionTable.PSVersion
```

### 🔹 Saber se a sessão está como administrador
Instalações com `msiexec` e o `wsl --install` pedem privilégio de administrador. Retorna `True` quando a janela foi aberta com **"Executar como administrador"**.
```powershell
([Security.Principal.WindowsPrincipal][Security.Principal.WindowsIdentity]::GetCurrent()).IsInRole([Security.Principal.WindowsBuiltInRole]::Administrator)
```

### 🔹 Descobrir onde um comando está (equivalente ao `which`)
Útil para confirmar se o PATH já enxerga o programa recém-instalado.
```powershell
Get-Command aws
Get-Command winget -ErrorAction SilentlyContinue
```

### 🔹 Recarregar o PATH sem fechar a janela
Depois de uma instalação, o PATH só vale para janelas novas. Este comando relê o PATH da máquina e do usuário na sessão atual.
```powershell
$env:Path = [Environment]::GetEnvironmentVariable("Path","Machine") + ";" + [Environment]::GetEnvironmentVariable("Path","User")
```

## 2. <span id="winget">📦 Gerenciador de Pacotes (winget)</span>

O `winget` faz parte do **App Installer** (Instalador de Aplicativo) da Microsoft. Vem por padrão no Windows 11 e nos Windows 10 atualizados, mas pode faltar em instalações novas, corporativas ou LTSC — nesse caso aparece o erro `O termo 'winget' não é reconhecido como nome de cmdlet...`.

### 🔹 Verificar se o winget está disponível
```powershell
winget --version
```

### 🔹 Registrar o App Installer que já está instalado
Em contas novas o pacote às vezes existe, mas ainda não foi registrado para o usuário. Tente este comando antes de baixar qualquer coisa e abra um PowerShell novo em seguida.
```powershell
Add-AppxPackage -RegisterByFamilyName -MainPackage Microsoft.DesktopAppInstaller_8wekyb3d8bbwe
```

### 🔹 Instalar o App Installer (winget) pelo PowerShell
Baixa o pacote oficial e instala para o usuário atual. Se reclamar de dependência (`Microsoft.VCLibs` ou `Microsoft.UI.Xaml`), instale o **"Instalador de Aplicativo"** pela Microsoft Store, que traz as dependências junto.
```powershell
Invoke-WebRequest https://aka.ms/getwinget -OutFile $env:TEMP\winget.msixbundle
Add-AppxPackage $env:TEMP\winget.msixbundle
```

### 🔹 Uso básico do winget
`-e` (`--exact`) faz o `--id` casar exatamente, evitando instalar um pacote de nome parecido.
```powershell
winget search awscli                  # procurar pacotes
winget install -e --id Amazon.AWSCLI  # instalar pelo ID exato
winget list --id Amazon.AWSCLI        # ver se está instalado e a versão
winget upgrade --all                  # atualizar tudo que o winget conhece
```

## 3. <span id="awscli-instalacao">☁️ Instalação da AWS CLI v2</span>

A AWS CLI tem versão nativa para Windows. Rodando no PowerShell sobre o disco do Windows ela é bem mais rápida do que a versão do WSL lendo `/mnt/c` ou `/mnt/d` (seção 5).

### 🔹 Instalar com winget
```powershell
winget install -e --id Amazon.AWSCLI
```

### 🔹 Instalar sem winget (MSI pelo PowerShell)
Baixa o instalador oficial e instala em modo `/passive` (mostra o progresso, sem perguntas). Rode num PowerShell como administrador.
```powershell
Invoke-WebRequest https://awscli.amazonaws.com/AWSCLIV2.msi -OutFile $env:TEMP\AWSCLIV2.msi
Start-Process msiexec.exe -Wait -ArgumentList "/i `"$env:TEMP\AWSCLIV2.msi`" /passive"
```

### 🔹 Conferir a instalação
Abra um **PowerShell novo** (ou recarregue o PATH, seção 1). Se ainda não for reconhecido, chame pelo caminho completo.
```powershell
aws --version
& "C:\Program Files\Amazon\AWSCLIV2\aws.exe" --version
```

## 4. <span id="awscli-ceph">🦑 AWS CLI com Ceph RGW (S3 on-premises)</span>

O Ceph RGW fala o protocolo S3, então a AWS CLI funciona com ele apontando o `--endpoint-url` para o gateway. As versões recentes da CLI, porém, precisam de alguns ajustes para conversar com o RGW.

### 🔹 Criar um perfil para o RGW
Cada perfil guarda um par de chaves. A região é exigida pela CLI, mas o Ceph a ignora. A configuração fica em `C:\Users\<usuário>\.aws` — separada da configuração do WSL, que fica em `~/.aws` dentro do Linux.
```powershell
aws configure --profile lab-site
#   AWS Access Key ID:     <ACCESS_KEY>
#   AWS Secret Access Key: <SECRET_KEY>
#   Default region name:   us-east-1
#   Default output format: json
```

### 🔹 Ajustes obrigatórios para o Ceph RGW
Sem as duas primeiras linhas, a CLI v2 recente falha com o erro genérico `argument of type 'NoneType' is not a container or iterable`. O `path` usa a URL no formato `endpoint/bucket`, que funciona com IP e sem DNS curinga. A última linha envia 32 arquivos em paralelo (o padrão é 10).
```powershell
aws configure set request_checksum_calculation  when_required --profile lab-site
aws configure set response_checksum_validation when_required --profile lab-site
aws configure set s3.addressing_style path      --profile lab-site
aws configure set s3.max_concurrent_requests 32 --profile lab-site
aws configure list --profile lab-site
```

### 🔹 Atalho para não repetir endpoint e perfil
Uma função do PowerShell faz o papel do `S3="aws ... s3"` do bash. `@args` repassa todos os parâmetros.
```powershell
$RGW = "http://10.10.10.45"
function S3 { aws --profile lab-site --endpoint-url $RGW s3 @args }

S3 ls                            # buckets do usuário
S3 mb s3://lab-site-estatico     # criar bucket (o dono é o usuário do perfil)
S3 ls s3://lab-site-estatico/
```

### 🔹 Enviar uma pasta com `sync`
O `sync` envia só o que mudou ou falta; se cair no meio, rode o mesmo comando de novo. Tipos que a CLI não conhece (`.pbf`, `.geojson`) precisam do `--content-type` explícito, num `sync` separado.
```powershell
cd C:\Users\gustb\projetos\site-estatico
$DEST = "s3://lab-site-estatico/v1.0"

S3 sync media/ "$DEST/media/"
S3 sync overlays/ "$DEST/overlays/" --exclude "*.qmd" --exclude "*.qml" `
     --exclude "*.pbf" --exclude "*.geojson"
S3 sync overlays/ "$DEST/overlays/" --exclude "*" --include "*.pbf"     --content-type application/x-protobuf
S3 sync overlays/ "$DEST/overlays/" --exclude "*" --include "*.geojson" --content-type application/geo+json
```

### 🔹 Conferir se o envio terminou
O `--dryrun` lista o que ainda seria enviado; `Measure-Object` conta as linhas (equivalente ao `wc -l`). Resultado `0` = tudo enviado.
```powershell
(S3 sync media/ "$DEST/media/" --dryrun | Measure-Object -Line).Lines
```

### 🔹 Entender a linha de progresso do `sync`
```text
Completed 1.1 MiB/~1.1 MiB (25.0 KiB/s) with ~0 file(s) remaining (calculating...)
```
| Trecho | Significado |
| :--- | :--- |
| `1.1 MiB/~1.1 MiB` | Enviado / total conhecido até agora (o `~` indica estimativa) |
| `(25.0 KiB/s)` | Velocidade média do comando |
| `~0 file(s) remaining` | Arquivos na fila no momento |
| `(calculating...)` | Ainda percorrendo as pastas e comparando com o bucket; o total vai crescer |

## 5. <span id="wsl2">🐧 WSL2 (Linux no Windows)</span>

O WSL2 roda um kernel Linux real numa VM leve, integrado ao Windows. Os discos do Windows aparecem em `/mnt/c`, `/mnt/d` etc.

### 🔹 Instalar o WSL2 com Ubuntu
Num PowerShell como administrador. Instala o recurso, o kernel e a distribuição padrão (Ubuntu). Reinicie o computador quando pedido; na primeira abertura, o Ubuntu pede usuário e senha do Linux.
```powershell
wsl --install
```

### 🔹 Escolher outra distribuição
```powershell
wsl --list --online              # distribuições disponíveis
wsl --install -d Ubuntu-24.04
```

### 🔹 Conferir distribuições e a versão do WSL
A coluna `VERSION` precisa mostrar `2`. Se alguma estiver em `1`, converta.
```powershell
wsl --list --verbose
wsl --set-default-version 2
wsl --set-version Ubuntu-24.04 2
```

### 🔹 Atualizar, desligar e status
`--shutdown` desliga todas as distribuições (útil depois de mudar configuração ou quando o WSL trava).
```powershell
wsl --update
wsl --status
wsl --shutdown
```

### 🔹 Acessar os arquivos de um lado no outro
```powershell
explorer.exe \\wsl$\Ubuntu-24.04\home     # abrir a home do Linux no Explorer
```
```bash
cd /mnt/c/Users/gustb/projetos            # dentro do WSL: disco C: do Windows
```

## 6. <span id="sintaxe">🔤 Sintaxe: PowerShell x Bash</span>

Os comandos de ferramentas (aws, git, kubectl) são os mesmos; mudam variáveis, continuação de linha e aspas.

| Ação | Bash (Linux/WSL) | PowerShell |
| :--- | :--- | :--- |
| Definir variável | `DEST=s3://bucket/v1.0` | `$DEST = "s3://bucket/v1.0"` |
| Usar variável | `$DEST` | `$DEST` (dentro de aspas duplas também) |
| Continuar linha | `\` no fim da linha | `` ` `` (crase) no fim da linha |
| Atalho para comando | `S3="aws ... s3"` e depois `$S3 ls` | `function S3 { aws ... s3 @args }` e depois `S3 ls` |
| Padrões nos parâmetros | `'*.pbf'` (aspas simples) | `"*.pbf"` (aspas duplas) |
| Contar linhas | `\| wc -l` | `\| Measure-Object -Line` |
| Variável de ambiente | `$HOME`, `export X=1` | `$env:USERPROFILE`, `$env:X = "1"` |
| Pasta temporária | `/tmp` | `$env:TEMP` |

## 7. <span id="boas-praticas">🧭 Boas Práticas e Pegadinhas</span>

### 🔹 Instalou e o comando "não é reconhecido"
O PATH de uma janela é lido quando ela abre. Feche e abra o PowerShell (ou recarregue o PATH, seção 1) antes de concluir que a instalação falhou.

### 🔹 WSL lendo `/mnt/c` ou `/mnt/d` é lento
Cada arquivo passa pela ponte entre Windows e Linux. Com milhares de arquivos pequenos (tiles de mapa, `node_modules`), um `sync` ou `git status` fica muito lento. Prefira a ferramenta nativa do Windows sobre pastas do Windows, ou copie antes para o disco do Linux (`rsync -a pasta/ ~/pasta/`).

### 🔹 NTFS não diferencia maiúsculas de minúsculas
`mv video.MP4 video.mp4` responde que são o mesmo arquivo, e o Git costuma ignorar a troca. Já o Linux, o S3 e os servidores web diferenciam: um nome com maiúscula errada vira 404. Troque em dois passos.
```powershell
git mv media/video.MP4 media/video.tmp.mp4
git mv media/video.tmp.mp4 media/video.mp4
```

### 🔹 Boas práticas gerais
- ✅ Instale por `winget` sempre que possível: fica registrado e é atualizado com `winget upgrade --all`.
- ✅ Um perfil da AWS CLI por credencial/projeto (`--profile`), nunca chaves soltas em variáveis de ambiente ou scripts versionados.
- ✅ A configuração da AWS CLI do Windows (`C:\Users\<usuário>\.aws`) e a do WSL (`~/.aws`) são independentes: configure as duas se usar as duas.
- ✅ Mantenha repositórios Git fora de pastas sincronizadas (OneDrive, Dropbox) — ex.: `C:\Users\<usuário>\projetos` (veja [git.md](git.md)).

## 8. <span id="referencias">📚 Referências</span>

- [Use o winget para instalar e gerenciar aplicativos — Microsoft Learn](https://learn.microsoft.com/pt-br/windows/package-manager/winget/)
- [Instalar ou atualizar a AWS CLI — AWS](https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html)
- [Ceph Object Gateway — S3 API](https://docs.ceph.com/en/latest/radosgw/s3/)
- [Como instalar o Linux no Windows com o WSL — Microsoft Learn](https://learn.microsoft.com/pt-br/windows/wsl/install)
