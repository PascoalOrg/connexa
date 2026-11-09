# ADR-001 — Arquitetura e stack do Connexa: monolito modular .NET + SPA React

- **Situação:** aceita
- **Data:** 2026-11-09
- **Participantes:** Cauê Santos Gonçalves da Silva, Lucas Frangetto Péri, Nicolas Gomes Pascoal, Pedro Schwartz de Souza Reina, Wendel Miranda

## Contexto

O Connexa será construído por uma equipe de cinco estudantes, ao longo de um semestre, em sprints de duas semanas. A carga esperada é modesta (uso acadêmico), e o sistema tem territórios de negócio bem distintos: identidade, perfil, grupos, encontros, acompanhamento, painel e materiais.

As restrições que pesam na decisão:

- **Equipe pequena e prazo curto:** não há tempo para operar vários serviços, um broker de mensagens e implantações independentes.
- **Fronteiras precisam ser garantidas, não só combinadas:** com um agente de IA escrevendo boa parte do código, regras que dependem só de disciplina se perdem; elas precisam ser conferidas pelo build.
- **Conhecimento prévio:** parte da equipe já trabalha com C#/.NET e com React + TypeScript.
- **Ambiente igual para todos:** cada integrante pode usar Codespaces ou a própria máquina, e o CI precisa se comportar do mesmo jeito.
- **Autenticação fora da Sprint 1:** os módulos precisam saber "quem é o usuário atual" sem depender de como o login será feito.

## Decisão

Adotamos um **monolito modular em .NET 10 (ASP.NET Core Minimal APIs, C#)**, com um projeto `Connexa.<Modulo>` por território e um schema por módulo no **PostgreSQL 17** via **EF Core + Npgsql**, servido a uma **SPA em React 19 + TypeScript (Vite)**, conforme o modelo `connexa-modelo-dotnet`.

Fazem parte da decisão:

- Tudo o que é interno a um módulo fica em `Internal/` com visibilidade `internal`; o namespace raiz do módulo é a sua API pública. `Connexa.Comum` não tem regra de negócio e `Connexa.Api` só monta a aplicação.
- Cada módulo é dono dos seus dados: um `DbContext` e um schema por módulo, sem chave estrangeira entre módulos (guarda-se só o identificador). O banco só muda por migração do EF Core revisada em pull request, aplicada quando a aplicação sobe.
- Reações entre módulos acontecem por eventos em processo (`IDespachanteDeEventos` / `ITratadorDeEvento<T>`).
- Erros da API seguem Problem Detail (RFC 9457) com a propriedade `codigo`; testes e front-end comparam o `codigo`, nunca o texto.
- Identidade simulada por `IUsuarioAtual` (cabeçalho `X-Connexa-Usuario-Simulado`), sem ASP.NET Core Identity na Sprint 1.
- Testes de aceitação com MSTest + `WebApplicationFactory` em um PostgreSQL real separado (`connexa_teste`); front-end com Vitest + Testing Library.
- Versões fixas em `Directory.Packages.props` e `package.json`, com `TreatWarningsAsErrors` ligado.

## Alternativas consideradas

| Alternativa | Por que não foi escolhida |
|---|---|
| Microsserviços | Uma equipe de cinco pessoas, em um semestre e com carga modesta, não paga o custo de vários serviços, implantações independentes e um broker de mensagens. |
| Monolito em camadas (sem módulos) | Camadas organizam o código tecnicamente, mas não impedem que um território use as entidades e os dados de outro; o acoplamento cresceria sem que o build percebesse. |
| Modelo em Java (Spring Boot) ou Node.js | Também atenderiam ao estilo, mas a equipe tem mais familiaridade com C#/.NET, e o modelo .NET já traz as fronteiras garantidas pelo compilador (`internal` por projeto) e pelo NetArchTest. |
| Testcontainers para os testes | Exigiria Docker dentro do Codespace; o banco `connexa_teste` do Compose funciona igual no Codespaces, no CI e na máquina local. Pode ser adotado depois, por um novo ADR. |
| ASP.NET Core Identity já na Sprint 1 | A autenticação está fora do escopo da sprint; `IUsuarioAtual` permite trocar a implementação depois mexendo só no interior de `Connexa.Identidade`. |

## Consequências

- **Positivas:**
  - Uma única unidade de back-end para implantar, depurar e testar.
  - O compilador impede o uso do interior de outro módulo e recusa referências circulares entre projetos; o que ele não alcança, o NetArchTest confere em todo build.
  - Cada história vira uma fatia vertical (cenário Gherkin → teste de aceitação → migração → código do módulo → contrato em `/swagger` → tela), sempre no mesmo caminho.
  - Mesmo comportamento em desenvolvimento, testes e CI, sem passos manuais de banco.
- **Negativas:**
  - Mais cerimônia que um projeto único: cada módulo exige projeto, contexto, schema e registro próprios.
  - Consultas que cruzam módulos não podem usar join com chave estrangeira; precisam passar pela API pública do outro módulo ou por eventos.
  - Incluir ou atualizar uma dependência, criar um módulo novo ou referenciar outro módulo exige um ADR.
  - Escalar partes do sistema de forma independente não é possível sem extrair módulos para serviços no futuro.
- **Como verificar:**
  - `ArquiteturaModularTest` (NetArchTest) e `SchemasDosModulosTest` no `dotnet test`, rodados pelo `./verificar` e pelo check **Back-end (testes e fronteiras)** do CI.
  - Regra `no-restricted-imports` do ESLint no check **Front-end (lint, testes e build)**.
  - `TreatWarningsAsErrors` no build e revisão obrigatória de pull request (ruleset `proteger-main`) para toda migração e toda mudança de dependência.
