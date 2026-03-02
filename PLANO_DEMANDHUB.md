# DemandHub — Plano de Projeto

> Plataforma de Planejamento de Demanda (Forecast) — Ecosistema StockHub

---

## 1. Visão Geral

O **DemandHub** é uma plataforma de planejamento de demanda que permite a criação, comparação e gestão de múltiplos planos de forecast em diferentes níveis de uma hierarquia comercial. A ferramenta será parte do mesmo ecossistema do StockHub, mantendo consistência visual total (cores, tipografia, componentes, animações).

### 1.1 Objetivos Principais

- Planejamento de demanda multinível (SKU × Hierarquia Comercial × Período)
- Comparação entre planos (Histórico, Orçamento, Forecast 1, Forecast 2…)
- Indicadores estatísticos e sugestões inteligentes de forecast
- Sistema de rateio hierárquico
- Controle de permissões granular
- Gestão de etapas e rodadas de planejamento
- Gerenciamento de preços para análises financeiras
- Sistema de de-para (mapeamento) de produtos e hierarquia
- Detecção de lacunas e inconsistências entre planos
- Dashboard modular de KPIs
- Internacionalização (pt-BR e en)

---

## 2. Stack Tecnológica

| Camada | Tecnologia |
|---|---|
| **Framework** | Next.js 16 (App Router, React 19) |
| **Estilização** | Tailwind CSS 3.4 + CSS Variables (HSL) |
| **Componentes UI** | shadcn/ui (estilo `new-york`, base `zinc`) |
| **Gráficos** | Recharts |
| **Estado Global** | Zustand |
| **Validação** | Zod |
| **Banco de Dados** | Supabase (PostgreSQL) |
| **Autenticação** | Supabase Auth |
| **Data Fetching** | TanStack React Query |
| **Tabelas** | TanStack React Table |
| **Toasts** | Sonner |
| **Ícones** | Lucide React |
| **Fontes** | Inter (Google Fonts) |
| **Linting** | Biome |
| **Hospedagem** | Cloudflare (Pages/Workers) |
| **i18n** | Zustand store (mesmo padrão StockHub) |
| **API Management** | tRPC ou Next.js Route Handlers |
| **CSV Parsing** | PapaParse |
| **Exports** | PapaParse (CSV) + ExcelJS (XLSX opcional) |

### 2.1 Design System (Herança do StockHub)

O DemandHub herdará **exatamente** o design system do StockHub:

- **Temas**: Dark/Light via `next-themes` com CSS variables HSL
- **Cores Base**: Zinc (cinza neutro) para background, foreground, cards
- **Cores Semânticas**: Green, Yellow, Red, Blue, Purple via CSS variables
- **Cores de Gráficos**: 5 cores de chart (`--chart-1` a `--chart-5`)
- **Border Radius**: `0.5rem` (--radius)
- **Tipografia**: Inter, `text-sm` como padrão do body
- **Focus States**: Sem outline, `box-shadow: none`, border-color no foreground
- **Scrollbar**: Customizada (thin, muted-foreground no thumb)
- **Animações**: fadeIn, fadeInUp, accordion-down/up com cubic-bezier
- **Seleção de Texto**: user-select: none global, liberado em inputs/conteúdo
- **Sidebar**: Collapsible com modos expanded/compact/closed

---

## 3. Arquitetura do Banco de Dados

### 3.1 Diagrama de Entidades

```
┌─────────────────────────────────────────────────────────────────┐
│                      ESTRUTURA CENTRAL                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  organizations ─┬─── users                                      │
│                 ├─── hierarchy_levels (níveis da hierarquia)     │
│                 ├─── hierarchy_nodes  (nós: regional, vendedor)  │
│                 ├─── products         (SKUs)                     │
│                 ├─── plans            (planos: histórico, fc1…)  │
│                 ├─── rounds           (rodadas de planejamento)  │
│                 ├─── stages           (etapas das rodadas)       │
│                 ├─── price_tables     (tabelas de preços)        │
│                 └─── mapping_rules    (de-para)                  │
│                                                                 │
├─────────────────────────────────────────────────────────────────┤
│                     DADOS DE DEMANDA                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  demand_data ──── (plan × product × hierarchy_path × period)    │
│  price_entries ── (price_table × product × hierarchy_path)      │
│                                                                 │
├─────────────────────────────────────────────────────────────────┤
│                    PERMISSÕES & CONTROLE                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  roles ─────────── role_permissions                              │
│  user_assignments  (user × hierarchy_node × plan × permissões)  │
│  period_locks      (round × period × editável ou não)           │
│                                                                 │
├─────────────────────────────────────────────────────────────────┤
│                      INDICADORES                               │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  kpi_definitions ── kpi_user_preferences (dashboard modular)    │
│  gap_alerts ─────── (lacunas detectadas entre planos)           │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 3.2 Tabelas Detalhadas

#### `organizations`
```sql
CREATE TABLE organizations (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  name TEXT NOT NULL,
  slug TEXT UNIQUE NOT NULL,
  settings JSONB DEFAULT '{}',
  created_at TIMESTAMPTZ DEFAULT now(),
  updated_at TIMESTAMPTZ DEFAULT now()
);
```

#### `hierarchy_levels` — Definição dos níveis da hierarquia comercial
```sql
-- Ex: Nível 1 = "Regional", Nível 2 = "Gerente", Nível 3 = "Vendedor"
CREATE TABLE hierarchy_levels (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  org_id UUID REFERENCES organizations(id) ON DELETE CASCADE,
  name TEXT NOT NULL,           -- Ex: "Regional", "Gerente", "Vendedor"
  depth INT NOT NULL,           -- 1, 2, 3... (ordem hierárquica)
  slug TEXT NOT NULL,
  created_at TIMESTAMPTZ DEFAULT now(),
  UNIQUE(org_id, depth),
  UNIQUE(org_id, slug)
);
```

#### `hierarchy_nodes` — Nós reais da hierarquia (instâncias)
```sql
-- Ex: Regional "Sul", Gerente "João Silva", Vendedor "Maria Santos"
CREATE TABLE hierarchy_nodes (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  org_id UUID REFERENCES organizations(id) ON DELETE CASCADE,
  level_id UUID REFERENCES hierarchy_levels(id) ON DELETE CASCADE,
  parent_id UUID REFERENCES hierarchy_nodes(id) ON DELETE SET NULL,
  code TEXT NOT NULL,            -- Código externo (ex: "REG-001")
  name TEXT NOT NULL,            -- Nome descritivo
  is_active BOOLEAN DEFAULT true,
  created_at TIMESTAMPTZ DEFAULT now(),
  updated_at TIMESTAMPTZ DEFAULT now(),
  UNIQUE(org_id, level_id, code)
);
CREATE INDEX idx_hierarchy_nodes_parent ON hierarchy_nodes(parent_id);
CREATE INDEX idx_hierarchy_nodes_level ON hierarchy_nodes(level_id);
```

#### `products` — Cadastro de SKUs
```sql
CREATE TABLE products (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  org_id UUID REFERENCES organizations(id) ON DELETE CASCADE,
  sku TEXT NOT NULL,
  name TEXT NOT NULL,
  category TEXT,
  subcategory TEXT,
  unit TEXT DEFAULT 'un',       -- Unidade de medida
  is_active BOOLEAN DEFAULT true,
  metadata JSONB DEFAULT '{}',
  created_at TIMESTAMPTZ DEFAULT now(),
  updated_at TIMESTAMPTZ DEFAULT now(),
  UNIQUE(org_id, sku)
);
```

#### `mapping_rules` — De-Para de produtos e hierarquia
```sql
-- Registra mapeamentos: código antigo → código novo
CREATE TABLE mapping_rules (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  org_id UUID REFERENCES organizations(id) ON DELETE CASCADE,
  entity_type TEXT NOT NULL CHECK (entity_type IN ('product', 'hierarchy_node')),
  source_id UUID NOT NULL,       -- ID da entidade origem
  target_id UUID NOT NULL,       -- ID da entidade destino
  effective_date DATE NOT NULL,  -- A partir de quando vale
  merge_history BOOLEAN DEFAULT true, -- Se deve unificar histórico
  notes TEXT,
  created_by UUID REFERENCES auth.users(id),
  created_at TIMESTAMPTZ DEFAULT now()
);
CREATE INDEX idx_mapping_source ON mapping_rules(entity_type, source_id);
CREATE INDEX idx_mapping_target ON mapping_rules(entity_type, target_id);
```

#### `plans` — Tipos de plano
```sql
-- Ex: "Faturamento Histórico", "Orçamento 2025", "Forecast 1", "Forecast 2"
CREATE TABLE plans (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  org_id UUID REFERENCES organizations(id) ON DELETE CASCADE,
  name TEXT NOT NULL,
  slug TEXT NOT NULL,
  type TEXT NOT NULL CHECK (type IN ('historical', 'budget', 'forecast')),
  description TEXT,
  is_editable BOOLEAN DEFAULT true,
  display_order INT DEFAULT 0,
  created_at TIMESTAMPTZ DEFAULT now(),
  updated_at TIMESTAMPTZ DEFAULT now(),
  UNIQUE(org_id, slug)
);
```

#### `rounds` — Rodadas de planejamento
```sql
-- Ex: "Rodada Janeiro 2025", "Rodada Fevereiro 2025"
CREATE TABLE rounds (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  org_id UUID REFERENCES organizations(id) ON DELETE CASCADE,
  name TEXT NOT NULL,
  reference_month DATE NOT NULL,   -- Mês de referência da rodada
  start_date DATE NOT NULL,        -- Início da rodada
  end_date DATE NOT NULL,          -- Fim da rodada
  status TEXT DEFAULT 'draft' CHECK (status IN ('draft', 'open', 'in_progress', 'closed', 'archived')),
  created_at TIMESTAMPTZ DEFAULT now(),
  updated_at TIMESTAMPTZ DEFAULT now()
);
```

#### `stages` — Etapas dentro de uma rodada
```sql
-- Ex: "Etapa 1: Preenchimento Vendedor", "Etapa 2: Revisão Gerente"
CREATE TABLE stages (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  round_id UUID REFERENCES rounds(id) ON DELETE CASCADE,
  name TEXT NOT NULL,
  description TEXT,
  stage_order INT NOT NULL,
  start_date TIMESTAMPTZ NOT NULL,
  end_date TIMESTAMPTZ NOT NULL,
  can_edit BOOLEAN DEFAULT true,
  allowed_hierarchy_levels UUID[],  -- Quais níveis hierárquicos podem editar nessa etapa
  allowed_plans UUID[],             -- Quais planos podem ser editados nessa etapa
  created_at TIMESTAMPTZ DEFAULT now()
);
```

#### `period_locks` — Travamento de períodos
```sql
CREATE TABLE period_locks (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  org_id UUID REFERENCES organizations(id) ON DELETE CASCADE,
  round_id UUID REFERENCES rounds(id) ON DELETE CASCADE,
  period_start DATE NOT NULL,
  period_end DATE NOT NULL,
  is_locked BOOLEAN DEFAULT false,
  locked_by UUID REFERENCES auth.users(id),
  locked_at TIMESTAMPTZ,
  reason TEXT,
  created_at TIMESTAMPTZ DEFAULT now()
);
```

#### `demand_data` — Dados de demanda (tabela principal)
```sql
-- Armazena todas as entradas: histórico, orçamento, forecasts
CREATE TABLE demand_data (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  org_id UUID REFERENCES organizations(id) ON DELETE CASCADE,
  plan_id UUID REFERENCES plans(id) ON DELETE CASCADE,
  product_id UUID REFERENCES products(id) ON DELETE CASCADE,
  round_id UUID REFERENCES rounds(id) ON DELETE SET NULL,
  -- Hierarquia: armazena o caminho completo como array de node IDs
  hierarchy_path UUID[] NOT NULL,  -- [regional_id, gerente_id, vendedor_id]
  period_date DATE NOT NULL,       -- Mês/período do dado (ex: 2025-01-01)
  volume NUMERIC(18,4) DEFAULT 0,  -- Volume/quantidade
  notes TEXT,
  last_edited_by UUID REFERENCES auth.users(id),
  created_at TIMESTAMPTZ DEFAULT now(),
  updated_at TIMESTAMPTZ DEFAULT now(),
  -- Garante unicidade por chave composta
  UNIQUE(org_id, plan_id, product_id, hierarchy_path, period_date)
);
CREATE INDEX idx_demand_plan ON demand_data(plan_id);
CREATE INDEX idx_demand_product ON demand_data(product_id);
CREATE INDEX idx_demand_period ON demand_data(period_date);
CREATE INDEX idx_demand_hierarchy ON demand_data USING GIN(hierarchy_path);
CREATE INDEX idx_demand_round ON demand_data(round_id);
```

#### `price_tables` — Tabelas de preços
```sql
CREATE TABLE price_tables (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  org_id UUID REFERENCES organizations(id) ON DELETE CASCADE,
  name TEXT NOT NULL,
  effective_date DATE NOT NULL,
  currency TEXT DEFAULT 'BRL',
  is_active BOOLEAN DEFAULT true,
  created_at TIMESTAMPTZ DEFAULT now(),
  updated_at TIMESTAMPTZ DEFAULT now()
);
```

#### `price_entries` — Preços por produto
```sql
CREATE TABLE price_entries (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  price_table_id UUID REFERENCES price_tables(id) ON DELETE CASCADE,
  product_id UUID REFERENCES products(id) ON DELETE CASCADE,
  hierarchy_path UUID[],          -- Pode variar por hierarquia (preço regional etc.)
  unit_price NUMERIC(18,4) NOT NULL,
  created_at TIMESTAMPTZ DEFAULT now(),
  updated_at TIMESTAMPTZ DEFAULT now(),
  UNIQUE(price_table_id, product_id, hierarchy_path)
);
```

#### `roles` e `role_permissions` — Sistema de permissões
```sql
CREATE TABLE roles (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  org_id UUID REFERENCES organizations(id) ON DELETE CASCADE,
  name TEXT NOT NULL,
  description TEXT,
  is_system BOOLEAN DEFAULT false, -- Roles do sistema (admin, viewer) não podem ser deletados
  created_at TIMESTAMPTZ DEFAULT now(),
  UNIQUE(org_id, name)
);

CREATE TABLE role_permissions (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  role_id UUID REFERENCES roles(id) ON DELETE CASCADE,
  resource TEXT NOT NULL,          -- Ex: 'plan', 'round', 'hierarchy', 'admin', 'price'
  action TEXT NOT NULL,            -- Ex: 'view', 'edit', 'create', 'delete', 'export'
  conditions JSONB DEFAULT '{}',   -- Condições extras (ex: {"plan_types": ["forecast"]})
  UNIQUE(role_id, resource, action)
);
```

#### `user_assignments` — Atribuição de usuários
```sql
CREATE TABLE user_assignments (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  org_id UUID REFERENCES organizations(id) ON DELETE CASCADE,
  user_id UUID REFERENCES auth.users(id) ON DELETE CASCADE,
  role_id UUID REFERENCES roles(id) ON DELETE CASCADE,
  hierarchy_node_id UUID REFERENCES hierarchy_nodes(id) ON DELETE SET NULL, -- Nível onde atua
  plan_ids UUID[],                 -- Quais planos pode acessar (null = todos)
  can_edit BOOLEAN DEFAULT false,
  can_export BOOLEAN DEFAULT false,
  created_at TIMESTAMPTZ DEFAULT now(),
  updated_at TIMESTAMPTZ DEFAULT now(),
  UNIQUE(org_id, user_id, hierarchy_node_id)
);
```

#### `kpi_definitions` — Definições de KPIs
```sql
CREATE TABLE kpi_definitions (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  org_id UUID REFERENCES organizations(id) ON DELETE CASCADE,
  name TEXT NOT NULL,
  slug TEXT NOT NULL,
  description TEXT,
  formula TEXT,                    -- Fórmula ou referência ao cálculo
  category TEXT,                   -- Ex: 'accuracy', 'bias', 'comparison', 'trend'
  chart_type TEXT DEFAULT 'bar',   -- Tipo de gráfico padrão (bar, line, area, pie)
  default_visible BOOLEAN DEFAULT true,
  display_order INT DEFAULT 0,
  created_at TIMESTAMPTZ DEFAULT now(),
  UNIQUE(org_id, slug)
);
```

#### `kpi_user_preferences` — Dashboard modular
```sql
CREATE TABLE kpi_user_preferences (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id UUID REFERENCES auth.users(id) ON DELETE CASCADE,
  org_id UUID REFERENCES organizations(id) ON DELETE CASCADE,
  kpi_id UUID REFERENCES kpi_definitions(id) ON DELETE CASCADE,
  is_visible BOOLEAN DEFAULT true,
  position INT DEFAULT 0,         -- Posição no dashboard
  size TEXT DEFAULT 'medium',     -- 'small', 'medium', 'large'
  created_at TIMESTAMPTZ DEFAULT now(),
  UNIQUE(user_id, kpi_id)
);
```

#### `gap_alerts` — Alertas de lacunas
```sql
CREATE TABLE gap_alerts (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  org_id UUID REFERENCES organizations(id) ON DELETE CASCADE,
  alert_type TEXT NOT NULL CHECK (alert_type IN (
    'missing_in_plan',        -- Chave existe em um plano mas não em outro
    'unmapped_product',       -- Produto sem de-para definido
    'unmapped_hierarchy',     -- Nó hierárquico sem de-para
    'orphan_data'             -- Dados referenciando entidade inativa
  )),
  source_plan_id UUID REFERENCES plans(id),
  target_plan_id UUID REFERENCES plans(id),
  product_id UUID REFERENCES products(id),
  hierarchy_path UUID[],
  period_date DATE,
  description TEXT,
  is_resolved BOOLEAN DEFAULT false,
  resolved_by UUID REFERENCES auth.users(id),
  resolved_at TIMESTAMPTZ,
  created_at TIMESTAMPTZ DEFAULT now()
);
CREATE INDEX idx_gap_alerts_org ON gap_alerts(org_id, is_resolved);
```

#### `import_logs` — Log de importações
```sql
CREATE TABLE import_logs (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  org_id UUID REFERENCES organizations(id) ON DELETE CASCADE,
  user_id UUID REFERENCES auth.users(id),
  file_name TEXT NOT NULL,
  entity_type TEXT NOT NULL,      -- 'demand_data', 'products', 'hierarchy', 'prices'
  target_plan_id UUID REFERENCES plans(id),
  rows_total INT DEFAULT 0,
  rows_success INT DEFAULT 0,
  rows_error INT DEFAULT 0,
  errors JSONB DEFAULT '[]',
  status TEXT DEFAULT 'pending' CHECK (status IN ('pending', 'processing', 'completed', 'failed')),
  created_at TIMESTAMPTZ DEFAULT now(),
  completed_at TIMESTAMPTZ
);
```

---

## 4. Sistema de Rateio

### 4.1 Conceito

Quando um volume é editado em um nível da hierarquia (ex: um SKU para um vendedor), o sistema distribui (rateia) esse volume para os demais níveis da hierarquia com base em uma **proporção selecionável**.

### 4.2 Proporções Disponíveis

| Proporção | Descrição |
|---|---|
| **Proporcional ao Histórico** | Distribui com base na participação histórica de cada nó |
| **Proporcional ao Plano Anterior** | Usa a distribuição do forecast/orçamento anterior |
| **Distribuição Igual** | Divide igualmente entre todos os nós filhos |
| **Peso Manual** | O usuário define pesos/percentuais para cada nó |
| **Proporcional ao Volume Atual** | Baseado nos volumes já preenchidos no plano atual |

### 4.3 Fluxo de Rateio

```
Edição no nível SKU (Vendedor "Maria" → Produto "X" → Jan/25 = 1000)
                    │
                    ▼
        Selecionar método de rateio
                    │
         ┌──────────┴──────────┐
         │                     │
    Top-Down              Bottom-Up
         │                     │
   Distribui para          Agrega para
   níveis inferiores      níveis superiores
         │                     │
    ▼ Recalcula            ▼ Recalcula
    nós filhos             nós pai
```

### 4.4 Implementação

```typescript
// Tipos de rateio
type ApportionMethod =
  | 'historical_share'
  | 'previous_plan_share'
  | 'equal_distribution'
  | 'manual_weights'
  | 'current_volume_share';

// Tabela auxiliar para pesos de rateio
CREATE TABLE apportion_configs (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  org_id UUID REFERENCES organizations(id) ON DELETE CASCADE,
  plan_id UUID REFERENCES plans(id) ON DELETE CASCADE,
  hierarchy_node_id UUID REFERENCES hierarchy_nodes(id),
  product_id UUID REFERENCES products(id),
  method TEXT NOT NULL,
  reference_plan_id UUID REFERENCES plans(id),  -- Plano de referência para proporção
  manual_weights JSONB,                          -- {"node_id": 0.3, "node_id": 0.7}
  created_at TIMESTAMPTZ DEFAULT now()
);
```

---

## 5. Sistema de De-Para (Mapeamento)

### 5.1 Conceito

Quando um produto muda de código (ex: SKU-001 → SKU-002) ou quando um vendedor é substituído por outro, o sistema deve manter a continuidade dos dados históricos e forecasts.

### 5.2 Fluxo

```
SKU-001 (descontinuado)  ──────mapping_rule────── ▶  SKU-002 (novo)
         │                                                  │
         │  Histórico Fev/24                                │  Histórico Fev/24
         │  = 500 unidades                                  │  = 150 unidades
         │                                                  │
         └───── merge_history = true ──────────────▶  Total = 650 unidades
                                                    (histórico unificado)
```

### 5.3 Comportamento

- **Com merge**: Histórico, orçamento e forecasts são consolidados na entidade destino
- **Sem merge**: Entidade origem é marcada como inativa, mas dados permanecem separados
- **Data efetiva**: O merge só afeta dados a partir da `effective_date`
- **Impacto nos alertas**: O sistema de lacunas considera os mapeamentos ativos

---

## 6. Indicadores e KPIs

### 6.1 KPIs de Acurácia

| KPI | Descrição | Fórmula |
|---|---|---|
| **MAPE** | Mean Absolute Percentage Error | `Σ|Ai - Fi| / Σ|Ai| × 100` |
| **WMAPE** | Weighted MAPE | `Σ|Ai - Fi| / Σ Ai × 100` |
| **Bias** | Viés do forecast | `Σ(Fi - Ai) / Σ Ai × 100` |
| **MAE** | Mean Absolute Error | `Σ|Ai - Fi| / n` |
| **RMSE** | Root Mean Square Error | `√(Σ(Ai - Fi)² / n)` |
| **Tracking Signal** | Sinal de rastreamento | `Σ(Ai - Fi) / MAD` |

### 6.2 KPIs Comparativos entre Planos

| KPI | Descrição |
|---|---|
| **Variação (%)** | Diferença percentual entre dois planos |
| **Variação Absoluta** | Diferença em volume entre dois planos |
| **Hit Rate** | % de SKUs dentro de uma margem de erro aceitável |
| **Correlação** | Coeficiente de Pearson entre dois planos |
| **Contribuição por Nível** | Participação de cada nível hierárquico no volume total |

### 6.3 KPIs de Gestão

| KPI | Descrição |
|---|---|
| **Fill Rate** | Taxa de preenchimento (SKUs com dados / Total SKUs) |
| **Cobertura Hierárquica** | % de nós hierárquicos com dados preenchidos |
| **Evolução por Rodada** | Evolução do forecast entre rodadas |
| **Concentração** | Índice de concentração (Pareto) por SKU ou hierarquia |
| **Receita Projetada** | Volume × Preço (integração com tabela de preços) |
| **Gap Count** | Quantidade de lacunas não resolvidas |

### 6.4 Sugestões Inteligentes de Forecast

O sistema deve oferecer sugestões estatísticas:

- **Média Móvel** (3, 6, 12 períodos)
- **Média Ponderada** (pesos configuráveis)
- **Tendência Linear** (regressão simples)
- **Sazonalidade** (decomposição do histórico)
- **Crescimento YoY** (Year over Year com ajuste)

---

## 7. Sistema de Permissões

### 7.1 Modelo RBAC (Role-Based Access Control)

```
┌────────────────────────────────────────────────────┐
│                    RECURSOS                        │
├────────────────────────────────────────────────────┤
│  plan          → view, edit, create, delete        │
│  round         → view, edit, create, close         │
│  stage         → view, edit, create                │
│  hierarchy     → view, edit, create, delete        │
│  product       → view, edit, create, delete        │
│  price         → view, edit, create, delete        │
│  mapping       → view, create, delete              │
│  demand_data   → view, edit, import, export        │
│  kpi           → view, configure                   │
│  user          → view, edit, create, delete        │
│  admin         → full_access                       │
│  api           → generate_key, revoke_key          │
│  gap_alerts    → view, resolve, dismiss            │
└────────────────────────────────────────────────────┘
```

### 7.2 Roles Padrão

| Role | Descrição | Permissões |
|---|---|---|
| **Admin** | Administrador total | Acesso total a todas funcionalidades |
| **Manager** | Gerente de demanda | Edita planos, gerencia rodadas, vê todos os níveis |
| **Planner** | Planejador | Edita o forecast no seu nível hierárquico |
| **Viewer** | Visualizador | Apenas consulta dados e KPIs |
| **API User** | Integração | Acesso via API apenas |

### 7.3 Permissões Granulares

- Associação **user ↔ hierarchy_node**: O usuário só vê/edita dados do seu nó e filhos
- Associação **user ↔ plan**: O usuário pode estar limitado a certos planos
- Override por **stage**: Na etapa 1 só vendedores editam; na etapa 2 só gerentes

---

## 8. Sistema de Etapas e Rodadas

### 8.1 Ciclo de Vida de uma Rodada

```
  DRAFT → OPEN → IN_PROGRESS → CLOSED → ARCHIVED
    │       │          │            │         │
    │       │     Etapas ativas     │         │
    │       │          │            │         │
    │       ▼          ▼            ▼         │
    │   Configurar   Edição      Dados        │
    │   etapas,     conforme     finalizados,  │
    │   períodos    permissões   somente       │
    │   e planos                 leitura       │
    │                                          │
    └──────────── Pode reabrir ────────────────┘
```

### 8.2 Períodos Editáveis

Cada rodada define quais períodos futuros podem ser editados:

```
Rodada de Janeiro/2025
├── Período Jan/25: TRAVADO (realizado)
├── Período Fev/25: EDITÁVEL
├── Período Mar/25: EDITÁVEL
├── Período Abr/25: EDITÁVEL
└── Período Mai/25 em diante: EDITÁVEL

Rodada de Fevereiro/2025 (móvel)
├── Período Jan/25: TRAVADO
├── Período Fev/25: TRAVADO (realizado)
├── Período Mar/25: EDITÁVEL
├── Período Abr/25: EDITÁVEL
└── Período Mai/25 em diante: EDITÁVEL
```

---

## 9. Alertas de Lacunas e Sincronização

### 9.1 Tipos de Alertas

| Tipo | Situação | Ação Sugerida |
|---|---|---|
| `missing_in_plan` | SKU×Hierarquia existe no Histórico mas não no Forecast | Criar entrada no Forecast |
| `unmapped_product` | Produto novo sem de-para configurado | Configurar mapeamento |
| `unmapped_hierarchy` | Nó hierárquico novo sem de-para | Configurar mapeamento |
| `orphan_data` | Dados referenciando entidade desativada sem mapeamento | Resolver mapeamento |

### 9.2 Motor de Detecção

Um job (cron ou trigger) que:

1. Compara as chaves únicas (product × hierarchy_path × period) entre planos
2. Verifica se entidades inativas têm mapeamento válido
3. Gera/atualiza registros na tabela `gap_alerts`
4. Considera as `mapping_rules` ativas ao avaliar as lacunas

---

## 10. Importação e Exportação de Dados

### 10.1 Importação via CSV

| Entidade | Campos Obrigatórios | Campos Opcionais |
|---|---|---|
| **Produtos** | sku, name | category, subcategory, unit |
| **Hierarquia** | level_slug, code, name | parent_code |
| **Demanda** | plan_slug, sku, period, hierarchy_codes*, volume | notes |
| **Preços** | price_table_name, sku, unit_price | hierarchy_codes* |
| **Mapeamentos** | entity_type, source_code, target_code, effective_date | merge_history |

> *hierarchy_codes é uma coluna por nível: `regional_code`, `gerente_code`, `vendedor_code`

### 10.2 Fluxo de Importação

```
Upload CSV → Parsing (PapaParse) → Validação (Zod) → Preview
     │                                                   │
     │                                                   ▼
     │                                              Confirmar
     │                                                   │
     │                                                   ▼
     └─────────── Import Log ◄──── Inserção no Banco ────┘
                                       (com transaction)
```

### 10.3 Exportação

- Todos os dados exportáveis em CSV
- Filtros aplicados na tela são mantidos na exportação
- Opção de incluir metadados (cabeçalhos descritivos, data de exportação)

---

## 11. API de Integração

### 11.1 Endpoints Planejados

```
GET  /api/v1/plans
GET  /api/v1/plans/:id/data
GET  /api/v1/demand?plan=forecast1&period=2025-01
GET  /api/v1/products
GET  /api/v1/hierarchy
GET  /api/v1/kpis?plan=forecast1&compare_to=historical
GET  /api/v1/prices
POST /api/v1/demand/export
```

### 11.2 Autenticação da API

- API Keys geradas por usuário com role `api`
- Chaves associadas a permissões específicas
- Rate limiting
- Formato de resposta: JSON (para Power BI, Tableau) ou CSV

---

## 12. Internacionalização (i18n)

### 12.1 Implementação

Mesmo padrão do StockHub: **Zustand store** com arquivos de tradução `pt.ts` e `en.ts`.

```
src/lib/i18n/
├── store.ts        (Zustand store com função `t()`)
├── provider.tsx    (Provider que detecta idioma)
├── types.ts        (Tipos das chaves de tradução)
├── pt.ts           (Traduções pt-BR)
└── en.ts           (Traduções en)
```

### 12.2 Namespaces de Tradução

```
common.*          → Botões, labels genéricos
sidebar.*         → Menu lateral
planning.*        → Área de planejamento
rounds.*          → Rodadas e etapas
imports.*         → Importação/exportação
kpis.*            → Indicadores
admin.*           → Painel administrativo
permissions.*     → Sistema de permissões
alerts.*          → Alertas e lacunas
prices.*          → Preços
mappings.*        → De-para
validation.*      → Mensagens de validação (Zod)
```

---

## 13. Estrutura de Pastas do Projeto

```
demand-hub/
├── public/
│   └── images/               (logos, ícones)
├── src/
│   ├── app/
│   │   ├── globals.css       (CSS variables — herança StockHub)
│   │   ├── layout.tsx        (Root layout com providers)
│   │   ├── providers.tsx     (React Query, Theme, etc.)
│   │   ├── page.tsx          (Dashboard principal)
│   │   ├── login/
│   │   ├── planning/         (Área de planejamento)
│   │   │   ├── page.tsx      (Grid de planejamento principal)
│   │   │   ├── [planId]/
│   │   │   │   └── page.tsx
│   │   │   └── compare/
│   │   │       └── page.tsx  (Comparação entre planos)
│   │   ├── rounds/           (Rodadas)
│   │   │   ├── page.tsx
│   │   │   └── [roundId]/
│   │   │       └── page.tsx
│   │   ├── products/         (Cadastro de SKUs)
│   │   │   └── page.tsx
│   │   ├── hierarchy/        (Estrutura comercial)
│   │   │   └── page.tsx
│   │   ├── mappings/         (De-para)
│   │   │   └── page.tsx
│   │   ├── prices/           (Tabelas de preço)
│   │   │   └── page.tsx
│   │   ├── indicators/       (Dashboard de KPIs)
│   │   │   └── page.tsx
│   │   ├── imports/          (Importação/exportação)
│   │   │   └── page.tsx
│   │   ├── alerts/           (Lacunas e alertas)
│   │   │   └── page.tsx
│   │   ├── settings/         (Configurações do usuário)
│   │   │   └── page.tsx
│   │   ├── admin/            (Painel administrativo)
│   │   │   ├── page.tsx
│   │   │   ├── users/
│   │   │   ├── roles/
│   │   │   ├── plans/
│   │   │   ├── rounds/
│   │   │   └── api-keys/
│   │   └── api/
│   │       └── v1/           (API routes)
│   ├── auth/                 (Helpers de autenticação Supabase)
│   ├── components/
│   │   ├── ui/               (shadcn/ui components)
│   │   ├── sidebar-left.tsx
│   │   ├── navbar.tsx
│   │   ├── layout.tsx        (Shell com sidebar + navbar)
│   │   ├── planning/         (Componentes de planejamento)
│   │   │   ├── planning-grid.tsx
│   │   │   ├── plan-comparison.tsx
│   │   │   ├── apportion-dialog.tsx
│   │   │   └── forecast-suggestions.tsx
│   │   ├── import/
│   │   │   ├── csv-uploader.tsx
│   │   │   └── import-preview.tsx
│   │   └── kpis/
│   │       ├── kpi-card.tsx
│   │       ├── kpi-grid.tsx
│   │       └── dashboard-configurator.tsx
│   ├── contexts/
│   │   ├── sidebar-context.tsx
│   │   ├── theme-context.tsx
│   │   └── notification-context.tsx
│   ├── hooks/
│   │   ├── use-demand-data.ts
│   │   ├── use-plans.ts
│   │   ├── use-hierarchy.ts
│   │   ├── use-permissions.ts
│   │   ├── use-kpis.ts
│   │   └── use-apportion.ts
│   ├── lib/
│   │   ├── utils.ts          (cn, formatters)
│   │   ├── supabase/
│   │   │   ├── client.ts
│   │   │   ├── server.ts
│   │   │   └── middleware.ts
│   │   ├── i18n/
│   │   │   ├── store.ts
│   │   │   ├── provider.tsx
│   │   │   ├── types.ts
│   │   │   ├── pt.ts
│   │   │   └── en.ts
│   │   ├── statistics/       (Funções estatísticas)
│   │   │   ├── forecast.ts   (Média móvel, tendência, sazonalidade)
│   │   │   ├── comparison.ts (MAPE, WMAPE, Bias, correlação)
│   │   │   └── apportion.ts  (Lógica de rateio)
│   │   └── validators/       (Schemas Zod)
│   │       ├── demand.ts
│   │       ├── product.ts
│   │       ├── hierarchy.ts
│   │       └── import.ts
│   ├── services/             (API calls / Supabase queries)
│   │   ├── demand.ts
│   │   ├── plans.ts
│   │   ├── hierarchy.ts
│   │   ├── products.ts
│   │   ├── prices.ts
│   │   ├── mappings.ts
│   │   ├── rounds.ts
│   │   ├── permissions.ts
│   │   └── gap-alerts.ts
│   ├── stores/               (Zustand stores)
│   │   ├── planning-store.ts
│   │   ├── filter-store.ts
│   │   └── kpi-store.ts
│   ├── types/
│   │   ├── database.ts       (Tipos gerados do Supabase)
│   │   ├── demand.ts
│   │   ├── hierarchy.ts
│   │   ├── plan.ts
│   │   └── kpi.ts
│   └── utils/
│       ├── csv.ts            (Helpers CSV)
│       ├── date.ts           (Helpers de datas)
│       └── number.ts         (Formatadores numéricos)
├── biome.json
├── components.json
├── next.config.mjs
├── package.json
├── postcss.config.mjs
├── tailwind.config.ts
└── tsconfig.json
```

---

## 14. Fases de Implementação

### Fase 1 — Fundação (Semanas 1-3)
- [ ] Criar projeto Next.js com Tailwind, shadcn/ui e configuração base
- [ ] Herdar design system do StockHub (globals.css, tailwind.config, componentes ui)
- [ ] Configurar Supabase (auth, banco, RLS)
- [ ] Implementar i18n (pt-BR + en)
- [ ] Criar layout base (sidebar, navbar, shell)
- [ ] Criar tela de login/auth
- [ ] Deploy inicial na Cloudflare

### Fase 2 — Estrutura de Dados (Semanas 3-5)
- [ ] Criar migrations do banco de dados
- [ ] Implementar CRUD de produtos (SKUs)
- [ ] Implementar CRUD da hierarquia comercial (níveis e nós)
- [ ] Implementar CRUD de planos
- [ ] Sistema de importação CSV (produtos, hierarquia)
- [ ] Validação com Zod

### Fase 3 — Planejamento Core (Semanas 5-8)
- [ ] Grid de planejamento (editável, com filtros por plano/hierarquia/período)
- [ ] Entrada de dados de demanda
- [ ] Sistema de rateio (apportion)
- [ ] Importação de dados de demanda via CSV
- [ ] Comparação entre planos (side-by-side)
- [ ] Exportação de dados em CSV

### Fase 4 — Rodadas e Permissões (Semanas 8-10)
- [ ] CRUD de rodadas e etapas
- [ ] Sistema de permissões (roles, atribuições)
- [ ] Travamento/liberação de períodos
- [ ] Controle de edição por etapa e nível hierárquico
- [ ] Painel Admin (gestão de usuários, roles, planos, rodadas)

### Fase 5 — Indicadores (Semanas 10-12)
- [ ] Funções estatísticas (MAPE, WMAPE, Bias, correlação)
- [ ] Sugestões inteligentes de forecast
- [ ] Dashboard modular de KPIs (drag & drop, configurável)
- [ ] Gráficos com Recharts (bar, line, area, pie)

### Fase 6 — De-Para e Alertas (Semanas 12-14)
- [ ] CRUD de mapeamentos (de-para)
- [ ] Motor de detecção de lacunas
- [ ] Tela de alertas
- [ ] Integração de-para com rateio e indicadores

### Fase 7 — Preços e Financeiro (Semanas 14-15)
- [ ] CRUD de tabelas de preços
- [ ] Importação de preços via CSV
- [ ] Análises financeiras (Receita Projetada = Volume × Preço)
- [ ] KPIs financeiros no dashboard

### Fase 8 — API e Polish (Semanas 15-17)
- [ ] API REST v1 (endpoints de leitura)
- [ ] Sistema de API Keys
- [ ] Otimização de performance
- [ ] Testes e2e
- [ ] Documentação

---

## 15. Decisões Técnicas e Premissas

### 15.1 Premissas Assumidas

1. **Granularidade de período**: Mensal (DATE armazenado como primeiro dia do mês). Se precisar de outras granularidades (semanal, diária), podemos ajustar.

2. **hierarchy_path como UUID[]**: Armazena o caminho completo da hierarquia como array. Permite queries flexíveis com GIN index. Alternativa seria tabela de closure.

3. **Cloudflare Pages**: O Next.js App Router com SSR será hospedado via Cloudflare Pages (com Edge runtime onde necessário). Verificar compatibilidade de features.

4. **Single-tenant por organização**: Cada organização tem seus dados isolados via `org_id` + RLS.

5. **API com Route Handlers**: Inicialmente usando Next.js Route Handlers. Se a complexidade crescer, migrar para tRPC.

### 15.2 Pontos que Requerem Decisão

> [!IMPORTANT]
> Os pontos abaixo precisam da sua validação antes de iniciar a implementação.

1. **Granularidade de período**: Está tudo mensal ou pode haver semanal/diário?

2. **Quantidade máxima de níveis hierárquicos**: Existe um limite prático? (3, 5, ilimitado?)

3. **Multi-organização**: O DemandHub terá suporte a múltiplas organizações desde o início ou será single-tenant?

4. **Dados de demanda**: O volume é sempre numérico ou pode haver outros campos (ex: valor financeiro direto, quantidades em diferentes unidades)?

5. **Cálculos de KPI**: Devem rodar no front-end (real-time) ou no backend (pre-computed / materialized views)?

6. **Histórico de edições**: Precisamos de audit trail (quem editou o quê e quando) na tabela de demanda?

7. **Nome do domínio/subdomínio planejado**: Para configurar o deploy na Cloudflare.

8. **Existem dados reais para testes iniciais** ou devemos criar um gerador de dados de exemplo (seeder)?

---

## 16. Referências Visuais (StockHub)

O DemandHub deve seguir **exatamente** os mesmos padrões visuais do StockHub:

| Elemento | Padrão StockHub |
|---|---|
| **Cores** | HSL CSS variables, zinc base, dark/light |
| **Bordas** | `0.5rem` radius, `border-foreground` em componentes Radix |
| **Botões** | shadcn/ui Button, variantes outline/default/destructive |
| **Tabelas** | TanStack Table com header sticky, zebra striping sutil |
| **Gráficos** | Recharts com palette `chart-1` a `chart-5` |
| **Sidebar** | Collapsible (expanded/compact/closed), logo com transição |
| **Toasts** | Sonner (top-right) |
| **Selects** | Radix Select, sem box-shadow no focus |
| **Fonts** | Inter, text-sm como base |
| **Scrollbar** | Customizada thin, muted-foreground |
| **Animações** | fadeIn, fadeInUp, accordion-down/up (cubic-bezier) |
| **Tema** | `next-themes` com class strategy |
