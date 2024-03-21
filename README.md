# Case-Churn
A equipe de assinaturas tem como objetivo reduzir a perda de assinantes, também conhecida como "Churn", na equipe de assinaturas. O conceito de "Churn" refere-se à perda de qualquer usuário que tenha assinado o serviço de assinatura e posteriormente o cancelou.
O objetivo principal deste case é identificar os principais fatores que contribuem para o Churn dos assinantes e a fim de desenvolver estratégias eficazes para reduzir essa perda. Isso envolve a análise de dados relacionados aos assinantes, incluindo informações sobre a data de criação da assinatura, a data de cancelamento, o comportamento do usuário, a receita gerada e outros dados relevantes.  

### Tecnologias utilizadas
* [BigQuery](https://www.googleadservices.com/pagead/aclk?sa=L&ai=DChcSEwiQkoS66YGFAxUqXkgAHZyyC8QYABAAGgJjZQ&gclid=CjwKCAjw7-SvBhB6EiwAwYdCAU-yl5hc3WyK1gb5OJRQl9_eJOrWxA83gTryphXXFx_VEWMWoEFqWRoCQ9QQAvD_BwE&ohost=www.google.com&cid=CAESVuD2TQwm3kU8xvOk3AzUr6VOhQCjdT4mLvrZ-qZI5lbufVtG-bIaVWBNcC5ccJcrlqQ6nrT0xedjJcmYv4xZXFKgEY8NHUdxSijbsvWvZtkbOQcZ_yzX&sig=AOD64_2DD-ziKq6QSQdS_VQGbU1C-Lft9A&q&adurl&ved=2ahUKEwjp9v656YGFAxXSqZUCHashB6MQ0Qx6BAgGEAE)
* [PowerBI](https://github.com/thais-matos/Case-Churn/blob/main/Case_Churn.pbix](https://www.microsoft.com/pt-br/power-platform/products/power-bi/))
* Planilhas Google

### Códigos utilizados (Query)  
Essa consulta tem o objetivo de fornecer informações sobre a taxa de cancelamentos e pausas nas assinaturas por canal de aquisição correlacionados a geolocalização do cliente, permitindo uma análise com o intuito de identificar quais canais de aquisição têm melhor desempenho em reter clientes e analisar as taxas de churn por estado, a fim de identificar possíveis ações regionais que estejam funcionando na retenção dos clientes.

Funções utilizadas:
* COUNT - Para contar o total de assinaturas da base;
* COUNTIF - Para contar de forma condicionada o Total de assinaturas canceladas e pausadas;
* Operadores matemáticos para calcular a taxa de cancelamentos;
```ruby
--Taxa de churn por Canal de Aquisição correlacionada a geolocalização (Estado) do cliente
  SELECT 
    marketing_source,
    state,
    COUNT(DISTINCT id) AS Total_assinaturas,
    COUNTIF(deleted_at IS NOT NULL) AS Assinaturas_canceladas,
    COUNTIF(status = 'paused') AS Assinaturas_pausadas,
    (COUNTIF(deleted_at IS NOT NULL) / COUNT(DISTINCT id)) * 100 AS Taxa_churn
  FROM 
    `case-417720.Churn.Case`
  GROUP BY
    state,
    marketing_source
```
Nessa consulta foi realizadas as conversões das colunas deleted_at (cancelamento) e created_at (assinatura) para o tipo Date e calculada a diferença em dias entre as datas de cancelamento e assinatura para cada cliente, usada para calcular o tempo médio que os clientes permaneceram ativos antes de cancelar a assinatura. Com intuito de ajudar a empresa a entender a longevidade típica do cliente.
Também foi calculada a soma da receita total, soma do ticket médio e soma do total de pedidos de cada cliente, tornando possível analisar a receita gerada pelos clientes ao longo do tempo antes do churn ou da pausa na assinatura, assim como o tempo desde a última compra do cliente (recency). Auxiliando a empresa a entender o valor desses clientes e identificar clientes que não realizam compras há muito tempo mostrando um maior risco de churn e oportunidades para aumentar a retenção.  

Funções utilizadas:  
* PARSE_TIMESTAMP() - Para converte a string de data e hora em um valor de data e hora do tipo TIMESTAMP especificando que a string de entrada segue o formato mês/dia/ano hora:minuto AM/PM;  
* DATE - Para converter o resultado da função acima em data;  
* DATE_DIFF - Para calcular o número de dias entre a data de cancelamento da assinatura e a data de criação da assinatura;  
* SUM - Para somar o total das métricas necessárias;

* Foram utilizadas subqueries com a finalidade de filtrar e/ou tratar dados específicos para serem utilizados na consulta final.

```ruby
WITH Datas AS (
  SELECT
    id,
    DATE(PARSE_TIMESTAMP('%m/%d/%y %I:%M %p', deleted_at)) AS Data_cancelamento,
    DATE(PARSE_TIMESTAMP('%m/%d/%y %I:%M %p', created_at)) AS Data_assinatura,
    recency,
    status
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
    status
  FROM Datas
),

ReceitaPorGeolocalizacao AS (
  SELECT
    id,
    state,
    city,
    SUM(all_revenue) AS Receita_total,
    SUM(all_orders) AS Total_pedidos,
    SUM(average_ticket) as Ticket_medio
  FROM
    `case-417720.Churn.Case`
  GROUP BY
    id,
    state,
    city
)

SELECT
  RPG.id,
  RPG.state,
  RPG.city,
  RPG.Receita_total,
  RPG.Total_pedidos,
  RPG.Ticket_medio,
  DD.Dias_entre_assinatura_e_cancelamento,
  DD.recency,
  DD.status
FROM 
  DiferencaDatas AS DD
JOIN 
  ReceitaPorGeolocalizacao AS RPG ON DD.id = RPG.id
WHERE DD.status in ('paused', 'canceled')
```

Essa consulta retorna a contagem de clientes como status 'paused' ou 'canceled', agrupados de acordo com a versão da assinatura e por Estado, fornecendo informações que podem ajudar a identificar padrões específicos de churn ou possível churn de acordo com a versão da assinatura do cliente.  

Funções utilizadas:  
* COUNT - Para contar o total de clientes de assinatura;
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

