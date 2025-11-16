# On Market - modelo dimensional: KPIs e métricas

Documento do **modelo dimensional** do **On Market DW** focado nas **métricas e KPIs** que serão calculados a partir da fato `dw.vendas` e das dimensões `dw.cliente`, `dw.produto`, `dw.distribuidor` e `dw.data`.

Este arquivo complementa:

* `docs/01_definicao_problema/02_perguntas_negocio.md`
* `docs/03_modelo_dimensional/01_fato_dimensoes.md`

A ideia é sair das perguntas de negócio e amarrar em fórmulas claras, grãos de cálculo e dependências de dados.

---
## 1) Objetivo do documento

1. Definir KPIs principais do On Market DW para a v1.
2. Especificar fórmulas em termos de colunas da fato e dimensões.
3. Indicar o grão recomendado de cálculo e de exposição em relatórios.
4. Garantir rastreabilidade com perguntas de negócio e regras conceituais.

---
## 2) Medidas base da fato `dw.vendas`

Todas as métricas derivadas partem das medidas base da fato.

Medidas base:

1. `quantidade_vendida`
   * Fonte: `dw.vendas.quantidade_vendida`
   * Tipo: inteiro maior que zero
   * Significado: quantidade de unidades do produto na linha de venda

2. `faturamento`
   * Fonte: `dw.vendas.faturamento`
   * Tipo: decimal em BRL com duas casas
   * Significado: valor monetário da venda daquela linha

3. `custo_frete`
   * Fonte: `dw.vendas.custo_frete`
   * Tipo: decimal em BRL com duas casas
   * Significado: valor de frete associado àquela linha de venda

Para qualquer agregação:

* `qtde_total` é a soma de `quantidade_vendida` na janela analisada.
* `faturamento_total` é a soma de `faturamento`.
* `custo_frete_total` é a soma de `custo_frete`.

Essas três somas são a base de praticamente todos os KPIs.

---
## 3) KPIs comerciais de vendas e receita

### 3.1 Faturamento total

* Nome: faturamento_total
* Definição: soma de `faturamento` em uma janela de tempo e recortes dimensionais específicos.
* Fórmula:

  `faturamento_total = sum(dw.vendas.faturamento)`

* Grão típico de exposição:
  * por mês e ano
  * por mês e categoria
  * por produto

### 3.2 Quantidade total vendida

* Nome: quantidade_total
* Definição: soma de `quantidade_vendida` em uma janela de tempo.
* Fórmula:

  `quantidade_total = sum(dw.vendas.quantidade_vendida)`

* Grão típico:
  * por mês
  * por produto
  * por categoria

### 3.3 Ticket médio

Sem `transacao_id` explícito na v1, o ticket real por transação é aproximado.

* Nome: ticket_medio
* Definição conceitual: faturamento total dividido pelo número de transações.
* Definição v1: proxy baseada em linhas da fato ou em nível de agregação coerente.

Exemplos de fórmulas:

1. Proxy simples por mês (mais aproximado, mas consistente):

   `ticket_medio_mensal = sum(faturamento) / count(*)`

   onde `count(*)` conta linhas de venda na janela mensal.

2. Proxy mais conservador, se for usado um nível de agregação superior:

   `ticket_medio_por_produto = sum(faturamento) / count(distinct produto_id)`

A recomendação para v1 é documentar no relatório qual proxy está em uso.

### 3.4 Receita por categoria

* Nome: faturamento_por_categoria
* Definição: faturamento total agrupado por `nome_categoria` da dimensão de produto.
* Fórmula:

  `faturamento_por_categoria = sum(faturamento) agrupado por produto.nome_categoria`

Requer join com `dw.produto`.

### 3.5 Mix de vendas por categoria

* Nome: participacao_categoria
* Definição: quanto cada categoria representa do faturamento total do período.
* Fórmula:

  `participacao_categoria = faturamento_categoria / faturamento_total`

onde `faturamento_categoria` é a soma de `faturamento` da categoria na janela e `faturamento_total` é a soma de `faturamento` de todas as categorias na mesma janela.

### 3.6 Top produtos

* Nome: top_n_produtos
* Definição: ranking de produtos ordenado por `faturamento_total` ou `quantidade_total`.
* Fórmula conceitual:

  ordenar produtos por `sum(faturamento)` ou `sum(quantidade_vendida)` na janela e selecionar os primeiros N.

Grão típico: produto por mês ou produto em um período definido.

---
## 4) KPIs de frete e operação

### 4.1 Custo de frete total

* Nome: custo_frete_total
* Definição: soma de `custo_frete` na janela de análise.
* Fórmula:

  `custo_frete_total = sum(dw.vendas.custo_frete)`

Pode ser analisado por:

* categoria
* produto
* distribuidor
* período

### 4.2 Custo de frete médio por venda ou por unidade

Existem duas perspectivas comuns.

1. Custo de frete médio por linha de venda em uma janela:

   `custo_frete_medio_linha = sum(custo_frete) / count(*)`

2. Custo de frete médio por unidade de produto:

   `custo_frete_medio_unidade = sum(custo_frete) / sum(quantidade_vendida)`

Ambas podem ser exibidas por categoria, distribuidor e período.

### 4.3 Peso do frete sobre o faturamento

* Nome: peso_do_frete
* Definição: proporção de custo de frete em relação ao faturamento.
* Fórmula:

  `peso_do_frete = custo_frete_total / faturamento_total`

Recomendações:

* Sempre calcular na mesma janela de tempo.
* Evitar interpretação de linhas isoladas, preferindo visão agregada por mês ou categoria.

---
## 5) KPIs de clientes e geografia

### 5.1 Faturamento por cidade e país

* Nome: faturamento_por_localidade
* Definição: soma do faturamento agrupada por cidade e país do cliente.
* Fórmula:

  `faturamento_por_localidade = sum(faturamento) agrupado por cliente.cidade, cliente.pais`

Requer join com `dw.cliente`.

### 5.2 Participação geográfica

* Nome: participacao_geografica
* Definição: participação percentual da receita por localidade em relação ao total.
* Fórmula:

  `participacao_geografica = faturamento_da_localidade / faturamento_total`

Pode ser calculada por cidade, estado, país, dependendo de quais atributos existirem na dimensão.

### 5.3 Clientes ativos (proxy simples v1)

Na v1 não há tabela de clientes com datas de última compra historizada em outra estrutura, então a definição é simples.

* Nome: clientes_ativos_no_periodo
* Definição: clientes que tiveram ao menos uma linha de venda na janela de data escolhida.
* Fórmula:

  `clientes_ativos_no_periodo = count(distinct cliente_id)`

A combinação com recência ou frequência mais refinada pode ficar para versões futuras.

---
## 6) KPIs de sazonalidade e calendário

### 6.1 Faturamento por mês

* Nome: faturamento_mensal
* Definição: soma de faturamento agrupada por mês e ano da dimensão de data.
* Fórmula:

  `faturamento_mensal = sum(faturamento) agrupado por data.ano, data.mes`

### 6.2 Faturamento por dia da semana

* Nome: faturamento_por_dia_da_semana
* Definição: soma de faturamento agrupada por `dia_da_semana`.
* Fórmula:

  `faturamento_por_dia_da_semana = sum(faturamento) agrupado por data.dia_da_semana`

### 6.3 Quantidade vendida por período

* Nome: quantidade_por_periodo
* Definição: soma de quantidade por dia, mês ou ano, conforme interesse.
* Fórmula:

  `quantidade_por_periodo = sum(quantidade_vendida) agrupado pela granularidade escolhida na dimensão data`

Esses KPIs suportam análises de sazonalidade e tendência.

---
## 7) Grão de cálculo e boas práticas

Para evitar inconsistências:

1. Sempre declarar a janela de tempo do KPI, usando `dw.data`.
2. Deixar claro o grão na visualização: por exemplo, “ticket médio mensal” difere de “ticket médio diário”.
3. Quando usar proxies (como ticket médio sem `transacao_id`), documentar explicitamente a fórmula aplicada.
4. Evitar misturar agregações com grãos diferentes na mesma métrica sem indicar isso em nome ou descrição.

---
## 8) Exemplos de consultas em SQL

A seguir alguns exemplos em SQL padrão PostgreSQL, usando os nomes lógicos.

### 8.1 Faturamento e quantidade por mês

```sql
SELECT
  d.ano,
  d.mes,
  SUM(v.quantidade_vendida) AS quantidade_total,
  SUM(v.faturamento)        AS faturamento_total
FROM dw.vendas v
JOIN dw.data d
  ON d.data_id = v.data_id
GROUP BY
  d.ano,
  d.mes
ORDER BY
  d.ano,
  d.mes;
```

### 8.2 Ticket médio mensal (proxy v1)

```sql
SELECT
  d.ano,
  d.mes,
  SUM(v.faturamento) AS faturamento_total,
  COUNT(*)           AS linhas_fato,
  SUM(v.faturamento) / COUNT(*) AS ticket_medio_mensal
FROM dw.vendas v
JOIN dw.data d
  ON d.data_id = v.data_id
GROUP BY
  d.ano,
  d.mes
ORDER BY
  d.ano,
  d.mes;
```

### 8.3 Peso do frete por categoria e mês

```sql
SELECT
  d.ano,
  d.mes,
  p.nome_categoria,
  SUM(v.faturamento)  AS faturamento_total,
  SUM(v.custo_frete)  AS custo_frete_total,
  SUM(v.custo_frete) / NULLIF(SUM(v.faturamento), 0) AS peso_do_frete
FROM dw.vendas v
JOIN dw.data d
  ON d.data_id = v.data_id
JOIN dw.produto p
  ON p.produto_id = v.produto_id
GROUP BY
  d.ano,
  d.mes,
  p.nome_categoria
ORDER BY
  d.ano,
  d.mes,
  p.nome_categoria;
```

### 8.4 Top produtos por faturamento no período

```sql
SELECT
  p.produto_id,
  p.nome,
  SUM(v.faturamento) AS faturamento_total
FROM dw.vendas v
JOIN dw.produto p
  ON p.produto_id = v.produto_id
GROUP BY
  p.produto_id,
  p.nome
ORDER BY
  faturamento_total DESC
LIMIT 10;
```

---
## 9) Mapeamento KPIs → perguntas de negócio

Relação direta com o documento de perguntas de negócio (`docs/01_definicao_problema/02_perguntas_negocio.md`):

1. “Quanto vendemos por mês, categoria e produto?”
   * KPIs: `faturamento_total`, `quantidade_total`, `faturamento_por_categoria`

2. “Quais são os top produtos por faturamento e quantidade por mês?”
   * KPIs: `top_n_produtos` por faturamento, `top_n_produtos` por quantidade

3. “Qual a participação de cada categoria no faturamento?”
   * KPI: `participacao_categoria`

4. “Qual o ticket médio por mês, categoria e cidade?”
   * KPI: `ticket_medio`, com cortes em tempo (`dw.data`), produto (`dw.produto`) e cliente (`dw.cliente`)

5. “Qual o custo de frete total e médio por produto e categoria?”
   * KPIs: `custo_frete_total`, `custo_frete_medio_linha`, `custo_frete_medio_unidade`

6. “Qual o peso do frete sobre o faturamento por categoria?”
   * KPI: `peso_do_frete` por categoria

7. “Como variam vendas por dia da semana e por mês?”
   * KPIs: `faturamento_mensal`, `faturamento_por_dia_da_semana`, `quantidade_por_periodo`

8. Perguntas geográficas (receita por cidade/estado/país)
   * KPIs: `faturamento_por_localidade`, `participacao_geografica`

---
## 10) Critérios de aceite dos KPIs

Os KPIs descritos neste documento são considerados implantados quando:

1. Todas as métricas base (`quantidade_vendida`, `faturamento`, `custo_frete`) estão disponíveis e consistentes na fato `dw.vendas`.
2. Consultas de referência para faturamento, quantidade e custo de frete por mês retornam valores estáveis e reprodutíveis.
3. Painéis mínimos conseguem exibir, no mínimo:
   * faturamento e quantidade por mês
   * top produtos em faturamento
   * mix por categoria
   * ticket médio mensal
   * `peso_do_frete` por categoria e mês
4. A implementação em SQL das métricas segue as fórmulas documentadas neste arquivo, sem definições divergentes em relatórios ou camadas de BI.

---
## 11) Histórico do documento

1. v0.3.0 primeira versão do documento de KPIs, alinhada ao modelo dimensional v1
---
