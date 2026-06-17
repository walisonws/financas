# Categorias + Lista de hoje + Confirmar exclusão — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Deixar as categorias agradáveis de usar (seletor recolhido + sugestão por descrição + emoji visual + tela gerenciar), encurtar "Últimas movimentações" pra só as de hoje, e exigir confirmação bonita antes de qualquer exclusão.

**Architecture:** App single-page em um único `index.html` (HTML + CSS + JS modular Firebase, sem build, sem framework). Tudo é vanilla JS com funções globais em `window.*` e estado em variáveis de módulo (`_dados`, `_customCats`, etc.). Firebase Realtime Database via `onValue`/`ref`/`push`/`update`/`remove`/`set`. Não há test runner — a verificação é feita pelo **preview no navegador** (console sem erros + checagem visual), padrão já usado no projeto.

**Tech Stack:** HTML/CSS/JS vanilla, Firebase Realtime Database (modular SDK via CDN), GitHub Pages.

**Spec:** `docs/superpowers/specs/2026-06-16-categorias-redesign-design.md`

**Convenções do repo a respeitar:**
- Funções públicas viram `window.nome = function(){}`; helpers internos são `function nome(){}`.
- Texto livre do usuário em `innerHTML` SEMPRE passa por `esc()` (anti-XSS).
- Comentários e strings em PT-BR.
- Fechar overlays usa o helper existente `_fecharOverlay(id, cb)`.
- Toda exclusão mantém o padrão "otimista + Desfazer 4s" via `_pendingDeletes` + `showToastUndo`.

---

## File Structure

Único arquivo modificado: **`index.html`**. Nenhum arquivo novo (app é single-file por design). As mudanças são organizadas por região:

- **JS — helpers de categoria** (perto de `CAT_INFO` ~linha 704 e `getAllCats` ~1321): mapa de palavras-chave, `sugerirCategoria`, merge de `categoriasMeta` em `getAllCats`, emojis padrão repaginados.
- **JS — listener Firebase** (~linha 553): carregar `categoriasMeta`.
- **JS — componentes de UI novos** (perto dos outros modais, ~linha 1380): `escolherEmoji`, modal "categoria" (`abrirModalCategoria`/salvar), modal "gerenciar" (`abrirGerenciarCats`), `confirmarAcao`.
- **JS — pickers do lançamento** (`renderModalCats` ~1329): seletor recolhido + expandir + sugestão ao digitar.
- **JS — lista Início** (`renderInicio`, trecho `tx` ~900-942): só de hoje.
- **JS — exclusões** (`delGanho`/`delGasto`/`delFixo`/`delHistorico` ~2041-2118): embrulhar com confirmação.
- **HTML — novos overlays** (perto de `modal-tx` ~3745): `modal-emoji`, `modal-cat`, `modal-gerenciar`, `modal-confirm`.
- **CSS** (perto de `.modal-cats` ~2976): grade de emoji, lista do gerenciar, chip recolhido.

---

## Task 1: Helper de sugestão de categoria por descrição

**Files:**
- Modify: `index.html` (logo após o bloco `CAT_INFO`, ~linha 711)

- [ ] **Step 1: Adicionar o mapa de palavras-chave e o helper `sugerirCategoria`**

Logo depois do objeto `CAT_INFO` (que termina em `'Outros' :{emoji:'💼', color:'#8a7d65'}` `}`), inserir:

```javascript
// Mapa palavra-chave → categoria (PT-BR, comparado sem acento/minúsculo). Ampliável na mão.
const CAT_KEYWORDS = {
  'Transporte': ['uber','99','taxi','gasolina','combustivel','etanol','alcool','diesel','onibus','metro','passagem','estacionamento','pedagio','moto','corrida'],
  'Alimentação':['mercado','supermercado','ifood','rappi','almoco','janta','jantar','lanche','padaria','restaurante','comida','cafe','pizza','feira','acougue','hortifruti'],
  'Saúde':      ['farmacia','drogaria','remedio','medico','consulta','dentista','exame','plano','academia','psicologo','hospital'],
  'Lazer':      ['cinema','bar','balada','netflix','spotify','show','viagem','passeio','game','hbo','disney','jogo'],
  'Compras':    ['roupa','sapato','tenis','calca','shopping','presente','amazon','shopee','mercadolivre','loja'],
  'Moradia':    ['aluguel','luz','energia','agua','internet','wifi','gas','condominio','faxina','movel','iptu'],
  'Educação':   ['curso','livro','faculdade','escola','mensalidade','material','apostila','aula']
};

// Normaliza texto pra comparar (minúsculo, sem acento).
function normalizarTxt(s){
  return String(s||'').toLowerCase().normalize('NFD').replace(/[̀-ͯ]/g,'');
}

// Sugere categoria pela descrição digitada. Sem match → categoria mais usada → 'Outros'.
function sugerirCategoria(desc){
  const txt = normalizarTxt(desc);
  if(txt){
    for(const [cat, palavras] of Object.entries(CAT_KEYWORDS)){
      if(palavras.some(p => txt.includes(p))) return cat;
    }
  }
  const freq = {};
  (_dados.gastos||[]).forEach(g=>{ if(g.cat) freq[g.cat]=(freq[g.cat]||0)+1; });
  const top = Object.entries(freq).sort((a,b)=>b[1]-a[1])[0];
  return top ? top[0] : 'Outros';
}
```

- [ ] **Step 2: Verificar no preview (console) que a sugestão funciona**

Inicie o preview (`preview_start`) e rode via `preview_eval`:

```javascript
[sugerirCategoria('uber pro centro'), sugerirCategoria('ifood janta'), sugerirCategoria('xyz aleatorio'), sugerirCategoria('')]
```

Esperado: `['Transporte','Alimentação', <mais usada ou 'Outros'>, <mais usada ou 'Outros'>]`. Sem erro no console (`preview_console_logs`).

- [ ] **Step 3: Commit**

```bash
git add index.html
git commit -m "feat(cat): helper sugerirCategoria por palavra-chave da descrição"
```

---

## Task 2: Carregar overrides de emoji (categoriasMeta) + repaginar emojis padrão

**Files:**
- Modify: `index.html` — declaração de `_customCats`, listener Firebase (~553), `getAllCats` (~1321), `CAT_INFO` (~704)

- [ ] **Step 1: Declarar o estado `_catMeta`**

Encontre a declaração de `_customCats` (busque `let _customCats`) e adicione na linha seguinte:

```javascript
let _catMeta = {}; // overrides de emoji das categorias PADRÃO (grupos/{code}/categoriasMeta)
```

- [ ] **Step 2: Carregar `categoriasMeta` do Firebase**

No `setupListeners` (ou onde ficam os `onValue`), logo após o bloco `onValue(ref(db,`grupos/${grupoCode}/categorias`), ...)` (que termina na linha com `});` antes de `}` ~linha 560), inserir:

```javascript
  onValue(ref(db,`grupos/${grupoCode}/categoriasMeta`), snap=>{
    _catMeta = snap.val() || {};
    // Defensivo: alguns elementos/funções só existem depois de outras tasks.
    const mtx = document.getElementById('modal-tx');
    const mfx = document.getElementById('modal-fixo');
    const mgr = document.getElementById('modal-gerenciar');
    if(mtx && mtx.classList.contains('show')) renderModalCats();
    if(mfx && mfx.classList.contains('show')) renderModalCatsFixo();
    if(mgr && mgr.classList.contains('show') && typeof renderGerenciarCats==='function') renderGerenciarCats();
    renderTudo();
  });
```

- [ ] **Step 3: Mesclar override de emoji em `getAllCats`**

Substitua a função `getAllCats` inteira por:

```javascript
function getAllCats(){
  const result = {};
  // Padrão, aplicando override de emoji (categoriasMeta) se houver.
  Object.entries(CAT_INFO).forEach(([nome,info])=>{
    const ov = _catMeta[nome];
    result[nome] = {...info, emoji: (ov && ov.emoji) ? ov.emoji : info.emoji};
  });
  // Categorias criadas pelo usuário.
  _customCats.forEach(c=>{
    result[c.nome] = {emoji:c.emoji, color:c.color||'#d4af37', custom:true, _key:c._key};
  });
  return result;
}
```

- [ ] **Step 4: Repaginar os emojis das 8 categorias padrão**

Substitua o objeto `CAT_INFO` por:

```javascript
const CAT_INFO = {
  'Alimentação':{emoji:'🍽️', color:'#ef4444'},
  'Transporte' :{emoji:'🚗', color:'#f97316'},
  'Lazer'      :{emoji:'🎉', color:'#d4af37'},
  'Saúde'      :{emoji:'🩺', color:'#10b981'},
  'Compras'    :{emoji:'🛍️', color:'#ec4899'},
  'Moradia'    :{emoji:'🏡', color:'#3b82f6'},
  'Educação'   :{emoji:'📚', color:'#8b5cf6'},
  'Outros'     :{emoji:'📦', color:'#8a7d65'}
};
```

- [ ] **Step 5: Verificar no preview**

Recarregue (`preview_eval: window.location.reload()`), confirme console sem erros e tire `preview_screenshot` da aba Início mostrando os ícones das movimentações com os emojis novos. Rode `preview_eval: getAllCats()['Alimentação'].emoji` → esperado `'🍽️'`.

- [ ] **Step 6: Commit**

```bash
git add index.html
git commit -m "feat(cat): override de emoji das padrão (categoriasMeta) + emojis repaginados"
```

---

## Task 3: Seletor visual de emoji (componente reutilizável)

**Files:**
- Modify: `index.html` — novo overlay HTML (perto de `modal-tx`, ~3745), JS do componente (perto dos modais, ~1380), CSS (~2976)

- [ ] **Step 1: Adicionar o overlay HTML do seletor de emoji**

Logo após o fechamento do `<div class="modal-overlay" id="modal-tx">...</div>` (a linha `</div>` que fecha esse overlay, ~3745), inserir:

```html
<div class="modal-overlay" id="modal-emoji" onclick="if(event.target===this)fecharEmoji()">
  <div class="modal-card" style="max-width:360px">
    <div class="modal-handle"></div>
    <div class="modal-titulo">Escolha um emoji</div>
    <div class="emoji-grid" id="emoji-grid"></div>
    <div class="modal-actions">
      <button class="modal-btn modal-btn-cancel" onclick="fecharEmoji()">Cancelar</button>
    </div>
  </div>
</div>
```

- [ ] **Step 2: Adicionar o CSS da grade de emoji**

Perto da regra `.modal-cats{...}` (~2976), inserir:

```css
.emoji-grid{display:grid;grid-template-columns:repeat(6,1fr);gap:6px;max-height:240px;overflow-y:auto;margin-bottom:14px;}
.emoji-cell{font-size:22px;line-height:1;padding:8px 0;text-align:center;background:var(--surface2);border:1px solid var(--border);border-radius:10px;cursor:pointer;}
.emoji-cell:active{transform:scale(0.92);}
```

- [ ] **Step 3: Adicionar o JS do componente `escolherEmoji`**

Perto das funções de categoria (logo após `selecionarCatFixo`, ~1483), inserir:

```javascript
// ── SELETOR VISUAL DE EMOJI ─────────────────────────────────────────────────
const EMOJI_OPCOES = [
  '🍽️','🍔','🍕','☕','🍺','🛒','🥗','🍎',
  '🚗','⛽','🚌','✈️','🏍️','🚕','🅿️','🛣️',
  '🏠','🏡','💡','💧','🌐','🔥','🛋️','🧹',
  '🩺','💊','🏥','🦷','🏋️','🧠','💉','🩹',
  '🎉','🎮','🎬','🎵','📺','⚽','🏖️','🎸',
  '🛍️','👕','👟','🎁','💻','📱','💳','🛒',
  '📚','✏️','🎓','🏫','📝','🔬','💼','🧾',
  '💰','💵','📦','⭐','❤️','🐶','🐱','✂️'
];
let _emojiCb = null;
window.escolherEmoji = function(onPick){
  _emojiCb = onPick;
  document.getElementById('emoji-grid').innerHTML = EMOJI_OPCOES.map(e=>
    `<div class="emoji-cell" onclick="_pickEmoji('${e}')">${e}</div>`
  ).join('');
  document.getElementById('modal-emoji').classList.add('show');
};
window._pickEmoji = function(e){
  const cb = _emojiCb; _emojiCb = null;
  _fecharOverlay('modal-emoji', ()=>{ if(cb) cb(e); });
};
window.fecharEmoji = function(){ _emojiCb = null; _fecharOverlay('modal-emoji'); };
```

- [ ] **Step 4: Verificar no preview**

Recarregue. Rode `preview_eval: escolherEmoji(e=>console.log('escolhido', e))` → deve abrir o overlay com a grade. `preview_screenshot` da grade. `preview_click` numa célula e confirme em `preview_console_logs` o log `escolhido <emoji>` e que o overlay fechou.

- [ ] **Step 5: Commit**

```bash
git add index.html
git commit -m "feat(cat): seletor visual de emoji reutilizável"
```

---

## Task 4: Modal "categoria" (criar/editar) usando o seletor de emoji

Substitui os `prompt()` de criação por um modal estilizado com input de nome + botão de emoji. Serve pra **criar nova** e, vindo do Gerenciar, **editar** (nome+emoji de criadas; só emoji das padrão).

**Files:**
- Modify: `index.html` — novo overlay HTML (após `modal-emoji`), JS (substitui `criarCategoria`), reapontar `renderModalCats`/`renderModalCatsFixo`

- [ ] **Step 1: Adicionar o overlay HTML `modal-cat`**

Após o `</div>` que fecha `modal-emoji`, inserir:

```html
<div class="modal-overlay" id="modal-cat" onclick="if(event.target===this)fecharModalCat()">
  <div class="modal-card" style="max-width:360px">
    <div class="modal-handle"></div>
    <div class="modal-titulo" id="modal-cat-titulo">Nova categoria</div>
    <div style="display:flex;flex-direction:column;align-items:center;gap:10px;margin:6px 0 14px">
      <button type="button" id="modal-cat-emoji-btn" onclick="trocarEmojiCat()"
        style="font-size:38px;line-height:1;width:78px;height:78px;border-radius:18px;background:var(--surface2);border:1px solid var(--border);cursor:pointer">＋</button>
      <div style="font-size:11px;color:var(--text3)">toque pra escolher o emoji</div>
    </div>
    <label class="modal-label" id="modal-cat-nome-label">Nome</label>
    <input type="text" class="modal-input" id="modal-cat-nome" placeholder="Ex: Pet, Investimentos..."/>
    <div class="modal-actions">
      <button class="modal-btn modal-btn-cancel" onclick="fecharModalCat()">Cancelar</button>
      <button class="modal-btn modal-btn-confirm in" id="modal-cat-salvar" onclick="salvarCategoria()">Salvar</button>
    </div>
  </div>
</div>
```

- [ ] **Step 2: Substituir `criarCategoria` pelo novo fluxo de modal**

Substitua a função `window.criarCategoria = async function(origem){ ... }` inteira por:

```javascript
// Estado do modal categoria. _catEdit = null (nova) | {nome, emoji, _key|null, custom}
let _catEmoji = '📦';
let _catEdit  = null;
let _catOrigem = 'tx'; // 'tx' | 'fixo' | 'gerenciar'

window.criarCategoria = function(origem){
  if(!isPro()){ abrirModalPro('feature'); return; }
  abrirModalCategoria('nova', origem || 'tx');
};

window.abrirModalCategoria = function(modo, origem, cat){
  _catOrigem = origem || 'tx';
  if(modo === 'nova'){
    _catEdit = null;
    _catEmoji = '📦';
    document.getElementById('modal-cat-titulo').textContent = 'Nova categoria';
    document.getElementById('modal-cat-nome').value = '';
    document.getElementById('modal-cat-nome').disabled = false;
    document.getElementById('modal-cat-nome-label').style.opacity = '1';
  } else {
    _catEdit = cat; // {nome, emoji, custom, _key}
    _catEmoji = cat.emoji || '📦';
    document.getElementById('modal-cat-titulo').textContent = cat.custom ? 'Editar categoria' : 'Editar emoji';
    document.getElementById('modal-cat-nome').value = cat.nome;
    // Padrão: nome fixo (não renomeável).
    document.getElementById('modal-cat-nome').disabled = !cat.custom;
    document.getElementById('modal-cat-nome-label').style.opacity = cat.custom ? '1' : '0.4';
  }
  document.getElementById('modal-cat-emoji-btn').textContent = _catEmoji;
  document.getElementById('modal-cat').classList.add('show');
};

window.trocarEmojiCat = function(){
  escolherEmoji(e=>{
    _catEmoji = e;
    document.getElementById('modal-cat-emoji-btn').textContent = e;
  });
};

window.fecharModalCat = function(){ _catEdit = null; _fecharOverlay('modal-cat'); };

window.salvarCategoria = async function(){
  const nome = (document.getElementById('modal-cat-nome').value || '').trim();
  if(!nome){ showToast('Digite o nome da categoria','error'); return; }
  const all = getAllCats();

  if(!_catEdit){
    // CRIAR NOVA
    if(all[nome]){ showToast('Já existe categoria com esse nome','error'); return; }
    await push(ref(db,`grupos/${grupoCode}/categorias`),{nome, emoji:_catEmoji, color:'#d4af37'});
    if(_catOrigem === 'tx'){ _modalCat = nome; _catTocadaManual = true; renderModalCats(); }
    else if(_catOrigem === 'fixo'){ _fixoCat = nome; renderModalCatsFixo(); }
    showToast('Categoria criada! ✓');
  } else if(_catEdit.custom){
    // EDITAR CRIADA — emoji + possível rename (migra gastos/fixos vigentes)
    const nomeAntigo = _catEdit.nome;
    if(nome !== nomeAntigo && all[nome]){ showToast('Já existe categoria com esse nome','error'); return; }
    await update(ref(db,`grupos/${grupoCode}/categorias/${_catEdit._key}`),{nome, emoji:_catEmoji});
    if(nome !== nomeAntigo) await migrarNomeCategoria(nomeAntigo, nome);
    showToast('Categoria atualizada! ✓');
  } else {
    // EDITAR PADRÃO — só emoji (override em categoriasMeta)
    await update(ref(db,`grupos/${grupoCode}/categoriasMeta/${_catEdit.nome}`),{emoji:_catEmoji});
    showToast('Emoji atualizado! ✓');
  }
  fecharModalCat();
};

// Migra cat dos gastos e fixos VIGENTES (não mexe em histórico arquivado, que é snapshot).
async function migrarNomeCategoria(antigo, novo){
  const updates = {};
  (_dados.gastos||[]).forEach(g=>{ if(g.cat===antigo) updates[`gastos/${g._key}/cat`]=novo; });
  (_dados.fixos ||[]).forEach(f=>{ if(f.cat===antigo) updates[`fixos/${f._key}/cat`]=novo; });
  if(_modalCat===antigo) _modalCat=novo;
  if(_fixoCat ===antigo) _fixoCat =novo;
  if(Object.keys(updates).length) await update(ref(db,`grupos/${grupoCode}`), updates);
}
```

- [ ] **Step 3: Declarar o estado compartilhado dos pickers**

`salvarCategoria` (acima) lê/escreve `_catTocadaManual`, e a Task 5 usa também `_modalCatExpandido`. Como o app roda em módulo (strict mode), atribuir a variável não declarada quebra. Busque `let _modalCat` (~1234) e logo abaixo adicione **as duas** (a Task 5 reaproveita, não redeclara):

```javascript
let _modalCatExpandido = false; // grade de categorias aberta?
let _catTocadaManual = false;   // usuário escolheu categoria na mão neste lançamento?
```

- [ ] **Step 4: Verificar no preview**

Recarregue. Estando logado num grupo (Pro), abra o modal de Gastei, toque em ＋ Nova (após a Task 5) OU rode `preview_eval: abrirModalCategoria('nova','tx')`. Confirme: o modal abre, o botão de emoji abre o seletor da Task 3, escolher um emoji atualiza o botão, salvar com nome cria a categoria. `preview_console_logs` sem erros.

- [ ] **Step 5: Commit**

```bash
git add index.html
git commit -m "feat(cat): modal criar/editar categoria com emoji visual + rename com migração"
```

---

## Task 5: Seletor recolhido + sugestão ao digitar (modal de lançamento)

**Files:**
- Modify: `index.html` — `renderModalCats` (~1329), `selecionarCat` (~1351), `abrirModal` (~1237), HTML do `modal-cats-wrap` (~3735)

- [ ] **Step 1: Confirmar o estado de expansão**

As variáveis `_modalCatExpandido` e `_catTocadaManual` já foram declaradas na Task 4 (Step 3), logo abaixo de `let _modalCat`. **Não redeclare** (redeclaração de `let` é erro). Apenas confirme que existem antes de prosseguir.

- [ ] **Step 2: Reescrever `renderModalCats` para recolhido/expandido**

Substitua `renderModalCats` inteira por:

```javascript
function renderModalCats(){
  const cont = document.getElementById('modal-cats');
  if(!cont) return;
  const all = getAllCats();
  const escapar = s => String(s).replace(/'/g,"\\'");
  const info = all[_modalCat] || {emoji:'📦'};

  if(!_modalCatExpandido){
    // RECOLHIDO — só a sugerida + "Trocar"
    cont.className = 'modal-cats-recolhido';
    cont.innerHTML = `
      <div class="cat-chip-sel" onclick="expandirCats()">
        <span class="cat-chip-emoji">${esc(info.emoji)}</span>
        <span class="cat-chip-nome">${esc(_modalCat)}</span>
      </div>
      <button type="button" class="cat-trocar-btn" onclick="expandirCats()">Trocar ▾</button>`;
    return;
  }

  // EXPANDIDO — grade completa + Nova + Gerenciar
  cont.className = 'modal-cats';
  cont.innerHTML = Object.entries(all).map(([nome,inf])=>`
    <div class="cat-pick ${nome===_modalCat?'active':''}" onclick="selecionarCat('${escapar(nome)}')">
      <div class="cat-pick-emoji">${esc(inf.emoji)}</div>
      <div class="cat-pick-label">${esc(nome)}</div>
    </div>`).join('') + `
    <div class="cat-pick cat-nova" onclick="criarCategoria('tx')">
      <div class="cat-pick-emoji">＋</div>
      <div class="cat-pick-label">Nova</div>
    </div>
    <div class="cat-pick cat-nova" onclick="abrirGerenciarCats()">
      <div class="cat-pick-emoji">⚙️</div>
      <div class="cat-pick-label">Gerenciar</div>
    </div>`;
}

window.expandirCats = function(){ _modalCatExpandido = true; renderModalCats(); };
```

- [ ] **Step 3: Atualizar `selecionarCat` para recolher ao escolher**

Substitua `window.selecionarCat` por:

```javascript
window.selecionarCat = function(nome){
  _modalCat = nome;
  _catTocadaManual = true;
  _modalCatExpandido = false;
  renderModalCats();
};
```

- [ ] **Step 4: Em `abrirModal`, resetar estado e sugerir categoria**

Na função `window.abrirModal = function(tipo, editKey){...}`:

(a) No bloco de **edição** (`if(_modalEditKey){ ... }`), onde já existe `if(tipo==='out') _modalCat = item.cat || 'Alimentação';`, adicione logo depois:

```javascript
    _catTocadaManual = true; // edição já tem categoria definida
```

(b) No bloco de **criação** (`else { ... }`, após `descEl.value = '';`), adicione:

```javascript
    _catTocadaManual = false;
    _modalCat = sugerirCategoria('');
```

(c) Antes da chamada `renderModalCats();` (perto do fim da função, ~1303), adicione:

```javascript
  _modalCatExpandido = false;
```

(d) Logo após `renderModalCats();`, anexar o listener de digitação (idempotente):

```javascript
  if(!descEl._sugListener){
    descEl._sugListener = true;
    descEl.addEventListener('input', ()=>{
      if(_modalTipo!=='out' || _modalEditKey || _catTocadaManual) return;
      const nova = sugerirCategoria(descEl.value);
      if(nova !== _modalCat){ _modalCat = nova; if(!_modalCatExpandido) renderModalCats(); }
    });
  }
```

- [ ] **Step 5: CSS do chip recolhido**

Perto de `.cat-pick-emoji` (~2985), inserir:

```css
.modal-cats-recolhido{display:flex;align-items:center;gap:8px;margin-bottom:14px;}
.cat-chip-sel{display:flex;align-items:center;gap:8px;background:var(--gold-bg);border:1.5px solid var(--gold);border-radius:12px;padding:9px 12px;cursor:pointer;}
.cat-chip-emoji{font-size:18px;line-height:1;}
.cat-chip-nome{font-size:14px;font-weight:600;color:var(--text);}
.cat-trocar-btn{background:transparent;border:1px solid var(--border);color:var(--gold);border-radius:10px;padding:9px 12px;font-size:13px;font-weight:600;cursor:pointer;font-family:inherit;}
```

- [ ] **Step 6: Verificar no preview**

Recarregue. Logado num grupo Pro: abra Gastei. Esperado: aparece só o chip sugerido + "Trocar". Digite "uber" no campo Descrição → chip vira Transporte (use `preview_fill` no `#modal-desc` e `preview_snapshot`). Toque "Trocar" → grade abre com ＋Nova e ⚙️Gerenciar. Escolha uma → recolhe. `preview_console_logs` sem erros. `preview_screenshot` dos dois estados.

- [ ] **Step 7: Commit**

```bash
git add index.html
git commit -m "feat(cat): seletor recolhido com sugestão ao digitar no lançamento"
```

---

## Task 6: Tela "Gerenciar categorias"

**Files:**
- Modify: `index.html` — overlay HTML (após `modal-cat`), JS (perto das funções de categoria), CSS

- [ ] **Step 1: Adicionar o overlay HTML `modal-gerenciar`**

Após o `</div>` que fecha `modal-cat`, inserir:

```html
<div class="modal-overlay" id="modal-gerenciar" onclick="if(event.target===this)fecharGerenciar()">
  <div class="modal-card" style="max-width:380px">
    <div class="modal-handle"></div>
    <div class="modal-titulo">⚙️ Gerenciar categorias</div>
    <div class="modal-sub">Toque pra trocar o emoji. Renomear/apagar só nas que você criou.</div>
    <div class="ger-lista" id="ger-lista"></div>
    <div class="modal-actions">
      <button class="modal-btn modal-btn-cancel" onclick="fecharGerenciar()">Fechar</button>
      <button class="modal-btn modal-btn-confirm in" onclick="criarCategoria('gerenciar')">＋ Nova</button>
    </div>
  </div>
</div>
```

- [ ] **Step 2: Adicionar o CSS da lista**

Perto do CSS de categorias (~2985), inserir:

```css
.ger-lista{display:flex;flex-direction:column;gap:8px;max-height:320px;overflow-y:auto;margin-bottom:14px;}
.ger-item{display:flex;align-items:center;gap:10px;background:var(--surface2);border:1px solid var(--border);border-radius:12px;padding:10px 12px;}
.ger-emoji{font-size:22px;line-height:1;cursor:pointer;}
.ger-nome{flex:1;font-size:14px;color:var(--text);}
.ger-tag{font-size:10px;color:var(--text3);background:var(--surface);border:1px solid var(--border);border-radius:6px;padding:1px 6px;}
.ger-acao{background:transparent;border:1px solid var(--border);color:var(--gold);border-radius:8px;padding:6px 9px;font-size:12px;cursor:pointer;font-family:inherit;}
.ger-acao.del{color:var(--red);}
```

- [ ] **Step 3: Adicionar o JS `abrirGerenciarCats` / `renderGerenciarCats` / `fecharGerenciar`**

Perto das funções de categoria (após `salvarCategoria`/`migrarNomeCategoria` da Task 4), inserir:

```javascript
// ── GERENCIAR CATEGORIAS ────────────────────────────────────────────────────
window.abrirGerenciarCats = function(){
  if(!isPro()){ abrirModalPro('feature'); return; }
  renderGerenciarCats();
  document.getElementById('modal-gerenciar').classList.add('show');
};
window.fecharGerenciar = function(){ _fecharOverlay('modal-gerenciar'); };

function renderGerenciarCats(){
  const cont = document.getElementById('ger-lista');
  if(!cont) return;
  const all = getAllCats();
  const escapar = s => String(s).replace(/'/g,"\\'");
  cont.innerHTML = Object.entries(all).map(([nome,info])=>{
    const cat = {nome, emoji:info.emoji, custom:!!info.custom, _key:info._key||null};
    const payload = encodeURIComponent(JSON.stringify(cat));
    const acoes = info.custom
      ? `<button class="ger-acao" onclick="abrirModalCategoria('editar','gerenciar', JSON.parse(decodeURIComponent('${payload}')))">Editar</button>
         <button class="ger-acao del" onclick="removerCategoria('${info._key}','${escapar(nome)}')">Apagar</button>`
      : `<button class="ger-acao" onclick="abrirModalCategoria('editar','gerenciar', JSON.parse(decodeURIComponent('${payload}')))">Emoji</button>
         <span class="ger-tag">padrão</span>`;
    return `
      <div class="ger-item">
        <span class="ger-emoji" onclick="abrirModalCategoria('editar','gerenciar', JSON.parse(decodeURIComponent('${payload}')))">${esc(info.emoji)}</span>
        <span class="ger-nome">${esc(nome)}</span>
        ${acoes}
      </div>`;
  }).join('');
}
```

- [ ] **Step 4: Re-renderizar o gerenciar quando categorias mudam**

No listener `onValue(ref(db,`grupos/${grupoCode}/categorias`), ...)` (~553), dentro do callback, após as duas linhas que re-renderizam `modal-tx`/`modal-fixo`, adicionar:

```javascript
    if(document.getElementById('modal-gerenciar').classList.contains('show')) renderGerenciarCats();
```

- [ ] **Step 5: Verificar no preview**

Recarregue. Logado Pro: abra Gastei → Trocar → ⚙️ Gerenciar (ou `preview_eval: abrirGerenciarCats()`). Esperado: lista com as 8 padrão (tag "padrão", botão "Emoji") e as criadas (botões "Editar"/"Apagar"). Toque numa padrão → modal de emoji; troque → persiste (recarregue e confira). Renomeie uma criada → confira que os gastos dela aparecem com o nome novo na lista de movimentações. `preview_screenshot` da tela. Console sem erros.

- [ ] **Step 6: Commit**

```bash
git add index.html
git commit -m "feat(cat): tela Gerenciar categorias (trocar emoji, renomear, apagar)"
```

---

## Task 7: Apontar o ＋Nova das contas fixas pro novo modal

**Files:**
- Modify: `index.html` — `renderModalCatsFixo` (~1458)

- [ ] **Step 1: Repontar o create do modal fixo**

Em `renderModalCatsFixo`, o bloco `cat-nova` chama `criarCategoria('fixo')` — isso já cai no novo modal (Task 4). Nenhuma mudança de chamada é necessária. Apenas **confirme** que o `onclick` do chip "Nova" é `onclick="criarCategoria('fixo')"`. Se estiver diferente, ajuste pra:

```html
    <div class="cat-pick cat-nova" onclick="criarCategoria('fixo')">
      <div class="cat-pick-emoji">＋</div>
      <div class="cat-pick-label">Nova</div>
    </div>
```

- [ ] **Step 2: Verificar no preview**

Abra "Nova conta" fixa → toque ＋ Nova nas categorias → deve abrir o modal estilizado (não `prompt()`). Criar uma categoria seleciona ela no modal fixo. Console sem erros.

- [ ] **Step 3: Commit (se houve mudança)**

```bash
git add index.html
git commit -m "chore(cat): create de categoria no modal fixo usa o novo modal"
```

---

## Task 8: Últimas movimentações = só de hoje

**Files:**
- Modify: `index.html` — `renderInicio`, trecho das "ÚLTIMAS MOVIMENTAÇÕES" (~900-942), botão "Ver todas" (HTML ~3369)

- [ ] **Step 1: Filtrar a lista do Início para só hoje (mês atual)**

Localize o bloco que começa em `// ÚLTIMAS MOVIMENTAÇÕES` (~900). Substitua **desde a linha `const txCard = document.getElementById('card-tx');`** até **a linha `if(emptyEl) emptyEl.style.display = tx.length === 0 ? 'block' : 'none';`** (inclusive — todo o miolo: definição de `tx`, o `if(tx.length>0){...}else{...}`, o `attachSwipe('tx-lista')` e o bloco de empty-state) pelo código abaixo. O resultado já inclui `const txCard`, `const emptyEl`, o render e o `attachSwipe` uma única vez — não deixe duplicatas dessas linhas. As linhas seguintes (`// META MENSAL` / `renderMeta()`) permanecem intactas:

```javascript
  const txCard = document.getElementById('card-tx');
  const todasTx = [
    ...ganhos.map(g=>({...g, _tipo:'in', _label:g.desc||'Ganho'})),
    ...gastos.map(g=>({...g, _tipo:'out', _label:g.nome||'Gasto'}))
  ].sort((a,b)=>(b.ts||0)-(a.ts||0));

  // Mês atual: só as de hoje. Mês passado: as mais recentes daquele mês (hoje não existe lá).
  const ehMesAtual = getMesAtivo() === getML();
  let tx;
  if(ehMesAtual){
    const hojeStr = new Date().toDateString();
    tx = todasTx.filter(t => t.ts && new Date(t.ts).toDateString() === hojeStr);
  } else {
    tx = todasTx.slice(0, isTodos ? 30 : 6);
  }
  const ocultas = Math.max(0, todasTx.length - tx.length);

  // Atualiza o rótulo do botão "Ver todas" com a contagem do que está oculto.
  const verBtn = document.getElementById('btn-ver-todas');
  if(verBtn) verBtn.textContent = ocultas > 0 ? `Ver mais (${ocultas}) ›` : 'Ver todas ›';

  const emptyEl = document.getElementById('empty-state');
  if(todasTx.length === 0){
    // Mês realmente vazio → guia de estado vazio.
    txCard.style.display = 'none';
    if(emptyEl) emptyEl.style.display = 'block';
  } else {
    txCard.style.display = 'block';
    if(emptyEl) emptyEl.style.display = 'none';
    if(tx.length > 0){
      const txAllCats = getAllCats();
      document.getElementById('tx-lista').innerHTML = tx.map(t=>{
        const sinal = t._tipo==='in' ? '+' : '−';
        const catInfo = (t._tipo === 'out' && t.cat) ? (txAllCats[t.cat] || null) : null;
        const iconContent = catInfo ? catInfo.emoji : (t._tipo==='in' ? '↑' : '↓');
        const catTag = catInfo ? ` · <span class="cat-inline">${esc(t.cat)}</span>` : '';
        const dono = isTodos ? membros.find(mm=>mm.id===t.pid) : null;
        const donoTag = dono ? `<span style="background:${dono.cor}22;color:${dono.cor};padding:1px 7px;border-radius:6px;font-size:10px;font-weight:600;margin-right:6px">${esc(dono.nome)}</span>` : '';
        const isOwnerOfTx = t.pid === meuPerfil;
        const onClickFn = isOwnerOfTx ? `onclick="abrirModal('${t._tipo}','${t._key}')"` : '';
        const cursor = isOwnerOfTx ? 'cursor:pointer' : '';
        const editableClass = isOwnerOfTx ? ' editable' : '';
        return `
          <div class="tx-item">
            <div class="tx-icon-circle ${t._tipo}${editableClass}" style="${cursor}" ${onClickFn}>${iconContent}</div>
            <div class="tx-info" style="${cursor}" ${onClickFn}>
              <div class="tx-nome">${donoTag}${esc(t._label)}</div>
              <div class="tx-data">${formatarDataTx(t)}${catTag}</div>
            </div>
            <div class="tx-valor ${t._tipo}" style="${cursor}" ${onClickFn}>${sinal}${fmt(t.valor)}</div>
            ${isOwnerOfTx?`<button class="tx-del" onclick="event.stopPropagation();${t._tipo==='in'?'delGanho':'delGasto'}('${t._key}')">×</button>`:''}
          </div>`;
      }).join('');
    } else {
      // Mês atual com movimentos, mas nada hoje.
      document.getElementById('tx-lista').innerHTML =
        '<div class="empty" style="padding:16px;text-align:center;color:var(--text3);font-size:13px">Nada lançado hoje ainda</div>';
    }
  }

  // SWIPE-TO-DELETE nas movimentações
  attachSwipe('tx-lista');
```

> Atenção: o bloco original tinha `attachSwipe('tx-lista');` e o empty-state logo após o `if(tx.length>0){...}else{...}`. O novo código acima já incorpora o `attachSwipe`. Remova quaisquer linhas duplicadas remanescentes (`txCard`, `attachSwipe`, empty-state) que ficaram do bloco antigo.

- [ ] **Step 2: Dar um id ao botão "Ver todas"**

No HTML (~3369), o botão de expandir tem `class="expandir-btn" onclick="toggleTodasMovimentacoes()"`. Adicione `id="btn-ver-todas"`:

```html
<button class="expandir-btn" id="btn-ver-todas" onclick="toggleTodasMovimentacoes()" style="background:var(--gold-bg);border:1px solid var(--border2);color:var(--gold);cursor:pointer;font-size:11px;font-weight:600;padding:3px 10px;border-radius:8px;margin-left:auto;font-family:inherit">Ver todas ›</button>
```

- [ ] **Step 3: Verificar no preview**

Recarregue. No mês atual: o card "Últimas movimentações" mostra só lançamentos de hoje; se nada hoje, mostra "Nada lançado hoje ainda" e o botão "Ver mais (N) ›". "Ver todas" abre a lista completa do mês (inalterada). Navegue pra um mês passado (setinha) → mostra as recentes daquele mês. `preview_screenshot` dos casos. Console sem erros.

- [ ] **Step 4: Commit**

```bash
git add index.html
git commit -m "feat(ux): Últimas movimentações mostram só as de hoje (resto em Ver mais)"
```

---

## Task 9: Confirmar exclusão (caixa estilizada) em tudo que apaga

**Files:**
- Modify: `index.html` — overlay HTML (após `modal-gerenciar`), JS `confirmarAcao` (perto de `showToastUndo` ~277), funções `delGanho`/`delGasto`/`delFixo`/`delHistorico` (~2041-2118)

- [ ] **Step 1: Adicionar o overlay HTML `modal-confirm`**

Após o `</div>` que fecha `modal-gerenciar`, inserir:

```html
<div class="modal-overlay" id="modal-confirm" onclick="if(event.target===this)confirmCancelar()">
  <div class="modal-card" style="max-width:340px">
    <div class="modal-handle"></div>
    <div class="modal-titulo" id="confirm-titulo">Tem certeza?</div>
    <div class="modal-sub" id="confirm-msg" style="margin-bottom:18px"></div>
    <div class="modal-actions">
      <button class="modal-btn modal-btn-cancel" onclick="confirmCancelar()">Cancelar</button>
      <button class="modal-btn modal-btn-confirm" id="confirm-ok" style="background:var(--red);color:#fff" onclick="confirmOk()">Apagar</button>
    </div>
  </div>
</div>
```

- [ ] **Step 2: Adicionar o helper `confirmarAcao`**

Logo após `showToastUndo` (~277), inserir:

```javascript
// Caixa de confirmação estilizada (substitui confirm() nativo nas exclusões).
let _confirmCb = null;
function confirmarAcao({titulo, msg, textoBotao='Apagar', onConfirm}){
  _confirmCb = onConfirm;
  document.getElementById('confirm-titulo').textContent = titulo || 'Tem certeza?';
  document.getElementById('confirm-msg').textContent = msg || '';
  document.getElementById('confirm-ok').textContent = textoBotao;
  document.getElementById('modal-confirm').classList.add('show');
}
window.confirmCancelar = function(){ _confirmCb = null; _fecharOverlay('modal-confirm'); };
window.confirmOk = function(){
  const cb = _confirmCb; _confirmCb = null;
  _fecharOverlay('modal-confirm', ()=>{ if(cb) cb(); });
};
```

- [ ] **Step 3: Embrulhar `delGanho` e `delGasto` com confirmação**

Substitua `window.delGanho` e `window.delGasto` por:

```javascript
window.delGanho = function(key){
  const g = (_dados.ganhos||[]).find(x=>x._key===key);
  if(!g) return;
  confirmarAcao({
    titulo:'Apagar entrada?',
    msg:`"${g.desc||'Ganho'}" (${fmt(g.valor||0)}) será removido.`,
    onConfirm:()=>{
      const timer = setTimeout(async()=>{ await remove(ref(db,`grupos/${grupoCode}/ganhos/${key}`)); _pendingDeletes.delete(key); }, 4000);
      _pendingDeletes.set(key, timer); renderTudo();
      showToastUndo('Entrada removida', ()=>{ clearTimeout(timer); _pendingDeletes.delete(key); renderTudo(); });
    }
  });
}
window.delGasto = function(key){
  const g = (_dados.gastos||[]).find(x=>x._key===key);
  if(!g) return;
  confirmarAcao({
    titulo:'Apagar gasto?',
    msg:`"${g.nome||'Gasto'}" (${fmt(g.valor||0)}) será removido.`,
    onConfirm:()=>{
      const timer = setTimeout(async()=>{ await remove(ref(db,`grupos/${grupoCode}/gastos/${key}`)); _pendingDeletes.delete(key); }, 4000);
      _pendingDeletes.set(key, timer); renderTudo();
      showToastUndo('Gasto removido', ()=>{ clearTimeout(timer); _pendingDeletes.delete(key); renderTudo(); });
    }
  });
}
```

- [ ] **Step 4: Embrulhar `delFixo` com confirmação**

Substitua `window.delFixo` por:

```javascript
window.delFixo = function(key){
  const f = (_dados.fixos||[]).find(x=>x._key===key);
  if(!f) return;
  confirmarAcao({
    titulo:'Apagar conta fixa?',
    msg:`"${f.nome||'Conta'}" (${fmt(f.valor||0)}) será removida.`,
    onConfirm:()=>{
      const timer = setTimeout(async()=>{ await remove(ref(db,`grupos/${grupoCode}/fixos/${key}`)); _pendingDeletes.delete(key); }, 4000);
      _pendingDeletes.set(key, timer); renderTudo();
      showToastUndo('Conta fixa removida', ()=>{ clearTimeout(timer); _pendingDeletes.delete(key); renderTudo(); });
    }
  });
}
```

- [ ] **Step 5: Embrulhar `delHistorico` com confirmação**

Substitua `window.delHistorico` por:

```javascript
window.delHistorico = function(key, mes){
  if(!(_dados.historico||[]).find(h=>h._key===key)) return;
  confirmarAcao({
    titulo:'Apagar arquivo do histórico?',
    msg:`O resumo arquivado de ${mes} será removido.`,
    onConfirm:()=>{
      const timer = setTimeout(async()=>{ await remove(ref(db,`grupos/${grupoCode}/historico/${key}`)); _pendingDeletes.delete(key); }, 4000);
      _pendingDeletes.set(key, timer); renderTudo();
      showToastUndo(`Histórico de ${mes} removido`, ()=>{ clearTimeout(timer); _pendingDeletes.delete(key); renderTudo(); });
    }
  });
}
```

- [ ] **Step 6: Converter `removerCategoria` (apagar categoria na tela Gerenciar) pra caixa estilizada**

A função `removerCategoria` ainda usa `confirm()` nativo. Substitua `window.removerCategoria = async function(key, nome){ ... }` por:

```javascript
window.removerCategoria = function(key, nome){
  confirmarAcao({
    titulo:'Remover categoria?',
    msg:`Gastos com "${nome}" continuam, mas voltam pra "Outros".`,
    textoBotao:'Remover',
    onConfirm: async ()=>{
      await remove(ref(db,`grupos/${grupoCode}/categorias/${key}`));
      if(_modalCat === nome) _modalCat = 'Outros';
      if(_fixoCat  === nome) _fixoCat  = 'Outros';
      renderModalCats();
      renderModalCatsFixo();
      if(document.getElementById('modal-gerenciar').classList.contains('show')) renderGerenciarCats();
      showToast('Categoria removida');
    }
  });
}
```

- [ ] **Step 7: Verificar no preview**

Recarregue. Logado Pro:
- Toque o `×` (e teste o **swipe**) numa movimentação → abre a caixa "Apagar gasto?"; **Cancelar** não apaga; **Apagar** remove E mostra o toast "Desfazer" por 4s; "Desfazer" restaura.
- Repita pra conta fixa e pra um card do Histórico.
- Na tela Gerenciar, "Apagar" numa categoria criada abre a caixa estilizada (não o `confirm()` feio).
`preview_screenshot` da caixa de confirmação. Console sem erros.

- [ ] **Step 8: Commit**

```bash
git add index.html
git commit -m "feat(ux): confirmação estilizada antes de apagar (movimentação, fixa, histórico)"
```

---

## Verificação final (todas as tarefas)

- [ ] Console sem erros em toda a navegação (`preview_console_logs`).
- [ ] Lançar Gastei: chip recolhido sugerido; digitar "uber"→Transporte, "ifood"→Alimentação, texto desconhecido→mais usada/Outros; "Trocar" expande; escolher recolhe.
- [ ] ＋Nova abre modal estilizado com emoji visual (sem `prompt()`); cria categoria.
- [ ] Gerenciar: trocar emoji de padrão persiste após reload; renomear criada migra gastos vigentes; apagar criada volta gastos pra "Outros".
- [ ] Início: mês atual mostra só hoje (+ aviso quando vazio); "Ver mais"/"Ver todas" abre lista completa; mês passado mostra recentes.
- [ ] Excluir (×, swipe) em ganho/gasto/fixa/histórico pede confirmação; Cancelar aborta; Apagar remove e ainda oferece Desfazer.
- [ ] Lançamentos antigos seguem com a categoria certa (nada quebrou no `getAllCats`).

## Notas para publicar (fora do plano técnico)

Depois de tudo verificado, pra ir pro celular: `git checkout main`, merge da branch, `git push origin main` (GitHub Pages). Confirmar com o usuário antes de publicar.
