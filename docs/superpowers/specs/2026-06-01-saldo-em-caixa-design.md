# Design — Saldo em Caixa (valor inicial que carrega entre meses)

**Data:** 2026-06-01
**App:** Grana Pro (`index.html`, single-file PWA + Firebase Realtime Database)

## Problema

Hoje o saldo de cada pessoa é calculado só com o fluxo do mês:

```
Saldo = Renda do mês − Gastos diários − Contas fixas pagas
```

Cada mês começa do zero. O dinheiro que sobra de um mês não é levado pro próximo,
então o número mostrado no app **não bate** com o que o usuário realmente tem na
conta/no bolso. Ex.: tinha R$ 2.000 guardados, ganhou R$ 3.000, gastou R$ 4.000 →
o app mostra −R$ 1.000, mas na vida real ele ainda tem R$ 1.000.

## Decisões (confirmadas com o usuário)

1. **Caixa por pessoa** — cada membro tem o seu valor em caixa, não é um pote único
   do casal. Combina com o modelo atual (cada membro já tem saldo/renda/gastos próprios).
2. **Automático + editável** — ao fechar o mês, o que sobrou vira o caixa inicial do
   próximo mês automaticamente; o usuário também pode editar na mão a qualquer momento.
3. **Sempre visível** — a linha "Em caixa" aparece sempre, mesmo quando for R$ 0
   (lembra o usuário de preencher).

## Nova fórmula do saldo

```
Saldo = Em caixa (inicial do mês) + Renda − Gastos diários − Contas fixas pagas
```

Aplicada em todos os pontos que hoje calculam saldo:

- Saldo hero da tela Início (`render()` / atualização do `#meu-saldo`, ~linha 705)
- Resumo Individual modo mês (`renderResumoSolo`, ~linha 1507)
- Resumo Grupo por membro (`renderResumoGrupo`, ~linha 1561)
- Compartilhar resumo (`compartilharResumo`, ~linha 2006)
- Snapshot do histórico no fechar mês (`fecharMes`, ~linha 1941)

> Observação: o caixa **não** entra nos modos "Hoje"/"Semana" do Resumo — só no modo
> "Mês", porque caixa é um conceito mensal.

## Modelo de dados (Firebase)

Novo nó, separado dos lançamentos (não é afetado por fechar mês ou apagar gastos):

```
grupos/{code}/caixa/{mesSlug}/{pid} = <número>
```

- `mesSlug`: mesmo formato já usado no histórico (`getMesAtivo().toLowerCase().replace(/\s+/g,'-')`),
  ex.: `"junho-2026"`.
- `pid`: id do membro (no modo Individual, `meuPerfil`).
- Valor ausente = R$ 0.

Helpers novos:

- `getCaixa(pid, mesSlug?)` — lê o caixa do membro no mês (default: mês ativo). Retorna 0 se ausente.
- `setCaixa(pid, valor, mesSlug?)` — grava via `set(...)`.

Os dados de caixa entram no `onValue` que já carrega o grupo, populando `_dados.caixa`.

## UI

Na tela **Início**, logo abaixo do saldo hero, uma linha sempre visível:

```
💰 Em caixa: R$ 1.200,00  ✏️
```

- Toque abre um campo (prompt/modal simples no padrão dos outros modais do app) pra
  digitar/corrigir o valor **do mês e da pessoa atualmente em foco** (`membroAtivo || meuPerfil`).
- No modo grupo com "Todos" selecionado: a linha mostra a soma dos caixas de todos e
  **não é editável** (edição é sempre individual, igual aos lançamentos). Mensagem curta
  orientando a selecionar a pessoa pra editar.
- Salvou → re-renderiza, saldo hero atualiza com a animação de contador já existente.

## Fechar mês (carry-over automático)

`fecharMes` hoje fecha o grupo inteiro de uma vez. Ajuste:

1. Para cada membro, calcular o saldo final do mês **já incluindo o caixa daquele mês**:
   `saldoFinal = caixaDoMes(pid) + renda(pid) − gastos(pid) − fixosPagos(pid)`.
2. Gravar `saldoFinal` como o caixa do **mês seguinte** daquele membro:
   `setCaixa(pid, saldoFinal, mesSeguinteSlug)`.
3. Manter o comportamento atual: remover ganhos/gastos do mês, manter contas fixas,
   salvar snapshot no histórico (agora o `saldo` do snapshot inclui o caixa).
4. Não sobrescrever o caixa do mês seguinte se ele já tiver sido editado na mão? →
   **Decisão:** sobrescreve (o saldo final é a verdade no fechamento). Edição manual
   posterior continua possível.

Helper: `mesSeguinteSlug(mesLabel)` — calcula o mês+1 a partir do label do mês fechado
(reusa `mesParaIndice` / `MESES`).

## Compatibilidade

- Dados antigos sem `caixa`: tratados como R$ 0 → comportamento idêntico ao de hoje até
  o usuário preencher. Sem migração necessária.
- Modo Individual (`solo`): funciona igual, `pid = meuPerfil`.

## Fora de escopo (YAGNI)

- Histórico/gráfico da evolução do caixa ao longo dos meses.
- Múltiplas "contas"/carteiras por pessoa (poupança vs corrente).
- Transferência de caixa entre membros.
