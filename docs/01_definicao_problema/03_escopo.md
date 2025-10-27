# On Market - escopo do projeto

Documento de escopo do **On Market**, estudo de caso público de **Data Warehouse (DW) de e-commerce** com implementação local via **Docker + PostgreSQL 16** e uso do **VS Code**. Este arquivo detalha objetivos, limites, requisitos, entregáveis, critérios de aceite e padrões técnicos.

## 1. objetivo do documento
Estabelecer **o que entra e o que não entra** na primeira versão do DW, como os dados serão modelados, carregados e validados, quais decisões técnicas foram tomadas e quais são os critérios para considerar a versão pronta.

## 2. visão geral do escopo
1. Construção de um **DW analítico mínimo viável** para e-commerce generalista.
2. Foco exclusivo em **transações efetivadas** (vendas) e métricas de **vendas, operação e cliente**.
3. Implementação local, reprodutível, com **scripts SQL** e **dados sintéticos**.
4. Documentação completa do processo em `docs/`.

## 3. no escopo (v1)
1. **Modelagem dimensional (star schema)** com:
   * fato `dw.vendas` no grão **produto × transação × data**.
   * dimensões `dw.cliente`, `dw.produto`, `dw.distribuidor`, `dw.data`.
   * dimensões opcionais documentadas (`categoria`, `avaliacao`) para versões futuras.
2. **Modelo físico** em PostgreSQL 16: criação do schema `dw`, tabelas, FKs.
3. **ETL sintético** por funções PL/pgSQL que populam dimensões e fato.
4. **Consultas de smoke/performance** em `tests/sql/` para validar consistência.
5. **Ambiente Docker** para subir PostgreSQL de forma reproduzível.
6. **Padrões de versionamento** e organização de repositório (snake_case, EOL LF, .gitignore/.gitattributes).
7. **Documentação** de contexto, perguntas de negócio, escopo, dimensional e físico.

## 4. fora do escopo (v1)
1. Sistema transacional real, APIs, front-end ou back-office de e-commerce.
2. **Marketplace multi-seller completo** (vendedor, oferta, comissão/repasse).
3. Impostos detalhados, múltiplas moedas, conciliação financeira bancária.
4. Orquestração de pipelines, agendamento e monitoramento (Airflow/dbt/etc.).
5. Dados reais/PII; o estudo usa **dados fictícios** gerados no ETL.
6. Métricas de **margem** (sem custo de produto) e **SLA logístico** detalhado.
7. Histórico de **preço temporal** (preço estático no v1).

## 5. requisitos de dados
1. **Grão da fato**: uma linha por **produto** vendido **em uma transação** **em uma data**.
2. **Chaves**:
   * FKs na fato: `cliente_id`, `produto_id`, `distribuidor_id`, `data_id`.
   * PKs simples nas dimensões com `serial` (v1).
3. **Atributos mínimos**:
   * cliente: nome, endereço, cidade, país (sem PII sensível).
   * produto: nome, descrição, preço, categoria (como atributo).
   * distribuidor: nome, cidade, país.
   * data: data, dia, mês, ano, dia_da_semana.
4. **Medidas**:
   * `quantidade_vendida`, `faturamento`, `custo_frete`.
5. **Geração**: dataset sintético via funções PL/pgSQL; ranges e aleatoriedade controlados.

## 6. modelo de dados alvo
1. **Fato** `dw.vendas`
   * FKs: `cliente_id`, `produto_id`, `distribuidor_id`, `data_id`.
   * Medidas: `quantidade_vendida` (INT), `faturamento` (DECIMAL), `custo_frete` (DECIMAL).
2. **Dimensões**
   * `dw.cliente`: identidade e localização.
   * `dw.produto`: especificação e preço (preço atual, não histórico).
   * `dw.distribuidor`: identidade e localização.
   * `dw.data`: hierarquia temporal padrão.
3. **Opcionais (backlog)**
   * `dw.categoria` como dimensão normalizada (snowflake leve).
   * estrutura para **avaliação** (análise de satisfação).

## 7. regras de negócio
1. **Pedido ≠ venda**: apenas transações confirmadas entram na fato.
2. **Distribuidor** só participa após confirmação da transação.
3. Produto pertence a **uma categoria** (v1).
4. Métricas derivadas usam apenas dados do DW (sem fontes externas).

## 8. métricas e KPIs
1. Básicas: quantidade vendida, faturamento, custo de frete.
2. Derivadas:
   * ticket_medio = faturamento / número_de_transacoes
     observação: ideal ter `transacao_id`; v1 usa aproximação por agregação.
   * peso_do_frete = custo_frete / faturamento.
   * participação_categoria = faturamento_categoria / faturamento_total.

## 9. qualidade de dados (checks mínimos)
1. **Integridade referencial**: 100% das FKs válidas.
2. **Not null**: chaves e medidas não nulas na fato.
3. **Domínios/ranges**:
   * `quantidade_vendida` > 0
   * `faturamento` ≥ 0
   * `custo_frete` ≥ 0 e ≤ `faturamento` × limite (ex.: 0,3)
4. **Cardinalidade**: PKs únicas nas dimensões.
5. **Consistência temporal**: `data_id` existente na dimensão data.

## 10. desempenho e projeto físico
1. **Índices**:
   * FKs da fato (btree) para acelerar joins por dimensão.
   * Índice composto opcional em (`data_id`, `produto_id`) para agregações comuns.
2. **Particionamento** (opcional, backlog): por mês/ano em `dw.vendas`.
3. **Manutenção**: `ANALYZE`, `VACUUM` conforme necessidade.
4. **Volumetria de referência**:
   * dimensão data: 5 anos ≈ 1826 linhas.
   * clientes: 10k–100k (ajustável no ETL).
   * produtos: 1k–50k.
   * fato vendas: 10k–5M (sintético, conforme testes).

## 11. segurança e privacidade
1. **Sem PII real**; nomes e endereços são fictícios.
2. Credenciais de exemplo **não** representam produção.
3. `.env` e arquivos sensíveis **fora do versionamento**.
4. Padrão de EOL LF e binários sinalizados em `.gitattributes`.

## 12. ambiente e execução
1. **Docker**: PostgreSQL 16 com container `on_market_db`.
2. **VS Code**: tasks para subir/parar/remover container e abrir psql.
3. **Extensões recomendadas** em `.vscode/extensions.json`.
4. **Execução local**:
   * criar estruturas com `ddl/fisico/modelo_fisico.sql`.
   * popular com `etl/sql/*` na ordem definida (ver seção 14).

## 13. entregáveis
1. **Documentos** em `docs/`: contexto, perguntas, escopo, dimensional, lógico/físico, KPIs.
2. **DDL** em `ddl/fisico/modelo_fisico.sql`.
3. **ETL sintético** em `etl/sql/`:
   * `processo_etl_clientes.sql`
   * `processo_etl_produtos.sql`
   * `processo_etl_distribuidor.sql`
   * `processo_etl_data.sql`
   * `processo_etl_vendas.sql`
4. **Testes**:
   * `tests/sql/smoke_queries.sql`
   * `tests/sql/performance_queries.sql`

## 14. ordem de execução (local)
1. Subir PostgreSQL via task **docker: up postgres**.
2. Executar `ddl/fisico/modelo_fisico.sql`.
3. Executar ETL na ordem:
   * `etl/sql/processo_etl_clientes.sql`
   * `etl/sql/processo_etl_produtos.sql`
   * `etl/sql/processo_etl_distribuidor.sql`
   * `etl/sql/processo_etl_data.sql`
   * `etl/sql/processo_etl_vendas.sql`
4. Validar com `tests/sql/smoke_queries.sql`.

## 15. critérios de aceite (DoD)
1. Todas as tabelas e FKs criadas no schema `dw`.
2. ETL executa sem erro e popula dimensões e fato com volumes coerentes.
3. Queries de smoke retornam:
   * totais de quantidade/faturamento por mês.
   * top produtos por faturamento.
   * ticket_medio calculável.
4. Checks de qualidade atendidos (integridade, ranges, not null).
5. Documentação atualizada em `docs/` e README.

## 16. riscos e mitigação
1. **Simplificação do domínio**: documentar suposições; abrir backlog de extensões.
2. **Dados aleatórios enviesados**: ajustar funções geradoras para distribuições mais realistas.
3. **Escala local**: indicar limites; planejar particionamento e índices se necessário.
4. **Ausência de `transacao_id`**: previsto em backlog para métricas “por transação”.

## 17. backlog e extensões (v2+)
1. **Adicionar `transacao_id`** na fato e contagem de transações.
2. **Dimensão categoria** normalizada (snowflake leve).
3. **Entrega/OTIF**: datas de expedição/entrega e status logístico.
4. **Histórico de preços** (SCD / tabela de vigência).
5. **Avaliações** com nota/comentário e análises de correlação.
6. **Marketplace**: vendedor, oferta, comissão/repasse.

## 18. convenções e padrões
1. **Snake_case** para arquivos/pastas e nomes de objetos do banco.
2. **EOL LF** para `.sql`, `.md`, `.yml`; CRLF apenas para `.bat/.cmd`.
3. **Commits** com prefixos curtos (ex.: `feat`, `chore`, `docs`, `fix`).
4. **Workflow**: desenvolvedor único; trabalhar direto em `main` ou usar branches `feature/…` curtas.
5. **Diretórios chave**:
   * `docs/01_definicao_problema/contexto_negocio.md`
   * `docs/01_definicao_problema/perguntas_negocio.md`
   * `docs/01_definicao_problema/escopo.md` (este arquivo)
   * `ddl/fisico/modelo_fisico.sql`
   * `etl/sql/*.sql`
   * `tests/sql/*.sql`

## 19. estimativas de esforço (v1)
1. Modelagem e DDL: 1–2 dias.
2. ETL sintético: 1 dia.
3. Testes e ajustes de performance: 0,5–1 dia.
4. Documentação: 0,5–1 dia.

## 20. plano de testes
1. **Smoke**: contagens por tabela; joins fato×dimensões; totais por mês.
2. **Qualidade**: checagem de ranges, not null, integridade referencial.
3. **Performance**: agregações por mês e categoria com e sem índices.
4. **Reprocesso**: rerun das funções ETL para idempotência controlada (limpeza prévia, se necessário).

## 21. dependências
1. Docker Desktop.
2. Imagem `postgres:16.1`.
3. VS Code e extensões recomendadas.
4. Git.

## 22. plano de comunicação
1. README enxuto com visão geral e passos rápidos.
2. Documentos detalhados em `docs/`.
3. Changelog evolutivo em `CHANGELOG.md` (quando adotado).

---

Este escopo orienta a entrega da **v1** do **On Market DW**, garantindo reprodutibilidade local, clareza de limites e base sólida para evoluções futuras (logística, marketplace, preço histórico e avaliações).
