# Templates — backend .NET

Substitua `{P}` pelo prefixo da solution (ex.: `CRUD.PRODUTOS`), `{E}` pela entidade
(ex.: `Produto`) e `{Es}` pelo plural (ex.: `Produtos`). Adapte campos ao caso real.

## Índice
1. Arquivos da raiz (global.json, Directory.Packages.props, csproj)
2. DOMAIN (EntityBase, entidade, exceções, repositório, UoW)
3. DATA (DbContext, UnitOfWork, Configuration, Repository, DI)
4. APPLICATION (DTOs, Mappings, ResultadoPaginado, Service, Options, DI)
5. API (Program, Controller, GlobalExceptionHandler, Extensions, appsettings)
6. TESTS (Builder, DbContext de teste, service, repository, validação)
7. Comandos

---

## 1. Raiz

### global.json
```json
{
  "sdk": {
    "version": "10.0.100",
    "rollForward": "latestFeature"
  }
}
```

### Directory.Packages.props
```xml
<Project>

    <!--
      Versões centralizadas: cada csproj referencia o pacote sem <Version>,
      o que evita que as camadas divirjam (origem dos conflitos MSB3277).
    -->
    <PropertyGroup>
        <ManagePackageVersionsCentrally>true</ManagePackageVersionsCentrally>
    </PropertyGroup>

    <ItemGroup Label="Infraestrutura">
        <PackageVersion Include="BCrypt.Net-Next" Version="..." />
        <PackageVersion Include="Microsoft.EntityFrameworkCore" Version="..." />
        <PackageVersion Include="Microsoft.EntityFrameworkCore.Design" Version="..." />
        <PackageVersion Include="Microsoft.EntityFrameworkCore.Relational" Version="..." />
        <PackageVersion Include="Npgsql.EntityFrameworkCore.PostgreSQL" Version="..." />
    </ItemGroup>

    <ItemGroup Label="Extensions">
        <PackageVersion Include="Microsoft.Extensions.DependencyInjection.Abstractions" Version="..." />
        <PackageVersion Include="Microsoft.Extensions.Options.ConfigurationExtensions" Version="..." />
        <PackageVersion Include="Microsoft.Extensions.Options.DataAnnotations" Version="..." />
    </ItemGroup>

    <ItemGroup Label="API">
        <PackageVersion Include="Microsoft.AspNetCore.Authentication.JwtBearer" Version="..." />
        <PackageVersion Include="System.IdentityModel.Tokens.Jwt" Version="..." />
        <PackageVersion Include="Swashbuckle.AspNetCore" Version="..." />
    </ItemGroup>

    <ItemGroup Label="Testes">
        <PackageVersion Include="Microsoft.EntityFrameworkCore.InMemory" Version="..." />
        <PackageVersion Include="Microsoft.NET.Test.Sdk" Version="..." />
        <PackageVersion Include="Moq" Version="..." />
        <PackageVersion Include="Shouldly" Version="..." />
        <PackageVersion Include="coverlet.collector" Version="..." />
        <PackageVersion Include="xunit" Version="..." />
        <PackageVersion Include="xunit.runner.visualstudio" Version="..." />
    </ItemGroup>

</Project>
```
Use as versões estáveis mais recentes compatíveis com o TargetFramework (verifique no NuGet).

### PropertyGroup comum (DOMAIN, APPLICATION, DATA)
```xml
<PropertyGroup>
    <TargetFramework>net10.0</TargetFramework>
    <ImplicitUsings>enable</ImplicitUsings>
    <Nullable>enable</Nullable>
    <TreatWarningsAsErrors>true</TreatWarningsAsErrors>
</PropertyGroup>
```
API acrescenta:
```xml
<GenerateDocumentationFile>true</GenerateDocumentationFile>
<NoWarn>$(NoWarn);1591</NoWarn>
<UserSecretsId>...</UserSecretsId>  <!-- gerado por dotnet user-secrets init -->
```
e referencia o Design assim:
```xml
<!-- Necessário para o dotnet-ef enxergar o projeto de inicialização. -->
<PackageReference Include="Microsoft.EntityFrameworkCore.Design">
    <PrivateAssets>all</PrivateAssets>
    <IncludeAssets>runtime; build; native; contentfiles; analyzers; buildtransitive</IncludeAssets>
</PackageReference>
```

---

## 2. DOMAIN

### Models/EntityBase.cs
```csharp
namespace {P}.DOMAIN.Models;

public abstract class EntityBase
{
    public int Id { get; set; }

    /// <summary>
    /// Preenchida automaticamente pelo <c>UnitOfWork</c> ao persistir.
    /// </summary>
    public DateTime DataCriacao { get; set; }

    /// <summary>
    /// Preenchida automaticamente pelo <c>UnitOfWork</c> a cada alteração.
    /// </summary>
    public DateTime DataAlteracao { get; set; }
}
```

### Models/{E}.cs
```csharp
namespace {P}.DOMAIN.Models;

public class Produto : EntityBase
{
    public required string Nome { get; set; }
    public string? Descricao { get; set; }
    public decimal Preco { get; set; }
    public int QuantidadeEmEstoque { get; set; }
}
```

### Exceptions/
```csharp
namespace {P}.DOMAIN.Exceptions;

/// <summary>
/// Base para as exceções previsíveis do domínio, traduzidas em respostas HTTP
/// pelo handler global da API.
/// </summary>
public abstract class DomainException : Exception
{
    protected DomainException(string mensagem) : base(mensagem)
    {
    }
}

/// <summary>
/// Recurso solicitado não existe. Traduzida em 404.
/// </summary>
public class NaoEncontradoException : DomainException
{
    public NaoEncontradoException(string mensagem) : base(mensagem)
    {
    }

    public static NaoEncontradoException Produto(int id) =>
        new($"Produto {id} não encontrado");
}

/// <summary>
/// Violação de uma regra de negócio. Traduzida em 400.
/// </summary>
public class RegraDeNegocioException : DomainException
{
    public RegraDeNegocioException(string mensagem) : base(mensagem)
    {
    }
}

/// <summary>
/// Login ou senha inválidos. Traduzida em 401, sempre com mensagem genérica
/// para não revelar quais logins existem.
/// </summary>
public class CredenciaisInvalidasException : DomainException
{
    public CredenciaisInvalidasException() : base("Login ou senha inválidos")
    {
    }
}
```
(um arquivo por classe)

### Common/Roles.cs
```csharp
namespace {P}.DOMAIN.Common;

/// <summary>
/// Perfis de acesso reconhecidos pela aplicação.
/// </summary>
public static class Roles
{
    public const string Admin = "Admin";
    public const string Padrao = "Padrao";

    public static bool EhValida(string role) =>
        role is Admin or Padrao;
}
```

### Repositories/I{E}Repository.cs e IUnitOfWork.cs
```csharp
namespace {P}.DOMAIN.Repositories;

public interface IProdutoRepository
{
    Task AdicionarAsync(Produto produto, CancellationToken cancellationToken = default);

    /// <param name="rastrear">
    /// <c>true</c> quando a entidade será alterada e precisa ser acompanhada pelo
    /// change tracker; <c>false</c> (padrão) para consultas somente leitura.
    /// </param>
    Task<Produto?> ObterPorIdAsync(int id, bool rastrear = false, CancellationToken cancellationToken = default);

    /// <summary>
    /// Retorna a página solicitada e o total de itens que satisfazem o filtro.
    /// A montagem do resultado paginado é responsabilidade da camada de aplicação.
    /// </summary>
    Task<(IReadOnlyList<Produto> Itens, int TotalItens)> BuscarAsync(
        string? nomeProduto,
        int page,
        int limit,
        CancellationToken cancellationToken = default);

    void Remover(Produto produto);
}

public interface IUnitOfWork
{
    Task CommitAsync(CancellationToken cancellationToken = default);
}
```

### Security/IPasswordHasher.cs
```csharp
namespace {P}.DOMAIN.Security;

/// <summary>
/// Abstrai o algoritmo de hash de senha para que o domínio não dependa
/// de uma biblioteca de criptografia específica.
/// </summary>
public interface IPasswordHasher
{
    string Hash(string senha);

    bool Verify(string senha, string hash);
}
```

---

## 3. DATA

### Data/AppDBContext.cs
```csharp
using System.Reflection;
using {P}.DOMAIN.Models;
using Microsoft.EntityFrameworkCore;

namespace {P}.DATA.Data;

public class AppDBContext : DbContext
{
    public AppDBContext(DbContextOptions<AppDBContext> options) : base(options)
    {
    }

    public DbSet<Produto> Produtos => Set<Produto>();

    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        base.OnModelCreating(modelBuilder);

        modelBuilder.ApplyConfigurationsFromAssembly(Assembly.GetExecutingAssembly());
    }
}
```

### Data/UnitOfWork.cs
```csharp
using {P}.DOMAIN.Models;
using {P}.DOMAIN.Repositories;
using Microsoft.EntityFrameworkCore;

namespace {P}.DATA.Data;

public class UnitOfWork : IUnitOfWork
{
    private readonly AppDBContext _dbContext;
    private readonly TimeProvider _timeProvider;

    public UnitOfWork(AppDBContext dbContext, TimeProvider timeProvider)
    {
        _dbContext = dbContext;
        _timeProvider = timeProvider;
    }

    public async Task CommitAsync(CancellationToken cancellationToken = default)
    {
        AplicarDatasDeAuditoria();

        await _dbContext.SaveChangesAsync(cancellationToken);
    }

    /// <summary>
    /// Preenche DataCriacao/DataAlteracao de forma centralizada. Sem isso as
    /// colunas "timestamp with time zone" recebem DateTime.MinValue (Kind
    /// Unspecified), que o Npgsql rejeita em tempo de execução.
    /// </summary>
    private void AplicarDatasDeAuditoria()
    {
        var agora = _timeProvider.GetUtcNow().UtcDateTime;

        foreach (var entry in _dbContext.ChangeTracker.Entries<EntityBase>())
        {
            switch (entry.State)
            {
                case EntityState.Added:
                    entry.Entity.DataCriacao = agora;
                    entry.Entity.DataAlteracao = agora;
                    break;

                case EntityState.Modified:
                    entry.Property(e => e.DataCriacao).IsModified = false;
                    entry.Entity.DataAlteracao = agora;
                    break;
            }
        }
    }
}
```

### Data/Configurations/{E}Configuration.cs
```csharp
using {P}.DOMAIN.Models;
using Microsoft.EntityFrameworkCore;
using Microsoft.EntityFrameworkCore.Metadata.Builders;

namespace {P}.DATA.Data.Configurations;

public class ProdutoConfiguration : IEntityTypeConfiguration<Produto>
{
    public void Configure(EntityTypeBuilder<Produto> builder)
    {
        builder.ToTable("Produtos");

        builder.HasKey(p => p.Id);

        builder.Property(p => p.Nome)
            .IsRequired()
            .HasMaxLength(150);

        builder.Property(p => p.Descricao)
            .HasMaxLength(1000);

        // numeric sem precisão aceita qualquer escala; fixar evita divergência
        // entre o decimal do .NET e o que é gravado no Postgres.
        builder.Property(p => p.Preco)
            .HasPrecision(18, 2);

        builder.HasIndex(p => p.Nome);
    }
}
```

### Repositories/{E}Repository.cs
```csharp
using {P}.DATA.Data;
using {P}.DOMAIN.Models;
using {P}.DOMAIN.Repositories;
using Microsoft.EntityFrameworkCore;

namespace {P}.DATA.Repositories;

public class ProdutoRepository : IProdutoRepository
{
    private readonly AppDBContext _dbContext;

    public ProdutoRepository(AppDBContext dbContext)
    {
        _dbContext = dbContext;
    }

    public Task AdicionarAsync(Produto produto, CancellationToken cancellationToken = default)
    {
        // Add (e não AddAsync): AddAsync só é necessário para geradores de valor
        // que consultam o banco, como o HiLo. Aqui a identity é resolvida no insert.
        _dbContext.Produtos.Add(produto);

        return Task.CompletedTask;
    }

    public async Task<Produto?> ObterPorIdAsync(
        int id,
        bool rastrear = false,
        CancellationToken cancellationToken = default)
    {
        var query = _dbContext.Produtos.AsQueryable();

        if (!rastrear)
            query = query.AsNoTracking();

        return await query.FirstOrDefaultAsync(p => p.Id == id, cancellationToken);
    }

    public async Task<(IReadOnlyList<Produto> Itens, int TotalItens)> BuscarAsync(
        string? nomeProduto,
        int page,
        int limit,
        CancellationToken cancellationToken = default)
    {
        var query = _dbContext.Produtos.AsNoTracking();

        if (!string.IsNullOrWhiteSpace(nomeProduto))
        {
            // ILIKE é resolvido pelo Postgres e, ao contrário de LOWER(...) LIKE,
            // permite o uso de índice (com pg_trgm) na coluna Nome.
            var termo = $"%{nomeProduto.Trim()}%";
            query = query.Where(p => EF.Functions.ILike(p.Nome, termo));
        }

        var totalItens = await query.CountAsync(cancellationToken);

        var itens = await query
            .OrderBy(p => p.Id)
            .Skip((page - 1) * limit)
            .Take(limit)
            .ToListAsync(cancellationToken);

        return (itens, totalItens);
    }

    public void Remover(Produto produto)
    {
        _dbContext.Produtos.Remove(produto);
    }
}
```

### Security/BCryptPasswordHasher.cs
```csharp
namespace {P}.DATA.Security;

/// <summary>
/// Implementação de <see cref="IPasswordHasher"/> com BCrypt. Fica na camada de
/// infraestrutura para manter o domínio livre de dependências de criptografia.
/// </summary>
public class BCryptPasswordHasher : IPasswordHasher
{
    public string Hash(string senha) => BCrypt.Net.BCrypt.HashPassword(senha);

    public bool Verify(string senha, string hash) => BCrypt.Net.BCrypt.Verify(senha, hash);
}
```

### DependencyInjection.cs
```csharp
namespace {P}.DATA;

public static class DependencyInjection
{
    public static IServiceCollection AddPersistence(this IServiceCollection services, IConfiguration configuration)
    {
        var connectionString = configuration.GetConnectionString("DefaultConnection");

        if (string.IsNullOrWhiteSpace(connectionString))
        {
            throw new InvalidOperationException(
                "ConnectionStrings:DefaultConnection não configurada. " +
                "Use 'dotnet user-secrets set \"ConnectionStrings:DefaultConnection\" \"<conexão>\"'.");
        }

        services.AddDbContext<AppDBContext>(options => options.UseNpgsql(connectionString));

        services.AddScoped<IUnitOfWork, UnitOfWork>();
        services.AddScoped<IProdutoRepository, ProdutoRepository>();
        services.AddSingleton<IPasswordHasher, BCryptPasswordHasher>();

        return services;
    }
}
```

---

## 4. APPLICATION

### DTOs/{E}/
```csharp
using System.ComponentModel.DataAnnotations;

namespace {P}.APPLICATION.DTOs.Produto;

public class CriarProdutoDTO   // EditarProdutoDTO tem a mesma forma
{
    [Required(ErrorMessage = "O nome é obrigatório.")]
    [StringLength(150, MinimumLength = 2, ErrorMessage = "O nome deve ter entre 2 e 150 caracteres.")]
    public string Nome { get; init; } = null!;

    [StringLength(1000, ErrorMessage = "A descrição deve ter no máximo 1000 caracteres.")]
    public string? Descricao { get; init; }

    [Range(0.01, (double)decimal.MaxValue, ErrorMessage = "O preço deve ser maior que zero.")]
    public decimal Preco { get; init; }

    [Range(0, int.MaxValue, ErrorMessage = "A quantidade em estoque não pode ser negativa.")]
    public int QuantidadeEmEstoque { get; init; }
}

public class VisualizarProdutoDTO
{
    public required int Id { get; init; }
    public required string Nome { get; init; }
    public string? Descricao { get; init; }
    public required decimal Preco { get; init; }
    public required int QuantidadeEmEstoque { get; init; }
}

/// <summary>
/// Filtro e paginação da listagem. Os limites impedem que
/// um cliente peça uma página arbitrariamente grande.
/// </summary>
public class FiltroProdutoDTO
{
    public const int LimiteMaximo = 100;

    [StringLength(150)]
    public string? NomeProduto { get; init; }

    [Range(1, int.MaxValue, ErrorMessage = "A página deve ser maior ou igual a 1.")]
    public int Page { get; init; } = 1;

    [Range(1, LimiteMaximo, ErrorMessage = "O limite deve estar entre 1 e 100.")]
    public int Limit { get; init; } = 10;
}
```

### Common/ResultadoPaginado.cs
```csharp
namespace {P}.APPLICATION.Common;

/// <summary>
/// Página de resultados devolvida pelos serviços de consulta.
/// </summary>
public class ResultadoPaginado<T>
{
    public IReadOnlyList<T> Itens { get; init; } = [];
    public int TotalItens { get; init; }
    public int PaginaAtual { get; init; }
    public int TotalPaginas { get; init; }

    public static ResultadoPaginado<T> Criar(IReadOnlyList<T> itens, int totalItens, int page, int limit) =>
        new()
        {
            Itens = itens,
            TotalItens = totalItens,
            PaginaAtual = page,
            TotalPaginas = limit <= 0 ? 0 : (int)Math.Ceiling(totalItens / (double)limit)
        };
}
```

### Mappings/{E}Mappings.cs
```csharp
namespace {P}.APPLICATION.Mappings;

/// <summary>
/// Conversões entre a entidade e seus DTOs, centralizadas para
/// não se repetirem em cada serviço.
/// </summary>
public static class ProdutoMappings
{
    public static VisualizarProdutoDTO ParaVisualizacao(this Produto produto) =>
        new()
        {
            Id = produto.Id,
            Nome = produto.Nome,
            Descricao = produto.Descricao,
            Preco = produto.Preco,
            QuantidadeEmEstoque = produto.QuantidadeEmEstoque
        };

    public static IReadOnlyList<VisualizarProdutoDTO> ParaVisualizacao(this IEnumerable<Produto> produtos) =>
        produtos.Select(ParaVisualizacao).ToList();

    public static Produto ParaEntidade(this CriarProdutoDTO dto) =>
        new()
        {
            Nome = dto.Nome.Trim(),
            Descricao = dto.Descricao?.Trim(),
            Preco = dto.Preco,
            QuantidadeEmEstoque = dto.QuantidadeEmEstoque
        };

    public static void AplicarEm(this EditarProdutoDTO dto, Produto produto)
    {
        produto.Nome = dto.Nome.Trim();
        produto.Descricao = dto.Descricao?.Trim();
        produto.Preco = dto.Preco;
        produto.QuantidadeEmEstoque = dto.QuantidadeEmEstoque;
    }
}
```

### Services/{E}Service.cs
```csharp
namespace {P}.APPLICATION.Services;

public class ProdutoService : IProdutoService
{
    private readonly IProdutoRepository _produtoRepository;
    private readonly IUnitOfWork _unitOfWork;

    public ProdutoService(IProdutoRepository produtoRepository, IUnitOfWork unitOfWork)
    {
        _produtoRepository = produtoRepository;
        _unitOfWork = unitOfWork;
    }

    public async Task<VisualizarProdutoDTO> ListarProdutoAsync(int id, CancellationToken cancellationToken = default)
    {
        var produto = await _produtoRepository.ObterPorIdAsync(id, cancellationToken: cancellationToken)
                      ?? throw NaoEncontradoException.Produto(id);

        return produto.ParaVisualizacao();
    }

    public async Task<ResultadoPaginado<VisualizarProdutoDTO>> ListarProdutosAsync(
        FiltroProdutoDTO filtro,
        CancellationToken cancellationToken = default)
    {
        var (itens, totalItens) = await _produtoRepository.BuscarAsync(
            filtro.NomeProduto, filtro.Page, filtro.Limit, cancellationToken);

        return ResultadoPaginado<VisualizarProdutoDTO>.Criar(
            itens.ParaVisualizacao(), totalItens, filtro.Page, filtro.Limit);
    }

    public async Task<int> CriarProdutoAsync(CriarProdutoDTO dto, CancellationToken cancellationToken = default)
    {
        var produto = dto.ParaEntidade();

        await _produtoRepository.AdicionarAsync(produto, cancellationToken);
        await _unitOfWork.CommitAsync(cancellationToken);

        return produto.Id;
    }

    public async Task EditarProdutoAsync(int id, EditarProdutoDTO dto, CancellationToken cancellationToken = default)
    {
        var produto = await _produtoRepository.ObterPorIdAsync(id, rastrear: true, cancellationToken)
                      ?? throw NaoEncontradoException.Produto(id);

        dto.AplicarEm(produto);

        await _unitOfWork.CommitAsync(cancellationToken);
    }

    public async Task DeletarProdutoAsync(int id, CancellationToken cancellationToken = default)
    {
        var produto = await _produtoRepository.ObterPorIdAsync(id, rastrear: true, cancellationToken)
                      ?? throw NaoEncontradoException.Produto(id);

        _produtoRepository.Remover(produto);

        await _unitOfWork.CommitAsync(cancellationToken);
    }
}
```

### Configuration/JwtOptions.cs (modelo de Options tipada)
```csharp
using System.ComponentModel.DataAnnotations;

namespace {P}.APPLICATION.Configuration;

/// <summary>
/// Configuração do JWT, validada na inicialização da aplicação.
/// </summary>
public class JwtOptions
{
    public const string SectionName = "Jwt";

    /// <summary>Chave simétrica de assinatura. Nunca versionar: use user-secrets ou variável de ambiente.</summary>
    [Required(ErrorMessage = "Jwt:Key não configurada. Use 'dotnet user-secrets set \"Jwt:Key\" \"<chave>\"'.")]
    [MinLength(32, ErrorMessage = "Jwt:Key deve ter no mínimo 32 caracteres.")]
    public string Key { get; init; } = string.Empty;

    [Required]
    public string Issuer { get; init; } = string.Empty;

    [Required]
    public string Audience { get; init; } = string.Empty;

    [Range(1, 1440)]
    public int ExpireMinutes { get; init; } = 60;
}
```

### DependencyInjection.cs
```csharp
namespace {P}.APPLICATION;

public static class DependencyInjection
{
    public static IServiceCollection AddApplication(this IServiceCollection services, IConfiguration configuration)
    {
        services
            .AddOptions<JwtOptions>()
            .Bind(configuration.GetSection(JwtOptions.SectionName))
            .ValidateDataAnnotations()
            .ValidateOnStart();

        services.AddScoped<IProdutoService, ProdutoService>();

        return services;
    }
}
```

### Geração de token (AuthService)
```csharp
private TokenResponseDTO GerarToken(Usuario usuario)
{
    var expiraEm = DateTime.UtcNow.AddMinutes(_jwtOptions.ExpireMinutes);

    var claims = new List<Claim>
    {
        new(JwtRegisteredClaimNames.Sub, usuario.Id.ToString()),
        new(JwtRegisteredClaimNames.Jti, Guid.NewGuid().ToString()),
        new(ClaimTypes.NameIdentifier, usuario.Id.ToString()),
        new(ClaimTypes.Name, usuario.Login),
        new(ClaimTypes.Role, usuario.Role)
    };

    var credenciais = new SigningCredentials(
        new SymmetricSecurityKey(Encoding.UTF8.GetBytes(_jwtOptions.Key)),
        SecurityAlgorithms.HmacSha256);

    var token = new JwtSecurityToken(
        issuer: _jwtOptions.Issuer,
        audience: _jwtOptions.Audience,
        claims: claims,
        expires: expiraEm,
        signingCredentials: credenciais);

    return new TokenResponseDTO
    {
        Token = new JwtSecurityTokenHandler().WriteToken(token),
        ExpiraEm = expiraEm
    };
}
```
Registro: checar `ExisteLoginAsync` → `RegraDeNegocioException("Login já cadastrado")`;
`Role = Roles.Padrao` fixo. Login: usuário inexistente **e** senha errada lançam
`CredenciaisInvalidasException`.

---

## 5. API

### Program.cs
```csharp
using {P}.API.Extensions;
using {P}.API.ExceptionHandling;
using {P}.APPLICATION;
using {P}.DATA;

var builder = WebApplication.CreateBuilder(args);

builder.Services.AddControllers();
builder.Services.AddProblemDetails();
builder.Services.AddExceptionHandler<GlobalExceptionHandler>();
builder.Services.AddSingleton(TimeProvider.System);

builder.Services.AddPersistence(builder.Configuration);
builder.Services.AddApplication(builder.Configuration);
builder.Services.AddJwtAuthentication();
builder.Services.AddSwaggerDocs();

var app = builder.Build();

app.UseExceptionHandler();

if (app.Environment.IsDevelopment())
{
    app.UseSwagger();
    app.UseSwaggerUI(c => c.SwaggerEndpoint("/swagger/v1/swagger.json", "API V1"));
}

app.UseHttpsRedirection();

app.UseAuthentication();
app.UseAuthorization();

app.MapControllers();

app.Run();

/// <summary>Exposto para os testes de integração.</summary>
public partial class Program;
```

### Controllers/{E}Controller.cs
```csharp
namespace {P}.API.Controllers;

[ApiController]
[Route("api/[controller]")]
[Authorize]
[Produces("application/json")]
public class ProdutoController : ControllerBase
{
    private readonly IProdutoService _produtoService;

    public ProdutoController(IProdutoService produtoService)
    {
        _produtoService = produtoService;
    }

    /// <summary>
    /// Retorna um produto pelo Id.
    /// </summary>
    /// <param name="id">Identificador do produto.</param>
    /// <param name="cancellationToken">Token de cancelamento da requisição.</param>
    /// <response code="200">Produto encontrado.</response>
    /// <response code="404">Produto não encontrado.</response>
    [HttpGet("{id:int}")]
    [ProducesResponseType(typeof(VisualizarProdutoDTO), StatusCodes.Status200OK)]
    [ProducesResponseType(typeof(ProblemDetails), StatusCodes.Status404NotFound)]
    public async Task<ActionResult<VisualizarProdutoDTO>> GetById(int id, CancellationToken cancellationToken)
    {
        return Ok(await _produtoService.ListarProdutoAsync(id, cancellationToken));
    }

    /// <summary>
    /// Lista produtos de forma paginada, opcionalmente filtrando pelo nome.
    /// </summary>
    /// <response code="200">Lista de produtos retornada.</response>
    /// <response code="400">Parâmetros de paginação inválidos.</response>
    [HttpGet]
    [ProducesResponseType(typeof(ResultadoPaginado<VisualizarProdutoDTO>), StatusCodes.Status200OK)]
    [ProducesResponseType(typeof(ValidationProblemDetails), StatusCodes.Status400BadRequest)]
    public async Task<ActionResult<ResultadoPaginado<VisualizarProdutoDTO>>> Get(
        [FromQuery] FiltroProdutoDTO filtro,
        CancellationToken cancellationToken)
    {
        return Ok(await _produtoService.ListarProdutosAsync(filtro, cancellationToken));
    }

    /// <summary>
    /// Cadastra um novo produto.
    /// </summary>
    /// <response code="201">Produto criado com sucesso.</response>
    /// <response code="400">Dados inválidos.</response>
    /// <response code="403">Usuário autenticado não é administrador.</response>
    [Authorize(Roles = Roles.Admin)]
    [HttpPost]
    [ProducesResponseType(StatusCodes.Status201Created)]
    [ProducesResponseType(typeof(ValidationProblemDetails), StatusCodes.Status400BadRequest)]
    public async Task<IActionResult> Post([FromBody] CriarProdutoDTO dto, CancellationToken cancellationToken)
    {
        var id = await _produtoService.CriarProdutoAsync(dto, cancellationToken);

        return CreatedAtAction(nameof(GetById), new { id }, null);
    }

    // PUT {id:int} → EditarProdutoAsync → NoContent()
    // DELETE {id:int} → DeletarProdutoAsync → NoContent()
}
```

### ExceptionHandling/GlobalExceptionHandler.cs
```csharp
using {P}.DOMAIN.Exceptions;
using Microsoft.AspNetCore.Diagnostics;
using Microsoft.AspNetCore.Mvc;

namespace {P}.API.ExceptionHandling;

/// <summary>
/// Traduz as exceções de domínio em respostas ProblemDetails (RFC 7807),
/// dispensando os try/catch repetidos nos controllers.
/// </summary>
public class GlobalExceptionHandler : IExceptionHandler
{
    private readonly IProblemDetailsService _problemDetailsService;
    private readonly ILogger<GlobalExceptionHandler> _logger;

    public GlobalExceptionHandler(
        IProblemDetailsService problemDetailsService,
        ILogger<GlobalExceptionHandler> logger)
    {
        _problemDetailsService = problemDetailsService;
        _logger = logger;
    }

    public async ValueTask<bool> TryHandleAsync(
        HttpContext httpContext,
        Exception exception,
        CancellationToken cancellationToken)
    {
        var (status, titulo, detalhe) = Mapear(exception);

        if (status == StatusCodes.Status500InternalServerError)
        {
            // Erros inesperados são registrados por inteiro, mas o detalhe
            // técnico nunca vai para o cliente.
            _logger.LogError(exception, "Erro não tratado ao processar {Metodo} {Caminho}",
                httpContext.Request.Method, httpContext.Request.Path);
        }
        else
        {
            _logger.LogInformation("Requisição rejeitada ({Status}): {Mensagem}", status, exception.Message);
        }

        httpContext.Response.StatusCode = status;

        return await _problemDetailsService.TryWriteAsync(new ProblemDetailsContext
        {
            HttpContext = httpContext,
            Exception = exception,
            ProblemDetails = new ProblemDetails
            {
                Status = status,
                Title = titulo,
                Detail = detalhe,
                Instance = $"{httpContext.Request.Method} {httpContext.Request.Path}"
            }
        });
    }

    private static (int Status, string Titulo, string Detalhe) Mapear(Exception exception) => exception switch
    {
        NaoEncontradoException ex =>
            (StatusCodes.Status404NotFound, "Recurso não encontrado", ex.Message),

        CredenciaisInvalidasException ex =>
            (StatusCodes.Status401Unauthorized, "Não autorizado", ex.Message),

        RegraDeNegocioException ex =>
            (StatusCodes.Status400BadRequest, "Requisição inválida", ex.Message),

        _ => (StatusCodes.Status500InternalServerError, "Erro interno",
            "Ocorreu um erro inesperado ao processar a requisição.")
    };
}
```

### Extensions/AuthenticationExtensions.cs
```csharp
public static class AuthenticationExtensions
{
    public static IServiceCollection AddJwtAuthentication(this IServiceCollection services)
    {
        services
            .AddAuthentication(options =>
            {
                options.DefaultAuthenticateScheme = JwtBearerDefaults.AuthenticationScheme;
                options.DefaultChallengeScheme = JwtBearerDefaults.AuthenticationScheme;
            })
            .AddJwtBearer();

        // A chave vem do JwtOptions já validado — sem fallback embutido no código.
        services
            .AddOptions<JwtBearerOptions>(JwtBearerDefaults.AuthenticationScheme)
            .Configure<IOptions<JwtOptions>>((bearer, jwt) =>
            {
                var options = jwt.Value;

                bearer.TokenValidationParameters = new TokenValidationParameters
                {
                    ValidateIssuer = true,
                    ValidateAudience = true,
                    ValidateLifetime = true,
                    ValidateIssuerSigningKey = true,
                    ValidIssuer = options.Issuer,
                    ValidAudience = options.Audience,
                    IssuerSigningKey = new SymmetricSecurityKey(Encoding.UTF8.GetBytes(options.Key)),
                    ClockSkew = TimeSpan.FromSeconds(30)
                };
            });

        services.AddAuthorization();

        return services;
    }
}
```

### Extensions/SwaggerExtensions.cs (Swashbuckle 10 / OpenAPI.NET v2)
```csharp
using System.Reflection;
using Microsoft.OpenApi;

public static class SwaggerExtensions
{
    private const string EsquemaBearer = "Bearer";

    public static IServiceCollection AddSwaggerDocs(this IServiceCollection services)
    {
        services.AddEndpointsApiExplorer();

        services.AddSwaggerGen(options =>
        {
            options.SwaggerDoc("v1", new OpenApiInfo { Title = "{Nome} API", Version = "v1" });

            var xmlFile = $"{Assembly.GetExecutingAssembly().GetName().Name}.xml";
            var xmlPath = Path.Combine(AppContext.BaseDirectory, xmlFile);

            if (File.Exists(xmlPath))
                options.IncludeXmlComments(xmlPath);

            options.AddSecurityDefinition(EsquemaBearer, new OpenApiSecurityScheme
            {
                Name = "Authorization",
                Type = SecuritySchemeType.Http,
                Scheme = "bearer",
                BearerFormat = "JWT",
                In = ParameterLocation.Header,
                Description = "Digite apenas o token JWT (sem a palavra Bearer)"
            });

            // OpenAPI.NET v2+ referencia o esquema por um tipo dedicado, em vez
            // do par OpenApiReference/ReferenceType usado nas versões anteriores.
            options.AddSecurityRequirement(documento => new OpenApiSecurityRequirement
            {
                { new OpenApiSecuritySchemeReference(EsquemaBearer, documento), [] }
            });
        });

        return services;
    }
}
```

### appsettings.json (segredos sempre vazios)
```json
{
  "ConnectionStrings": {
    "DefaultConnection": ""
  },
  "Jwt": {
    "Key": "",
    "Issuer": "{P}.API",
    "Audience": "{P}.CLIENTS",
    "ExpireMinutes": 60
  },
  "Logging": {
    "LogLevel": {
      "Default": "Information",
      "Microsoft.AspNetCore": "Warning"
    }
  },
  "AllowedHosts": "*"
}
```

---

## 6. TESTS

### Factories/{E}Builder.cs
```csharp
namespace {P}.TESTS.Factories;

/// <summary>
/// Dados de teste com valores padrão válidos, para que cada teste declare
/// apenas o que é relevante para ele.
/// </summary>
public static class ProdutoBuilder
{
    public static Produto Entidade(int id = 0, string nome = "Camiseta", decimal preco = 80m) =>
        new() { Id = id, Nome = nome, Preco = preco };

    public static CriarProdutoDTO Criar(string nome = "Camiseta", decimal preco = 80m) =>
        new() { Nome = nome, Preco = preco };

    public static EditarProdutoDTO Editar(string nome = "Camiseta Editada", decimal preco = 50m) =>
        new() { Nome = nome, Preco = preco };
}
```

### Factories/TestAppDbContextFactory.cs
```csharp
public static class TestAppDbContextFactory
{
    /// <summary>
    /// Contexto isolado por teste. O provider InMemory não traduz SQL real —
    /// serve para exercitar o mapeamento e o change tracker, não para validar
    /// as consultas específicas do Postgres.
    /// </summary>
    public static AppDBContext Create()
    {
        var options = new DbContextOptionsBuilder<AppDBContext>()
            .UseInMemoryDatabase(Guid.NewGuid().ToString())
            .EnableSensitiveDataLogging()
            .Options;

        return new AppDBContext(options);
    }
}
```
Obs.: `EF.Functions.ILike` não roda no InMemory — teste o filtro por nome no service (mock)
ou com Testcontainers/Postgres real.

### ServicesTests/{E}ServiceTests.cs
```csharp
public class ProdutoServiceTests
{
    private readonly Mock<IProdutoRepository> _produtoRepository = new(MockBehavior.Strict);
    private readonly Mock<IUnitOfWork> _unitOfWork = new(MockBehavior.Strict);
    private readonly IProdutoService _produtoService;

    public ProdutoServiceTests()
    {
        _unitOfWork
            .Setup(u => u.CommitAsync(It.IsAny<CancellationToken>()))
            .Returns(Task.CompletedTask);

        _produtoService = new ProdutoService(_produtoRepository.Object, _unitOfWork.Object);
    }

    [Fact]
    public async Task Should_Throw_Editar_Produto_Nao_Encontrado()
    {
        //Arrange
        _produtoRepository
            .Setup(r => r.ObterPorIdAsync(99, true, It.IsAny<CancellationToken>()))
            .ReturnsAsync((Produto?)null);

        //Act
        var acao = () => _produtoService.EditarProdutoAsync(99, ProdutoBuilder.Editar());

        //Assert
        await acao.ShouldThrowAsync<NaoEncontradoException>();
        _unitOfWork.Verify(u => u.CommitAsync(It.IsAny<CancellationToken>()), Times.Never);
    }
}
```

### RepositoriesTests/{E}RepositoryTests.cs
```csharp
public class ProdutoRepositoryTests : IDisposable
{
    private readonly AppDBContext _dbContext;
    private readonly IProdutoRepository _produtoRepository;
    private readonly IUnitOfWork _unitOfWork;

    public ProdutoRepositoryTests()
    {
        _dbContext = TestAppDbContextFactory.Create();
        _produtoRepository = new ProdutoRepository(_dbContext);
        _unitOfWork = new UnitOfWork(_dbContext, TimeProvider.System);
    }

    public void Dispose() => _dbContext.Dispose();

    private async Task<int> SemearAsync(string nome = "Camiseta")
    {
        var produto = ProdutoBuilder.Entidade(nome: nome);

        await _produtoRepository.AdicionarAsync(produto);
        await _unitOfWork.CommitAsync();

        return produto.Id;
    }

    [Fact]
    public async Task Should_Retornar_Entidade_Sem_Rastreio_Por_Padrao()
    {
        //Arrange
        var id = await SemearAsync();
        _dbContext.ChangeTracker.Clear();

        //Act
        await _produtoRepository.ObterPorIdAsync(id);

        //Assert: consulta de leitura não deve sujar o change tracker
        _dbContext.ChangeTracker.Entries().ShouldBeEmpty();
    }
}
```

### ValidationTests/{E}DtoValidationTests.cs
```csharp
public class ProdutoDtoValidationTests
{
    private static IReadOnlyList<ValidationResult> Validar(object dto)
    {
        var resultados = new List<ValidationResult>();

        Validator.TryValidateObject(dto, new ValidationContext(dto), resultados, validateAllProperties: true);

        return resultados;
    }

    [Theory]
    [InlineData(-80)]
    [InlineData(0)]
    public void Should_Rejeitar_Preco_Invalido(decimal preco)
    {
        Validar(ProdutoBuilder.Criar(preco: preco)).ShouldNotBeEmpty();
    }
}
```
Para garantir que um DTO **não** expõe um campo (ex.: `Role` no registro):
`typeof(RegistrarUsuarioDTO).GetProperty("Role").ShouldBeNull();`

---

## 7. Comandos

```bash
# Solution
dotnet new sln -n {P}
dotnet new classlib -n {P}.DOMAIN && dotnet new classlib -n {P}.APPLICATION && dotnet new classlib -n {P}.DATA
dotnet new webapi --use-controllers -n {P}.API
dotnet new xunit -n {P}.TESTS
dotnet sln add */*.csproj

# Banco dedicado (escolha porta livre ≠ 5432; confira com docker ps)
docker run -d --name {projeto}-postgres \
  -e POSTGRES_USER={usuario} -e POSTGRES_PASSWORD=SUA_SENHA -e POSTGRES_DB={banco} \
  -p {PORTA}:5432 --restart unless-stopped postgres:17

# Segredos
dotnet user-secrets init --project {P}.API
dotnet user-secrets set "ConnectionStrings:DefaultConnection" \
  "Host=localhost;Port={PORTA};Database={banco};Username={usuario};Password=SUA_SENHA" --project {P}.API
dotnet user-secrets set "Jwt:Key" "$(openssl rand -base64 48)" --project {P}.API

# Migrations
dotnet ef migrations add NomeDescritivo --project {P}.DATA --startup-project {P}.API
dotnet ef database update --project {P}.DATA --startup-project {P}.API

# Verificação
dotnet build && dotnet test
```
