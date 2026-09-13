# Handoff — Frontend

## Estado em 13/09/2026

Este repositório existe até agora só como design: `docs/PRD.md` e `docs/SDD.md` estão escritos e
cobrem escopo, papéis (administrador único + operadores criados por ele), os quatro cenários
habilitados (`UC-CW01`, `UC-CW02`, `UC-CW04`, `UC-CW07`), o contrato de autenticação novo que
precisa entrar na `protected-areas-sc-api`, arquitetura (Nginx com reverse proxy, sessão em cookie
`httpOnly`), segurança (seção 5 do SDD, com rastreabilidade explícita aos achados V1–V7 do
trabalho de Segurança de Sistemas do mesmo autor), estratégia de testes unitários e mutação
(seção 6) e o mapeamento de práticas DevSecOps ainda pendentes (seção 7).

**Nenhum código de aplicação existe ainda** — nem o módulo de autenticação na API, nem o scaffold
do React. Nenhum `.github/workflows` de CI existe neste nem nos dois repositórios irmãos.

## Controle de versão entre os três repositórios

Commit e push só acontecem quando o responsável pedir explicitamente, nunca automaticamente
depois de uma implementação — vale para este repositório e para `pipeline-protected-areas-sc` e
`fast-api-protected-areas-sc`. Qualquer mudança que não estivesse prevista no desenho do TCC 3 já
defendido perante a banca deve atualizar
`X:\fast-api-protected-areas-sc\PLANO_REESCRITA_TCC3.md` no momento em que acontece,
independentemente de em qual dos três repositórios ela ocorreu — esse documento é a base para
reescrever os capítulos finais do TCC.

## Dependência bloqueante

O frontend não consegue autenticar de verdade enquanto `protected-areas-sc-api` não implementar o
módulo `auth` descrito em `docs/SDD.md` (seção 2): migração `006_auth_users.sql`
(`app_user`, `app_session`, `login_attempt`), rotas `/api/v1/auth/*`, bootstrap do primeiro
administrador por variável de ambiente. Até lá, o desenvolvimento do frontend pode avançar contra
um mock desse contrato (MSW, seção 3.5/6.2), mas a integração ponta a ponta depende dessa entrega
na API.

## Retomada

1. Implementar o módulo `auth` na API (`docs/SDD.md`, seções 2, 5.4 e 6.1), incluindo os testes
   unitários e a primeira rodada de `mutmut` antes de considerar o módulo pronto.
2. Scaffold do React (Vite + TypeScript) com `vite.config.ts` já endurecido
   (`build.sourcemap: false`, `esbuild.drop` de `console`/`debugger`) desde o primeiro commit, não
   como ajuste posterior — seção 5.2 do SDD.
3. `ScenarioForm` parametrizado + `AdminDashboardPage`, com MSW cobrindo os testes antes da
   integração real.
4. `Dockerfile` + `nginx.conf` com os cabeçalhos de segurança (5.3, 7) desde o primeiro build
   funcional.
5. CI mínimo (lint + unitários + `npm audit`/`pip-audit` + `gitleaks`) — hoje não existe em
   nenhum dos três repositórios do projeto; ver `docs/SDD.md`, seção 7.

## Cuidados operacionais

- Não declarar nenhum cenário de `UC-CW03`/`UC-CW05`/`UC-CW06` como "implementado" no frontend
  antes de a API expor o endpoint correspondente — devem aparecer desabilitados, nunca escondidos
  (PRD, seção 3).
- Toda checagem de autorização é decidida no backend a partir da sessão validada no servidor;
  nunca tratar a UI escondendo um botão como controle de acesso (PRD, `FE-SEC-02`).
- Antes de qualquer exposição fora da rede local/apresentação, resolver TLS — `Secure` no cookie
  de sessão não tem efeito sem HTTPS real (SDD, seção 5.6).

Nenhum commit de código de aplicação foi criado até esta entrega; só os documentos de design.
