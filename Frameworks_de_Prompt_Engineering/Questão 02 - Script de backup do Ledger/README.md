# Questão 02 - Script de backup do Ledger

Lorraine chegou à conclusão de que o Ledger, o PostgreSQL que o George levantou na EC2 anos atrás, nunca teve rotina de backup automatizada. Hoje isso é uma dependência aberta no radar da SRE, e ela quer fechar com uma cron diária. O ambiente onde o script vai rodar:

Host: ledger-db.internal.hvt.io
Porta: 5432
Banco: ledger_prod
Usuário de backup: backup_user
Senha: variável de ambiente PGPASSWORD, populada pelo AWS Secrets Manager via IAM role da instância
Região AWS: us-east-1
SO da instância: Ubuntu 22.04 LTS
Diretório de trabalho com 80 GB livres: /var/backups/ledger
Tamanho médio atual do dump compactado: ~12 GB
O script precisa fazer o dump com pg_dump, compactar com gzip, subir o arquivo pro bucket S3 hvt-ledger-backups via aws s3 cp, manter 30 dias de retenção no S3 (removendo os mais antigos), registrar cada execução em /var/log/ledger-backup.log com timestamp, e sair com exit code adequado em caso de falha.

Tarefa. Aplicando o framework R-T-F, escrever o prompt de IA que produza esse script bash.

Entregue. Prompt, modelo, output e justificativa mostrando como Role, Task e Format aparecem no prompt.