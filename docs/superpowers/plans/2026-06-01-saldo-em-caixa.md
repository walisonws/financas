# Saldo em Caixa — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Adicionar um "valor em caixa" por pessoa por mês que entra no cálculo do saldo e carrega automaticamente de um mês pro outro ao fechar o mês.

**Architecture:** App single-file (`index.html`) com Firebase Realtime Database. O caixa fica num nó novo `grupos/{code}/caixa/{mesSlug}/{pid}`, separado dos lançamentos. Quatro pontos de cálculo de saldo passam a somar o caixa; uma linha editável aparece no Início; `fecharMes` grava o saldo final de cada membro como caixa do mês seguinte.

**Tech Stack:** HTML/CSS/JS vanilla, Firebase RTDB (modular SDK v10.12), GitHub Pages.

**Nota sobre testes:** este projeto não tem framework de testes nem build. A verificação é (a) automatizada via Node só para a lógica pura de datas (Task 1), e (b) manual no navegador para o resto. Cada task de UI traz um roteiro de verificação manual explícito.

**Convenção de edição:** os números de linha mudam a cada edição. Use o `old_string` único mostrado em cada passo (nome de função / string exata) em vez de confiar em números de linha.

---

## Estrutura de arquivos

- **Modificar:** `C:\Users\WSzin\Desktop\financas\index.html` (único arquivo do app)
- **Criar (temporário, descartável):** `C:\Users\WSzin\Desktop\financas\_test_caixa.mjs` (teste Node das funções de data; removido ao final)
- **Já existe:** spec em `docs/superpowers/specs/2026-06-01-saldo-em-caixa-design.md`

---

## Task 1: Lógica pura de datas + helpers de caixa

**Files:**
- Modify: `index.html` (bloco de utils, logo após `mesParaIndice`)
- Test: `_test_caixa.mjs` (Node, temporário)

- [ ] **Step 1: Escrever o teste que falha**

Criar `C:\Users\WSzin\Desktop\financas\_test_caixa.mjs`:

```js
import assert from 'node:assert';

const MESES = ['Janeiro','Fevereiro','Março','Abril','Maio','Junho','Julho','Agosto','Setembro','Outubro','Novembro','Dezembro'];

function mesParaIndice(mesStr){
  if(!mesStr) return -1;
  const parts = String(mesStr).trim().split(' ');
  const mi = MESES.indexOf(parts[0]);
  const yr = parseInt(parts[1]);
  if(mi < 0 || isNaN(yr)) return -1;
  return yr*12 + mi;
}
function mesSlugDe(mesLabel){ return mesLabel.toLowerCase().replace(/\s+/g,'-'); }
function mesSeguinteSlug(mesLabel){
  const idx = mesParaIndice(mesLabel);
  if(idx < 0) return null;
  const prox = idx + 1;
  const ano = Math.floor(prox/12);
  const mes = prox % 12;
  return mesSlugDe(MESES[mes] + ' ' + ano);
}

// mês normal
assert.strictEqual(mesSeguinteSlug('Junho 2026'), 'julho-2026');
// virada de ano: Dezembro -> Janeiro do ano seguinte
assert.strictEqual(mesSeguinteSlug('Dezembro 2026'), 'janeiro-2027');
// slug normaliza acento/caixa preservando o texto original
assert.strictEqual(mesSlugDe('Março 2026'), 'março-2026');
// inválido
assert.strictEqual(mesSeguinteSlug('xxx'), null);

console.log('OK — todos os asserts passaram');
```

- [ ] **Step 2: Rodar o teste e ver passar (funções já corretas)**

Run: `node "C:\Users\WSzin\Desktop\financas\_test_caixa.mjs"`
Expected: `OK — todos os asserts passaram`

> Observação: aqui o teste serve pra travar o comportamento da matemática de datas (em especial a virada de Dezembro→Janeiro). Se algum assert falhar, corrija `mesSeguinteSlug` antes de seguir.

- [ ] **Step 3: Adicionar os helpers no `index.html`**

Localizar o fim da função `mesParaIndice` (a linha `}` logo antes do comentário `// Roda 1x por sessão.`). Inserir ANTES desse comentário o bloco abaixo (`old_string` = a linha do comentário; prefixe o novo bloco):

`old_string`:
```js
// Roda 1x por sessão. No início de um mês novo, copia as contas fixas do mês
```

`new_string`:
```js
// ── SALDO EM CAIXA ──────────────────────────────────────────────────────────
// Converte "Junho 2026" → "junho-2026" (mesma regra do slug do histórico).
function mesSlugDe(mesLabel){ return String(mesLabel||'').toLowerCase().replace(/\s+/g,'-'); }
// Caixa inicial de um membro num mês. Ausente = 0.
function getCaixa(pid, mesSlug){
  const slug = mesSlug || mesSlugDe(getMesAtivo());
  const v = (((_dados.caixa||{})[slug])||{})[pid];
  return typeof v === 'number' ? v : 0;
}
// Grava o caixa inicial de um membro num mês.
async function setCaixa(pid, valor, mesSlug){
  const slug = mesSlug || mesSlugDe(getMesAtivo());
  await set(ref(db,`grupos/${grupoCode}/caixa/${slug}/${pid}`), valor||0);
}
// "Junho 2026" → "julho-2026" (vira o ano em Dezembro).
function mesSeguinteSlug(mesLabel){
  const idx = mesParaIndice(mesLabel);
  if(idx < 0) return null;
  const prox = idx + 1;
  return mesSlugDe(MESES[prox % 12] + ' ' + Math.floor(prox/12));
}

// Roda 1x por sessão. No início de um mês novo, copia as contas fixas do mês
```

- [ ] **Step 4: Inicializar `_dados.caixa` nas duas declarações**

Edit 1 — `old_string`:
```js
let _dados = { ganhos:[], gastos:[], fixos:[], historico:[] };
```
`new_string`:
```js
let _dados = { ganhos:[], gastos:[], fixos:[], historico:[], caixa:{} };
```

Edit 2 — `old_string`:
```js
  _dados = { ganhos:[], gastos:[], fixos:[], historico:[] };
```
`new_string`:
```js
  _dados = { ganhos:[], gastos:[], fixos:[], historico:[], caixa:{} };
```

- [ ] **Step 5: Carregar o caixa do Firebase (novo listener `onValue`)**

Localizar o listener de `historico`. Inserir um listener de `caixa` ANTES dele.

`old_string`:
```js
  onValue(ref(db,`grupos/${grupoCode}/historico`), snap=>{
```
`new_string`:
```js
  onValue(ref(db,`grupos/${grupoCode}/caixa`), snap=>{
    _dados.caixa = snap.val() || {};
    renderTudo();
  });
  onValue(ref(db,`grupos/${grupoCode}/historico`), snap=>{
```

- [ ] **Step 6: Verificação manual rápida no navegador (console)**

Abrir o app (servidor local — ver Task 6 para como servir, ou abrir o arquivo no navegador já logado num grupo). No console do navegador (F12):

```js
getCaixa('qualquer-pid')   // → 0 (ainda não há dado)
mesSeguinteSlug('Dezembro 2026')  // → "janeiro-2027"
```
Expected: `0` e `"janeiro-2027"`. Sem erros no console.

- [ ] **Step 7: Commit**

```bash
cd /c/Users/WSzin/Desktop/financas
git add index.html
git commit -m "feat: helpers e carregamento do saldo em caixa

Co-Authored-By: Claude Opus 4.8 <noreply@anthropic.com>"
```

---

## Task 2: Incluir o caixa no cálculo do saldo (4 pontos + histórico)

**Files:**
- Modify: `index.html` (saldo hero, renderResumoSolo, renderResumoGrupo, compartilharResumo, fecharMes)

- [ ] **Step 1: Saldo hero (tela Início)**

`old_string`:
```js
  const totalGastoReal   = totalGastoDiario + totalFixoPago;
  const saldo            = totalGanho - totalGastoReal;
```
`new_string`:
```js
  const totalGastoReal   = totalGastoDiario + totalFixoPago;
  const caixaMes         = isTodos
    ? membros.reduce((a,m)=>a+getCaixa(m.id),0)
    : getCaixa(target);
  const saldo            = caixaMes + totalGanho - totalGastoReal;
```

- [ ] **Step 2: Resumo Individual (modo mês)**

`old_string`:
```js
    const tFixoPago = fixosPagos.reduce((a,b)=>a+(b.valor||0),0);
    const tFixoPend = fixosPendentes.reduce((a,b)=>a+(b.valor||0),0);
    const saldo     = tG - tDiario - tFixoPago;
```
`new_string`:
```js
    const tFixoPago = fixosPagos.reduce((a,b)=>a+(b.valor||0),0);
    const tFixoPend = fixosPendentes.reduce((a,b)=>a+(b.valor||0),0);
    const caixaMes  = getCaixa(meuPerfil);
    const saldo     = caixaMes + tG - tDiario - tFixoPago;
```

E logo abaixo, junto dos outros `textContent` do detalhamento, preencher a linha nova (criada na Task 3, mas o JS pode referenciar com segurança usando `?.`). Adicionar após a linha `document.getElementById('rs-fixo-pend').textContent = fmt(tFixoPend);`:

`old_string`:
```js
    document.getElementById('rs-fixo-pend').textContent   = fmt(tFixoPend);
```
`new_string`:
```js
    document.getElementById('rs-fixo-pend').textContent   = fmt(tFixoPend);
    const rsCaixaEl = document.getElementById('rs-caixa'); if(rsCaixaEl) rsCaixaEl.textContent = fmt(caixaMes);
```

- [ ] **Step 3: Resumo Grupo (por membro)**

`old_string`:
```js
    const tFixo   = fixosPagos.reduce((a,b)=>a+(b.valor||0),0);
    const saldo   = tG - tDiario - tFixo;
```
`new_string`:
```js
    const tFixo   = fixosPagos.reduce((a,b)=>a+(b.valor||0),0);
    const caixaM  = incluiFixos ? getCaixa(m.id) : 0;
    const saldo   = caixaM + tG - tDiario - tFixo;
```

E na string HTML do card do membro, adicionar a linha de caixa logo após a linha de "Entradas". Localizar:

`old_string`:
```js
      <div class="rm-row"><span>Entradas</span><span style="color:var(--gold)">${fmt(tG)} <small style="color:var(--text3)">(${ganhos.length})</small></span></div>
```
`new_string`:
```js
      ${incluiFixos ? `<div class="rm-row"><span>Em caixa (início)</span><span style="color:var(--green)">${fmt(caixaM)}</span></div>` : ''}
      <div class="rm-row"><span>Entradas</span><span style="color:var(--gold)">${fmt(tG)} <small style="color:var(--text3)">(${ganhos.length})</small></span></div>
```

- [ ] **Step 4: Compartilhar resumo (texto do WhatsApp)**

`old_string`:
```js
  const saldo = tG - tD - tF;
```
`new_string`:
```js
  const caixaMes = (p === 'mes') ? getCaixa(meuPerfil) : 0;
  const saldo = caixaMes + tG - tD - tF;
```

- [ ] **Step 5: Snapshot do histórico no fechar mês**

`old_string`:
```js
  const tF  = fixos.filter(f=>f.pago).reduce((a,b)=>a+(b.valor||0),0);
```
`new_string`:
```js
  const tF  = fixos.filter(f=>f.pago).reduce((a,b)=>a+(b.valor||0),0);
  const tCaixa = membros.length
    ? membros.reduce((a,m)=>a+getCaixa(m.id),0)
    : getCaixa(meuPerfil);
```

Depois, no objeto gravado no histórico:

`old_string`:
```js
    totalGanho:tG, totalGastoDiario:tGD, totalFixo:tF,
    totalGasto:tGD+tF, saldo:tG-tGD-tF
```
`new_string`:
```js
    totalGanho:tG, totalGastoDiario:tGD, totalFixo:tF, totalCaixa:tCaixa,
    totalGasto:tGD+tF, saldo:tCaixa+tG-tGD-tF
```

- [ ] **Step 6: Verificação manual no navegador**

Servir o app (Task 6) e abrir logado num grupo de teste. No console, definir um caixa de teste e conferir o saldo:

```js
// pega o id do seu perfil e grava R$ 100 de caixa no mês ativo
setCaixa(meuPerfil, 100);
```
Expected: após o re-render, o **saldo hero** aumenta exatamente R$ 100 em relação ao que era antes (com renda/gastos do mês mantidos). Na aba **Resumo** (modo mês), o saldo também reflete o +100. Sem erros no console.

Limpar o teste depois: `setCaixa(meuPerfil, 0);`

- [ ] **Step 7: Commit**

```bash
cd /c/Users/WSzin/Desktop/financas
git add index.html
git commit -m "feat: somar saldo em caixa no cálculo do saldo (hero, resumo, histórico)

Co-Authored-By: Claude Opus 4.8 <noreply@anthropic.com>"
```

---

## Task 3: Linha "Em caixa" sempre visível no Início + linha no Resumo

**Files:**
- Modify: `index.html` (HTML do saldo-hero, HTML do detalhamento do resumo, função de render do Início)

- [ ] **Step 1: Adicionar a linha no HTML do saldo-hero**

`old_string`:
```html
        <div id="saldo-comparacao" style="display:none;margin-top:8px;font-size:12px;font-weight:600;opacity:0.85"></div>
      </div>
```
`new_string`:
```html
        <div id="saldo-comparacao" style="display:none;margin-top:8px;font-size:12px;font-weight:600;opacity:0.85"></div>
        <div id="caixa-linha" onclick="abrirEditarCaixa()" style="margin-top:12px;display:inline-flex;align-items:center;gap:6px;padding:6px 12px;border-radius:999px;background:rgba(212,175,55,0.12);border:1px solid rgba(212,175,55,0.35);font-size:13px;font-weight:600;color:var(--gold);cursor:pointer">
          💰 Em caixa: <strong id="caixa-valor">R$ 0,00</strong><span id="caixa-edit-ic" style="opacity:0.7">✏️</span>
        </div>
```

- [ ] **Step 2: Adicionar a linha "Em caixa (início)" no detalhamento do Resumo Individual**

`old_string`:
```html
        <div class="card" id="rs-detalhamento-mes">
          <div class="card-title">Detalhamento de gastos</div>
          <div class="balanco-row">
            <span style="color:var(--text2)">Gastos diários</span>
```
`new_string`:
```html
        <div class="card" id="rs-detalhamento-mes">
          <div class="card-title">Detalhamento do mês</div>
          <div class="balanco-row">
            <span style="color:var(--text2)">Em caixa (início do mês)</span>
            <span style="color:var(--green);font-weight:600" id="rs-caixa">R$ 0</span>
          </div>
          <div class="balanco-row">
            <span style="color:var(--text2)">Gastos diários</span>
```

- [ ] **Step 3: Renderizar a linha de caixa no Início (valor + comportamento Todos)**

Localizar, na função de render do Início, o bloco logo após a COMPARAÇÃO MÊS ANTERIOR e antes de `// MINI CARDS`. Inserir o preenchimento da linha.

`old_string`:
```js
  // MINI CARDS
  document.getElementById('meu-ganho').textContent      = fmt(totalGanho);
```
`new_string`:
```js
  // LINHA EM CAIXA (sempre visível)
  const caixaLinhaEl = document.getElementById('caixa-linha');
  const caixaValEl   = document.getElementById('caixa-valor');
  const caixaIcEl    = document.getElementById('caixa-edit-ic');
  if(caixaLinhaEl && caixaValEl){
    caixaValEl.textContent = fmt(caixaMes);
    if(isTodos){
      // visão Todos: mostra a soma, sem edição (edição é individual)
      caixaLinhaEl.style.cursor = 'default';
      if(caixaIcEl) caixaIcEl.style.display = 'none';
    } else {
      caixaLinhaEl.style.cursor = 'pointer';
      if(caixaIcEl) caixaIcEl.style.display = 'inline';
    }
    caixaLinhaEl.style.display = 'inline-flex';
  }

  // MINI CARDS
  document.getElementById('meu-ganho').textContent      = fmt(totalGanho);
```

- [ ] **Step 4: Verificação manual no navegador**

Servir o app e abrir logado. Conferir:
1. Na tela **Início** aparece a pílula "💰 Em caixa: R$ 0,00 ✏️" abaixo do saldo, sempre (mesmo em zero).
2. No modo grupo, ao tocar na aba **Todos**, a pílula aparece sem o lápis ✏️ e mostra a soma dos caixas.
3. Na aba **Resumo** (modo mês) aparece a linha "Em caixa (início do mês)".

(O toque pra editar ainda não faz nada — é a Task 4.)

- [ ] **Step 5: Commit**

```bash
cd /c/Users/WSzin/Desktop/financas
git add index.html
git commit -m "feat: linha Em caixa sempre visível no Início e no Resumo

Co-Authored-By: Claude Opus 4.8 <noreply@anthropic.com>"
```

---

## Task 4: Modal de edição do caixa (com numpad)

**Files:**
- Modify: `index.html` (novo modal HTML + estado/funções JS, ao lado do modal de fixo)

- [ ] **Step 1: Adicionar o modal HTML**

Localizar o início do modal de fixo e inserir o modal de caixa ANTES dele.

`old_string`:
```html
<div class="modal-overlay" id="modal-fixo" onclick="if(event.target===this)fecharModalFixo()">
```
`new_string`:
```html
<div class="modal-overlay" id="modal-caixa" onclick="if(event.target===this)fecharModalCaixa()">
  <div class="modal-card">
    <div class="modal-handle"></div>
    <div class="modal-titulo" style="color:var(--gold)">💰 Valor em caixa</div>
    <div class="modal-sub" id="modal-caixa-sub">Quanto você tinha guardado no começo do mês</div>

    <div class="numpad-display">R$ <span id="numpad-caixa-val">0,00</span></div>
    <input type="hidden" id="modal-caixa-valor"/>

    <div class="numpad-grid">
      <button class="numpad-key" onclick="numpadCaixaDigit('1')">1</button>
      <button class="numpad-key" onclick="numpadCaixaDigit('2')">2</button>
      <button class="numpad-key" onclick="numpadCaixaDigit('3')">3</button>
      <button class="numpad-key" onclick="numpadCaixaDigit('4')">4</button>
      <button class="numpad-key" onclick="numpadCaixaDigit('5')">5</button>
      <button class="numpad-key" onclick="numpadCaixaDigit('6')">6</button>
      <button class="numpad-key" onclick="numpadCaixaDigit('7')">7</button>
      <button class="numpad-key" onclick="numpadCaixaDigit('8')">8</button>
      <button class="numpad-key" onclick="numpadCaixaDigit('9')">9</button>
      <button class="numpad-key numpad-back" onclick="numpadCaixaBack()">⌫</button>
      <button class="numpad-key" onclick="numpadCaixaDigit('0')">0</button>
      <button class="numpad-key numpad-clear" onclick="numpadCaixaClear()">C</button>
    </div>

    <div class="modal-actions">
      <button class="modal-btn modal-btn-cancel" onclick="fecharModalCaixa()">Cancelar</button>
      <button class="modal-btn" style="background:linear-gradient(135deg,#f0c84a,#d4af37);color:#0a0800" onclick="confirmarCaixa()">Salvar</button>
    </div>
  </div>
</div>

<div class="modal-overlay" id="modal-fixo" onclick="if(event.target===this)fecharModalFixo()">
```

- [ ] **Step 2: Adicionar o numpad + funções de abrir/fechar/confirmar**

Localizar o fim do bloco de numpad do fixo (a função `numpadFixoSet`). Inserir o bloco do caixa logo após ela.

`old_string`:
```js
function numpadFixoSet(valor){
  _numpadFixoCents = Math.round((valor||0) * 100);
  numpadFixoSync();
}
```
`new_string`:
```js
function numpadFixoSet(valor){
  _numpadFixoCents = Math.round((valor||0) * 100);
  numpadFixoSync();
}

// ── NUMPAD + MODAL DO CAIXA ────────────────────────────────────────────────
let _numpadCaixaCents = 0;
let _caixaEditPid = null;
function numpadCaixaSync(){
  const reais = _numpadCaixaCents / 100;
  document.getElementById('numpad-caixa-val').textContent =
    reais.toLocaleString('pt-BR',{minimumFractionDigits:2,maximumFractionDigits:2});
  document.getElementById('modal-caixa-valor').value = reais;
}
window.numpadCaixaDigit = function(d){
  if(_numpadCaixaCents >= 999999999) return;
  _numpadCaixaCents = _numpadCaixaCents * 10 + parseInt(d);
  numpadCaixaSync();
}
window.numpadCaixaBack  = function(){ _numpadCaixaCents = Math.floor(_numpadCaixaCents/10); numpadCaixaSync(); }
window.numpadCaixaClear = function(){ _numpadCaixaCents = 0; numpadCaixaSync(); }

window.abrirEditarCaixa = function(){
  const target = membroAtivo || meuPerfil;
  if(target === 'todos'){
    showToast('Selecione a pessoa pra editar o caixa','error');
    return;
  }
  _caixaEditPid = target;
  const isEu = (target === meuPerfil);
  const nome = isEu ? 'você' : ((membros.find(m=>m.id===target)||{}).nome || 'a pessoa');
  document.getElementById('modal-caixa-sub').textContent =
    `Quanto ${nome} tinha guardado no começo de ${getMesAtivo()}`;
  _numpadCaixaCents = Math.round(getCaixa(target) * 100);
  numpadCaixaSync();
  document.getElementById('modal-caixa').classList.add('show');
}
window.fecharModalCaixa = function(){ _caixaEditPid = null; _fecharOverlay('modal-caixa'); }
window.confirmarCaixa = async function(){
  const val = parseFloat(document.getElementById('modal-caixa-valor').value) || 0;
  const pid = _caixaEditPid || meuPerfil;
  await setCaixa(pid, val);
  fecharModalCaixa();
  showToast('Valor em caixa salvo!');
}
```

- [ ] **Step 3: Verificação manual no navegador**

Servir o app e abrir logado:
1. Tocar na pílula "💰 Em caixa" → abre o modal com o numpad.
2. Digitar `150000` (vira R$ 1.500,00 no display), tocar **Salvar**.
3. Modal fecha, toast "Valor em caixa salvo!", e a pílula agora mostra **R$ 1.500,00**; o saldo hero subiu R$ 1.500.
4. Reabrir o modal → ele já vem preenchido com 1.500,00 (lê o valor salvo).
5. Recarregar a página → o valor persiste (veio do Firebase).
6. No modo grupo, selecionar **outra pessoa** e editar o caixa dela funciona de forma independente.

- [ ] **Step 4: Commit**

```bash
cd /c/Users/WSzin/Desktop/financas
git add index.html
git commit -m "feat: modal de edição do valor em caixa com numpad

Co-Authored-By: Claude Opus 4.8 <noreply@anthropic.com>"
```

---

## Task 5: Carry-over automático ao fechar o mês

**Files:**
- Modify: `index.html` (`fecharMes`)

- [ ] **Step 1: Gravar o saldo final de cada membro como caixa do mês seguinte**

Localizar, em `fecharMes`, a linha do comentário sobre as contas fixas (logo antes do `showToast`). Inserir o carry-over ANTES dela.

`old_string`:
```js
  // Contas fixas NÃO são apagadas: servem de molde pra recorrência do próximo mês.
  showToast('Mês fechado! Histórico salvo.');
```
`new_string`:
```js
  // Contas fixas NÃO são apagadas: servem de molde pra recorrência do próximo mês.

  // Carry-over: o saldo final de cada pessoa vira o caixa inicial do mês seguinte.
  const proxSlug = mesSeguinteSlug(mesLabel);
  if(proxSlug){
    const alvos = membros.length ? membros.map(m=>m.id) : [meuPerfil];
    for(const pid of alvos){
      const rendaP = ganhos.filter(g=>g.pid===pid).reduce((a,b)=>a+(b.valor||0),0);
      const gastoP = gastos.filter(g=>g.pid===pid).reduce((a,b)=>a+(b.valor||0),0);
      const fixoP  = fixos.filter(f=>f.pid===pid && f.pago).reduce((a,b)=>a+(b.valor||0),0);
      const saldoFinalP = getCaixa(pid) + rendaP - gastoP - fixoP;
      await setCaixa(pid, saldoFinalP, proxSlug);
    }
  }

  showToast('Mês fechado! Histórico salvo.');
```

- [ ] **Step 2: Verificação manual no navegador (grupo de teste)**

Servir o app e abrir num grupo **de teste** (não use os dados reais ainda):
1. No mês ativo, defina caixa do seu perfil = R$ 100 (pela pílula).
2. Lance uma entrada de R$ 50 e um gasto de R$ 20.
   - Saldo esperado do mês: 100 + 50 − 20 = **R$ 130**.
3. Vá no Resumo → **Fechar mês e salvar histórico** → confirmar.
4. Avance pro mês seguinte (seta › no topo).
   - A pílula "Em caixa" do mês seguinte deve mostrar **R$ 130** (o saldo final virou caixa).
5. Confira no Histórico que o snapshot do mês fechado tem saldo R$ 130.

- [ ] **Step 3: Commit**

```bash
cd /c/Users/WSzin/Desktop/financas
git add index.html
git commit -m "feat: carry-over do saldo final pro caixa do mês seguinte ao fechar mês

Co-Authored-By: Claude Opus 4.8 <noreply@anthropic.com>"
```

---

## Task 6: Limpeza, verificação final e deploy

**Files:**
- Delete: `_test_caixa.mjs`
- Deploy: push pra `main` (GitHub Pages)

- [ ] **Step 1: Remover o arquivo de teste temporário**

```bash
cd /c/Users/WSzin/Desktop/financas
rm -f _test_caixa.mjs
```

- [ ] **Step 2: Smoke test final no navegador**

Servir localmente e validar o fluxo inteiro de uma vez:

```bash
cd /c/Users/WSzin/Desktop/financas
python -m http.server 8000
```
Abrir `http://localhost:8000` (ou só abrir o `index.html` no navegador). Conferir, sem erros no console:
1. Pílula "Em caixa" aparece sempre no Início.
2. Editar caixa → saldo atualiza e persiste após reload.
3. Resumo mostra a linha "Em caixa (início do mês)" e o saldo bate.
4. Modo Individual (solo) também funciona (pílula edita o próprio caixa).

Parar o servidor com Ctrl+C.

- [ ] **Step 3: Commit da limpeza + push pro deploy**

```bash
cd /c/Users/WSzin/Desktop/financas
git add -A
git commit -m "chore: remove teste temporário do saldo em caixa

Co-Authored-By: Claude Opus 4.8 <noreply@anthropic.com>"
git push origin main
```

- [ ] **Step 4: Atualizar a memória do projeto**

Adicionar um bullet em `C:\Users\WSzin\.claude\projects\C--Users-WSzin-Desktop-financas\memory\projeto_grana_pro.md` na seção "Features já implementadas no main":
> **Saldo em caixa** — valor inicial por pessoa por mês (`caixa/{mesSlug}/{pid}`); entra no saldo (`saldo = caixa + renda − gastos − fixos pagos`); linha "Em caixa" sempre visível no Início, editável por modal/numpad; carry-over automático no `fecharMes` (saldo final → caixa do mês seguinte).

---

## Self-Review (preenchido)

**1. Cobertura da spec:**
- Caixa por pessoa por mês em `caixa/{mesSlug}/{pid}` → Task 1 (helpers + listener). ✅
- Fórmula `saldo = caixa + renda − gastos − fixos pagos` nos 4 pontos + histórico → Task 2. ✅
- Linha "Em caixa" sempre visível, editável, soma+sem-edição em Todos → Tasks 3 e 4. ✅
- Carry-over automático no fechar mês → Task 5. ✅
- Sempre visível (mesmo em zero) → Task 3 Step 3 (display sempre). ✅
- Compatibilidade com dados antigos (ausente = 0) → `getCaixa` retorna 0. ✅

**2. Placeholders:** nenhum "TBD"/"TODO"; todo passo de código mostra o código completo. ✅

**3. Consistência de tipos/nomes:** `getCaixa(pid, mesSlug?)`, `setCaixa(pid, valor, mesSlug?)`, `mesSlugDe`, `mesSeguinteSlug`, `_dados.caixa`, ids `caixa-linha`/`caixa-valor`/`caixa-edit-ic`/`rs-caixa`/`modal-caixa`/`numpad-caixa-val`/`modal-caixa-valor`, estado `_numpadCaixaCents`/`_caixaEditPid` — usados de forma idêntica entre tasks. ✅
