# UTOME · Changelog de Alterações

> **Regra:** Antes de qualquer alteração no `utome-index.html`, este arquivo deve ser lido e uma nova entrada deve ser adicionada ANTES de aplicar as mudanças.

---

## [v3.1] — Fase 5 Web App: duração de tarefas, alerta offline e forçar sincronização
**Data:** 2026-06-19

### ✅ Implementações

- **Campo duração nas tarefas** — novo campo numérico opcional (1–240 min) no modal de nova/editar tarefa. Salvo como `duration` (integer) no Firestore. Usado pelo ESP32 para countdown na tela. Se não preenchido, removido via `FieldValue.delete()` ao editar. Pré-populado corretamente no modal de edição.

- **Toast de alerta — dispositivo offline > 30 min** — `checkDeviceOfflineAlert(device)` chamado a cada update do `onSnapshot`. Verifica `status != "online"` e `lastSync` > 30 min atrás. Dispara `showToast()` com nome do dispositivo e tempo offline. Exibido uma vez por sessão por dispositivo (controle via `_offlineAlertShown`).

- **Botão "↻ Sincronizar"** no card do dispositivo — `forceDeviceSync(deviceId)` escreve `forceSyncAt: serverTimestamp()` em `/devices/{id}`. O ESP32 detecta via polling e sincroniza imediatamente. Registra auditoria `DEVICE_FORCE_SYNC`.

### 📋 Fase 5 MVP — status
- ✅ w1 — Expiração do pairingCode (TTL + timer visual)
- ✅ w2 — Status online/offline em tempo real (v3.0)
- ✅ w3 — Toast offline > 30min
- ✅ w4 — Campo duração nas tarefas
- ✅ w5 — Botão forçar sincronização

**Fase 5 completa. ✓**

---

## [v3.0] — Integração UTOME Gen 1 (ESP32-S3): dispositivos, pareamento e sincronização
**Data:** 2026-06-18

### 🤖 Estrutura Firestore — Novas coleções

- **`devices/{deviceId}`** — Representa um dispositivo físico UTOME Gen 1 vinculado a uma criança.
  - Campos: `childId`, `userId`, `name`, `status` ("online"/"offline"), `battery` (0–100), `firmware` (string de versão), `lastSync` (Timestamp), `linkedAt`, `unlinkedAt`.
  - Preparado para escrita direta pelo ESP32-S3 (atualiza `status`, `battery`, `lastSync`, `firmware` autonomamente via SDK Firebase para Arduino/ESP).

- **`pairingCodes/{code}`** — Códigos temporários de pareamento gerados pelo app web.
  - Campos: `code` (string UTM-XXXX-XXXX), `childId`, `userId`, `used` (boolean), `expiresAt` (Timestamp, TTL 10 minutos), `createdAt`.
  - O ESP32 consultará esta coleção para vincular-se à criança correta ao receber o código via teclado ou QR.

- **`syncLogs/{logId}`** — Histórico de sincronizações entre ESP32 e Firestore.
  - Campos: `deviceId`, `childId`, `userId`, `type`, `details`, `timestamp`.
  - Escrito pelo app web via `logDeviceSync()` e futuramente pelo próprio ESP32 a cada ciclo de sincronização.

- **Relacionamento completo documentado:** `users/{userId}` → `children/{childId}` (campo `userId`) → `devices/{deviceId}` (campos `childId` + `userId`) → `tasks` (campo `childId`).

### 🔐 Segurança Firestore — Regras atualizadas (arquivo `firestore.rules`)

```
// devices: responsável só acessa dispositivos vinculados aos seus próprios filhos
match /devices/{deviceId} {
  allow read, write: if request.auth != null
    && (resource == null || resource.data.userId == request.auth.uid)
    && (request.resource == null || request.resource.data.userId == request.auth.uid);
}

// pairingCodes: responsável só cria/lê seus próprios códigos; ESP32 pode ler (autenticado)
match /pairingCodes/{code} {
  allow create: if request.auth != null
    && request.resource.data.userId == request.auth.uid;
  allow read: if request.auth != null;
  allow update: if request.auth != null; // ESP32 marca used=true
  allow delete: if false;
}

// syncLogs: somente leitura/escrita do próprio usuário ou dispositivo autenticado
match /syncLogs/{logId} {
  allow create: if request.auth != null;
  allow read: if request.auth != null
    && resource.data.userId == request.auth.uid;
  allow update, delete: if false;
}
```

> **Atenção:** Colar as regras acima no Firebase Console → Firestore → Regras, integrando às regras existentes de `users`, `children`, `tasks` e `audit_logs`.

### 📱 Nova área "Meu UTOME" (sidebar)

- **Seção `#device-section`** adicionada ao painel de crianças (sidebar esquerda), exibida somente quando uma criança está selecionada.
- **Estado sem dispositivo:** mensagem informativa + botão "Conectar dispositivo" que abre o modal de pareamento.
- **Estado com dispositivo:** card compacto com nome do dispositivo, indicador de status online/offline (dot verde com glow animado para online), bateria com barra de progresso colorida (verde → amarelo → vermelho conforme nível), versão de firmware e tempo relativo da última sincronização ("agora mesmo", "há X min", "há Xh", "há X dias").
- **Ações no card:** "Renomear" (prompt inline, salva no Firestore) e "Desvincular" (modal de confirmação, zera `childId` no doc do device).
- Renderização por `renderDevicePanel()` — chamada via `onSnapshot` em tempo real; atualiza automaticamente quando ESP32 altera bateria ou status.

### 🔗 Fluxo de pareamento

- Função `generatePairCode()` — formato `UTM-XXXX-XXXX`, alfanumérico maiúsculo sem caracteres ambíguos (0/O/I/1).
- `createPairingCode(childId)` — cria doc em `pairingCodes` com TTL de 10 minutos (`expiresAt`), campos `code`, `childId`, `userId`, `used: false`.
- **Modal `#modal-pair`** — exibe o código gerado com countdown regressivo (mm:ss) até expiração; botão "Gerar novo código" cria novo doc e reinicia o timer; botão "Fechar" limpa o interval.
- `startPairTimer(expiresAt)` — atualiza `#pair-timer-display` a cada segundo; quando expirado, exibe "Expirado" e para.
- Log de auditoria `PAIR_CODE_GENERATED` / `PAIR_CODE_REGENERATED` a cada geração.

### 📊 Widget de dispositivo no dashboard

- **`#device-dashboard-widget`** inserido dinamicamente dentro do `#routine-panel` logo após o `.routine-header`, via `injectDeviceDashboardWidget(panel)` chamado no fim de `renderRoutine()`.
- Exibe: ícone do robô, nome do dispositivo, status (● Online / ● Offline com cor semântica), tempo da última sincronização e percentual de bateria.
- Visível apenas quando há dispositivo vinculado; oculto (`display:none`) caso contrário.
- Atualizado em tempo real pelo mesmo `onSnapshot` de `subscribeDevice`.

### 🔌 Preparação para sincronização ESP32-S3

- `subscribeDevice(childId)` — listener `onSnapshot` em `devices` filtrado por `childId` + `userId`; atualiza `deviceState.device` e re-renderiza painéis automaticamente. Cancela listener anterior ao trocar criança (`deviceState.unsubscribe()`).
- `logDeviceSync(deviceId, childId, type, details)` — utilitário para registrar eventos de sincronização em `syncLogs`.
- **Estrutura de tasks já pronta para leitura pelo ESP32:** `db.collection("tasks").where("childId","==",childId)` — o ESP32 autentica com Firebase SDK (service account ou anon auth com regras permissivas para o próprio dispositivo) e lê diretamente as tarefas da criança vinculada. Campos disponíveis: `name`, `emoji`, `time`, `days`, `color`.
- O ESP32 escreverá: `devices/{deviceId}.battery`, `devices/{deviceId}.status`, `devices/{deviceId}.lastSync`, `devices/{deviceId}.firmware`.

### 🔧 Integrações e ajustes internos

- `selectChild()` agora chama `subscribeDevice(childId)` imediatamente após atualizar `state.activeChildId`, antes de renderizar o skeleton.
- `logout-btn` listener limpa `deviceState.unsubscribe()` junto com `taskUnsubscribe`, evitando listeners órfãos após logout.
- `renderRoutine()` chama `injectDeviceDashboardWidget(panel)` ao fim, após `renderTaskList()`.
- `formatRelativeTime(date)` — helper novo para exibir datas relativas em português ("agora mesmo", "há N min", "há Nh", "há N dias").

### 🎨 CSS adicionado (sem quebrar estilos existentes)

- `.device-section`, `.device-section-title` — container da seção na sidebar.
- `.device-empty`, `.de-icon`, `.btn-pair-device` — estado vazio.
- `.device-card`, `.device-card-header`, `.device-icon-wrap`, `.device-name` — card com dispositivo.
- `.device-status-row`, `.status-dot.online`, `.status-dot.offline`, `.status-label` — indicador de status.
- `.device-metrics`, `.device-metric`, `.dm-label`, `.dm-value` — grid de métricas.
- `.battery-bar`, `.battery-fill`, `.battery-fill.low`, `.battery-fill.mid` — barra de bateria colorida.
- `.device-actions`, `.btn-device-sm`, `.btn-device-sm.danger` — botões de ação do card.
- `.pair-box`, `.pair-code-wrap`, `.pair-code`, `.pair-code-note`, `.pair-timer`, `.pair-actions` — modal de pareamento.
- `.device-dash-row`, `.device-dash-icon`, `.device-dash-info`, `.device-dash-name`, `.device-dash-meta`, `.device-dash-battery` — widget do dashboard.
- Suporte dark mode para `.device-card` e `.pair-box`.

### ⚠️ Observações e limitações desta fase

- O ESP32 ainda não existe fisicamente; toda a UI reflete o estado do Firestore. O modal de pareamento gera e persiste o código, mas a "escuta" do ESP32 pelo código ainda não está implementada no firmware.
- O campo `used` no `pairingCode` é reservado para o ESP32 marcar como `true` ao completar o vínculo — o web app não valida isso ainda.
- A autenticação do ESP32 no Firebase (anon auth vs. service account vs. custom token) será definida na fase de firmware — as regras de segurança propostas acima precisarão ser ajustadas conforme a escolha.
- `logDeviceSync()` está disponível mas não é chamado automaticamente nesta fase — será invocado pelo ESP32 ao escrever em `syncLogs`.

### 🧪 Validação

- Todos os blocos `<script>` validados sintaticamente.
- Balanceamento de `<div>` validado.
- Listener de logout limpa `deviceState.unsubscribe` — sem memory leaks.
- `subscribeDevice` cancela listener anterior ao trocar criança — sem listeners duplos.

---

## [v2.9] — Regras do Firestore corrigidas + Edição de tarefa + Aviso de limite (80%) + Copiar rotina entre filhos
**Data:** 2026-06-18

### 🔐 Regras do Firestore (novo arquivo `firestore.rules`)
- Criado arquivo separado `firestore.rules` com a regra revisada — **este arquivo não é parte do `index.html`**, precisa ser colado manualmente em Firebase Console → Firestore Database → Regras (ou via `firebase deploy --only firestore:rules` se usar Firebase CLI).
- **Corrigido:** adicionada regra para a subcoleção `users/{userId}/audit_logs/{logId}`, que faltava por completo na regra original. Sem essa regra, o Firestore nega tudo por padrão nesse caminho (nenhum `match` cobria), e toda chamada de `logAudit()` no `index.html` falhava silenciosamente (o código já ignora esse erro com `catch(_) {}`, então não quebrava a UX, mas os logs de segurança nunca eram gravados de fato). Agora: o próprio usuário pode criar/ler seus logs; ninguém pode editar ou apagar um log já gravado (imutabilidade).
- **Mantido sem alteração funcional, mas documentado:** a regra de `tasks` continua usando `get()/exists()` apontando para `children/{childId}` dentro da condição de uma query `.where()` — usuário confirmou que isso já foi testado e funciona em produção. Adicionado apenas um comentário no arquivo explicando que esse padrão é sensível a documentos "órfãos" (tarefa com `childId` de uma criança já deletada) e que migrar para subcoleção `children/{childId}/tasks/{taskId}` eliminaria essa dependência — registrado como melhoria futura, não aplicado agora para não exigir migração de dados existentes.

### ✏️ Edição de tarefas (horário, nome, ícone, estado, dias)
- Adicionado botão de editar (ícone de lápis) ao lado do botão de excluir em cada `task-card`, em `createTaskCard()`.
- `setupTaskModal()` agora aceita um parâmetro opcional `task` — quando fornecido, preenche todos os campos do modal (`#modal-task`) com os dados atuais da tarefa (horário, nome, emoji selecionado, estado/cor, dias da semana marcados) e altera o título do modal para "Editar Tarefa" e o texto do botão para "Salvar alterações".
- Nova função `openEditTaskModal(taskId)` — busca a tarefa em `state.tasks`, chama `setupTaskModal(task)` e abre o modal.
- `addTask()` renomeada para `saveTask()`: agora decide entre `db.collection("tasks").add(...)` (criar) ou `.doc(editingTaskId).update(...)` (editar) com base na variável de estado `editingTaskId`. O limite de recursos (`enforceResourceLimit`) só é verificado ao **criar**, nunca ao editar (editar uma tarefa existente não deve ser bloqueado pelo limite de 50 tarefas/criança).
- Como confirmado pelo usuário, hoje não existe conclusão de tarefa por dia (a tarefa é só a definição da rotina recorrente, sem histórico ainda) — por isso a edição é um `update()` simples e direto, sem nenhuma lógica de "afeta só o futuro vs. já concluído". Se no futuro for implementado o registro de conclusão diária, este ponto precisará ser revisitado.

### ⚠️ Aviso de limite (~80%) para crianças e tarefas
- Nova função `checkResourceWarning(type, newCount)` — não bloqueia nada (diferente de `enforceResourceLimit`, que bloqueia exatamente no limite), apenas mostra um toast informativo quando o responsável atinge 80% do limite (`Math.ceil(max * 0.8)`):
  - Crianças: avisa ao atingir 8 de 10 cadastradas.
  - Tarefas: avisa por criança, ao atingir 40 de 50 tarefas.
- Cada aviso aparece **no máximo uma vez por sessão** por tipo (e por criança, no caso de tarefas), controlado pela variável `_warnedLimits`, para não repetir o toast a cada tarefa criada depois do gatilho.
- Conectado em `saveTask()` (após criar tarefa) e no handler de `nc-save` (após criar criança).

### 👥 Copiar rotina entre filhos
- Novo modal `#modal-copy-routine` (HTML + CSS reaproveitando o estilo visual de `.modal-confirm-box` já existente) — lista todos os outros filhos cadastrados com checkbox, permitindo copiar para um ou mais de uma vez.
- Novo botão "Copiar rotina" no cabeçalho do painel de rotina (`routine-header`), ao lado do toggle Hoje/Semana — abre o modal para a criança atualmente selecionada.
- `openCopyRoutineModal(sourceChildId, sourceChildName)`: monta a lista de destinos disponíveis; se não houver outra criança cadastrada, mostra toast informativo em vez de abrir o modal vazio.
- Handler de confirmação: busca todas as tarefas da criança de origem (`db.collection("tasks").where("childId","==",sourceChildId).get()`), e para cada criança de destino selecionada, calcula quantos slots ainda restam até o limite de 50 tarefas (`UTOME_SEC.LIMITS.MAX_TASKS_PER_CHILD`), copia apenas até esse limite via `batch.set()` em lote, e avisa via toast se algum destino já estiver no limite. Cada cópia gera um log de auditoria `COPY_ROUTINE` com origem, destino e quantidade copiada.

### 🧪 Validação
- Todos os blocos `<script>` validados sintaticamente via parser real (`new Function()`), sem erros.
- Balanceamento de `<div>` validado em todo o `<body>` antes do script (113 abertas / 113 fechadas).
- Testado visualmente via Playwright com dados simulados (mock de Firebase, já que o ambiente de teste não tem acesso à rede): confirmado que o botão "Copiar rotina" aparece corretamente no cabeçalho, o modal de cópia lista as outras crianças com checkbox, e o modal de edição abre pré-preenchido com horário/nome/emoji/estado/dias corretos da tarefa selecionada, com título e botão trocados para o modo de edição.

---

## [v2.8] — CAUSA RAIZ CONFIRMADA e corrigida: login Google/Apple voltava para tela de login (storage partitioning)
**Data:** 2026-06-18

### 🔍 Investigação completa solicitada pelo usuário
- Auditoria linha a linha de todo o `<script>` principal (mais de 1300 linhas), desde `setPersistence` até `onAuthStateChanged`, incluindo ordem de execução, race conditions e todos os pontos que tocam Firestore no caminho pós-login.
- Hipóteses descartadas durante a investigação: erros do F12 enviados pelo usuário (são ruído do próprio `accounts.google.com`/ad-blocker, sem relação com o site); domínio ausente em Authorized Domains (usuário confirmou que já estava configurado); falta de try/catch no `onAuthStateChanged` (corrigido na v2.7, mas o sintoma persistiu); ordem entre `setPersistence` e `onAuthStateChanged` (tecnicamente frágil, mas não a causa raiz).

### ✅ Causa raiz real (confirmada na documentação oficial do Firebase, consultada nesta sessão)
- Fonte: [firebase.google.com/docs/auth/web/redirect-best-practices](https://firebase.google.com/docs/auth/web/redirect-best-practices)
- O fluxo `signInWithRedirect()` depende de um **iframe cross-origin** que o SDK do Firebase carrega a partir do `authDomain` (no caso, `tomi-ia.firebaseapp.com`) para recuperar o resultado do login após o redirect de volta.
- **Desde 24 de junho de 2024, isso é bloqueado por padrão no Google Chrome M115+** (e já era bloqueado antes no Firefox 109+ e Safari 16.1+) em qualquer app que **não esteja hospedado no próprio domínio `*.firebaseapp.com`** — que é exatamente o caso do UTOME, hospedado na Vercel.
- Resultado prático: o usuário autentica corretamente no popup do Google, mas ao retornar para o domínio da Vercel, o navegador bloqueia o acesso ao armazenamento (cookies/IndexedDB) que o iframe do Firebase usaria para repassar a sessão — o Firebase então reporta `user = null` no `onAuthStateChanged`, e o app exibe a tela de login novamente, **mesmo com o try/catch da v2.7 funcionando perfeitamente** (porque tecnicamente não houve erro — o Firebase genuinamente não recebeu o usuário autenticado).

### 🔧 Corrigido
- `signInWithRedirect(provider)` → `signInWithPopup(provider)` para **Google** e **Apple**. Essa é a correção oficialmente recomendada pelo Firebase como alternativa mais simples ao invés de reconfigurar `authDomain`/proxy/self-host (que exigiriam domínio customizado e mudanças de infraestrutura na Vercel).
- Removido `auth.getRedirectResult().then(...)` — não é mais necessário, já que `signInWithPopup` resolve a Promise diretamente no clique do botão, sem depender de reload de página nem do iframe cross-origin.
- Tratamento de erro nos novos handlers: erros esperados de UX (`auth/popup-closed-by-user`, `auth/cancelled-popup-request` — usuário fechou o popup ou clicou duas vezes) não mostram mensagem de erro; qualquer outro erro é logado no console com `console.error("[UTOME] Erro no login com Google/Apple:", e)` e mostra mensagem amigável.
- `logAudit("LOGIN_SUCCESS", { method: "google"/"apple" })` chamado diretamente após o popup resolver (antes dependia do redirect + getRedirectResult, que não tocava nesse log).

### ⚠️ Trade-off consciente
- Popups podem ser bloqueados por configurações agressivas de bloqueio de pop-up do navegador (raro, mas existe) e a experiência em mobile é levemente menos fluida que um redirect nativo. Esse é o trade-off reconhecido pela própria documentação do Firebase ao recomendar essa opção como a mais simples diante de hospedagem fora do domínio `firebaseapp.com`.
- Caso no futuro o domínio customizado do UTOME seja apontado via Firebase Hosting (em vez de/além da Vercel), vale reavaliar a Option 1 da documentação (usar o domínio customizado como `authDomain`), que permitiria voltar a usar `signInWithRedirect` com a experiência de navegação completa, sem popup.

---

## [v2.7] — Fix: login Google autentica mas volta para tela de login (falha silenciosa no onAuthStateChanged)
**Data:** 2026-06-17

### 🐛 Bug confirmado
- Após a remoção do App Check (v2.6), o problema de "autentica no Google mas a página volta para o login" persistiu — usuário reportou erros no console do navegador (F12) relacionados a `accounts.youtube.com`, CSP `frame-ancestors` e `net::ERR_BLOCKED_BY_CLIENT` em `play.google.com/log`.
- **Análise dos erros do F12:** são ruído do próprio ambiente OAuth do Google (página interna do account chooser) e de um ad-blocker/extensão do navegador bloqueando telemetria do Google — não têm relação com o UTOME e não explicam o bug. Confirmado que o domínio de teste já estava na lista de Authorized Domains do Firebase Auth, eliminando essa causa comum.

### 🔍 Causa raiz real
- `auth.onAuthStateChanged(async (user) => {...})` não tinha **nenhum `try/catch`** envolvendo as chamadas `await db.collection("users").doc(user.uid).get()` / `.set()`.
- Se essa leitura/escrita no Firestore falhasse por qualquer motivo (regra de segurança, latência logo após o redirect, erro de permissão), a `Promise` rejeitada lançava uma exceção não tratada **dentro do callback**, interrompendo a execução antes de chegar em `showApp()`. Como a tela de login (`#auth-screen`) já começa com `display:flex` por padrão, o resultado visual era "nada muda" — exatamente como voltar para a tela de login, mas na real o app trava silenciosamente no meio do carregamento pós-autenticação, sem nenhum erro visível na UI.
- Adicionalmente, **tanto `getRedirectResult()` quanto `onAuthStateChanged()` faziam a mesma leitura/escrita no Firestore em paralelo** assim que o usuário retornava do redirect do Google — duas operações concorrentes no mesmo documento, redundantes e potencial fonte de condição de corrida.

### 🔧 Corrigido
- `onAuthStateChanged`: todo o bloco do `if (user) {...}` agora está dentro de um `try/catch`. Em caso de falha ao carregar/criar o doc do usuário: loga o erro real no console (`console.error`), libera o splash (`firebaseResolved = true; tryHideSplash()`), desativa o loading, mostra mensagem de erro visível no card de login (`"Não foi possível carregar sua conta. Tente novamente em alguns instantes."`) e garante `showAuth()` — ou seja, o usuário agora **vê um erro real** em vez de simplesmente cair de volta numa tela de login "muda".
- `getRedirectResult()`: removida a leitura/escrita duplicada no Firestore. Agora apenas captura e loga erros do próprio fluxo de redirect (popup fechado, conta cancelada, domínio não autorizado); a criação/leitura do doc do usuário ficou centralizada e protegida só em `onAuthStateChanged`.

### ⚠️ Próximos passos se o problema persistir
- Abrir o F12 → Console **imediatamente após** tentar logar com Google e procurar por uma linha começando com `[UTOME] Falha ao carregar dados do usuário após login:` — ela vai conter o erro real do Firestore (ex: `permission-denied`, `unavailable`, etc.) e aponta exatamente a causa.
- Se o erro for `permission-denied`, o próximo ponto a revisar são as **Firestore Security Rules** publicadas no Firebase Console (documentadas desde a v1.5) — regras mal configuradas para a coleção `users` bloqueariam exatamente esse `get()/set()` no primeiro login.

---

## [v2.6] — Remoção completa do App Check / reCAPTCHA v3 (fix login Google travando)
**Data:** 2026-06-17

### 🐛 Bug relatado
- Botão de login social demorando muito para abrir o seletor de conta Google.
- Após escolher a conta Google e autenticar, o app voltava para a tela de login em vez de abrir o painel — sugeria falha silenciosa em algum ponto entre `signInWithRedirect` e `showApp()`.

### 🔍 Diagnóstico
- O script `firebase-app-check-compat.js` era carregado de forma síncrona/bloqueante antes da inicialização do app, adicionando uma chamada de rede extra antes de qualquer outra coisa rodar — possível causa da demora percebida ao abrir o popup/redirect.
- O código chamava `firebase.appCheck()` (instanciando o serviço) mesmo com `RECAPTCHA_KEY` ainda como placeholder (`'SUA_CHAVE_RECAPTCHA_AQUI'`) e sem `.activate()` ser executado. Caso o projeto tenha "Enforce App Check" habilitado no Firebase Console para Firestore/Auth sem o token sendo emitido, as chamadas pós-login (`db.collection("users").doc(uid).get()/.set()` dentro do `onAuthStateChanged`) podem ser rejeitadas pelo servidor sem erro visível na UI, fazendo o fluxo nunca chegar em `showApp()` e cair de volta para a tela de auth.

### 🔧 Removido
- `<script src=".../firebase-app-check-compat.js">` — removido do carregamento de scripts.
- Bloco de inicialização do App Check (`try { const RECAPTCHA_KEY = ...; firebase.appCheck().activate(...) } catch...`) — removido por completo do `<script>` principal.
- Menções a "reCAPTCHA" e "App Check" no texto da **Política de Privacidade** (seção 1.A "dados coletados" e seção 3 "compartilhamento e transferência") e nos **Termos de Serviço** (seção 3 "uso aceitável") — ajustadas para não declarar uma proteção que deixou de existir no sistema.

### ✅ Mantido (não é App Check, não foi tocado)
- `RecaptchaVerifier` invisível (`#recaptcha-mfa`) usado exclusivamente no fluxo de **MFA por telefone** (`getMfaRecaptcha()`, `phoneAuthProvider.verifyPhoneNumber`) — é uma exigência própria do Firebase Auth para 2FA via SMS, mecanismo totalmente independente do App Check/reCAPTCHA v3 que foi removido.
- `signInWithRedirect` para Google/Apple, `getRedirectResult`, e toda a lógica de `onAuthStateChanged`/`tryHideSplash` — inalterados.

### ⚠️ Observações
- Se o login social ainda apresentar lentidão ou loop de volta para a tela de auth após este fix, o próximo ponto a checar é a configuração do Firebase Console: **Authentication → Sign-in method → Google/Apple** (domínios autorizados, `authDomain` correto) e, principalmente, se **App Check → Enforce** está ativado para Firestore/Auth no Console — se estiver, precisa ser desativado manualmente lá também, já que removê-lo só do código não desliga a exigência do lado do servidor.
- Caso a proteção contra bots/scraping seja necessária no futuro, recomenda-se reimplementar com a chave reCAPTCHA real já configurada e testada em ambiente de homologação antes de subir para produção, evitando o cenário de "serviço instanciado sem token válido".

---

## [v2.5] — Reformulação completa da tela de Login (layout de referência + inputs/botões brancos)
**Data:** 2026-06-17

### 🎨 Redesign
- **Novo layout de duas colunas**, inspirado em imagem de referência fornecida pelo usuário:
  - **Coluna esquerda** (`.auth-left`): logo `utome.png` + nome da marca, headline "Automação **inteligente** para crescer sem limites." com destaque em laranja, subtítulo institucional, e mascote (`utome.png`) flutuando sobre um glow radial laranja (`.auth-mascot-glow`). Visível **apenas no tema claro** (`[data-theme="dark"] .auth-left{display:none;}`), por decisão do usuário.
  - **Coluna direita** (`.auth-right` / `.auth-card`): card branco flutuante com sombra suave, título "Bem-vindo de volta! 👋", subtítulo com destaque "utome" em laranja, campos de Email/Senha, botão "Continuar", divisor "OU", botões sociais Google/Apple, link de cadastro e rodapé legal.
- **Logo:** removido o antigo `.auth-logo-icon` com gradiente/box-shadow envolvendo um ícone pequeno — agora usa `utome.png` diretamente (caminho relativo já existente no projeto), em tamanho maior (40px), sem container colorido.

### ⚪ Inputs e botões agora brancos (claro E escuro)
- `.field input,.field select` — fundo `#fff` fixo com texto `#16151F` (quase preto), inclusive no tema escuro (`[data-theme="dark"] .field input{background:#fff;color:#16151F;}`) — substitui o fix anterior da v2.4 que usava `var(--card)`.
- `.btn-social` (Google/Apple) — fundo `#fff` fixo com texto escuro em ambos os temas (`[data-theme="dark"] .btn-social{background:#fff;color:#16151F;}`).
- Card (`.auth-card`) continua usando `var(--card)` para se adaptar ao tema, mas os controles internos (inputs/botões) permanecem brancos conforme solicitado.

### 🧊 Efeito 3D nos botões
- `.btn` ("Continuar"): gradiente vertical de 3 tons (`#FF9433 → #FF7A00 → #F06A00`) + `box-shadow` combinando highlight interno superior, sombra interna inferior e sombra externa projetada — simula relevo/profundidade. Hover eleva o botão (`translateY(-2px)`) e intensifica a sombra; active recolhe.
- `.btn-social`: sombra sutil inset + drop-shadow externa para leve elevação; hover sobe 1px e aumenta a sombra.

### 🔧 Ajustado
- `#auth-screen` — de `display:flex` com painel lateral colorido (gradiente animado) para layout centralizado com `gap` entre as colunas, fundo neutro claro (`#FBFAF8`) que vira `var(--bg)` no escuro.
- Removidos: `.auth-right` com gradiente animado (`gradientMove`), `.auth-blob-1/2`, `.auth-version` (selo de versão sobre o painel) — não fazem parte do novo layout de referência.
- `auth-toggle` (JS): trocado `textContent` por `innerHTML` ao alternar Login/Cadastro, para preservar o emoji e o `<span class="accent">` nos textos de título/subtítulo do card.
- Responsivo (`max-width:900px`): coluna esquerda (headline + mascote) oculta, mantendo apenas o card de login centralizado em telas pequenas.

### ⚠️ Observações
- O efeito de glow do mascote (`.auth-mascot-glow`) é puramente decorativo (radial-gradient), sem dependência de imagem adicional.
- Como a coluna esquerda fica oculta no tema escuro, o tema escuro exibe somente o card de login centralizado na tela.

---

## [v2.4] — Fix: campos de Email/Senha "pretos" no tema escuro (sem contraste)
**Data:** 2026-06-17

### 🐛 Bug identificado
- No tema escuro, os campos `.field input` e `.field select` (Email, Senha, etc.) usavam `background:#1A1928` fixo — exatamente o mesmo valor hexadecimal de `--bg` no tema escuro (`[data-theme="dark"]{--bg:#1A1928;...}`).
- Resultado: o input se misturava visualmente com o fundo da página, criando o efeito de "caixa preta sólida" sem nenhum contraste perceptível de campo de formulário (placeholder e borda ficavam quase invisíveis).
- No tema claro o comportamento era correto: o input usa `#FAFAFE`, um tom levemente diferente do `--bg:#F6F4FB`, mantendo a hierarquia visual.

### 🔧 Corrigido
- `[data-theme="dark"] .field input,[data-theme="dark"] .field select` — `background:#1A1928` (cor fixa = `--bg`) trocado para `background:var(--card)` (`#242337`), restaurando o contraste entre o campo e o fundo da página, espelhando a lógica já usada no tema claro.
- **Nota:** esta correção foi substituída na v2.5, que passou a usar fundo branco fixo nos inputs em ambos os temas, por pedido explícito do usuário.

---

## [v2.3] — Documentos Legais Integrados (Política de Privacidade e Termos de Serviço)
**Data:** 2026-06-16

### 📝 Adicionado / Implementado
- **Arquivo de políticas ([politica.md](file:///c:/Users/sinua/Downloads/utome-main/politica.md)):** Criação de documento legal contendo a Política de Privacidade (conforme com a LGPD e consentimento específico para dados de menores) e os Termos de Serviço do UTOME em português.
- **Modal de Documentos Legais (`#modal-legal`):** Criação de um modal responsivo e moderno no `index.html` com abas dinâmicas para navegação entre a Política de Privacidade e os Termos de Serviço diretamente na plataforma.
- **Design integrado e responsivo:** Estilização utilizando variáveis CSS globais do UTOME (`--card`, `--ink`, `--line`, `--orange`), garantindo suporte automático e contraste adequado no Modo Escuro, além de efeito translúcido de fundo (`backdrop-filter: blur(6px)`).

### 🔧 Ajustado
- **Links no rodapé de Login/Registro (`.auth-footer`):** Substituição do texto estático por links interativos que ativam o modal diretamente na aba selecionada pelo usuário.

---

## [v2.2] — Camada de Segurança Completa (MFA, Rate Limit, Anomaly, Resource Limits)
**Data:** 2026-06-16

### 🔒 Alta prioridade — reforço e consolidação
- **CSP aprimorada (`<meta>` + `vercel.json`):** `base-uri`, `form-action`, `object-src`, `frame-ancestors 'none'` (headers); clickjacking bloqueado via `X-Frame-Options: DENY`
- **HTTPS Enforcement (client-side):** redirect automático `http → https` fora de localhost
- **HSTS:** mantido em `vercel.json` (`max-age=31536000; includeSubDomains; preload`)
- **Módulo `UTOME_SEC`:** camada centralizada com `sanitizeText`, `validateEmail`, `validatePassword`, `validateImageFile`, `esc()` (escape HTML), `checkRateLimit`, `recordAction`, `detectAnomaly`, `enforceResourceLimit`, `initSessionSecurity`
- **Input Validation + Sanitization:** todos os campos de texto passam por validação tipada antes do Firestore; renderização DOM usa `esc()` contra XSS refletido/armazenado
- **Secure File Upload:** helper único com whitelist MIME, limite 500KB, reset do input após rejeição; `accept` restrito no HTML
- **Enumeration Protection:** mensagens genéricas em login/registro/social/MFA; sem vazamento de existência de e-mail
- **Session Management:** persistência `LOCAL`; `getIdToken(true)` no login para rotacionar token
- **Session Fixation Protection:** `sessionStorage` com `utome_session_id` (UUID) por sessão; limpeza no logout
- **Audit Logging ampliado:** `DELETE_TASK`, `RATE_LIMIT_HIT`, `ANOMALY_*`, `MFA_*`, `RESOURCE_LIMIT`; inclui `sessionId` e contagem de ações

### 🔒 Média prioridade — implementado
- **MFA / 2FA (Firebase Phone):** seção no modal de perfil para cadastro SMS; modal de verificação no login quando `auth/multi-factor-auth-required`
- **Rate Limiting:** janela deslizante via `localStorage` — login (5/15min), registro (3/h), criação criança/tarefa, upload
- **Bot Protection:** App Check (reCAPTCHA v3) + botões desabilitados via `withGuard()` + `RecaptchaVerifier` no fluxo MFA
- **Anti-Scraping:** App Check + detecção de `navigator.webdriver` (audit log)
- **Resource Limiting:** máx. 10 crianças por conta, 50 tarefas por criança, validação de tamanho base64 no save de foto
- **Anomaly Detection:** monitora >40 ações/min, >5 falhas de login, spikes de upload rejeitado — gera audit `CRITICAL`/`WARNING`

### ⚠️ Configuração manual necessária
- Registrar **reCAPTCHA v3** no Firebase Console → App Check (`RECAPTCHA_KEY` no HTML)
- Habilitar **Multi-Factor Authentication (Phone)** no Firebase Console → Authentication → Sign-in method
- Colar **Firestore Security Rules** (documentadas em v1.5) no Firebase Console

---

## [v2.1] — Sistemas de Segurança (CSP, Sanitização, Vercel, App Check, Audit)
**Data:** 2026-06-16

### 🔒 Adicionado / Implementado
- **Vercel Security Headers (`vercel.json`):** Adicionado arquivo de configuração de hospedagem para injetar headers de proteção: Content-Security-Policy (CSP), X-Frame-Options (Clickjacking), HSTS (HTTPS Enforcement) e X-Content-Type-Options.
- **CSP `<meta>` tag:** Adicionada tag Content-Security-Policy direto no HTML.
- **Input Sanitization e Validação:** Adicionado DOMPurify via CDN (v3.0.3). Todos os inputs de texto do usuário (nome, nick, nome da tarefa) são purificados antes da inserção no Firestore para prevenir Stored XSS.
- **Secure File Upload:** Validação restrita de tipo de arquivo (`image/jpeg`, `image/png`, `image/webp`) e tamanho máximo (500KB) para as fotos de perfil e das crianças, prevenindo abuso de armazenamento.
- **Session Management e Anti-Enumeração:** Persistência configurada ativamente (`LOCAL`) e erros de autenticação exibem mensagens genéricas ("Credenciais inválidas") para evitar o vazamento da existência ou não de e-mails no banco de dados.
- **Audit Logging:** Função `logAudit(action, details, severity)` implementada para registrar eventos importantes (login, criações e deleções) em `/users/{uid}/audit_logs/` do Firestore.
- **Rate Limiting & Bot Protection:** Botões entram em estado "disabled" durante o processamento para evitar submissões múltiplas.
- **Firebase App Check (reCAPTCHA):** Lógica do App Check inicializada no boot do app. O reCAPTCHA protegerá as APIs do Firebase contra requisições não autorizadas, scraping e bots (exige cadastro de chave do reCAPTCHA no Firebase Console).

---

## [v2.0] — Fix tema escuro no modal + Fix Google login redirect
**Data:** 2026-06-16

### 🔧 Corrigido — Tema escuro

- `.modal-task-box` e `.modal-confirm-box` usavam `background:#fff` fixo → trocado para `var(--card)`
- `.modal-task-days button` usava `background:#fff` fixo → trocado para `var(--card)`
- `.btn-modal-cancel` e `.btn-cancel-modal` usavam `background:none` → trocado para `var(--card)` + `color:var(--ink)` explícito
- Dark overrides adicionados: `.emoji-grid button`, `.day-toggle button`, `.add-task-floating:hover`
- Resultado: todos os textos e controles do modal ficam visíveis no tema escuro

### 🔧 Corrigido — Google login com `signInWithRedirect`

**Causa do bug:** race condition entre o `setTimeout` de 2s do splash e a inicialização assíncrona do Firebase após o redirect do Google. O timer expirava com `auth.currentUser` ainda `null`, exibindo a tela de auth antes do `onAuthStateChanged` resolver.

**Solução:** substituição do `setTimeout` simples por um sistema de duas flags (`firebaseResolved` + `splashTimerDone`). O splash só some quando ambas são `true`. O `onAuthStateChanged` sinaliza `firebaseResolved = true` e decide a navegação (app ou auth/onboarding) — eliminando a decisão do timer.

- `tryHideSplash()` — função que aguarda as duas condições antes de esconder o splash
- `onAuthStateChanged` agora é a única fonte de verdade para navegação pós-autenticação
- Lógica do onboarding movida para dentro do `onAuthStateChanged` (branch `user === null`)
- `getRedirectResult` mantido apenas para criação do doc Firestore em novos usuários

---

## [v1.9] — Perfil do Responsável + Icons8 Apple + Upload de Foto (Criança e Usuário)
**Data:** 2025-06-15

### ➕ Adicionado
- **Modal de Perfil do Responsável** — acessível via dropdown "Meu Perfil"
  - Upload de foto de perfil (base64, armazenado no Firestore em `users/{uid}.photoURL`)
  - Campo **Nome Completo** editável
  - Campo **Como o UTOME te chama** (`nick`) — aparece no "Olá, [nick]" do topbar
  - **Seletor de Tema** — Claro, Escuro, Sistema (persiste em `localStorage("utome_theme")`)
  - Avatar do topbar atualiza para foto ao salvar; fallback para inicial do nome
- **Sistema de Tema Claro/Escuro/Sistema**
  - `[data-theme="dark"]` no `<html>` com variáveis CSS invertidas
  - `applyTheme(pref)` — aplica tema e persiste preferência
  - Escuta `prefers-color-scheme` para modo Sistema em tempo real
  - Dark overrides para `.panel`, `.field input`, `.modal-*`, `.skeleton`, `.user-dropdown`
- **Upload de foto para crianças** no modal "Nova Criança"
  - Botão "Enviar foto" com input `<file accept="image/*">`
  - Preview ao vivo do avatar (base64 → `nc-avatar-preview`)
  - Campo emoji mantido para quem prefere emoji em vez de foto
  - `photoURL` salvo no Firestore em `children/{id}`
  - `renderChildren()` agora prioriza `photoURL` sobre emoji no avatar da sidebar
- **Item "Meu Perfil"** no dropdown do usuário (antes só tinha "Sair")

### 🔄 Alterado — Icons8 Apple (substituição completa de emojis do sistema)
- `SUGGESTED_EMOJIS[]` → `SUGGESTED_ICONS[]` — array de objetos `{name, src}` com URLs Icons8 `ios-filled/32`
- Grid de emojis da tarefa agora exibe `<img>` Icons8 28px em vez de caracteres unicode
- `task-heart` (círculo colorido da tarefa) exibe ícone Icons8 com `filter: brightness(0) invert(1)` (branco sobre cor)
- Hero card "Próxima tarefa" usa Icons8 para o ícone grande
- Ícone de lixeira `🗑️` → `<img>` Icons8 `trash` 18px
- Ícones de estado vazio: `🎯` → Icons8 `target`, `📋` → Icons8 `task-planning`, `🌟` → Icons8 `sun`
- Dropdown: ícone de usuário e sair usam Icons8 (`user`, `exit`, `settings`)
- Botão upload filho usa Icons8 `upload`; câmera do perfil usa Icons8 `camera`
- `COLORS` array: campos `emoji` removidos; select de estado usa label puro (ícone era decorativo)
- Onboarding step 1: `<text>👧</text>` em SVG substituído por rosto SVG desenhado inline (sem dependência de unicode)
- `selectedTaskEmoji` muda de string unicode para `name` (key do ícone)
- `addTask()` / `createTaskCard()` usam `SUGGESTED_ICONS.find(ic => ic.name === t.emoji)` para lookup

### 🔧 Corrigido
- `showApp()` agora lê `state.user.nick` para o greeting ("Olá, ...")
- `onAuthStateChanged` carrega `nick` e `photoURL` do Firestore no `state.user`
- `h1,h2,h3` agora usam `color:var(--ink)` (compatível com tema escuro)

### ⚠️ Observações
- Fotos de perfil e de crianças são armazenadas como **base64 no Firestore** — adequado para MVP mas pode crescer rapidamente; migrar para Firebase Storage em versão futura
- Icons8 free tier exige atribuição; ícones são carregados via CDN `img.icons8.com` (sem API key)
- `photoURL` em `users/` sobrescreve o `photoURL` do Firebase Auth (não modifica Auth, só Firestore)
- Tema persiste via `localStorage`; ao fazer logout e relogin o tema é mantido (comportamento esperado)



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