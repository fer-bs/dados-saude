# Dicionário de Dados — Despesas SMS Goiânia

Este documento descreve a estrutura dos arquivos brutos em `data/raw/`, coletados do Portal da Transparência da Prefeitura de Goiânia (sistema TransWeb, consulta de Despesas Gerais filtrada pela Secretaria Municipal de Saúde). Ele é a referência para quem for limpar, analisar ou reutilizar os dados.

## 1. Visão geral dos arquivos

| Item | Descrição |
|---|---|
| Localização | `data/raw/2025/` (jan–dez/2025) e `data/raw/2026/` (jan–jun/2026) |
| Nomenclatura | `Despesas_AAAAMMDD_AAAAMMDD.csv` (1 arquivo por mês; datas = início e fim do mês) |
| Separador | `;` (ponto e vírgula) |
| Encoding | Latin-1 / ISO-8859-1 (**não é UTF-8** — abrir como UTF-8 corrompe acentos) |
| Decimal | vírgula (`600000,00`), sem separador de milhar |
| Cabeçalho | 1ª linha de cada arquivo, com os nomes de coluna originais do TransWeb |
| Rodapé | últimas ~4 linhas de cada arquivo são totalizações (`Total Empenhado`, `Total Liquidado`, `Total Pago`) e uma linha separadora — **não são registros de despesa**, devem ser descartadas na limpeza |
| Granularidade | cada linha é um evento de **pagamento** dentro de um empenho. Um mesmo `Empenho` se repete em quantas linhas quantos forem os seus pagamentos (observado até 132 repetições) |
| Escopo de cada arquivo mensal | contém os empenhos **emitidos** naquele mês, com **todos** os seus pagamentos, inclusive os ocorridos em meses/anos posteriores. Ex.: um empenho de janeiro/2025 pode ter uma linha com `DataPagamento` em maio/2025 — ela aparece no arquivo de janeiro, não no de maio |

## 2. Colunas

| # | Coluna | Tipo | Descrição | Exemplo |
|---|---|---|---|---|
| 1 | `Empenho` | texto (18 dígitos) | Número do empenho orçamentário. Identifica a reserva de verba para a despesa. Repete-se em várias linhas (1 por pagamento/liquidação do mesmo empenho). | `20252150000630001` |
| 2 | `DataEmpenho` | data/hora (`dd/mm/aaaa 00:00:00`) | Data em que o empenho foi emitido. | `16/01/2025 00:00:00` |
| 3 | `VlEmpenhado` | decimal (BR, vírgula) | Valor total empenhado. **Repete-se em todas as linhas do mesmo empenho** — somar direto infla o total (~6x observado). Para valor empenhado, deduplicar por `Empenho` antes de somar. | `600000,00` |
| 4 | `Liquidacao` | texto (10 dígitos) | Número do documento de liquidação (etapa em que a Prefeitura reconhece a dívida, após a entrega do bem/serviço). | `0009692025` |
| 5 | `DataLiquidacao` | data/hora | Data da liquidação correspondente à linha. | `29/01/2025 00:00:00` |
| 6 | `VlLiquidado` | decimal (BR) | Valor liquidado nessa linha (ao contrário de `VlEmpenhado`, varia por linha — é o valor da parcela liquidada). | `4347,87` |
| 7 | `OrdemPagamento` | texto (22 dígitos) | Número da ordem de pagamento (etapa final, o desembolso efetivo). | `202521500006300010001` |
| 8 | `DataPagamento` | data/hora | Data em que o pagamento foi efetivamente feito. **Coluna a usar para qualquer análise temporal/mensal** (não o mês do nome do arquivo). | `30/01/2025 00:00:00` |
| 9 | `VlPago` | decimal (BR) | Valor efetivamente pago nessa linha. **Coluna a usar para "quanto foi gasto"**. Pode ser `0,00` quando o empenho/liquidação ainda não foi pago na data da extração — não é erro de dado. | `4347,87` |
| 10 | `NaturezaDespesa` | código (8 dígitos) | Código da natureza de despesa, conforme classificação orçamentária padrão (categoria + grupo + modalidade + elemento). | `31909100` |
| 11 | `DsNaturezadaDespesa` | texto | Descrição textual do código acima (ex.: pessoal, serviços de terceiros, material de consumo). 18 valores distintos observados no mês de referência. | `SENTENCAS JUDICIAIS` |
| 12 | `CNPJ` | texto (14 dígitos, sem máscara) | CNPJ do órgão gestor da despesa (não do favorecido — ver `NmFavorecido`). Nos dados observados, corresponde ao CNPJ da própria SMS/Prefeitura ou de fundações intermediárias. | `60701190000104` |
| 13 | `NmOrgao` | texto | Nome do órgão orçamentário. No recorte coletado, é sempre `SECRETARIA MUNICIPAL DE SAUDE` (filtro já aplicado na coleta). | `SECRETARIA MUNICIPAL DE SAUDE` |
| 14 | `VlAnulado` | decimal (BR) | Valor anulado (estorno/cancelamento parcial) associado ao empenho. Repete-se por empenho, como `VlEmpenhado` — mesmo cuidado de deduplicação ao somar. `0,00` quando não há anulação. | `3112,54` |
| 15 | `UnidadeOrcamentaria` | texto (largura fixa, com padding de espaços) | Unidade orçamentária responsável. Requer `.Trim()` ao processar (vem preenchida com espaços à direita até largura fixa). No recorte coletado, é sempre `SECRETARIA MUNICIPAL DE SAUDE`. | `SECRETARIA MUNICIPAL DE SAUDE` |
| 16 | `Funcao` | texto categórico | Função orçamentária (classificação funcional, nível mais alto). Valores observados: `SAUDE - 10`, `ENCARGOS ESPECIAIS - 28`. | `SAUDE - 10` |
| 17 | `SubFuncao` | texto categórico | Subfunção orçamentária. Valores observados: `ADMINISTRACAO GERAL - 122`, `ATENCAO BASICA - 301`, `ASSISTENCIA HOSPITALAR E AMBULATORIAL - 302`, `SUPORTE PROFILATICO E TERAPEUTICO - 303`, `VIGILANCIA EPIDEMIOLOGICA - 305`, `OUTROS ENCARGOS ESPECIAIS - 846`. | `ASSISTENCIA HOSPITALAR E AMBULATORIAL - 302` |
| 18 | `FonteRecurso` | texto categórico | Fonte orçamentária do recurso (de onde vem o dinheiro: impostos, transferências SUS, taxas, etc.). ~6 valores distintos por mês, todos ligados a receitas vinculadas à saúde. | `RECEITAS DE IMPOSTOS E DE TRANSFERENCIAS DE IMPOSTOS - SAUDE - 102` |
| 19 | `NmFavorecido` | texto | Nome do beneficiário/credor do pagamento (banco operador da folha, fornecedor, hospital, fundação, pessoa física/jurídica). **Coluna-chave para a pergunta "para quem?"**. | `ITAU UNIBANCO S.A.` |
| 20 | `Objeto` | texto longo, livre | Campo de texto livre com o histórico/justificativa do empenho — geralmente concatena descrição do objeto, número de solicitação financeira, SEI, período de referência e, às vezes, um recibo formatado ("VALOR TOTAL......:"). Alta variação de formato, não estruturado; útil para auditoria pontual, não para agregações diretas. | `SOLICITACAO FINANCEIRA NR. 158044-2025 SEI: 25.29.000000934-7 ...` |
| 21 | `Modalidade` | texto categórico | Modalidade de licitação/contratação. Valores observados: `Não se aplica (Ex.: Despesas com Pessoal)`, `Outros (Convênios, ajustes, similares, etc.)`, `Dispensa de Licitação`, `Pregão`, `Inexigibilidade de Licitação`. | `Pregão` |
| 22 | `NumeroLicitacao` | texto (12 dígitos) | Número do processo licitatório associado, quando houver. `000000000000` quando não se aplica (despesas de pessoal, convênios, etc. — coerente com `Modalidade`). | `000034162024` |

## 3. Qualidade dos dados conhecida (sujeira)

- **Linhas de rodapé**: cada arquivo termina com uma linha separadora (`------...`) e 3 linhas de totalização (`Total Empenhado`, `Total Liquidado`, `Total Pago`). Devem ser removidas antes de qualquer análise — na limpeza, filtrar linhas cujo `Empenho` não seja um número válido é suficiente para descartá-las.
- **Linhas integralmente duplicadas**: existem linhas 100% repetidas (todas as colunas iguais) dentro do mesmo arquivo. Observado no arquivo de jan/2025: 1.300 linhas de dados, 1.296 únicas (4 duplicadas exatas). Deduplicar por linha completa antes de agregar.
- **`VlPago = 0,00`**: não é erro — indica que o empenho foi liquidado mas ainda não pago na data de extração. No arquivo de jan/2025, 340 das 1.300 linhas têm `VlPago = 0,00` (~26%). Incluir ou não essas linhas depende da pergunta: para "quanto foi pago" devem ser mantidas (contribuem 0) mas não descartadas, pois representam liquidações reais; para "quanto está pendente de pagamento" são o próprio universo de interesse.
- **`UnidadeOrcamentaria` e `NmOrgao` com padding**: `UnidadeOrcamentaria` vem preenchido com espaços à direita até largura fixa (~250 caracteres). Aplicar `TRIM()` sempre que for usada como chave de agrupamento ou junção.
- **`Objeto` não estruturado**: texto livre, às vezes com múltiplas informações concatenadas sem separador consistente (histórico de empenho, SEI, solicitação financeira, recibos formatados com pontos de preenchimento). Não usar para agregação; útil apenas para leitura manual/auditoria de casos específicos.

## 4. Regras de negócio para agregação

| Pergunta | Coluna a somar | Cuidado |
|---|---|---|
| Quanto foi efetivamente gasto (por período, favorecido, natureza etc.) | `VlPago` | Somar direto — o valor já é por linha/pagamento, não se repete por empenho. |
| Quanto foi empenhado no total | `VlEmpenhado` | **Deduplicar por `Empenho` antes de somar** (pegar 1 linha por empenho, ex. `MIN`/`FIRST`), senão o total é inflado pelo número de pagamentos de cada empenho. |
| Quanto foi anulado | `VlAnulado` | Mesmo cuidado do `VlEmpenhado` — repete por empenho, deduplicar por `Empenho`. |
| Distribuição mensal de pagamentos | agrupar por `DataPagamento` (mês) | Não usar o mês do nome do arquivo — um arquivo de janeiro contém pagamentos de meses futuros. |
| Maiores favorecidos | agrupar `VlPago` por `NmFavorecido` | Considerar separar o(s) favorecido(s) que operam a folha de pessoal (ex.: banco) do restante, pois dominam o ranking e distorcem a leitura de "fornecedores". |

## 5. Chaves e granularidade

- **Chave natural da linha**: não há uma coluna de ID único de linha; a combinação `Empenho` + `OrdemPagamento` identifica um evento de pagamento de forma única (ou a linha completa, dado o caso de duplicatas exatas).
- **Hierarquia**: 1 `Empenho` → N `Liquidacao` → N `OrdemPagamento`. Nem toda liquidação já teve pagamento (daí `VlPago = 0,00`).
- **Junção entre arquivos/anos**: os arquivos de 2025 e 2026 têm o mesmo esquema de colunas e podem ser empilhados diretamente (`UNION`) após a limpeza, desde que a deduplicação de `VlEmpenhado`/`VlAnulado` seja feita por `Empenho` já no conjunto empilhado (um empenho de dez/2025 pode ter pagamentos capturados só no arquivo de 2026, se a coleta janela permitir).

## 6. Fora de escopo (o que estes dados não têm)

- **Localização geográfica** do serviço prestado (bairro/região) — não existe no dataset.
- **Remuneração individual de servidores** — a folha aparece agregada por natureza de despesa e paga via banco operador; não há detalhamento por servidor.
