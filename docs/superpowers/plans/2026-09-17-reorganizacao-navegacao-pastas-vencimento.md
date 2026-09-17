# Reorganização em Abas (Visão Geral / Pastas / Vencimento) Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Reorganizar `index.html` (hoje uma única tela empilhada) em 3 abas navegáveis por hash (Visão Geral, Pastas, Vencimento) mais um modal flutuante de cadastro, sem tocar em nada do Firestore.

**Architecture:** Continua sendo um único arquivo `index.html` estático, sem build/framework. Um roteador simples baseado em `location.hash` decide qual `<div class="view-secao">` fica visível e chama a função de renderização correspondente. Todas as funções de renderização leem do mesmo `window.registrosGlobais` que já existe (populado pelo `onSnapshot` do Firestore) — nenhum dado novo é lido/gravado, tudo é derivado em memória no cliente.

**Tech Stack:** HTML + CSS + JS vanilla (ES modules), Firebase Firestore modular SDK (já em uso), Chart.js (já em uso). Nenhuma dependência nova.

**Spec:** `docs/superpowers/specs/2026-09-17-reorganizacao-navegacao-pastas-vencimento-design.md`

## Global Constraints

- Nenhuma mudança de schema/leitura/escrita no Firestore. Campos do documento continuam: `empresa, placa, motorista, dataServico, tipoServico, prazoDias, descricao, proximaLubrificacao, proximaTrocaOleo, valor, statusPagamento, comprovanteUrl, comprovanteNome, criadoEm`.
- Nenhum framework, bundler ou dependência nova. Um único arquivo `index.html`.
- Todo texto vindo de dados do usuário (empresa, placa, motorista, descrição, tipo, nome de comprovante) que for inserido via `innerHTML` deve passar por `escapeHtml()` (função já existente no arquivo).
- Modal de cadastro é um `<div>` overlay fixo — **não** usar `<dialog>` nativo (motivo: fecha sozinho com ESC/gesto de voltar em mobile e perderia dados digitados).
- Gráficos de uma "pasta" reaproveitam **dois canvases fixos únicos** (nunca criar canvas novo por empresa) para não disparar o erro "Canvas is already in use" do Chart.js.
- Botão de cobrança do WhatsApp usa `https://wa.me/?text=...` sem número de telefone — o usuário escolhe o contato manualmente.
- Comparação/agrupamento por nome de empresa deve ser tolerante a espaço/maiúscula: normalizar com `.trim().toLowerCase()` para comparar, mas exibir sempre o texto original (`.trim()`, sem lowercase).
- Sem framework de testes automatizados neste projeto (nunca existiu). **Convenção de verificação desta plan:** abrir `index.html` na aba do navegador embutido (`mcp__Claude_Browser__*`, sessão já autenticada com dados reais), rodar checagens via `javascript_tool` (assertions em `window.registrosGlobais` e no DOM), tirar screenshot quando for uma checagem visual, e ler `read_console_messages(onlyErrors: true)` para garantir que não apareceu nenhum erro nunca visto antes. Funções puras (sem DOM) são verificadas com dados sintéticos direto no console do navegador, do mesmo jeito que já foi validado para `calcularUltimosVencimentosPorPlaca` nesta conversa.
- Cada task termina com commit próprio (mensagens em português, sem `--no-verify`).

---

### Task 1: Casca de navegação (abas + roteador), sem mudar o que já funciona

**Files:**
- Modify: `index.html` — bloco `<style>` (adicionar CSS de abas), bloco `<div id="appConteudo">` (envolver conteúdo existente), segundo `<script type="module">` (adicionar roteador), primeiro `<script type="module">` (redirecionar o hook do Firestore).

**Interfaces:**
- Produces: `window.renderizarViewAtual()` — função sem parâmetros, decide a aba ativa pelo `location.hash` e chama a função de renderização daquela aba. É isso que as próximas tasks vão popular por dentro.
- Produces: `window.renderizarTabelaCompleta` continua existindo exatamente como hoje (nenhuma mudança nela nesta task).

- [ ] **Step 1: Adicionar CSS das abas e das seções de view**

Em `index.html`, localizar a última regra do bloco `<style>` (procurar pelo texto exato `.header-row { display: flex; align-items: center; width: 100%; gap: 12px; }`) e inserir logo depois, ainda dentro do `<style>`:

```css
        .tabs-principais {
            display: flex;
            gap: 6px;
            margin-bottom: 12px;
            border-bottom: 1px solid var(--border);
        }
        .tab-link {
            padding: 9px 16px;
            font-size: 0.82rem;
            font-weight: 600;
            font-family: 'Oswald', sans-serif;
            text-transform: uppercase;
            letter-spacing: 0.5px;
            color: var(--text-muted);
            text-decoration: none;
            border-bottom: 2px solid transparent;
            cursor: pointer;
        }
        .tab-link:hover { color: var(--text); }
        .tab-link.ativo {
            color: var(--primary);
            border-bottom-color: var(--primary);
        }
        .view-secao { display: none; }
        .view-secao.ativa { display: block; }
        .view-placeholder {
            background: var(--card-bg);
            border: 1px dashed var(--border);
            border-radius: 6px;
            padding: 24px;
            text-align: center;
            color: var(--text-muted);
            font-size: 0.85rem;
        }
```

- [ ] **Step 2: Envolver o conteúdo existente em `#viewVisaoGeral` e criar as abas vazias**

Localizar o trecho exato (logo após o cabeçalho):

```html
    <div class="hazard-divider"></div>

    <div class="agenda-alertas">
```

Substituir por:

```html
    <div class="hazard-divider"></div>

    <nav class="tabs-principais">
        <a href="#visao-geral" class="tab-link" data-tab="visao-geral">Visão Geral</a>
        <a href="#pastas" class="tab-link" data-tab="pastas">Pastas</a>
        <a href="#vencimento" class="tab-link" data-tab="vencimento">Vencimento</a>
    </nav>

    <div id="viewVisaoGeral" class="view-secao">
    <div class="agenda-alertas">
```

Localizar o trecho exato mais abaixo (fecha a tabela geral, antes do toast):

```html
    <div class="table-container">
        <table>
            <thead>
                <tr>
                    <th>Empresa</th><th>Placa</th><th>Motorista</th><th>Data</th><th>Tipo</th>
                    <th>Descrição</th><th>Próx. Lub.</th><th>Próx. Óleo</th><th>Valor</th>
                    <th>Status</th><th>Comp.</th><th>Ações</th>
                </tr>
            </thead>
            <tbody id="tabelaCorpo"></tbody>
        </table>
    </div>

    <div id="toastMsg" class="toast"></div>
```

Substituir por (fecha `viewVisaoGeral` e adiciona as duas abas novas, ainda vazias/placeholder):

```html
    <div class="table-container">
        <table>
            <thead>
                <tr>
                    <th>Empresa</th><th>Placa</th><th>Motorista</th><th>Data</th><th>Tipo</th>
                    <th>Descrição</th><th>Próx. Lub.</th><th>Próx. Óleo</th><th>Valor</th>
                    <th>Status</th><th>Comp.</th><th>Ações</th>
                </tr>
            </thead>
            <tbody id="tabelaCorpo"></tbody>
        </table>
    </div>
    </div><!-- /viewVisaoGeral -->

    <div id="viewPastas" class="view-secao">
        <div class="view-placeholder">Em construção — chega na próxima etapa.</div>
    </div>

    <div id="viewVencimento" class="view-secao">
        <div class="view-placeholder">Em construção — chega em breve.</div>
    </div>

    <div id="toastMsg" class="toast"></div>
```

- [ ] **Step 3: Adicionar o roteador no segundo `<script type="module">`**

Localizar o texto exato:

```js
    function formatarData(dataISO) {
        if (!dataISO) return '—';
        const [ano, mes, dia] = dataISO.split('-');
        return `${dia}/${mes}/${ano}`;
    }
```

Adicionar logo depois (mesma indentação):

```js

    const VIEWS_VALIDAS = ['visao-geral', 'pastas', 'vencimento'];

    function obterViewAtualDoHash() {
        const hash = (location.hash || '').replace('#', '');
        if (hash.startsWith('pasta/')) return 'pastas';
        return VIEWS_VALIDAS.includes(hash) ? hash : 'visao-geral';
    }

    window.renderizarViewAtual = function() {
        const viewAtiva = obterViewAtualDoHash();

        document.querySelectorAll('.view-secao').forEach(el => el.classList.remove('ativa'));
        const elIds = { 'visao-geral': 'viewVisaoGeral', 'pastas': 'viewPastas', 'vencimento': 'viewVencimento' };
        document.getElementById(elIds[viewAtiva]).classList.add('ativa');

        document.querySelectorAll('.tab-link').forEach(a => {
            a.classList.toggle('ativo', a.dataset.tab === viewAtiva);
        });

        if (viewAtiva === 'visao-geral' && typeof window.renderizarTabelaCompleta === 'function') {
            window.renderizarTabelaCompleta();
        }
        if (viewAtiva === 'pastas' && typeof window.renderizarListaPastas === 'function') {
            window.renderizarListaPastas();
        }
        if (viewAtiva === 'vencimento' && typeof window.renderizarVencimento === 'function') {
            window.renderizarVencimento();
        }
    }

    window.addEventListener('hashchange', window.renderizarViewAtual);
```

- [ ] **Step 4: Redirecionar o hook do Firestore pro roteador**

No **primeiro** `<script type="module">` (bem no topo do arquivo, perto da linha 44), localizar o texto exato:

```js
                // Quando os dados chegarem ou mudarem, redesenha a tela
                if (typeof window.renderizarTabelaCompleta === 'function') {
                    window.renderizarTabelaCompleta();
                }
```

Substituir por:

```js
                // Quando os dados chegarem ou mudarem, redesenha a view que estiver ativa
                if (typeof window.renderizarViewAtual === 'function') {
                    window.renderizarViewAtual();
                }
```

- [ ] **Step 5: Chamar o roteador uma vez no carregamento inicial**

No segundo `<script type="module">`, localizar o texto exato (perto do fim do arquivo):

```js
    window.filtrarDados = function() {
        window.renderizarTabelaCompleta();
    }
```

Deixar como está (sem mudança — é usado pelos filtros de busca da Visão Geral hoje), mas logo abaixo do bloco do `window.renderizarViewAtual`/`hashchange` do Step 3, adicionar a chamada inicial:

```js
    window.renderizarViewAtual();
```

(Isso garante que, assim que o script carrega, a aba correta já aparece — sem depender só do evento `hashchange`, que não dispara sozinho no primeiro load.)

- [ ] **Step 6: Verificar no navegador (regressão)**

Usar `mcp__Claude_Browser__navigate` para abrir `file:///C:/Users/Usuario/Desktop/Site%20Magr%C3%A3o/index.html` (sessão já logada), depois:

1. `read_console_messages(onlyErrors: true)` → deve vir vazio.
2. Screenshot → confirmar que a aba "Visão Geral" aparece ativa (laranja) e mostra exatamente o que mostrava antes (cards de alerta, resumo financeiro, gráficos, formulário, filtros, tabela) — nada sumiu.
3. Clicar em "Pastas" → mostra o texto placeholder, aba "Pastas" fica destacada.
4. Clicar em "Vencimento" → mostra o texto placeholder.
5. Clicar em "Visão Geral" de novo → volta a mostrar tudo certinho, sem erro no console.
6. Testar `navigate` direto pra URL com `#pastas` no final → já abre com a aba Pastas ativa (confirma que o Step 5 funcionou).

- [ ] **Step 7: Commit**

```bash
git add index.html
git commit -m "$(cat <<'EOF'
adiciona casca de navegacao por abas (Visao Geral / Pastas / Vencimento)

Introduz roteador simples baseado em location.hash. Visao Geral continua
identica a hoje; Pastas e Vencimento ainda sao placeholders, preenchidos
nas proximas etapas. Zero mudanca no Firestore.

Co-Authored-By: Claude Sonnet 5 <noreply@anthropic.com>
EOF
)"
```

---

### Task 2: Aba "Pastas" — lista de empresas com bolinha de status + detalhe da pasta

**Files:**
- Modify: `index.html` — CSS, move `.filtros-pastas-horizontal` e `.table-container` para dentro de `#viewPastas`, JS (nova função `calcularStatusPorEmpresa`, `renderizarListaPastas`, `renderizarDetalhePasta`, remove `renderizarSidebarPastas` e a montagem de linhas da tabela geral de dentro de `renderizarTabelaCompleta`).

**Interfaces:**
- Consumes: `calcularUltimosVencimentosPorPlaca(dados)` (Task já existente antes deste plano) — retorna `{ [placa]: { lub: registroOuNull, oleo: registroOuNull } }`.
- Consumes: `escapeHtml(texto)`, `formatarData(dataISO)` (já existentes).
- Produces: `function calcularStatusPorEmpresa(dados)` → retorna `{ [empresaNormalizada]: 'vencido' | 'urgente' | 'ok' }`, chave em `trim().toLowerCase()` (quem for consumir precisa normalizar a chave do mesmo jeito antes de buscar).
- Produces: `window.renderizarListaPastas()` — sem parâmetros, popula `#gridPastas`.
- Produces: `window.renderizarDetalhePasta(empresaCodificada)` — recebe o segmento do hash já como veio (ainda codificado), decodifica internamente.
- Produces: `window.abrirPasta(nomeEmpresa)` — helper que seta `location.hash = '#pasta/' + encodeURIComponent(nomeEmpresa)`.

- [ ] **Step 1: CSS dos cards de pasta e da bolinha de status**

Inserir, no mesmo lugar do CSS do Task 1 (depois do bloco `.view-placeholder { ... }`):

```css
        .grid-pastas {
            display: grid;
            grid-template-columns: repeat(auto-fill, minmax(220px, 1fr));
            gap: 10px;
            margin-top: 12px;
        }
        .pasta-card {
            background: var(--card-bg);
            border: 1px solid var(--border);
            border-radius: 6px;
            padding: 14px 16px;
            cursor: pointer;
            display: flex;
            align-items: center;
            justify-content: space-between;
            gap: 8px;
            transition: border-color 0.15s;
        }
        .pasta-card:hover { border-color: var(--primary); }
        .pasta-card .status-dot {
            width: 10px;
            height: 10px;
            border-radius: 50%;
            flex-shrink: 0;
        }
        .status-dot.vencido { background: var(--warning); }
        .status-dot.urgente { background: var(--pending); }
        .status-dot.ok { background: transparent; }
        .btn-voltar-pasta {
            background: transparent;
            border: 1px solid var(--border);
            color: var(--text-muted);
            margin-bottom: 12px;
        }
        .btn-voltar-pasta:hover { border-color: var(--primary); color: var(--primary); }
        .resumo-pasta {
            display: flex;
            gap: 16px;
            flex-wrap: wrap;
            margin-bottom: 12px;
        }
        .resumo-pasta .finance-item { font-size: 0.85rem; }
```

- [ ] **Step 2: Mover `.filtros-pastas-horizontal` e `.table-container` para dentro de `#viewPastas`**

Localizar o texto exato (que hoje está dentro de `#viewVisaoGeral`, depois do formulário de cadastro):

```html
    <div class="filtros-pastas-horizontal">
        <div class="filtros-top-row">
            <span style="font-size: 0.85rem; font-weight: bold;">🔍 Filtros / Pastas:</span>
            <input type="text" id="filterEmpresa" class="search-input-compact" placeholder="Buscar Empresa..." oninput="filtrarPastasEEstados()">
            <input type="text" id="filterPlaca" class="search-input-compact" placeholder="Buscar Placa..." oninput="filtrarPastasEEstados()">
            <select id="filterStatus" class="select-periodo" style="height: 30px;" onchange="filtrarDados()">
                <option value="TODOS">Todos os Status</option>
                <option value="Pendente">Apenas Pendentes</option>
                <option value="Pago">Apenas Pagos</option>
            </select>
            <button class="btn-limpar-chip" onclick="limparFiltroLateral()">Limpar Filtros</button>
        </div>
        <div class="pastas-chips-container" id="pastasListaContainer"></div>
    </div>

    <div class="table-container">
        <table>
            <thead>
                <tr>
                    <th>Empresa</th><th>Placa</th><th>Motorista</th><th>Data</th><th>Tipo</th>
                    <th>Descrição</th><th>Próx. Lub.</th><th>Próx. Óleo</th><th>Valor</th>
                    <th>Status</th><th>Comp.</th><th>Ações</th>
                </tr>
            </thead>
            <tbody id="tabelaCorpo"></tbody>
        </table>
    </div>
    </div><!-- /viewVisaoGeral -->

    <div id="viewPastas" class="view-secao">
        <div class="view-placeholder">Em construção — chega na próxima etapa.</div>
    </div>
```

Substituir por (remove o bloco de dentro da Visão Geral, remove o velho container de chips, monta a Pastas de verdade com sub-view de lista e sub-view de detalhe):

```html
    </div><!-- /viewVisaoGeral -->

    <div id="viewPastas" class="view-secao">
        <div id="pastasSubviewLista">
            <div class="filtros-pastas-horizontal">
                <div class="filtros-top-row">
                    <span style="font-size: 0.85rem; font-weight: bold;">🔍 Buscar Pasta:</span>
                    <input type="text" id="filterEmpresa" class="search-input-compact" placeholder="Buscar Empresa..." oninput="window.renderizarListaPastas()">
                    <input type="text" id="filterPlaca" class="search-input-compact" placeholder="Buscar Placa..." oninput="window.renderizarListaPastas()">
                    <select id="filterStatus" class="select-periodo" style="height: 30px;" onchange="window.renderizarListaPastas()">
                        <option value="TODOS">Todos os Status</option>
                        <option value="Pendente">Apenas Pendentes</option>
                        <option value="Pago">Apenas Pagos</option>
                    </select>
                    <button class="btn-limpar-chip" onclick="limparFiltroLateral()">Limpar Filtros</button>
                </div>
            </div>
            <div class="grid-pastas" id="gridPastas"></div>
        </div>

        <div id="pastasSubviewDetalhe" style="display: none;">
            <button type="button" class="btn-voltar-pasta" onclick="location.hash = '#pastas'">◀ Voltar pra lista de pastas</button>
            <div id="conteudoDetalhePasta"></div>
            <div class="table-container">
                <table>
                    <thead>
                        <tr>
                            <th>Placa</th><th>Motorista</th><th>Data</th><th>Tipo</th>
                            <th>Descrição</th><th>Próx. Lub.</th><th>Próx. Óleo</th><th>Valor</th>
                            <th>Status</th><th>Comp.</th><th>Ações</th>
                        </tr>
                    </thead>
                    <tbody id="tabelaCorpoPasta"></tbody>
                </table>
            </div>
        </div>
    </div>
```

- [ ] **Step 3: Remover `limparFiltroLateral` do jeito antigo e adaptar pro novo fluxo**

Localizar o texto exato:

```js
    window.limparFiltroLateral = function() {
        document.getElementById('filterEmpresa').value = '';
        document.getElementById('filterPlaca').value = '';
        document.getElementById('filterStatus').value = 'TODOS';
        renderizarSidebarPastas();
        window.filtrarDados();
    }
```

Substituir por:

```js
    window.limparFiltroLateral = function() {
        document.getElementById('filterEmpresa').value = '';
        document.getElementById('filterPlaca').value = '';
        document.getElementById('filterStatus').value = 'TODOS';
        window.renderizarListaPastas();
    }
```

- [ ] **Step 4: Remover a função antiga `renderizarSidebarPastas` e sua chamada**

Localizar o texto exato:

```js
    function renderizarSidebarPastas() {
        const container = document.getElementById('pastasListaContainer');
        container.innerHTML = '';
        const empresasSet = new Set();
        window.registrosGlobais.forEach(reg => { if (reg.empresa) empresasSet.add(reg.empresa.trim()); });

        if (empresasSet.size === 0) {
            container.innerHTML = '<span style="color: var(--text-muted); font-size: 0.75rem;">Nenhuma pasta encontrada.</span>';
            return;
        }

        const termoBuscaEmpresa = document.getElementById('filterEmpresa').value.toLowerCase().trim();
        Array.from(empresasSet).sort().forEach(empresaNome => {
            if (termoBuscaEmpresa !== "" && !empresaNome.toLowerCase().includes(termoBuscaEmpresa)) return;
            const chip = document.createElement('div');
            chip.className = 'pasta-chip';
            chip.innerHTML = `📁 ${escapeHtml(empresaNome)}`;
            chip.onclick = () => {
                document.getElementById('filterEmpresa').value = empresaNome;
                window.filtrarDados();
            };
            container.appendChild(chip);
        });
    }
```

Apagar essa função inteira (será substituída por `renderizarListaPastas` no Step 6).

Agora localizar, dentro de `window.renderizarTabelaCompleta`, o texto exato:

```js
        atualizarPainelAvisos(vencidosList, urgentesList);
        renderizarSidebarPastas();
        atualizarGraficos(lucroMesSelecionado, pendenteMesSelecionado, tiposContagemMes);
```

Substituir por (remove a chamada pra função que acabou de ser apagada):

```js
        atualizarPainelAvisos(vencidosList, urgentesList);
        atualizarGraficos(lucroMesSelecionado, pendenteMesSelecionado, tiposContagemMes);
```

- [ ] **Step 5: Remover a montagem da tabela geral de dentro de `renderizarTabelaCompleta`**

Localizar o texto exato (bloco grande — do início do `forEach` de linhas até o fechamento):

```js
        const filterEmp = document.getElementById('filterEmpresa').value.toLowerCase();
        const filterPlac = document.getElementById('filterPlaca').value.toLowerCase();
        const filterStatus = document.getElementById('filterStatus').value;

        const dadosParaExibir = dados.filter(reg => {
            const bateEmpresa = (reg.empresa || '').toLowerCase().includes(filterEmp);
            const batePlaca = (reg.placa || '').toLowerCase().includes(filterPlac);
            const bateStatus = filterStatus === "TODOS" || reg.statusPagamento === filterStatus;
            return bateEmpresa && batePlaca && bateStatus;
        });

        // Vencimento "vigente" de cada placa/tipo (sempre baseado em TODOS os registros,
        // não só nos filtrados, para o painel de alertas não sumir quando um filtro é aplicado)
        const ultimosVencimentos = calcularUltimosVencimentosPorPlaca(dados);

        Object.keys(ultimosVencimentos).forEach(placaKey => {
            const entry = ultimosVencimentos[placaKey];

            if (entry.lub) {
                const [anoL, mesL, diaL] = entry.lub.proximaLubrificacao.split('-');
                const dataLub = new Date(anoL, mesL - 1, diaL);
                const diff = Math.round((dataLub - hojePadrao) / (1000 * 60 * 60 * 24));
                if (diff < 0) {
                    vencidosList.push({ idDoc: entry.lub.idDoc, placa: entry.lub.placa, empresa: entry.lub.empresa, tipo: 'Lubrificação', data: entry.lub.proximaLubrificacao, dias: Math.abs(diff) });
                } else if (diff <= 5) {
                    urgentesList.push({ idDoc: entry.lub.idDoc, placa: entry.lub.placa, empresa: entry.lub.empresa, tipo: 'Lubrificação', data: entry.lub.proximaLubrificacao, dias: diff });
                }
            }

            if (entry.oleo) {
                const [anoO, mesO, diaO] = entry.oleo.proximaTrocaOleo.split('-');
                const dataOleo = new Date(anoO, mesO - 1, diaO);
                const diff = Math.round((dataOleo - hojePadrao) / (1000 * 60 * 60 * 24));
                if (diff < 0) {
                    vencidosList.push({ idDoc: entry.oleo.idDoc, placa: entry.oleo.placa, empresa: entry.oleo.empresa, tipo: 'Troca de Óleo', data: entry.oleo.proximaTrocaOleo, dias: Math.abs(diff) });
                } else if (diff <= 5) {
                    urgentesList.push({ idDoc: entry.oleo.idDoc, placa: entry.oleo.placa, empresa: entry.oleo.empresa, tipo: 'Troca de Óleo', data: entry.oleo.proximaTrocaOleo, dias: diff });
                }
            }
        });
```

manter esse trecho acima **exatamente como está** (ele continua alimentando o painel de avisos da Visão Geral até a Task 5). O que muda é **só** o que vem depois dele. Localizar o texto exato logo em seguida:

```js
        dadosParaExibir.forEach(reg => {
            const tr = document.createElement('tr');
            let diffLubDias = null;
            let diffOleoDias = null;
            let alertaAtivo = false;

            const entryPlaca = ultimosVencimentos[reg.placa] || {};
            // Só é o vencimento "vigente" se este registro for o mais recente daquele tipo para a placa
            const ehVigenteLub = !!(reg.proximaLubrificacao && entryPlaca.lub && entryPlaca.lub.idDoc === reg.idDoc);
            const ehVigenteOleo = !!(reg.proximaTrocaOleo && entryPlaca.oleo && entryPlaca.oleo.idDoc === reg.idDoc);

            if (reg.proximaLubrificacao) {
                const [anoL, mesL, diaL] = reg.proximaLubrificacao.split('-');
                const dataLub = new Date(anoL, mesL - 1, diaL);
                // Math.round é mais seguro que Math.ceil para contagem exata de dias
                diffLubDias = Math.round((dataLub - hojePadrao) / (1000 * 60 * 60 * 24));

                if (ehVigenteLub && diffLubDias <= 5) alertaAtivo = true;
            }

            if (reg.proximaTrocaOleo) {
                const [anoO, mesO, diaO] = reg.proximaTrocaOleo.split('-');
                const dataOleo = new Date(anoO, mesO - 1, diaO);
                diffOleoDias = Math.round((dataOleo - hojePadrao) / (1000 * 60 * 60 * 24));

                if (ehVigenteOleo && diffOleoDias <= 5) alertaAtivo = true;
            }

            if (alertaAtivo) tr.classList.add('alert-row');

            const dataServicoBR = formatarData(reg.dataServico);
            const proxLubBR = reg.proximaLubrificacao ? formatarData(reg.proximaLubrificacao) : '—';
            const proxOleoBR = reg.proximaTrocaOleo ? formatarData(reg.proximaTrocaOleo) : '—';

            const exibicaoProxLub = (ehVigenteLub && diffLubDias !== null && diffLubDias <= 5) ? `<span class="alert-text">⚠️ ${proxLubBR}</span>` : proxLubBR;
            const exibicaoProxOleo = (ehVigenteOleo && diffOleoDias !== null && diffOleoDias <= 5) ? `<span class="alert-text">⚠️ ${proxOleoBR}</span>` : proxOleoBR;

            // LÓGICA DO BOTÃO DA AGENDA NA TABELA
            let dataAgenda = reg.proximaTrocaOleo || reg.proximaLubrificacao;
            let btnAgendaHtml = '';
            if (dataAgenda) {
                const dataStr = dataAgenda.replace(/-/g, '');
                const titulo = encodeURIComponent(`Revisão/Óleo - ${reg.placa} (${reg.empresa})`);
                const detalhes = encodeURIComponent(`Fazer ${reg.tipoServico || 'serviço'} do caminhão placa ${reg.placa}.`);
                const linkCal = `https://calendar.google.com/calendar/render?action=TEMPLATE&text=${titulo}&dates=${dataStr}/${dataStr}&details=${detalhes}`;
                btnAgendaHtml = `<a href="${linkCal}" target="_blank" class="action-btn" title="Salvar no Google Agenda" style="text-decoration: none; font-size: 1rem;">📅</a>`;
            }

            tr.innerHTML = `
                <td data-label="Empresa" title="${escapeHtml(reg.empresa)}"><strong>${escapeHtml(reg.empresa)}</strong></td>
                <td data-label="Placa">${escapeHtml(reg.placa)}</td>
                <td data-label="Motorista" title="${escapeHtml(reg.motorista)}">${escapeHtml(reg.motorista)}</td>
                <td data-label="Data">${dataServicoBR}</td>
                <td data-label="Tipo">${escapeHtml(reg.tipoServico || 'Lubrificação')}</td>
                <td data-label="Descrição" class="col-descricao" title="${escapeHtml(reg.descricao || '')}">${escapeHtml(reg.descricao) || '—'}</td>
                <td data-label="Próx. Lub.">${exibicaoProxLub}</td>
                <td data-label="Próx. Óleo">${exibicaoProxOleo}</td>
                <td data-label="Valor">R$ ${reg.valor}</td>
                <td data-label="Status"><span class="badge ${reg.statusPagamento === 'Pago' ? 'badge-pago' : 'badge-pendente'}">${reg.statusPagamento}</span></td>
                <td data-label="Comp.">
                    ${reg.comprovanteUrl ? `<a href="${escapeHtml(reg.comprovanteUrl)}" target="_blank" class="comprovante-btn">📄 Ver</a>` : '—'}
                </td>
                <td data-label="Ações">
                    <div class="flex-actions">
                        ${btnAgendaHtml}
                        <button type="button" class="action-btn btn-editar-acao" onclick="editarRegistro('${reg.idDoc}')" title="Editar">📝</button>
                        <button type="button" class="action-btn btn-excluir-acao" onclick="excluirRegistro('${reg.idDoc}')" title="Excluir">🗑️</button>
                    </div>
                </td>
            `;
            corpo.appendChild(tr);
        });

**Apagar** todo o trecho reproduzido acima nesta Step 5 (do `dadosParaExibir.forEach(reg => {` até o `});` logo antes de `document.getElementById('lucroMes')`) — Visão Geral não mostra mais tabela. Também apagar, um pouco mais acima nessa mesma função, estas duas linhas (não usadas mais depois de apagar o forEach):

```js
        const corpo = document.getElementById('tabelaCorpo');
        corpo.innerHTML = '';
```

E apagar a declaração de `dadosParaExibir` (o filtro por `filterEmp`/`filterPlac`/`filterStatus`), já que só era usada pelo forEach que acabou de ser removido:

```js
        const filterEmp = document.getElementById('filterEmpresa').value.toLowerCase();
        const filterPlac = document.getElementById('filterPlaca').value.toLowerCase();
        const filterStatus = document.getElementById('filterStatus').value;

        const dadosParaExibir = dados.filter(reg => {
            const bateEmpresa = (reg.empresa || '').toLowerCase().includes(filterEmp);
            const batePlaca = (reg.placa || '').toLowerCase().includes(filterPlac);
            const bateStatus = filterStatus === "TODOS" || reg.statusPagamento === filterStatus;
            return bateEmpresa && batePlaca && bateStatus;
        });
```

(Esse bloco tinha ficado órfão desde que `filterEmpresa`/`filterPlaca`/`filterStatus` se mudaram pra dentro de `#viewPastas` no Step 2 — a partir de agora esses três inputs são usados só por `renderizarListaPastas`, que criamos no Step 6.)

O que **permanece intacto** dentro de `window.renderizarTabelaCompleta` depois dessas remoções: o cálculo financeiro do mês (primeiro `dados.forEach`), o cálculo de `vencidosList`/`urgentesList` via `calcularUltimosVencimentosPorPlaca` (mantido até a Task 5), e as 3 linhas finais:

```js
        document.getElementById('lucroMes').innerText = `R$ ${lucroMesSelecionado.toFixed(2)}`;
        document.getElementById('qtdServicosMes').innerText = `${qtdServicosMesSelecionado}`;
        document.getElementById('totalPendenteGeral').innerText = `R$ ${totalPendenteGeral.toFixed(2)}`;

        atualizarPainelAvisos(vencidosList, urgentesList);
        atualizarGraficos(lucroMesSelecionado, pendenteMesSelecionado, tiposContagemMes);
    }
```

- [ ] **Step 6: Extrair a função de montar uma linha da tabela (reaproveitada pela Pasta)**

Adicionar, logo depois do fechamento de `window.renderizarTabelaCompleta` (a função `atualizarPainelAvisos` continua logo em seguida, sem mudança nesta task):

```js
    // Monta o <tr> de um registro para a tabela histórica de uma pasta (sem coluna Empresa,
    // já sabemos qual é: estamos dentro da pasta dela).
    function montarLinhaHistoricoPasta(reg, ultimosVencimentos, hojePadrao) {
        let diffLubDias = null;
        let diffOleoDias = null;
        let alertaAtivo = false;

        const entryPlaca = ultimosVencimentos[reg.placa] || {};
        const ehVigenteLub = !!(reg.proximaLubrificacao && entryPlaca.lub && entryPlaca.lub.idDoc === reg.idDoc);
        const ehVigenteOleo = !!(reg.proximaTrocaOleo && entryPlaca.oleo && entryPlaca.oleo.idDoc === reg.idDoc);

        if (reg.proximaLubrificacao) {
            const [anoL, mesL, diaL] = reg.proximaLubrificacao.split('-');
            const dataLub = new Date(anoL, mesL - 1, diaL);
            diffLubDias = Math.round((dataLub - hojePadrao) / (1000 * 60 * 60 * 24));
            if (ehVigenteLub && diffLubDias <= 5) alertaAtivo = true;
        }

        if (reg.proximaTrocaOleo) {
            const [anoO, mesO, diaO] = reg.proximaTrocaOleo.split('-');
            const dataOleo = new Date(anoO, mesO - 1, diaO);
            diffOleoDias = Math.round((dataOleo - hojePadrao) / (1000 * 60 * 60 * 24));
            if (ehVigenteOleo && diffOleoDias <= 5) alertaAtivo = true;
        }

        const dataServicoBR = formatarData(reg.dataServico);
        const proxLubBR = reg.proximaLubrificacao ? formatarData(reg.proximaLubrificacao) : '—';
        const proxOleoBR = reg.proximaTrocaOleo ? formatarData(reg.proximaTrocaOleo) : '—';
        const exibicaoProxLub = (ehVigenteLub && diffLubDias !== null && diffLubDias <= 5) ? `<span class="alert-text">⚠️ ${proxLubBR}</span>` : proxLubBR;
        const exibicaoProxOleo = (ehVigenteOleo && diffOleoDias !== null && diffOleoDias <= 5) ? `<span class="alert-text">⚠️ ${proxOleoBR}</span>` : proxOleoBR;

        let dataAgenda = reg.proximaTrocaOleo || reg.proximaLubrificacao;
        let btnAgendaHtml = '';
        if (dataAgenda) {
            const dataStr = dataAgenda.replace(/-/g, '');
            const titulo = encodeURIComponent(`Revisão/Óleo - ${reg.placa} (${reg.empresa})`);
            const detalhes = encodeURIComponent(`Fazer ${reg.tipoServico || 'serviço'} do caminhão placa ${reg.placa}.`);
            const linkCal = `https://calendar.google.com/calendar/render?action=TEMPLATE&text=${titulo}&dates=${dataStr}/${dataStr}&details=${detalhes}`;
            btnAgendaHtml = `<a href="${linkCal}" target="_blank" class="action-btn" title="Salvar no Google Agenda" style="text-decoration: none; font-size: 1rem;">📅</a>`;
        }

        const tr = document.createElement('tr');
        if (alertaAtivo) tr.classList.add('alert-row');
        tr.innerHTML = `
            <td data-label="Placa">${escapeHtml(reg.placa)}</td>
            <td data-label="Motorista" title="${escapeHtml(reg.motorista)}">${escapeHtml(reg.motorista)}</td>
            <td data-label="Data">${dataServicoBR}</td>
            <td data-label="Tipo">${escapeHtml(reg.tipoServico || 'Lubrificação')}</td>
            <td data-label="Descrição" class="col-descricao" title="${escapeHtml(reg.descricao || '')}">${escapeHtml(reg.descricao) || '—'}</td>
            <td data-label="Próx. Lub.">${exibicaoProxLub}</td>
            <td data-label="Próx. Óleo">${exibicaoProxOleo}</td>
            <td data-label="Valor">R$ ${reg.valor}</td>
            <td data-label="Status"><span class="badge ${reg.statusPagamento === 'Pago' ? 'badge-pago' : 'badge-pendente'}">${reg.statusPagamento}</span></td>
            <td data-label="Comp.">
                ${reg.comprovanteUrl ? `<a href="${escapeHtml(reg.comprovanteUrl)}" target="_blank" class="comprovante-btn">📄 Ver</a>` : '—'}
            </td>
            <td data-label="Ações">
                <div class="flex-actions">
                    ${btnAgendaHtml}
                    <button type="button" class="action-btn btn-editar-acao" onclick="editarRegistro('${reg.idDoc}')" title="Editar">📝</button>
                    <button type="button" class="action-btn btn-excluir-acao" onclick="excluirRegistro('${reg.idDoc}')" title="Excluir">🗑️</button>
                </div>
            </td>
        `;
        return tr;
    }
```

- [ ] **Step 7: `calcularStatusPorEmpresa` — agrupar o vencimento por placa num status por empresa**

Adicionar logo depois de `calcularUltimosVencimentosPorPlaca` (mesmo local onde ela já está definida):

```js
    function calcularStatusPorEmpresa(dados) {
        const ultimos = calcularUltimosVencimentosPorPlaca(dados);
        const dataHoje = new Date();
        const hojePadrao = new Date(dataHoje.getFullYear(), dataHoje.getMonth(), dataHoje.getDate());
        const statusPorEmpresa = {};

        // Chave normalizada (minúsculo/sem espaço nas pontas) — a mesma empresa pode ter sido
        // salva com variação de maiúscula/espaço em registros diferentes; sem isso, duas chaves
        // separadas apareceriam pra "a mesma" empresa e a bolinha da lista de pastas erraria.
        const registrarStatus = (empresa, status) => {
            if (!empresa) return;
            const chave = empresa.trim().toLowerCase();
            const atual = statusPorEmpresa[chave];
            const prioridade = { vencido: 2, urgente: 1, ok: 0 };
            if (!atual || prioridade[status] > prioridade[atual]) statusPorEmpresa[chave] = status;
        };

        const avaliarData = (dataIso) => {
            const [ano, mes, dia] = dataIso.split('-');
            const data = new Date(ano, mes - 1, dia);
            const diff = Math.round((data - hojePadrao) / (1000 * 60 * 60 * 24));
            if (diff < 0) return 'vencido';
            if (diff <= 5) return 'urgente';
            return 'ok';
        };

        Object.values(ultimos).forEach(entry => {
            if (entry.lub) registrarStatus(entry.lub.empresa, avaliarData(entry.lub.proximaLubrificacao));
            if (entry.oleo) registrarStatus(entry.oleo.empresa, avaliarData(entry.oleo.proximaTrocaOleo));
        });

        return statusPorEmpresa;
    }
```

- [ ] **Step 8: `renderizarListaPastas`, `abrirPasta` e `renderizarDetalhePasta`**

Adicionar no lugar de onde ficava a antiga `renderizarSidebarPastas` (já apagada no Step 4):

```js
    window.abrirPasta = function(nomeEmpresa) {
        location.hash = '#pasta/' + encodeURIComponent(nomeEmpresa);
    }

    window.renderizarListaPastas = function() {
        const dados = window.registrosGlobais || [];
        const grid = document.getElementById('gridPastas');
        const empresasMap = new Map(); // chave normalizada -> nome original (primeiro visto)
        dados.forEach(reg => {
            if (!reg.empresa) return;
            const chave = reg.empresa.trim().toLowerCase();
            if (!empresasMap.has(chave)) empresasMap.set(chave, reg.empresa.trim());
        });

        const statusPorEmpresa = calcularStatusPorEmpresa(dados);
        const filterEmp = document.getElementById('filterEmpresa').value.toLowerCase().trim();
        const filterPlac = document.getElementById('filterPlaca').value.toLowerCase().trim();
        const filterStatus = document.getElementById('filterStatus').value;

        const empresasFiltradas = Array.from(empresasMap.values()).filter(empresaNome => {
            const registrosDaEmpresa = dados.filter(r => (r.empresa || '').trim().toLowerCase() === empresaNome.toLowerCase());
            const bateEmpresa = empresaNome.toLowerCase().includes(filterEmp);
            const batePlaca = filterPlac === '' || registrosDaEmpresa.some(r => (r.placa || '').toLowerCase().includes(filterPlac));
            const bateStatus = filterStatus === 'TODOS' || registrosDaEmpresa.some(r => r.statusPagamento === filterStatus);
            return bateEmpresa && batePlaca && bateStatus;
        }).sort((a, b) => a.localeCompare(b));

        if (empresasFiltradas.length === 0) {
            grid.innerHTML = '<div class="view-placeholder">Nenhuma pasta encontrada.</div>';
            return;
        }

        grid.innerHTML = empresasFiltradas.map(empresaNome => {
            // statusPorEmpresa é indexado por nome normalizado (minúsculo) — ver nota de
            // normalização na Task 7. Usar a mesma normalização aqui pra bater certinho.
            const status = statusPorEmpresa[empresaNome.toLowerCase()] || 'ok';
            return `
                <div class="pasta-card" data-empresa="${escapeHtml(empresaNome)}" onclick="window.abrirPasta(this.dataset.empresa)">
                    <span>📁 ${escapeHtml(empresaNome)}</span>
                    <span class="status-dot ${status}" title="${status === 'vencido' ? 'Tem vencimento atrasado' : status === 'urgente' ? 'Vence em breve' : 'Em dia'}"></span>
                </div>
            `;
        }).join('');
    }

    window.renderizarDetalhePasta = function(empresaCodificada) {
        const nomeEmpresa = decodeURIComponent(empresaCodificada);
        const dados = window.registrosGlobais || [];
        const registrosDaEmpresa = dados.filter(r => (r.empresa || '').trim().toLowerCase() === nomeEmpresa.trim().toLowerCase());

        const subviewLista = document.getElementById('pastasSubviewLista');
        const subviewDetalhe = document.getElementById('pastasSubviewDetalhe');

        if (registrosDaEmpresa.length === 0) {
            // Empresa não existe (renomeada/apagada) — volta pra lista
            location.hash = '#pastas';
            return;
        }

        subviewLista.style.display = 'none';
        subviewDetalhe.style.display = 'block';

        const totalGanho = registrosDaEmpresa.filter(r => r.statusPagamento === 'Pago').reduce((soma, r) => soma + parseFloat(r.valor || 0), 0);
        const totalDeve = registrosDaEmpresa.filter(r => r.statusPagamento === 'Pendente').reduce((soma, r) => soma + parseFloat(r.valor || 0), 0);

        document.getElementById('conteudoDetalhePasta').innerHTML = `
            <h2 style="margin-bottom: 10px;">📁 ${escapeHtml(registrosDaEmpresa[0].empresa)}</h2>
            <div class="resumo-pasta">
                <div class="finance-item">Serviços feitos: <strong>${registrosDaEmpresa.length}</strong></div>
                <div class="finance-item">Total ganho: <span class="val-lucro">R$ ${totalGanho.toFixed(2)}</span></div>
                <div class="finance-item">Total a receber: <span class="val-pendente">R$ ${totalDeve.toFixed(2)}</span></div>
            </div>
        `;

        const corpoPasta = document.getElementById('tabelaCorpoPasta');
        corpoPasta.innerHTML = '';
        const ultimosVencimentos = calcularUltimosVencimentosPorPlaca(dados);
        const dataHoje = new Date();
        const hojePadrao = new Date(dataHoje.getFullYear(), dataHoje.getMonth(), dataHoje.getDate());
        registrosDaEmpresa.forEach(reg => {
            corpoPasta.appendChild(montarLinhaHistoricoPasta(reg, ultimosVencimentos, hojePadrao));
        });
    }
```

Nota: `renderizarDetalhePasta` recebe `ultimosVencimentos` calculado sobre **todos** os `dados` (não só os da empresa) porque `calcularUltimosVencimentosPorPlaca` já é por placa — placas são únicas por empresa na prática, então calcular sobre todos os dados dá o mesmo resultado que calcular só sobre os da empresa, sem precisar filtrar de novo.

- [ ] **Step 9: Fazer o roteador (`renderizarViewAtual`) chamar `renderizarDetalhePasta` quando o hash tiver `/`**

No trecho adicionado na Task 1 Step 3, localizar:

```js
        if (viewAtiva === 'pastas' && typeof window.renderizarListaPastas === 'function') {
            window.renderizarListaPastas();
        }
```

Substituir por:

```js
        if (viewAtiva === 'pastas') {
            const hash = (location.hash || '').replace('#', '');
            if (hash.startsWith('pasta/')) {
                window.renderizarDetalhePasta(hash.substring('pasta/'.length));
            } else {
                document.getElementById('pastasSubviewDetalhe').style.display = 'none';
                document.getElementById('pastasSubviewLista').style.display = 'block';
                window.renderizarListaPastas();
            }
        }
```

- [ ] **Step 10: Verificar no navegador com dados reais**

1. Abrir `index.html` já logado, ir na aba "Pastas".
2. `read_console_messages(onlyErrors: true)` → vazio.
3. Confirmar que a empresa "Nutri+" (dona da placa RRZ9C05, que hoje está "urgente") aparece com bolinha **amarela** no card.
4. Rodar via `javascript_tool`: `JSON.stringify(calcularStatusPorEmpresa(window.registrosGlobais))` e conferir que bate com o que a tela mostra.
5. Clicar no card da empresa → vai pra `#pasta/Nutri%2B` (ou nome equivalente), mostra resumo (quantidade, ganho, a receber) e a tabela histórica só dela.
6. Conferir manualmente: `window.registrosGlobais.filter(r => r.empresa === 'Nutri+').length` deve bater com o número mostrado em "Serviços feitos".
7. Clicar em "◀ Voltar pra lista de pastas" → volta pra lista.
8. Digitar no campo de busca de empresa → filtra os cards corretamente.
9. Voltar pra aba "Visão Geral" → confirmar que ela continua mostrando os números/gráficos certos (sem tabela agora, isso é esperado) e sem erro no console.

- [ ] **Step 11: Commit**

```bash
git add index.html
git commit -m "$(cat <<'EOF'
adiciona aba Pastas: lista de empresas com indicador de vencimento e
detalhe por pasta

Cada pasta (empresa) ganha um card com bolinha vermelha/amarela quando
tem placa vencida/perto de vencer (calcularStatusPorEmpresa agrega o
calculo por placa ja existente). Ao entrar numa pasta, mostra resumo
(quantidade de servicos, total ganho, total a receber) e o historico
completo so daquela empresa. A tabela e os filtros de empresa/placa/status
saem da Visao Geral (que passa a ser so resumo + graficos) e viram parte
da aba Pastas.

Co-Authored-By: Claude Sonnet 5 <noreply@anthropic.com>
EOF
)"
```

---

### Task 3: Gráficos dentro da Pasta (sem duplicar canvas, sem quebrar o Chart.js)

**Files:**
- Modify: `index.html` — HTML (2 canvases novos dentro de `#pastasSubviewDetalhe`), JS (extrai `construirDadosGraficos`, adiciona `atualizarGraficosPasta`).

**Interfaces:**
- Consumes: nenhuma nova — usa `Chart` (Chart.js, já carregado via `<script src="https://cdn.jsdelivr.net/npm/chart.js">` no `<head>`).
- Produces: `function construirDadosGraficos(totalPago, totalPendente, contagemTipos)` → `{ dataCaixa, dataTipos }`.
- Produces: `function atualizarGraficosPasta(totalPago, totalPendente, contagemTipos)`.

- [ ] **Step 1: Adicionar os dois canvases da pasta**

Localizar o texto exato (adicionado na Task 2 Step 2):

```html
        <div id="pastasSubviewDetalhe" style="display: none;">
            <button type="button" class="btn-voltar-pasta" onclick="location.hash = '#pastas'">◀ Voltar pra lista de pastas</button>
            <div id="conteudoDetalhePasta"></div>
            <div class="table-container">
```

Substituir por:

```html
        <div id="pastasSubviewDetalhe" style="display: none;">
            <button type="button" class="btn-voltar-pasta" onclick="location.hash = '#pastas'">◀ Voltar pra lista de pastas</button>
            <div id="conteudoDetalhePasta"></div>
            <div class="charts-container">
                <div class="chart-card">
                    <h3>📊 Pago x Pendente (histórico da pasta)</h3>
                    <div class="chart-wrapper"><canvas id="chartBarrasPasta"></canvas></div>
                </div>
                <div class="chart-card">
                    <h3>🥧 Tipos de Serviço (histórico da pasta)</h3>
                    <div class="chart-wrapper"><canvas id="chartPizzaPasta"></canvas></div>
                </div>
            </div>
            <div class="table-container">
```

- [ ] **Step 2: Extrair `construirDadosGraficos` de dentro de `atualizarGraficos`**

Localizar a função `atualizarGraficos` inteira (do `function atualizarGraficos(totalPago, totalPendente, contagemTipos) {` até o `}` que a fecha, é a primeira função do segundo `<script type="module">`) e substituir por:

```js
    function construirDadosGraficos(totalPago, totalPendente, contagemTipos) {
        const dataCaixa = {
            labels: ['Recebido (Pago)', 'A Receber (Pendente)'],
            datasets: [{
                data: [totalPago, totalPendente],
                backgroundColor: ['#22c55e', '#eab308'],
                borderRadius: 5,
                barPercentage: 0.5
            }]
        };

        const labelsTipos = Object.keys(contagemTipos);
        const valoresTipos = Object.values(contagemTipos);
        const coresTipos = ['#f5a623', '#38bdf8', '#8890a0', '#22c55e'];
        const dataTipos = {
            labels: labelsTipos,
            datasets: [{
                data: valoresTipos,
                backgroundColor: coresTipos,
                borderColor: '#1a1e25',
                borderWidth: 3
            }]
        };

        return { dataCaixa, dataTipos };
    }

    const OPCOES_GRAFICO_BARRAS = {
        responsive: true,
        maintainAspectRatio: false,
        plugins: { legend: { display: false } },
        scales: {
            x: { ticks: { color: '#8890a0', font: { size: 11 } }, grid: { display: false } },
            y: { beginAtZero: true, ticks: { color: '#8890a0', font: { family: 'monospace', size: 10 } }, grid: { color: 'rgba(255,255,255,0.06)' } }
        }
    };

    const OPCOES_GRAFICO_PIZZA = {
        responsive: true,
        maintainAspectRatio: false,
        cutout: '65%',
        plugins: {
            legend: {
                position: 'bottom',
                labels: { color: '#8890a0', font: { size: 11 }, boxWidth: 10, padding: 10 }
            }
        }
    };

    function atualizarGraficos(totalPago, totalPendente, contagemTipos) {
        const ctxCaixa = document.getElementById('chartBarrasMensal');
        const ctxTipos = document.getElementById('chartPizzaServicos');
        const { dataCaixa, dataTipos } = construirDadosGraficos(totalPago, totalPendente, contagemTipos);

        if (chartCaixaInstance) {
            chartCaixaInstance.data = dataCaixa;
            chartCaixaInstance.update();
        } else {
            chartCaixaInstance = new Chart(ctxCaixa, { type: 'bar', data: dataCaixa, options: OPCOES_GRAFICO_BARRAS });
        }

        if (chartTiposInstance) {
            chartTiposInstance.data = dataTipos;
            chartTiposInstance.update();
        } else {
            chartTiposInstance = new Chart(ctxTipos, { type: 'doughnut', data: dataTipos, options: OPCOES_GRAFICO_PIZZA });
        }
    }

    let chartCaixaPastaInstance = null;
    let chartTiposPastaInstance = null;

    function atualizarGraficosPasta(totalPago, totalPendente, contagemTipos) {
        const ctxCaixa = document.getElementById('chartBarrasPasta');
        const ctxTipos = document.getElementById('chartPizzaPasta');
        const { dataCaixa, dataTipos } = construirDadosGraficos(totalPago, totalPendente, contagemTipos);

        // Mesmos 2 canvases sempre reaproveitados ao trocar de empresa — nunca cria um Chart
        // novo em cima de um canvas que já tem instância ativa (evita "Canvas is already in use").
        if (chartCaixaPastaInstance) {
            chartCaixaPastaInstance.data = dataCaixa;
            chartCaixaPastaInstance.update();
        } else {
            chartCaixaPastaInstance = new Chart(ctxCaixa, { type: 'bar', data: dataCaixa, options: OPCOES_GRAFICO_BARRAS });
        }

        if (chartTiposPastaInstance) {
            chartTiposPastaInstance.data = dataTipos;
            chartTiposPastaInstance.update();
        } else {
            chartTiposPastaInstance = new Chart(ctxTipos, { type: 'doughnut', data: dataTipos, options: OPCOES_GRAFICO_PIZZA });
        }
    }
```

- [ ] **Step 3: Chamar `atualizarGraficosPasta` a partir de `renderizarDetalhePasta`**

Localizar, dentro de `window.renderizarDetalhePasta` (adicionada na Task 2), o texto exato:

```js
        registrosDaEmpresa.forEach(reg => {
            corpoPasta.appendChild(montarLinhaHistoricoPasta(reg, ultimosVencimentos, hojePadrao));
        });
    }
```

Substituir por:

```js
        registrosDaEmpresa.forEach(reg => {
            corpoPasta.appendChild(montarLinhaHistoricoPasta(reg, ultimosVencimentos, hojePadrao));
        });

        const tiposContagemPasta = {};
        registrosDaEmpresa.forEach(r => {
            const tipo = r.tipoServico || 'Outros';
            tiposContagemPasta[tipo] = (tiposContagemPasta[tipo] || 0) + 1;
        });
        atualizarGraficosPasta(totalGanho, totalDeve, tiposContagemPasta);
    }
```

- [ ] **Step 4: Verificar que trocar de pasta não quebra o gráfico**

1. Abrir a aba Pastas, entrar numa empresa (ex: "Nutri+"), tirar screenshot dos gráficos.
2. Voltar pra lista, entrar em **outra** empresa (se só existir uma no banco de teste, entrar duas vezes seguidas na mesma já serve pra provar que não duplica instância).
3. `read_console_messages(onlyErrors: true)` → **não pode conter** a string "Canvas is already in use" nem qualquer outro erro.
4. Conferir visualmente que os números do gráfico batem com o resumo (ganho/a receber) mostrado acima dele.

- [ ] **Step 5: Commit**

```bash
git add index.html
git commit -m "$(cat <<'EOF'
adiciona graficos (pago x pendente, tipos de servico) dentro da pasta

Extrai construirDadosGraficos de atualizarGraficos para reaproveitar a
montagem dos dados do Chart.js. A pasta usa dois canvases fixos e unicos
(chartBarrasPasta/chartPizzaPasta) atualizados por instancia, nunca
recriados, evitando o erro 'Canvas is already in use' ao trocar de
empresa.

Co-Authored-By: Claude Sonnet 5 <noreply@anthropic.com>
EOF
)"
```

---

### Task 4: Seção "Pendentes" + botão "Gerar cobrança (WhatsApp)" na pasta

**Files:**
- Modify: `index.html` — CSS, JS (`renderizarDetalhePasta` ganha uma chamada nova, mais 3 funções: `renderizarPendentesPasta`, `gerarMensagemCobranca`, `window.gerarCobrancaWhatsapp`).

**Interfaces:**
- Consumes: `window.registrosGlobais`, `formatarData`, `escapeHtml` (já existentes).
- Produces: `function gerarMensagemCobranca(nomeEmpresa, pendentes)` → string pura (sem DOM), usada tanto pelo botão quanto pela verificação manual.
- Produces: `window.gerarCobrancaWhatsapp(nomeEmpresa)`.

- [ ] **Step 1: CSS da lista de pendentes e do botão de cobrança**

Adicionar no mesmo bloco de CSS das tasks anteriores:

```css
        .lista-pendentes-pasta { margin: 10px 0; font-size: 0.82rem; }
        .pendente-item-pasta {
            display: flex;
            justify-content: space-between;
            padding: 6px 0;
            border-bottom: 1px dashed var(--border);
        }
        .btn-cobranca-whatsapp {
            background-color: #22c55e;
            margin-top: 8px;
        }
        .btn-cobranca-whatsapp:hover { background-color: #16a34a; }
```

- [ ] **Step 2: Adicionar o container de pendentes no HTML gerado do resumo da pasta**

Localizar, dentro de `window.renderizarDetalhePasta`, o texto exato:

```js
        document.getElementById('conteudoDetalhePasta').innerHTML = `
            <h2 style="margin-bottom: 10px;">📁 ${escapeHtml(registrosDaEmpresa[0].empresa)}</h2>
            <div class="resumo-pasta">
                <div class="finance-item">Serviços feitos: <strong>${registrosDaEmpresa.length}</strong></div>
                <div class="finance-item">Total ganho: <span class="val-lucro">R$ ${totalGanho.toFixed(2)}</span></div>
                <div class="finance-item">Total a receber: <span class="val-pendente">R$ ${totalDeve.toFixed(2)}</span></div>
            </div>
        `;
```

Substituir por:

```js
        document.getElementById('conteudoDetalhePasta').innerHTML = `
            <h2 style="margin-bottom: 10px;">📁 ${escapeHtml(registrosDaEmpresa[0].empresa)}</h2>
            <div class="resumo-pasta">
                <div class="finance-item">Serviços feitos: <strong>${registrosDaEmpresa.length}</strong></div>
                <div class="finance-item">Total ganho: <span class="val-lucro">R$ ${totalGanho.toFixed(2)}</span></div>
                <div class="finance-item">Total a receber: <span class="val-pendente">R$ ${totalDeve.toFixed(2)}</span></div>
            </div>
            <div id="pendentesPastaContainer"></div>
        `;
        renderizarPendentesPasta(registrosDaEmpresa[0].empresa, registrosDaEmpresa);
```

- [ ] **Step 3: Funções `renderizarPendentesPasta`, `gerarMensagemCobranca` e `window.gerarCobrancaWhatsapp`**

Adicionar logo depois do fechamento de `window.renderizarDetalhePasta`:

```js
    function renderizarPendentesPasta(nomeEmpresa, registrosDaEmpresa) {
        const pendentes = registrosDaEmpresa.filter(r => r.statusPagamento === 'Pendente');
        const container = document.getElementById('pendentesPastaContainer');

        if (pendentes.length === 0) {
            container.innerHTML = '<div class="view-placeholder">Nenhum pendente para essa empresa.</div>';
            return;
        }

        const itensHtml = pendentes.map(r => `
            <div class="pendente-item-pasta">
                <span>${formatarData(r.dataServico)} — ${escapeHtml(r.tipoServico || 'Serviço')} (placa ${escapeHtml(r.placa)})</span>
                <strong>R$ ${r.valor}</strong>
            </div>
        `).join('');

        container.innerHTML = `
            <h3 style="margin: 14px 0 6px 0; font-size: 0.8rem; color: var(--text-muted); text-transform: uppercase;">Pendentes</h3>
            <div class="lista-pendentes-pasta">${itensHtml}</div>
            <button type="button" class="btn-cobranca-whatsapp" data-empresa="${escapeHtml(nomeEmpresa)}" onclick="window.gerarCobrancaWhatsapp(this.dataset.empresa)">💬 Gerar cobrança (WhatsApp)</button>
        `;
    }

    // Função pura (sem DOM) — recebe já a lista de pendentes e devolve o texto da mensagem.
    // Isso permite testar o formato da mensagem direto no console, sem precisar clicar em nada.
    function gerarMensagemCobranca(nomeEmpresa, pendentes) {
        const linhas = pendentes.map(r => `- ${formatarData(r.dataServico)} - ${r.tipoServico || 'Serviço'} (placa ${r.placa}): R$ ${r.valor}`);
        const total = pendentes.reduce((soma, r) => soma + parseFloat(r.valor || 0), 0);
        return `Olá! Segue o resumo dos serviços pendentes da ${nomeEmpresa}:\n\n${linhas.join('\n')}\n\nTotal pendente: R$ ${total.toFixed(2)}`;
    }

    window.gerarCobrancaWhatsapp = function(nomeEmpresa) {
        const dados = window.registrosGlobais || [];
        const pendentes = dados.filter(r =>
            (r.empresa || '').trim().toLowerCase() === nomeEmpresa.trim().toLowerCase() &&
            r.statusPagamento === 'Pendente'
        );
        if (pendentes.length === 0) return;
        const mensagem = gerarMensagemCobranca(nomeEmpresa, pendentes);
        window.open('https://wa.me/?text=' + encodeURIComponent(mensagem), '_blank');
    }
```

- [ ] **Step 4: Verificar a função pura no console (sem clicar em nada ainda)**

Via `javascript_tool`, com a página carregada:

```js
gerarMensagemCobranca('Nutri+', [
  { dataServico: '2026-07-11', tipoServico: 'Lubrificação', placa: 'RRZ9C05', valor: '150.00' },
  { dataServico: '2026-07-20', tipoServico: 'Troca de Óleo', placa: 'RRZ9C05', valor: '300.00' }
]);
```

Esperado (exatamente):

```
Olá! Segue o resumo dos serviços pendentes da Nutri+:

- 11/07/2026 - Lubrificação (placa RRZ9C05): R$ 150.00
- 20/07/2026 - Troca de Óleo (placa RRZ9C05): R$ 300.00

Total pendente: R$ 450.00
```

- [ ] **Step 5: Verificar visualmente na pasta real (sem clicar no botão de verdade, pra não abrir o WhatsApp de verdade durante o teste)**

1. Entrar numa pasta que tenha pendente real (dado de produção).
2. Conferir que a seção "Pendentes" lista exatamente os registros com `statusPagamento === 'Pendente'` daquela empresa, e que o total bate com "Total a receber" do resumo acima.
3. Se a empresa **não** tiver pendente, conferir que aparece "Nenhum pendente para essa empresa." e nenhum botão.
4. `read_console_messages(onlyErrors: true)` → vazio.
5. (Opcional, só se o usuário confirmar que quer ver de verdade) clicar no botão manualmente pelo próprio navegador do usuário — não clicar por automação, porque abre uma aba externa do WhatsApp.

- [ ] **Step 6: Commit**

```bash
git add index.html
git commit -m "$(cat <<'EOF'
adiciona secao Pendentes e botao de gerar cobranca via WhatsApp na pasta

Lista os servicos pendentes daquela empresa com data/tipo/placa/valor e
soma o total. O botao monta a mensagem (gerarMensagemCobranca, funcao
pura) e abre https://wa.me/?text=... para o usuario escolher o contato
manualmente - nenhum numero de telefone e armazenado.

Co-Authored-By: Claude Sonnet 5 <noreply@anthropic.com>
EOF
)"
```

---

### Task 5: Aba "Vencimento" completa (tira os cards de alerta da Visão Geral)

**Files:**
- Modify: `index.html` — HTML (move `.agenda-alertas` de `#viewVisaoGeral` pra `#viewVencimento`), CSS (remove o limite de altura só dentro de `#viewVencimento`), JS (`atualizarPainelAvisos` ganha clique; nova `window.renderizarVencimento`; remove de `renderizarTabelaCompleta` o cálculo de vencidos/urgentes que se muda pra cá).

**Interfaces:**
- Consumes: `calcularUltimosVencimentosPorPlaca(dados)`, `window.abrirPasta(nomeEmpresa)` (Task 2), `escapeHtml`.
- Produces: `window.renderizarVencimento()` — sem parâmetros, já era esperada pelo dispatcher desde a Task 1 Step 3 (`if (viewAtiva === 'vencimento' && typeof window.renderizarVencimento === 'function')`), então não precisa mexer no roteador nesta task.

- [ ] **Step 1: CSS — remover o limite de altura dentro da aba Vencimento**

Adicionar no bloco de CSS:

```css
        #viewVencimento .lista-avisos { max-height: none; }
```

- [ ] **Step 2: Tirar `.agenda-alertas` de dentro da Visão Geral**

Localizar o texto exato:

```html
    <div id="viewVisaoGeral" class="view-secao">
    <div class="agenda-alertas">
        <div class="alerta-card vencido" id="cardVencidos">
            <h3>🔴 Próximos Vencidos (Passou do prazo)</h3>
            <div class="lista-avisos" id="listaVencidos">
                <div style="color: var(--text-muted);">Nenhum item vencido.</div>
            </div>
        </div>
        <div class="alerta-card urgente" id="cardUrgentes">
            <h3>🟡 Vencem nos próximos 5 dias</h3>
            <div class="lista-avisos" id="listaUrgentes">
                <div style="color: var(--text-muted);">Nenhum vencimento próximo.</div>
            </div>
        </div>
    </div>

    <div class="filters-panel">
```

Substituir por:

```html
    <div id="viewVisaoGeral" class="view-secao">
    <div class="filters-panel">
```

- [ ] **Step 3: Colocar `.agenda-alertas` de verdade dentro de `#viewVencimento`**

Localizar o texto exato (placeholder criado na Task 1):

```html
    <div id="viewVencimento" class="view-secao">
        <div class="view-placeholder">Em construção — chega em breve.</div>
    </div>
```

Substituir por:

```html
    <div id="viewVencimento" class="view-secao">
        <div class="agenda-alertas">
            <div class="alerta-card vencido" id="cardVencidos">
                <h3>🔴 Vencidos (Passou do prazo)</h3>
                <div class="lista-avisos" id="listaVencidos">
                    <div style="color: var(--text-muted);">Nenhum item vencido.</div>
                </div>
            </div>
            <div class="alerta-card urgente" id="cardUrgentes">
                <h3>🟡 Vencem nos próximos 5 dias</h3>
                <div class="lista-avisos" id="listaUrgentes">
                    <div style="color: var(--text-muted);">Nenhum vencimento próximo.</div>
                </div>
            </div>
        </div>
    </div>
```

- [ ] **Step 4: Remover o cálculo de vencidos/urgentes de dentro de `renderizarTabelaCompleta`**

Localizar o texto exato:

```js
        let vencidosList = [];
        let urgentesList = [];
        let totalPendenteGeral = 0;
```

Substituir por:

```js
        let totalPendenteGeral = 0;
```

Localizar o texto exato (bloco que soma vencidos/urgentes, hoje logo depois do filtro de mês):

```js
        // Vencimento "vigente" de cada placa/tipo (sempre baseado em TODOS os registros,
        // não só nos filtrados, para o painel de alertas não sumir quando um filtro é aplicado)
        const ultimosVencimentos = calcularUltimosVencimentosPorPlaca(dados);

        Object.keys(ultimosVencimentos).forEach(placaKey => {
            const entry = ultimosVencimentos[placaKey];

            if (entry.lub) {
                const [anoL, mesL, diaL] = entry.lub.proximaLubrificacao.split('-');
                const dataLub = new Date(anoL, mesL - 1, diaL);
                const diff = Math.round((dataLub - hojePadrao) / (1000 * 60 * 60 * 24));
                if (diff < 0) {
                    vencidosList.push({ idDoc: entry.lub.idDoc, placa: entry.lub.placa, empresa: entry.lub.empresa, tipo: 'Lubrificação', data: entry.lub.proximaLubrificacao, dias: Math.abs(diff) });
                } else if (diff <= 5) {
                    urgentesList.push({ idDoc: entry.lub.idDoc, placa: entry.lub.placa, empresa: entry.lub.empresa, tipo: 'Lubrificação', data: entry.lub.proximaLubrificacao, dias: diff });
                }
            }

            if (entry.oleo) {
                const [anoO, mesO, diaO] = entry.oleo.proximaTrocaOleo.split('-');
                const dataOleo = new Date(anoO, mesO - 1, diaO);
                const diff = Math.round((dataOleo - hojePadrao) / (1000 * 60 * 60 * 24));
                if (diff < 0) {
                    vencidosList.push({ idDoc: entry.oleo.idDoc, placa: entry.oleo.placa, empresa: entry.oleo.empresa, tipo: 'Troca de Óleo', data: entry.oleo.proximaTrocaOleo, dias: Math.abs(diff) });
                } else if (diff <= 5) {
                    urgentesList.push({ idDoc: entry.oleo.idDoc, placa: entry.oleo.placa, empresa: entry.oleo.empresa, tipo: 'Troca de Óleo', data: entry.oleo.proximaTrocaOleo, dias: diff });
                }
            }
        });

        document.getElementById('lucroMes').innerText = `R$ ${lucroMesSelecionado.toFixed(2)}`;
```

Substituir por:

```js
        document.getElementById('lucroMes').innerText = `R$ ${lucroMesSelecionado.toFixed(2)}`;
```

Localizar o texto exato:

```js
        atualizarPainelAvisos(vencidosList, urgentesList);
        atualizarGraficos(lucroMesSelecionado, pendenteMesSelecionado, tiposContagemMes);
    }
```

Substituir por:

```js
        atualizarGraficos(lucroMesSelecionado, pendenteMesSelecionado, tiposContagemMes);
    }
```

Por fim, `hojePadrao`/`dataHoje` ficaram sem uso dentro de `renderizarTabelaCompleta` depois dessas remoções (só serviam pro cálculo que acabou de sair). Localizar o texto exato, logo no início da função:

```js
        const dataHoje = new Date();
        // Lógica mais segura para garantir o "Hoje" na meia-noite sem fuso
        const hojePadrao = new Date(dataHoje.getFullYear(), dataHoje.getMonth(), dataHoje.getDate());

        const mesSelecionado = parseInt(document.getElementById('selectMes').value);
```

Substituir por:

```js
        const mesSelecionado = parseInt(document.getElementById('selectMes').value);
```

- [ ] **Step 5: `atualizarPainelAvisos` ganha clique pra abrir a pasta + nova `window.renderizarVencimento`**

Localizar a função `atualizarPainelAvisos` inteira (do `function atualizarPainelAvisos(vencidos, urgentes) {` até o `}` que a fecha) e substituir por:

```js
    function atualizarPainelAvisos(vencidos, urgentes) {
        const listaVencidos = document.getElementById('listaVencidos');
        const listaUrgentes = document.getElementById('listaUrgentes');

        function linkGoogleCalendar(item) {
            const titulo = encodeURIComponent(`Vencimento ${item.tipo} - Placa ${item.placa} (${item.empresa})`);
            const detalhes = encodeURIComponent(`O serviço de ${item.tipo} do caminhão placa ${item.placa} da empresa ${item.empresa} venceu ou está próximo.`);
            const dataStr = item.data.replace(/-/g, '');
            return `https://calendar.google.com/calendar/render?action=TEMPLATE&text=${titulo}&dates=${dataStr}/${dataStr}&details=${detalhes}`;
        }

        if (vencidos.length === 0) {
            listaVencidos.innerHTML = '<div style="color: var(--text-muted);">Nenhum item vencido.</div>';
        } else {
            listaVencidos.innerHTML = vencidos.map(item => `
                <div class="aviso-item" style="cursor:pointer;" data-empresa="${escapeHtml(item.empresa)}" onclick="window.abrirPasta(this.dataset.empresa)">
                    <span>⚠️ <strong>${escapeHtml(item.placa)}</strong> (${escapeHtml(item.empresa)}) - <span style="color: #f87171;">${escapeHtml(item.tipo)}</span> (${item.dias}d)</span>
                    <a href="${linkGoogleCalendar(item)}" target="_blank" class="btn-agenda-rapida" title="Adicionar à Agenda do Google" onclick="event.stopPropagation()">📅 Add Agenda</a>
                </div>
            `).join('');
        }

        if (urgentes.length === 0) {
            listaUrgentes.innerHTML = '<div style="color: var(--text-muted);">Nenhum vencimento próximo.</div>';
        } else {
            listaUrgentes.innerHTML = urgentes.map(item => {
                let textoDias = item.dias === 0 ? "Vence hoje!" : `Vence em ${item.dias}d`;
                return `
                    <div class="aviso-item" style="cursor:pointer;" data-empresa="${escapeHtml(item.empresa)}" onclick="window.abrirPasta(this.dataset.empresa)">
                        <span>📅 <strong>${escapeHtml(item.placa)}</strong> (${escapeHtml(item.empresa)}) - <span style="color: #fbbf24;">${escapeHtml(item.tipo)}</span>: ${textoDias}</span>
                        <a href="${linkGoogleCalendar(item)}" target="_blank" class="btn-agenda-rapida" title="Adicionar à Agenda do Google" onclick="event.stopPropagation()">📅 Add Agenda</a>
                    </div>
                `;
            }).join('');
        }
    }

    window.renderizarVencimento = function() {
        const dados = window.registrosGlobais || [];
        const ultimosVencimentos = calcularUltimosVencimentosPorPlaca(dados);
        const dataHoje = new Date();
        const hojePadrao = new Date(dataHoje.getFullYear(), dataHoje.getMonth(), dataHoje.getDate());
        let vencidosList = [];
        let urgentesList = [];

        Object.keys(ultimosVencimentos).forEach(placaKey => {
            const entry = ultimosVencimentos[placaKey];

            if (entry.lub) {
                const [anoL, mesL, diaL] = entry.lub.proximaLubrificacao.split('-');
                const dataLub = new Date(anoL, mesL - 1, diaL);
                const diff = Math.round((dataLub - hojePadrao) / (1000 * 60 * 60 * 24));
                if (diff < 0) {
                    vencidosList.push({ placa: entry.lub.placa, empresa: entry.lub.empresa, tipo: 'Lubrificação', data: entry.lub.proximaLubrificacao, dias: Math.abs(diff) });
                } else if (diff <= 5) {
                    urgentesList.push({ placa: entry.lub.placa, empresa: entry.lub.empresa, tipo: 'Lubrificação', data: entry.lub.proximaLubrificacao, dias: diff });
                }
            }

            if (entry.oleo) {
                const [anoO, mesO, diaO] = entry.oleo.proximaTrocaOleo.split('-');
                const dataOleo = new Date(anoO, mesO - 1, diaO);
                const diff = Math.round((dataOleo - hojePadrao) / (1000 * 60 * 60 * 24));
                if (diff < 0) {
                    vencidosList.push({ placa: entry.oleo.placa, empresa: entry.oleo.empresa, tipo: 'Troca de Óleo', data: entry.oleo.proximaTrocaOleo, dias: Math.abs(diff) });
                } else if (diff <= 5) {
                    urgentesList.push({ placa: entry.oleo.placa, empresa: entry.oleo.empresa, tipo: 'Troca de Óleo', data: entry.oleo.proximaTrocaOleo, dias: diff });
                }
            }
        });

        atualizarPainelAvisos(vencidosList, urgentesList);
    }
```

- [ ] **Step 6: Verificar no navegador com dados reais**

1. Abrir o sistema, ir na aba "Vencimento".
2. Confirmar que a placa RRZ9C05 (empresa "Nutri+") aparece em "🟡 Vencem nos próximos 5 dias" — mesmo texto/dias que já validamos antes.
3. Clicar no item da lista (fora do botão "Add Agenda") → deve navegar pra `#pasta/Nutri%2B` e abrir a pasta certa.
4. Clicar especificamente no botão "📅 Add Agenda" → deve abrir o link do Google Agenda numa aba nova, **sem** também navegar pra pasta (confirma que o `stopPropagation` funcionou).
5. Ir na aba "Visão Geral" → confirmar que os cards de alerta **não aparecem mais** ali (foram pra Vencimento), mas o resto (resumo financeiro + gráficos) continua certo.
6. `read_console_messages(onlyErrors: true)` → vazio.

- [ ] **Step 7: Commit**

```bash
git add index.html
git commit -m "$(cat <<'EOF'
move os avisos de vencido/urgente para uma aba Vencimento dedicada

Os cards que ficavam sempre visiveis no topo da Visao Geral (com scroll
limitado a 80px) agora vivem so na aba Vencimento, sem limite de altura,
e cada item e clicavel e abre a pasta da empresa correspondente. O
calculo de vencidos/urgentes sai de renderizarTabelaCompleta e vira
window.renderizarVencimento, usando a mesma calcularUltimosVencimentosPorPlaca
ja existente.

Co-Authored-By: Claude Sonnet 5 <noreply@anthropic.com>
EOF
)"
```

---

### Task 6: Modal "Novo Registro" flutuante — tipo obrigatório + autocomplete

**Files:**
- Modify: `index.html` — CSS (botão flutuante + overlay do modal), HTML (tira o form de dentro de `#viewVisaoGeral`, vira modal + botão flutuante + datalists), JS (`abrirModalNovoRegistro`, `fecharModalNovoRegistro`, `atualizarDatalists`, ajustes em `editarRegistro`/`cancelarEdicao`/dispatcher).

**Interfaces:**
- Consumes: `escapeHtml` (já existente).
- Produces: `window.abrirModalNovoRegistro()`, `window.fecharModalNovoRegistro()`, `function atualizarDatalists()`.

- [ ] **Step 1: CSS do botão flutuante e do overlay do modal**

Adicionar no bloco de CSS:

```css
        .btn-flutuante {
            position: fixed;
            bottom: 24px;
            right: 24px;
            z-index: 500;
            border-radius: 30px;
            padding: 14px 22px;
            box-shadow: 0 6px 16px rgba(0,0,0,0.4);
        }
        .modal-overlay {
            display: none;
            position: fixed;
            inset: 0;
            background: rgba(0,0,0,0.6);
            z-index: 1000;
            align-items: center;
            justify-content: center;
            padding: 16px;
        }
        .modal-overlay.aberto { display: flex; }
        .modal-box {
            background: var(--card-bg);
            border: 1px solid var(--border);
            border-top: 3px solid var(--primary);
            border-radius: 8px;
            padding: 20px;
            width: 100%;
            max-width: 640px;
            max-height: 90vh;
            overflow-y: auto;
        }
        .modal-grupo-titulo {
            font-size: 0.72rem;
            text-transform: uppercase;
            letter-spacing: 0.6px;
            color: var(--text-muted);
            margin: 14px 0 6px 0;
            font-family: 'Oswald', sans-serif;
        }
        .modal-grupo-titulo:first-of-type { margin-top: 0; }
```

- [ ] **Step 2: Tirar o formulário de dentro de `#viewVisaoGeral`**

Localizar o texto exato (o formulário completo, hoje ainda dentro da Visão Geral, logo depois do `.charts-container`):

```html
    <div class="card-horizontal">
        <h2 id="tituloForm">➕ Novo Registro</h2>
        <form id="lubForm" onsubmit="salvarRegistroFirebase(event)">
            <input type="hidden" id="registroId" value="">
            <div class="form-grid-horizontal">
                <div class="form-group"><label>Empresa</label><input type="text" id="empresa" required placeholder="Ex: RodoNutri"></div>
                <div class="form-group"><label>Placa</label><input type="text" id="placa" required placeholder="Ex: ABC1D23"></div>
                <div class="form-group"><label>Motorista/Contato</label><input type="text" id="motorista" required placeholder="Ex: João"></div>
                <div class="form-group"><label>Data Serviço</label><input type="date" id="dataServico" required></div>
                <div class="form-group">
                    <label>Tipo Serviço</label>
                    <select id="tipoServico" onchange="atualizarCamposPrazo()" required>
                        <option value="Lubrificação" selected>Lubrificação</option>
                        <option value="Troca de Óleo">Troca de Óleo</option>
                        <option value="Lubrificação + Troca de Óleo">Lubrificação + Troca de Óleo</option>
                        <option value="Outros">Outros</option>
                    </select>
                </div>
                <div class="form-group"><label>Prazo (Dias)</label><input type="number" id="prazoDias" required value="30"></div>
                <div class="form-group" style="grid-column: span 2;"><label>Descrição do Serviço</label><input type="text" id="descricao" placeholder="Ex: Troca de óleo de motor e filtros"></div>
                <div class="form-group"><label>Valor (R$)</label><input type="number" step="0.01" id="valor" required placeholder="1500.00"></div>
                <div class="form-group">
                    <label>Status Pagam.</label>
                    <select id="statusPagamento">
                        <option value="Pago">Pago</option>
                        <option value="Pendente" selected>Pendente</option>
                    </select>
                </div>
                <div class="form-group"><label>Comprovante</label><input type="file" id="comprovante" accept="image/*,application/pdf"></div>
                <div class="form-group" style="flex-direction: row; gap: 4px;">
                    <button type="submit" id="btnSalvar" style="flex: 1;">Salvar</button>
                    <button type="button" id="btnCancelarEdicao" onclick="cancelarEdicao()" style="background-color: var(--border); display: none;">X</button>
                </div>
            </div>
            <div id="comprovanteAtual" style="font-size: 0.75rem; color: var(--text-muted); margin-top: 6px;"></div>
        </form>
    </div>
    </div><!-- /viewVisaoGeral -->
```

Substituir por (Visão Geral fica só com resumo + gráficos):

```html
    </div><!-- /viewVisaoGeral -->
```

- [ ] **Step 3: Adicionar o botão flutuante, o modal e os datalists**

Localizar o texto exato (perto do fim do `<div class="container">`, depois de `#viewVencimento`):

```html
    <div id="toastMsg" class="toast"></div>
```

Substituir por:

```html
    <button type="button" id="btnNovoRegistroFloat" class="btn-flutuante" onclick="window.abrirModalNovoRegistro()">+ Novo Registro</button>

    <div id="modalOverlay" class="modal-overlay">
        <div class="modal-box">
            <div style="display:flex; justify-content:space-between; align-items:center; margin-bottom:12px;">
                <h2 id="tituloForm">➕ Novo Registro</h2>
                <button type="button" onclick="window.fecharModalNovoRegistro()" style="background:transparent; color: var(--text-muted); font-size:1.1rem; padding:0 6px;">✕</button>
            </div>
            <form id="lubForm" onsubmit="salvarRegistroFirebase(event)">
                <input type="hidden" id="registroId" value="">

                <h3 class="modal-grupo-titulo">Identificação</h3>
                <div class="form-grid-horizontal">
                    <div class="form-group"><label>Empresa</label><input type="text" id="empresa" required placeholder="Ex: RodoNutri" list="listaEmpresas"></div>
                    <div class="form-group"><label>Placa</label><input type="text" id="placa" required placeholder="Ex: ABC1D23" list="listaPlacas"></div>
                    <div class="form-group"><label>Motorista/Contato</label><input type="text" id="motorista" required placeholder="Ex: João"></div>
                </div>

                <h3 class="modal-grupo-titulo">Serviço</h3>
                <div class="form-grid-horizontal">
                    <div class="form-group"><label>Data Serviço</label><input type="date" id="dataServico" required></div>
                    <div class="form-group">
                        <label>Tipo Serviço</label>
                        <select id="tipoServico" onchange="atualizarCamposPrazo()" required>
                            <option value="" disabled selected>Selecione o tipo de serviço</option>
                            <option value="Lubrificação">Lubrificação</option>
                            <option value="Troca de Óleo">Troca de Óleo</option>
                            <option value="Lubrificação + Troca de Óleo">Lubrificação + Troca de Óleo</option>
                            <option value="Outros">Outros</option>
                        </select>
                    </div>
                    <div class="form-group"><label>Prazo (Dias)</label><input type="number" id="prazoDias" required value="30"></div>
                    <div class="form-group" style="grid-column: span 3;"><label>Descrição do Serviço</label><input type="text" id="descricao" placeholder="Ex: Troca de óleo de motor e filtros"></div>
                </div>

                <h3 class="modal-grupo-titulo">Financeiro</h3>
                <div class="form-grid-horizontal">
                    <div class="form-group"><label>Valor (R$)</label><input type="number" step="0.01" id="valor" required placeholder="1500.00"></div>
                    <div class="form-group">
                        <label>Status Pagam.</label>
                        <select id="statusPagamento">
                            <option value="Pago">Pago</option>
                            <option value="Pendente" selected>Pendente</option>
                        </select>
                    </div>
                    <div class="form-group"><label>Comprovante</label><input type="file" id="comprovante" accept="image/*,application/pdf"></div>
                    <div class="form-group" style="flex-direction: row; gap: 4px;">
                        <button type="submit" id="btnSalvar" style="flex: 1;">Salvar</button>
                        <button type="button" id="btnCancelarEdicao" onclick="cancelarEdicao()" style="background-color: var(--border); display: none;">X</button>
                    </div>
                </div>
                <div id="comprovanteAtual" style="font-size: 0.75rem; color: var(--text-muted); margin-top: 6px;"></div>
            </form>
        </div>
    </div>

    <datalist id="listaEmpresas"></datalist>
    <datalist id="listaPlacas"></datalist>

    <div id="toastMsg" class="toast"></div>
```

- [ ] **Step 4: `abrirModalNovoRegistro` / `fecharModalNovoRegistro`, e fazer `cancelarEdicao` fechar o modal**

Localizar o texto exato:

```js
    window.cancelarEdicao = function() {
        document.getElementById('lubForm').reset();
        document.getElementById('registroId').value = "";
        document.getElementById('comprovanteAtual').innerHTML = "";
        document.getElementById('dataServico').valueAsDate = new Date();
        document.getElementById('tituloForm').innerText = "➕ Novo Registro";
        document.getElementById('btnSalvar').innerText = "Salvar";
        document.getElementById('btnCancelarEdicao').style.display = 'none';
        comprovanteRemovidoNaEdicao = false;
        window.atualizarCamposPrazo();
    }
```

Substituir por:

```js
    window.cancelarEdicao = function() {
        document.getElementById('lubForm').reset();
        document.getElementById('registroId').value = "";
        document.getElementById('comprovanteAtual').innerHTML = "";
        document.getElementById('dataServico').valueAsDate = new Date();
        document.getElementById('tituloForm').innerText = "➕ Novo Registro";
        document.getElementById('btnSalvar').innerText = "Salvar";
        document.getElementById('btnCancelarEdicao').style.display = 'none';
        comprovanteRemovidoNaEdicao = false;
        window.atualizarCamposPrazo();
        document.getElementById('modalOverlay').classList.remove('aberto');
    }

    window.abrirModalNovoRegistro = function() {
        document.getElementById('modalOverlay').classList.add('aberto');
    }

    window.fecharModalNovoRegistro = function() {
        window.cancelarEdicao();
    }
```

Isso cobre os 3 jeitos de o modal fechar: botão "✕" do cabeçalho (chama `fecharModalNovoRegistro`), botão "X" de cancelar edição já existente (chama `cancelarEdicao` direto), e salvamento bem-sucedido (`salvarRegistroFirebase` já chama `window.cancelarEdicao()` no final — nenhuma mudança necessária lá).

- [ ] **Step 5: Abrir o modal automaticamente ao clicar em "Editar"**

Localizar o texto exato, no fim de `window.editarRegistro`:

```js
        document.getElementById('tituloForm').innerText = "📝 Editar Registro";
        document.getElementById('btnSalvar').innerText = "Atualizar";
        document.getElementById('btnCancelarEdicao').style.display = 'block';
    }
```

Substituir por:

```js
        document.getElementById('tituloForm').innerText = "📝 Editar Registro";
        document.getElementById('btnSalvar').innerText = "Atualizar";
        document.getElementById('btnCancelarEdicao').style.display = 'block';
        window.abrirModalNovoRegistro();
    }
```

- [ ] **Step 6: `atualizarDatalists`, chamada a cada re-render**

Adicionar logo depois de `formatarData`:

```js
    function atualizarDatalists() {
        const dados = window.registrosGlobais || [];
        const empresas = new Set();
        const placas = new Set();
        dados.forEach(reg => {
            if (reg.empresa) empresas.add(reg.empresa.trim());
            if (reg.placa) placas.add(reg.placa.trim());
        });
        document.getElementById('listaEmpresas').innerHTML = Array.from(empresas).sort()
            .map(e => `<option value="${escapeHtml(e)}"></option>`).join('');
        document.getElementById('listaPlacas').innerHTML = Array.from(placas).sort()
            .map(p => `<option value="${escapeHtml(p)}"></option>`).join('');
    }
```

Localizar o texto exato (topo de `window.renderizarViewAtual`, criado na Task 1):

```js
    window.renderizarViewAtual = function() {
        const viewAtiva = obterViewAtualDoHash();
```

Substituir por:

```js
    window.renderizarViewAtual = function() {
        atualizarDatalists();
        const viewAtiva = obterViewAtualDoHash();
```

- [ ] **Step 7: Verificar no navegador**

1. Recarregar o sistema já logado. `read_console_messages(onlyErrors: true)` → vazio.
2. Clicar no botão flutuante "+ Novo Registro" → modal abre, campo "Tipo Serviço" mostra "Selecione o tipo de serviço" (não "Lubrificação").
3. Via `javascript_tool`: `document.getElementById('tipoServico').checkValidity()` deve ser `false` (bloqueia envio sem escolha).
4. Selecionar um tipo → `checkValidity()` vira `true`.
5. Digitar as 3 primeiras letras de uma empresa real que já existe (ex: "Nut") no campo Empresa → conferir via `document.getElementById('listaEmpresas').innerHTML` que a opção com o nome completo está lá.
6. Clicar em "✕" no cabeçalho do modal → modal fecha, formulário limpa.
7. Ir numa pasta com registros reais, clicar em "Editar" (📝) numa linha → modal abre sozinho, já preenchido com os dados daquele registro.
8. Clicar no botão "X" de cancelar dentro do modal → modal fecha (esse mesmo caminho é o que confirma que o fechamento automático após salvar também funciona, porque `salvarRegistroFirebase` chama exatamente essa mesma `window.cancelarEdicao()` no sucesso — não é necessário criar um registro real de teste no banco pra confirmar isso).
9. Voltar na Visão Geral → conferir que não sobrou nenhum formulário solto ali, só resumo + gráficos.

- [ ] **Step 8: Commit**

```bash
git add index.html
git commit -m "$(cat <<'EOF'
transforma Novo Registro num modal flutuante com tipo obrigatorio e
autocomplete de empresa/placa

Resolve os dois problemas relatados pelo cliente: o campo Tipo Servico
nao vem mais pre-marcado em Lubrificacao (forca escolha ativa), e os
campos Empresa/Placa sugerem valores ja existentes via <datalist>,
reduzindo pasta duplicada por nome parecido. O formulario sai do meio da
tela (Visao Geral fica so com resumo + graficos) e vira um popup aberto
pelo botao flutuante "+ Novo Registro", reaproveitado tambem para editar
um registro existente.

Co-Authored-By: Claude Sonnet 5 <noreply@anthropic.com>
EOF
)"
```

---

### Task 7: Limpeza final, renome, e bateria completa de regressão

**Files:**
- Modify: `index.html` — renomeia `renderizarTabelaCompleta` → `renderizarVisaoGeral` (3 pontos), remove CSS morto (`.card-horizontal`, `.pasta-chip`, `.pastas-chips-container`).

**Interfaces:**
- Produces: `window.renderizarVisaoGeral()` (substitui `window.renderizarTabelaCompleta`, mesmo comportamento).

- [ ] **Step 1: Renomear `renderizarTabelaCompleta` → `renderizarVisaoGeral`**

Localizar o texto exato:

```js
    window.renderizarTabelaCompleta = function() {
```

Substituir por:

```js
    window.renderizarVisaoGeral = function() {
```

Localizar o texto exato (dentro do dispatcher, criado na Task 1):

```js
        if (viewAtiva === 'visao-geral' && typeof window.renderizarTabelaCompleta === 'function') {
            window.renderizarTabelaCompleta();
        }
```

Substituir por:

```js
        if (viewAtiva === 'visao-geral' && typeof window.renderizarVisaoGeral === 'function') {
            window.renderizarVisaoGeral();
        }
```

Localizar o texto exato:

```js
    window.filtrarDados = function() {
        window.renderizarTabelaCompleta();
    }
```

Substituir por:

```js
    window.filtrarDados = function() {
        window.renderizarVisaoGeral();
    }
```

(O `window.renderizarViewAtual` chamado pelo `onSnapshot` do Firestore, adicionado na Task 1 Step 4, não referencia `renderizarTabelaCompleta` diretamente — nada mais precisa mudar.)

- [ ] **Step 2: Remover CSS morto**

Localizar e apagar o texto exato:

```css
        .card-horizontal {
            background: var(--card-bg);
            padding: 12px 16px;
            border-radius: 6px;
            border: 1px solid var(--border);
            border-top: 2px solid var(--primary);
            margin-bottom: 12px;
        }

        .card-horizontal h2 {
            font-size: 0.9rem;
            margin-bottom: 10px;
            color: var(--text);
            font-weight: 600;
            font-family: 'Oswald', sans-serif;
            letter-spacing: 0.3px;
        }
```

Localizar e apagar o texto exato:

```css
        .pastas-chips-container {
            display: flex;
            gap: 6px;
            overflow-x: auto;
            padding-bottom: 4px;
            align-items: center;
        }

        .pasta-chip {
            background: rgba(255, 255, 255, 0.03);
            border: 1px solid var(--border);
            border-radius: 6px;
            padding: 4px 8px;
            font-size: 0.75rem;
            cursor: pointer;
            white-space: nowrap;
            display: flex;
            align-items: center;
            gap: 4px;
        }

        .pasta-chip:hover, .pasta-chip.ativo {
            border-color: var(--primary);
            color: var(--primary);
        }
```

(Não apagar `.btn-limpar-chip` — ainda é usado pelo botão "Limpar Filtros" dentro da aba Pastas.)

- [ ] **Step 3: Bateria completa de regressão (os 9 itens da seção 11 da spec), com dados reais**

Abrir o sistema já logado e, em sequência:

1. Cadastrar (ou tentar) um serviço pelo modal sem tocar no tipo → bloqueia o envio (native validation).
2. Digitar uma empresa parecida com uma já existente no campo Empresa do modal → autocomplete sugere a existente.
3. Abrir a aba "Pastas", conferir que a bolinha vermelha/amarela de cada card bate com o que a aba "Vencimento" mostra pra mesma empresa (comparar as duas abas lado a lado).
4. Entrar numa pasta com pendentes reais, conferir a seção "Pendentes" e o texto que `gerarMensagemCobranca` produziria (via console, sem clicar de fato no botão pra não abrir o WhatsApp).
5. Entrar numa pasta sem nenhum vencimento, conferir que não aparece nenhum aviso de vencido/urgente dentro dela.
6. Navegar direto pra `index.html#pasta/<empresa real>` colando a URL, confirmar que abre certinho; usar o botão "voltar" do navegador saindo de uma pasta e confirmar que volta pra lista.
7. Confirmar que a Visão Geral mostra os números/gráficos mensais exatamente como antes de toda essa reorganização (comparar com o screenshot já tirado na Task 1 Step 6).
8. Abrir duas pastas diferentes em sequência (ou a mesma duas vezes) e confirmar que os gráficos não travam nem geram "Canvas is already in use" no console.
9. Editar um registro de dentro de uma pasta, salvar, e confirmar que a pasta atualiza sozinha (sem precisar trocar de aba) assim que o Firestore confirma — **esse é o único passo desta bateria que grava de verdade no banco**; usar um registro já existente e reverter a edição depois (ou usar um campo inofensivo, como a Descrição) para não deixar lixo de teste nos dados do cliente.

Em todos os passos: `read_console_messages(onlyErrors: true)` deve continuar vazio.

- [ ] **Step 4: Commit final**

```bash
git add index.html
git commit -m "$(cat <<'EOF'
finaliza reorganizacao: renomeia renderizarTabelaCompleta para
renderizarVisaoGeral e remove CSS morto

Ultima etapa da reorganizacao em abas (Visao Geral / Pastas / Vencimento)
iniciada nas tasks anteriores. Sem mudanca de comportamento - so renome
e limpeza do que ficou sem uso (.card-horizontal, .pasta-chip,
.pastas-chips-container).

Co-Authored-By: Claude Sonnet 5 <noreply@anthropic.com>
EOF
)"
```

---

## Self-Review

**Cobertura da spec:** Navegação por hash (Task 1), Pastas lista+detalhe+bolinha (Task 2), gráficos da pasta sem duplicar canvas (Task 3), Pendentes+WhatsApp (Task 4), Vencimento completo e clicável (Task 5), modal com tipo obrigatório e autocomplete (Task 6), normalização de empresa + rename + regressão final (Task 7). Todos os itens do documento de spec têm uma task correspondente.

**Placeholders:** nenhum "TBD"/"implementar depois" — todo step tem código real.

**Consistência de tipos/nomes:** `window.abrirPasta(nomeEmpresa)`, `calcularUltimosVencimentosPorPlaca(dados)`, `calcularStatusPorEmpresa(dados)`, `montarLinhaHistoricoPasta(reg, ultimosVencimentos, hojePadrao)`, `construirDadosGraficos(totalPago, totalPendente, contagemTipos)`, `gerarMensagemCobranca(nomeEmpresa, pendentes)` são usados com a mesma assinatura em todas as tasks que os consomem.

**Correções aplicadas durante a escrita deste plano (antes de qualquer execução):**
- `calcularStatusPorEmpresa` e o lookup em `renderizarListaPastas` normalizam a chave de empresa (`trim().toLowerCase()`) dos dois lados — sem isso, duas grafias da mesma empresa dariam bolinhas inconsistentes.
- Todo `onclick` que precisava do nome da empresa passou a usar `data-empresa="${escapeHtml(...)}"` + `this.dataset.empresa`, em vez de interpolar a string direto dentro de aspas simples no JS do atributo — o jeito antigo quebrava se a empresa tivesse aspas simples ou duplas no nome (ex: "Transportes O'Brien").
