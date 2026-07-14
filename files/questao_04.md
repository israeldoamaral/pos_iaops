# Questão 04 - Relatório mensal de transações do Ledger
## Entrega

---

## Framework utilizado: T-A-G (Task - Action - Goal)

---

# Prompt

## T (Task)

Sua tarefa é escrever uma query SQL compatível com PostgreSQL para gerar um relatório mensal consolidado de crescimento de transações por categoria.

## A (Action)

Para construir a query, utilize as informações do banco de dados e siga as regras de negócio abaixo:

Schemas das tabelas:
- transactions (id, customer_id, category, amount_cents, status, payment_method, created_at, completed_at)
- customers (id, segment, country, signup_at)

Regras de filtragem, cálculo e formatação:
1. Filtre apenas transações que tenham status = 'completed'.
2. Defina o período para os últimos 6 meses corridos a partir de '2026-04-24' (ou seja, de '2025-10-24' até '2026-04-24'). Use o campo `created_at`.
3. Converta o valor do campo `amount_cents` (que está em centavos) para Reais (com duas casas decimais).
4. Agrupe os dados por mês (no formato YYYY-MM) e por categoria.
5. A saída deve conter as seguintes colunas por linha:
   - Mês (formatado como YYYY-MM)
   - Categoria (category)
   - Quantidade de transações
   - Volume total em Reais (com duas casas decimais)
6. Ordene o resultado final por mês em ordem crescente e, em seguida, por categoria em ordem crescente.

## G (Goal)

O objetivo final é obter uma query SQL otimizada (aproveitando os índices criados na tabela) que retorne exatamente os dados consolidados que a Jennifer precisa para a apresentação de negócios da Goldie, sem a necessidade de tratamento de dados posterior.

---

# Modelo

**Modelo sugerido:** GPT-4o (OpenAI)


---

# Output

```
SELECT 
    TO_CHAR(created_at, 'YYYY-MM') AS mes,
    category,
    COUNT(id) AS quantidade_transacoes,
    ROUND(SUM(amount_cents) / 100.0, 2) AS volume_total_reais
FROM 
    transactions
WHERE 
    status = 'completed'
    AND created_at >= '2026-04-24'::TIMESTAMPTZ - INTERVAL '6 months'
    AND created_at <= '2026-04-24'::TIMESTAMPTZ
GROUP BY 
    TO_CHAR(created_at, 'YYYY-MM'),
    category
ORDER BY 
    mes ASC,
    category ASC;
```

---

# Justificativa

| Elemento | Como aparece no Prompt |
|----------|-------------------------|
| **Task** | Define claramente o papel e a entrega esperada da IA (escrever uma query SQL para PostgreSQL). Isso evita que o modelo tente explicar conceitos de banco de dados ou crie códigos em outras linguagens, focando estritamente na geração do código SQL. |
| **Action** | É o coração do prompt. Aqui passamos o contexto do banco (schemas) e as regras lógicas de transformação passo a passo. É onde instruímos a IA sobre a conversão de centavos (/ 100.0), a formatação de data (TO_CHAR), os filtros de status/período e as métricas de agregação (COUNT e SUM).. |
| **Goal** | Alinha a expectativa de qualidade e o contexto de uso. Ao explicar que o objetivo é gerar dados limpos para uma apresentação executiva (Jennifer para Goldie) e que a query precisa ser otimizada para os índices existentes, a IA prioriza o uso das colunas indexadas no WHERE e entrega um resultado com formatação visualmente limpa (usando ROUND e apelidos amigáveis nas colunas).

---
