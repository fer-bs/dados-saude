# Passo a Passo — Desafio 1.1: Do Caos ao Dashboard

**Projeto:** Gastos da Secretaria Municipal de Saúde de Goiânia (2025 + validação 1º sem. 2026)
**Ferramentas:** Google Sheets + Looker Studio + GitHub
**Regra anti-paralisia:** cada etapa tem um critério de "pronto". Bateu o critério, avança. Perfeição fica para a v2.

> Este passo a passo já incorpora o perfil real dos dados (exploração feita em jul/2026):
> CSV `;` em Latin-1, granularidade por pagamento, rodapés sujos e risco de dupla contagem.

---

## Etapa 0 — Matéria-prima no lugar ✅ (concluída)

- [x] 12 arquivos de 2025 + 6 de 2026 em `data/raw/`, commitados no GitHub
- [x] Formato confirmado: CSV, separador `;`, encoding Latin-1, decimal com vírgula

---

## Etapa 1 — Reconhecimento e dicionário de dados

- [ ] Importar UM arquivo no Google Sheets: `Arquivo > Importar > separador ";"`. Se acentos vierem quebrados (ç, ã), reimportar indicando encoding ISO-8859-1 — registrar isso no log
- [ ] Escrever o `docs/dicionario-dados.md` com as 22 colunas. Dedicar atenção especial ao trio do ciclo da despesa:
  - **Empenho** (`DataEmpenho`, `VlEmpenhado`) = compromisso/reserva do orçamento
  - **Liquidação** (`DataLiquidacao`, `VlLiquidado`) = reconhecimento de que o serviço foi entregue
  - **Pagamento** (`DataPagamento`, `VlPago`) = dinheiro que efetivamente saiu
- [ ] Registrar a decisão central: **"quanto se gasta" = soma de `VlPago`**. `VlEmpenhado` só pode ser somado após deduplicar por `Empenho` (ele se repete em cada linha do mesmo empenho — somar direto infla ~6x)
- [ ] Registrar a natureza do filtro dos arquivos: cada arquivo traz os empenhos **emitidos** naquele mês, com pagamentos que podem cair em meses/anos seguintes → análise mensal usa `DataPagamento`

**Pronto quando:** o dicionário existe e você explica de memória a diferença entre empenhado, liquidado e pago — e por que somar `VlEmpenhado` bruto é errado.

---

## Etapa 2 — Consolidação

- [ ] Conferir que todos os arquivos têm as mesmas 22 colunas na mesma ordem
- [ ] Empilhar os 12 meses de 2025 na aba `base_2025` e os 6 de 2026 na aba `base_2026_validacao`
- [ ] Criar coluna `arquivo_origem` para rastrear de qual mês/arquivo veio cada linha
- [ ] Conferência de sanidade: nº de linhas da base = soma das linhas dos arquivos (esperado 2025: ~8,6 mil linhas úteis + rodapés que sairão na limpeza)

**Pronto quando:** existe uma única tabela por ano e a contagem de linhas bate com os originais.

---

## Etapa 3 — Limpeza (com log de decisões)

Para cada item, registrar no `docs/log-limpeza.md`: **o que encontrei → o que decidi → por quê**. Problemas já mapeados na exploração:

- [ ] **Rodapés do portal:** remover as linhas de totalização ("Total Empenhado", "Total Liquidado") e as linhas de traços — são ~4 por arquivo (~48 em 2025), identificáveis por `NmFavorecido` vazio
- [ ] **Duplicatas integrais:** existem ~242 linhas 100% idênticas em 2025. Investigar por amostragem e deduplicar usando `OrdemPagamento` como chave (cada ordem de pagamento deve aparecer uma vez)
- [ ] **Valores:** converter texto → número (remover ponto de milhar, trocar vírgula por ponto se necessário conforme configuração regional da planilha)
- [ ] **Datas:** padronizar `dd/mm/aaaa hh:mm:ss` → data simples
- [ ] **Linhas com `VlPago = 0` e sem `DataPagamento`** (~1,6 mil em 2025): NÃO são erro — são empenhos/liquidações ainda não pagos. Decidir: manter na base (servem para análise de empenhado vs. pago) e filtrar nos cálculos de gasto
- [ ] **Favorecidos:** padronizar caixa/espaços; verificar variações do mesmo nome usando `CNPJ` como chave
- [ ] **Coluna derivada `grupo_gasto`:** classificar cada linha em **Pessoal** (vencimentos, obrigações patronais, auxílios, terceirização de mão de obra) vs. **Custeio/Serviços** (demais) — essa separação sustenta a pergunta 1 e evita a leitura distorcida do banco da folha como "maior fornecedor"
- [ ] **Validação final:** total de `VlPago` da base limpa vs. total exibido no próprio portal para o período (registrar a conferência, mesmo aproximada)

**Pronto quando:** a `base_limpa` não tem problema conhecido sem registro no log, e o log explica cada decisão.

---

## Etapa 4 — Análise exploratória (respondendo as perguntas)

Uma aba por pergunta, com tabelas dinâmicas sobre a `base_limpa` (filtrando `VlPago > 0` quando a métrica for gasto):

- [ ] **P1 — Quanto e em quê:** total pago no ano; abertura Pessoal vs. Custeio; dentro de cada grupo, as maiores naturezas (`DsNaturezadaDespesa`)
- [ ] **P2 — Para quem:** ranking de favorecidos por `VlPago` **excluindo o grupo Pessoal**; % acumulado (Pareto). Referências da exploração: top 10 geral ≈ 78% do total; principais entidades: Assoc. Combate ao Câncer, Fundação de Apoio à Gestão em Saúde, Santa Casa, Hospital Ruy Azeredo
- [ ] **P3 — Quando:** total pago por mês de `DataPagamento`; gráfico de linha; base estável ~R$ 170–200 mi/mês
- [ ] **P4 — Outliers:** investigar o pico de outubro/2025 (R$ 227 mi): qual natureza/favorecido explica? Listar os 5 maiores pagamentos individuais do ano com uma linha de contexto cada (usar a coluna `Objeto`, que descreve a despesa)
- [ ] Escrever 3–5 achados em linguagem de negócio (frase que um jornalista usaria)

**Pronto quando:** cada pergunta tem resposta escrita em uma ou duas frases, com o número que a sustenta.

---

## Etapa 5 — Validação com 2026

- [ ] Aplicar na base 2026 a MESMA limpeza documentada no log (teste de reprodutibilidade do seu processo)
- [ ] Atenção ao recorte: os arquivos de 2026 trazem empenhos de 2026; pagamentos de empenhos de 2025 que caíram em 2026 estão nos arquivos de 2025. Definir e registrar o critério de comparação (sugestão: pago por mês de `DataPagamento`, cada ano com seus empenhos, comparando jan–jun/26 vs. jan–jun/25)
- [ ] Comparar mês contra mês do ano anterior (jan vs. jan) — nunca contra média anual
- [ ] Responder por escrito: a tendência se confirmou? O ranking de favorecidos (ex-folha) mudou? Alguma categoria acelerou ou caiu?

**Pronto quando:** existe um parágrafo dizendo explicitamente se 2026 confirma ou refuta o padrão de 2025 — e sob qual critério de comparação.

---

## Etapa 6 — Dashboard no Looker Studio

- [ ] Conectar o Looker Studio à planilha limpa
- [ ] Tela única com: (a) big numbers — total pago, % pessoal vs. custeio, nº de favorecidos; (b) linha do tempo mensal; (c) barras por natureza de despesa; (d) top 10 favorecidos ex-folha; (e) filtros por mês e por grupo de gasto
- [ ] Título que conta a história (ex.: "Para onde foi o dinheiro da saúde de Goiânia em 2025")
- [ ] Teste do usuário leigo: dá para responder as 4 perguntas só clicando?
- [ ] Publicar e gerar link compartilhável

**Pronto quando:** uma pessoa que nunca viu o projeto responde as 4 perguntas usando só o dashboard.

---

## Etapa 7 — Publicação no portfólio

- [ ] Atualizar o README: link do dashboard + "Principais achados" com os números finais
- [ ] Conferir estrutura do repo (raw intocado, `data/clean` e `docs` completos)
- [ ] Revisão final de clareza — o README é sua vitrine para clientes na Workana
- [ ] Commit final + push

**Pronto quando:** você mandaria o link do repositório para um cliente em potencial sem hesitar.
