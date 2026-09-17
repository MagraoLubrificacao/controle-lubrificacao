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
- Cada card de pasta (uma por empresa) ganha um indicador de status calculado a
  partir de `calcularUltimosVencimentosPorPlaca` (função já existente,
  implementada na correção do vencimento fantasma) considerando **todas as placas
  daquela empresa**:
  - 🔴 se qualquer placa da empresa tem lubrificação ou troca de óleo vencida
    (vigente, ou seja, o registro mais recente daquele tipo).
  - 🟡 se nenhuma vencida, mas alguma vence em até 5 dias.
  - sem bolinha caso contrário.
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
   `tipoServico`. Para não duplicar `atualizarGraficos` (que hoje está acoplado
   aos ids `chartBarrasMensal`/`chartPizzaServicos` da Visão Geral), a
   implementação vai extrair a lógica de montagem do `data` do Chart.js para uma
   função compartilhada que recebe `(canvasEl, totalPago, totalPendente,
   contagemTipos)`, usada tanto pela Visão Geral quanto pela pasta (com
   `Chart` instances e canvases próprios da pasta).
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

Passa a ser um popup (`<dialog>` ou overlay fixo) aberto pelo botão flutuante,
com os mesmos campos de hoje, mas:

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

## 12. Riscos e mitigação

- **Nomes de empresa com variação de maiúsculas/espaços** poderiam duplicar
  pastas mesmo com autocomplete (ex.: "RodoNutri " vs "RodoNutri"). Mitigação:
  normalizar (`trim()`) a empresa ao salvar (já feito na correção anterior) e
  fazer o agrupamento de pastas por nome normalizado
  (`trim().toLowerCase()` para comparar, mantendo o texto original para exibir).
- **Regressão visual/funcional na Visão Geral** ao mover código: extrair com
  cuidado, testar lado a lado antes/depois com os mesmos dados reais.
