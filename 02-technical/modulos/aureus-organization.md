# 🏢 AUREUS Organization - Módulo de Estrutura Organizacional

## 📋 **Visão Geral**

O **AUREUS Organization** é um módulo completo de gestão organizacional que implementa estrutura hierárquica, controle de alçada e integrações com serviços externos para o AUREUS Core Banking.

**Status**: ✅ **Implementado**  
**Versão**: 1.0.0  
**Porta**: 8087  
**Última Atualização**: Janeiro 2025  

---

## 🏗️ **Arquitetura do Módulo**

### **Estrutura de Diretórios**
```
aureus-organization/
├── src/main/java/com/aureus/platform/organization/
│   ├── controller/          # APIs REST
│   ├── service/            # Lógica de negócio
│   ├── repository/         # Acesso a dados
│   ├── entity/             # Entidades JPA
│   ├── dto/                # Objetos de transferência
│   ├── config/             # Configurações
│   └── integration/        # Integrações externas
│       ├── rh/             # Sistemas de RH
│       ├── governo/        # eSocial, Receita Federal
│       ├── social/         # LinkedIn
│       ├── analytics/      # BI e Analytics
│       └── webhook/        # Sistema de webhooks
└── src/main/resources/
    └── application.yml     # Configurações
```

---

## 🎯 **Funcionalidades Implementadas**

### **1. Estrutura Organizacional**
- ✅ **Empresas** - Gestão de instituições
- ✅ **Departamentos** - Hierarquia organizacional
- ✅ **Cargos** - Posições e responsabilidades
- ✅ **Funcionários** - Gestão de pessoas
- ✅ **Hierarquia** - Relacionamentos entre entidades

### **2. Controle de Alçada**
- ✅ **Workflows** - Processos de aprovação personalizáveis
- ✅ **Etapas de Workflow** - Níveis de aprovação
- ✅ **Solicitações** - Sistema de solicitações
- ✅ **Aprovações** - Controle de aprovações
- ✅ **Delegação de Poderes** - Transferência temporária

### **3. Integrações Externas**
- ✅ **Sistemas de RH** - Sincronização de funcionários
- ✅ **eSocial** - Envio de eventos S-1000 e S-1005
- ✅ **Receita Federal** - Validação de CNPJ/CPF
- ✅ **LinkedIn** - Perfis e competências
- ✅ **Business Intelligence** - Analytics e métricas
- ✅ **Webhooks** - Notificações em tempo real

---

## 📊 **Entidades Principais**

### **Empresa**
```java
@Entity
public class Empresa {
    private Long id;
    private String codigoEmpresa;
    private String nomeEmpresa;
    private String cnpj;
    private StatusEmpresa status;
    private String dadosEmpresa; // JSONB
    private String configuracoesEmpresa; // JSONB
}
```

### **Funcionário**
```java
@Entity
public class Funcionario {
    private Long id;
    private String matricula;
    private String nomeCompleto;
    private String cpf;
    private String email;
    private Empresa empresa;
    private Departamento departamento;
    private Cargo cargo;
    private Funcionario gestor;
    private StatusFuncionario status;
    private BigDecimal salarioAtual;
}
```

### **Workflow**
```java
@Entity
public class Workflow {
    private Long id;
    private String codigoWorkflow;
    private String nomeWorkflow;
    private TipoWorkflow tipoWorkflow;
    private StatusWorkflow status;
    private Empresa empresa;
    private String regrasWorkflow; // JSONB
    private Integer timeoutHoras;
}
```

### **Solicitação de Aprovação**
```java
@Entity
public class SolicitacaoAprovacao {
    private Long id;
    private String codigoSolicitacao;
    private Workflow workflow;
    private Funcionario solicitante;
    private TipoSolicitacao tipoSolicitacao;
    private StatusSolicitacao status;
    private BigDecimal valorSolicitado;
    private String dadosSolicitacao; // JSONB
}
```

---

## 🔌 **APIs Disponíveis**

### **Gestão de Empresas**
```
GET    /api/empresas                    - Listar empresas
GET    /api/empresas/{id}               - Buscar empresa por ID
GET    /api/empresas/codigo/{codigo}    - Buscar empresa por código
POST   /api/empresas                    - Criar empresa
PUT    /api/empresas/{id}               - Atualizar empresa
DELETE /api/empresas/{id}               - Excluir empresa
GET    /api/empresas/ativas             - Listar empresas ativas
```

### **Gestão de Funcionários**
```
GET    /api/funcionarios                    - Listar funcionários
GET    /api/funcionarios/{id}               - Buscar funcionário por ID
GET    /api/funcionarios/matricula/{mat}    - Buscar por matrícula
GET    /api/funcionarios/empresa/{id}       - Por empresa
GET    /api/funcionarios/departamento/{id}  - Por departamento
GET    /api/funcionarios/cargo/{id}         - Por cargo
GET    /api/funcionarios/gestor/{id}        - Subordinados
POST   /api/funcionarios                    - Criar funcionário
PUT    /api/funcionarios/{id}               - Atualizar funcionário
DELETE /api/funcionarios/{id}               - Excluir funcionário
```

### **Controle de Alçada**
```
POST   /api/controle-alcada/solicitar       - Solicitar aprovação
GET    /api/controle-alcada/solicitacoes    - Listar solicitações
GET    /api/controle-alcada/solicitacoes/{id} - Buscar solicitação
POST   /api/controle-alcada/aprovar/{id}    - Aprovar solicitação
POST   /api/controle-alcada/rejeitar/{id}   - Rejeitar solicitação
POST   /api/controle-alcada/cancelar/{id}   - Cancelar solicitação
GET    /api/controle-alcada/dashboard/{id}  - Dashboard de aprovações
```

### **Integrações Externas**
```
POST   /api/integrations/rh/sincronizar              - Sincronizar RH
GET    /api/integrations/receita/validar-cnpj/{cnpj} - Validar CNPJ
GET    /api/integrations/receita/validar-cpf/{cpf}   - Validar CPF
POST   /api/integrations/esocial/enviar-evento-s1000 - Evento eSocial
GET    /api/integrations/linkedin/perfil/{email}     - Perfil LinkedIn
GET    /api/integrations/bi/dashboard-rh/{empresaId} - Dashboard BI
```

---

## 🔧 **Configuração**

### **Variáveis de Ambiente**
```bash
# RH Integration
RH_API_KEY=your_rh_api_key

# eSocial Integration
ESOCIAL_CERT_PATH=/path/to/certificate.p12
ESOCIAL_CERT_PASSWORD=your_certificate_password

# LinkedIn Integration
LINKEDIN_CLIENT_ID=your_linkedin_client_id
LINKEDIN_CLIENT_SECRET=your_linkedin_client_secret
LINKEDIN_ACCESS_TOKEN=your_linkedin_access_token

# BI/Analytics Integration
BI_API_KEY=your_bi_api_key

# Webhooks
WEBHOOK_API_KEY=your_webhook_api_key
```

### **Configuração do Banco**
```yaml
spring:
  datasource:
    url: jdbc:postgresql://localhost:5432/aureus_db
    username: aureus_user
    password: aureus_pass
  jpa:
    hibernate:
      ddl-auto: update
```

---

## 📈 **Métricas e Monitoramento**

### **Health Checks**
```
GET /aureus-organization/api/health      - Status do serviço
GET /aureus-organization/api/health/ready - Readiness check
```

### **Métricas Disponíveis**
- Número de funcionários por empresa
- Solicitações de aprovação pendentes
- Taxa de aprovação por workflow
- Tempo médio de aprovação
- Integrações ativas

---

## 🚀 **Deploy e Execução**

### **Docker Compose**
```yaml
aureus-organization:
  build: ./aureus-organization
  ports:
    - "8087:8087"
  environment:
    SPRING_DATASOURCE_URL: jdbc:postgresql://postgres:5432/aureus_db
    RH_API_KEY: ${RH_API_KEY}
  depends_on:
    - postgres
```

### **Execução Local**
```bash
cd aureus-organization
mvn spring-boot:run
```

---

## 🔗 **Integrações Implementadas**

### **1. Sistemas de RH**
- Sincronização automática de funcionários
- Dados de folha de pagamento
- Estrutura organizacional
- Notificações de mudanças

### **2. Serviços do Governo**
- **eSocial**: Eventos S-1000 e S-1005
- **Receita Federal**: Validação CNPJ/CPF
- **CBOs**: Classificação de ocupações

### **3. Redes Sociais**
- **LinkedIn**: Perfis e competências
- Experiências profissionais
- Busca de candidatos

### **4. Business Intelligence**
- Métricas organizacionais
- Dashboard RH
- Relatórios de turnover
- Análise de competências
- Previsão de demissões

### **5. Sistema de Webhooks**
- Notificações em tempo real
- Sincronização automática
- Eventos de funcionários e empresas

---

## 📊 **Banco de Dados**

### **Tabelas Principais**
- `empresas` - Dados das empresas
- `departamentos` - Estrutura departamental
- `cargos` - Posições e responsabilidades
- `funcionarios` - Dados dos funcionários
- `workflows` - Processos de aprovação
- `etapas_workflow` - Etapas dos workflows
- `solicitacoes_aprovacao` - Solicitações
- `aprovacoes` - Aprovações individuais
- `delegacoes_poder` - Delegações temporárias

### **Índices Otimizados**
- Índices por empresa, departamento, cargo
- Índices por funcionário e gestor
- Índices por workflow e status
- Índices por data e prioridade

---

## 🎯 **Próximos Passos**

### **Melhorias Planejadas**
1. **Interface Web** - Dashboard visual para configuração
2. **Notificações Push** - Alertas em tempo real
3. **Relatórios Avançados** - Dashboards executivos
4. **Mobile App** - Aplicativo para aprovações
5. **IA/ML** - Análise preditiva de aprovações

### **Integrações Futuras**
1. **Slack/Teams** - Notificações em chat
2. **Email** - Notificações por email
3. **SMS** - Alertas por SMS
4. **WhatsApp** - Notificações via WhatsApp

---

## 📚 **Documentação Técnica**

### **Swagger/OpenAPI**
- URL: `http://localhost:8087/aureus-organization/swagger-ui.html`
- Documentação interativa das APIs
- Testes de integração

### **Logs e Debugging**
```yaml
logging:
  level:
    com.aureus.platform.organization: DEBUG
    org.springframework.security: DEBUG
    org.hibernate.SQL: DEBUG
```

---

## 🏆 **Benefícios Implementados**

### **Para a Organização**
- ✅ **Estrutura Hierárquica** clara e organizada
- ✅ **Controle de Alçada** personalizável
- ✅ **Integrações** com sistemas externos
- ✅ **Compliance** regulatório
- ✅ **Auditoria** completa

### **Para os Funcionários**
- ✅ **Aprovações** digitais e rápidas
- ✅ **Transparência** no processo
- ✅ **Notificações** em tempo real
- ✅ **Histórico** completo de solicitações

### **Para TI**
- ✅ **APIs REST** padronizadas
- ✅ **Webhooks** para integração
- ✅ **Monitoramento** completo
- ✅ **Escalabilidade** horizontal

---

**Desenvolvido por**: Equipe AUREUS  
**Última Atualização**: Janeiro 2025  
**Status**: ✅ Produção  
**Versão**: 1.0.0
