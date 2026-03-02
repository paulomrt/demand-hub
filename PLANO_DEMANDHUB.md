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
│   │   │   ├── engagement/   (Dashboard de engajamento/audit)
│   │   │   │   └── page.tsx
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
│   │   ├── audit.ts          (Consultas ao audit_log)
│   │   ├── kpi-cache.ts      (KPI cache / materialized views)
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
├── scripts/
│   └── seed/                (Gerador de dados de exemplo)
│       ├── index.ts
│       ├── organizations.ts
│       ├── hierarchy.ts
│       ├── products.ts
│       ├── plans.ts
│       ├── demand.ts
│       ├── prices.ts
│       ├── rounds.ts
│       ├── users.ts
│       └── utils.ts
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
- [ ] Configurar granularidade de período no JSONB de settings da organização
- [ ] Deploy inicial na Cloudflare (`demandhub.com.br`)
- [ ] Criar seeder básico (organizações e hierarquia)

### Fase 2 — Estrutura de Dados (Semanas 3-5)
- [ ] Criar migrations do banco de dados
- [ ] Implementar CRUD de produtos (SKUs)
- [ ] Implementar CRUD da hierarquia comercial (níveis adaptativos, até 5 níveis)
- [ ] Implementar CRUD de planos
- [ ] Sistema de importação CSV (produtos, hierarquia)
- [ ] Validação com Zod
- [ ] Seeder completo (produtos, planos, demanda, preços)

### Fase 3 — Planejamento Core (Semanas 5-8)
- [ ] Grid de planejamento (editável, com filtros por plano/hierarquia/período, adaptativo por granularidade)
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
- [ ] Configurar audit trail (tabelas, triggers, audit_log)
- [ ] Dashboard de engajamento (Top Contributors, Heatmap, Inativos)

### Fase 5 — Indicadores (Semanas 10-12)
- [ ] Materialized Views e SQL Functions para KPIs pesados
- [ ] Tabela kpi_cache com TTL e invalidação
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
- [ ] Configuração DNS final `demandhub.com.br` + SSL + Page Rules
- [ ] Revisão e refinamento do seeder para demonstrações

---

## 15. Decisões Técnicas Confirmadas

> [!NOTE]
> Todas as decisões abaixo foram validadas e confirmadas. Servem como referência arquitetural para a implementação.

### 15.1 Granularidade de Período — Configurável por Organização

**Decisão**: O período é **configurável por organização**, podendo ser `monthly` (mensal), `weekly` (semanal) ou `daily` (diário).

#### Regras de Funcionamento

| Aspecto | Comportamento |
|---|---|
| **Quando é definido** | Na criação da organização, no campo `settings.period_granularity` |
| **Pode trocar depois?** | **Sim**, mas com restrições e processo de conversão |
| **Armazenamento** | Sempre `DATE` — o significado muda conforme a granularidade |
| **Padrão** | `monthly` (primeiro dia do mês: `2025-01-01`, `2025-02-01`…) |

#### Representação por Granularidade

| Granularidade | `period_date` representa | Exemplo |
|---|---|---|
| `monthly` | Primeiro dia do mês | `2025-01-01` = Janeiro/2025 |
| `weekly` | Segunda-feira da semana | `2025-01-06` = Semana 2 de Jan/2025 |
| `daily` | O dia exato | `2025-01-15` = 15/Jan/2025 |

#### Mudança de Granularidade

A troca de granularidade é um processo administrativo com as seguintes regras:

```
Cenários de transição:

1. MENSAL → SEMANAL (desagregação)
   - O volume mensal é dividido igualmente entre as semanas do mês
   - Ex: Jan/2025 = 1000un → Semana 1 = 250, Semana 2 = 250, Semana 3 = 250, Semana 4 = 250
   - Ajustes manuais são necessários após a conversão (sazonalidade semanal se perde)

2. MENSAL → DIÁRIO (desagregação)
   - O volume mensal é dividido igualmente entre os dias úteis do mês
   - Ex: Jan/2025 = 1000un (22 dias úteis) → ~45.45 por dia útil
   - Dias não-úteis recebem volume 0 por padrão

3. SEMANAL → MENSAL (agregação)
   - As semanas são somadas para compor o mês
   - Semanas que cruzam meses são rateadas proporcionalmente (dias no mês / dias na semana)

4. DIÁRIO → MENSAL (agregação)
   - Todos os dias do mês são somados diretamente

5. DIÁRIO → SEMANAL (agregação)
   - Os 7 dias da semana são somados
```

#### Impacto no Banco de Dados

```sql
-- A granularidade é armazenada nas settings da organização
-- organizations.settings JSONB inclui:
{
  "period_granularity": "monthly",        -- "monthly" | "weekly" | "daily"
  "fiscal_year_start_month": 1,           -- Mês de início do ano fiscal
  "week_start_day": 1,                    -- 1=Segunda, 0=Domingo
  "consider_business_days_only": false    -- Para granularidade diária
}
```

#### Impacto na UI

- **Filtros de período**: Adaptam-se dinamicamente (seletor de mês, semana ou dia)
- **Grid de planejamento**: Colunas representam o período conforme a granularidade
- **Importação CSV**: Valida o formato de data conforme a granularidade ativa
- **KPIs**: Cálculos ajustam a janela temporal conforme a granularidade

---

### 15.2 Hierarquia Adaptativa — Até 5 Níveis

**Decisão**: O sistema suporta **até 5 níveis hierárquicos**, adaptando-se automaticamente à quantidade efetivamente utilizada por cada organização.

#### Como Funciona na Prática

A hierarquia é **dinâmica** — os níveis são criados conforme a necessidade da organização. Se uma empresa precisa de apenas 3 níveis, ela simplesmente cria 3 `hierarchy_levels` e o sistema inteiro se adapta.

| Empresa | Nível 1 | Nível 2 | Nível 3 | Nível 4 | Nível 5 |
|---|---|---|---|---|---|
| **Empresa A** (3 níveis) | Diretoria | Gerência | Vendedor | — | — |
| **Empresa B** (5 níveis) | VP | Diretoria | Regional | Gerência | Vendedor |
| **Empresa C** (2 níveis) | Região | Representante | — | — | — |

#### Comportamento Adaptativo do Sistema

```
┌─────────────────────────────────────────────────────────────────┐
│           ADAPTAÇÃO AUTOMÁTICA POR CAMADA                       │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  hierarchy_levels (org_id = Empresa A)                          │
│  ├── depth=1, name="Diretoria", slug="diretoria"               │
│  ├── depth=2, name="Gerência", slug="gerencia"                 │
│  └── depth=3, name="Vendedor", slug="vendedor"                 │
│                                                                 │
│  hierarchy_path UUID[] para essa empresa:                       │
│  → [diretoria_node_id, gerencia_node_id, vendedor_node_id]     │
│  → Sempre tamanho 3 (= número de níveis da org)                │
│                                                                 │
├─────────────────────────────────────────────────────────────────┤
│                     IMPACTO NA UI                               │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Sidebar / Filtros:                                             │
│  → Mostra APENAS os níveis criados                              │
│  → Ex: Empresa A vê filtros "Diretoria > Gerência > Vendedor"  │
│  → Ex: Empresa C vê filtros "Região > Representante"           │
│                                                                 │
│  Grid de Planejamento:                                          │
│  → Colunas de hierarquia ajustam-se ao nº de níveis            │
│  → Breadcrumb de navegação reflete os níveis existentes        │
│                                                                 │
│  Importação CSV:                                                │
│  → Colunas esperadas = slugs dos níveis criados                │
│  → Ex: Empresa A espera: diretoria_code, gerencia_code,        │
│        vendedor_code                                            │
│  → Ex: Empresa C espera: regiao_code, representante_code       │
│                                                                 │
│  Rateio:                                                        │
│  → Funciona entre quaisquer níveis adjacentes                  │
│  → Top-down: distribui do nível N para N+1                     │
│  → Bottom-up: agrega do nível N para N-1                       │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

#### Validações

- **Limite**: Máximo de 5 registros em `hierarchy_levels` por `org_id` (validado na API e no banco com CHECK constraint)
- **Profundidade contínua**: Os `depth` devem ser sequenciais (1, 2, 3…), sem pular valores
- **Exclusão de nível**: Não é permitido excluir um nível que possua `hierarchy_nodes` associados. É necessário realocar ou remover os nós primeiro.
- **Reordenação**: Alterar a profundidade de um nível exige migração dos `hierarchy_path` em `demand_data`

```sql
-- Constraint no banco para limitar a 5 níveis
CREATE OR REPLACE FUNCTION check_max_hierarchy_levels()
RETURNS TRIGGER AS $$
BEGIN
  IF (SELECT COUNT(*) FROM hierarchy_levels WHERE org_id = NEW.org_id) >= 5 THEN
    RAISE EXCEPTION 'Máximo de 5 níveis hierárquicos por organização';
  END IF;
  RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trg_max_hierarchy_levels
  BEFORE INSERT ON hierarchy_levels
  FOR EACH ROW EXECUTE FUNCTION check_max_hierarchy_levels();
```

---

### 15.3 Multi-organização (Multi-tenant)

**Decisão**: O DemandHub terá suporte a **múltiplas organizações desde o início**.

- Todas as tabelas possuem `org_id` como foreign key
- RLS (Row Level Security) garante isolamento total de dados entre organizações
- Um usuário pode pertencer a mais de uma organização
- O contexto de organização ativa é mantido via sessão/cookie
- Admin global (superadmin) pode acessar dados de qualquer organização para suporte

---

### 15.4 Dados de Demanda — Volume Numérico

**Decisão**: O campo `volume` em `demand_data` é exclusivamente **numérico** (`NUMERIC(18,4)`).

- Representa **quantidades/volumes** (unidades, caixas, kg, litros etc.)
- A **valorização financeira** é derivada multiplicando `volume × unit_price` da tabela de preços (`price_entries`)
- A unidade de medida é definida no cadastro do produto (`products.unit`)
- Não há campos de valor financeiro direto na tabela de demanda — isso mantém a separação de responsabilidades e permite trocar tabelas de preço sem reprocessar a demanda

---

### 15.5 Cálculos de KPI — Backend com Materialized Views

**Decisão**: Os KPIs são calculados no **backend**, utilizando uma estratégia híbrida de **Materialized Views + SQL Functions + Cache**.

#### Arquitetura de Cálculo

```
┌─────────────────────────────────────────────────────────────────┐
│                 ESTRATÉGIA DE KPIs                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Camada 1 — Materialized Views (Heavy Aggregations)            │
│  ├── Refreshed via cron (a cada 15-30min ou sob demanda)       │
│  ├── KPIs pesados: MAPE, WMAPE, Bias por plano/hierarquia     │
│  ├── Agregações de volume por período/hierarquia               │
│  └── Ideal para Dashboard principal e relatórios               │
│                                                                 │
│  Camada 2 — SQL Functions (On-Demand)                          │
│  ├── Chamadas via RPC do Supabase                              │
│  ├── KPIs leves: totais, contagens, variações simples          │
│  ├── Comparações entre 2 planos específicos                    │
│  └── Ideal para drill-down e análises ad-hoc                   │
│                                                                 │
│  Camada 3 — Cache de Resultados (kpi_cache)                    │
│  ├── Tabela que armazena resultados pré-computados             │
│  ├── Chave: org_id + kpi_slug + filtros (JSONB)               │
│  ├── TTL configurável por tipo de KPI                          │
│  └── Invalidado quando demand_data é modificado                │
│                                                                 │
│  Camada 4 — Frontend (Formatação e Micro-cálculos)             │
│  ├── Formatação de números, percentuais, cores                 │
│  ├── Cálculos triviais entre KPIs já recebidos                 │
│  └── Sem queries diretas ao banco para KPIs                    │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

#### Tabela `kpi_cache`

```sql
CREATE TABLE kpi_cache (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  org_id UUID REFERENCES organizations(id) ON DELETE CASCADE,
  kpi_slug TEXT NOT NULL,
  filters JSONB DEFAULT '{}',     -- Ex: {"plan_id": "...", "hierarchy_path": [...]}
  result JSONB NOT NULL,          -- Ex: {"value": 12.5, "trend": "up", "details": {...}}
  computed_at TIMESTAMPTZ DEFAULT now(),
  expires_at TIMESTAMPTZ NOT NULL,
  created_at TIMESTAMPTZ DEFAULT now()
);
CREATE INDEX idx_kpi_cache_lookup ON kpi_cache(org_id, kpi_slug, filters);
CREATE INDEX idx_kpi_cache_expiry ON kpi_cache(expires_at);
```

#### Por que Backend e não Frontend?

| Critério | Frontend | Backend (escolhido) |
|---|---|---|
| **Volume de dados** | Transfere milhares de registros para o browser | Processa no banco, retorna apenas resultado |
| **Performance** | Lento com grandes datasets | Rápido — PostgreSQL otimizado para aggregations |
| **Consistência** | Cada usuário calcula independentemente | Resultado único e compartilhado |
| **Segurança** | Dados brutos expostos ao client | Apenas KPIs calculados são retornados |
| **Offline** | Requer recalcular a cada reload | Cache persistente |

---

### 15.6 Domínio e Deploy

**Decisão**: O domínio de produção será **`demandhub.com.br`**.

| Ambiente | URL | Hospedagem |
|---|---|---|
| **Produção** | `demandhub.com.br` | Cloudflare Pages |
| **Preview/Staging** | `staging.demandhub.com.br` | Cloudflare Pages (branch preview) |
| **API** | `demandhub.com.br/api/v1/...` | Route Handlers (same origin) |

#### Configuração DNS (Cloudflare)

```
Tipo    Nome                  Valor                         Proxy
CNAME   demandhub.com.br      <pages-project>.pages.dev     ✅ Proxied
CNAME   www                   demandhub.com.br              ✅ Proxied
CNAME   staging               <pages-project>.pages.dev     ✅ Proxied
```

- SSL/TLS gerenciado automaticamente pelo Cloudflare
- Redirect `www.demandhub.com.br` → `demandhub.com.br` via Page Rule
- Cabeçalhos de segurança (CSP, HSTS) configurados via `_headers` file

---

### 15.7 Premissas Técnicas Adicionais

1. **hierarchy_path como UUID[]**: Armazena o caminho completo da hierarquia como array. Permite queries flexíveis com GIN index. O tamanho do array corresponde ao número de níveis da organização.

2. **Cloudflare Pages com Edge Runtime**: O Next.js App Router com SSR será hospedado via Cloudflare Pages. Features que exigem Node.js runtime serão executadas via Route Handlers com adapter `@cloudflare/next-on-pages`.

3. **API com Route Handlers**: Inicialmente usando Next.js Route Handlers. Se a complexidade crescer, avaliar migração para tRPC.

---

## 16. Sistema de Auditoria e Engajamento

O DemandHub mantém um **audit trail completo** de todas as alterações realizadas na plataforma, servindo tanto para rastreabilidade de dados quanto para avaliação de desempenho e engajamento dos usuários.

### 16.1 Tabela `audit_log` — Registro de Todas as Ações

```sql
CREATE TABLE audit_log (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  org_id UUID REFERENCES organizations(id) ON DELETE CASCADE,
  user_id UUID REFERENCES auth.users(id) ON DELETE SET NULL,
  action TEXT NOT NULL,               -- 'create', 'update', 'delete', 'import', 'export', 'login', 'logout'
  entity_type TEXT NOT NULL,          -- 'demand_data', 'product', 'hierarchy_node', 'plan', 'round', 'price', 'mapping', 'role', 'user_assignment'
  entity_id UUID,                     -- ID do registro afetado
  changes JSONB,                      -- Snapshot do antes/depois: {"old": {...}, "new": {...}}
  metadata JSONB DEFAULT '{}',        -- Contexto adicional: IP, user-agent, plano editado, etc.
  created_at TIMESTAMPTZ DEFAULT now()
);
CREATE INDEX idx_audit_log_org ON audit_log(org_id, created_at DESC);
CREATE INDEX idx_audit_log_user ON audit_log(user_id, created_at DESC);
CREATE INDEX idx_audit_log_entity ON audit_log(entity_type, entity_id);
CREATE INDEX idx_audit_log_action ON audit_log(action);
```

### 16.2 Trigger Automático para `demand_data`

```sql
-- Registra automaticamente toda edição na tabela de demanda
CREATE OR REPLACE FUNCTION fn_audit_demand_data()
RETURNS TRIGGER AS $$
BEGIN
  IF TG_OP = 'UPDATE' THEN
    INSERT INTO audit_log (org_id, user_id, action, entity_type, entity_id, changes)
    VALUES (
      NEW.org_id,
      NEW.last_edited_by,
      'update',
      'demand_data',
      NEW.id,
      jsonb_build_object(
        'old', jsonb_build_object('volume', OLD.volume, 'notes', OLD.notes),
        'new', jsonb_build_object('volume', NEW.volume, 'notes', NEW.notes),
        'plan_id', NEW.plan_id,
        'product_id', NEW.product_id,
        'period_date', NEW.period_date,
        'hierarchy_path', NEW.hierarchy_path
      )
    );
  ELSIF TG_OP = 'INSERT' THEN
    INSERT INTO audit_log (org_id, user_id, action, entity_type, entity_id, changes)
    VALUES (
      NEW.org_id,
      NEW.last_edited_by,
      'create',
      'demand_data',
      NEW.id,
      jsonb_build_object('new', jsonb_build_object('volume', NEW.volume, 'plan_id', NEW.plan_id))
    );
  ELSIF TG_OP = 'DELETE' THEN
    INSERT INTO audit_log (org_id, user_id, action, entity_type, entity_id, changes)
    VALUES (
      OLD.org_id,
      OLD.last_edited_by,
      'delete',
      'demand_data',
      OLD.id,
      jsonb_build_object('old', jsonb_build_object('volume', OLD.volume, 'plan_id', OLD.plan_id))
    );
  END IF;
  RETURN COALESCE(NEW, OLD);
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trg_audit_demand_data
  AFTER INSERT OR UPDATE OR DELETE ON demand_data
  FOR EACH ROW EXECUTE FUNCTION fn_audit_demand_data();
```

### 16.3 Tabela `user_activity_summary` — Agregações de Engajamento

```sql
-- Tabela pre-computada (refreshed via cron) para queries rápidas no dashboard admin
CREATE TABLE user_activity_summary (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  org_id UUID REFERENCES organizations(id) ON DELETE CASCADE,
  user_id UUID REFERENCES auth.users(id) ON DELETE CASCADE,
  period_start DATE NOT NULL,         -- Início do período de agregação (dia/semana/mês)
  period_type TEXT NOT NULL CHECK (period_type IN ('daily', 'weekly', 'monthly')),
  -- Métricas de Engajamento
  total_logins INT DEFAULT 0,
  total_edits INT DEFAULT 0,          -- Edições em demand_data
  total_creates INT DEFAULT 0,        -- Novos registros de demanda criados
  total_imports INT DEFAULT 0,        -- Importações CSV realizadas
  total_exports INT DEFAULT 0,        -- Exportações realizadas
  cells_edited INT DEFAULT 0,         -- Células únicas (SKU×Hierarquia×Período) editadas
  plans_touched INT DEFAULT 0,        -- Planos distintos editados
  hierarchy_nodes_touched INT DEFAULT 0, -- Nós da hierarquia editados
  products_touched INT DEFAULT 0,     -- SKUs distintos editados
  avg_session_duration_min NUMERIC(8,2) DEFAULT 0, -- Duração média da sessão (minutos)
  first_action_at TIMESTAMPTZ,
  last_action_at TIMESTAMPTZ,
  computed_at TIMESTAMPTZ DEFAULT now(),
  UNIQUE(org_id, user_id, period_start, period_type)
);
CREATE INDEX idx_user_activity_org ON user_activity_summary(org_id, period_start DESC);
CREATE INDEX idx_user_activity_user ON user_activity_summary(user_id, period_start DESC);
```

### 16.4 Indicadores de Engajamento e Desempenho

Estes indicadores são calculados a partir das tabelas `audit_log` e `user_activity_summary` e apresentados no **Painel Admin** e no **Dashboard de KPIs**.

#### Indicadores de Atividade Individual

| Indicador | Descrição | Fórmula / Query Base |
|---|---|---|
| **Total de Edições** | Número total de edições de demanda por usuário | `COUNT(*)` de audit_log WHERE action='update' e entity_type='demand_data' |
| **Cobertura de Preenchimento** | % de células (SKU×Hierarquia×Período) que o usuário preencheu vs. total sob sua responsabilidade | `cells_edited / total_cells_assigned × 100` |
| **Taxa de Contribuição** | Participação do usuário no total de edições da organização no período | `edits_user / edits_total_org × 100` |
| **Frequência de Acesso** | Número de dias com pelo menos 1 login no período | `COUNT(DISTINCT DATE(created_at))` de audit_log WHERE action='login' |
| **Recência** | Dias desde a última atividade do usuário | `NOW() - MAX(last_action_at)` |
| **Volume Médio por Sessão** | Quantidade de edições por sessão de acesso | `total_edits / total_logins` |

#### Indicadores de Equipe / Organização

| Indicador | Descrição | Uso |
|---|---|---|
| **Top Contributors** | Ranking dos usuários que mais contribuíram com edições no período | Reconhecimento e gestão |
| **Usuários Inativos** | Usuários que não acessaram a plataforma nos últimos N dias | Gestão de engajamento |
| **Heatmap de Atividade** | Mapa de calor por dia da semana × hora do dia | Identificar padrões de uso |
| **Evolução por Rodada** | Comparação do nº de edições entre rodadas (crescendo, caindo?) | Avaliar adesão ao processo |
| **Fill Rate por Usuário** | % de preenchimento das células atribuídas ao usuário | Identificar gargalos |
| **Tempo de Resposta** | Tempo entre abertura da etapa e primeira edição do usuário | Avaliar proatividade |

#### Indicadores de Qualidade

| Indicador | Descrição | Uso |
|---|---|---|
| **Taxa de Revisão** | % de células editadas mais de uma vez (indicando refinamento) | Avaliar profundidade da análise |
| **Variabilidade de Edição** | Desvio padrão das alterações de volume (mudanças muito drásticas?) | Detectar outliers |
| **Importações vs. Edições Manuais** | Proporção de dados importados via CSV vs. editados manualmente | Entender fluxo de trabalho |

### 16.5 Visualização no Admin

```
┌─────────────────────────────────────────────────────────────────┐
│                  PAINEL DE ENGAJAMENTO (ADMIN)                  │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐          │
│  │ Usuários     │  │ Edições      │  │ Fill Rate    │          │
│  │ Ativos: 12   │  │ Hoje: 347    │  │ Geral: 78%   │          │
│  │ Total: 15    │  │ Semana: 2.1k │  │ Meta: 95%    │          │
│  └──────────────┘  └──────────────┘  └──────────────┘          │
│                                                                 │
│  📊 Top Contributors (Rodada Atual)                             │
│  ┌─────┬──────────────────┬────────┬────────┬────────┐         │
│  │ #   │ Usuário          │ Edições│ Cobert.│ Freq.  │         │
│  ├─────┼──────────────────┼────────┼────────┼────────┤         │
│  │ 1   │ Maria Santos     │ 423    │ 92%    │ 18/20d │         │
│  │ 2   │ João Silva       │ 312    │ 85%    │ 15/20d │         │
│  │ 3   │ Ana Costa        │ 198    │ 71%    │ 12/20d │         │
│  └─────┴──────────────────┴────────┴────────┴────────┘         │
│                                                                 │
│  🔥 Heatmap de Atividade (últimos 30 dias)                     │
│  ┌────┬────┬────┬────┬────┬────┬────┐                          │
│  │ Seg│ Ter│ Qua│ Qui│ Sex│ Sab│ Dom│                          │
│  │ ██ │ ███│ ██ │ ███│ █  │ ░  │ ░  │  ← intensidade          │
│  └────┴────┴────┴────┴────┴────┴────┘                          │
│                                                                 │
│  ⚠️  Usuários Inativos (> 7 dias)                              │
│  → Carlos Ferreira (último acesso: 12 dias atrás)              │
│  → Pedro Lima (último acesso: 9 dias atrás)                    │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 17. Gerador de Dados de Exemplo (Seeder)

Como não existem dados reais para testes iniciais, o DemandHub incluirá um **sistema de seed** que gera dados realistas para desenvolvimento e demonstrações.

### 17.1 Estratégia de Geração

O seeder cria um ecossistema completo e coerente, respeitando as relações entre as tabelas:

```
Ordem de execução do seeder:

1. organizations (1-2 organizações de exemplo)
2. hierarchy_levels (3 níveis para Org A, 5 níveis para Org B)
3. hierarchy_nodes (árvore completa com nós realistas)
4. products (50-100 SKUs com categorias variadas)
5. plans (Histórico, Orçamento, Forecast 1, Forecast 2)
6. rounds (2-3 rodadas com etapas configuradas)
7. stages (2-3 etapas por rodada)
8. price_tables (1-2 tabelas de preços)
9. price_entries (preços por produto)
10. demand_data (dados volumétricos com padrões realistas)
11. roles + role_permissions (roles padrão)
12. user_assignments (atribuições de exemplo)
13. mapping_rules (2-3 mapeamentos de exemplo)
14. kpi_definitions (KPIs padrão do sistema)
15. kpi_user_preferences (configurações de dashboard)
```

### 17.2 Padrões Realistas nos Dados

```typescript
// Os dados de demanda devem incluir padrões reais:

const seedPatterns = {
  // 1. Sazonalidade — picos em meses específicos
  seasonality: {
    // Ex: produtos alimentícios com pico em Dezembro
    monthly_factors: [0.8, 0.7, 0.9, 0.9, 1.0, 1.0, 1.0, 1.1, 1.1, 1.2, 1.3, 1.5],
  },

  // 2. Tendência — crescimento ou declínio ao longo dos meses
  trend: {
    growth_rate: 0.02, // 2% de crescimento ao mês
  },

  // 3. Ruído — variação aleatória realista
  noise: {
    std_deviation: 0.1, // ±10% de variação aleatória
  },

  // 4. Hierarquia — distribuição desigual entre nós
  hierarchy_distribution: {
    // "Vendedor A" tem 40% do volume, "B" tem 35%, "C" tem 25%
    weights: [0.4, 0.35, 0.25],
  },

  // 5. Forecast vs Histórico — introduzir bias realista
  forecast_bias: {
    optimistic_bias: 0.05, // Forecast tende a ser 5% maior que o realizado
  },
};
```

### 17.3 Dados de Exemplo

| Entidade | Quantidade | Exemplos |
|---|---|---|
| **Organizações** | 2 | "DemoFood Ltda" (3 níveis), "TechParts Inc" (5 níveis) |
| **Hierarquia (Org A)** | 3 níveis, ~20 nós | Regional (3) → Gerente (6) → Vendedor (12) |
| **Hierarquia (Org B)** | 5 níveis, ~50 nós | VP (2) → Dir (4) → Reg (8) → Ger (16) → Vend (20) |
| **Produtos** | 80 SKUs | Alimentos, bebidas, limpeza — com categorias |
| **Planos** | 4 | Histórico 2024, Orçamento 2025, FC1, FC2 |
| **Demanda** | ~50.000 registros | 12-18 meses × 80 SKUs × 12-20 vendedores × 4 planos |
| **Preços** | 80 entradas | R$ 5,00 a R$ 500,00 por SKU |
| **Rodadas** | 3 | Jan/25, Fev/25, Mar/25 |
| **Usuários** | 5-8 | Admin, Manager, 3-5 Planners, 1 Viewer |

### 17.4 Execução

```bash
# O seeder será executável via script npm
npm run seed               # Seed completo (ambas organizações)
npm run seed:org-a          # Apenas Org A (3 níveis, mais simples)
npm run seed:org-b          # Apenas Org B (5 níveis, mais complexa)
npm run seed:clean          # Remove todos os dados de seed

# Implementação sugerida:
# src/scripts/seed/
# ├── index.ts              (orquestrador)
# ├── organizations.ts      (seed de orgs)
# ├── hierarchy.ts          (seed de hierarquia)
# ├── products.ts           (seed de produtos)
# ├── plans.ts              (seed de planos)
# ├── demand.ts             (seed de demanda com padrões)
# ├── prices.ts             (seed de preços)
# ├── rounds.ts             (seed de rodadas e etapas)
# ├── users.ts              (seed de usuários e permissões)
# └── utils.ts              (geradores de dados, ruído, sazonalidade)
```

---

## 18. Referências Visuais (StockHub)

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
