# On Market - contexto de negócio

Documento de contexto do estudo de caso **On Market**, um e-commerce generalista inspirado em grandes players (Mercado Livre, Magalu, Shopee, AliExpress). Este repositório demonstra, de ponta a ponta, a construção de um **Data Warehouse (DW)** para análise de vendas, operação e comportamento do cliente, com implementação local via **Docker + PostgreSQL 16** e uso do **VS Code**.

## 1. propósito do estudo

Quero um projeto público e reproduzível que mostre, de forma objetiva, como eu desenho e implemento um DW de e-commerce. O foco é **análise** e **decisão**, não o sistema transacional. O DW parte de um cenário simplificado e generalizável para diferentes e-commerces.

## 2. objetivos de negócio

1. Medir desempenho comercial e operacional de forma consistente ao longo do tempo.
2. Permitir análises por **produto, cliente, tempo e distribuição**.
3. Construir um modelo **simples, extensível e documentado** para servir de base a exercícios de SQL, BI e ML.
4. Garantir reprodutibilidade local com Docker, sem dependências de dados sensíveis.

## 3. escopo e não escopo

**No escopo**
- Vendas B2C com foco em eletrônicos e eletrodomésticos.
- Fluxos: descoberta → pedido (intenção) → transação (confirmação) → entrega → avaliação.
- Métricas: quantidade vendida, faturamento, custo de frete, ticket e indicadores derivados.

**Fora do escopo (por ora)**
- Marketplace completo de múltiplos vendedores com repasse/commissionamento.
- Promoções avançadas, cupons, preços dinâmicos, múltiplas moedas e impostos detalhados.
- Orquestração real de ETL; aqui uso **dados sintéticos** via funções SQL.

## 4. stakeholders e personas

- **Diretoria/gestão**: visão macro de faturamento e crescimento.
- **Operações/logística**: custo de frete, lead time, performance de entrega.
- **Comercial/produto**: mix, preço, categorias, itens mais/menos vendidos.
- **Analytics/DS**: base única e confiável para análises exploratórias e modelos.

## 5. perguntas de negócio prioritárias

- Qual o faturamento e a quantidade vendida por **mês, categoria e produto**?
- Qual o **ticket médio** por período e por segmento de clientes?
- Quais produtos e categorias concentram **maior custo de frete**?
- Quais **tendências sazonais** aparecem nas vendas?
- Qual o comportamento de compra por **cidade/estado** do cliente?
- Como as **avaliações** se relacionam com vendas (quando usadas)?

## 6. kpis e métricas alvo

- **Quantidade vendida**
- **Faturamento** (soma do valor de transações confirmadas)
- **Custo de frete**
- **Ticket médio** = faturamento / número de transações
- **Receita por categoria**
- **Mix de vendas** por produto/categoria
- **Participação geográfica** por cidade/estado do cliente

Observação: métricas partem de **transações efetivadas** (não contam pedidos cancelados ou não pagos).

## 7. regras de negócio essenciais

- **Pedido ≠ venda**: pedido é intenção; só vira venda ao **confirmar pagamento** (transação).
- Cada transação tem **uma data** analítica (dimensão tempo).
- **Distribuidor** é acionado apenas após transação efetivada.
- Produtos pertencem a **uma categoria** no escopo base.
- **Avaliações** são opcionais no DW; podem ser modeladas depois conforme necessidade.

## 8. jornada resumida

Pesquisa e descoberta → seleção de produto → criação de pedido (intenção) → pagamento efetuado (transação) → expedição e entrega → avaliação opcional do produto.

Para análise, o grão adotado é **“uma linha por produto vendido em uma transação em uma data”**.

## 9. entidades e dados (visão conceitual)

- **Produto**: id, nome, descrição, preço, categoria.
- **Categoria**: id, nome, descrição.
- **Cliente**: id, nome, endereço, cidade, estado/país, data de cadastro.
- **Pedido**: id, data, status, valor itens, frete.
- **Transação**: id, pedido_id, data, valor, meio de pagamento, status.
- **Distribuidor**: id, nome, cidade, país.
- **Entrega**: pedido_id, opção de entrega, custo, status, datas.
- **Avaliação (opcional)**: id, produto_id, cliente_id, nota, comentário, data.

No DW, **pedidos** não entram no escopo base; foco é **transação**.

## 10. decisões de modelagem para o DW

- **Esquema dimensional** em **star schema**.
- **Fato**: `dw.vendas`
  - Grain: produto × transação × data.
  - Medidas: `quantidade_vendida`, `faturamento`, `custo_frete`.
  - FKs: `cliente_id`, `produto_id`, `distribuidor_id`, `data_id`.

- **Dimensões**
  - `dw.cliente`: atributos de identificação e localização.
  - `dw.produto`: nome, descrição, preço, categoria (denormalizada no star).
  - `dw.distribuidor`: identificação e localização.
  - `dw.data`: data, dia, mês, ano, dia da semana.
  - **Opcionais**: `categoria` e `avaliacao` caso análises exijam normalização extra ou KPIs de satisfação.

- **Por que não levar pedido?** Para o escopo base, avaliar **vendas efetivas** simplifica KPIs e reduz ambiguidade.

## 11. requisitos não funcionais

- **Reprodutibilidade** local com Docker e PostgreSQL 16.
- **Performance** para consultas agregadas comuns (por data, produto, categoria).
- **Qualidade de dados** mínima garantida por ETL sintético controlado.
- **Segurança/privacidade**: sem PII real; apenas dados fictícios.
- **Portabilidade**: scripts `.sql` com **EOL LF** e estrutura de pastas em **snake_case**.

## 12. ambiente e ferramentas

- **Docker Desktop** para subir o PostgreSQL (`on_market_db`).
- **PostgreSQL 16** como SGBD do DW.
- **VS Code** com extensões recomendadas (Docker, PostgreSQL, SQLTools, Markdown).
- **Git/GitHub** para versionamento e documentação do estudo de caso.

## 13. estrutura do repositório (resumo)

- `docs/01_definicao_problema/contexto_negocio.md` (este arquivo)
- `ddl/fisico/modelo_fisico.sql`
- `etl/sql/processo_etl_clientes.sql`
- `etl/sql/processo_etl_produtos.sql`
- `etl/sql/processo_etl_distribuidor.sql`
- `etl/sql/processo_etl_data.sql`
- `etl/sql/processo_etl_vendas.sql`
- `docker/postgres/` com instruções e auxiliares
- `tests/sql/` para smoke/performance queries

## 14. roadmap incremental

- v0.1.0 documentação do problema e escopo
- v0.2.0 modelo conceitual e glossário
- v0.3.0 star schema e KPIs
- v0.4.0 modelo lógico e físico (PostgreSQL)
- v0.5.0 ETL sintético e testes
- v1.0.0 release pública do estudo de caso

## 15. riscos e mitigação

- **Simplificações do domínio**: documentar suposições e abrir pontos de extensão.
- **Dados sintéticos**: calibrar ranges e distribuições para evitar vieses irreais.
- **Escala local**: indicar limites e caminhos de evolução (particionamento, índices, cloud DW).

## 16. critérios de aceitação

1. Banco `on_market_dw` criado e acessível via Docker.
2. Esquema `dw` com dimensões e fato implantados conforme DDL.
3. Funções de ETL executadas com sucesso e dados carregados.
4. Consultas de validação em `tests/sql` retornando volumes e agregações coerentes.
5. Documentação mínima publicada em `docs/` e README do repositório.

## 17. extensões futuras

- Dimensão **vendedor** e **oferta** para cenários de marketplace puro.
- **Promoções** e **campanhas** com impacto em preço e margem.
- **SLA de entrega** (OTIF) com granularidade logística mais fina.
- **NLU** para análise de sentimento em avaliações (quando ativadas).
- Orquestração de ETL com ferramentas dedicadas.

## 18. glossário rápido

- **DW (Data Warehouse)**: repositório analítico, consolidado e orientado a consulta.
- **Star schema**: fato no centro e dimensões ao redor, modelo desnormalizado.
- **Grão (grain)**: nível de detalhe de uma linha da fato.
- **Pedido**: intenção de compra; não é venda até confirmação do pagamento.
- **Transação**: confirmação financeira de um pedido.
- **Distribuidor**: responsável pela entrega após transação.
- **Ticket médio**: faturamento dividido pelo número de transações.
- **ETL sintético**: geração controlada de dados fictícios para estudo/teste.

## 19. como reproduzir localmente (visão rápida)

1. Subir o PostgreSQL via Docker (task do VS Code ou comando `docker run`).
2. Executar `ddl/fisico/modelo_fisico.sql`.
3. Executar os scripts de `etl/sql` na ordem: clientes → produtos → distribuidor → data → vendas.
4. Rodar consultas de smoke em `tests/sql`.

Este contexto guia as decisões de modelagem e a implementação do DW do **On Market** como estudo de caso público e reprodutível.
