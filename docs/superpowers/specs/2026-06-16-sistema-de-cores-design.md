# Sistema de cores — Grana Pro (tema claro/escuro + cores por categoria)

**Data:** 2026-06-16
**Arquivo afetado:** `index.html` (app single-page)
**Sub-projeto 1 de 3** (os outros: Cofrinho/Meta; Polimento+PWA — specs próprios depois).

## Problema

1. O app só tem o tema **preto/dourado**. Algumas pessoas acham pesado de dia e querem um tema claro.
2. Toda categoria é **dourada** (`#d4af37` fixo). A pizza do Resumo fica tudo da mesma cor (ilegível) e as listas não têm distinção visual entre categorias.

## Objetivo

- Adicionar um **tema claro (bege/creme)** opcional, com **escuro como padrão** (preserva a identidade atual). Troca nas Configurações, sem piscar, persistente por aparelho.
- Dar uma **cor distinta a cada categoria**, com cor automática boa + possibilidade de trocar. A cor aparece na pizza e nas bolinhas de emoji.

## Caminho escolhido

As variáveis CSS já são centralizadas num `:root`. O tema claro é um bloco `[data-theme="light"]` que **sobrescreve** as variáveis — o escuro continua sendo o `:root` atual (risco mínimo de regressão). As cores de categoria reaproveitam o campo `color` que já existe nos dados.

---

## Parte A — Tema claro/escuro

### Comportamento
- **Padrão: escuro** (atual). Quem quiser troca pra claro nas Configurações (⚙️).
- Preferência guardada em **`localStorage` (`granapro_tema` = `'claro'` | `'escuro'`)** — por aparelho. No modo Casal, cada celular escolhe o seu.
- Aplicado **antes da primeira pintura** pra não piscar: um `<script>` clássico curto no `<head>` lê o localStorage e, se `'claro'`, seta `document.documentElement.setAttribute('data-theme','light')`.

### Técnica
- `:root { ... }` permanece **igual** (tema escuro = ausência de `data-theme`).
- Novo bloco `html[data-theme="light"] { ... }` sobrescreve as mesmas variáveis com a paleta clara.
- A meta `theme-color` (barra de status do celular) acompanha o tema (`#070707` escuro / `#f4ede0` claro), atualizada no toggle.

### Paleta clara (bege/creme)
| Variável | Escuro (atual) | Claro (novo) |
|---|---|---|
| `--bg` | `#070707` | `#f4ede0` |
| `--surface` | `#0f0f0f` | `#fbf6ec` |
| `--surface2` | `#181818` | `#ffffff` |
| `--surface3` | `#222` | `#efe6d5` |
| `--border` | `rgba(212,175,55,0.09)` | `rgba(168,138,32,0.14)` |
| `--border2` | `rgba(212,175,55,0.2)` | `rgba(168,138,32,0.28)` |
| `--text` | `#f5f0e8` | `#241f15` |
| `--text2` | `#8a7d65` | `#7c6f57` |
| `--text3` | `#4a4030` | `#a99f88` |
| `--green` | `#22c55e` | `#16a34a` |
| `--green-bg` | `rgba(34,197,94,0.08)` | `rgba(22,163,74,0.10)` |
| `--red` | `#ef4444` | `#dc2626` |
| `--red-bg` | `rgba(239,68,68,0.08)` | `rgba(220,38,38,0.10)` |
| `--gold` | `#d4af37` | `#b8941f` |
| `--gold-light` | `#f0c84a` | `#d4af37` |
| `--gold-dark` | `#a88a20` | `#8a6f15` |
| `--gold-bg` | `rgba(212,175,55,0.09)` | `rgba(212,175,55,0.16)` |
| `--gold-border` | `rgba(212,175,55,0.28)` | `rgba(168,138,32,0.32)` |
| `--orange` | `#f97316` | `#ea6a0e` |

> `--radius` / `--radius-sm` não mudam.

### UI do toggle
- No painel de Configurações (⚙️), uma linha **"Tema"** com dois botões/pílulas: **Claro** e **Escuro** (o ativo destacado).
- `window.setTema(t)`: seta/remove o `data-theme`, grava no localStorage, atualiza a meta `theme-color` e o estado visual das pílulas.

### Bordas / casos
- A tela de boas-vindas usa título dourado em gradiente (hex fixo) sobre `var(--bg)`. No claro, conferir contraste; se ficar fraco, dar a essa tela um fundo escuro fixo (ela só aparece no 1º acesso). Decidir na verificação do preview.
- Nenhum dado muda. Trocar tema é instantâneo e reversível.

---

## Parte B — Cores por categoria

### Paleta curada (9 cores, legíveis no claro e no escuro)
`#ef4444` vermelho · `#f97316` laranja · `#d4af37` dourado · `#22c55e` verde · `#14b8a6` teal · `#3b82f6` azul · `#8b5cf6` roxo · `#ec4899` rosa · `#6b7280` cinza.

Constante nova: `const CORES_CAT = ['#ef4444','#f97316','#d4af37','#22c55e','#14b8a6','#3b82f6','#8b5cf6','#ec4899','#6b7280'];`

### Cores padrão por categoria (em `CAT_INFO`)
| Categoria | Cor |
|---|---|
| 🍽️ Alimentação | `#ef4444` |
| 🚗 Transporte | `#3b82f6` |
| 🎉 Lazer | `#8b5cf6` |
| 🩺 Saúde | `#22c55e` |
| 🛍️ Compras | `#ec4899` |
| 🏡 Moradia | `#14b8a6` |
| 📚 Educação | `#f97316` |
| 📦 Outros | `#6b7280` |

### Atribuição
- **Categoria nova:** recebe automaticamente a **primeira cor da `CORES_CAT` ainda não usada** por nenhuma categoria atual (se todas usadas, cicla pela paleta). 
- **Trocar cor:** um seletor de cor (bolinhas da paleta) aparece **no modal criar/editar categoria** (ao lado do emoji) **e** em cada linha da tela **Gerenciar**. Estado novo `_catColor`.

### Armazenamento (sem migração destrutiva)
- **Custom:** `grupos/{code}/categorias/{key}/color` — o campo `color` já existe (hoje fixo em `#d4af37`); passa a receber a cor escolhida/automática.
- **Padrão (override):** `grupos/{code}/categoriasMeta/{nome}/color` — guardado junto do override de emoji que já existe.
- **`getAllCats()`** passa a mesclar a cor: base = cor de `CAT_INFO`; override de `categoriasMeta[nome].color`; custom = `c.color || cor automática`.
- Categorias antigas sem `color` definido → caem na cor padrão por nome (se for padrão) ou recebem uma cor automática da paleta (se custom sem cor). Nada quebra.

### Onde a cor aparece
1. **Pizza do Resumo** — já usa `item.color` (`renderPizzaChart`); agora fica colorida de verdade.
2. **Bolinha do emoji tingida** — fundo = cor com ~16–18% de opacidade; usada em:
   - linhas de movimentação (gastos com categoria),
   - chip da categoria selecionada no modal de lançamento,
   - linhas da tela Gerenciar e a grade de seleção.
   - Helper novo `hexA(hex, a)` converte `#rrggbb` → `rgba(...)` pro fundo translúcido (funciona nos dois temas).

---

## Fora de escopo (YAGNI)
- Tema "automático seguindo o sistema" (decidiu-se padrão escuro fixo + toggle manual).
- Sincronizar o tema entre aparelhos (é preferência local de propósito).
- Cores totalmente livres (color picker RGB) — só a paleta curada.
- Renomear/recolorir categorias padrão além de emoji+cor.

## Verificação (preview)
- **Tema:** alternar Claro/Escuro nas Configurações troca tudo na hora; reabrir o app mantém a escolha; nada fica ilegível (texto, bordas, pílulas, hero, pizza, modais, nav) em nenhum dos temas; barra de status do celular acompanha; tela de boas-vindas legível no claro.
- **Cores:** pizza do Resumo aparece multicolorida; bolinhas de emoji tingidas nas listas/chip/Gerenciar; criar categoria nova já vem com cor automática; trocar a cor no modal e no Gerenciar persiste (recarregar mantém); categoria antiga sem cor não quebra.
- Sem erros no console.
