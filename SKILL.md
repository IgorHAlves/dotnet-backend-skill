---
name: dotnet-backend
description: Padrões de arquitetura e boas práticas do Igor para backend .NET (ASP.NET Core Web API, EF Core, PostgreSQL, JWT, xUnit). Use SEMPRE que a tarefa envolver backend em .NET/C# — criar projeto/solution, nova entidade ou CRUD, endpoint/controller, service, repository, DTO, migration, autenticação, tratamento de erros, configuração, testes de backend, ou revisar/refatorar código .NET — mesmo que o usuário não peça explicitamente "siga o padrão".
---

# Backend .NET — padrão do Igor

Arquitetura e convenções extraídas do projeto de referência `~/Projects/CRUD-PRODUTOS`
(API REST .NET 10 + PostgreSQL + JWT). Aplique-as em qualquer backend .NET.

- **Projeto novo:** aplique tudo abaixo desde o início.
- **Projeto existente com outro padrão já estabelecido:** siga o padrão local no código novo
  e *sugira* a migração para este — não reestruture sem pedir.
- Templates de código prontos: [references/templates.md](references/templates.md). Leia-o
  antes de criar arquivos de uma camada que ainda não existe no projeto.
- Na dúvida sobre um detalhe, consulte o projeto de referência se ele existir na máquina.

## 1. Arquitetura em camadas

Solution com 5 projetos, nomeados `{PROJETO}.{CAMADA}` (ex.: `CRUD.PRODUTOS.API`):

| Projeto | Conteúdo | Referencia |
|---|---|---|
| `DOMAIN` | Entidades (`Models/`), `EntityBase`, interfaces de repositório e `IUnitOfWork` (`Repositories/`), exceções de domínio (`Exceptions/`), constantes (`Common/`, ex. `Roles`), abstrações de infra (`Security/IPasswordHasher`) | nada (zero pacotes) |
| `APPLICATION` | Services + interfaces (`Services/`), DTOs por agregado (`DTOs/{Entidade}/`), mapeamentos (`Mappings/`), `ResultadoPaginado<T>` (`Common/`), Options tipadas (`Configuration/`), `DependencyInjection.AddApplication` | DOMAIN |
| `DATA` | `AppDBContext`, `UnitOfWork`, `Configurations/` (Fluent API), `Repositories/`, `Migrations/`, implementações de infra (`Security/BCryptPasswordHasher`), `DependencyInjection.AddPersistence` | DOMAIN |
| `API` | Controllers, `ExceptionHandling/GlobalExceptionHandler`, `Extensions/` (JWT, Swagger), `Program.cs` enxuto | APPLICATION, DATA |
| `TESTS` | `Factories/` (builders, DbContext de teste), `ServicesTests/`, `RepositoriesTests/`, `ValidationTests/` | APPLICATION, DATA |

Regras de dependência:
- O domínio não conhece EF Core, ASP.NET, BCrypt nem JWT. Se o domínio precisa de algo de
  infra, crie uma interface no DOMAIN e implemente no DATA (ex.: `IPasswordHasher`).
- Controllers só falam com **interfaces de service**. Nunca com repositório ou DbContext.
- Services recebem DTOs e devolvem DTOs; entidades não saem da camada de aplicação.
- Cada camada expõe **um** método de extensão de DI (`AddPersistence`, `AddApplication`);
  o `Program.cs` só os encadeia.

## 2. Build e pacotes

- `global.json` fixando o SDK (`rollForward: latestFeature`).
- **Central Package Management**: `Directory.Packages.props` com
  `ManagePackageVersionsCentrally=true`, versões agrupadas por `Label`
  (Infraestrutura, Extensions, API, Testes); os `.csproj` referenciam pacotes **sem** `Version`.
- Todos os projetos de produção: `Nullable=enable`, `ImplicitUsings=enable`,
  `TreatWarningsAsErrors=true`.
- API: `GenerateDocumentationFile=true` + `NoWarn 1591` (XML docs alimentam o Swagger).
- `Microsoft.EntityFrameworkCore.Design` com `PrivateAssets=all` na API e no DATA.
- `public partial class Program;` no fim do `Program.cs` (para testes de integração).
- Stack padrão: EF Core + Npgsql, JWT Bearer, Swashbuckle, BCrypt.Net-Next,
  xUnit + Moq + Shouldly + EF InMemory + coverlet. Use a versão estável mais recente do .NET.

## 3. Domínio

- Toda entidade herda `EntityBase` (`Id`, `DataCriacao`, `DataAlteracao`).
  As datas de auditoria **nunca** são preenchidas por quem chama — o `UnitOfWork` faz isso.
- Strings obrigatórias como `required string`; opcionais como `string?`.
- Valores enumerados como constantes em classe estática (`Roles.Admin`, `Roles.EhValida(...)`),
  usadas também nos atributos `[Authorize(Roles = Roles.Admin)]`.
- Exceções previsíveis herdam `DomainException` (abstrata):
  `NaoEncontradoException` → 404, `RegraDeNegocioException` → 400,
  `CredenciaisInvalidasException` → 401. Use factories estáticas para mensagens repetidas
  (`NaoEncontradoException.Produto(id)`). Crie novos tipos só se precisar de outro status HTTP.

## 4. Persistência (DATA)

- `AppDBContext` com `DbSet<T> X => Set<T>()` e `ApplyConfigurationsFromAssembly`.
- Uma classe `IEntityTypeConfiguration<T>` por entidade: `ToTable`, `HasKey`, `IsRequired`,
  `HasMaxLength` em **toda** string (igual ao limite do DTO), `HasPrecision(18, 2)` em decimal,
  índices nos campos de busca, `IsUnique()` em chaves naturais (login, e-mail).
- **Unit of Work**: repositórios só *marcam* alterações (`Add`, `Remove`, alterar entidade
  rastreada); quem persiste é o service via `IUnitOfWork.CommitAsync`. O `CommitAsync`
  preenche as datas de auditoria em UTC via `TimeProvider` (injetado como singleton
  `TimeProvider.System`) e protege `DataCriacao` de alteração.
- Repositórios:
  - `Add` (não `AddAsync`) retornando `Task.CompletedTask`.
  - Leituras com `AsNoTracking()` por padrão; `ObterPorIdAsync(id, bool rastrear = false, ct)`
    quando a entidade será editada/removida.
  - Paginação: o repositório devolve `(IReadOnlyList<T> Itens, int TotalItens)`, com
    `OrderBy` determinístico antes de `Skip/Take`; quem monta `ResultadoPaginado` é o service.
  - Filtro textual no banco com `EF.Functions.ILike` (Postgres), nunca em memória.
- Migrations com nomes descritivos em PT-BR (`MigracaoInicial`, `AjusteRestricoesDeColunas`).
  Comando: `dotnet ef migrations add Nome --project X.DATA --startup-project X.API`.
- PostgreSQL local em **container Docker dedicado por projeto**, numa porta livre diferente
  de 5432 (confira com `docker ps` antes de escolher). Documente o `docker run` no README.

## 5. Aplicação (APPLICATION)

- Interface + implementação por service (`IProdutoService`/`ProdutoService`), registrados
  como `Scoped`. Métodos `async`, sufixo `Async`, **sempre** com
  `CancellationToken cancellationToken = default` repassado até o EF.
- Padrão de busca-ou-erro:
  `var x = await _repo.ObterPorIdAsync(id, rastrear: true, ct) ?? throw NaoEncontradoException.X(id);`
- Regras de negócio lançam `RegraDeNegocioException`; nunca retornam `null`/bool de erro.
- DTOs separados por intenção: `Criar{E}DTO`, `Editar{E}DTO`, `Visualizar{E}DTO`,
  `Filtro{E}DTO`. Classes com `init`; entrada validada com DataAnnotations e
  `ErrorMessage` em PT-BR; saída com `required`. Filtro com `Page`/`Limit` limitados
  (`[Range(1, LimiteMaximo)]`, máx. 100).
- Mapeamento manual em métodos de extensão estáticos (`ParaVisualizacao`, `ParaEntidade`,
  `AplicarEm`) em `Mappings/{E}Mappings.cs`. Sem AutoMapper. Faça `Trim()` nas strings.
- Configuração via **Options tipadas** com `SectionName`, DataAnnotations e
  `.ValidateDataAnnotations().ValidateOnStart()` — a app não sobe com config inválida.

## 6. API

- Controller: `[ApiController]`, `[Route("api/[controller]")]`, `[Produces("application/json")]`,
  `[Authorize]` na classe quando tudo exige login; `[Authorize(Roles = Roles.Admin)]` em escrita.
- Construtor injeta **só** a interface do service. Sem try/catch, sem lógica — uma chamada
  ao service e o retorno HTTP.
- Retornos: GET → `Ok(dto)`; POST → `CreatedAtAction(nameof(GetById), new { id }, null)`;
  PUT/DELETE/ações sem corpo → `NoContent()`. Rotas com constraint (`{id:int}`).
- Toda action documentada: `/// <summary>`, `<param>`, `<response code="...">` e
  `[ProducesResponseType]` para cada status (erros como `ProblemDetails` /
  `ValidationProblemDetails`).
- Erros: `GlobalExceptionHandler : IExceptionHandler` + `AddProblemDetails()` +
  `app.UseExceptionHandler()`. Mapeia `DomainException`s para status via `switch`;
  500 loga a exceção inteira mas devolve mensagem genérica (nunca stack trace ao cliente).
- `Program.cs` enxuto: Controllers, ProblemDetails, handler, TimeProvider, `AddPersistence`,
  `AddApplication`, `AddJwtAuthentication`, `AddSwaggerDocs`. Swagger só em Development.
- Configuração do JWT Bearer e do Swagger (com esquema Bearer e XML comments) em
  `Extensions/`, lendo `IOptions<JwtOptions>` — nada de valor embutido.

## 7. Segurança (não negociável)

- **Nenhum segredo versionado.** `appsettings.json` traz as chaves vazias; valores reais via
  `dotnet user-secrets` (dev) ou variáveis de ambiente (prod). Falhe na inicialização com
  mensagem que ensina o comando `user-secrets` quando faltar.
- JWT: chave ≥ 32 caracteres validada no start; validar issuer, audience, lifetime e
  signing key; `ClockSkew` curto (30s); claims `sub`, `jti`, `NameIdentifier`, `Name`, `Role`.
- Senhas com BCrypt atrás de `IPasswordHasher`; mínimo 8 caracteres.
- Auto-cadastro **nunca** aceita perfil/role do cliente (evita mass assignment); promoção
  só por endpoint restrito a Admin.
- Login com mensagem genérica ("Login ou senha inválidos") para usuário inexistente e
  senha errada.
- DTOs de entrada nunca expõem campos que o cliente não deve controlar.
- Limites de paginação no DTO para impedir páginas gigantes.

## 8. Testes

- xUnit + **Moq** (`MockBehavior.Strict`) + **Shouldly**.
- Nomes em PT-BR: `Should_Criar_Produto`, `Should_Throw_Editar_Produto_Nao_Encontrado`.
- Corpo com comentários `//Arrange`, `//Act`, `//Assert`.
- `Factories/{E}Builder` com métodos estáticos de parâmetros opcionais e valores válidos
  por padrão — cada teste declara só o que importa.
- **Services**: isolados do banco com mocks de repositório e `IUnitOfWork`; verifique que
  `CommitAsync` foi (ou não) chamado.
- **Repositories**: EF InMemory, um banco por teste (`Guid.NewGuid()` no nome),
  `IDisposable` na classe; teste auditoria (UTC, `DataCriacao` preservada) e `AsNoTracking`.
- **Validation**: `Validator.TryValidateObject(dto, new ValidationContext(dto), resultados, true)`
  cobrindo limites e campos que não devem ser expostos.
- Rode `dotnet build` e `dotnet test` ao terminar; build tem de sair sem warnings.

## 9. Estilo de código

- Identificadores de negócio em **português** (`Produto`, `ListarProdutosAsync`, `SenhaHash`,
  `ResultadoPaginado`); termos técnicos consagrados podem ficar em inglês (`Page`, `Limit`,
  `Hash`, `Verify`, `CommitAsync`).
- Namespaces file-scoped; campos privados `_camelCase` `readonly`; injeção por construtor.
- Comentários e XML docs em PT-BR explicando o **porquê** (ex.: por que `Add` e não
  `AddAsync`), não o óbvio.
- Coleções de retorno como `IReadOnlyList<T>`; inicialização com `[]`.
- README em PT-BR com: tecnologias, funcionalidades, como subir o banco (docker),
  como configurar segredos, migrations, rodar, testes e fluxo de autenticação.

## Checklist — nova entidade/CRUD

1. `DOMAIN/Models/{E}.cs` herdando `EntityBase`.
2. `DOMAIN/Repositories/I{E}Repository.cs` (+ factory em `NaoEncontradoException` se útil).
3. `DATA/Data/Configurations/{E}Configuration.cs` + `DbSet` no `AppDBContext`.
4. `DATA/Repositories/{E}Repository.cs` + registro em `AddPersistence`.
5. `APPLICATION/DTOs/{E}/` (Criar, Editar, Visualizar, Filtro) com limites iguais ao Fluent API.
6. `APPLICATION/Mappings/{E}Mappings.cs`.
7. `APPLICATION/Services/I{E}Service.cs` + `{E}Service.cs` + registro em `AddApplication`.
8. `API/Controllers/{E}Controller.cs` documentado.
9. Migration descritiva.
10. Testes: builder, service, repository, validação dos DTOs. `dotnet build && dotnet test`.
