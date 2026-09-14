# Análise Técnica — `app-diana-guardian-web` (App do Responsável)

> A **tela do responsável**: onde o alerta chega e a decisão humana acontece. Consome a
> `guardian-api` (**já implementada e mergeada** em `app-diana-guardian-api`, branch `develop`) para
> exibir o **alerta estruturado** (score, prioridade, categorias, sinais, fatores contextuais,
> justificativa, recomendação) e **nunca a conversa integral** (RF-16). **Sem autenticação no MVP**
> (mesma decisão da API — ver §8).
>
> Este documento é o **plano técnico de implementação**. Não implementa nada ainda: orienta os PRs
> de desenvolvimento (§10). Em caso de divergência de contrato, o contrato **real** é o código de
> `app-diana-guardian-api` (branch `develop`) — `src/domain/viewTypes.ts` e `src/contracts/*` — e,
> por trás dele, `app-diana-monitoring/docs/contracts.md`.

Selos usados: 🧊 **PRINCÍPIO CONGELADO** (não muda sem revisão) · ⚙️ **CONFIGURÁVEL** (parâmetro/heurística de MVP).

---

## 0. Contexto e fronteiras

```text
Núcleo (app-diana-monitoring)         guardian-api (JÁ implementada)      guardian-web (ESTE repo)
ingestor → pipeline → risk-engine ──▶ Object Storage (JSON) ──lê──▶ HTTP  ──consome──▶  app do responsável
                                       alerts/<conv>/<ts>.json      fina        fetch/render
```

- O guardian-web **não** acessa o núcleo, o Object Storage nem a `Conversation` diretamente. Sua
  **única** fonte de dados é a `guardian-api` (HTTP/JSON), via `GET /alerts`, `GET /alerts/:id`,
  `POST /alerts/:id/feedback`, `GET/PUT /settings`, `GET /health`.
- 🧊 **P4/RF-16 (reforço na última milha):** a UI **renderiza** o que a API já resumiu; nunca tenta
  reconstruir ou exibir texto de mensagem/`Conversation`. `AlertViewSignal.messageIds` é tratado como
  referência opaca (a UI usa só `.length`/`.occurrences`), nunca exibido como conteúdo.
- O guardian-web é um **SPA estático** (sem SSR/backend próprio): hospedagem no MVP é
  **OCI Object Storage (site estático)** ou, alternativamente, uma Container Instance servindo os
  arquivos buildados (ver `app-diana-monitoring/docs/06-mapeamento-oci.md`).

---

## 1. Objetivo e princípios de porte

| # | Princípio | Selo |
| --- | --- | --- |
| W1 | **Consumidor fino do contrato real da API.** A UI não recalcula risco, prioridade ou score; apenas exibe o que `guardian-api` já resumiu (`AlertSummary`/`AlertView`). | 🧊 |
| W2 | **Reaproveitar a UX já prototipada.** Portar/adaptar `app-diana-monitoring-lading-page/src/components/guardian/*` (Dashboard, AlertsList, AlertDetail, Settings, SafetyCenter) ao contrato real, não recriar do zero. | 🧊 |
| W3 | **RF-16 na tela.** Nenhum componente recebe ou renderiza `Conversation`/texto de mensagem bruto — só os tipos de `viewTypes.ts` da API. | 🧊 |
| W4 | **Todo estado de rede é explícito.** Toda tela que busca dados trata `loading`, `erro` (400/404/503) e vazio — nunca tela em branco silenciosa. | 🧊 |
| W5 | **Auth é um seam, não uma feature do MVP.** Sem login funcional; existe apenas uma tela placeholder e o ponto de extensão para OIDC na Fase 2 (§8). | ⚙️ |
| W6 | **Cliente HTTP tipado com os tipos reais da API.** Os tipos de resposta são portados de `app-diana-guardian-api/src/domain/viewTypes.ts` (cópia local, mesma estratégia de `src/contracts/` na API), não reinventados. | 🧊 |

---

## 2. Stack e scaffolding

### 2.1 Escolhas de stack

O repo está vazio (só `.gitignore` + commit inicial) — escolha própria, alinhada ao **protótipo**
(`app-diana-monitoring-lading-page`, cuja UX será portada) e à necessidade de **site estático** na
OCI (§0):

| Item | Escolha | Por quê |
| --- | --- | --- |
| Build tool | **Vite 7** | mesmo do protótipo; build estático (`dist/`) pronto para Object Storage. |
| Framework | **React 18** + **TypeScript 5.9** | mesma base do protótipo; componentes `guardian/*` são React. |
| Roteamento | **react-router-dom 7** | já usado no protótipo (mesmo que hoje só na landing). |
| Estilo | **Tailwind CSS 3** + **shadcn/ui** (Radix) | reaproveita `components/ui/*` e o design system do protótipo praticamente 1:1. |
| Ícones | **lucide-react** | usado nos componentes portados. |
| Gráficos | **recharts** | usado no `Dashboard` (gráfico de atividade). |
| Data fetching/estado servidor | **@tanstack/react-query 5** | já é dependência do protótipo; dá `loading`/`error`/cache "de fábrica" para W4, sem reinventar. |
| Validação de resposta | **zod** | valida o payload da API na borda do cliente HTTP (defesa contra deriva de contrato). |
| Testes | **vitest** + **@testing-library/react** | mesmo runner do resto do ecossistema DIANA (API/núcleo usam vitest). |
| Lint/format | **ESLint 9 (flat)** + **Prettier** | mesmo padrão dos outros repos DIANA (copiar `eslint.config.js` do protótipo, mantendo regras de React). |

> Não é Next.js: não há necessidade de SSR/rotas de servidor — o app é 100% client-side consumindo
> uma API externa, e a hospedagem-alvo é **site estático**. Vite + SPA é o caminho mais direto e o
> mais próximo do protótipo já validado (menor custo de porte dos componentes `guardian/*`).

### 2.2 `package.json` (esboço — dependências mínimas para o MVP)

```jsonc
{
  "name": "app-diana-guardian-web",
  "version": "0.1.0",
  "private": true,
  "type": "module",
  "engines": { "node": ">=22" },
  "scripts": {
    "dev": "vite",
    "build": "vite build",
    "preview": "vite preview",
    "lint": "eslint .",
    "typecheck": "tsc --noEmit",
    "test": "vitest run",
    "check": "npm run lint && npm run typecheck && npm run test",
    "format": "prettier --write ."
  },
  "dependencies": {
    "react": "^18.3.1",
    "react-dom": "^18.3.1",
    "react-router-dom": "^7.8.2",
    "@tanstack/react-query": "^5.86.0",
    "zod": "^3.23.8",
    "lucide-react": "^0.542.0",
    "recharts": "^3.1.2",
    "clsx": "^2.1.1",
    "tailwind-merge": "^3.3.1",
    "class-variance-authority": "^0.7.1"
  },
  "devDependencies": {
    "@vitejs/plugin-react": "^5.0.2",
    "@types/react": "^18.3.0",
    "@types/react-dom": "^18.3.0",
    "@types/node": "^22.8.0",
    "typescript": "^5.9.2",
    "typescript-eslint": "^8.11.0",
    "eslint": "^9.13.0",
    "@eslint/js": "^9.13.0",
    "eslint-plugin-react-hooks": "^5.2.0",
    "eslint-plugin-react-refresh": "^0.4.20",
    "globals": "^15.11.0",
    "prettier": "^3.3.3",
    "tailwindcss": "^3.4.17",
    "postcss": "^8.5.6",
    "autoprefixer": "^10.4.21",
    "tailwindcss-animate": "^1.0.7",
    "vite": "^7.1.4",
    "vitest": "^2.1.4",
    "@testing-library/react": "^16.0.1",
    "jsdom": "^25.0.1"
  }
}
```

> Radix UI individual (`@radix-ui/react-*`) entra **conforme os componentes `ui/*` forem portados**
> (PR-05, §10) — não tudo de uma vez; só o necessário para os componentes `guardian/*` usados no MVP
> (`Badge`, `Switch`, `Card`-like divs já são maioria HTML+Tailwind puro no protótipo).

### 2.3 `tsconfig.json` / ESLint

- `tsconfig.json`: `target ES2022`, `module ESNext`/`bundler moduleResolution`, `jsx react-jsx`,
  `strict: true`, `noUncheckedIndexedAccess: true` (mesmo rigor dos outros repos DIANA).
- ESLint 9 flat: base do protótipo (`js.configs.recommended` + `typescript-eslint` recomendado +
  `eslint-plugin-react-hooks` + `eslint-plugin-react-refresh`), sem as regras específicas de
  analytics/i18n do protótipo (não usadas no MVP do guardian-web).

### 2.4 Variáveis de ambiente — `.env.example`

```bash
# app-diana-guardian-web — app do responsável. Copie para .env (NUNCA versione .env).

# URL base da guardian-api (sem barra final). Em dev, aponta para a API local.
VITE_GUARDIAN_API_URL=http://localhost:8080

# Chave de API opcional (espelha GUARDIAN_API_KEY da API). Vazio = API aberta (demo).
# Enviada como header x-api-key em toda chamada. NUNCA um segredo real aqui — este
# valor vai para o bundle do cliente (é público); serve só de freio de demonstração.
VITE_GUARDIAN_API_KEY=

# Feature flag: exibe a tela placeholder de login (Fase 2). "false" no MVP.
VITE_ENABLE_AUTH_PLACEHOLDER=false
```

> ⚠️ Nota de segurança: qualquer variável `VITE_*` é **embutida no bundle público** (site estático).
> `VITE_GUARDIAN_API_KEY`, se usada, é visível a qualquer visitante — coerente com a nota de risco
> §8.2 da análise da API ("freio de demonstração, não controle de acesso real"). Reforça por que
> **dados sensíveis reais não devem trafegar** enquanto não houver auth real (Fase 2).

---

## 3. Layout de pastas

```text
app-diana-guardian-web/
├─ docs/
│  └─ analise-guardian-web.md        # este documento
├─ public/
├─ src/
│  ├─ main.tsx                       # bootstrap: React root, QueryClientProvider, router
│  ├─ App.tsx                        # shell do app (layout, navegação inferior)
│  ├─ router.tsx                     # rotas: /, /alerts, /alerts/:id, /settings, /safety, /login
│  ├─ env.ts                         # leitura/validação (zod) das VITE_* env vars
│  ├─ api/                           # cliente HTTP tipado para a guardian-api
│  │  ├─ client.ts                   # fetch wrapper: base URL, x-api-key, tratamento de erro HTTP
│  │  ├─ types.ts                    # PORTADO de guardian-api/src/domain/viewTypes.ts (cópia local)
│  │  ├─ schemas.ts                  # zod: valida a resposta da API contra os tipos acima
│  │  ├─ alerts.ts                   # getAlerts(), getAlert(id), postFeedback(id, body)
│  │  ├─ settings.ts                 # getSettings(), putSettings(body)
│  │  └─ health.ts                   # getHealth()
│  ├─ hooks/                         # React Query hooks por recurso
│  │  ├─ useAlerts.ts                # useQuery(GET /alerts) + filtros (priority/category)
│  │  ├─ useAlert.ts                 # useQuery(GET /alerts/:id)
│  │  ├─ useFeedbackMutation.ts      # useMutation(POST /alerts/:id/feedback)
│  │  ├─ useSettings.ts              # useQuery + useMutation(GET/PUT /settings)
│  │  └─ useHealth.ts                # useQuery(GET /health) — usado num banner de status opcional
│  ├─ components/
│  │  ├─ guardian/                   # PORTADO de app-diana-monitoring-lading-page/src/components/guardian/*
│  │  │  ├─ Dashboard.tsx            # adaptado: consome useAlerts()/useHealth() em vez de dados mock locais
│  │  │  ├─ AlertsList.tsx           # adaptado: consome useAlerts() com filtro de prioridade
│  │  │  ├─ AlertDetail.tsx          # adaptado: consome useAlert(id) + useFeedbackMutation()
│  │  │  ├─ SettingsView.tsx         # adaptado: consome useSettings()
│  │  │  └─ SafetyCenter.tsx         # conteúdo estático (riskCategories) — porta quase 1:1
│  │  ├─ shared/                     # SignalIcon, priorityTheme — portados quase 1:1
│  │  ├─ ui/                         # shadcn/ui portado sob demanda (Badge, Switch, Skeleton…)
│  │  ├─ feedback/                   # NOVO: estados de rede reutilizáveis
│  │  │  ├─ ErrorState.tsx           # 400/404/503 — mensagem + ação (retry)
│  │  │  ├─ LoadingState.tsx         # skeleton/spinner
│  │  │  └─ EmptyState.tsx           # lista vazia ("nenhum alerta")
│  │  └─ layout/
│  │     └─ BottomNav.tsx            # adaptado de GuardianPhone.tsx (tabs: início/alertas/segurança/config)
│  ├─ pages/                         # 1 página por rota, compõem os componentes acima
│  │  ├─ DashboardPage.tsx
│  │  ├─ AlertsPage.tsx
│  │  ├─ AlertDetailPage.tsx
│  │  ├─ SettingsPage.tsx
│  │  ├─ SafetyCenterPage.tsx
│  │  └─ LoginPlaceholderPage.tsx    # NOVO — placeholder estático (§8)
│  └─ lib/
│     └─ utils.ts                    # cn() — portado do protótipo
├─ test/
├─ .env.example  .gitignore
├─ eslint.config.js  .prettierrc.json  tsconfig.json
├─ tailwind.config.ts  postcss.config.js  vite.config.ts
├─ package.json  README.md
```

---

## 4. Cliente HTTP tipado (contrato real da API)

### 4.1 Tipos portados (`src/api/types.ts`)

🧊 **W6:** cópia local — mesma estratégia adotada em `app-diana-guardian-api/src/contracts/` — de
[`app-diana-guardian-api/src/domain/viewTypes.ts`](https://github.com/Tech4Change-Diana/app-diana-guardian-api/blob/develop/src/domain/viewTypes.ts)
(branch `develop`, real): `AlertSummary`, `AlertViewSignal`, `AlertViewExplanation`, `AlertView`,
`FeedbackVerdict`, `FeedbackRecord`, `GuardianSettings(Item/Section)`. **Não redefinir campos** —
copiar 1:1. Quando `@diana/contracts` existir, ambos os repos migram juntos (mesma nota de
sincronização da API, §4 de `analise-guardian-api.md`).

### 4.2 `src/api/client.ts` — fetch wrapper

```ts
interface RequestOptions { signal?: AbortSignal }

async function request<T>(path: string, init?: RequestInit & RequestOptions): Promise<T> {
  const res = await fetch(`${env.apiUrl}${path}`, {
    ...init,
    headers: {
      "content-type": "application/json",
      ...(env.apiKey ? { "x-api-key": env.apiKey } : {}),
      ...init?.headers,
    },
  });
  if (!res.ok) throw await ApiError.fromResponse(res); // §6
  return res.json() as Promise<T>;
}
```

- Um único ponto de montagem de URL/headers (`VITE_GUARDIAN_API_URL`, `x-api-key` opcional).
- `ApiError` (§6) carrega `status` + `body` já parseado (a API sempre responde
  `{ error, message?, issues? }` em erro — ver rotas reais) para a UI decidir a mensagem certa.

### 4.3 Funções por recurso (`src/api/alerts.ts`, `settings.ts`, `health.ts`)

Espelham exatamente as rotas reais (§7 de `analise-guardian-api.md`, verificadas no código):

```ts
getAlerts(params?: { priority?: GuardianPriority; category?: string; limit?: number; cursor?: string })
  : Promise<{ items: AlertSummary[]; nextCursor?: string }>
  // GET /alerts — pode 503 (source_unavailable) ou 400 (cursor/limit inválido).

getAlert(id: string): Promise<AlertView>
  // GET /alerts/:id — 400 (id malformado) · 404 (not_found) · 503.

postFeedback(id: string, body: { verdict: FeedbackVerdict; note?: string }): Promise<FeedbackRecord>
  // POST /alerts/:id/feedback — 201. 400 (validação) · 404 (alerta inexistente).

getSettings(): Promise<GuardianSettings>   // GET /settings
putSettings(body: GuardianSettings): Promise<GuardianSettings>  // PUT /settings — 400 se inválido.

getHealth(): Promise<{ status: "ok"; source: string; writeBackend: string; uptime: number }>
```

### 4.4 Validação de resposta (`src/api/schemas.ts`)

⚙️ zod schemas espelhando os tipos de `types.ts`, usados para `safeParse` da resposta antes de
devolver ao hook. Em caso de divergência (deriva de contrato), lança um erro tratável (§6) em vez de
deixar a UI renderizar campos `undefined` silenciosamente — mesma filosofia defensiva do
`contracts.parity.test.ts` sugerido na API.

---

## 5. Telas

Mapeamento **doc 05 (`experiencia-responsavel.md`) → componente do protótipo → página real**:

| Tela | Componente-base (protótipo) | Rota | Dados (API real) |
| --- | --- | --- | --- |
| **Dashboard** | `Dashboard.tsx` | `/` | `GET /alerts` (resumo/contagens client-side a partir de `AlertSummary[]`; sem endpoint de estatísticas dedicado — ver D1 em §11) |
| **Lista de Alertas** | `AlertsList.tsx` | `/alerts` | `GET /alerts?priority=&category=` |
| **Detalhe do Alerta** | `AlertDetail.tsx` | `/alerts/:id` | `GET /alerts/:id` + `POST /alerts/:id/feedback` |
| **Configurações** | `Settings.tsx` (`SettingsView`) | `/settings` | `GET /settings` · `PUT /settings` |
| **Central de Segurança** | `SafetyCenter.tsx` | `/safety` | conteúdo estático (`riskCategories`, portado do protótipo — a API não serve isso; é orientação/educação, não dado do responsável) |
| **Login (placeholder Fase 2)** | — (novo) | `/login` | nenhum — tela estática (§8) |

Navegação: `BottomNav` (adaptado de `GuardianPhone.tsx`) com 4 abas (Início/Alertas/Segurança/
Configurações) + navegação para o detalhe do alerta a partir da lista/dashboard — mesma estrutura
já validada no protótipo, só trocando estado local por rotas (`react-router-dom`) e dados mock por
`useQuery`.

### 5.1 Adaptações necessárias (protótipo → contrato real)

| Componente | O que muda |
| --- | --- |
| `AlertsList` | Troca `recentAlerts` (mock local) por `useAlerts()`; `alert.category`/`alert.priority` já vêm no formato certo (`GuardianPriority` = `alta\|media\|baixa`, idêntico ao `Priority` do protótipo — **sem conversão**). Precisa de loading/error/empty (§6). |
| `AlertDetail` | Troca `analyzeConversation(...)` (pipeline mock local) por `useAlert(id)`; o shape de `AlertView` é **quase idêntico** a `AnalysisResult` (mesmos `assessment`-like campos achatados: `priority/level/score/rationale/categories/signals/explanation`) — ver diffs em §5.2. Adiciona UI de feedback (§7). |
| `Dashboard` | Estatísticas (`dashboardStats`) e `activityData` (gráfico) **não têm endpoint na API real** — ver decisão D1 (§11): no MVP, derivar o possível (contagem por prioridade) de `GET /alerts`, e ocultar/placeholder o que depende de série histórica até existir endpoint. |
| `Settings` | Troca `settingsSections` (mock local) + estado local por `useSettings()` (`GET`) e `useMutation` (`PUT`) — o shape `GuardianSettingsSection[]` da API é **o mesmo formato** do protótipo (`id/title/items[{id,label,description,enabled}]`). |
| `SafetyCenter` | Não depende da API — porta quase sem alteração (dado estático de orientação). |

### 5.2 Diferenças de shape a resolver no port do `AlertDetail`

- Protótipo usa `AnalysisResult.assessment.{priority,level,score,rationale,categories}`; a API
  **achata** esses campos direto em `AlertView` (`view.priority`, `view.level`, `view.score`, …, sem
  o wrapper `assessment`). Ajustar os acessos no componente portado (não é uma mudança de dado, só de
  caminho de propriedade).
- Protótipo usa `signal.messageIds.length`; `AlertView` já expõe `occurrences` pronto (conveniência
  da API) — preferir `occurrences` ao portar.
- `childName`: a API pode retornar `"Criança"` como fallback (nota D1/§6.4 da análise da API) — a UI
  deve aceitar esse valor sem tratamento especial (é só uma string).

---

## 6. Estado de rede: loading / erro / vazio (W4)

🧊 Nenhuma tela busca dado sem cobrir os três estados. Usando React Query:

| Situação | Origem | Tratamento na UI |
| --- | --- | --- |
| `isLoading` | 1ª busca | `LoadingState` (skeleton compatível com o layout do card/lista, evita "pulo" de layout) |
| `error` com `status === 503` (`source_unavailable`) | `GET /alerts`, `GET /alerts/:id` | `ErrorState` com mensagem "Não foi possível carregar os alertas agora" + botão **Tentar novamente** (`refetch`) |
| `error` com `status === 404` (`not_found`) | `GET /alerts/:id` | Tela dedicada "Alerta não encontrado" (ex.: link direto expirado/inválido) com voltar para a lista |
| `error` com `status === 400` (`bad_request`) | id malformado, cursor inválido, body de feedback/settings inválido | Nunca deveria ocorrer por ação normal do usuário; loga no console e mostra `ErrorState` genérico (indica bug, não input do usuário final) |
| `items.length === 0` (sem erro) | `GET /alerts` filtrado | `EmptyState` ("Nenhum alerta nesta categoria" — já existe redação equivalente no protótipo) |
| `GET /health` falha/`status !== "ok"` | opcional, banner global | Aviso discreto "API indisponível no momento" no topo do app (⚙️ opcional no MVP; útil para debug em demo) |

`ApiError` centraliza o parse do corpo de erro (`{ error, message?, issues? }`) para que cada tela só
decida a **apresentação**, não o parsing.

---

## 7. Feedback do responsável (RF human-in-the-loop)

Integrado na tela de **Detalhe do Alerta** (`AlertDetail`), abaixo da seção "O que você pode fazer?":

- Dois/três botões: **Útil** (`verdict: "useful"`), **Falso positivo** (`"false_positive"`),
  **Não tenho certeza** (`"not_sure"`) — mapeando 1:1 o enum `FeedbackVerdict` real da API.
- Campo de nota opcional (`note`, texto curto) antes de enviar — opcional, não bloqueia o envio.
- `useFeedbackMutation()`: ao suceder (201), exibe confirmação inline (ex.: "Obrigado pelo retorno")
  e desabilita os botões para aquele alerta (idempotência visual — a API já sobrescreve no backend,
  mas a UI não deve convidar a reenviar sem necessidade).
- Erro (400 de validação, 404 se o alerta sumiu da fonte) → `ErrorState` local inline, sem navegar
  para fora da tela.
- 🧊 Nenhum dado além de `verdict`/`note` é coletado — não há campo livre que convide a colar texto
  da conversa (reforço de RF-16 também no fluxo de escrita).

---

## 8. Sem autenticação no MVP (🧊 decisão herdada da API) + placeholder Fase 2

Mesma decisão e mesma postura da `guardian-api` (§8 de `analise-guardian-api.md`,
`05-experiencia-responsavel.md`): **sem login funcional no MVP**.

### 8.1 Nota de risco (🧊)

> O app roda **sem autenticação de usuário**: qualquer pessoa com a URL acessa o mesmo conteúdo.
> Coerente com a API (sem IAM) e a postura **mock-first**. Enquanto isso não mudar, o app **não deve
> exibir alertas reais de crianças** fora de ambiente de demo controlado. A chave opcional
> (`VITE_GUARDIAN_API_KEY`) é, no máximo, um freio de demonstração — **está no bundle público**, não
> é segredo (§2.4).

### 8.2 Placeholder de tela de login (Fase 2)

- Rota `/login` com uma tela **estática** (formulário desabilitado ou "Em breve: login"), atrás da
  flag `VITE_ENABLE_AUTH_PLACEHOLDER` — **não** roteia nem bloqueia nada no MVP (todas as rotas reais
  continuam abertas).
- Serve para (a) já ocupar o espaço de navegação/design para a Fase 2 e (b) documentar o **seam**:
  quando OIDC (OCI IAM Identity Domains, mesmo mecanismo citado na API) existir, essa tela vira o
  fluxo real e um guard de rota passa a exigir sessão — **adição, não reescrita**, mesmo princípio
  usado em `src/http/auth.ts` da API.

---

## 9. Testes (mínimo do MVP)

- **Unit:** `api/schemas.ts` (parse válido/ inválido), `ApiError.fromResponse` (mapeamento de status
  → tipo de erro), hooks de dados com um cliente HTTP mockado (msw ou fetch mock simples).
- **Componente:** `AlertsList`/`AlertDetail`/`Settings` renderizando os 3 estados (loading/erro/dado)
  com `@testing-library/react`; `AlertDetail` cobrindo o fluxo de feedback (envio, sucesso, erro).
- **Asserção RF-16 (⚙️, espelhando a API):** um teste que garante que nenhum componente recebe ou
  renderiza uma prop com formato de `Conversation`/mensagem bruta — os únicos tipos aceitos pelos
  componentes `guardian/*` são os de `api/types.ts`.
- `npm run check` (lint + typecheck + test) verde antes de cada PR — mesma convenção dos outros repos
  DIANA.

---

## 10. CHECKLIST de PRs (pequenos e ordenados)

> Cada PR: branch `feature/...` → PR para `develop`; commits em português; `npm run check` verde.
> Ordem de valor: **PR-01…07** já entregam a demo completa consumindo a API real; PR-08/09 fecham
> settings e hardening de estado; PR-10 é o placeholder de auth (baixo risco, pode andar em paralelo).

- [ ] **PR-01 — Scaffolding.** Vite + React + TS, `package.json`, `tsconfig.json`,
      `eslint.config.js`, `.prettierrc.json`, Tailwind + PostCSS config, `.gitignore` (já existe),
      `README.md`, `src/main.tsx` mínimo servindo uma página em branco. `npm run check` verde.
- [ ] **PR-02 — Tipos e cliente HTTP.** `src/env.ts` (zod), `src/api/types.ts` (portado de
      `viewTypes.ts` da API real), `src/api/schemas.ts`, `src/api/client.ts` (+ `ApiError`),
      `.env.example`. Testes unitários do client/erro.
- [ ] **PR-03 — Funções de recurso + hooks.** `src/api/{alerts,settings,health}.ts` +
      `src/hooks/use{Alerts,Alert,Settings,Health}.ts` + `useFeedbackMutation`. `QueryClientProvider`
      em `main.tsx`. Testes com fetch mockado.
- [ ] **PR-04 — Design system portado.** `src/lib/utils.ts` (`cn`), `tailwind.config.ts` (tema do
      protótipo: cores `brand`, `success`, etc.), componentes `ui/*` mínimos usados pelo MVP (`Badge`,
      `Switch`), `components/shared/{SignalIcon,priorityTheme}.tsx` portados.
- [ ] **PR-05 — Estados de rede reutilizáveis.** `components/feedback/{LoadingState,ErrorState,
      EmptyState}.tsx` (§6). Testes de renderização dos 3 estados.
- [ ] **PR-06 — Rotas + navegação.** `router.tsx` (`/`, `/alerts`, `/alerts/:id`, `/settings`,
      `/safety`), `components/layout/BottomNav.tsx` (adaptado de `GuardianPhone.tsx`), `App.tsx`
      (shell). Navegação clicável entre as 5 telas (ainda com dados mock ou vazios).
- [ ] **PR-07 — Telas reais (Alertas + Detalhe).** `AlertsList`/`AlertDetail` portados e conectados a
      `useAlerts`/`useAlert`, cobrindo §5.1/§5.2 e os 3 estados de rede. **Entrega o fluxo principal
      da demo.**
- [ ] **PR-08 — Feedback do responsável.** UI de feedback em `AlertDetail` (§7) + `useFeedbackMutation`
      conectado a `POST /alerts/:id/feedback`. Testes do fluxo completo (envio/sucesso/erro).
- [ ] **PR-09 — Dashboard + Configurações + Central de Segurança.** `Dashboard` (derivado de
      `GET /alerts`, com nota sobre estatísticas ausentes — D1), `SettingsView` conectado a
      `useSettings`, `SafetyCenter` portado (estático). Fecha as 5 telas do MVP.
- [ ] **PR-10 — Placeholder de login (Fase 2).** Rota `/login` estática atrás de
      `VITE_ENABLE_AUTH_PLACEHOLDER` (§8.2). Sem lógica de sessão real.
- [ ] **PR-11 — Build estático + deploy.** `vite build` validado, notas de publicação em OCI Object
      Storage / Container Instance (§0, espelhando §2.5 da API). Opcional: workflow de build.

---

## 11. Riscos e decisões em aberto (candidatas a ADR)

| # | Assunto | Decisão pendente |
| --- | --- | --- |
| D1 | Estatísticas do `Dashboard` (`dashboardStats`, `activityData`) | A API real **não** expõe um endpoint de métricas/série histórica — só `GET /alerts`. Decidir: (a) derivar contagens simples client-side a partir da lista de alertas (MVP), ou (b) propor um endpoint agregador na API (fora do escopo deste repo). Preferência: (a) no MVP, registrar (b) como evolução. |
| D2 | Chave de API no cliente (`VITE_GUARDIAN_API_KEY`) | Confirmar se a demo roda com API aberta (sem chave) ou com chave simples visível no bundle (§2.4/§8.1) — mesma decisão D4 da análise da API, espelhada aqui. |
| D3 | CORS | `guardian-api` precisa incluir a origem do guardian-web em `CORS_ORIGINS` quando publicado; em dev local, `*` (default da API) já cobre. Registrar a origem de produção quando definida. |
| D4 | Paginação de `GET /alerts` | A API pagina por cursor (`nextCursor`); decidir se o MVP da UI implementa "carregar mais" (infinite query) ou apenas usa o `limit` default (50) sem paginação visível — suficiente para volume de demo. Preferência: sem paginação visível no MVP, registrar `nextCursor` como pronto para uso futuro. |
| D5 | Multi-responsável / `settings` por usuário | Só faz sentido com auth real (Fase 2) — a API já documenta isso como D6 dela; aqui a UI simplesmente assume um único responsável (mesma settings global). |

---

← Relacionados:
[`app-diana-guardian-api` — análise técnica](https://github.com/Tech4Change-Diana/app-diana-guardian-api/blob/develop/docs/analise-guardian-api.md) ·
[`app-diana-monitoring` — 05, experiência do responsável](https://github.com/Tech4Change-Diana/app-diana-monitoring/blob/develop/docs/05-experiencia-responsavel.md) ·
[`app-diana-monitoring` — 06, mapeamento OCI](https://github.com/Tech4Change-Diana/app-diana-monitoring/blob/develop/docs/06-mapeamento-oci.md) ·
[`app-diana-monitoring` — regras de negócio](https://github.com/Tech4Change-Diana/app-diana-monitoring/blob/develop/docs/regras-de-negocio.md) ·
[`app-diana-monitoring-lading-page` — componentes `guardian/*` (referência de UX)](https://github.com/Tech4Change-Diana/app-diana-monitoring-lading-page/tree/feature/init/src/components/guardian)
