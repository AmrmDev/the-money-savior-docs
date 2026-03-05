# 🗺️ Money Savior — Technical Roadmap

Documento técnico de planejamento de features futuras.  
Cada feature descreve motivação, impacto na arquitetura, mudanças necessárias e pontos de atenção.

---

## v2 — Sistema de Categorias Inteligentes

### Motivação
Atualmente a categoria é salva como texto livre sem normalização semântica.  
O objetivo é mapear inputs variados para categorias padronizadas e permitir que o usuário cadastre aliases customizados.

### Comportamento esperado
- `/gastei 25.00 uber pix` → categoria salva como `Transporte`
- `/gastei 15.00 mercadinho débito` → categoria salva como `Alimentação`
- Usuário pode cadastrar novo alias: `/categoria add academia Saúde`

### Categorias padrão

| Categoria | Aliases padrão |
|---|---|
| Alimentação | mercado, mercadinho, ifood, restaurante, lanche, almoço, janta, padaria, açougue |
| Transporte | uber, 99, ônibus, metrô, combustível, gasolina, estacionamento |
| Saúde | farmácia, remédio, médico, consulta, academia, plano de saúde |
| Moradia | aluguel, condomínio, luz, água, internet, gás |
| Lazer | cinema, bar, show, viagem, streaming, netflix, spotify |
| Educação | curso, livro, faculdade, escola |
| Outros | fallback para inputs não reconhecidos |

### Arquitetura

**Novo model:**
```go
// internal/models/category.go
type Category struct {
    UserID    int64     `dynamodbav:"user_id"`
    Alias     string    `dynamodbav:"alias"`       // PK secundária
    Category  string    `dynamodbav:"category"`
    CreatedAt time.Time `dynamodbav:"created_at"`
}
```

**Nova tabela DynamoDB:** `money-savior-categories`
- Partition key: `user_id` (N)
- Sort key: `alias` (S)

**Novo repository:**
```go
// internal/repository/category_repository.go
type CategoryRepository interface {
    FindByAlias(ctx context.Context, userID int64, alias string) (*models.Category, error)
    Save(ctx context.Context, category *models.Category) error
    ListByUser(ctx context.Context, userID int64) ([]models.Category, error)
    DeleteByAlias(ctx context.Context, userID int64, alias string) error
}
```

**Lógica de normalização:**
```go
// internal/utils/category.go
var defaultAliases = map[string]string{
    "uber":        "Transporte",
    "99":          "Transporte",
    "mercado":     "Alimentação",
    "ifood":       "Alimentação",
    "farmácia":    "Saúde",
    "academia":    "Saúde",
    // ...
}

func NormalizeCategory(ctx context.Context, repo repository.CategoryRepository, userID int64, input string) string {
    normalized := strings.ToLower(strings.TrimSpace(input))

    // 1. Busca alias customizado do usuário no DynamoDB
    if cat, err := repo.FindByAlias(ctx, userID, normalized); err == nil {
        return cat.Category
    }

    // 2. Fallback para aliases padrão em memória
    if category, ok := defaultAliases[normalized]; ok {
        return category
    }

    // 3. Retorna com título formatado como fallback
    return utils.FormatTitle(input)
}
```

**Novos comandos:**
```
/categoria list              → lista categorias customizadas do usuário
/categoria add <alias> <cat> → adiciona alias customizado
/categoria del <alias>       → remove alias customizado
```

**Novo handler:**
```go
// internal/handlers/category_handler.go
type CategoryHandler struct {
    categoryService *service.CategoryService
}
```

**Impacto no fluxo do `/gastei`:**
- `ExpenseService.CreateExpense` passa a receber `categoryService` ou chamar `NormalizeCategory` antes de salvar

### Permissões IAM adicionais
- `dynamodb:PutItem` na tabela `money-savior-categories`
- `dynamodb:GetItem` na tabela `money-savior-categories`
- `dynamodb:Query` na tabela `money-savior-categories`
- `dynamodb:DeleteItem` na tabela `money-savior-categories`

### Pontos de atenção
- A normalização deve acontecer no `service`, não no `handler`
- Aliases são case-insensitive — normalizar sempre com `strings.ToLower` antes de salvar e buscar
- Limite razoável de aliases por usuário (ex: 50) para evitar abuso

---

## v3 — Cartão de Crédito e Parcelas

### Motivação
Permitir controle de limite, lançamento de compras parceladas e recebimento de report mensal da fatura por cartão.

### Comportamento esperado
- `/cartao add Nubank 3000` → cadastra cartão com limite de R$3.000
- `/gastei 600 tênis crédito nubank 3x` → registra gasto parcelado em 3x no cartão Nubank
- Report automático todo dia 1º com resumo da fatura e parcelas do mês

### Arquitetura

**Novos models:**
```go
// internal/models/card.go
type Card struct {
    UserID    int64     `dynamodbav:"user_id"`
    CardID    string    `dynamodbav:"card_id"`
    Name      string    `dynamodbav:"name"`       // ex: "Nubank"
    Limit     float64   `dynamodbav:"limit"`
    CreatedAt time.Time `dynamodbav:"created_at"`
}

// internal/models/installment.go
type Installment struct {
    UserID      int64     `dynamodbav:"user_id"`
    ExpenseID   string    `dynamodbav:"expense_id"`
    CardID      string    `dynamodbav:"card_id"`
    Total       float64   `dynamodbav:"total"`
    Installments int      `dynamodbav:"installments"`
    Paid        int       `dynamodbav:"paid"`
    Monthly     float64   `dynamodbav:"monthly"`     // Total / Installments
    NextDue     time.Time `dynamodbav:"next_due"`
}
```

**Novas tabelas DynamoDB:**
- `money-savior-cards` — PK: `user_id` (N), SK: `card_id` (S)
- `money-savior-installments` — PK: `user_id` (N), SK: `expense_id` (S)

**Report mensal automático:**
- Implementar via **AWS EventBridge** (cron `0 9 1 * ? *`) triggerando uma Lambda separada
- Lambda de report busca todos os usuários ativos, calcula fatura de cada cartão e envia mensagem via Telegram/WhatsApp

**Novos comandos:**
```
/cartao list              → lista cartões cadastrados com limite e uso atual
/cartao add <nome> <lim>  → cadastra novo cartão
/cartao del <nome>        → remove cartão
/fatura <nome>            → exibe fatura atual do cartão
```

### Pontos de atenção
- Compra parcelada gera um `Expense` + um `Installment` vinculado
- O campo `method` do `Expense` deve aceitar `crédito <nome_cartão>` para vincular ao cartão
- Calcular uso atual do cartão = soma de parcelas pendentes do mês

---

## v4 — Migração para WhatsApp (WAHA API + IA)

### Motivação
Maior alcance — WhatsApp tem adoção massiva no Brasil comparado ao Telegram.  
Como o WhatsApp não tem o conceito de comandos estruturados, o usuário se comunica em **linguagem natural** e um LLM extrai as informações da mensagem.

### Stack
- **WAHA** (WhatsApp HTTP API) — self-hosted via Docker
- **Groq API** (LLaMA 3) — extração de dados via linguagem natural, free tier
- Hospedagem sugerida para WAHA: EC2 `t3.micro` ou Railway

### Fluxo de linguagem natural

```
Usuário: "gastei 12 reais em café no pix"
              ↓
        Groq API (LLaMA 3)
              ↓
  { intent: "expense", amount: 12.00, category: "café", method: "pix" }
              ↓
        ExpenseService.CreateExpense(...)
              ↓
  Bot responde: "✅ Gasto de R$12,00 em Alimentação registrado!"
```

### Intents suportados

| Mensagem do usuário | Intent detectado |
|---|---|
| "gastei 12 reais em café" | `expense` |
| "quanto gastei esse mês?" | `query_list` |
| "me mostra o gasto AM123456" | `query_detail` |
| "apaga o AM123456" | `delete` |
| "limpa tudo" | `delete_all` |
| "ajuda" / "o que você faz?" | `help` |

### Model de retorno esperado do LLM

```go
// internal/models/parsed_message.go
type ParsedMessage struct {
    Intent    string  `json:"intent"`     // expense, query_list, query_detail, delete, delete_all, help, unknown
    Amount    float64 `json:"amount"`     // preenchido só para intent=expense
    Category  string  `json:"category"`   // preenchido só para intent=expense
    Method    string  `json:"method"`     // preenchido só para intent=expense
    ExpenseID string  `json:"expense_id"` // preenchido só para intent=query_detail e delete
}
```

### Prompt para o LLM

```go
// internal/ai/parser.go
const systemPrompt = `Você é um assistente financeiro. Analise a mensagem do usuário e retorne APENAS um JSON válido, sem texto adicional.

Intents possíveis: expense, query_list, query_detail, delete, delete_all, help, unknown.

Exemplos:
- "gastei 12 reais em café no pix" → {"intent":"expense","amount":12.00,"category":"café","method":"pix","expense_id":""}
- "quanto gastei esse mês" → {"intent":"query_list","amount":0,"category":"","method":"","expense_id":""}
- "apaga o AM123456" → {"intent":"delete","amount":0,"category":"","method":"","expense_id":"AM123456"}

Retorne sempre todos os campos, usando valores vazios quando não aplicável.`
```

**Cliente Groq:**
```go
// internal/ai/groq_client.go
type GroqClient struct {
    apiKey     string
    httpClient *http.Client
}

func (c *GroqClient) ParseMessage(ctx context.Context, text string) (*models.ParsedMessage, error) {
    // POST https://api.groq.com/openai/v1/chat/completions
    // model: "llama3-8b-8192"
    // Faz parse do JSON retornado
}
```

**Novo roteador para WhatsApp:**
```go
// internal/bot/route_waha.go
func RouteWAHA(client *waha.WAHAClient, ai *ai.GroqClient, update waha.WebhookPayload, h *BotHandlers) {
    parsed, err := ai.ParseMessage(ctx, update.Body)
    if err != nil {
        client.SendMessage(update.ChatID, "❌ Não entendi. Tente: 'gastei 25 reais em uber no pix'")
        return
    }

    switch parsed.Intent {
    case "expense":
        h.Expense.HandleNatural(client, update.ChatID, update.UserID, parsed)
    case "query_list":
        h.Query.HandleListNatural(client, update.ChatID, update.UserID)
    case "query_detail":
        h.Query.HandleDetailNatural(client, update.ChatID, update.UserID, parsed.ExpenseID)
    case "delete":
        h.Delete.HandleDeleteNatural(client, update.ChatID, update.UserID, parsed.ExpenseID)
    case "delete_all":
        h.Delete.HandleDeleteAllNatural(client, update.ChatID, update.UserID)
    default:
        client.SendMessage(update.ChatID, "❓ Não entendi. Diga algo como: 'gastei 50 reais no mercado'")
    }
}
```

**Cliente WAHA:**
```go
// internal/waha/client.go
type WAHAClient struct {
    baseURL    string
    session    string
    httpClient *http.Client
}

func (c *WAHAClient) SendMessage(chatID, text string) error {
    // POST /api/sendText
}
```

### Variáveis de ambiente adicionais

| Variável | Descrição |
|---|---|
| `GROQ_API_KEY` | Chave da API Groq |
| `WAHA_BASE_URL` | URL do servidor WAHA (ex: http://localhost:3000) |
| `WAHA_SESSION` | Nome da sessão WAHA (ex: default) |

### Arquitetura geral

```
Telegram:   API Gateway → Lambda → RouteUpdate (comandos) → handlers → service → repository
WhatsApp:   API Gateway → Lambda → RouteWAHA (linguagem natural + Groq) → handlers → service → repository
```

A camada de `service` e `repository` permanece 100% intacta — apenas a camada de entrada muda.

### Pontos de atenção
- WAHA free tem limitações — avaliar plano Plus para uso em produção
- Groq tem rate limit no free tier — implementar retry com backoff exponencial
- Sempre validar o JSON retornado pelo LLM antes de usar — LLMs podem retornar JSON malformado
- `chatID` no WAHA tem formato diferente do Telegram: `5511999999999@c.us`
- Manter Telegram funcionando em paralelo durante a migração
- Para `intent=expense`, validar que `amount > 0` antes de salvar

---

## v5 — Inteligência Financeira

### Features planejadas

**Orçamento por categoria:**
- Novo model `Budget` com `user_id`, `category`, `limit`, `month`
- Comando `/orcamento add Alimentação 500`
- Alert automático quando gasto atingir 80% e 100% do limite

**Metas de economia:**
- Novo model `Goal` com `user_id`, `amount`, `month`, `description`
- Comando `/meta 300` — define meta de economia para o mês
- Calculado como: receita - gastos (requer input de receita)

**Gastos recorrentes:**
- Flag `recurring: true` no model `Expense`
- EventBridge cron mensal para replicar gastos recorrentes automaticamente

**Report mensal:**
- Gerado todo dia 1º via EventBridge
- Contém: total gasto, breakdown por categoria, comparativo com mês anterior, maior gasto do mês

**Exportação PDF/Excel:**
- Lambda separada triggerada por comando `/exportar`
- Gera arquivo via biblioteca Go, faz upload no S3 e envia link temporário (pre-signed URL)

---

## Visão geral das versões

| Versão | Feature | Complexidade | Dependências |
|---|---|---|---|
| v2 | Categorias inteligentes | Média | Nova tabela DynamoDB |
| v3 | Cartão de crédito + parcelas | Alta | 2 novas tabelas + EventBridge |
| v4 | Migração WhatsApp (WAHA + Groq IA) | Alta | WAHA self-hosted + Groq API |
| v5 | Inteligência financeira | Alta | EventBridge + S3 |
