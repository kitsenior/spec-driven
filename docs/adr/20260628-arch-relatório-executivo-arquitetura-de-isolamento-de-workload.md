# RELATÓRIO EXECUTIVO: ARQUITETURA DE ISOLAMENTO DE WORKLOAD
## Sistema Flask + PostgreSQL + Redis | Consolidação de Diagnóstico e Roadmap de Ação

---

## SUMÁRIO EXECUTIVO

O sistema atual enfrenta um **gargalo crítico de contenção de conexões PostgreSQL** que causa indisponibilidade de transações financeiras durante a geração de relatórios. O diagnóstico identificou um acoplamento estrutural entre processamento transacional (OLTP) e analítico (OLAP) na mesma aplicação monolítica.

**Problema:** Relatórios travam o banco por até 3 minutos, bloqueando 200 req/s de transações críticas.

**Causa Raiz:** Pool de conexões PostgreSQL compartilhado sem isolamento de workload.

**Solução Recomendada:** Implementar arquitetura de isolamento com read replica + fila assíncrona.

**Impacto Esperado:** Recuperar 99.9% de disponibilidade de transações com investimento de $200-450/mês e 80 horas de desenvolvimento.

---

## 1. DIAGNÓSTICO TÉCNICO

### 1.1 Classificação Arquitetural

**Estilo Atual:** Monolith Síncrono com OLTP/OLAP Acoplados

| Aspecto | Evidência |
|---------|-----------|
| **Estrutura** | Stack único: Flask + PostgreSQL + Redis (5 devs, 1 base de código) |
| **Processamento** | Síncrono: relatórios executam na mesma thread que transações HTTP |
| **Acoplamento** | Crítico: OLTP e OLAP compartilham pool de conexões PostgreSQL |
| **Escala** | 10k usuários ativos, 200 req/s no pico, 3 min de travamento |

**Conclusão:** Este é um monolith com problema de isolamento de workload, não de arquitetura de serviços. Microservices seria solução prematura.

---

### 1.2 Gargalos Estruturais Críticos

#### **Gargalo #1: Contenção de Conexões PostgreSQL** ⚠️ CRÍTICO

**Sintoma:** Relatórios travam transações por 3 minutos

**Causa Raiz:**
- Pool de conexões PostgreSQL compartilhado entre OLTP e OLAP
- Queries de relatório (full table scans, complexas) ocupam todas as conexões
- Transações em tempo real (200 req/s) aguardam liberação
- Sem read replicas ou isolamento de conexões

**Impacto Quantitativo:**
- 200 req/s = ~200 conexões simultâneas necessárias
- Pool típico Flask: 5-20 conexões
- Relatório executando = 100% das conexões bloqueadas por 180 segundos
- Resultado: timeout em transações críticas, taxa de erro ~5%

**Impacto no Negócio:**
- Indisponibilidade de transações financeiras
- Degradação de experiência de usuário
- Risco de perda de dados ou inconsistência

---

#### **Gargalo #2: Falta de Isolamento de Workload** ⚠️ CRÍTICO

**Sintoma:** Não há separação entre processamento transacional e analítico

**Causa Raiz:**
- Mesma aplicação Flask processa requisições de usuários E gera relatórios
- Sem fila de tarefas assíncrona (nenhuma menção a Celery, RabbitMQ, SQS)
- Sem workers dedicados para processamento pesado
- Redis usado apenas para sessão, não para fila de tarefas
- CPU e I/O de relatórios competem com requisições HTTP

**Impacto Técnico:**
- Impossível escalar relatórios sem afetar transações
- Impossível aplicar SLA diferenciado (transações: <100ms; relatórios: 5min)
- Falhas em relatórios podem afetar disponibilidade da aplicação
- Sem retry automático ou circuit breaker para jobs assíncronos

**Limitador de Escala:** Crescimento de 200 req/s para 500 req/s exigirá reescrita

---

#### **Gargalo #3: Cache Insuficiente e Estratégia Reativa** ⚠️ MODERADO

**Sintoma:** Redis usado apenas para sessão, não para dados de negócio

**Causa Raiz:**
- Redis não está na camada de dados (apenas sessão)
- Relatórios consultam PostgreSQL diretamente sem cache
- Sem invalidação inteligente de cache
- Sem pré-computação de agregações

**Impacto:**
- Cada execução de relatório = full scan no PostgreSQL
- Relatórios repetidos (ex: dashboard diário) executam 100% do trabalho
- Potencial para reduzir carga em 60-80% com cache

---

### 1.3 Acoplamentos Problemáticos

#### **Acoplamento #1: OLTP/OLAP no Mesmo Banco**

```
Fluxo Problemático Atual:
┌─────────────────────────────────────────────────────┐
│ HTTP Request: Transação Financeira                  │
│ (Precisa de <100ms latência)                        │
└────────────────┬────────────────────────────────────┘
                 │
                 ▼
        ┌────────────────────┐
        │ Pool PostgreSQL    │
        │ (5-20 conexões)    │
        └────────────────────┘
                 ▲
                 │
        ┌────────┴──────────────────────┐
        │                               │
        ▼                               ▼
   [OLTP Transação]          [OLAP Relatório Query]
   (Rápida, 10ms)            (Lenta, 180s)
   
   RESULTADO: Transação aguarda 180s → Timeout → Erro
```

**Manifestação:**
- Mudança em índices para relatórios afeta latência de transações
- Vacuum/Analyze para OLAP bloqueia OLTP
- Não há isolamento de recursos (CPU, I/O, conexões)

**Limite de Evolução:** Impossível otimizar relatórios sem risco a transações

---

#### **Acoplamento #2: Processamento Síncrono na Aplicação Web**

**Descrição:** Lógica de relatório executa no mesmo processo que serve requisições HTTP

**Manifestação:**
- Relatório inicia → Flask worker bloqueado por 3 minutos
- Outras requisições aguardam worker disponível
- Sem fila = sem priorização

**Limite de Evolução:** Impossível ter SLA diferenciado por tipo de requisição

---

#### **Acoplamento #3: Modelo de Dados Único para OLTP e OLAP**

**Descrição:** Mesmo schema PostgreSQL serve transações e relatórios

**Manifestação:**
- Índices otimizados para OLTP (B-tree, pequenos) prejudicam OLAP (full scans)
- Normalização para transações prejudica performance de agregações
- Sem tabelas desnormalizadas ou data warehouse

**Limite de Evolução:** Impossível otimizar schema sem trade-offs

---

## 2. PADRÕES RECOMENDADOS

### 2.1 Arquitetura Alvo: Isolamento de Workload

```
┌─────────────────────────────────────────────────────────────┐
│                    Flask Application                         │
│  (Transações + Endpoints de Relatório)                      │
│  - Requisições HTTP retornam imediatamente                  │
│  - Relatórios enfileirados, não bloqueantes                 │
└──────────────┬──────────────────────────────────────────────┘
               │
        ┌──────┴──────┐
        │             │
        ▼             ▼
   [OLTP]        [OLAP Queue]
        │             │
        ▼             ▼
┌──────────────┐  ┌──────────────────────┐
│ PostgreSQL   │  │ Redis (Message Broker)
│ Primary      │  │ + Cache              │
│ (Transações) │  │ (Celery Broker)      │
└──────────────┘  └──────────────────────┘
        │                │
        │                ▼
        │         ┌──────────────────┐
        │         │ Celery Workers   │
        │         │ (Relatórios)     │
        │         │ (Processo Separado)
        │         └──────────────────┘
        │                │
        ▼                ▼
┌──────────────┐  ┌──────────────┐
│ PostgreSQL   │  │ PostgreSQL    │
│ Primary      │  │ Read Replica  │
│ (Escrita)    │  │ (Leitura)     │
│ (OLTP)       │  │ (OLAP)        │
└──────────────┘  └──────────────┘
```

**Princípios:**
1. **Isolamento de Recursos:** OLTP e OLAP em conexões/processos separados
2. **Processamento Assíncrono:** Relatórios não bloqueiam requisições HTTP
3. **Escalabilidade Independente:** Escalar workers sem afetar aplicação web
4. **Consistência Eventual:** Read replica pode ter lag de 100-500ms (aceitável para relatórios)

---

### 2.2 Padrões de Implementação

#### **Padrão 1: Read Replica para OLAP**

```python
# Configuração de Conexões
DATABASES = {
    'default': {  # OLTP - Escrita
        'ENGINE': 'django.db.backends.postgresql',
        'NAME': 'production',
        'HOST': 'db-primary.rds.amazonaws.com',
        'PORT': 5432,
    },
    'analytics': {  # OLAP - Leitura
        'ENGINE': 'django.db.backends.postgresql',
        'NAME': 'production',
        'HOST': 'db-replica.rds.amazonaws.com',
        'PORT': 5432,
    }
}

# Uso em Relatórios
def gerar_relatorio_vendas():
    # Lê da replica, não afeta transações
    vendas = Venda.objects.using('analytics').filter(
        data__gte=datetime.now() - timedelta(days=30)
    ).values('produto').annotate(total=Sum('valor'))
    return vendas
```

**Benefícios:**
- Elimina contenção de conexões
- Transações não são afetadas por queries de relatório
- Lag de 100-500ms é aceitável para relatórios

---

#### **Padrão 2: Fila Assíncrona com Celery**

```python
# tasks.py - Definição de Tarefas
from celery import shared_task
import logging

@shared_task(bind=True, max_retries=3)
def gerar_relatorio_financeiro(self, relatorio_id, usuario_id):
    """
    Gera relatório financeiro de forma assíncrona
    """
    try:
        relatorio = Relatorio.objects.get(id=relatorio_id)
        relatorio.status = 'processando'
        relatorio.save()
        
        # Lê da read replica
        dados = Transacao.objects.using('analytics').filter(
            data__gte=relatorio.data_inicio,
            data__lte=relatorio.data_fim
        ).values('categoria').annotate(total=Sum('valor'))
        
        # Processa dados
        resultado = processar_dados_relatorio(dados)
        
        # Salva resultado
        relatorio.resultado = resultado
        relatorio.status = 'concluido'
        relatorio.data_conclusao = timezone.now()
        relatorio.save()
        
        # Notifica usuário
        notificar_usuario(usuario_id, f"Relatório {relatorio_id} concluído")
        
    except Exception as exc:
        # Retry automático com backoff exponencial
        raise self.retry(exc=exc, countdown=60 * (2 ** self.request.retries))

# views.py - Endpoint HTTP
@app.route('/api/relatorios', methods=['POST'])
def solicitar_relatorio():
    """
    Endpoint que enfileira relatório sem bloquear
    """
    dados = request.json
    
    # Cria registro de relatório
    relatorio = Relatorio.objects.create(
        usuario_id=current_user.id,
        tipo=dados['tipo'],
        data_inicio=dados['data_inicio'],
        data_fim=dados['data_fim'],
        status='enfileirado'
    )
    
    # Enfileira tarefa (retorna imediatamente)
    task = gerar_relatorio_financeiro.delay(
        relatorio_id=relatorio.id,
        usuario_id=current_user.id
    )
    
    # Retorna job ID para cliente
    return {
        'relatorio_id': relatorio.id,
        'job_id': task.id,
        'status': 'enfileirado'
    }, 202

# Polling de Status
@app.route('/api/relatorios/<relatorio_id>/status', methods=['GET'])
def status_relatorio(relatorio_id):
    """
    Cliente consulta status do relatório
    """
    relatorio = Relatorio.objects.get(id=relatorio_id)
    return {
        'status': relatorio.status,
        'data_conclusao': relatorio.data_conclusao,
        'resultado': relatorio.resultado if relatorio.status == 'concluido' else None
    }
```

**Benefícios:**
- Requisições HTTP retornam imediatamente (202 Accepted)
- Relatórios processam em background
- Retry automático em caso de falha
- Escalabilidade: adicionar workers sem afetar aplicação web

---

#### **Padrão 3: Cache de Resultados com Invalidação**

```python
# cache.py - Estratégia de Cache
from django.core.cache import cache
import hashlib

def gerar_chave_cache_relatorio(tipo, data_inicio, data_fim):
    """
    Gera chave de cache determinística
    """
    chave_base = f"relatorio:{tipo}:{data_inicio}:{data_fim}"
    return hashlib.md5(chave_base.encode()).hexdigest()

@shared_task
def gerar_relatorio_financeiro_com_cache(self, relatorio_id, usuario_id):
    """
    Gera relatório com cache
    """
    relatorio = Relatorio.objects.get(id=relatorio_id)
    
    # Verifica cache
    chave_cache = gerar_chave_cache_relatorio(
        relatorio.tipo,
        relatorio.data_inicio,
        relatorio.data_fim
    )
    
    resultado_cache = cache.get(chave_cache)
    if resultado_cache:
        relatorio.resultado = resultado_cache
        relatorio.status = 'concluido'
        relatorio.save()
        return resultado_cache
    
    # Se não está em cache, processa
    dados = Transacao.objects.using('analytics').filter(
        data__gte=relatorio.data_inicio,
        data__lte=relatorio.data_fim
    ).values('categoria').annotate(total=Sum('valor'))
    
    resultado = processar_dados_relatorio(dados)
    
    # Ar