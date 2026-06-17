# Sistema de cores — Plano de Implementação

> **For agentic workers:** execução inline nesta sessão (app single-file, sem test runner). Verificação por preview MCP + checagem de console. Steps com checkbox.

**Goal:** Adicionar tema claro/escuro (escuro padrão, toggle nas Configurações) e cores por categoria (auto + seletor) ao `index.html` do Grana Pro.

**Architecture:** Tema claro = bloco CSS `html[data-theme="light"]` que sobrescreve as variáveis do `:root`; preferência em localStorage aplicada por script no `<head>` (sem piscar). Cores de categoria reaproveitam o campo `color` já existente (custom em `categorias/.../color`, override das padrão em `categoriasMeta/.../color`); helper `hexA()` tinge as bolinhas de emoji.

**Tech Stack:** HTML/CSS/JS vanilla, Firebase RTDB (já integrado).

**Arquivo único:** `C:\Users\WSzin\Desktop\financas\index.html`

---

## Task 1: Motor de tema (claro/escuro)

**Files:** Modify `index.html` — `<head>` (~linha 6-11), `:root` (~2691), funções window (~perto de getAllCats), `modal-config` (~4159), `abrirConfiguracoes`.

- [ ] **Step 1.1** No `<head>`, logo após `<link rel="apple-touch-icon"...>` (linha 11), inserir script clássico que aplica o tema salvo antes da pintura:
```html
<script>try{if(localStorage.getItem('granapro_tema')==='claro')document.documentElement.setAttribute('data-theme','light');}catch(e){}</script>
```

- [ ] **Step 1.2** Após o `:root{...}` (linha 2691, antes de `*{...}`), adicionar o bloco do tema claro (paleta do spec):
```css
html[data-theme="light"]{
  --bg:#f4ede0; --surface:#fbf6ec; --surface2:#ffffff; --surface3:#efe6d5;
  --border:rgba(168,138,32,0.14); --border2:rgba(168,138,32,0.28);
  --text:#241f15; --text2:#7c6f57; --text3:#a99f88;
  --green:#16a34a; --green-bg:rgba(22,163,74,0.10);
  --red:#dc2626; --red-bg:rgba(220,38,38,0.10);
  --gold:#b8941f; --gold-light:#d4af37; --gold-dark:#8a6f15;
  --gold-bg:rgba(212,175,55,0.16); --gold-border:rgba(168,138,32,0.32);
  --orange:#ea6a0e;
}
```

- [ ] **Step 1.3** Adicionar `window.setTema` + `_syncTemaUI` (perto das outras window fns, ex. depois de `getAllCats`):
```js
window.setTema = function(t){
  if(t==='claro') document.documentElement.setAttribute('data-theme','light');
  else document.documentElement.removeAttribute('data-theme');
  try{ localStorage.setItem('granapro_tema', t==='claro'?'claro':'escuro'); }catch(e){}
  const mc = document.querySelector('meta[name="theme-color"]');
  if(mc) mc.setAttribute('content', t==='claro' ? '#f4ede0' : '#070707');
  _syncTemaUI();
  if(typeof renderTudo==='function') renderTudo();
};
function _syncTemaUI(){
  const claro = document.documentElement.getAttribute('data-theme')==='light';
  const bc=document.getElementById('tema-claro'), be=document.getElementById('tema-escuro');
  if(bc&&be){ bc.classList.toggle('ativo',claro); be.classList.toggle('ativo',!claro); }
}
```

- [ ] **Step 1.4** No `modal-config`, após o `modal-titulo` (linha 4159), inserir a linha de tema:
```html
    <div style="display:flex;gap:8px;margin:14px 0 4px">
      <button class="tema-pill" id="tema-claro" onclick="setTema('claro')">☀️ Claro</button>
      <button class="tema-pill" id="tema-escuro" onclick="setTema('escuro')">🌙 Escuro</button>
    </div>
```

- [ ] **Step 1.5** CSS das pílulas (no `<style>`, junto dos outros):
```css
.tema-pill{flex:1;padding:10px;border-radius:12px;background:var(--surface2);border:1px solid var(--border);color:var(--text2);font-size:14px;font-weight:600;cursor:pointer}
.tema-pill.ativo{border-color:var(--gold-border);color:var(--gold);background:var(--gold-bg)}
```

- [ ] **Step 1.6** Em `abrirConfiguracoes` (onde mostra o modal-config), chamar `_syncTemaUI()` pra marcar a pílula ativa ao abrir.

- [ ] **Step 1.7** Verificar no preview: abrir ⚙️, alternar Claro/Escuro troca tudo na hora; recarregar mantém; nada ilegível; sem erro no console. Commit.

---

## Task 2: Cores por categoria

**Files:** Modify `index.html` — constantes perto de `CAT_INFO` (~740), `getAllCats` (1414), estado `_catColor` (~1474), `abrirModalCategoria` (1483), `salvarCategoria` (1513), `renderGerenciarCats` (1555), lista de movimentações (991-1004), modal-cat HTML (4041), CSS.

- [ ] **Step 2.1** Após `CAT_INFO` (linha 740), adicionar paleta + helpers:
```js
const CORES_CAT = ['#ef4444','#f97316','#d4af37','#22c55e','#14b8a6','#3b82f6','#8b5cf6','#ec4899','#6b7280'];
function hexA(hex, a){
  const h = String(hex||'#6b7280').replace('#',''); if(h.length!==6) return `rgba(107,114,128,${a})`;
  const r=parseInt(h.slice(0,2),16), g=parseInt(h.slice(2,4),16), b=parseInt(h.slice(4,6),16);
  return `rgba(${r},${g},${b},${a})`;
}
function corAutoPorNome(nome){ let s=0; for(const c of String(nome)) s=(s+c.charCodeAt(0))%9999; return CORES_CAT[s % CORES_CAT.length]; }
function proximaCorLivre(){
  const usadas = new Set(Object.values(getAllCats()).map(c=>c.color));
  return CORES_CAT.find(c=>!usadas.has(c)) || CORES_CAT[usadas.size % CORES_CAT.length];
}
```

- [ ] **Step 2.2** `getAllCats` — aplicar override de cor (default) e fallback (custom):
  - Default (linha ~1419): `result[nome] = {...info, emoji:(ov&&ov.emoji)?ov.emoji:info.emoji, color:(ov&&ov.color)?ov.color:info.color};`
  - Custom (linha 1423): `result[c.nome] = {emoji:c.emoji, color:c.color||corAutoPorNome(c.nome), custom:true, _key:c._key};`

- [ ] **Step 2.3** Estado: junto de `_catEmoji` (linha 1474) adicionar `let _catColor = '#6b7280';`

- [ ] **Step 2.4** `abrirModalCategoria`: definir `_catColor` e renderizar swatches.
  - No ramo `nova`: `_catColor = proximaCorLivre();`
  - No ramo editar: `_catColor = cat.color || corAutoPorNome(cat.nome);`
  - No fim (antes de `.add('show')`): `renderCatCores();` e tingir o botão de emoji: `document.getElementById('modal-cat-emoji-btn').style.background = hexA(_catColor,0.22);`

- [ ] **Step 2.5** Adicionar `renderCatCores` + atualizar `trocarEmojiCat` pra manter o fundo:
```js
function renderCatCores(){
  const cont = document.getElementById('modal-cat-cores'); if(!cont) return;
  cont.innerHTML = CORES_CAT.map(c=>`<span class="cat-cor ${c===_catColor?'ativo':''}" style="background:${c}" onclick="selecionarCorCat('${c}')"></span>`).join('');
}
window.selecionarCorCat = function(c){
  _catColor = c; renderCatCores();
  document.getElementById('modal-cat-emoji-btn').style.background = hexA(c,0.22);
};
```

- [ ] **Step 2.6** modal-cat HTML — após o bloco do emoji (linha 4045, antes do label Nome), inserir:
```html
    <div class="cat-cores" id="modal-cat-cores"></div>
```

- [ ] **Step 2.7** `salvarCategoria` — gravar a cor:
  - nova (1520): `await push(ref(db,`grupos/${grupoCode}/categorias`),{nome, emoji:_catEmoji, color:_catColor});`
  - custom (1527): `await update(ref(db,`grupos/${grupoCode}/categorias/${_catEdit._key}`),{nome, emoji:_catEmoji, color:_catColor});`
  - padrão (1531): `await update(ref(db,`grupos/${grupoCode}/categoriasMeta/${_catEdit.nome}`),{emoji:_catEmoji, color:_catColor});`

- [ ] **Step 2.8** `renderGerenciarCats` — tingir a bolinha: trocar `<span class="ger-emoji" ...>` por incluir `style="background:${hexA(info.color,0.20)}"`.

- [ ] **Step 2.9** Lista de movimentações (linha 1004) — tingir o círculo quando há categoria. Antes do `return`, computar `const catBg = catInfo ? `background:${hexA(catInfo.color,0.20)};` : '';` e usar `style="${catBg}${cursor}"` no `tx-icon-circle`. Repetir o mesmo padrão no segundo render (linha 1072) se aplicável.

- [ ] **Step 2.10** Tingir emoji do chip recolhido e da grade no modal de lançamento (renderModalCats): no chip (`cat-chip-emoji`) e em cada `cat-pick-emoji`, aplicar `style="background:${hexA(inf.color||'#6b7280',0.20)}"`. (info da sugerida via `all[_modalCat]`.)

- [ ] **Step 2.11** CSS dos swatches:
```css
.cat-cores{display:flex;gap:8px;justify-content:center;flex-wrap:wrap;margin:0 0 14px}
.cat-cor{width:26px;height:26px;border-radius:50%;cursor:pointer;border:2px solid transparent}
.cat-cor.ativo{border-color:var(--text)}
```

- [ ] **Step 2.12** Verificar no preview: pizza colorida; bolinhas tingidas nas listas/chip/grade/Gerenciar; criar categoria nova já vem com cor; trocar cor no modal e no Gerenciar persiste (recarregar mantém); categoria antiga sem cor não quebra; sem erro no console. Commit.

---

## Self-review
- **Cobertura do spec:** Parte A (tema) = Task 1; Parte B (cores) = Task 2. ✓
- **Tipos/nomes:** `_catColor`, `CORES_CAT`, `hexA`, `proximaCorLivre`, `corAutoPorNome`, `renderCatCores`, `selecionarCorCat`, `setTema`, `_syncTemaUI`, ids `tema-claro/tema-escuro/modal-cat-cores` — consistentes entre tasks.
- **Sem placeholders.**
