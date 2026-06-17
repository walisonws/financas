# Redesign das Categorias — Grana Pro

**Data:** 2026-06-16
**Arquivo afetado:** `index.html` (app single-page)
**Origem:** usuário acha as categorias desagradáveis de usar — emojis ruins e a grade que acumula/bagunça a tela.

## Problema

Hoje, no modal de lançar gasto (`modal-tx`, tipo `out`):

- A **grade inteira** de categorias (8 padrão + todas as criadas) aparece sempre, ocupando muito espaço e poluindo a tela.
- Criar categoria usa **dois `prompt()` nativos** (digitar emoji na mão + nome) — feio e travado.
- Não dá pra **editar o emoji** de uma categoria existente; só apagar (`×`) e recriar.
- Não dá pra **renomear**.

As duas dores priorizadas pelo usuário: **emojis ruins** e **grade bagunçada**.

## Objetivo

1. Lançamento com a tela limpa: mostrar só a **categoria sugerida**; o resto aparece sob demanda.
2. Sugestão **inteligente pela descrição** digitada.
3. Seletor de emoji **visual** (sem digitar) ao criar/editar.
4. Poder **editar o emoji** de qualquer categoria e **renomear** as criadas.
5. Repaginar os emojis das 8 padrão.

## Caminho escolhido

**Caminho A — seletor recolhido no lançamento + tela separada "Gerenciar categorias".**
Separa "escolher no dia a dia" (rápido, recolhido) de "organizar" (eventual, tela própria). Mantém o modal de lançamento enxuto.

## Design

### 1. Seletor recolhido (modal de lançamento)

Estado padrão ao abrir o modal de Gastei:

- Mostra **um único chip**: a categoria sugerida, já selecionada (emoji + nome), + um botão **"Trocar ▾"**.
- Tocar em "Trocar" (ou no chip) **expande** a grade atual de categorias.
- Ao escolher uma categoria na grade, ela **recolhe** de volta pro chip único.
- A grade expandida ganha, no fim, dois atalhos:
  - **＋ Nova** — cria categoria (agora com seletor visual de emoji — ver §3).
  - **⚙️ Gerenciar** — abre a tela de gerenciar categorias (§4).

Estado de expansão é **efêmero**: cada vez que o modal abre, começa recolhido.

### 2. Sugestão por palavra-chave

- Enquanto o usuário digita no campo Descrição (`modal-desc`), o app recalcula a sugestão **ao vivo** (evento `input`).
- Lógica:
  1. Normaliza a descrição (minúsculas, sem acento) e procura palavras-chave num mapa PT-BR → categoria.
  2. Se casar, sugere essa categoria.
  3. Se **não** casar, sugere a **categoria mais usada** historicamente (contagem em `_dados.gastos`).
  4. Se não houver histórico (app novo), sugere **"Outros"**.
- A sugestão só **pré-seleciona** — o usuário pode sempre trocar. Se ele já trocou manualmente nesse lançamento, parar de sobrescrever (não brigar com a escolha dele).
- Mapa inicial de palavras-chave (semente, ampliável):
  - **Transporte:** uber, 99, taxi, gasolina, combustivel, alcool, onibus, metro, passagem, estacionamento, pedagio, moto, corrida
  - **Alimentação:** mercado, supermercado, ifood, almoco, janta, jantar, lanche, padaria, restaurante, comida, cafe, pizza, feira, acougue
  - **Saúde:** farmacia, remedio, medico, consulta, dentista, exame, plano, academia
  - **Lazer:** cinema, bar, balada, jogo, netflix, spotify, show, viagem, passeio
  - **Compras:** roupa, sapato, shopping, presente, amazon, shopee, mercadolivre, loja
  - **Moradia:** aluguel, luz, energia, agua, internet, gas, condominio, faxina, movel
  - **Educação:** curso, livro, faculdade, escola, mensalidade, material, apostila

> Implementação: helper `sugerirCategoria(desc)` puro (testável) + um listener `input` no `modal-desc`.

### 3. Seletor visual de emoji

Substitui os `prompt()`. Um pequeno overlay/grade com ~40–60 emojis curados (offline, sem lib externa), organizados de forma simples:

- Grupos curtos: dinheiro/trabalho, comida, transporte, casa, saúde, lazer, compras, diversos.
- Tocar num emoji = escolhe. Sem digitação.

Usado em dois fluxos: **criar nova** (§1) e **trocar emoji** de existente (§4).

### 4. Tela "Gerenciar categorias"

Novo overlay/modal listando **todas** as categorias (padrão + criadas). Cada linha:

- Emoji (toca → abre o seletor visual de emoji §3 → grava a troca).
- Nome.
- Ação **renomear** — só habilitada nas **criadas**.
- Ação **apagar** — só nas **criadas** (gastos voltam pra "Outros", igual hoje).

Regras:

- **Emoji:** editável em **todas** (padrão e criadas).
- **Renomear:** só nas **criadas**. Ao renomear:
  - Atualiza o nome no registro Firebase da categoria.
  - **Migra** `cat` dos gastos e contas fixas **vigentes** (`_dados.gastos`, `_dados.fixos`) do nome antigo pro novo.
  - Histórico já arquivado fica **congelado** com o nome antigo (é snapshot — comportamento aceito).
  - Se a categoria renomeada estava selecionada num modal aberto, atualizar a seleção.
- **Padrão:** nome **fixo** (ligado às palavras-chave §2 e a gastos antigos). Só emoji muda.

### 5. Emojis padrão repaginados

Conjunto entregue por padrão (ajustável depois via §4):

| Categoria | Emoji |
|-----------|-------|
| Alimentação | 🍽️ |
| Transporte | 🚗 |
| Lazer | 🎉 |
| Saúde | 🩺 |
| Compras | 🛍️ |
| Moradia | 🏡 |
| Educação | 📚 |
| Outros | 📦 |

> São um ponto de partida; o usuário troca qualquer um na tela Gerenciar.

## Modelo de dados (Firebase)

Hoje:

- `grupos/{code}/categorias/{key}` = `{ nome, emoji, color }` — as **criadas**.
- Padrão são hardcoded em `CAT_INFO` no JS.

Mudança — permitir editar o emoji das **padrão** sem quebrar gastos antigos:

- Novo ramo `grupos/{code}/categoriasMeta/{nomePadrao}` = `{ emoji }` — guarda **apenas overrides de emoji** das categorias padrão.
- No carregamento, `getAllCats()` mescla: `CAT_INFO` (base) ← override de emoji de `categoriasMeta` ← categorias criadas de `categorias`.
- Gastos continuam referenciando categoria **pelo nome** (sem migração de schema). Nada de gasto antigo quebra.

> Nenhuma mudança destrutiva. Categorias criadas continuam no mesmo lugar; só entra o ramo novo `categoriasMeta` para overrides de emoji das padrão.

## Funções/pontos no código (referência)

- `CAT_INFO` (~linha 704) — emojis padrão a repaginar.
- `getAllCats()` (~1321) — mesclar `categoriasMeta`.
- `renderModalCats()` / `renderModalCatsFixo()` (~1329 / ~1458) — passar a renderizar recolhido + expandir.
- `criarCategoria()` (~1356) — trocar `prompt()` pelo seletor visual.
- `removerCategoria()` (~1371) — mantém.
- `abrirModalLanc` / fluxo `modal-tx` (~1243, HTML ~3696) — listener `input` no `modal-desc` + estado recolhido/expandido.
- Onde `onValue(... /categorias)` carrega (~553) — carregar também `categoriasMeta`.
- Novos: `sugerirCategoria(desc)`, seletor de emoji, modal "Gerenciar categorias", `renomearCategoria()`, `setEmojiCategoria()`.

## Fora de escopo (YAGNI)

- Cor por categoria (toda criada segue dourada por ora).
- Renomear categorias padrão.
- Aprendizado automático do mapa de palavras-chave (mapa é estático, ampliável na mão).
- Reordenar / arquivar / ocultar categorias.

## Verificação

- Abrir Gastei: aparece só o chip sugerido + "Trocar".
- Digitar "uber" → sugere Transporte; "ifood" → Alimentação; texto desconhecido → mais usada; app novo → Outros.
- "Trocar" expande a grade; escolher recolhe.
- ＋ Nova com seletor visual de emoji (sem prompt).
- Gerenciar: trocar emoji de uma padrão (persiste); renomear uma criada migra os gastos vigentes; apagar criada volta gastos pra Outros.
- Sem erros no console (preview).
