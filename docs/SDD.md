# SDD — Frontend de gestão geoespacial (Protected Areas SC)

Complementa `docs/PRD.md`. Aqui ficam as decisões técnicas: arquitetura, contrato de API novo
(autenticação), modelo de dados, componentização do frontend, empacotamento Docker, estratégia de
testes (unitários e mutação) e as práticas DevSecOps que fecham as lacunas que a seção 5 sozinha
não cobre.

## 1. Visão geral de arquitetura

```text
                         rede docker "pipeline" (external, já existe)
                     ┌───────────────────────────────────────────────────┐
 navegador  ── HTTP ──▶  protected-areas-sc-frontend (novo)               │
 (usuário)             │  nginx:alpine                                    │
                        │   - serve o build estático do React             │
                        │   - proxy_pass /api/*  ──▶  protected-areas-sc-api:8000
                        └───────────────────────────────────────────────────┘
                                          │
                                          ▼
                               protected-areas-sc-api (existe)
                               FastAPI: /api/v1/imports, /api/v1/ucs,
                                        /api/v1/auth (novo, este documento)
                                          │
                                          ▼
                               protected-areas-sc-db-main (existe)
                               PostGIS: domínio geoespacial + tabelas novas
                               app_user / app_session (este documento)
```

Decisão central: o navegador **só fala com a origem do frontend**. O Nginx do container do
frontend faz reverse proxy de `/api/*` para o container da API pelo nome de serviço Docker
(`protected-areas-sc-api:8000`), na mesma rede `pipeline` externa que a API já usa para falar com
Airflow/PostGIS. Isso evita CORS, evita expor a API diretamente ao navegador e permite cookie de
sessão `httpOnly` funcionando como se fosse "mesma origem" — sem isso, cada ambiente (dev, banca,
apresentação) exigiria configurar CORS e cookie cross-site à parte.

**Consequência de implementação na API:** o cookie de sessão emitido pelo backend não deve
declarar `Domain=` explícito, para funcionar corretamente atrás do proxy em qualquer host.

## 2. Novo backend: autenticação e contas (na própria FastAPI)

Decisão já tomada: não cria serviço novo. Adiciona um módulo `app/api/v1/auth_routes.py` +
`app/application/auth.py` seguindo a mesma separação de camadas (`api` → `application` →
`domain`/`infrastructure`) já usada em `ucs_routes.py`/`UCService`.

### 2.1 Modelo de sessão

MVP usa **sessão opaca em tabela**, não JWT: token aleatório (32 bytes, `secrets.token_hex`),
guardado no navegador como cookie `httpOnly`, `Secure`, `SameSite=Lax`; no banco, guarda-se
apenas o hash SHA-256 do token (nunca o token em claro), com expiração e possibilidade de
revogação imediata. Motivo da escolha em vez de JWT: com poucos usuários (um admin, alguns
operadores), a simplicidade de "desativei o usuário → próxima requisição falha" sem lidar com
revogação de JWT stateless pesa mais do que o ganho de estatelessness. Se o projeto crescer, dá
para migrar para JWT depois sem mudar o contrato de fora (login/me continuam iguais).

### 2.2 Migração `006_auth_users.sql`

Segue a numeração das migrações existentes (`001` a `005` em `X:\fast-api-protected-areas-sc\migrations`).

```sql
CREATE TABLE app_user (
    id              BIGSERIAL PRIMARY KEY,
    username        VARCHAR(120) NOT NULL UNIQUE,
    nome            VARCHAR(255) NOT NULL,
    password_hash   VARCHAR(255) NOT NULL,
    role            VARCHAR(20)  NOT NULL CHECK (role IN ('admin', 'operador')),
    ativo           BOOLEAN      NOT NULL DEFAULT TRUE,
    criado_por      BIGINT       REFERENCES app_user(id),
    criado_em       TIMESTAMP    NOT NULL DEFAULT CURRENT_TIMESTAMP,
    atualizado_em   TIMESTAMP    NOT NULL DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE app_session (
    token_hash      CHAR(64) PRIMARY KEY,
    user_id         BIGINT NOT NULL REFERENCES app_user(id),
    criado_em       TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    expira_em       TIMESTAMP NOT NULL,
    revogado_em     TIMESTAMP
);
CREATE INDEX idx_app_session_user ON app_session(user_id);
```

`password_hash` usa `argon2` (`argon2-cffi`, já recomendado sobre `bcrypt` puro por não ter
limite de 72 bytes e ser o padrão atual da OWASP). Adicionar `argon2-cffi` às dependências da API.

### 2.3 Bootstrap do primeiro administrador

Não existe tela de "primeiro cadastro". No startup da API, se `app_user` estiver vazia, criar um
admin a partir de variáveis de ambiente (mesmo padrão já usado para segredos no projeto, como
`FIRMS_MAP_KEY`): `PA_SC_ADMIN_BOOTSTRAP_USERNAME` e `PA_SC_ADMIN_BOOTSTRAP_PASSWORD`. Se essas
variáveis não estiverem definidas e a tabela estiver vazia, a API sobe normalmente mas loga um
aviso claro — ninguém consegue logar até alguém definir essas variáveis e reiniciar. Essas
variáveis não devem ser versionadas (mesma regra já aplicada a `FIRMS_MAP_KEY` no `.env.example`).

### 2.4 Contrato de endpoints novo

| Método/rota | Quem pode | Corpo/resposta |
|---|---|---|
| `POST /api/v1/auth/login` | Público | `{username, password}` → `Set-Cookie` de sessão + `200 {id, nome, role}`; `401` se credenciais inválidas ou conta inativa |
| `POST /api/v1/auth/logout` | Autenticado | Revoga a sessão atual (marca `revogado_em`), limpa o cookie |
| `GET /api/v1/auth/me` | Autenticado | `{id, nome, role}`; `401` se sessão ausente/expirada/revogada |
| `POST /api/v1/auth/users` | Somente `admin` | `{username, nome, password, role}` → `201` com o usuário criado (sem hash na resposta); `409` se `username` já existe |
| `GET /api/v1/auth/users` | Somente `admin` | Lista `{id, username, nome, role, ativo, criado_em}` |
| `PATCH /api/v1/auth/users/{id}` | Somente `admin` | `{ativo: bool}` — ativa/desativa; desativar revoga todas as sessões daquele usuário na mesma transação |

Erros seguem o mesmo padrão `application/problem+json` já usado no resto da API
(`app/domain/models.py::ProblemDetail`), para o frontend reaproveitar o mesmo tratamento de erro
dos cenários geoespaciais.

Uma dependency FastAPI `require_role("admin")` (análoga a `get_uc_service` em
`app/api/dependencies.py`) protege as três rotas de `/auth/users`; as rotas de `/imports` e
`/ucs` existentes passam a exigir apenas `get_current_user` (qualquer papel autenticado), sem
diferenciar admin/operador — a distinção de papel só importa para o painel de administração.

## 3. Frontend

### 3.1 Stack

- **React 18 + TypeScript + Vite** (build rápido, sem necessidade de servidor Node em produção —
  o runtime final é só Nginx servindo arquivos estáticos).
- **react-router-dom** para rotas e guarda de rota por papel.
- **TanStack Query** para chamadas à API (cache, retry, polling do estado da importação).
- **react-hook-form + zod** para os formulários de cenário e de criação de conta, validando no
  cliente os campos obrigatórios por `domain`/`operation` antes de habilitar o envio (FE-RF-07).
- Sem Redux/estado global pesado: sessão do usuário fica em um `AuthContext` simples, alimentado
  por `GET /api/v1/auth/me` ao carregar a aplicação.
- Sem CSS framework pesado (Tailwind é opcional, mas não obrigatório para um MVP de formulário);
  decisão de biblioteca visual fica em aberto — pode ser CSS puro ou um kit leve de componentes,
  não é uma decisão que muda a arquitetura.

### 3.2 Estrutura de páginas/rotas

```text
/login                          LoginPage (pública)
/                                redireciona: admin → /admin, operador → /cenarios
/admin                           AdminDashboardPage   (guarda: role === 'admin')
  ├─ lista de contas (GET /auth/users)
  └─ formulário "criar conta"   (POST /auth/users)
/cenarios                        ScenarioFormPage      (guarda: autenticado)
  ├─ seletor de cenário: UC-CW01 | UC-CW02 | UC-CW04 | UC-CW07 | (CW03/05/06 desabilitados)
  ├─ formulário específico do cenário escolhido
  └─ ImportStatusPanel           (acompanha o import_id até estado terminal)
/historico                       HistoricoPage         (guarda: autenticado; US-12)
```

`RequireAuth` e `RequireAdmin` são componentes de rota que checam `AuthContext`; se a sessão não
existir (`GET /auth/me` retornou 401), redireciona para `/login`. Não há necessidade de um "modo
convidado": tudo além de `/login` exige sessão válida (FE-RF-01).

### 3.3 Formulário de cenário — desenho único parametrizado

Os quatro cenários habilitados compartilham o mesmo contrato de envio
(`POST /api/v1/imports`, multipart) e o mesmo ciclo de acompanhamento
(`GET /imports/{id}/validation` → `POST /imports/{id}/publish` → poll `GET /imports/{id}`); só
mudam os campos de metadata exigidos. Em vez de quatro telas distintas, um único componente
`ScenarioForm` recebe uma configuração declarativa por cenário:

```ts
type ScenarioConfig = {
  id: "UC-CW01" | "UC-CW02" | "UC-CW04" | "UC-CW07";
  domain: "uc" | "za_oficial";
  operation: "create" | "update" | "replace_zas";
  label: string;
  enabled: boolean;                 // false para CW03/05/06
  fields: FieldSchema[];            // gera o formulário e a validação zod
};
```

Isso evita duplicar a lógica de upload/idempotência/polling quatro vezes e deixa o "cenário fora
de escopo" (FE-RF-06) ser só uma entrada na lista com `enabled: false`, sem branch especial na
tela.

Campos por cenário (derivados da tabela "Formatos e metadados" do `README.md` da API):

| Cenário | Campos do formulário |
|---|---|
| `UC-CW01` (criar UC) | arquivo; `source`; política de duplicidade (`reject_batch`/`skip_duplicates`); nome só aparece se a API sinalizar `MISSING_UC_NAME` na validação |
| `UC-CW02` (criar UC pontual) | arquivo (1 ponto); `source` |
| `UC-CW04` (atualizar UC) | arquivo; `source`; `reason`; `expected_version`; identificador forte (`official_identifier`/`cd_cnuc`/`wdpa_pid`) |
| `UC-CW07` (substituir ZAS por ZA oficial) | arquivo (polígono); `source`; `uc_identifier`; `reason` |

`Idempotency-Key` é gerada no cliente com `crypto.randomUUID()` a cada novo envio (FE-RF-08) —
nunca reaproveitada entre tentativas distintas do usuário, para não mascarar um novo envio como
replay de um antigo.

### 3.4 Tratamento e tradução de erro

Um mapa fixo traduz os códigos de erro já documentados no projeto para pt-BR; qualquer código não
mapeado cai num texto genérico, mas o `detail`/`type` originais do `problem+json` ficam
disponíveis num "ver detalhes técnicos" — nunca escondidos, só não são a mensagem principal.

```ts
const ERROR_MESSAGES: Record<string, string> = {
  UC_ALREADY_EXISTS: "Essa UC já está cadastrada.",
  UC_VERSION_CONFLICT: "Alguém alterou essa UC desde a última consulta. Recarregue a versão atual e tente de novo.",
  MISSING_UC_NAME: "O arquivo não traz o nome da UC; preencha o campo Nome.",
  ZA_CREATE_REQUIRES_BATCH: "Não é possível cadastrar uma ZA isolada; use \"Substituir ZAS por ZA oficial\" numa UC existente.",
  DUPLICATE: "Essa importação já foi enviada antes; mostrando o resultado original.",
};
```

### 3.5 Testes

- **Vitest + React Testing Library**: formulários (validação de campo obrigatório, geração de
  Idempotency-Key, guarda de rota por papel).
- **MSW (Mock Service Worker)**: mocka `/api/v1/*` nos testes de componente, sem precisar da API
  real rodando — permite desenvolver o frontend antes ou em paralelo à API de autenticação.
- E2E (Playwright) fica como incremento pós-MVP, não bloqueia a entrega inicial.

## 4. Empacotamento Docker

### 4.1 Dockerfile (multi-stage)

```dockerfile
FROM node:20-alpine AS build
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build

FROM nginx:1.27-alpine AS runtime
COPY --from=build /app/dist /usr/share/nginx/html
COPY nginx.conf /etc/nginx/conf.d/default.conf
EXPOSE 80
```

### 4.2 `nginx.conf`

Config única, já com os cabeçalhos e o `autoindex off` exigidos pela seção 5.3 — não existe uma
segunda cópia "de segurança" separada da cópia "funcional"; deriva risco de as duas divergirem.

```nginx
server_tokens off;   # não anunciar versão do Nginx (mitigação de V4 nesta camada)

server {
    listen 80;

    add_header X-Content-Type-Options "nosniff" always;
    add_header X-Frame-Options "DENY" always;
    add_header Referrer-Policy "strict-origin-when-cross-origin" always;
    add_header Content-Security-Policy
      "default-src 'self'; script-src 'self'; style-src 'self' 'unsafe-inline'; img-src 'self' data:; connect-src 'self'; frame-ancestors 'none'; base-uri 'self'"
      always;

    location /api/ {
        proxy_pass http://protected-areas-sc-api:8000/api/;
        proxy_set_header Host $host;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }

    location / {
        root /usr/share/nginx/html;
        autoindex off;                # nunca listar diretório (mitigação de V5)
        try_files $uri /index.html;   # SPA: rotas do react-router
    }
}
```

### 4.3 Entrada no `docker-compose`

Novo serviço, mesma rede externa `pipeline` já usada pela API (`X:\fast-api-protected-areas-sc\compose.yaml`):

```yaml
services:
  frontend:
    build:
      context: .
      target: runtime
    container_name: protected-areas-sc-frontend
    ports:
      - "${FRONTEND_PORT:-3000}:80"
    restart: unless-stopped
    networks:
      - pipeline

networks:
  pipeline:
    external: true
```

Fica em repositório próprio (`frontend-protected-areas-sc`, este repositório), assim como
`pipeline-protected-areas-sc` e `fast-api-protected-areas-sc` já são repositórios separados —
mesmo padrão de "um serviço, um repositório, um Dockerfile", consistente com o resto do projeto.

## 5. Segurança — regras não negociáveis

Esta seção implementa os requisitos `FE-SEC-01` a `FE-SEC-07` do PRD. Nenhuma regra abaixo é
recomendação genérica de tutorial: cada uma neutraliza uma classe de falha efetivamente
explorada, reproduzida e classificada por CVSS/CWE no trabalho da disciplina de Segurança de
Sistemas do mesmo autor
(`F:\Univali\2026_2\seguranca_sistemas\roteirot_trab_m1\m1-seg-system-diegos`,
achados V1–V7, `relatorio/tabela-de-achados.md`). Isso muda a postura de revisão: uma mudança que
reintroduz qualquer padrão da coluna "O que foi explorado" abaixo é bug de segurança conhecido,
não descuido genérico.

### 5.1 Tabela de rastreabilidade achado → mitigação

| Achado (M1) | CWE / CVSS | O que foi explorado | Mitigação aplicada neste projeto |
|---|---|---|---|
| V1 — SQLi no login | CWE-89 / 9.1 Crítico | `' OR 1=1--` no campo de login autenticava sem senha | Toda query em `app_user`/`app_session`/`login_attempt` usa parâmetros vinculados do driver Postgres, nunca f-string/concatenação — mesmo padrão já seguido pelo resto da API em `infrastructure/persistence` |
| V2 — hash de senha exposto no JWT | CWE-522/916 / 6.5–7.5 | Payload de JWT é só Base64 — qualquer um decodifica em F12 → Network ou jwt.io; hash em MD5 sem sal | MVP usa sessão opaca em tabela (seção 2.1), não JWT, para o cookie de sessão; senha com Argon2; nenhuma resposta JSON da API inclui `password_hash` em nenhuma circunstância (regra de DTO, 5.2) |
| V3 — IDOR na cesta | CWE-639 / 6.5 | Servidor aceitava id de recurso alheio só por o token ser válido, sem checar posse | Toda rota com escopo de usuário deriva o dono/papel da sessão validada no servidor (linha em `app_session`/`app_user`), nunca de um valor enviado pelo cliente; `require_role("admin")` consulta o banco, não um claim do cliente |
| V4 — erro verboso + CSP/HSTS ausentes | CWE-209, CWE-16 / 5.3 | Cabeçalho malformado devolvia página de erro completa com framework/versão; sem CSP/HSTS | `debug=False` em produção, exception handler genérico com `correlation_id` (5.3); headers de segurança obrigatórios em toda resposta, incluindo `server_tokens off` |
| V5 — listagem de diretório | CWE-548 / 7.5 | `/ftp/` listava arquivos, um confidencial, sem autenticação | Nginx com `autoindex off`; nenhum caminho estático além do bundle de build é servido; API nunca expõe um diretório bruto |
| V6 — força bruta sem bloqueio | CWE-307 / 6.5 | 15 tentativas de senha errada seguidas, todas aceitas, sem atraso nem bloqueio | Bloqueio progressivo por conta no backend (5.4) — nunca só no frontend, contornável chamando a API direto |
| V7 — reset por pergunta de segurança sem limite | CWE-640 / 9.1 Crítico (achado mais grave da disciplina) | Pergunta de segurança adivinhável, tentativas ilimitadas → apropriação total da conta | Não existe fluxo de autoatendimento de redefinição de senha no MVP; reset é sempre ação direta do administrador autenticado (FE-SEC-06) |

### 5.2 "Se eu apertar F12, o que dá pra ver?" — regra de design

Regra geral: **assuma que tudo que chega ao navegador já vazou.** Cookie `httpOnly` impede
leitura por JavaScript/console, mas a aba Network sempre mostra corpo de requisição e resposta —
a pergunta certa não é "como eu escondo isso na tela", é "isso precisava ter saído do servidor?".

Consequências concretas de implementação:

- **DTOs de resposta são allowlist, nunca a entidade crua.** `GET /api/v1/auth/users` devolve um
  Pydantic `UserView` com só `id`, `username`, `nome`, `role`, `ativo`, `criado_em` — o modelo de
  persistência `AppUser` (que tem `password_hash`) nunca é serializado diretamente em nenhuma
  rota. Isso é testável: um teste de contrato falha a build se `password_hash` aparecer em
  qualquer resposta documentada no OpenAPI.
- **Sem `console.log` de dado sensível em produção.** Build de produção do Vite remove
  `console.*`/`debugger` (`esbuild: { drop: ["console", "debugger"] }` em `vite.config.ts`), então
  um `console.log(usuario)` esquecido em desenvolvimento não chega ao bundle publicado.
- **Sem source map público.** `build.sourcemap: false` no `vite.config.ts`. Sem isso, a aba
  "Sources" do F12 reconstrói o TypeScript original completo — nomes de variável, comentários,
  estrutura de pastas internas.
- **Nenhum segredo de servidor com prefixo `VITE_`.** O Vite só expõe ao bundle do cliente
  variáveis prefixadas com `VITE_`; string de conexão do banco, segredo de sessão, credencial do
  Airflow nunca podem ganhar esse prefixo — é o mecanismo exato pelo qual um segredo de servidor
  vazaria para dentro do JavaScript público, sem precisar de nenhum ataque.
- **Sessão só em cookie `httpOnly`, nunca em `localStorage`/`sessionStorage`/estado do React**
  (reforça a seção 2.1) — ambos são legíveis por `Application`/`Console` no F12 e pelo primeiro
  XSS que aparecer, exatamente como V2/V3 demonstraram na prática.

### 5.3 Cabeçalhos HTTP obrigatórios

Config completa em `nginx.conf` (seção 4.2) — aplicada no Nginx do frontend, cobrindo tanto o
bundle estático quanto as respostas da API via proxy, já que o navegador só enxerga a origem do
frontend (seção 1). `style-src 'unsafe-inline'` é concessão pragmática (bibliotecas de formulário
injetam estilo inline com frequência); `script-src` **não** tem `'unsafe-inline'` nem
`'unsafe-eval'` — um XSS refletido não consegue executar script inline mesmo que consiga injetar
HTML na página. `Strict-Transport-Security` entra quando houver TLS de verdade (fora do MVP
local) — ver 5.6.

No lado da API, `debug=False` em produção e um exception handler genérico
(`app/core/errors.py`, já existente) garantem que nenhuma resposta de erro carregue stack trace
ou banner de framework — só `type`, `title`, `detail` (mensagem segura) e `correlation_id`,
mitigação direta de V4.

### 5.4 Bloqueio de força bruta no login

Nova tabela, mesma migração `006_auth_users.sql`:

```sql
CREATE TABLE login_attempt (
    id            BIGSERIAL PRIMARY KEY,
    username      VARCHAR(120) NOT NULL,
    sucesso       BOOLEAN NOT NULL,
    ip_origem     INET,
    criado_em     TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP
);
CREATE INDEX idx_login_attempt_username_time ON login_attempt(username, criado_em);
```

Regra em `POST /api/v1/auth/login`: antes de checar a senha, contar tentativas malsucedidas do
mesmo `username` nos últimos 15 minutos; a partir de 5 tentativas, responder `429` com tempo de
espera — mesmo que a senha enviada agora esteja correta. Isso reproduz exatamente o cenário que
faltou em V6 (15 tentativas seguidas, todas `401`, nunca um `429`) e fica no backend porque
bloqueio só no frontend é cosmético: nada impede chamar `POST /api/v1/auth/login` direto, fora do
navegador, como o próprio script `scripts/v6_bruteforce_login.py` do M1 demonstrou.

### 5.5 CSRF

Como a sessão vive em cookie (não em header `Authorization` montado manualmente), toda rota de
mutação (`POST`/`PATCH`) é alvo potencial de CSRF. Mitigação em duas camadas: cookie
`SameSite=Lax` (navegador não envia o cookie em navegação cross-site de terceiros) **e** exigir
`Content-Type: application/json` nessas rotas — um `<form>` HTML simples de um site malicioso não
monta essa requisição sem JavaScript, e JavaScript de outra origem esbarra em CORS, que não está
liberado para nenhuma origem externa por desenho (seção 1: o navegador só fala com a origem do
frontend). Um token CSRF explícito (`double-submit cookie`) fica registrado como incremento se o
projeto sair do MVP local para um ambiente com múltiplas origens confiáveis.

### 5.6 O que fica como risco assumido do MVP — registrado, não ignorado

- **TLS/HTTPS**: MVP roda local/apresentação sem TLS; `Secure` no cookie e
  `Strict-Transport-Security` só fazem sentido com HTTPS real. Ativar antes de qualquer exposição
  fora da rede local — já registrado como risco no PRD (seção 9).
- **Rate limiting por IP** (além do bloqueio por conta em 5.4) fica para depois do MVP.
- **Expiração de sessão**: tempo fixo (ex.: 8h); revogação antecipada (logout, desativação de
  conta) é imediata porque a sessão é validada contra o banco a cada requisição — diferente de um
  JWT stateless, aqui não existe o problema de "token que não dá para revogar antes de expirar".

## 6. Testes: unitários e mutação

Teste unitário prova que o comportamento está certo para as entradas pensadas. Mutation testing
prova a coisa que teste unitário sozinho não prova: que o teste **quebraria** se a implementação
quebrasse. Para lógica de autorização, isso não é purismo acadêmico — um `role === "admin"` que
vira `role !== "admin"` por engano é exatamente o tipo de mutante que um teste "de forma" (só
confere que a função não lança exceção) deixa passar.

### 6.1 Backend (`app/application/auth.py`, `app/infrastructure/persistence/*`)

Unitários (pytest, mesmo padrão já usado no resto da API) cobrindo no mínimo:

- hash/verificação de senha com Argon2 (senha certa aceita, senha errada rejeitada, hash nunca
  igual ao texto plano);
- geração, consulta e expiração/revogação do token de sessão;
- `require_role("admin")` rejeitando uma sessão de `operador` com `403`, não `401` (papel errado é
  diferente de sessão ausente);
- contagem e reset da janela de bloqueio de força bruta (5.4): a 5ª tentativa malsucedida bloqueia,
  a 6ª com senha certa ainda é rejeitada dentro da janela, e a janela expira;
- `UserView` nunca serializa `password_hash` — o teste monta o JSON de resposta de verdade e
  afirma a ausência do campo, não só que o Pydantic model não o declara (protege contra alguém
  reintroduzir o campo displicentemente no futuro).

Mutation testing com `mutmut`, ainda não usado em nenhum dos dois repositórios irmãos — é
ferramenta nova, não continuidade de um padrão existente:

```toml
[tool.mutmut]
paths_to_mutate = ["app/application/auth.py", "app/infrastructure/persistence/auth_repository.py"]
tests_dir = "tests/"
runner = "python -m pytest -q"
```

Rodar pelo menos uma vez antes de considerar o módulo pronto, registrando a taxa de mutantes
mortos na evidência do módulo (mesmo padrão de `EVIDENCIAS_*.md`), não silenciosamente.

### 6.2 Frontend

Unitários com Vitest + React Testing Library (3.5), cobrindo explicitamente:
validação de campo por `ScenarioConfig` (zod), a tabela `ERROR_MESSAGES` (caminho conhecido e
fallback), e as guardas de rota `RequireAuth`/`RequireAdmin` — de novo, papel errado passando é
bug de segurança, não só de UX.

Mutation testing com Stryker Mutator, escopado no mínimo às guardas de rota e aos schemas de
validação — os dois lugares onde um operador trocado (`===`→`!==`, `>`→`<`) é falha de segurança:

```json
{
  "testRunner": "vitest",
  "mutate": ["src/auth/**/*.ts", "src/scenarios/**/*.ts"],
  "thresholds": { "high": 80, "low": 60, "break": 50 }
}
```

`break: 50` falha a build se a pontuação de mutação cair abaixo de 50% nesses arquivos críticos —
não é 100% desde o primeiro dia, mas impede que a qualidade dos testes regrida caladamente.

## 7. DevSecOps — o que ainda não estava mapeado

A seção 5 cobre o que um usuário vê/explora pelo navegador. Um tech lead DevSecOps também olha a
cadeia de build e a infraestrutura por trás. Tabela honesta: o que já existe nos repositórios
irmãos hoje, o que não existe em lugar nenhum, e o que este projeto recomenda fechar — inclusive
dívida técnica que **não** foi criada por este trabalho, mas que a chegada do módulo de auth é
uma oportunidade natural de corrigir.

| Prática | Estado atual nos repositórios irmãos | O que se recomenda aqui |
|---|---|---|
| SAST Python | `pyproject.toml` da API seleciona só `E, F, I, UP, B` no ruff — sem `S` (regras de segurança, equivalente ao bandit) | Adicionar `"S"` ao `select` do ruff da API, cobrindo o módulo de auth novo desde o primeiro commit; exceções pontuais (`# noqa: S...`) exigem comentário justificando, nunca silenciosas |
| SAST/lint de segurança do frontend | Não existia frontend até agora | ESLint com `eslint-plugin-security` (ou regra estrita equivalente do `typescript-eslint`) no CI |
| SCA (dependências) | Nenhum scan de dependência em nenhum dos dois repositórios | `pip-audit` (API) e `npm audit --audit-level=high` (frontend) como etapa obrigatória de CI; lockfile (`package-lock.json`) sempre commitado |
| Secret scanning | Nenhum | `gitleaks` como hook de pre-commit e job de CI nos três repositórios — relevante porque o projeto já lida com segredo real (`FIRMS_MAP_KEY`, credencial de bootstrap do admin, 2.3) |
| Scan de imagem de container | Nenhum | Trivy (ou Grype) sobre a imagem final do frontend e da API antes de qualquer publicação; imagens-base já são fixadas por tag versionada — digest explícito (`@sha256:...`) fica como incremento |
| Gate de CI | Nenhum `.github/workflows` existe em nenhum dos dois repositórios hoje; lint/teste só rodam manualmente (`docker compose --profile test run`) | Pipeline mínimo de CI (lint + unitários + SCA + secret scan) bloqueando merge em `main`; mutation testing (6) como job separado, não bloqueante enquanto a baseline de pontuação não é conhecida |
| Privilégio mínimo no banco | `init_db.sql` não define nenhuma `CREATE ROLE`/`GRANT` — API, Airflow e todo o domínio leem/escrevem com a mesma role dona do banco | Dívida pré-existente, não introduzida aqui. Antes de expor a API além do ambiente de desenvolvimento, criar uma role dedicada com `GRANT` restrito às tabelas que cada serviço usa de fato — `app_user`/`app_session`/`login_attempt` inclusas — em vez de estender o padrão de privilégio total para mais três tabelas |
| Política de senha | Nenhuma regra além de existir | `POST /auth/users` exige comprimento mínimo (ex.: 12 caracteres) validado **no servidor**; validação client-side é conveniência de UX, nunca controle de segurança |
| Renovação de sessão no login | — | Cada login gera um token novo, nunca reaproveita um token antigo da mesma conta — fecha o vetor clássico de fixação de sessão |
| Cabeçalhos adicionais de isolamento | CSP/X-Frame-Options já cobertos em 5.3 | Somar ao mesmo `nginx.conf`: `Permissions-Policy: camera=(), microphone=(), geolocation=()`, `Cross-Origin-Opener-Policy: same-origin`, `Cross-Origin-Resource-Policy: same-origin` |
| Log de evento de segurança | Log operacional com correlation ID já existe | Login malsucedido, bloqueio por força bruta e criação/desativação de conta geram uma linha estruturada distinguível (`event_type=security`) do log operacional comum — insumo para alerta futuro, mesmo sem SIEM hoje |
| LGPD sobre dado pessoal do usuário | Achados do M1 já classificam impacto por CID/LGPD | `app_user` guarda nome e identificador de contato; desativação (US-04) evita exclusão física para auditoria, mas a política de retenção/anonimização de conta desativada há muito tempo fica registrada como decisão pendente, não esquecida |
| HTML não confiável no React | — | Regra explícita: nunca `dangerouslySetInnerHTML` com conteúdo vindo da API (ex.: detalhe de erro expandido); se algum dia for necessário, passar por `DOMPurify` antes |

## 8. Ordem de implementação sugerida

1. Migração `006_auth_users.sql` (`app_user`, `app_session`, `login_attempt`) + módulo `auth` na
   API (login com bloqueio de força bruta 5.4, `/me`, logout, bootstrap do admin), já com `"S"` no
   ruff (7) e testes unitários (6.1) desde o primeiro commit do módulo.
2. Rotas `/auth/users` (admin) + `UserView` como allowlist explícita (teste de contrato que falha
   se `password_hash` aparecer em qualquer resposta, 5.2) + testes automatizados no padrão já
   usado nos demais módulos da API; primeira rodada de `mutmut` (6.1) registrada como evidência.
3. Scaffold do frontend (Vite + rotas + `AuthContext` contra a API já com login funcionando);
   `vite.config.ts` já criado com `build.sourcemap: false` e `esbuild.drop` de `console`/`debugger`
   (5.2) — não como ajuste posterior.
4. `ScenarioForm` parametrizado para os 4 cenários habilitados, com MSW cobrindo os testes antes
   de integrar com a API real ponta a ponta; Stryker (6.2) rodado sobre guardas de rota e schemas.
5. `AdminDashboardPage` (lista + criação de conta).
6. `nginx.conf` (4.2) com os cabeçalhos (incluindo os de 7), `autoindex off` e `server_tokens off`
   desde o primeiro Dockerfile funcional, não como hardening de última hora; entrada no compose;
   smoke test manual: subir tudo, logar como admin, criar um operador, logar como operador, rodar
   um cenário completo até `SUCCEEDED`, e conferir no F12 → Network que nenhuma resposta carrega
   `password_hash` ou stack trace.
7. CI mínimo (7): lint + unitários + `pip-audit`/`npm audit` + `gitleaks`, nos três repositórios.
8. Registrar evidência da execução ponta a ponta em `docs/` deste repositório, no mesmo padrão de
   `EVIDENCIAS_*.md` já usado no repositório da API — incluindo o teste de bloqueio de força bruta
   (5.4), a ausência de dado sensível na resposta (5.2) e a pontuação de mutação obtida (6).
