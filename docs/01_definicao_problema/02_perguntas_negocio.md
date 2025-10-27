# On Market - perguntas de negócio

Documento que traduz o contexto de negócio em perguntas analíticas objetivas para o DW do **On Market**. Organizado por prioridade, com métricas, cortes e dependências de dados.

## 1. objetivo do documento
Definir quais perguntas o Data Warehouse deve responder, quais métricas utilizar e quais dimensões cruzar, garantindo alinhamento entre negócio e modelagem (fato `dw.vendas` e dimensões `dw.cliente`, `dw.produto`, `dw.distribuidor`, `dw.data`; opcionais: `categoria`, `avaliacao`).

## 2. perguntas essenciais (nível 1: must-have)

Vendas e receita
- Quanto vendemos (faturamento e quantidade) por mês, categoria e produto?
- Quais são os top produtos por faturamento e por quantidade em cada mês?
- Qual a participação de cada categoria no faturamento do período (mix de vendas)?

Ticket e clientes
- Qual é o ticket médio por mês? E por categoria? E por cidade do cliente?
- Como se distribuem os clientes por cidade/estado? Quais praças concentram receita?

Frete e operação
- Qual é o custo de frete total e médio por produto e por categoria?
- Qual a razão custo_de_frete sobre faturamento por categoria (peso do frete)?

Sazonalidade e calendário
- Como variam vendas por dia da semana e por mês? Existem picos sazonais?

## 3. perguntas diagnósticas (nível 2: should-have)

Mix e canibalização
- Há concentração excessiva de faturamento em poucos SKUs/categorias (curva ABC)?

Recência e frequência
- Qual a distribuição de clientes por recência de compra e frequência no período analisado?

Cesta e profundidade
- Quantos itens, em média, compõem uma transação?
  observação: requer um identificador de transação na fato; hoje não modelado, ver backlog

Preço e elasticidade (exploratório)
- Variações de preço do produto estão associadas a variações de quantidade vendida?
  observação: o preço atual está na dimensão produto; para análise temporal de preço, ver backlog

## 4. perguntas opcionais (nível 3: could-have)

Avaliações
- Produtos com melhores notas vendem mais? Há correlação entre nota média e quantidade vendida?
  observação: requer ativar dimensão/opção de avaliações

Geográfico detalhado
- Qual o share de faturamento por UF e por município ao longo do tempo?

Distribuidor
- Existe diferença relevante de custo de frete por distribuidor e região?

## 5. métricas e definições

Base
- quantidade_vendida: soma de `dw.vendas.quantidade_vendida`
- faturamento: soma de `dw.vendas.faturamento`
- custo_de_frete: soma de `dw.vendas.custo_frete`

Derivadas
- ticket_medio = faturamento / número_de_transacoes
  observação: na ausência de identificador de transação, usar proxy: faturamento / número_de_notas_fiscais ou ajustar o modelo para incluir `transacao_id`
- peso_do_frete = custo_de_frete / faturamento
- participação_categoria = faturamento_da_categoria / faturamento_total
- top_n_produtos = ranking por faturamento ou por quantidade no período

Dimensão tempo
- usar `dw.data` para agregações por dia, mês, ano e dia_da_semana

## 6. cortes (dimensões) recomendados

- tempo: `dw.data` (dia, mês, ano, dia_da_semana)
- produto: `dw.produto` (produto, categoria, preço atual, descrição)
- cliente: `dw.cliente` (cidade, estado/país)
- distribuidor: `dw.distribuidor` (nome, cidade, país)
- categoria (opcional como dimensão própria, se normalizada)
- avaliação (opcional)

## 7. limitações e premissas do escopo atual

- foco em transações efetivadas; pedidos (intenção) ficam fora da primeira versão
- fato `dw.vendas` no grão produto × transação × data, mas sem `transacao_id` explícito; agregar “por transação” exige incluir este campo no modelo (ver backlog)
- não há custo do produto nem impostos detalhados; não calcular margem/real profitability nesta fase
- preço na dimensão produto é estático; análise de preço no tempo requer histórico (ver backlog)
- logística detalhada (lead time, OTIF) não está modelada; apenas custo de frete

## 8. backlog de dados/modelagem

- adicionar `transacao_id` na fato `dw.vendas` para métricas “por transação” (itens por transação, conversão de carrinho)
- normalizar `categoria` como dimensão própria, se relatórios exigirem drill-down/slowly changing
- incluir tabela/atributos de **entrega** (datas de expedição e entrega) para KPIs de SLA
- historizar preço por período (tabela de preços vigentes) para análises de elasticidade
- ativar **avaliações** (dimensão/ponte) para correlação com vendas

## 9. mapeamento pergunta → tabelas

- vendas por mês, categoria, produto
  fato: `dw.vendas`
  dim: `dw.data`, `dw.produto` (categoria no atributo), opcional `categoria`
- top produtos por faturamento/quantidade
  fato: `dw.vendas`
  dim: `dw.data`, `dw.produto`
- ticket médio por período/segmento
  fato: `dw.vendas`
  dim: `dw.data`, `dw.produto`, `dw.cliente`
  observação: ideal incluir `transacao_id`
- distribuição geográfica de clientes e receita
  fato: `dw.vendas`
  dim: `dw.cliente`, `dw.data`
- custo de frete por produto/categoria e peso do frete
  fato: `dw.vendas`
  dim: `dw.produto`, `dw.data`
- sazonalidade (dia da semana, mês)
  fato: `dw.vendas`
  dim: `dw.data`
- avaliações x vendas (opcional)
  fato: `dw.vendas`
  dim: `avaliacao`, `dw.produto`, `dw.data`
  observação: depende de ativação da dimensão/opção de avaliações

## 10. critérios de aceite por pergunta

- cada pergunta deve ser respondida com agregações coerentes em janelas temporais (mês/ano) usando a dimensão data
- totais de faturamento e quantidade em relatórios mensais devem somar exatamente os totais do período
- rankings (top N) estáveis e reprodutíveis dado o mesmo dataset
- métricas derivadas (ticket, peso_do_frete) recalculáveis a partir das bases brutas do DW

## 11. exemplos de painéis mínimos

Painel comercial
- faturamento e quantidade por mês
- top 10 produtos por faturamento
- mix por categoria
- ticket médio mensal

Painel operacional
- custo de frete total e médio por categoria
- peso do frete por mês
- mapa de calor por UF/cidade (quantidade e faturamento)

Este documento deve orientar a priorização de desenvolvimento do DW e a validação das primeiras consultas em `tests/sql`.
