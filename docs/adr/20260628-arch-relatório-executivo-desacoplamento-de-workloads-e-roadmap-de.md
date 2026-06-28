# RELATÓRIO EXECUTIVO: DESACOPLAMENTO DE WORKLOADS E ROADMAP DE AÇÃO
## Consolidação de Diagnóstico, Padrões e ADR

---

## SUMÁRIO EXECUTIVO

**Situação Atual**: Sistema monolith Flask + PostgreSQL com acoplamento crítico entre transações financeiras (OLTP) e relatórios (OLAP), causando indisponibilidade de até 3 minutos durante picos de processamento.

**Diagnóstico**: Não é problema de arquitetura geral, mas de **isolamento de workloads**. Transações críticas e relatórios competem pelos mesmos recursos (pool de conexões, I/O do banco).

**Recomendação**: Implementar desacoplamento temporal e de recursos através de **Read Replica + Job Queue**, mantendo o monolith como base. **NÃO é necessária migração para microservices nesta fase.**

**Investimento**: Medium (estimado em 4-6 sprints, ~$80-120k em infraestrutura + desenvolvimento)

**ROI**: Eliminação de indisponibilidade crítica, melhoria de experiência de 10k usuários, preparação para crescimento futuro.

---

## 1. DIAGNÓSTICO CONSOLIDADO

### 1.1 Estilo Arquitetural Real

**Classificação**: Monolith Síncrono com Acoplamento OLTP↔OLAP

| Característica | Diagnóstico | Evidência |
|---|---|---|
| **Processamento** | 100% síncrono | Sem menção a filas, workers ou processamento assíncrono |
| **Workloads** | Misturados no mesmo banco | Relatórios travam transações por 3 minutos |
| **Isolamento de Recursos** | Nenhum | Pool de conexões único, sem priorização |
| **Cache** | Tático (sessão apenas) | Redis subutilizado, sem cache de dados ou filas |
| **Escalabilidade** | Acoplada | Impossível escalar relatórios sem afetar transações |

**Conclusão**: O sistema é funcional, mas estruturalmente frágil. Não é um "monolith bem-organizado", é um "monolith com contenção de recursos crítica".

---

### 1.2 Gargalos Críticos Identificados

#### **Gargalo #1: Processamento de Relatórios Síncrono (CRÍTICO)**

| Aspecto | Descrição |
|--------|-----------|
| **Sintoma** | Relatórios financeiros travam o banco por até 3 minutos |
| **Raiz** | Queries de longa duração (full table scans, agregações) executam na mesma instância que transações críticas |
| **Impacto** | Indisponibilidade de transações em tempo real durante picos de relatório |
| **Usuários Afetados** | 10.000 usuários ativos |
| **Severidade** | CRÍTICA - afeta operações de negócio |

**Manifestação Técnica**:
```
Requisição de Relatório
  ↓ (síncrono)
PostgreSQL Primary (lock contention)
  ↓ (3 minutos de processamento)
Transações bloqueadas (timeout)
  ↓
Degradação de experiência / Perda de transações
```

#### **Gargalo #2: Ausência de Desacoplamento Temporal (ALTO)**

| Aspecto | Descrição |
|--------|-----------|
| **Sintoma** | Operações de longa duração bloqueiam requisições HTTP |
| **Raiz** | Sem fila de processamento (Celery, RQ, etc.) |
| **Impacto** | Impossibilidade de escalar operações pesadas independentemente |
| **Limite Prático** | 200 req/s é o teto, sem elasticidade |
| **Severidade** | ALTA - limita crescimento |

**Manifestação Técnica**:
```
HTTP Request (Relatório)
  ↓ (síncrono, bloqueante)
Processamento no Flask (3 minutos)
  ↓
HTTP Response (após 3 minutos)
  ↓
Usuário aguarda 3 minutos (experiência ruim)
```

#### **Gargalo #3: Pool de Conexões Compartilhado (ALTO)**

| Aspecto | Descrição |
|--------|-----------|
| **Sintoma** | Transações críticas competem com relatórios pelo mesmo pool |
| **Raiz** | Configuração padrão de Flask + SQLAlchemy (um pool para toda aplicação) |
| **Impacto** | Impossibilidade de priorizar operações críticas |
| **Severidade** | ALTA - causa degradação em cascata |

**Manifestação Técnica**:
```
Pool de Conexões (20 conexões)
  ├─ 15 conexões: Relatório (bloqueadas por 3 minutos)
  └─ 5 conexões: Transações críticas (timeout)
```

---

### 1.3 Acoplamentos Estruturais

#### **Acoplamento #1: Temporal (OLTP ↔ OLAP)**

**Descrição**: Transações de negócio e relatórios executam na mesma instância PostgreSQL.

**Limitações de Evolução**:
- ❌ Impossível otimizar índices para ambos os workloads
- ❌ Impossível usar read replicas sem refatorar queries
- ❌ Impossível implementar CQRS sem desacoplamento
- ❌ Impossível escalar relatórios sem escalar transações

**Impacto no Roadmap**: Bloqueia evolução para arquitetura event-driven ou CQRS.

---

#### **Acoplamento #2: Estrutural (Lógica ↔ Transporte HTTP)**

**Descrição**: Operações de longa duração executam no contexto síncrono de requisições HTTP.

**Limitações de Evolução**:
- ❌ Impossível implementar processamento assíncrono sem reescrever endpoints
- ❌ Impossível adicionar priorização de operações
- ❌ Impossível implementar retry automático
- ❌ Timeout HTTP limita duração máxima de operações

**Impacto no Roadmap**: Bloqueia implementação de padrões resilientes (circuit breaker, bulkhead).

---

#### **Acoplamento #3: De Dados (Sessão ↔ Cache ↔ Banco)**

**Descrição**: Redis é usado apenas para sessão, sem separação clara de responsabilidades.

**Limitações de Evolução**:
- ❌ Impossível usar Redis como job queue
- ❌ Impossível implementar cache-aside pattern para relatórios
- ❌ Sem estratégia clara de invalidação de cache

**Impacto no Roadmap**: Subutiliza infraestrutura existente.

---

## 2. PADRÕES RECOMENDADOS

### 2.1 Padrão #1: Separação de Workloads (Read Replica)

**Objetivo**: Isolar OLAP (relatórios) de OLTP (transações).

**Implementação**:
```
PostgreSQL Primary (OLTP)
    ↓ replicação streaming
PostgreSQL Read Replica (OLAP)
```

**Benefícios**:
- ✅ Relatórios não travam transações
- ✅ Possibilidade de otimizar índices independentemente
- ✅ Escalabilidade de leitura
- ✅ Preparação para event sourcing

**Custo**: +1 instância PostgreSQL (~$200-400/mês)

**Timeline**: 2 sprints

---

### 2.2 Padrão #2: Desacoplamento Temporal (Job Queue)

**Objetivo**: Desacoplar operações de longa duração do ciclo de requisição HTTP.

**Implementação**:
```
HTTP Request (Relatório)
    ↓
Endpoint retorna job_id imediatamente
    ↓
Celery Worker processa em background
    ↓
Cliente consulta status via /jobs/{job_id}
    ↓
Resultado armazenado em Redis/S3
```

**Benefícios**:
- ✅ Requisições retornam imediatamente
- ✅ Possibilidade de processar múltiplos relatórios em paralelo
- ✅ Retry automático com backoff exponencial
- ✅ Priorização de jobs
- ✅ Preparação para saga pattern

**Custo**: +Redis (já existe), +Celery workers (~$100-200/mês)

**Timeline**: 2-3 sprints

---

### 2.3 Padrão #3: Isolamento de Pool de Conexões

**Objetivo**: Priorizar transações críticas sobre operações pesadas.

**Implementação**:
```python
# Pool OLTP (prioridade alta)
oltp_engine = create_engine(
    'postgresql://primary...',
    pool_size=20,
    max_overflow=10
)

# Pool OLAP (prioridade baixa)
olap_engine = create_engine(
    'postgresql://replica...',
    pool_size=10,
    max_overflow=5
)
```

**Benefícios**:
- ✅ Transações críticas têm recursos garantidos
- ✅ Relatórios não afetam transações
- ✅ Possibilidade de implementar circuit breaker

**Custo**: Nenhum (apenas refatoração de código)

**Timeline**: 1 sprint

---

### 2.4 Padrão #4: Circuit Breaker para Operações Pesadas

**Objetivo**: Prevenir cascata de falhas durante picos de carga.

**Implementação**:
```python
@circuit_breaker(failure_threshold=5, timeout=60)
def generate_financial_report(filters):
    # Se falhar 5 vezes em 60s, circuit abre
    # Requisições subsequentes falham rápido
    pass
```

**Benefícios**:
- ✅ Falha rápida em vez de timeout
- ✅ Recuperação automática
- ✅ Proteção de cascata

**Custo**: Nenhum (biblioteca open-source)

**Timeline**: 1 sprint

---

### 2.5 Padrão #5: Observabilidade Distribuída

**Objetivo**: Rastrear requisições através de múltiplos componentes (API → Worker → Banco).

**Implementação**:
```
Jaeger / Datadog para distributed tracing
  ├─ Rastrear job_id através de toda pipeline
  ├─ Alertas para lag de replica > 30s
  ├─ Métricas de job queue (throughput, latency, errors)
  └─ Dashboard de status de sistema
```

**Benefícios**:
- ✅ Debugging facilitado
- ✅ Alertas proativas
- ✅ Visibilidade operacional

**Custo**: ~$200-500/mês (Datadog) ou grátis (Jaeger self-hosted)

**Timeline**: 2 sprints

---

## 3. ARQUITETURA DECISÃO (ADR-001)

### 3.1 Decisão

**Implementar desacoplamento de workloads OLTP/OLAP através de Read Replica + Job Queue, mantendo monolith como base.**

**Não migrar para microservices nesta fase.**

---

### 3.2 Justificativa

#### Por que Read Replica + Job Queue?

| Critério | Solução | Justificativa |
|----------|---------|---------------|
| **Complexidade** | Baixa | Mudanças incrementais, sem big bang |
| **Timeline** | 4-6 sprints | Rápido time-to-value |
| **Custo** | Medium | ~$80-120k total |
| **Team** | 5 devs conseguem manter | Não requer expertise em microservices |
| **ROI** | Alto | Elimina gargalo crítico imediatamente |

#### Por que NÃO microservices?

| Razão | Impacto |
|-------|--------|
| **Team insuficiente** | 5 devs não conseguem manter 2+ serviços + orquestração |
| **Budget insuficiente** | Migração + Kubernetes + service mesh = $200k+ |
| **Problema não é arquitetura** | É isolamento de workloads, não separação de domínios |
| **Timeline longo** | 6+ meses, sem benefício imediato |
| **Risco alto** | Aumenta complexidade sem resolver problema |

**Conclusão**: Microservices são solução para escala de domínios, não para contenção de workloads. Desacoplamento dentro do monolith é mais pragmático.

---

### 3.3 Consequências Positivas

#### 1. **Eliminação do Gargalo Crítico** ⭐⭐⭐
- Relatórios não travam mais transações
- Transações críticas têm pool dedicado
- Possibilidade de escalar workloads independentemente

#### 2. **Melhoria de Experiência do Usuário** ⭐⭐⭐
- Requisições de relatório retornam imediatamente (job_id)
- Usuários não aguardam 3 minutos bloqueados
- Possibilidade de processar múltiplos relatórios em paralelo

#### 3. **Elasticidade de Processamento** ⭐⭐
- Adicionar workers Celery sem afetar API
- Escalar read replica independentemente
- Priorizar jobs críticos

#### 4. **Resiliência Melhorada** ⭐⭐
- Falha de relatório não afeta transações
- Circuit breaker previne cascata de falhas
- Retry automático com backoff exponencial

#### 5. **Observabilidade** ⭐⭐
- Tracking de status de jobs
- Métricas de latência de replicação
- Alertas para lag de replica > threshold

#### 6. **Compatibilidade com Team Pequeno** ⭐⭐⭐
- Não requer reescrita completa
- Mudanças incrementais
- 5 desenvolvedores conseguem manter

#### 7. **Preparação para Evolução Futura** ⭐⭐
- Desacoplamento temporal permite CQRS depois
- Read replica é base para event sourcing
- Job queue é base para saga pattern

---

### 3.4 Consequências Negativas e Mitigações

#### 1. **Complexidade Operacional Aumentada**

**Problema**: Necessário manter 2 instâncias PostgreSQL, monitorar replicação.

**Mitigação**:
- ✅ Alertas automáticos para lag > 30s
- ✅ Failover manual documentado
- ✅ Testes de recuperação mensais
- ✅ Runbook de troubleshooting

**Impacto**: +10% de overhead operacional

---

#### 2. **Consistência Eventual para Relatórios**

**Problema**: Relatórios podem mostrar dados até 5 segundos desatualizados.

**Mitigação**:
- ✅ Exibir timestamp de última sincronização
- ✅ Alertar se lag > 30s
- ✅ Documentar limitação para stakeholders
- ✅ Aceitável para relatórios financeiros (não são real-time)

**Impacto**: Negligenciável para caso de uso

---

#### 3. **Overhead de