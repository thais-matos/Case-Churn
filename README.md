# Case-Churn
A equipe de assinaturas tem como objetivo reduzir a perda de assinantes, também conhecida como "Churn", na equipe de assinaturas. O conceito de "Churn" refere-se à perda de qualquer usuário que tenha assinado o serviço de assinatura e posteriormente o cancelou.
O objetivo principal deste case é identificar os principais fatores que contribuem para o Churn dos assinantes e a fim de desenvolver estratégias eficazes para reduzir essa perda. Isso envolve a análise de dados relacionados aos assinantes, incluindo informações sobre a data de criação da assinatura, a data de cancelamento, o comportamento do usuário, a receita gerada e outros dados relevantes.  

### Tecnologias utilizadas
* [BigQuery](https://www.googleadservices.com/pagead/aclk?sa=L&ai=DChcSEwiQkoS66YGFAxUqXkgAHZyyC8QYABAAGgJjZQ&gclid=CjwKCAjw7-SvBhB6EiwAwYdCAU-yl5hc3WyK1gb5OJRQl9_eJOrWxA83gTryphXXFx_VEWMWoEFqWRoCQ9QQAvD_BwE&ohost=www.google.com&cid=CAESVuD2TQwm3kU8xvOk3AzUr6VOhQCjdT4mLvrZ-qZI5lbufVtG-bIaVWBNcC5ccJcrlqQ6nrT0xedjJcmYv4xZXFKgEY8NHUdxSijbsvWvZtkbOQcZ_yzX&sig=AOD64_2DD-ziKq6QSQdS_VQGbU1C-Lft9A&q&adurl&ved=2ahUKEwjp9v656YGFAxXSqZUCHashB6MQ0Qx6BAgGEAE)

### Códigos utilizados (Query)  
Essa query tem o objetivo de fornecer informações sobre canais aquisição de clientes, datas de assinatura e cancelamento, receita correlacionados a geolocalização do cliente, permitindo uma análise abrangente dessas variáveis:
```ruby
--Informações de Canal de Aquisição, Datas e Receita por Geolocalização
WITH ChurnPorCanal AS (
  SELECT 
    id,
    marketing_source,
    state,
    COUNT(DISTINCT id) AS total_assinaturas,
    COUNTIF(deleted_at IS NOT NULL) AS assinaturas_canceladas,
    (COUNTIF(deleted_at IS NOT NULL) / COUNT(DISTINCT id)) * 100 AS taxa_churn,
    status
  FROM 
    `case-417720.Churn.Case`
  GROUP BY
    id, 
    state,
    marketing_source,
    status
),

Datas AS (
  SELECT
    id,
    DATE(PARSE_TIMESTAMP('%m/%d/%y %I:%M %p', deleted_at)) AS Data_cancelamento,
    DATE(PARSE_TIMESTAMP('%m/%d/%y %I:%M %p', created_at)) AS Data_assinatura,
    recency
  FROM
    `case-417720.Churn.Case`
),

DiferencaDatas AS (
  SELECT 
    id,
    Data_assinatura,
    Data_cancelamento,
    DATE_DIFF(Data_cancelamento, Data_assinatura, DAY) AS Dias_entre_assinatura_e_cancelamento,
    recency,
  FROM Datas
),

ReceitaPorGeolocalizacao AS (
  SELECT
    state,
    city,
    SUM(all_revenue) AS Receita_total,
    SUM(all_orders) AS Total_pedidos,
    SUM(average_ticket) as Ticket_medio,
  FROM
    `case-417720.Churn.Case`
  GROUP BY
    state,
    city
)

SELECT
  CPC.state,
  RPG.* except(state),
  CPC.marketing_source,
  CPC.status,
  CPC.assinaturas_canceladas,
  CPC.total_assinaturas,
  CPC.taxa_churn,
  DD.Dias_entre_assinatura_e_cancelamento
FROM 
  ChurnPorCanal AS CPC
JOIN 
  DiferencaDatas AS DD ON CPC.id = DD.id
JOIN 
  ReceitaPorGeolocalizacao AS RPG ON CPC.state = RPG.state
group by 1,2,3,4,5,6,7,8,9,10,11
```
Essa consulta permite visualizar a contagem total de assinaturas, a contagem de assinaturas canceladas e a taxa de churn em percentual:
```ruby
-- Taxa de churn geral das assinaturas
SELECT 
  COUNT(DISTINCT id) AS total_assinaturas,
  COUNTIF(deleted_at IS NOT NULL) AS assinaturas_canceladas,
  (COUNTIF(deleted_at IS NOT NULL) / COUNT(DISTINCT id)) * 100 AS taxa_churn
FROM 
  `case-417720.Churn.Case`
```

Essa consulta retorna a contagem de clientes para diferentes versões (ou estados) das assinaturas, onde o status é 'paused' ou 'canceled':
```ruby
-- Quantidade de clientes por versão (status paused e canceled)
SELECT
  count(id) as Qtde_clientes,
  status,
  state,
  version
FROM
  `case-417720.Churn.Case`
WHERE status in ('paused','canceled')
group by 2,3
```

Nessa consulta é feita uma conversão das colunas deleted_at (cancelamento) e created_at (assinatura) para o tipo de dado Date. Além disso, são agregadas outras métricas como receita total, total de pedidos, recência e ticket médio. Também calcula a diferença em dias entre as datas de cancelamento e assinatura para cada cliente:
```ruby
-- Conversão em Date das colunas assinatura e cancelamento
with Datas as
(
  SELECT
  id,
  date(PARSE_TIMESTAMP('%m/%d/%y %I:%M %p', deleted_at)) AS Data_cancelamento,
  date(PARSE_TIMESTAMP('%m/%d/%y %I:%M %p', created_at)) AS Data_assinatura,
  SUM(all_revenue) AS Receita_total,
  SUM(all_orders) as Total_pedidos,
  recency,
  status,
  average_ticket
FROM
  `case-417720.Churn.Case`
group by 1,2,3,6,7,8
)

-- Diferença entre as datas de assinatura e cancelamento
Select 
  t1.*,
  date_diff(Data_cancelamento, Data_assinatura, day) AS Lifetime_cliente
From Datas t1
WHERE
  t1.Data_cancelamento IS NOT NULL
```
