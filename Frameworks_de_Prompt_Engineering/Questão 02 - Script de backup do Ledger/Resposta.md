# Resposta — Questão 02: Script de backup do Ledger

## Prompt

Role
Atue como uma pessoa engenheira SRE sênior, com experiência em Bash, PostgreSQL, backups e AWS em ambientes Linux.

Task 
Lorraine Baines, líder de SRE da Hill Valley Tech, precisa automatizar  o backup diário do Ledger, um PostgreSQL que roda em uma instância EC2. Escreva um script Bash para Ubuntu 22.04 LTS com estes dados:

```
Host: ledger-db.internal.hvt.io
Porta: 5432
Banco: ledger_prod
Usuário de backup: backup_user
Senha: PGPASSWORD, já populada no ambiente pelo AWS Secrets Manager
       por meio da IAM role da instância
Região AWS: us-east-1
Diretório de trabalho: /var/backups/ledger, com 80 GB livres
Tamanho médio do dump compactado: aproximadamente 12 GB
Bucket S3: hvt-ledger-backups
Log: /var/log/ledger-backup.log
```

O script deve gerar um dump SQL com pg_dump e compactá-lo com gzip em pipeline, evitando manter uma cópia SQL descompactada no disco. Deve enviar o arquivo usando aws s3 cp e, somente após upload bem-sucedido, remover do S3 os backups com mais de 30 dias, usando LastModified em UTC. Use um prefixo exclusivo ledger/ e nomes únicos identificáveis, sem remover arquivos de outras aplicações. Processe a listagem completa, com paginação automática da AWS CLI e tratamento seguro de JSON. Pode usar python3, disponível como pré-requisito explícito, para filtrar a listagem. Declare a premissa de bucket sem versionamento e explique o tratamento adicional necessário caso haja versões anteriores.

Inclua modo estrito com pipefail, validação de PGPASSWORD sem exibi-la, bloqueio com flock contra execuções simultâneas, permissões restritas, temporários exclusivos e limpeza local tanto no sucesso quanto na falha. Verifique dependências e espaço livre antes do dump, usando 24 GiB como margem inicial configurável, sem afirmar que ela garante espaço para qualquer crescimento. Verifique a integridade gzip antes do upload. Registre início, etapas, erros e término com timestamps. Preserve erros das ferramentas no log sem habilitar rastreamento que exponha segredos. Retorne zero apenas quando todas as etapas concluírem, retorne valor não zero para falhas, inclusive na retenção, e 75 para lock ocupado.

Não coloque credenciais no código ou na linha da cron. Não invente o identificador do secret ou sua estrutura. Explique que a cron precisa invocar o mecanismo existente de injeção de PGPASSWORD em cada execução.

Format
Entregue um bloco de código com o script completo ledger-backup.sh, um exemplo de cron diária às 02:00 e até seis tópicos curtos com pré-requisitos, permissões, premissas e limitações. Diferencie a checagem gzip de um teste real de restauração. Não afirme ter acessado PostgreSQL, AWS ou executado testes que não realizou.

## Modelo

GPT-6 da OpenAI.

## Output

Resposta produzida nesta sessão a partir do prompt acima. 

Salvar como `ledger-backup.sh`:

```bash
#!/bin/bash
set +x
set -Eeuo pipefail
umask 077

export PATH=/usr/local/bin:/usr/bin:/bin
export AWS_DEFAULT_REGION=us-east-1
export AWS_PAGER=""
export PGCONNECT_TIMEOUT=30

WORK_DIR=/var/backups/ledger
LOG_FILE=/var/log/ledger-backup.log
BUCKET=hvt-ledger-backups
PREFIX=ledger/
MIN_FREE_GIB=${MIN_FREE_GIB:-24}
tmp_dir=''
stage=inicializacao

# Diretório e log devem ser provisionados com permissões restritas.
exec >>"$LOG_FILE" 2>&1

log() {
    printf '[%s] %s\n' "$(date -u +%Y-%m-%dT%H:%M:%SZ)" "$*"
}

fail() {
    log "ERRO etapa=$stage: $*"
    exit 1
}

finish() {
    local rc=$?
    trap - EXIT ERR INT TERM
    set +e
    if [[ -n "$tmp_dir" ]]; then
        if ! rm -f -- "$tmp_dir/dump.sql.gz" "$tmp_dir/objects.json" \
                     "$tmp_dir/expired.txt" || ! rmdir -- "$tmp_dir"; then
            log 'ERRO ao remover temporários locais'
            if (( rc == 0 )); then rc=1; fi
        fi
    fi
    log "FIM etapa=$stage exit_code=$rc"
    exit "$rc"
}

trap finish EXIT
trap 'rc=$?; log "ERRO etapa=$stage linha=$LINENO exit_code=$rc"; exit "$rc"' ERR
trap 'log "Interrompido por SIGINT"; exit 130' INT
trap 'log "Interrompido por SIGTERM"; exit 143' TERM

log "INICIO pid=$$"
for cmd in pg_dump gzip aws python3 flock df awk mktemp date rm rmdir; do
    command -v "$cmd" >/dev/null || fail "Dependência ausente: $cmd"
done
[[ -n "${PGPASSWORD:-}" ]] || fail 'PGPASSWORD não foi injetada'
export PGPASSWORD
[[ -d "$WORK_DIR" && -w "$WORK_DIR" ]] || fail 'Diretório indisponível'
[[ "$MIN_FREE_GIB" =~ ^[1-9][0-9]{0,3}$ ]] || fail 'MIN_FREE_GIB inválido'

stage=lock
# O arquivo de lock é permanente: não removê-lo na limpeza.
exec 9>"$WORK_DIR/.backup.lock"
if flock -n -E 75 9; then
    :
else
    rc=$?
    log "Não foi possível adquirir o lock; exit_code=$rc (75=ocupado)"
    exit "$rc"
fi

stage=espaco
free_kib=$(df -Pk "$WORK_DIR" | awk 'NR == 2 {print $4}')
[[ "$free_kib" =~ ^[0-9]+$ ]] || fail 'Não foi possível medir espaço livre'
(( free_kib >= MIN_FREE_GIB * 1024 * 1024 )) || fail 'Espaço livre insuficiente'

tmp_dir=$(mktemp -d "$WORK_DIR/run.XXXXXXXX")
run_id=${tmp_dir##*/}
key="${PREFIX}ledger_prod_$(date -u +%Y%m%dT%H%M%SZ)_${run_id#run.}.sql.gz"
dump_file="$tmp_dir/dump.sql.gz"

stage=dump
log 'Iniciando pg_dump e gzip'
pg_dump --host=ledger-db.internal.hvt.io --port=5432 \
    --username=backup_user --dbname=ledger_prod --no-password \
    --format=plain | gzip -c >"$dump_file"
unset PGPASSWORD

stage=validacao
gzip -t "$dump_file"
log 'Dump concluído e integridade gzip verificada'

stage=upload
aws s3 cp "$dump_file" "s3://$BUCKET/$key" --only-show-errors
log "Upload concluído: s3://$BUCKET/$key"

stage=listagem
# Sem --no-paginate ou --max-items: a CLI reúne todas as páginas.
aws s3api list-objects-v2 --bucket "$BUCKET" --prefix "$PREFIX" \
    --output json >"$tmp_dir/objects.json"

stage=retencao
# Falha no parser interrompe o script antes de qualquer remoção.
python3 - "$tmp_dir/objects.json" >"$tmp_dir/expired.txt" <<'PY'
import json
import re
import sys
from datetime import datetime, timedelta, timezone

cutoff = datetime.now(timezone.utc) - timedelta(days=30)
pattern = re.compile(r"ledger/ledger_prod_\d{8}T\d{6}Z_[A-Za-z0-9]{8}\.sql\.gz")
with open(sys.argv[1], encoding="utf-8") as source:
    objects = json.load(source).get("Contents", [])
for obj in objects:
    key = obj["Key"]
    if not pattern.fullmatch(key):
        continue
    modified = datetime.fromisoformat(obj["LastModified"].replace("Z", "+00:00"))
    if modified < cutoff:
        print(key)
PY

while IFS= read -r old_key; do
    log "Removendo backup expirado: s3://$BUCKET/$old_key"
    aws s3 rm "s3://$BUCKET/$old_key" --only-show-errors
done <"$tmp_dir/expired.txt"

stage=concluido
log 'Backup e retenção concluídos'
exit 0
```

Exemplo do crontab do usuário de serviço, às 02:00 no fuso configurado no host:

```cron
SHELL=/bin/bash
PATH=/usr/local/bin:/usr/bin:/bin
0 2 * * * /usr/local/sbin/ledger-backup-with-secret >> /var/log/ledger-backup.log 2>&1
```

- **Integração com a cron:** `ledger-backup-with-secret` é o nome proposto para o wrapper do mecanismo existente de injeção. Ele precisa buscar o secret via IAM role, exportar `PGPASSWORD` e executar `exec /usr/local/sbin/ledger-backup.sh`. Esse wrapper deve ser implementado ou adaptado antes de ativar a cron; o enunciado não informa ID nem formato do secret. A cron não herda automaticamente variáveis da sessão interativa.
- **Pré-requisitos:** instalar Bash, AWS CLI, Python 3, gzip, util-linux (`flock`), coreutils e cliente PostgreSQL compatível com a versão do servidor, preferencialmente da mesma versão principal. Provisionar o diretório com modo `0700`, o log com `0600` e ambos pertencendo ao usuário de serviço. O script deve ser executável, e o log precisa de rotação.
- **Acessos:** `backup_user` precisa de conexão e leitura dos objetos necessários ao dump. A IAM role precisa de `s3:ListBucket` limitado ao prefixo e de `s3:PutObject`, `s3:DeleteObject` e `s3:AbortMultipartUpload` em `ledger/`*; a integração do secret requer `secretsmanager:GetSecretValue`. Acrescentar permissões KMS quando a configuração de criptografia exigir.
- **Retenção:** considera objetos com mais de 30 períodos de 24 horas, avaliados diariamente, apenas no prefixo e padrão de nomes gerenciados. A premissa é bucket sem versionamento e sem Object Lock impeditivo. Em bucket versionado, a remoção simples cria delete markers: é necessário também configurar expiração de versões não atuais e limpeza dos marcadores via Lifecycle. Recomenda-se Lifecycle para uploads multipart incompletos.
- **Disco e falhas:** o pipeline evita o SQL descompactado, 24 GiB são uma margem inicial, não uma garantia. Os temporários são removidos mesmo se o upload falhar, exigindo novo dump na próxima tentativa. `pipefail` impede que um erro do `pg_dump` seja mascarado pelo gzip. Falha na retenção retorna erro, mesmo que o novo backup já esteja no S3. SIGKILL ou queda da instância podem deixar temporários que exigem limpeza operacional.
- **Validação:** `gzip -t` verifica o arquivo compactado, mas não comprova restauração. É necessário testar a restauração em banco isolado e monitorar falhas e ausência de backups. O dump cobre o banco informado, sem fornecer PITR nem incluir objetos globais como roles. Não houve acesso ao PostgreSQL ou à AWS nesta resposta.

## Justificativa

**Role:** define uma pessoa SRE sênior com experiência em Bash, PostgreSQL e AWS, direcionando o conhecimento necessário para a tarefa.  
**Task:** fornece o ambiente do Ledger e exige dump, compactação, upload, retenção, logs e tratamento de falhas, com critérios verificáveis.  
**Format:** solicita o script completo, uma cron diária e até seis tópicos operacionais, organizando a saída para revisão e adaptação.

## Referências técnicas

A opção `--no-password` e os limites de compatibilidade do cliente estão descritos na [documentação do pg_dump](https://www.postgresql.org/docs/18/app-pgdump.html). A listagem utilizada mantém a [paginação automática da AWS CLI](https://docs.aws.amazon.com/cli/latest/reference/s3api/list-objects-v2.html). A ressalva sobre retenção em buckets versionados decorre do comportamento de [delete markers do Amazon S3](https://aws.amazon.com/about-aws/whats-new/2023/09/amazon-s3-last-modified-time-delete-markers-s3-head-get-apis/).

## Verificação da resposta

Após a geração, o bloco Bash passou em `bash -n` e em oito cenários simulados: sucesso, falha no dump, falha no upload, falha na retenção, lock ocupado, espaço insuficiente, senha ausente e JSON inválido na listagem. Os testes substituíram `pg_dump`, `aws`, `flock` e `df` por comandos simulados e redirecionaram os caminhos para um diretório temporário local. Foram conferidos os códigos de saída, a limpeza dos temporários, a ausência da senha nos logs e a preservação de objetos recentes e fora do padrão de nomes. Isso não valida a integração com PostgreSQL/S3, o bloqueio real no Ubuntu ou a restauração do backup.