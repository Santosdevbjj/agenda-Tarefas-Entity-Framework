# AgendaTarefas — Sistema de Agendamento com Entity Framework Core

![GFTNet001](https://github.com/user-attachments/assets/0c07fdcb-9e4c-457c-ab73-0ca31b495868)

> **Bootcamp GFT Start #7 .NET** | .NET 8 · Entity Framework Core · SQLite · xUnit · Azure Pipelines

[![Portfólio Sérgio Santos](https://img.shields.io/badge/Portfólio-Sérgio_Santos-111827?style=for-the-badge&logo=githubpages&logoColor=00eaff)](https://portfoliosantossergio.vercel.app)
[![LinkedIn](https://img.shields.io/badge/LinkedIn-Sérgio_Santos-0A66C2?style=for-the-badge&logo=linkedin&logoColor=white)](https://linkedin.com/in/santossergioluiz)

---

## 1. Problema de Negócio

Equipes que gerenciam tarefas recorrentes sem um sistema centralizado enfrentam dois problemas críticos: **perda de rastreabilidade** (ninguém sabe o status real de cada item) e **ausência de auditoria** (não há histórico confiável para decisões gerenciais).

Planilhas e e-mails resolvem o curto prazo, mas criam dívida operacional: dados inconsistentes, duplicidade de esforço e impossibilidade de escalar o processo conforme a equipe cresce.

O desafio central deste projeto é demonstrar como uma API REST bem estruturada, com persistência gerenciada por ORM e pipeline de entrega automatizado, elimina esses problemas e cria uma base auditável e evolutiva.

---

## 2. Contexto

O projeto simula o backend de uma aplicação de gerenciamento de tarefas corporativas, com as seguintes características operacionais:

- Tarefas possuem **título**, **descrição**, **data de vencimento** e **status** (`Pendente`, `EmAndamento`, `Concluída`)
- O status é armazenado como **string legível** no banco (não como inteiro), facilitando auditorias diretas no BD
- A arquitetura separa responsabilidades em **Controller → Service → Repository → DbContext**, padrão comum em sistemas críticos de médio/grande porte
- O pipeline CI/CD cobre dois ambientes (**Dev** e **Prod**) com aprovação manual obrigatória antes do deploy em produção

---

## 3. Premissas da Análise

- O campo `Status` é a fonte oficial de verdade sobre o estado de cada tarefa
- `Data` representa o prazo de execução da tarefa (armazenado como `TEXT` no SQLite, compatível com ISO 8601)
- SQLite foi escolhido como banco de dados **intencionalmente para desenvolvimento e testes** — a arquitetura permite troca para SQL Server/Azure SQL sem alterações no código de negócio
- Os testes de integração rodam contra um servidor in-memory (sem dependência de banco externo), garantindo isolamento e reprodutibilidade
- O pipeline Azure Pipelines reflete um fluxo real de entrega em times que trabalham com branches `dev` e `main`

---

## 4. Estratégia da Solução

### Stack e justificativa técnica

| Tecnologia | Por que foi escolhida | Alternativa considerada |
|---|---|---|
| **.NET 8 + Minimal Hosting** | Maturidade, performance e suporte LTS | .NET 6 (fim de suporte próximo) |
| **Entity Framework Core 8** | ORM com migrations versionadas e provider intercambiável | Dapper (sem migrations nativas) |
| **SQLite** | Zero-config para dev/testes; mesma interface EF para trocar depois | SQL Server (overhead desnecessário em bootcamp) |
| **xUnit + Moq** | Padrão da comunidade .NET; integração nativa com WebApplicationFactory | NUnit (menos adotado no ecossistema .NET moderno) |
| **Azure Pipelines YAML** | Templates reutilizáveis com stages parametrizados; aprovação por Environment | GitHub Actions (avaliado, mas o contexto GFT/Azure é natural aqui) |

### Arquitetura da solução

```
Request HTTP
     │
     ▼
TarefaController         ← valida entrada, retorna HTTP status corretos
     │
     ▼
TarefaService            ← regras de negócio isoladas (testáveis sem DB)
     │
     ▼
ITarefaRepository        ← abstração que permite mock em testes unitários
     │
     ▼
AppDbContext (EF Core)   ← mapeamento ORM + migrations versionadas
     │
     ▼
agendaTarefas.db (SQLite)
```

### Pipeline CI/CD

```
push dev / main
     │
     ▼
Stage Build: restore → build → unit tests → integration tests → publish artifact
     │
     ▼
Stage Dev: deploy automático → App Service Dev
     │
     ▼
Stage Prod: aguarda aprovação manual → deploy → App Service Prod
```

A decisão de usar **templates YAML reutilizáveis** (`stage-template.yml` + `variables-template.yml`) reflete a prática de times que operam múltiplos ambientes sem duplicar configuração — cada stage é parametrizado por `variableGroup`, `azureSubscription` e `azureWebAppName`.

---

## 5. Insights Técnicos

**Separação de concerns como proteção ao teste:**  
Ao isolar a lógica de negócio em `TarefaService`, foi possível escrever testes unitários com `Mock<ITarefaRepository>` sem nenhuma dependência de banco. O teste `CreateAsync_ShouldReturnDto_WithGeneratedId` valida o comportamento do service em milissegundos, independentemente de infraestrutura.

**`EnumToStringConverter` como decisão de auditoria:**  
Salvar `StatusTarefa` como string (`"Pendente"`, `"EmAndamento"`, `"Concluida"`) em vez de inteiro foi uma escolha deliberada. Em sistemas bancários e corporativos, queries diretas no banco por times de suporte e auditoria são comuns — legibilidade nativa elimina a necessidade de dicionários externos.

**`public partial class Program` como contrato de teste:**  
A instrução no final do `Program.cs` não é apenas um detalhe técnico — ela é o contrato que permite ao `WebApplicationFactory<Program>` construir o servidor de teste com a configuração real da aplicação, garantindo que os testes de integração validem o comportamento end-to-end real.

**Separação de template YAML como governança de pipeline:**  
O `stage-template.yml` parametrizável permite que novos ambientes (ex.: `Staging`, `QA`) sejam adicionados ao pipeline com 3 linhas de YAML, sem alterar a lógica central. Isso reflete maturidade em DevOps: configuração como código, não como procedimento manual.

---

## 6. Resultados

Com a implementação deste projeto:

- **Rastreabilidade garantida:** cada tarefa tem ciclo de vida auditável com status legível direto no banco
- **Confiabilidade verificável:** testes unitários (camada de serviço) e de integração (fluxo HTTP end-to-end) cobrem os casos críticos de criação e consulta
- **Entrega controlada:** pipeline multistage com aprovação manual em Prod impede deploys acidentais — o mesmo padrão adotado em times que trabalham com SLAs de disponibilidade
- **Arquitetura evolutiva:** a troca de SQLite por SQL Server/Azure SQL exige apenas mudança de provider no `csproj` e `Program.cs`, sem tocar nas regras de negócio

---

## 7. Como Executar o Projeto

### Pré-requisitos

```bash
# Verificar versão do .NET (requer >= 8.0)
dotnet --version

# Instalar EF CLI (caso não tenha)
dotnet tool install --global dotnet-ef
```

### Clonar e rodar

```bash
git clone https://github.com/Santosdevbjj/agenda-Tarefas-Entity-Framework
cd agenda-Tarefas-Entity-Framework

# Restaurar dependências e compilar
dotnet restore
dotnet build --configuration Release

# Aplicar migrations e criar o banco SQLite
dotnet ef database update

# Rodar a API
dotnet run --project AgendaTarefas.csproj
```

Acesse a documentação Swagger em: **`https://localhost:5001/swagger/index.html`**

### Rodar os testes

```bash
# Testes unitários
dotnet test ./Tests/AgendaTarefas.Tests.Unit/AgendaTarefas.Tests.Unit.csproj --configuration Release

# Testes de integração
dotnet test ./Tests/AgendaTarefas.Tests.Integration/AgendaTarefas.Tests.Integration.csproj --configuration Release
```

> **Atenção:** Para os testes de integração funcionarem, certifique-se de que o final do `Program.cs` contém:
> ```csharp
> public partial class Program { }
> ```

---

## 8. Configurar CI/CD (Azure Pipelines)

1. Faça commit do `azure-pipelines.yml` na raiz do repositório
2. Em **Project Settings → Service connections**: configure a conexão à sua assinatura Azure
3. Em **Pipelines → Library**: crie os Variable Groups `DevVariables` e `ProdVariables` com as variáveis `DevWebApp`, `DevSubscription`, `ProdWebApp`, `ProdSubscription`
4. Em **Pipelines → Environments**: crie os environments `Dev` e `Prod`; no Prod adicione **Approvals & checks** com aprovadores obrigatórios
5. Faça um push em `dev` ou `main` e observe o pipeline executar

---

## 9. Aprendizados

**O que foi mais desafiador:**  
Configurar o `WebApplicationFactory<Program>` para testes de integração exigiu entender como o .NET 8 minimal hosting expõe o entrypoint. A instrução `public partial class Program { }` é simples, mas sua ausência gera erros de compilação silenciosos nos testes — aprendi a identificar esse padrão como checklist obrigatório em qualquer projeto ASP.NET Core com testes de integração.

**Principal aprendizado de design:**  
Separar regras de negócio em `TarefaService` antes de escrever os testes, não depois. Quando o service está acoplado ao controller, o custo de testar explode. A sequência correta é: definir a interface do repositório → implementar o service → escrever os testes unitários → só então implementar o controller.

**O que faria diferente:**  
Adicionaria um banco SQLite in-memory dedicado para os testes de integração desde o início, evitando que os testes dependam do arquivo `.db` local e garantindo paralelismo seguro entre test runs.

---

## 10. Próximos Passos

- [ ] Implementar paginação no endpoint `GET /Tarefa` para suportar grandes volumes
- [ ] Adicionar autenticação JWT — o próximo passo natural para um sistema corporativo real
- [ ] Substituir SQLite por Azure SQL nos ambientes Dev e Prod
- [ ] Configurar cobertura de código no pipeline (publicar relatório de coverage como artefato)
- [ ] Implementar cache com `IMemoryCache` para leituras frequentes de tarefas por status

---

## Stack

![.NET](https://img.shields.io/badge/.NET_8-512BD4?style=for-the-badge&logo=dotnet&logoColor=white)
![C#](https://img.shields.io/badge/C%23-239120?style=for-the-badge&logo=csharp&logoColor=white)
![Entity Framework](https://img.shields.io/badge/Entity_Framework_Core-512BD4?style=for-the-badge&logo=dotnet&logoColor=white)
![SQLite](https://img.shields.io/badge/SQLite-003B57?style=for-the-badge&logo=sqlite&logoColor=white)
![xUnit](https://img.shields.io/badge/xUnit-5C2D91?style=for-the-badge&logo=dotnet&logoColor=white)
![Azure Pipelines](https://img.shields.io/badge/Azure_Pipelines-0078D7?style=for-the-badge&logo=azuredevops&logoColor=white)

---

## Contato

[![Portfólio Sérgio Santos](https://img.shields.io/badge/Portfólio-Sérgio_Santos-111827?style=for-the-badge&logo=githubpages&logoColor=00eaff)](https://portfoliosantossergio.vercel.app)
[![LinkedIn](https://img.shields.io/badge/LinkedIn-Sérgio_Santos-0A66C2?style=for-the-badge&logo=linkedin&logoColor=white)](https://linkedin.com/in/santossergioluiz)
