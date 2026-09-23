# Questão 01 - Dockerfile para o Lift

O Lift vai sair das VMs onde vem rodando e entrar no cluster Kubernetes da empresa. O código já está pronto: uma API Python/Flask na porta 8080, dependências declaradas em requirements.txt, e duas variáveis de ambiente que precisam estar presentes no runtime, DATABASE_URL e API_KEY.

Estrutura do projeto:

lift/
├── app.py
├── requirements.txt
├── lib/
│   ├── auth.py
│   └── storage.py
└── tests/
    └── test_app.py
Conteúdo de requirements.txt:

Flask==3.0.0
gunicorn==21.2.0
requests==2.31.0
python-dotenv==1.0.0
psycopg2-binary==2.9.9
Em produção o serviço sobe com gunicorn --bind 0.0.0.0:8080 --workers 4 app:app.

Falta o Dockerfile. Seguir todas as boas práticas de criação.

Tarefa. Aplicando o framework R-T-F, escrever o prompt de IA que produza esse Dockerfile.

Entregue. Prompt, modelo, output e justificativa mostrando como Role, Task e Format aparecem no prompt.