# Plano — Aba "Despesas" (BI por empresa × centro de custo)

**Escrito em:** 26/09/2026, sessão de casa (chat claude.ai, sem alterar código).
**Situação (corrigida às 19:40):** a nota anterior sobre repositório defasado
estava errada — vinha de um clone antigo. O GitHub está atualizado (último
commit 05/08). **Fases 1 e acesso por diretor foram implementados nesta mesma
sessão; ver HANDOFF.md para o estado atual.** O texto abaixo é o plano original.

## Objetivo

Nova aba **Despesas** no Apex Financeiro para que diretores de área enxerguem o
perfil de gastos do que é deles: por empresa do grupo, por centro de custo (CC),
por fornecedor e por natureza da despesa, mês a mês, com drill-down até o
lançamento.

## O que já existe e será reaproveitado

- `lancamentos` (~10 mil linhas): `cod`, `emp`, `data`, `tipo`, `nominal`
  (fornecedor), `hist`, `obra`, `desp`, `rec`, `origem`.
- **O campo `obra` É o centro de custo do UAU.** Ex.: `1AD - ADMINISTRACAO APEX
  ENGENHARIA`, `1AC - ALMOXARIFADO CENTRAL APEX`, `099OB - OBRA RUA 24 NORTE
  LOTE 01`, `099VE - LÉT PARQUE`, `185VE - VENDAS EDIFÍCIO RESIDENCIAL`.
  O prefixo antes do " - " é o código do CC; o sufixo (AD/AC/OB/VE/FA) indica
  a família: administração, almoxarifado, obra, vendas, fazenda.
- `tipo` separa o que interessa: `*Pago` (despesa real, ~5.800 linhas,
  R$ 29 MM) vs `Transferência` (intercompany/entre contas) e `Cont.Financ.`
  (financiamento). **A aba Despesas usa só `*Pago`** (mais o que a
  classificação `at` do app já marca como operacional, se for mais preciso).
- Funções prontas: `loadGrupo()`, `normU()`, `gbucket()`/`gPerLabel()`
  (períodos), `gdrillShow()` (drill-down), `gOpAgg()` (agregação),
  `_kmoney`/`BRL`/`MM`, `pdfButtons()`, `_ghdr()` (PDF), `biView()`/`drawBI()`
  (modelo de aba com gráficos Chart.js). Não reinventar: copiar o padrão do BI.

## O que falta: natureza da despesa

O fluxo de caixa do UAU não traz plano de contas. **Não há, por enquanto,
relatório do UAU com conta financeira por lançamento** (confirmado em 26/09).
Solução em duas fases.

### Fase 1 — classificação por regras (fazer agora)

Função `natDesp(r)` que retorna uma categoria a partir de
`trim(normU(nominal + ' ' + hist))`. **O `trim` é obrigatório** — sem ele a
regra de pessoa física falha (hist vazio deixa espaço no fim). Ordem importa
(primeira que casa vence). **Regras já testadas em SQL contra os 5.842
lançamentos `*Pago` reais em 26/09 — cobertura de 99,8% do valor:**

| Categoria       | Regras (regex, contém)                                                | Valor   |
|-----------------|-----------------------------------------------------------------------|---------|
| Sócios          | FABIANO AROEIRA · FABRICIO AROEIRA · RAFAEL AROEIRA · EDUARDO AROEIRA  | 21,4 %  |
| Impostos/taxas  | MINISTERIO DA FAZENDA · RECEITA · SECRETARIA DE ESTADO DE ECONOMIA · GDF · PREFEITURA · INSS · FGTS · DARF · ISS · IPTU · ITBI · CARTORIO · JUNTA COMERCIAL · CONSELHO · CREA · SIMPLES NACIONAL | 12,3 % |
| Folha/encargos  | SALARIO · FOLHA · RESCISAO · FERIAS · 13 · VALE · PLANO DE SAUDE · SINDICATO · PENSAO | 0,3 % |
| Condomínios     | CONDOMINIO · ASSOCIACAO DE MORADORES                                   | 3,0 %   |
| Concessionárias | CEB · NEOENERGIA · CAESB · CLARO · VIVO · TIM S · TELECOM · ENERGIA · SANEAMENTO | 0,4 % |
| Seguros/bancos  | SEGURO · ALLIANZ · PORTO  · TOKIO · TARIFA · BANCO · CAIXA ECONOMICA · BRB · ITAU · BRADESCO · SANTANDER · SICOOB | 21,1 % |
| Materiais       | COMERCIO · MATERIAIS · MATERIAL · DISTRIBUIDORA · FERRAGENS · CONCRETO · CIMENTO · MADEIRA · TINTA · ELETRIC · HIDRAUL · LIMPEZA · MAQUINAS · INDUSTRIA · LOJA · ATACAD | 0,9 % |
| Serviços PJ     | SERVICO · ENGENHARIA · CONSTRUTORA · INSTALAC · MANUTENC · CONSULTORIA · ADVOCACIA · ADVOGADO · CONTABIL · TECNOLOGIA · LOCACAO · TRANSPORTE · LTDA · EIRELI · S/A · S.A · EPP · ` ME ` · CIA · ASSOCIA · EMPREENDIMENTOS · PARTICIPACOES · INCORPORA | 30,5 % |
| Mão de obra PF  | regex `^[A-Z]+( [A-Z]+){1,6}$` (só letras e espaços, 2 a 7 palavras)   | 9,8 %   |
| Outros          | resto (294 linhas, R$ 0,05 MM)                                         | 0,2 %   |

Total `*Pago` no banco: R$ 29,1 MM. Os maiores blocos: Serviços PJ 8,9 MM ·
Sócios 6,2 MM · Seguros/bancos 6,2 MM · Impostos 3,6 MM · PF 2,9 MM.

Refinamentos que valem fazer logo depois (não bloqueiam a fase 1):
- "Seguros/bancos" com 21% está gordo demais — provavelmente mistura
  amortização de financiamento paga como `*Pago` com seguro e tarifa. Separar em
  "Financiamento (parcelas)" e "Seguros/tarifas" olhando o `hist`.
- "Serviços PJ" com 30% também é amplo; quando tiver o relatório do UAU com
  conta financeira (Fase 2) ele se abre sozinho.
- "Sócios" é pró-labore/distribuição — para os diretores talvez deva ficar
  como categoria à parte (e possivelmente oculta do consolidado), decisão do
  Eduardo.

- Guardar as regras em `regras_snapshot.payload.natureza` (tabela já existe)
  para editar sem republicar o app; fallback hardcoded no código.
- Reaproveitar `reclass` para exceções manuais: acrescentar coluna `natureza`
  (migração simples). O gerente corrige no drill-down e a regra manual vence a
  automática (mesma lógica de `computeElim`).
- Mostrar sempre a fatia "Outros" e o % classificado no KPI.

### Fase 2 — importar plano de contas do UAU (quando houver relatório)

Novo parser `parseUauContas(wb, fname)` no padrão de `parseUau`, lendo um
relatório de contas a pagar com conta financeira. Casar com `lancamentos` por
(`cod`, `data`, `nominal`, valor) e gravar em coluna nova
`lancamentos.conta` (text, nullable). Quando `conta` existir, ela vence
`natDesp()`. Nada da Fase 1 é jogado fora.

## Desenho da aba "Despesas"

Botão de nav novo entre **Grupo** e **BI** (ícone de "pizza" ou etiqueta).
Layout copia `biView()`: no desktop vira grade 2 colunas (`body[data-tab="despesas"]`).

**Controles no topo**
- Empresa (chips: Todas / cada `cod` de `empresas`) — reaproveitar `gsel`.
- Período (mês / trimestre / ano / tudo + setas) — reaproveitar `gper`/`gnav`.
- Centro de custo (select com os CCs da empresa filtrada; opção "Todos").
- Toggle "incluir prévia" (respeita `origem`), igual ao BI.

**Cartões (ordem)**
1. KPI strip: total despesas do período · variação vs período anterior ·
   nº de lançamentos · % classificado.
2. **Despesas por centro de custo** — barras horizontais, top 10 + "demais"
   (clique → drill-down do CC).
3. **Despesas por natureza** — rosca (doughnut) com legenda e %
   (clique → drill-down da categoria).
4. **Evolução mensal** — barras empilhadas por natureza, últimos 12 meses.
5. **Top 10 fornecedores** — barras horizontais (clique → drill-down).
6. **Tabela CC × natureza** — matriz compacta com totais; é o que o diretor
   vai querer levar para reunião. Botão PDF (`pdfDespesas`) no padrão de
   `pdfGrupo`, com tabela via autotable.

Drill-down: `gdrillShow()` já existente; acrescentar no modal a ação
"reclassificar natureza" visível só para `isManager`.

## Acesso por diretor

- Tabela nova `usuarios_cc` (`email` text PK, `ccs` text[] — códigos de CC
  visíveis, `empresas` text[] opcional, `nome` text). RLS: leitura pelo próprio
  e-mail; escrita só pelo gerente.
- No `onAuth`: se o e-mail não é MANAGER e está em `usuarios_cc`, o app filtra
  `GDATA` para os CCs autorizados e esconde os controles de empresa/CC que não
  se aplicam. Se não está na tabela, comportamento atual (leitor global).
- Cadastro dos diretores pelo menu do gerente (lista simples: e-mail + CCs).
- Criar os usuários no Supabase Auth (painel) — não há auto-cadastro.

## Ordem de execução sugerida (sessão do escritório)

1. `git pull`/download do repositório; ler HANDOFF.md; confirmar que a pasta
   local é a versão mais nova (tem `origem`/`reclass`). Subir esse index.html
   para o GitHub ANTES de começar, para o repositório parar de estar defasado.
2. `natDesp()` com as regras da tabela acima (já validadas). Rodar sobre
   `GDATA` e imprimir no console a cobertura por categoria para conferir que
   bate com os 99,8% do SQL.
3. Aba Despesas com cartões 1–3 (KPIs, CC, natureza) + drill-down. Publicar.
4. Cartões 4–6 + PDF. Publicar.
5. `usuarios_cc` + filtro por e-mail + cadastro no menu. Publicar.
6. Atualizar HANDOFF.md (entrada nova no topo) e push.

Cada passo publica algo usável; se a sessão acabar no meio, nada quebra.

## Cuidados

- Não incluir `Transferência` nem `Cont.Financ.` como despesa — infla o número.
- Intercompany: linhas `*Pago` cujo nominal é empresa do grupo devem ir para
  categoria própria "Intercompany" e ficar fora do consolidado "Todas" (mesma
  regra do DFC).
- `hist` às vezes é continuação do nominal cortado (ex.: nominal "SANTANA
  COMERCIO DE PRODUTOS DE LIMPEZA &" + hist "SERVICOS ENGENHARIA"). Classificar
  sobre `nominal + ' ' + hist`, como `classLanc` já faz.
- Chart.js: destruir instâncias ao trocar de aba (`_biDestroy` como modelo),
  senão vaza memória no celular.
- Avisos do Supabase (não bloqueia, mas anotar): `memoria_import_staging` está
  sem RLS — é do app de reuniões, fechar quando mexer nele.

## Fora de escopo por enquanto

Orçado × realizado por CC (precisa de orçamento por CC no UAU ou planilha);
rateio de administração entre obras; comparação entre empresas normalizada
por porte.
