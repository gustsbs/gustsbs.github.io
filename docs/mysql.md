# 🐬 MySQL/MariaDB - Comandos Rápidos

Comandos de backup (dump) e restauração de bancos de dados MySQL/MariaDB — usados, por exemplo, no fluxo de replicação de ambientes de teste a partir da produção (ver [kubernetes.md](kubernetes.md) para o mesmo fluxo aplicado à transferência de arquivos via `kubectl exec`).

## 📑 Índice

1. [Dump de Bancos de Dados](#dump)
2. [Restauração de Bancos de Dados](#restauracao)
3. [Boas Práticas](#boas-praticas)
---

## 1. <span id="dump">💾 Dump de Bancos de Dados</span>

### 🔹 Exportar um banco completo para arquivo .sql
`mysqldump` conecta no servidor, autentica e exporta o schema + os dados do banco inteiro como uma sequência de comandos SQL num arquivo texto — usado tanto para backup quanto para migrar/replicar um banco entre ambientes.
```bash
mysqldump -u root -h bd.gustbrito.local -psenha --lock-tables=false banco > banco-dump.sql
```
- `-u root` — usuário de conexão.
- `-h bd.gustbrito.local` — host do servidor MySQL/MariaDB.
- `-psenha` — senha colada direto após o `-p`, **sem espaço** (`-p senha` com espaço faz o `mysqldump` interpretar `senha` como o nome de mais um banco a exportar, não como a senha).
- `--lock-tables=false` — não trava (`LOCK TABLES`) as tabelas durante o dump. Necessário quando o usuário não tem o privilégio `LOCK TABLES`, ou quando não se quer bloquear escritas concorrentes no banco durante o processo. Ver a ressalva sobre consistência em [Boas Práticas](#boas-praticas).
- `banco` — nome do banco a exportar.
- `> banco-dump.sql` — redireciona a saída (o SQL gerado) para um arquivo local.

### 🔹 Alternativa consistente para tabelas InnoDB
Em vez de simplesmente desligar o lock, `--single-transaction` abre uma transação e tira uma "foto" consistente do banco no início do dump (via MVCC do InnoDB), sem bloquear leituras/escritas de outras conexões durante o processo — o melhor dos dois mundos quando as tabelas são InnoDB (não funciona para MyISAM, que não tem transação).
```bash
mysqldump -u root -h bd.gustbrito.local -psenha --single-transaction banco > banco-dump.sql
```

### 🔹 Exportar o banco incluindo o comando de criação (CREATE DATABASE)
Por padrão o dump não recria o banco no destino — só as tabelas dentro dele. `--databases` inclui o `CREATE DATABASE`/`USE`, útil quando o banco de destino ainda não existe.
```bash
mysqldump -u root -h bd.gustbrito.local -psenha --lock-tables=false --databases banco > banco-dump.sql
```

## 2. <span id="restauracao">♻️ Restauração de Bancos de Dados</span>

### 🔹 Importar um dump .sql para um banco existente
Processo inverso do dump: o cliente `mysql` lê o arquivo `.sql` e reexecuta os comandos SQL nele contra o banco de destino informado. O banco precisa já existir (a menos que o dump tenha sido feito com `--databases`, ver acima).
```bash
mysql -u root -h bd.gustbrito.local -psenha banco < banco-dump.sql
```

### 🔹 Criar o banco de destino antes de restaurar
Quando o dump não inclui `--databases`, crie o banco manualmente antes do `mysql < banco-dump.sql` — para WordPress/GLPI, use `utf8mb4` (charset que suporta emoji e caracteres especiais completos, diferente do `utf8` antigo do MySQL, que é na verdade um subconjunto de 3 bytes).
```sql
CREATE DATABASE banco CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
```

## 3. <span id="boas-praticas">🧭 Boas Práticas</span>

⚠️ **Cuidado:** restaurar um dump sobre um banco que já tem dados mescla/sobrescreve as tabelas existentes sem aviso nenhum — confirme sempre qual banco é o destino antes de rodar `mysql < banco-dump.sql`, principalmente fora de ambiente de teste.

| Opção | O que faz | Quando usar |
| :--- | :--- | :--- |
| `--lock-tables=false` | Não bloqueia as tabelas durante o dump | Usuário sem privilégio `LOCK TABLES`, ou quando um lock total é inaceitável — mas o dump pode sair inconsistente entre tabelas se houver escrita concorrente |
| `--single-transaction` | Tira uma foto consistente via transação (MVCC), sem bloquear outras conexões | Tabelas InnoDB — é a opção mais segura para produção |

- ✅ Evite deixar a senha em texto puro na linha de comando — ela fica visível no histórico do shell (`history`) e na lista de processos do sistema (`ps aux`, visível a qualquer usuário local). Prefira omitir o valor depois do `-p` (o `mysql`/`mysqldump` pede a senha interativamente) ou usar um arquivo `~/.my.cnf` com permissão `600`.
- ✅ Para bancos grandes, comprima o dump direto no pipe (`mysqldump ... | gzip > banco-dump.sql.gz`) e restaure com `gunzip < banco-dump.sql.gz | mysql ...` — economiza espaço em disco e tempo de transferência.
- ✅ Depois de restaurar um banco de WordPress em outro domínio (ex. produção → teste), lembre que URLs podem estar hardcoded nas tabelas (`wp_options`, `guid` dos posts) — use um `search-replace` (ex. `wp search-replace` do WP-CLI) em vez de editar `wp-config.php` sozinho.
