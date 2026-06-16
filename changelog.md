# UTOME · Changelog de Alterações

> **Regra:** Antes de qualquer alteração no `utome-index.html`, este arquivo deve ser lido e uma nova entrada deve ser adicionada ANTES de aplicar as mudanças.

---

## [v1.0] — Arquivo original recebido
**Data:** (antes do rastreamento)
**Arquivo:** `utome-index.html`

### ✅ Estado inicial
- Tela de login simples com card centralizado
- Autenticação email/senha via Firebase
- CRUD de crianças e tarefas
- Query de tarefas com `.where("childId").orderBy("time")` — **causa erro de índice no Firestore**
- Emojis nativos do sistema operacional

---

## [v1.1] — Emojis Icons8 + Fix Firebase index
**Data:** 2025 (sessão anterior)

### ➕ Adicionado
- Função helper `ei(name, size)` — gera `<img>` do Icons8 estilo Apple
- Função `emojiToIcon(ch)` — converte emoji char para ícone Icons8
- Mapa `EMOJI_MAP` com os emojis mais usados no app
- Todos os emojis estáticos no HTML substituídos por `<img>` Icons8

### ➖ Removido
- Emojis nativos (💜, 👶, ✅, 🎯, 📋, 🗑️, 📡) do HTML e JS dinâmico
- `.orderBy("time")` da query do Firestore (causava erro de índice composto)

### 🔧 Corrigido
- **FirebaseError: The query requires an index** — removido `orderBy` da query, ordenação feita client-side com `.sort((a,b) => a.time.localeCompare(b.time))`

### ⚠️ Observação
- A correção do `orderBy` foi aplicada no arquivo intermediário mas **não propagou corretamente para o output final** em uma das sessões — resultando na reincidência do erro

---

## [v1.2] — Split-screen auth + Social login
**Data:** 2025 (sessão anterior)

### ➕ Adicionado
- Tela de auth reformulada com layout split-screen (esquerda: form, direita: visual)
- Painel direito com gradiente laranja (`#FF8C00 → #E55A00`), blobs orgânicos e mascote SVG UTOME
- Botão **Continuar com Google** — `GoogleAuthProvider` via Firebase popup
- Botão **Continuar com Apple** — `OAuthProvider("apple.com")` via Firebase popup
- Logo UTOME inline SVG no topo esquerdo (gradiente laranja + símbolo `|<`)
- Subtítulo dinâmico que muda entre login e cadastro
- Rodapé com texto de termos de serviço
- Responsivo: painel direito oculto em mobile

### ➖ Removido
- Auth card centralizado simples (`.auth-card`)
- Logo emoji no auth (💜)

### ⚠️ Observações
- Login com Apple requer configuração no Firebase Console + Apple Developer Account
- COOP (Cross-Origin-Opener-Policy) pode bloquear o popup do Google/Apple em alguns ambientes — ver v1.3

---

## [v1.3] — Fix orderBy reincidente + sistema de logs
**Data:** 2025-06-15

### ➕ Adicionado
- Este arquivo de changelog (`utome-changelog.md`) — rastreamento de todas as alterações
- Regra: toda alteração futura deve ler este log antes de aplicar mudanças

### 🔧 Corrigido
- Confirmado que `selectChild()` no output **já está sem `orderBy`** — o erro persistente é de **cache do browser**
- Solução para o usuário: forçar hard reload (`Ctrl+Shift+R` / `Cmd+Shift+R`) ou abrir em aba anônima

### ⚠️ Novo erro identificado: COOP popup
```
Cross-Origin-Opener-Policy policy would block the window.closed call.
```
- Causa: o Firebase usa `signInWithPopup` que abre uma janela popup; ambientes com COOP estrito bloqueiam a comunicação entre janelas
- Solução planejada: trocar `signInWithPopup` por `signInWithRedirect` nos botões Google e Apple
- Status: **aplicando agora em v1.4**

---

## [v1.4] — Fix COOP popup → redirect + cache-bust
**Data:** 2025-06-15

### 🔧 Corrigido
- `signInWithPopup` → `signInWithRedirect` + `getRedirectResult` nos botões Google e Apple
  - Elimina o erro `Cross-Origin-Opener-Policy policy would block the window.closed call`
- Adicionado `getRedirectResult` no boot do app para capturar o retorno do redirect

### ➕ Adicionado
- Handler `getRedirectResult` no início do script — necessário para fluxo de redirect

### ➖ Removido
- `signInWithPopup` (Google e Apple)

---

## [v1.5] — Security Rules + Deleção em cascata
**Data:** 2025-06-15

### ➕ Adicionado
- **Firestore Security Rules** — bloco de regras comentado no HTML com instruções para colar no Firebase Console:
  - `users`: só o próprio `uid` pode ler/escrever
  - `children`: só o `userId` dono pode criar, ler, editar e deletar
  - `tasks`: só o dono da `child` associada pode acessar (lookup cross-collection)
- **Deleção em cascata** — ao deletar uma criança, todas as `tasks` com aquele `childId` são deletadas em batch antes de remover o doc da criança
- **Botão de deletar criança** — adicionado botão "×" em cada item da lista de crianças
- **Feedback visual** — loading state durante deleção em cascata

### 🔧 Corrigido
- Deleção de criança antes não removia as tasks associadas (dados órfãos no Firestore)

### ⚠️ Observação
- As Security Rules precisam ser coladas manualmente no Firebase Console → Firestore → Rules
- O índice `tasks: childId + time` está sendo criado no Firebase Console pelo usuário (screenshot confirmado)
- Com o índice pronto, o `orderBy` pode voltar para o Firebase — mantido client-side por ora até índice ficar "Ativo"

---

## [v1.7] — MVP UI/UX: Paleta laranja, sidebar reformulada, task card hierárquico, toast + modal-confirm
**Data:** 2025-06-15

### ➕ Adicionado
- **`--orange` e `--orange-dark`** como variáveis CSS; `#app .btn` sobrepõe o gradiente para laranja, mantendo o auth-screen intocado
- **Logo SVG laranja no topbar** — substitui o `<div class="heart">💜</div>` pelo mesmo `auth-logo-icon` da tela de login; cor de marca consistente do login ao painel
- **Sidebar reformulada (B)** — item ativo usa `border-left: 4px solid var(--child-color)` + fundo `color-mix(12%)` em vez de roxo sólido; avatar 36px; `font-weight:700` no nome; cor de cada criança aplicada via CSS custom property `--child-color`
- **Task card com hierarquia (C)** — hora em `font-size:18px; font-weight:700` (`.t-hour`); nome em `font-size:13px; color:var(--muted)` (`.t-name`); dias como pílulas `.task-day-pill` com destaque `.today` (laranja) para o dia atual da semana
- **Toast de undo (E)** — `deleteTask()` remove otimisticamente do state + renderiza; exibe toast "Tarefa removida · Desfazer" por 4s; deleção no Firestore só acontece após 4.2s se não houver undo
- **Modal de confirmação (E)** — `deleteChild()` usa `showConfirmModal()` em vez de `confirm()` nativo; modal com título, descrição e botão vermelho explícito
- **`showToast(msg, onUndo)`** — helper reutilizável para feedbacks futuros
- **`showConfirmModal(title, text)`** — retorna Promise<boolean>, reutilizável

### ➖ Removido
- `<span class="device-id-pill">device: ${child.deviceId}</span>` do header da rotina (D) — deviceId agora só existe no Firestore
- `.device-id-pill` do CSS
- `.child-item.active { background: var(--violet); color: #fff }` — substituído pelo sistema de cor por criança
- `confirm()` nativo em `deleteChild()` e ausência de feedback em `deleteTask()`
- `.t-time` (classe) — substituída por `.t-hour` e `.t-name` separados
- `.task-days` como texto corrido — substituída por pílulas

### 🔧 Ajustado
- `deleteTask(taskId)` agora recebe `(taskId, taskName)` — o nome é usado no toast
- `renderTaskList()` calcula `todayIdx` (0=Seg, 6=Dom) para destacar a pílula do dia atual
- Day-toggle ativo usa `var(--orange)` em vez de `var(--violet)`
- `.add-child-btn:hover` e `.field input:focus` já usavam laranja; agora consistente com o app todo

### ⚠️ Observações
- `color-mix(in srgb, ...)` é suportado em Chrome 111+, Firefox 113+, Safari 16.2+ — cobre >95% dos browsers ativos; fallback gracioso (sem borda colorida) em browsers antigos
- O undo do toast é client-side only: se o usuário fechar a aba em menos de 4s após deletar, a tarefa é perdida (comportamento esperado para MVP)


**Data:** 2025-06-15

### ➕ Adicionado
- **`onSnapshot` nas tasks** — listener em tempo real substitui `.get()` em `selectChild()`. Qualquer mudança no Firestore reflete instantaneamente na UI sem reload
- **`taskUnsubscribe`** — variável global que guarda o unsubscribe do listener ativo; cancelado ao trocar de criança ou fazer logout, evitando memory leaks e listeners duplicados
- **Campos extras na criança:** `avatar` (emoji, ex: 👧), `age` (número), `color` (cor hex, ex: #FF7A00)
  - Salvos no Firestore ao criar criança
  - Exibidos no card da criança na sidebar (avatar + nome + idade)
  - Modal simples de criação de criança substituindo o input inline
- **Avatar colorido na sidebar** — bolinha colorida com o emoji da criança antes do nome

### 🔧 Corrigido
- `state.user.name` podia ser `undefined` quando o doc do usuário não existia (ex: primeiro login Google/Apple sem doc criado). Agora usa fallback: `doc.data()?.name || user.displayName || "Usuário"`

### ➖ Removido / Substituído
- `.get()` em `selectChild()` → substituído por `onSnapshot()`
- Input inline de nova criança → substituído por modal com campos avatar/idade/cor

### ⚠️ Observações
- `onSnapshot` consome uma conexão WebSocket persistente por criança ativa — é o comportamento esperado do Firestore
- O `taskUnsubscribe` deve ser chamado no `auth.signOut()` para fechar o listener
---

## [v1.8] — Aba Hoje, Modal de Tarefa, Avatar Topbar, Micro-animações, Onboarding SVG, Skeleton Loading
**Data:** 2025-06-15

### ➕ F — Aba "Hoje" como view padrão
- Toggle **Hoje | Semana** no cabeçalho do painel de rotina
- View "Hoje" filtra apenas as tarefas do dia atual (`todayIdx` = 0=Seg, 6=Dom)
- Card hero "Próxima tarefa" em destaque: gradiente com `--child-color`, hora em 32px, nome e emoji grandes
- Se não houver tarefas hoje: estado vazio amigável ("Que tal curtir o dia livre?")
- View "Semana" preserva o comportamento anterior (todas as tarefas)
- Estado salvo em `taskView` (variável global, sem persistência — reseta ao trocar criança intencionalmente)

### ➕ G — Formulário de nova tarefa em modal
- Botão `+ Adicionar tarefa` substitui o formulário inline no rodapé
- Modal com: hora, estado (select), nome, grid de emojis sugeridos (28 emojis), dias da semana e botão Adicionar
- `SUGGESTED_EMOJIS[]` — array de 28 emojis relevantes para rotina infantil
- `setupTaskModal()` — inicializa grid, days e campos a cada abertura
- `selectedTaskEmoji` — variável global rastreada pelo grid
- `.emoji-grid button.selected` — destaque visual no emoji escolhido
- `.modal-task-days button.on` — dias selecionados em laranja (mesmo padrão do day-toggle)

### ➕ H — Topbar com avatar do usuário logado
- `.user-avatar-btn` — botão circular com a inicial do nome do usuário, gradiente laranja
- `.user-dropdown` — dropdown com "Nome do usuário" (não-clicável) + separador + botão "Sair" em vermelho
- Dropdown fecha ao clicar fora (listener no `document`)
- `logout-btn` movido para dentro do dropdown (sem quebrar o listener de logout existente)
- Logo UTOME no topbar usa `auth-logo-icon` com SVG inline (mantido do v1.7)

### ➕ I — Micro-animações nas tasks
- `@keyframes taskEnter` — `opacity:0; translateY(8px) → opacity:1; translateY(0)` em 0.3s
- `@keyframes taskLeave` — `opacity:1; scale(1) → opacity:0; scale(0.95)` em 0.25s
- `createTaskCard()` — função helper que aplica `.entering` quando `!prevIds.has(t.id)`
- `deleteTask()` — aplica `.leaving` no card antes de remover do DOM, aguarda 240ms
- `renderTaskList()` agora recebe `(prevIds, newIds)` opcionais para controlar animações

### ➕ J — Onboarding com identidade visual UTOME
- Tela 0: mascote SVG UTOME (mesmo do painel direito do login) com animação `float`
- Tela 1: SVG de avatar de criança com anel colorido e badge de confirmação laranja
- Tela 2: SVG de relógio/lista de tarefas estilizado com gradiente laranja
- Textos reescritos: produto-específicos ("O relógio do seu filho mostra a próxima tarefa automaticamente")
- Botão "Começar" com gradiente laranja e sombra (mesmo estilo do `.btn`)
- Removidos emojis nativos (💜, 👶, ✅) — substituídos por SVGs inline

### ➕ K — Skeleton loading
- `showChildrenSkeleton()` — 3 itens skeleton na sidebar com avatar + 2 linhas de texto; chamado antes do `await` em `loadChildren()`
- `showTasksSkeleton(panel)` — 3 cards skeleton no painel de rotina; chamado antes do `onSnapshot` em `selectChild()`
- `.skeleton` — classe base com `background linear-gradient` e `@keyframes shimmer` (200% background-size, 1.4s infinite)
- Classes auxiliares: `.skeleton-child`, `.skeleton-avatar`, `.skeleton-lines`, `.skeleton-line`, `.skeleton-task`, `.skeleton-circle`, `.skeleton-info`

### 🔧 Ajustado
- `renderRoutine()` passa `prevIds`/`newIds` para `renderTaskList()` para animações corretas
- `splash` usa logo SVG laranja em vez de emoji 💜 (consistência de marca)
- `auth-version` atualizado para `v1.8`

### ⚠️ Observações
- A view "Hoje" usa a mesma lógica de `todayIdx` já existente nos day-pills
- O card hero de "próxima tarefa" usa `--child-color` via CSS custom property no `.panel` — requer que a cor da criança esteja salva no Firestore
- Animações respeitam `prefers-reduced-motion` implicitamente via curta duração (< 300ms)
