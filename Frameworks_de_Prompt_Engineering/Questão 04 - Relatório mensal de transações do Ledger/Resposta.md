# Resposta — Framework T-A-G

## Prompt

### Task (Tarefa)

Você é especialista em PostgreSQL. Escreva uma query SQL de leitura para um relatório mensal de transações do Ledger que Jennifer apresentará a Goldie, com quantidade de transações e volume total em reais por mês e categoria nos últimos seis meses corridos.

Considere este esquema:

```sql
transactions (
  id BIGSERIAL PRIMARY KEY,
  customer_id BIGINT NOT NULL REFERENCES customers(id),
  category VARCHAR(32) NOT NULL,
  amount_cents BIGINT NOT NULL,
  status VARCHAR(16) NOT NULL,
  payment_method VARCHAR(16),
  created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  completed_at TIMESTAMPTZ
)

customers (
  id BIGSERIAL PRIMARY KEY,
  segment VARCHAR(16) NOT NULL,
  country CHAR(2) NOT NULL,
  signup_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
)
```

Existem índices individuais em `transactions.created_at`, `transactions.status` e `transactions.category`. As categorias atuais são `subscription`, `one_time`, `refund` e `credit_adjustment`.

### Action (Ação)

Construa a query seguindo estas regras:

1. Use a data de referência fixa `2026-04-24`, indicada no enunciado, para tornar o resultado reproduzível. Calcule o início do intervalo subtraindo `INTERVAL '6 months'` dessa data.
2. Adote explicitamente estas premissas para os pontos não definidos no pedido: use `created_at` como data da transação e UTC como fuso para os limites e o agrupamento mensal. Inclua todo o dia de referência: limite inferior inclusivo em `2025-10-24 00:00:00+00` e superior exclusivo em `2026-04-25 00:00:00+00`. Não substitua o recorte por seis meses completos de calendário.
3. Inclua somente transações com `status = 'completed'`.
4. Agrupe por mês de `created_at` em UTC, apresentado como `YYYY-MM`, e por `category`.
5. Calcule a quantidade com `COUNT(*)` e o volume com a soma de `amount_cents` dividida por `100`, usando aritmética decimal e resultado numérico com duas casas decimais. Preserve o sinal armazenado dos valores, inclusive para estornos e ajustes.
6. Retorne apenas as colunas `mes`, `categoria`, `quantidade_transacoes` e `volume_total_reais`, nessa ordem. Ordene por mês crescente e categoria crescente.
7. Use somente `transactions`, pois não há filtros nem métricas de clientes. Retorne apenas combinações de mês e categoria com transações elegíveis, sem preencher linhas ausentes com zero.
8. Aplique os limites diretamente a `created_at` no `WHERE`, sem funções sobre essa coluna no filtro, para permitir o aproveitamento do índice existente.

### Goal (Objetivo)

Entregue uma única query PostgreSQL pronta para execução, que consolide os números necessários para Jennifer analisar a evolução mensal por categoria. Não invente resultados nem acrescente métricas de crescimento percentual. Após a query, explique brevemente as premissas de data e fuso, a conversão de centavos para reais e a ordenação.

## Modelo

Modelo proposto: GPT-4.1 da OpenAi.

## Output

```sql
WITH limites AS (
    SELECT
        (DATE '2026-04-24' - INTERVAL '6 months')
            AT TIME ZONE 'UTC' AS inicio,
        (DATE '2026-04-24' + INTERVAL '1 day')
            AT TIME ZONE 'UTC' AS fim_exclusivo
)
SELECT
    TO_CHAR(t.created_at AT TIME ZONE 'UTC', 'YYYY-MM') AS mes,
    t.category AS categoria,
    COUNT(*) AS quantidade_transacoes,
    CAST(SUM(t.amount_cents) / 100.0 AS NUMERIC(22, 2))
        AS volume_total_reais
FROM transactions AS t
CROSS JOIN limites AS l
WHERE t.status = 'completed'
  AND t.created_at >= l.inicio
  AND t.created_at < l.fim_exclusivo
GROUP BY
    TO_CHAR(t.created_at AT TIME ZONE 'UTC', 'YYYY-MM'),
    t.category
ORDER BY mes ASC, categoria ASC;
```

A consulta usa `created_at` e UTC como premissas explícitas, porque o enunciado não escolhe entre criação e conclusão nem informa um fuso. O intervalo vai de 24/10/2025 até o final de 24/04/2026. Por ser uma janela móvel, pode abranger sete rótulos mensais, com outubro e abril parciais, isso é diferente de selecionar seis meses completos.

A divisão por `100.0` evita divisão inteira, e `NUMERIC(22, 2)` produz um valor decimal em reais com duas casas, preservando os sinais registrados. A ordenação de `YYYY-MM` corresponde à ordem cronológica e, dentro de cada mês, as categorias aparecem em ordem crescente. Sem dados do Ledger, não é possível apresentar os totais reais.

## Justificativa do uso de T-A-G

| Elemento | Como aparece no prompt | Por que ajuda |
| --- | --- | --- |
| **Task — Tarefa** | Define a produção de uma query PostgreSQL para o relatório mensal e fornece o esquema disponível. | Delimita a tarefa e evita a invenção de tabelas ou colunas. |
| **Action — Ação** | Detalha filtros, limites de datas, agrupamento, conversão monetária, colunas de saída e ordenação. | Traduz a necessidade de negócio em operações SQL verificáveis e explicita as ambiguidades do recorte. |
| **Goal — Objetivo** | Solicita uma query pronta para execução que permita analisar a evolução mensal por categoria, acompanhada de uma explicação curta. | Define o resultado esperado e mantém a resposta alinhada ao uso de Jennifer na apresentação. |
