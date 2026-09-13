# Frontend — Protected Areas SC

Interface web para operar os cenários de cadastro geoespacial da
[`protected-areas-sc-api`](https://github.com/diisilva/fast-api-protected-areas-sc) sem depender
de Postman/Swagger, com contas nominais (administrador + operador).

## Estado atual

Fase de design: `docs/PRD.md` e `docs/SDD.md` definem escopo, papéis, contrato de autenticação
novo (a implementar na própria FastAPI) e arquitetura. Nenhum código de aplicação foi escrito
ainda. Ver `HANDOFF.md` para o estado exato e o que falta antes do primeiro código.

## Documentação

- [`docs/PRD.md`](docs/PRD.md) — o quê e por quê: problema, papéis, histórias de usuário,
  requisitos funcionais e de segurança, fora de escopo do MVP.
- [`docs/SDD.md`](docs/SDD.md) — como: arquitetura, contrato de autenticação, modelo de dados,
  componentização do frontend, empacotamento Docker, testes (unitários e mutação) e práticas
  DevSecOps.
- [`HANDOFF.md`](HANDOFF.md) — estado técnico consolidado para retomada do trabalho.

## Repositórios relacionados

- `protected-areas-sc-api` (FastAPI, backend REST + autenticação nova).
- `pipeline-protected-areas-sc` (Airflow, pipelines Medallion e publicação PostGIS).

Este repositório entra na mesma rede Docker externa `pipeline` que os outros dois já usam, como
um serviço a mais (ver `docs/SDD.md`, seção 4).
