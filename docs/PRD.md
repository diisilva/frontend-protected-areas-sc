# PRD — Frontend de gestão geoespacial (Protected Areas SC)

## 1. Contexto

A `protected-areas-sc-api` (FastAPI) já implementa, ponta a ponta, quatro dos sete casos de uso
de interface definidos no TCC 2/TCC 3 (`UC-CW01`, `UC-CW02`, `UC-CW04`, `UC-CW07`), com o
caminho de escrita único `API → Bronze → Airflow → PostGIS → derivados temáticos`. Hoje o único
cliente é o Swagger/Postman (`docs/GUIA_TESTES_POSTMAN.md`), documentado no próprio README da API
como estado temporário: "a interface web visual ainda será implementada sobre esse contrato".

Este documento define o produto mínimo viável (MVP) de uma interface web que substitui o
Postman por um formulário operável por uma pessoa não técnica, com controle de acesso por
contas nominais.

Não existe hoje nenhuma autenticação/autorização na API (`HANDOFF.md` da API, item pendente
4: "Adicionar autenticação/autorização antes de exposição fora do ambiente de
desenvolvimento"). Este PRD assume que esse backend de autenticação é construído junto com o
frontend, como já decidido: estende a própria FastAPI existente, sem serviço novo. O SDD
correspondente (`docs/SDD.md`) detalha esse desenho.

## 2. Problema e objetivo

**Problema:** hoje, publicar ou atualizar uma UC/ZA exige montar manualmente uma requisição
multipart no Postman/Swagger, conhecer o contrato de `domain`/`operation`/`metadata` e ler a
resposta JSON para saber se deu certo. Isso não escala para outra pessoa operar o cadastro sem
apoio técnico direto.

**Objetivo do MVP:** uma página web simples, em formulário, autenticada, onde:

1. o administrador (o autor do TCC) entra com sua conta e tem acesso a um painel próprio para
   criar/gerenciar contas de outras pessoas;
2. qualquer pessoa autenticada (administrador ou operador) acessa um formulário para executar os
   cenários de cadastro geoespacial já implementados na API e acompanhar o resultado até o estado
   final, sem precisar ler JSON bruto ou montar requisições manualmente.

## 3. Fora de escopo do MVP

Registrado aqui para não ser esquecido, não para ser descartado — quando a API implementar cada
item, o front volta a este documento:

- **`UC-CW03`** (criar UC + ZA oficial juntas, lote atômico) — API ainda não implementa
  `/import-batches`. O formulário deve deixar essa opção **visível e desabilitada**, com um aviso
  "ainda não disponível", em vez de simplesmente escondida — mantém o desenho dos 7 cenários
  rastreável para a banca sem prometer uma função inexistente.
- **`UC-CW05`** (extinção de UC — "deletar") — API não tem esse endpoint/operação hoje. Mesmo
  tratamento: opção visível, desabilitada, com aviso.
- **`UC-CW06`** (substituir ponto por polígono) — idem.
- **`za_oficial/create` isolado** — bloqueado por desenho na própria API
  (`ZA_CREATE_REQUIRES_BATCH`); não existe cenário de "subir ZA do zero" sem UC associada. O que
  existe e fica no MVP é `UC-CW07`, que **substitui** a ZAS de uma UC já cadastrada por uma ZA
  oficial.
- Permissões granulares por cenário (ex.: "este operador só pode atualizar, não pode criar").
  MVP tem só dois papéis fixos: administrador e operador.
- Autoatendimento de conta (cadastro público) e recuperação de senha por e-mail — não há
  infraestrutura de e-mail no projeto. O administrador define/reseta a senha diretamente.
- Qualquer tela para PRODES, MapBiomas, MapBiomas Alerta ou FIRMS — esses domínios não são
  entrada da API (`ATCC-023`) e não têm cenário de interface no TCC.
- Internacionalização (só pt-BR), tema escuro, layout mobile-first pixel perfect — a ferramenta é
  de uso interno, em desktop.

## 4. Usuários e papéis

| Papel | Quem | Acesso |
|---|---|---|
| **Administrador** | Só o autor do TCC nesta fase (conta única, pode criar outras) | Painel de administração (listar/criar/desativar contas de operador) **+** formulário de cenários, igual a um operador |
| **Operador** | Pessoas convidadas pelo administrador | Somente o formulário de cenários. Sem acesso ao painel de administração, sem visão de outras contas |

Não existe autocadastro. Toda conta nasce criada pelo administrador pelo próprio painel.

## 5. Histórias de usuário

### Administrador

- **US-01** Como administrador, eu entro com usuário e senha e sou redirecionado para o painel
  de administração (não para o formulário), porque meu papel me dá acesso extra.
- **US-02** Como administrador, eu crio uma conta de operador informando nome, e-mail/usuário e
  uma senha inicial, para que essa pessoa possa operar o formulário de cenários.
- **US-03** Como administrador, eu vejo a lista de contas existentes (nome, papel, status
  ativo/inativo, data de criação), para saber quem tem acesso.
- **US-04** Como administrador, eu desativo uma conta (sem excluir o registro, preservando
  histórico de quem publicou o quê), para revogar acesso sem perder rastreabilidade — mesmo
  princípio de "vigência sem exclusão física" já usado no resto do projeto para UC/ZA.
- **US-05** Como administrador, eu também acesso o formulário de cenários (US-06 a US-12), porque
  meu papel inclui tudo que um operador pode fazer.

### Administrador ou operador

- **US-06** Como usuário autenticado, eu entro com usuário e senha e sou redirecionado para o
  formulário de cenários.
- **US-07** Como usuário autenticado, eu escolho o cenário "Cadastrar UC nova" (`UC-CW01`), envio
  um arquivo (Shapefile ZIP, GeoJSON ou KML) com uma ou mais UCs, preencho os metadados exigidos
  (fonte; nome só se o arquivo não trouxer), escolho a política de duplicidade
  (`reject_batch`/`skip_duplicates`) e envio.
- **US-08** Como usuário autenticado, eu escolho o cenário "Cadastrar UC sem polígono"
  (`UC-CW02`), envio um arquivo com exatamente um ponto e os metadados mínimos.
- **US-09** Como usuário autenticado, eu escolho o cenário "Atualizar geometria de UC existente"
  (`UC-CW04`), informo o identificador forte da UC (`cd_cnuc`/`wdpa_pid`/outro), a versão que
  acredito ser a vigente, a justificativa e o novo arquivo, e recebo um erro claro se a versão
  informada já estiver desatualizada (conflito de concorrência), sem me obrigar a interpretar o
  código de erro bruto da API.
- **US-10** Como usuário autenticado, eu escolho o cenário "Substituir ZAS por ZA oficial"
  (`UC-CW07`), informo o identificador da UC, a justificativa e o arquivo da ZA oficial (polígono).
- **US-11** Como usuário autenticado, depois de enviar qualquer cenário, eu acompanho o
  andamento (`PROCESSING` → `SUCCEEDED`/`FAILED`) sem precisar atualizar a página manualmente, e
  vejo mensagens de erro traduzidas para linguagem de negócio quando falha (ex.: "Essa UC já
  existe" em vez de `UC_ALREADY_EXISTS`).
- **US-12** Como usuário autenticado, eu vejo minhas últimas importações enviadas (histórico
  simples: cenário, arquivo, data, estado final), para conferir o que já publiquei sem precisar
  anotar em outro lugar.

## 6. Requisitos funcionais

| ID | Requisito |
|---|---|
| FE-RF-01 | O sistema deve exigir autenticação (usuário/senha) para qualquer tela além da de login. |
| FE-RF-02 | O sistema deve distinguir papel `admin` e `operador` e redirecionar após login conforme o papel (US-01/US-06). |
| FE-RF-03 | Somente contas `admin` podem criar novas contas; a tela de criação de conta não deve ser alcançável por um operador, nem por URL direta. |
| FE-RF-04 | O painel de administração deve listar, criar e desativar contas (US-02 a US-04). O MVP não inclui edição de papel após criação nem exclusão física. |
| FE-RF-05 | O formulário de cenários deve oferecer exatamente os cenários habilitados na API: criar UC (polígono/lote), criar UC pontual, atualizar UC, substituir ZAS por ZA oficial. |
| FE-RF-06 | Os cenários fora de escopo (`UC-CW03`, `UC-CW05`, `UC-CW06`) devem aparecer na lista, visivelmente desabilitados, com o texto "ainda não implementado na API". |
| FE-RF-07 | Cada formulário de cenário deve validar no cliente os campos obrigatórios do contrato antes de habilitar o envio (evita ida e volta desnecessária ao servidor para erro óbvio). |
| FE-RF-08 | O sistema deve gerar automaticamente a `Idempotency-Key` de cada envio (o usuário não deve digitar isso). |
| FE-RF-09 | Após o envio, o sistema deve acompanhar o estado da importação (polling) até um estado terminal e exibir o resultado (sucesso, com link/id publicado; ou falha, com mensagem traduzida e o código original disponível em detalhe expansível). |
| FE-RF-10 | O sistema deve manter, por usuário, um histórico local das últimas importações enviadas por ele nesta sessão/conta (US-12). Fonte de verdade continua sendo a API; não duplicar armazenamento de domínio no frontend. |
| FE-RF-11 | Erros da API em `application/problem+json` devem ser mapeados para mensagens em português, com fallback genérico para códigos não mapeados (nunca esconder o erro, só traduzir o já conhecido). |

## 7. Requisitos não funcionais

| ID | Requisito |
|---|---|
| FE-RNF-01 | Interface e mensagens em português (pt-BR). |
| FE-RNF-02 | Uso interno, desktop-first; responsividade básica é desejável, não é critério de aceite do MVP. |
| FE-RNF-03 | Sessão de login não deve expor token em `localStorage`; ver SDD para desenho de sessão. |
| FE-RNF-04 | Toda ação de mutação (criar/atualizar/desativar conta; publicar cenário) deve ficar auditável do lado da API (quem fez, quando) — o frontend não é a fonte de auditoria, só a exibe quando disponível. |
| FE-RNF-05 | O frontend não deve reimplementar nenhuma regra de negócio geoespacial (CRS, validação de geometria, duplicidade) — toda validação de domínio continua exclusiva da API, o frontend só valida forma/obrigatoriedade de campo. |
| FE-RNF-06 | Rodar como serviço Docker dedicado, na mesma rede `pipeline` externa dos demais serviços do projeto. |

## 8. Critérios de aceite do MVP

1. Um administrador consegue: logar, criar uma conta de operador, deslogar; o operador criado
   consegue logar com a senha definida e **não** vê o painel de administração.
2. Um usuário autenticado (admin ou operador) consegue executar `UC-CW01`, `UC-CW02`, `UC-CW04` e
   `UC-CW07` de ponta a ponta pela interface, sem usar Postman, e ver o resultado final
   (`SUCCEEDED`/`FAILED`) na tela.
3. Um erro conhecido da API (ex.: `UC_ALREADY_EXISTS`, `UC_VERSION_CONFLICT`) aparece traduzido
   na tela, não como JSON bruto.
4. Os três cenários fora de escopo aparecem na lista, desabilitados, sem quebrar a navegação.
5. O front sobe via `docker compose up` como um serviço próprio, documentado no SDD.

## 9. Requisitos de segurança (o que o produto garante)

Estas regras não são recomendação genérica de mercado: cada uma neutraliza uma classe de falha
que foi efetivamente explorada, reproduzida e classificada por CVSS/CWE no trabalho da disciplina
de Segurança de Sistemas do mesmo autor
(`F:\Univali\2026_2\seguranca_sistemas\roteirot_trab_m1\m1-seg-system-diegos`,
`relatorio/tabela-de-achados.md`, achados V1–V7). O detalhamento técnico de cada mitigação está
em `docs/SDD.md`, seção 5; aqui ficam os requisitos do ponto de vista do produto — o que o
sistema **garante**, não como.

| ID | Requisito | Achado que motiva |
|---|---|---|
| FE-SEC-01 | Nenhum dado sensível (hash de senha, segredo, corpo de erro interno) pode aparecer em qualquer resposta HTTP visível na aba Network do navegador (F12), nem em `localStorage`/`sessionStorage`/console. | V2 — hash de senha (MD5) vazado no payload do JWT, trivialmente decodificável em qualquer decodificador Base64/jwt.io. |
| FE-SEC-02 | Toda decisão de autorização (quem vê o quê, quem pode fazer o quê) é resolvida no backend a partir da sessão validada no servidor — nunca a partir de um valor enviado pelo cliente, e nunca apenas por a interface esconder um botão ou link. | V3 — IDOR: o servidor aceitava acessar um recurso de outro usuário só porque o token era válido, sem checar posse do recurso. |
| FE-SEC-03 | Erros retornados ao navegador nunca contêm stack trace, nome/versão de framework nem detalhe de infraestrutura interna; o detalhe técnico fica só no log do servidor, correlacionado por um identificador reportável pelo usuário. | V4 — cabeçalho malformado provocava página de erro completa expondo `Express ^4.22.1`. |
| FE-SEC-04 | Nenhum diretório do servidor (estático ou de dados) é listável publicamente; todo caminho servido é um arquivo explicitamente esperado pela aplicação. | V5 — listagem de diretório expôs um documento marcado "confidencial", sem autenticação. |
| FE-SEC-05 | O login tem proteção contra força bruta (bloqueio/atraso após tentativas malsucedidas consecutivas), aplicada no backend, não só na tela. | V6 — 15 tentativas de senha errada seguidas foram todas aceitas, sem bloqueio nem atraso. |
| FE-SEC-06 | Não existe, e não está planejado, nenhum fluxo de redefinição de senha por pergunta de segurança nem qualquer segredo de baixa entropia adivinhável; redefinir senha é sempre uma ação direta do administrador (US-04/painel). | V7 — pergunta de segurança sem limite de tentativas permitiu apropriação total de conta; **achado mais crítico da disciplina (CVSS 9.1)**. |
| FE-SEC-07 | Toda consulta às tabelas novas de autenticação usa parâmetros vinculados, nunca concatenação de string — sem exceção, incluindo código gerado rapidamente sob pressão de prazo. | V1 — SQL Injection no login (`' OR 1=1--`) autenticava sem senha; **CVSS 9.1**, achado didático mais citado da disciplina. |

## 10. Riscos e dependências

- **Dependência bloqueante:** o backend de autenticação (login, papéis, criação de conta) não
  existe hoje na API. Este PRD assume que ele é implementado em paralelo — ver `docs/SDD.md` para
  o contrato exato de endpoints necessário antes do frontend poder autenticar de verdade. Até lá,
  o frontend pode ser desenvolvido contra um mock desse contrato.
- **Risco de escopo:** o usuário final (você) também é o único administrador hoje; qualquer
  decisão de "múltiplos administradores" fica para depois do MVP.
- **Risco de rastreabilidade acadêmica:** como os 7 casos de uso do TCC já são um contrato
  formal citado em `PLANO_REESCRITA_TCC3.md`/`AJUSTES_TCC3.md`, qualquer divergência de nome ou
  fluxo entre este PRD e esses documentos deve ser registrada como ajuste rastreável, não só
  corrigida silenciosamente no front.
