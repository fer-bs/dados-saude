# Do Caos ao Dashboard: Gastos da Secretaria Municipal de Saúde de Goiânia — 2025

Projeto de portfólio: análise exploratória e dashboard interativo sobre a execução de despesas da Secretaria Municipal de Saúde (SMS) de Goiânia no exercício de 2025, com validação do padrão observado no 1º semestre de 2026.

## Contexto (cliente fictício)

Um veículo de jornalismo local contratou uma análise independente dos gastos da SMS de Goiânia para embasar uma série de reportagens sobre orçamento público. O cliente não é técnico: o produto final precisa responder perguntas de negócio em linguagem clara, com um dashboard que qualquer editor consiga explorar sozinho.

## Perguntas de negócio

1. **Quanto e em quê?** Qual o total efetivamente pago em 2025 e como ele se distribui entre folha de pessoal e custeio/serviços? Quais as maiores naturezas de despesa em cada grupo?
2. **Para quem?** Excluída a folha (operacionalizada via banco), quais fornecedores e entidades de saúde mais receberam? Os 10 maiores concentram quanto % do total (análise de Pareto)?
3. **Quando?** Como os pagamentos se distribuem mês a mês (por data de pagamento)? Existe tendência ou sazonalidade?
4. **Algo fora do padrão?** Há outliers — meses, lançamentos ou favorecidos com comportamento fora da curva (ex.: o pico de outubro/2025)?
5. **(Validação)** O padrão de 2025 se sustenta nos empenhos e pagamentos de janeiro a junho de 2026?

### Decisões de escopo (perguntas descartadas conscientemente)

- **Distribuição geográfica dos gastos** — os dados não trazem localização (bairro/região) dos serviços prestados. Fora do escopo.
- **Remuneração individual de servidores** — a folha aparece apenas de forma agregada (via natureza de despesa). Detalhamento exigiria o dataset de folha de pagamento, que é outra fonte.

Documentar o que os dados **não** respondem faz parte da entrega: evita conclusões indevidas pelo cliente.

## Fonte de dados

Portal da Transparência da Prefeitura de Goiânia — consulta de Despesas Gerais (sistema TransWeb), com filtro pelo órgão Secretaria Municipal de Saúde:
`https://www.goiania.go.gov.br/sing_transparencia/despesas-gerais/`

Coleta por download manual, mês a mês: jan–dez/2025 (análise) e jan–jun/2026 (validação). Arquivos brutos preservados sem alteração em `data/raw/` — toda limpeza acontece em cópias.

## Características técnicas dos dados (leia antes de reutilizar)

- **Formato:** CSV com separador `;`, encoding **Latin-1 (ISO-8859-1)**, decimal com vírgula.
- **Granularidade:** cada linha é um evento de **pagamento/liquidação**; um mesmo empenho aparece em várias linhas (até 132). Consequência: `VlEmpenhado` se repete por linha — **somar essa coluna diretamente infla o total em ~6x**. Para valor empenhado, deduplicar por `Empenho`; para "quanto foi gasto", usar `VlPago`.
- **Filtro dos arquivos:** cada arquivo mensal contém os empenhos **emitidos** naquele mês, com todos os seus pagamentos (inclusive de meses e anos seguintes). Análises mensais devem usar `DataPagamento`, não o mês do arquivo.
- **Sujeira conhecida:** linhas de totalização e separadores no rodapé de cada arquivo; linhas integralmente duplicadas; linhas com `VlPago = 0` (empenhado/liquidado ainda não pago — não são erro).
- **Colunas principais:** `Empenho`, `DataEmpenho`, `VlEmpenhado`, `DataLiquidacao`, `VlLiquidado`, `OrdemPagamento`, `DataPagamento`, `VlPago`, `NaturezaDespesa`/`DsNaturezadaDespesa`, `CNPJ`, `NmFavorecido`, `Funcao`, `SubFuncao`, `FonteRecurso`, `Objeto`, `Modalidade`, `NumeroLicitacao`.

Dicionário completo em `docs/dicionario-dados.md`; decisões de limpeza em `docs/log-limpeza.md`.

### Nota sobre dados pessoais (LGPD)

A fonte oficial inclui, no campo `Objeto`, dados pessoais de pessoas físicas (CPF e dados bancários de beneficiários de auxílios), publicados pela própria Prefeitura por força da legislação de transparência. Este projeto não realça nem redistribui esses dados: as análises e o dashboard trabalham apenas com informações agregadas (favorecido, natureza, valor e data), e nenhuma visualização expõe CPF ou conta bancária de indivíduos.

## Stack (100% gratuita)

- **Coleta:** download manual no portal (a API de dados abertos do município não expôs microdados de despesa no momento da coleta)
- **Limpeza e análise exploratória (EDA):** Google Sheets, com log de decisões documentado — [planilha de trabalho](https://docs.google.com/spreadsheets/d/1eGKH3fbfMlbLgkpZey8lyECBnbktY34-Gfe5AjESHYU/edit?usp=sharing)
- **Dashboard:** Looker Studio (link publicado abaixo)
- **Versionamento:** Git + GitHub

## Estrutura do repositório

```
dados-saude/
├── data/
│   ├── raw/2025/        # 12 arquivos mensais originais, intocados
│   ├── raw/2026/        # jan–jun para validação
│   └── clean/           # base tratada e consolidada
├── docs/
│   ├── dicionario-dados.md
│   └── log-limpeza.md
├── README.md
└── LICENSE
```

## Dashboard

🔗 *[link do Looker Studio — publicar ao final]*

## Principais achados

*[Preencher ao final — candidatos já identificados na exploração inicial:]*

- R$ 2,29 bilhões efetivamente pagos em 2025, com pagamentos estáveis entre R$ 170–200 mi/mês e pico em outubro (R$ 227 mi)
- O maior favorecido nominal é o banco operador da folha (R$ 1,07 bi, ~47% do total) — separar folha de custeio muda completamente a leitura de "quem recebe"
- Excluída a folha, os maiores recebedores são entidades hospitalares e filantrópicas (Associação de Combate ao Câncer, Fundação de Apoio à Gestão em Saúde, Santa Casa)
- Alta concentração: os 10 maiores favorecidos respondem por ~78% do total pago, entre ~800 favorecidos

## Está bom quando…

- [ ] O dashboard responde as 4 perguntas principais em uma tela
- [ ] O dicionário de dados e o log de limpeza existem e são compreensíveis por um não-técnico
- [ ] Os totais publicados não sofrem de dupla contagem de `VlEmpenhado` (conferido contra a soma deduplicada)
- [ ] A validação com 2026 confirma ou refuta explicitamente a tendência de 2025