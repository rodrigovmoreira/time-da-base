# Pauta — próximos desenvolvimentos (Time da Base)

> **Status:** documento de planejamento. Versão 2 — reorganizada a partir da lista original de 6 itens.
> **Escopo desta rodada:** somente este arquivo. Nenhuma alteração de código, configuração ou
> infraestrutura deve ser executada a partir desta pauta antes da priorização dos épicos abaixo.

## Como usar

- Cada item da lista original virou um **épico** com frase de origem, objetivo, escopo, tarefas e
  critérios de aceite — o suficiente para abrir tarefas sem inventar requisitos na hora.
- Onde a frase original é ambígua, a ambiguidade está **explicitada** e há uma recomendação.
- Tamanho relativo: **P** ≤ 1 dia · **M** 2–3 dias · **G** ≥ 4 dias (sem contar revisão/deploy).
- Referências de código são caminhos relativos a `frontend/`.

---

## 1. Fotografia do que existe hoje

Fatos verificados no repositório — base para as decisões de escopo:

| Fato | Evidência |
| --- | --- |
| SPA React 18 + Vite 6, sem backend; estado só em memória | `src/context/CampaignsContext.jsx`, `src/context/SelectionContext.jsx` |
| Nenhuma persistência: recarregar a página zera seleção, reservas e campanhas criadas | ausência de `localStorage`/`fetch` no código |
| Dados fictícios e determinísticos (8 campanhas, 1.000 números cada) | `src/data/campaigns.js`, `src/data/numbers.js` |
| `/admin` é acessível sem qualquer barreira | `src/App.jsx`, `src/layouts/AdminLayout.jsx`, CTA no `src/components/Header.jsx` |
| Arrecadação agregada aparece em **duas** telas: hero da Home e Dashboard | `src/pages/Home.jsx:132`, `src/pages/admin/Dashboard.jsx:45` |
| Páginas públicas de campanha **não** exibem arrecadação (só preço unitário e ocupação) | `src/components/CampaignCard.jsx`, `src/pages/CampaignDetails.jsx`, `src/utils/stats.js` |
| Participante guarda **quantidade** de números, não **quais** números | `src/context/SelectionContext.jsx` (registro em `confirmPayment`) |
| Telefone já aparece no painel, porém mascarado | `src/components/ParticipantsTable.jsx:31` + `maskPhone` |
| Nav do admin no mobile é uma faixa horizontal com rolagem invisível | `src/components/AdminSidebar.jsx:20` (`no-scrollbar overflow-x-auto`) |
| Menu mobile do site não fecha com `Esc` nem gerencia foco | `src/components/Header.jsx` |
| Deploy hoje é apenas `vite build` / `vite preview`; sem Dockerfile, CI ou `.env.example` | `package.json`, raiz do repo |

---

## 2. Ordem de execução sugerida

Os épicos não são independentes. Auth e dados são fundação; publicar sem eles só expõe um
protótipo acessível publicamente.

| Fase | Épicos | Por quê nesta ordem |
| --- | --- | --- |
| 0 — Evidência e ajustes de baixo risco (agora) | **E0**, **E2**, **E3** | Não dependem de backend, reduzem retrabalho e geram material para revisar o restante |
| 1 — Fundação (2 semanas) | **E1**, **E6** | Definir modelo de dados e contrato de autenticação antes da API crescer |
| 2 — Empacotamento e entrega (1 semana) | **E5** + CI mínimo | Só faz sentido empacotar algo que já tem dados e login |
| 3 — Operação do painel | **E4** | Depende de E1 (números por participante) e de E6 (quem pode ver) |
| 4 — Fora desta pauta | Pix real, expiração de reservas, jurídico | Ver seção 7 |

**Dependências explícitas**

- `E4 → E1` (sem persistir números por participante não há o que exibir).
- `E5 → E1 + E6` (a imagem precisa apontar para uma API e proteger `/admin`).
- `E2` e `E3` são independentes de tudo e podem entrar imediatamente.

---

## 3. Épicos

### E0 — Evidências visuais (prints de tela) · P

**Frase original:** “Pegar os prints de tela para melhorar a evidência”

**Objetivo:** registrar o estado atual da interface (antes/depois) como evidência de UX e insumo para
priorizar os demais épicos. Serve também como baseline para comparação após E1/E3/E6.

**Escopo**

- Capturar as rotas públicas e administrativas em duas larguras (mobile ~390px, desktop ~1440px).
- Capturar **estados**, não só telas vazias: grade com números disponíveis/reservados/pagos, resumo com
  formulário preenchido, pagamento pendente e confirmado, painel sem campanha, campanha em rascunho.
- Registrar os problemas encontrados como itens de backlog (numerados), ligando cada print ao épico
  correspondente (ex.: print do menu admin mobile → E3).
- Definir onde as imagens vivem: repositório (`docs/evidencias/AAAA-MM-DD/`) ou anexo em issue.
  Recomendação: anexar em issue e manter no repositório apenas o índice textual — prints binários
  envelhecem rápido e aumentam o clone.

**Tarefas**

1. Fechar a lista de 10 telas × 2 viewports (tabela de cobertura na issue).
2. Definir nomenclatura: `AAAA-MM-DD-<rota>-<viewport>-<estado>.png`.
3. Capturar manualmente **ou** criar script Playwright (`npm run screenshots`) para reprodutibilidade —
   a seed determinística do protótipo favorece isso.
4. Montar índice `docs/evidencias/README.md` com data, rota, viewport e link do print.
5. Abrir issues de problema a partir dos prints, com severidade (bloqueia / atrapalha / estético).

**Critérios de aceite**

- [ ] Todo épico desta pauta tem ao menos um print que ilustra o problema que ele resolve.
- [ ] Índice de evidências commitado e navegável.
- [ ] Prints capturados em sessão única (sem recarregar entre estados que dependem de contexto).

**Riscos:** as telas usam dados fictícios com CPF/telefone inventados — não usar prints com dados
realistas em material público sem revisar a legenda “dados fictícios”.

---

### E1 — Banco de dados não relacional · G

**Frase original:** “Criar banco de dados não relacional”

**Objetivo:** tirar o protótipo do estado “tudo em memória” e permitir que campanhas, números,
reservas e participantes sobrevivam a recarregar a página e sejam compartilhados entre usuários.

**Ponto de partida:** não existe API nem cliente HTTP; o estado nasce dos seeds em `src/data/*.js`.
Qualquer backend precisa começar por um **modo mock** no front para não travar o desenvolvimento.

**Escopo**

- Escolher engine e hospedagem (recomendação: MongoDB 7; local em container para paridade com E5,
  produção em Atlas free/M10 na região `sa-east-1`).
- Modelar as coleções e **índices** antes de escrever telas:

| Coleção | Campos principais | Índices |
| --- | --- | --- |
| `campaigns` | `slug`, `name`, `status`, `pricePerNumber`, `totalNumbers`, `drawDate`, `regulation` | `slug` único |
| `numbers` | `campaignId`, `index` (0–999), `status`, `reservationId` | composto único (`campaignId`,`index`); (`campaignId`,`status`) |
| `reservations` | `campaignId`, `numbers[]`, `status`, `expiresAt` | TTL em `expiresAt` |
| `participants` | `campaignId`, `name`, `phone`, `cpfHash`, `quantity`, `numbers[]`, `status` | (`campaignId`,`createdAt`) |
| `payments` | `reservationId`, `provider`, `amount`, `status`, `paidAt` | (`reservationId`) único |
| `adminAudit` | `actorId`, `action`, `target`, `before/after`, `at` | (`at`) |

> **Decisão de modelagem:** um documento por número (1.000 docs/campanha), **não** um array de 1.000
> posições dentro da campanha. Array embedded gera contenção e impede atualização atômica por número —
> exatamente o requisito de “duas pessoas não compram o mesmo número”.

- Implementar API mínima (Node + Express/Fastify), sem expandir escopo:
  `GET /campaigns`, `GET /campaigns/:slug`, `GET /campaigns/:id/numbers`,
  `POST /reservations`, `POST /reservations/:id/confirm`, `GET /admin/participants`.
- Garantir **atomicidade da reserva**: `findOneAndUpdate({ index, status: 'available' }, …)` por número,
  com retry e rollback do lote se qualquer número falhar.
- Expiração de reserva (15 min) via TTL + job leve de reconciliação.
- Seed script que importa `src/data/*.js` para manter a demo idêntica ao protótipo atual.
- Camada de dados no front com dois adaptadores: `VITE_DATA_SOURCE=mock|api`, preservando
  `useCampaigns` como interface única.

**Fora de escopo:** Pix real, notificações, relatórios, multi-tenant.

**Critérios de aceite**

- [ ] Duas abas reservando o mesmo número ao mesmo tempo: apenas uma vence, a outra recebe conflito tratado na UI.
- [ ] Reserva não paga expira em 15 min e o número volta a “disponível”.
- [ ] Recarregar a página preserva campanhas, números e participantes.
- [ ] `npm run seed` recria as 8 campanhas originais sem divergência visual.
- [ ] Suposição de que “não relacional” é a escolha certa registrada em ADR curto (1 página), com o
      trade-off de relatórios relacionais (participante ↔ pagamento ↔ números) explícito.

**Riscos e atenção**

- **Sem E6, a API fica aberta**: qualquer cliente pode marcar números como pagos. Não publicar antes do login.
- Escolher NoSQL por conveniência não elimina a necessidade de integridade; os índices acima são o
  que impede duplicidade.
- CPF é dado pessoal sensível: guardar hash/criptografado, nunca em texto puro, e limitar o retorno da API.

---

### E2 — Retirar arrecadação da página principal · P

**Frase original:** “Tirar arrecadação da pagina principal”

**Objetivo:** não expor valor arrecadado ao público. Arrecadação é indicador interno/financeiro; na
página inicial ela sugere retorno financeiro do projeto, cria expectativa sobre destinação do dinheiro e
antecipa uma discussão jurídica que ainda não foi validada.

**Onde está hoje**

- `src/pages/Home.jsx:132` — célula “Arrecadação ilustrativa” no cartão do hero.
- `src/pages/admin/Dashboard.jsx:45` — **permanece** (é visão administrativa).
- Nenhuma outra tela pública calcula `revenue`.

**Escopo**

- Remover apenas a célula pública; **manter** o layout de 4 células do hero.
- Substituir por um indicador público coerente com a seção “Transparência na ocupação”:
  recomendação **“Números pagos”** (`summary.paid`), que já é usado nas demais telas públicas.
- Revisar o texto de apoio do cartão (“Dados simulados…”) para não prometer valores.
- Não alterar `aggregateCampaigns` (o Dashboard continua consumindo `revenue`).

**Critérios de aceite**

- [ ] Nenhuma rota pública renderiza soma de arrecadação; busca por “Arrecada” retorna só `admin/`.
- [ ] Hero mantém 4 células e não quebra em 320px/360px.
- [ ] Dashboard continua exibindo arrecadação com o rótulo “ilustrativa”.
- [ ] Print antes/depois anexado à issue (E0).

**Decisão pendente:** transparência financeira pública é requisito do projeto? Se sim, isso deve virar
uma seção própria (com destinação dos recursos), não uma célula solta no hero.

---

### E3 — Navegação mobile (site e admin) · M

**Frase original:** “Barra de navegacao ajustar para mobile” + trecho HTML colado da nav administrativa

O HTML colado corresponde ao menu renderizado por `src/components/AdminSidebar.jsx` na variante
`mobile`: quatro pílulas (`Dashboard`, `Campanhas`, `Números`, `Participantes`) dentro de um contêiner
`overflow-x-auto` com `no-scrollbar`.

**Problemas concretos**

1. **Admin:** com 4 itens + ícones, em telas ≤ 360px o último item fica fora da área visível e não há
   indicador de rolagem (a barra está oculta por `no-scrollbar`). Destino “descoberto por acidente”.
2. **Admin:** a faixa não é fixa; ao rolar a página ela desaparece.
3. **Site:** o menu sanfona (`Header.jsx`) não fecha com `Esc`, não devolve o foco ao botão, não bloqueia
   o scroll do body e não marca `aria-current` nos itens.
4. **Site:** o CTA “Área admin” existe só a partir de `md` no topo — no mobile depende do menu aberto.

**Opções para o admin (escolher uma)**

| Opção | Prós | Contras |
| --- | --- | --- |
| A. Barra inferior fixa (bottom tabs) | Padrão de app; sempre visível; alvo de toque grande | Consome altura; exige `safe-area-inset-bottom` |
| B. Grade 2×2 acima do conteúdo | Todos os itens visíveis sem rolar; simples | Menos “app”, ocupa mais espaço |
| C. Manter rolagem horizontal + fade + snap | Mudança mínima | Ainda esconde itens |

**Recomendação:** **B** agora (4 itens, zero scroll, menor risco) e migrar para **A** se o menu passar de
4–5 destinos.

**Tarefas**

1. Implementar a opção escolhida em `AdminSidebar` (variante `mobile`) mantendo `NavLink` e estado ativo.
2. `Header`: fechar com `Esc`, foco de retorno, `aria-current` nos links, `overflow-hidden` no body
   enquanto aberto, fechar ao navegar (já ocorre via `useEffect`).
3. Garantir alvos de toque ≥ 44×44px e contraste AA nos estados ativo/inativo.
4. Testar 320 / 360 / 390 / 430px + landscape 640px, iOS Safari e Android Chrome.
5. Verificar que nenhuma tela cria scroll horizontal de página.

**Critérios de aceite**

- [ ] Todos os destinos do admin são alcançáveis sem rolagem horizontal em 320px.
- [ ] `Esc` fecha o menu do site e o foco volta ao botão que o abriu.
- [ ] Navegação completa por teclado no menu mobile, com foco visível.
- [ ] Sem overflow horizontal em nenhuma rota nas larguras testadas.
- [ ] Prints mobile antes/depois anexados (E0).

---

### E4 — Exibir números no painel administrativo · M

**Frase original:** “exibir numero no painel administrativo”

**Ambiguidade (resolver antes de codar):** “número” pode significar duas coisas diferentes.

| Leitura | Situação hoje | Trabalho |
| --- | --- | --- |
| **(a) Quais números (000–999) cada participante comprou** | Não existe: participante guarda só `quantity`; a grade do admin mostra status, não dono | Modelar `numbers[]` no participante (E1) e exibir |
| **(b) Número de telefone/contato do participante** | Já existe, porém mascarado (`maskPhone`) | Decidir política de exibição/revelação |

**Recomendação:** tratar as duas, começando por **(a)** — é lacuna real de dados e responde “quem ficou
com o 042?”.

**Escopo (a)**
- Persistir os índices reservados/pagos no registro do participante/reserva.
- Coluna “Números” na tabela: primeiros 5 como chips + `+N` com expansão; na versão mobile, lista
  colapsável.
- Na grade de `/admin/numeros`, clicar em número reservado/pago abre detalhe: participante, data,
  status da reserva.
- Busca por número (ex.: `042`) e por nome/telefone.

**Escopo (b)**
- Manter máscara por padrão; ação explícita “revelar” com registro em `adminAudit` (E1) e aviso de dado
  pessoal (LGPD).
- Atalhos `copiar` e `WhatsApp` (`wa.me/55…`) apenas após revelar.

**Critérios de aceite**

- [ ] Todo número reservado/pago tem um responsável identificável no painel.
- [ ] Nenhum número aparece como “sem dono” após confirmação de pagamento.
- [ ] Telefone permanece mascarado até ação explícita; revelação registrada em auditoria.
- [ ] Versão mobile da tabela não quebra layout com listas longas de números.

---

### E5 — Docker para subir em servidor · M

**Frase original:** “Criar um arquivo docker para subir em servidor”

**Objetivo:** transformar o repositório em algo que sobe em um servidor com um comando, sem depender de
Node instalado na máquina e com o mesmo comportamento do ambiente local.

**Entregáveis**

1. `Dockerfile` multi-stage: `node:20-alpine` (build) → `nginx:alpine` (serve), rodando como usuário
   não-root.
2. `nginx.conf` com **fallback de SPA** (`try_files $uri /index.html`) — sem isso, deep links como
   `/admin/numeros` retornam 404 ao recarregar.
3. `.dockerignore` (`node_modules`, `dist`, `.git`).
4. `docker-compose.yml` com `frontend`, `api` e `mongo`, healthchecks e `depends_on: service_healthy`.
5. Proxy reverso de `/api` no nginx para o serviço `api` — assim o front usa **caminho relativo** e a
   imagem é promovível entre ambientes (evita `VITE_API_URL` congelada no build).
6. Seção “Rodando com Docker” no `README.md`.

**Fora de escopo:** Kubernetes, autoscaling, multi-região.

**Critérios de aceite**

- [ ] `docker build -t time-da-base .` conclui sem warnings de segurança relevantes.
- [ ] `docker run -p 8080:80` serve a Home e devolve **200** ao recarregar `/admin/numeros`.
- [ ] `docker compose up` sobe front + api + mongo e a Home lista as campanhas vindas da API.
- [ ] Imagem final ≤ ~70MB; assets com hash em cache longo, `index.html` sem cache.
- [ ] Nenhum segredo no `Dockerfile`, no compose ou no bundle do front.
- [ ] README documenta portas, variáveis e como resetar o volume do banco.

**Riscos e atenção**

- Fontes vêm do Google Fonts (`index.html`): em servidor sem saída externa a tipografia degrada —
  avaliar self-host das fontes no build.
- TLS e domínio não estão cobertos aqui: decidir Caddy/Traefik na frente do container.
- Build em macOS ARM vs VPS x86: usar `docker buildx` com `--platform linux/amd64`.

---

### E6 — Login (Squamata-login) no painel administrativo · G

**Frase original:** “Incluir o Squamata-login para acessar o painel administrativa”

**Objetivo:** impedir acesso anônimo a `/admin` e às rotas administrativas da API. Hoje qualquer visitante
edita campanhas e marca números como pagos.

**Decisão bloqueante:** o que é “Squamata-login”? (serviço interno de SSO, provedor OIDC ou pacote
próprio). Enquanto não estiver definido, os itens abaixo valem para qualquer provedor.

**Escopo**

- Definir contrato: `POST /auth/login`, `GET /auth/me`, `POST /auth/logout`; papéis `admin` e
  `operador` (mínimo: `admin`).
- Preferir **cookie `httpOnly` + `SameSite=Lax`** (ou `None; Secure` se o domínio do login for outro) a
  token em `localStorage`.
- Front: rota `/admin/login`, guarda de rota envolvendo `AdminLayout`, preservação do destino original
  (`from`) após login, contexto de sessão, botão de sair.
- API: middleware exigindo sessão em **todas** as rotas `/admin/*`; `401`/`403` consistentes; rate limit
  no login; registro de acesso em `adminAudit`.
- Interceptor de `401` no cliente → redireciona para login sem perder o que estava aberto.
- Modo de demonstração explícito (`VITE_AUTH_MODE=squamata|mock`) para preview público continuar
  navegável, **com decisão registrada** sobre manter ou fechar a demo.

**Critérios de aceite**

- [ ] `/admin` e `/admin/*` sem sessão redirecionam para `/admin/login`.
- [ ] Após login, o usuário volta exatamente à rota que tentou abrir.
- [ ] Chamadas diretas à API administrativa sem sessão retornam `401` (validado por `curl`).
- [ ] Logout invalida a sessão no servidor, não só no cliente.
- [ ] Nenhum segredo (client secret, service token) no bundle; apenas no backend.
- [ ] Sessão expirada durante edição avisa antes de descartar o formulário.

**Riscos e atenção**

- Bloquear a demo pública sem combinar com o time pode “quebrar” apresentações — decidir o modo demo
  antes de publicar.
- Cookies cross-site exigem HTTPS, `SameSite=None; Secure` e CORS com credenciais.
- Se o Squamata-login for OIDC público, usar PKCE e nunca embutir segredo no front.

---

## 4. Pré-requisitos técnicos transversais

Não são épicos, mas precisam existir antes de E1/E5 ficarem sustentáveis:

- **Lint** (`eslint` + `npm run lint`) — hoje não há script.
- **Testes mínimos** (`vitest` + Testing Library) para: cálculo de `getCampaignStats`, fluxo de reserva,
  guarda de rota do admin.
- **CI** (GitHub Actions): `install → lint → build → test` em PR.
- **Config por ambiente**: `src/config.js` + `.env.example` documentando cada `VITE_*`.
- **Contrato de API** (OpenAPI curto ou tipos JSDoc) para front e back não divergirem.

---

## 5. Perguntas em aberto

| # | Pergunta | Bloqueia |
| --- | --- | --- |
| 1 | “Squamata-login” é serviço interno, OIDC ou pacote? Quem provisiona as credenciais? | E6 |
| 2 | A demo pública continua acessível sem login ou o `/admin` fica fechado em produção? | E5, E6 |
| 3 | Onde hospedar: VPS própria, PaaS ou container gerenciado? Há domínio definido? | E5 |
| 4 | Banco: Mongo em container no mesmo servidor ou Atlas? Quem opera backup? | E1, E5 |
| 5 | Telefone pode ser exibido completo no admin ou permanece mascarado? | E4 |
| 6 | Existe backend/API já iniciado em outro repositório? | E1 |
| 7 | Transparência financeira pública é requisito legal do projeto? | E2 |

Enquanto 1–7 não forem respondidas, valem as recomendações marcadas neste documento.

---

## 6. Definition of Ready / Done

**Ready (por tarefa)**

- Frase de origem e problema descritos; critério de aceite verificável; print ou evidência anexada;
  dependências (E1/E6) concluídas.

**Done (por épico)**

- Critérios de aceite marcados; sem regressão nas rotas públicas; `npm run build` limpo; prints
  antes/depois anexados; `README.md` atualizado quando o comportamento mudar (hoje ele afirma “não há
  banco de dados, autenticação…”, o que deixará de ser verdade em E1/E6).

---

## 7. Fora desta pauta (backlog futuro)

- Pix real com webhook e conciliação (substitui o QR ilustrativo de `src/components/PixQrCode.jsx`).
- Notificação por e-mail/WhatsApp de reserva e confirmação.
- Validação jurídica do regulamento e autorização da modalidade (Sorteio/Promoção).
- Política de privacidade, consentimento e retenção de dados pessoais.
- Relatórios/exportação CSV de participantes e conciliação financeira.

---

## 8. Registro de mudanças deste documento

| Versão | Mudança |
| --- | --- |
| 1 | Lista bruta de 6 itens + HTML colado da nav administrativa. |
| 2 | Itens convertidos em épicos E0–E6 com escopo, tarefas, critérios de aceite, dependências, riscos e decisões pendentes; adicionadas fases de execução, perguntas em aberto e Definition of Done. Nenhum código alterado. |
