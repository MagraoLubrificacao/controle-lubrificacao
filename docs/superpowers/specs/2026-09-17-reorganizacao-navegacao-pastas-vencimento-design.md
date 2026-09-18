# Reorganização da navegação: Visão Geral / Pastas / Vencimento

Data: 2026-09-17
Arquivo afetado: `index.html` (aplicação de página única, sem build step, sem framework)

## 1. Contexto e problema

O sistema hoje (`index.html`) é uma única tela vertical: cards de alerta, resumo
financeiro, gráficos, formulário de cadastro sempre visível e a tabela completa de
todos os registros, tudo empilhado. Dois problemas relatados pelo cliente final:

1. O campo "Tipo Serviço" do formulário vem pré-selecionado em "Lubrificação", então
   ele frequentemente esquece de trocar ao cadastrar outro tipo de serviço.
2. A tela é poluída — tudo (form, filtros, tabela de todas as empresas/placas
   misturadas) aparece junto, dificultando achar o que interessa.

O dono do sistema (usuário desta conversa) quer reestruturar a navegação em abas,
separar as pastas (empresas) numa visão própria com detalhe por empresa, ter uma
aba dedicada a vencimentos, e adicionar um fluxo de cobrança via WhatsApp a partir
de uma pasta.

## 2. Objetivos

- Reduzir a poluição visual reorganizando o conteúdo em 3 destinos: **Visão Geral**,
  **Pastas**, **Vencimento**, mais um formulário de cadastro em popup.
- Impedir o erro de cadastro com tipo de serviço errado por padrão.
- Reduzir pastas duplicadas por nome parecido (autocomplete).
- Dar visibilidade de pendência (vencido/perto de vencer) tanto na lista de pastas
  quanto dentro de cada pasta.
- Permitir gerar uma cobrança (texto + WhatsApp) a partir dos pendentes de uma
  empresa.

## 3. Não-objetivos (fora de escopo)

- Nenhuma mudança de schema no Firestore (`registros_lubrificacao` continua com os
  mesmos campos: `empresa, placa, motorista, dataServico, tipoServico, prazoDias,
  descricao, proximaLubrificacao, proximaTrocaOleo, valor, statusPagamento,
  comprovanteUrl, comprovanteNome, criadoEm`).
- Não vamos cadastrar telefone/WhatsApp por empresa. O botão de cobrança abre o
  WhatsApp com o texto pronto e o usuário escolhe o contato manualmente
  (`https://wa.me/?text=...`), sem número pré-definido.
- Sem framework novo, sem bundler, sem dependência nova. Continua um único
  `index.html` com Firebase modular (como já é hoje).
- Sem paginação/infinite-scroll — o volume de dados do cliente é pequeno.

## 4. Navegação

Roteamento simples via `location.hash`, com um `hashchange` listener central que
decide qual container mostrar:

- `#visao-geral` (padrão, quando não há hash ou hash desconhecido)
- `#pastas` — lista de todas as pastas (empresas)
- `#pasta/<nomeEmpresaCodificado>` — detalhe de uma empresa (`encodeURIComponent`
  no nome ao montar o link, `decodeURIComponent` ao ler)
- `#vencimento` — lista completa de vencidos/urgentes

Uma barra de abas fixa no topo (Visão Geral | Pastas | Vencimento) fica destacada
conforme a aba ativa. O botão flutuante **"+ Novo Registro"** fica sempre visível
(`position: fixed`) em qualquer uma das telas e abre o modal de cadastro — ele não
faz parte do roteamento por hash (é um overlay).

Cada seção de conteúdo (`viewVisaoGeral`, `viewPastas`, `viewVencimento`) é um
`<div>` que passa a ser escondido/exibido via `style.display`, reutilizando o
padrão que o próprio arquivo já usa para `telaLogin`/`appConteudo`.

**Ponto de atenção:** hoje existe um único ponto de entrada de redesenho —
`window.renderizarTabelaCompleta`, chamado pelo listener do Firestore
(`onSnapshot`) toda vez que os dados mudam. Com 3 telas diferentes, esse hook
único vira uma função `renderizarViewAtual()` que olha o hash atual e chama
somente a função de renderização da tela ativa (Visão Geral, lista de Pastas,
detalhe de uma Pasta, ou Vencimento). É essa função — e não mais
`renderizarTabelaCompleta` diretamente — que o `onSnapshot` e o `hashchange`
passam a chamar. Sem essa troca, editar/excluir um registro estando dentro de
uma Pasta não atualizaria a tela sozinho (só atualizaria quando o usuário
trocasse de aba e voltasse).

## 5. Aba "Visão Geral"

Mantém exatamente o que já existe hoje nessa área: os seletores de mês/ano, os 3
números (Lucro do mês, Serviços no mês, Total Pendente Geral) e os dois gráficos
(Fluxo de Caixa Mensal, Tipos de Serviço). Não sai nada daqui, só a tabela geral e
o formulário deixam de ficar junto (eles se mudam para Pastas e para o modal,
respectivamente).

## 6. Aba "Pastas"

### 6.1 Lista (`#pastas`)

- Reaproveita a busca por empresa/placa e o filtro de status que já existem
  (`filterEmpresa`, `filterPlaca`, `filterStatus`).
- Cada card de pasta (uma por empresa) ganha um indicador de status. Importante:
  `calcularUltimosVencimentosPorPlaca` é indexada por **placa**, não por empresa —
  então não dá pra usar o mapa dela direto pra colorir um card de empresa. Nova
  função `calcularStatusPorEmpresa(dados)` faz o agrupamento: para cada placa do
  mapa de vencimentos, descobre a empresa dona (via `entry.lub.empresa` /
  `entry.oleo.empresa`) e guarda o **pior status** já visto pra aquela empresa
  (`vencido` > `urgente` > `ok`). Resultado: `{ [empresa]: 'vencido' | 'urgente' | 'ok' }`.
  - 🔴 (`vencido`) se qualquer placa da empresa tem lubrificação ou troca de óleo
    vencida (vigente, ou seja, o registro mais recente daquele tipo).
  - 🟡 (`urgente`) se nenhuma vencida, mas alguma vence em até 5 dias.
  - sem bolinha (`ok`) caso contrário.
- Clicar num card navega para `#pasta/<empresa>`.

### 6.2 Detalhe da pasta (`#pasta/<empresa>`)

Ao entrar, filtra `window.registrosGlobais` pela empresa (`reg.empresa ===
nomeEmpresa`) e monta:

1. **Cabeçalho**: nome da empresa + botão "◀ Voltar" (`#pastas`).
2. **Resumo** (todo o histórico da empresa, sem filtro de mês):
   - Quantidade de serviços (`length` dos registros filtrados).
   - Total ganho = soma de `valor` onde `statusPagamento === 'Pago'`.
   - Total a receber = soma de `valor` onde `statusPagamento === 'Pendente'`.
3. **Aviso de vencimento da pasta**: reaproveita `calcularUltimosVencimentosPorPlaca`
   restrito às placas dessa empresa, listando placa + tipo + "vencido há Xd" ou
   "vence em Xd" (mesmo texto/estilo já usado no painel de avisos atual).
4. **Gráficos da empresa**: os dois mesmos componentes de gráfico
   (`atualizarGraficos`-like), mas alimentados com os totais **all-time** da
   empresa (não do mês selecionado) — Pago x Pendente, e contagem por
   `tipoServico`. A view de Pasta usa **dois canvases fixos e únicos**
   (`chartBarrasPasta`/`chartPizzaPasta`, existem uma vez só no HTML, dentro do
   container da pasta) — ao entrar em empresas diferentes, os mesmos dois
   canvases têm seus dados atualizados (`instance.data = novoData;
   instance.update()`), nunca criando uma `new Chart()` num canvas que já tem
   instância ativa. Isso evita o erro clássico do Chart.js "Canvas is already in
   use" que aconteceria se cada empresa tentasse ter seu próprio canvas
   recriado a cada clique. A lógica de montagem do `data` é extraída para uma
   função compartilhada (recebe `totalPago, totalPendente, contagemTipos` e
   devolve os objetos `data` do Chart.js), reaproveitada tanto pela Visão Geral
   quanto pela Pasta, mas cada uma mantém sua própria variável de instância
   (`chartCaixaInstance`/`chartTiposInstance` para Visão Geral,
   `chartCaixaPastaInstance`/`chartTiposPastaInstance` para Pasta).
5. **Seção "Pendentes"**: lista só os registros dessa empresa com
   `statusPagamento === 'Pendente'`, mostrando data do serviço, tipo, placa e
   valor. Botão **"Gerar cobrança (WhatsApp)"** monta um texto assim:

   ```
   Olá! Segue o resumo dos serviços pendentes da <Empresa>:

   - 11/07/2026 - Lubrificação (placa ABC1D23): R$ 150,00
   - 20/07/2026 - Troca de Óleo (placa ABC1D23): R$ 300,00

   Total pendente: R$ 450,00
   ```

   e abre `https://wa.me/?text=<encodeURIComponent(texto)>` em nova aba (mesmo
   padrão de link externo que o botão de Google Agenda já usa hoje). Todo texto
   inserido (nome da empresa, placa, etc.) passa por `escapeHtml`/está apenas em
   texto puro — sem HTML aqui, então não há risco de injeção nesse ponto (é uma
   string de texto simples para a URL do WhatsApp).
6. **Tabela histórica completa** da empresa: reaproveita a mesma renderização de
   linha (`<tr>`) que a tabela de hoje já usa (empresa, placa, motorista, data,
   tipo, descrição, próx. lub., próx. óleo, valor, status, comprovante, ações),
   só que sem a coluna "Empresa" (já sabemos qual é, estamos dentro da pasta
   dela) e sem os filtros de busca (já estamos filtrados pela empresa).

## 7. Aba "Vencimento"

Mesma lógica de dados que os cards "🔴 Próximos Vencidos" / "🟡 Vencem nos
próximos 5 dias" de hoje (usando o mapa vindo de
`calcularUltimosVencimentosPorPlaca` sobre todos os registros, não só os
filtrados), mas:

- Sem limite de altura/scroll (hoje limitado a 80px) — lista inteira visível.
- Cada item (placa + empresa + tipo + dias) é clicável e navega para
  `#pasta/<empresa>`.

## 8. Modal "Novo Registro"

Passa a ser um popup aberto pelo botão flutuante — implementado como um `<div>`
overlay fixo (`position: fixed`, fundo escurecido, `display: none` por padrão),
**não** como `<dialog>` nativo. Motivo: `<dialog>` fecha sozinho com ESC/gesto de
"voltar" em vários navegadores mobile, o que perderia o que a pessoa tava
digitando sem aviso — o overlay fixo só fecha pelo botão "X"/Cancelar ou após
salvar com sucesso. Com os mesmos campos de hoje, mas:

- Agrupados visualmente em 3 blocos: **Identificação** (Empresa, Placa,
  Motorista), **Serviço** (Tipo, Data, Prazo, Descrição), **Financeiro** (Valor,
  Status, Comprovante).
- `<select id="tipoServico">` passa a ter uma primeira `<option value=""
  disabled selected>Selecione o tipo de serviço</option>`, removendo o
  `selected` que hoje está em "Lubrificação". Como o campo já é `required`, o
  navegador impede salvar sem escolha ativa.
- Campos Empresa e Placa ganham `<datalist>` (`list="empresasExistentes"` /
  `list="placasExistentes"`) preenchido a partir dos valores distintos já
  presentes em `window.registrosGlobais`, recalculado a cada snapshot do
  Firestore (mesmo lugar onde `renderizarSidebarPastas` já monta o `Set` de
  empresas hoje).
- O modal é reaproveitado tanto para "Novo Registro" quanto para "Editar
  Registro" (fluxo de edição que já existe hoje via `editarRegistro`), sem
  duplicar formulário.
- Ao salvar com sucesso, o modal fecha e, se o usuário estava numa pasta
  (`#pasta/<empresa>`) ou na lista de pastas, a tela recarrega os dados
  filtrados normalmente (o listener do Firestore já dispara
  `renderizarTabelaCompleta`/re-render da view ativa).

## 9. Fluxo de dados

Não muda nada na leitura/escrita do Firestore. O único dado novo é **derivado em
memória**, no cliente, a partir de `window.registrosGlobais` (que já existe):

- Mapa de últimos vencimentos por placa (`calcularUltimosVencimentosPorPlaca`,
  já implementado) — reaproveitado por Visão Geral (não muda), Pastas (bolinha +
  aviso dentro da pasta) e Vencimento (lista cheia).
- Lista de empresas distintas — já existe (`renderizarSidebarPastas`), passa a
  alimentar tanto os cards de Pastas quanto o `<datalist>` do modal.

## 10. Tratamento de erros

- Hash desconhecido ou `#pasta/<empresa-inexistente>` (ex.: empresa foi
  renomeada/apagada) cai de volta em `#pastas` com uma mensagem
  ("Pasta não encontrada").
- Botão "Gerar cobrança" com zero pendentes fica desabilitado (nada para
  cobrar).
- Mantém as validações que já existem hoje (tamanho de arquivo, campos
  obrigatórios) sem alteração.

## 11. Como será testado

Sem framework de testes automatizados no projeto (não existe hoje). A validação
será manual, no navegador, cobrindo:

1. Cadastrar um serviço pelo modal sem tocar no tipo → deve bloquear o envio
   pedindo a seleção.
2. Digitar uma empresa parecida com uma já existente → autocomplete sugere a
   existente.
3. Abrir `#pastas`, conferir bolinha vermelha/amarela batendo com o que a aba
   Vencimento mostra pra mesma empresa.
4. Entrar numa pasta com pendentes, gerar cobrança, conferir soma e texto
   gerado.
5. Entrar numa pasta sem nenhum registro vencido, conferir que a seção de aviso
   não aparece.
6. Testar navegação direta por URL (`file:///.../index.html#pasta/RodoNutri`)
   após login, e o botão "voltar" do navegador entre pastas.
7. Conferir que a Visão Geral continua mostrando os números/gráficos mensais
   exatamente como hoje (nada deve regressar aqui).
8. Abrir a pasta de uma empresa, depois abrir a pasta de outra empresa em
   seguida (sem recarregar a página) — os gráficos devem trocar de dados sem
   travar nem gerar erro no console (teste específico do risco do Chart.js
   descrito na seção 6.2).
9. Com uma pasta aberta, editar um registro dela pelo modal e confirmar que a
   pasta atualiza sozinha (sem precisar trocar de aba) assim que o Firestore
   confirma a gravação.

## 12. Riscos e mitigação

- **Nomes de empresa com variação de maiúsculas/espaços** poderiam duplicar
  pastas mesmo com autocomplete (ex.: "RodoNutri " vs "RodoNutri"). Mitigação:
  normalizar (`trim()`) a empresa ao salvar (já feito na correção anterior) e
  fazer o agrupamento de pastas por nome normalizado
  (`trim().toLowerCase()` para comparar, mantendo o texto original para exibir).
- **Regressão visual/funcional na Visão Geral** ao mover código: extrair com
  cuidado, testar lado a lado antes/depois com os mesmos dados reais.
- **Mensagem de cobrança muito longa** (empresa com dezenas de pendentes de
  uma vez): `wa.me` tem um limite prático de tamanho de URL. Para o volume
  esperado (uma transportadora com alguns caminhões) isso não deve ocorrer;
  não vamos criar um limite artificial agora, mas fica registrado como algo a
  revisitar se algum dia acontecer.
