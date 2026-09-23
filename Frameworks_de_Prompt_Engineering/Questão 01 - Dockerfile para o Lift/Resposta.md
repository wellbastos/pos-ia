# Resposta — Questão 01: Dockerfile para o Lift

## Prompt

Role
Atue como um DevOps sênior, com experiência em
containerização de aplicações Python/Flask para Kubernetes.

Task
Crie um Dockerfile para o Lift, produto em beta da Hill Valley Tech que
será migrado de VMs para Kubernetes. A API Flask escuta na porta 8080.
Considere lift/ como contexto de build, com esta estrutura:

Estrutura do Projeto
```
lift/
├── app.py
├── requirements.txt
├── lib/
│   ├── auth.py
│   └── storage.py
└── tests/
    └── test_app.py
````

O requirements.txt contém exatamente:
```
Flask==3.0.0
gunicorn==21.2.0
requests==2.31.0
python-dotenv==1.0.0
psycopg2-binary==2.9.9
````

O comando de produção é:
gunicorn --bind 0.0.0.0:8080 --workers 4 app:app

DATABASE_URL e API_KEY são obrigatórias no runtime e serão injetadas
pelo Kubernetes. Não inclua valores, arquivos .env ou segredos na imagem,
nem use ARG ou ENV para armazenar essas credenciais no Dockerfile.

Use uma imagem oficial Python slim com versão explícita, sem latest.
Como a versão do Python não foi informada, declare sua escolha como
premissa a validar com a aplicação. Defina WORKDIR, execute com usuário
e grupo sem privilégios e IDs numéricos fixos, aproveite o cache de
dependências copiando requirements.txt antes do código e instale com pip
sem cache. Copie somente os arquivos necessários à execução, excluindo
testes e arquivos de desenvolvimento. Configure Python para não gerar
arquivos .pyc e para não reter sua saída em buffer. Declare a porta 8080
e use CMD no formato exec, preservando o comando de produção fornecido.
Evite ferramentas de compilação e pacotes de sistema sem necessidade
demonstrada. Não altere as versões de dependências do enunciado.

Format
Responda nesta ordem:
1. Um bloco de código Dockerfile completo, para salvar em lift/Dockerfile.
2. Um bloco de código com um .dockerignore complementar para lift/.
3. Até seis tópicos curtos explicando as decisões, a premissa de versão
   do Python e a injeção das variáveis no runtime.
Não gere manifests Kubernetes, não invente endpoints de healthcheck e
não afirme ter executado build ou testes sem realmente executá-los.


## Modelo

GPT-6, da OpenAI — escolhido pela capacidade de gerar código e explicar como as restrições do prompt aparecem na solução.

## Output

Resposta produzida nesta sessão a partir do prompt acima.

```dockerfile
FROM python:3.11-slim-bookworm

ENV PYTHONDONTWRITEBYTECODE=1 \
    PYTHONUNBUFFERED=1

WORKDIR /app

RUN groupadd --gid 10001 lift \
    && useradd --uid 10001 --gid 10001 --no-create-home \
       --shell /usr/sbin/nologin lift

COPY requirements.txt ./
RUN python -m pip install --no-cache-dir -r requirements.txt

COPY app.py ./
COPY lib/ ./lib/

USER 10001:10001

EXPOSE 8080

CMD ["gunicorn", "--bind", "0.0.0.0:8080", "--workers", "4", "app:app"]
```

```.dockerignore
.git/
.venv/
venv/
**/__pycache__/
**/*.py[cod]
.pytest_cache/
.mypy_cache/
.ruff_cache/
.coverage
htmlcov/
tests/
.env
.env.*
*.log
```

- **Imagem base:** Python 3.11 é uma premissa a validar com o código do Lift. A variante slim mantém uma base enxuta; a tag pode receber atualizações, portanto uma tag seria necessário para fixar exatamente a imagem.
- **Dependências:** copiar `requirements.txt` antes do código permite reutilizar a camada de instalação quando apenas a aplicação muda. `--no-cache-dir` evita armazenar o cache do pip na imagem.
- **Permissões:** o processo roda como `10001:10001`. O código copiado permanece pertencendo ao root e disponível para leitura pelo serviço, sem conceder ao processo permissão para modificá-lo.
- **Conteúdo:** somente `app.py` e `lib/` são copiados como código da aplicação. O `.dockerignore` também reduz o contexto enviado ao build, excluindo testes, ambientes locais e arquivos `.env`.
- **Inicialização:** o `CMD` em formato exec inicia o Gunicorn diretamente, com a porta e os quatro workers solicitados. `EXPOSE` documenta a porta; sua publicação depende da configuração do ambiente.
- **Runtime e validação:** `DATABASE_URL` e `API_KEY` devem ser injetadas via Secrets do Kubernetes, o Dockerfile não garante sua presença. Build e testes não foram executados, pois o enunciado fornece a estrutura, mas não os arquivos reais da aplicação.

## Justificativa

**Role:** define um DevOps sênior com experiência em Python/Flask e Kubernetes, orientando a especialidade da resposta.  
**Task:** apresenta o cenário do Lift, os arquivos, as dependências, o comando de produção e as restrições de segurança e construção da imagem.  
**Format:** determina dois blocos de código e até seis tópicos explicativos, tornando a entrega objetiva e fácil de conferir.