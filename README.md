# dotnet-backend — Skill de padrão para backend .NET

Skill para agentes de IA com o meu padrão de arquitetura
e boas práticas para APIs REST em .NET (ASP.NET Core, EF Core, PostgreSQL, JWT, xUnit).

Com a skill instalada, quando você pede algo de backend .NET (criar projeto, CRUD, endpoint,
service, migration, testes...), o agente carrega essas regras sozinho e gera o código já
dentro do padrão, sem você repetir as instruções a cada pedido.

## Como surgiu

1. Escrevi uma API REST em .NET inteira à mão, sem IA (o projeto **CRUD.PRODUTOS**).
2. Revisei o projeto com a IA e ajustei o que estava fora de clean code e da arquitetura
   que eu queria.
3. Transformei as decisões desse projeto nesta skill: regras, o porquê de cada uma,
   templates de cada camada e um checklist para criar entidades novas.

## O que a skill cobre

| Seção | Resumo |
|---|---|
| Arquitetura | 5 projetos: `DOMAIN`, `APPLICATION`, `DATA`, `API`, `TESTS`, com regras de dependência |
| Build e pacotes | `global.json`, Central Package Management, `TreatWarningsAsErrors` |
| Domínio | `EntityBase`, exceções de domínio mapeadas para status HTTP |
| Persistência | Fluent API, Unit of Work, auditoria em UTC, paginação no banco |
| Aplicação | Services com interface, DTOs por intenção, mapeamento manual, Options tipadas |
| API | Controllers enxutos, `GlobalExceptionHandler` → ProblemDetails, Swagger documentado |
| Segurança | Nenhum segredo versionado, JWT validado no start, BCrypt, sem mass assignment |
| Testes | xUnit + Moq + Shouldly, builders, EF InMemory, testes de validação |
| Estilo | Nomes de negócio em PT-BR, comentários explicando o porquê |
| Checklist | 10 passos para uma nova entidade/CRUD, do model aos testes |

## Estrutura

```
dotnet-backend/
├─ SKILL.md              # nome + descrição (gatilho) + regras + checklist
└─ references/
   └─ templates.md       # código pronto de cada camada, lido só quando necessário
```

## Instalação

**Para você, em todos os projetos:**

```bash
git clone https://github.com/IgorHAlves/dotnet-backend-skill.git ~/.claude/skills/dotnet-backend
```

**Para o time, em um repositório específico** (a skill vai versionada junto com o código):

```bash
git clone https://github.com/IgorHAlves/dotnet-backend-skill.git .claude/skills/dotnet-backend
rm -rf .claude/skills/dotnet-backend/.git
```

Para atualizar a instalação pessoal: `git -C ~/.claude/skills/dotnet-backend pull`.

## Como usar

Não precisa chamar a skill. Peça normalmente, por exemplo:

> Cria o backend de um sistema de quiz com cadastro de perguntas e respostas.

O agente lê a descrição da skill, percebe que é backend .NET e aplica o padrão. Em projetos
que já têm outro padrão estabelecido, a skill segue o padrão local e só *sugere* a migração.

## Adaptando para o seu time

A skill é só Markdown. Faça um fork, edite o `SKILL.md` com as decisões do seu time e
mantenha o **porquê** de cada regra: é isso que ajuda a IA a acertar nos casos não previstos.
Trate como código: mudanças por PR, com revisão.
