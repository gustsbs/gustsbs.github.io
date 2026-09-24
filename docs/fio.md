# 🧪 Fio - Comandos Rápidos

Anotações de uso do `fio` (*Flexible I/O Tester*) para medir desempenho de armazenamento (IOPS, vazão e latência) em volumes usados por aplicações no Kubernetes. Útil para comparar backends como NFS, CephFS e Ceph S3 antes de escolher onde hospedar uma aplicação — veja também [ceph.md](ceph.md) e [kubernetes.md](kubernetes.md).

## 📑 Índice

1. [Instalação](#instalacao)
2. [Teste Padrão: Carga Aleatória Mista 70/30](#teste-padrao)
3. [Rodando o Teste Dentro do Kubernetes](#kubernetes)
4. [Lendo o Resultado](#lendo-resultado)
5. [Boas Práticas e Pegadinhas](#boas-praticas)
6. [Referências](#referencias)
---

## 1. <span id="instalacao">📦 Instalação</span>

### 🔹 Debian/Ubuntu
```bash
apt update && apt install -y fio
```

### 🔹 Alpine (imagens de contêiner como `php-fpm-alpine`)
```bash
apk add --no-cache fio
```

### 🔹 Conferir a versão instalada
```bash
fio --version
```

## 2. <span id="teste-padrao">⚙️ Teste Padrão: Carga Aleatória Mista 70/30</span>

Perfil de carga sugerido: leitura e escrita aleatórias misturadas (70% leitura / 30% escrita), blocos pequenos de 4 KiB e 4 processos concorrentes durante 60 segundos. Simula bem uma aplicação PHP (GLPI, WordPress), que lê e grava muitos arquivos pequenos.

### 🔹 Rodar o teste em um diretório montado
O `--directory` precisa apontar para dentro do volume que se quer medir (NFS, CephFS, bucket S3 montado etc.), não para o disco local do contêiner.
```bash
fio --name=nfs \
    --directory=/var/www/html/files \
    --rw=randrw \
    --rwmixread=70 \
    --bs=4k \
    --size=1G \
    --numjobs=4 \
    --runtime=60 \
    --time_based \
    --group_reporting
```

### 🔹 O que cada parâmetro faz

| Parâmetro | Função |
| :--- | :--- |
| `--name=nfs` | Nome do job; também vira prefixo dos arquivos de teste (`nfs.0.0`, `nfs.1.0`...). |
| `--directory=` | Onde os arquivos de teste são criados — define **qual volume** está sendo medido. |
| `--rw=randrw` | Leitura e escrita aleatórias misturadas (`read`, `write`, `randread`, `randwrite` são as outras opções comuns). |
| `--rwmixread=70` | Proporção de leitura na carga mista (70% leitura / 30% escrita). |
| `--bs=4k` | Tamanho do bloco de cada operação. Blocos pequenos medem IOPS; blocos grandes (`1M`) medem vazão sequencial. |
| `--size=1G` | Tamanho do arquivo de teste **por processo**. |
| `--numjobs=4` | Quantidade de processos concorrentes (4 × 1G = 4 GiB de espaço necessário). |
| `--runtime=60` + `--time_based` | Roda por 60 s, repetindo o arquivo se ele terminar antes. |
| `--group_reporting` | Soma os 4 processos em um único relatório, em vez de um bloco por job. |

### 🔹 Variante sequencial com blocos grandes
Útil para comparar com o teste acima: armazenamento de objetos (S3) tende a se sair bem melhor aqui do que com blocos de 4 KiB.
```bash
fio --name=seq --directory=/mnt/volume-teste --rw=read --bs=1M --size=2G \
    --numjobs=1 --runtime=60 --time_based --group_reporting
```

## 3. <span id="kubernetes">☸️ Rodando o Teste Dentro do Kubernetes</span>

Para medir o volume exatamente como a aplicação o enxerga, o teste roda dentro de um Pod que monta o mesmo volume.

### 🔹 Entrar em um Pod já existente da aplicação
```bash
kubectl exec -it -n glpi-teste deploy/glpi -- sh
```

### 🔹 Criar um Pod dedicado só para o teste
Permite medir qualquer volume (NFS, CephFS, bucket S3 montado) sem mexer na aplicação.
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: fio-bench
  namespace: wordpress-teste
spec:
  containers:
    - name: fio
      image: alpine:3.20
      command: ["sh", "-c", "apk add --no-cache fio && sleep infinity"]
      volumeMounts:
        - name: alvo
          mountPath: /mnt/alvo
  volumes:
    - name: alvo
      persistentVolumeClaim:
        claimName: wp-uploads
```

```bash
kubectl apply -f fio-bench.yaml
kubectl exec -it -n wordpress-teste fio-bench -- \
  fio --name=teste --directory=/mnt/alvo --rw=randrw --rwmixread=70 --bs=4k \
      --size=1G --numjobs=4 --runtime=60 --time_based --group_reporting
```

### 🔹 Remover o Pod de teste ao final
```bash
kubectl delete pod -n wordpress-teste fio-bench
```

## 4. <span id="lendo-resultado">📊 Lendo o Resultado</span>

### 🔹 Onde olhar na saída
Exemplo de saída (volume NFS), com os campos mais importantes:
```text
read: IOPS=516, BW=2066KiB/s (2115kB/s)(144MiB/71599msec)
    clat (usec): min=256, max=27707k, avg=7565.42, stdev=289753.77
     clat percentiles (usec):
     | 50.00th=[  832], 99.00th=[77071], 99.99th=[17112761]
write: IOPS=227, BW=909KiB/s (930kB/s)(63.5MiB/71599msec)
    clat (usec): min=5, max=1052, avg=26.28, stdev=20.24
cpu  : usr=0.29%, sys=0.95%
```

| Campo | Significado |
| :--- | :--- |
| `IOPS` | Operações de E/S por segundo — principal métrica para blocos pequenos. |
| `BW` | Vazão (*bandwidth*). |
| `clat` | Latência de conclusão (*completion latency*) de cada operação; `avg` é a média. |
| `clat percentiles` | Distribuição da latência: `50.00th` é a mediana, `99.00th` mostra os piores 1%. |
| `run=71599msec` | Tempo real de execução — se passar do `--runtime`, houve operações travadas. |
| `cpu usr/sys` | Uso de CPU do cliente; valores baixos indicam gargalo em rede/armazenamento, não em processamento. |

## 5. <span id="boas-praticas">🧭 Boas Práticas e Pegadinhas</span>

### 🔹 O valor `17112761` nos percentis não é um travamento de 17 s "igual" em todos os testes
É o limite da última faixa do histograma de latência do `fio`: toda latência acima de ~17,1 s cai nessa mesma faixa. Dois testes mostrando esse mesmo número não provam um gargalo comum — olhe o `max=` real de cada teste, que costuma ser diferente em cada um.

### 🔹 Apague os arquivos de teste depois
O `fio` deixa os arquivos (`<name>.<n>.0`) no diretório testado — 4 GiB no teste padrão.

⚠️ **Atenção:** em volumes de produção, apague apenas os arquivos do teste, conferindo o nome antes; nunca use curinga amplo dentro do diretório da aplicação.
```bash
ls -lh /var/www/html/files/nfs.*.0
rm /var/www/html/files/nfs.*.0
```

### 🔹 Cache infla os números de escrita
Sem `--direct=1`, as escritas passam pelo cache de memória (do kernel ou do cliente NFS/S3) — por isso a escrita em NFS pode aparecer com latências de dezenas de microssegundos, bem abaixo do que o disco realmente entrega. Para comparar volumes entre si isso é aceitável se todos forem testados igual; para medir o disco "de verdade", teste com `--direct=1` (nem todo sistema de arquivos em rede/FUSE aceita).

### 🔹 Rode mais de uma vez
Execuções seguidas no mesmo volume podem variar bastante (aquecimento de cache, concorrência na rede). Use pelo menos duas execuções e registre as duas.

### 🔹 Confira o `--directory` antes de rodar
Um volume que não montou de verdade grava no disco efêmero do contêiner e o teste mede a coisa errada. Confirme com `df -h <diretório>` que o caminho está no volume de rede esperado.
```bash
df -h /var/www/html/files
```

### 🔹 Evite horário de pico em produção
O teste gera carga real de E/S no servidor de armazenamento compartilhado — rode fora do horário de uso ou em volume de teste.

## 6. <span id="referencias">📚 Referências</span>

- [fio — Documentação oficial](https://fio.readthedocs.io/)
- [fio — Repositório no GitHub (axboe/fio)](https://github.com/axboe/fio)
- [fio — Interpretando a saída (*Interpreting the output*)](https://fio.readthedocs.io/en/latest/fio_doc.html#interpreting-the-output)
