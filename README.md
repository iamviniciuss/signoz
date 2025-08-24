# SigNoz Observability Setup

Este projeto fornece uma configuração completa de observabilidade usando SigNoz, uma plataforma de monitoramento open-source que combina métricas, traces e logs em uma única aplicação.

## 📋 Sumário

- [O que é Observabilidade?](#o-que-é-observabilidade)
- [Arquitetura do Sistema](#arquitetura-do-sistema)
- [Pré-requisitos](#pré-requisitos)
- [Instalação e Configuração](#instalação-e-configuração)
- [Monitoramento Configurado](#monitoramento-configurado)
- [Como Usar](#como-usar)
- [Dashboards](#dashboards)
- [Troubleshooting](#troubleshooting)
- [Conceitos Importantes](#conceitos-importantes)

## 🔍 O que é Observabilidade?

Observabilidade é a capacidade de entender o estado interno de um sistema através de seus dados externos. É composta por três pilares:

- **Métricas**: Dados numéricos agregados ao longo do tempo (CPU, memória, latência)
- **Logs**: Registros de eventos que acontecem no sistema
- **Traces**: Rastreamento de requisições através de múltiplos serviços

### Por que usar SigNoz?

- **Tudo em um só lugar**: Métricas, logs e traces em uma única interface
- **Open Source**: Sem vendor lock-in, código aberto
- **Compatível com OpenTelemetry**: Padrão da indústria para observabilidade
- **Interface moderna**: Dashboard intuitivo e poderoso

## 🏗️ Arquitetura do Sistema

```
┌─────────────────┐    ┌──────────────────┐    ┌─────────────────┐
│   Aplicações    │───▶│ OpenTelemetry    │───▶│   ClickHouse    │
│   (sua app)     │    │   Collector      │    │   (Database)    │
└─────────────────┘    └──────────────────┘    └─────────────────┘
                                │                        ▲
┌─────────────────┐             │                        │
│   cAdvisor      │─────────────┘                        │
│ (containers)    │                                      │
└─────────────────┘                                      │
                                                         │
┌─────────────────┐    ┌──────────────────┐             │
│   PostgreSQL    │───▶│     SigNoz       │─────────────┘
│  (database)     │    │      (UI)        │
└─────────────────┘    └──────────────────┘
```

### Componentes:

1. **SigNoz**: Interface web principal (porta 8080)
2. **ClickHouse**: Banco de dados para armazenar métricas, traces e logs
3. **ZooKeeper**: Coordenação para clustering do ClickHouse
4. **OpenTelemetry Collector**: Recebe e processa dados de telemetria
5. **cAdvisor**: Monitor de containers Docker
6. **Schema Migrator**: Gerencia migrações do banco de dados

## 📋 Pré-requisitos

- **Docker** e **Docker Compose** instalados
- **Mínimo 4GB RAM** disponível (ClickHouse é intensivo em memória)
- **PostgreSQL** rodando localmente (se quiser monitorar banco)
- **Conhecimento básico** de Docker e containers

### Verificando Pré-requisitos

```bash
# Verificar Docker
docker --version
docker-compose --version

# Verificar recursos disponíveis
docker system df
free -h  # Linux/macOS
```

## 🚀 Instalação e Configuração

### 1. Clone e Inicie os Serviços

```bash
# Clone o repositório (se aplicável)
git clone <seu-repositorio>
cd signoz-setup

# Inicie todos os serviços
docker-compose up -d

# Verifique se todos os serviços estão rodando
docker-compose ps
```

### 2. Aguarde a Inicialização

Os serviços têm uma ordem de inicialização:
1. ZooKeeper (coordenação)
2. ClickHouse (database)
3. Migrações de schema
4. SigNoz (interface)
5. OpenTelemetry Collector

Aguarde alguns minutos para todos ficarem "healthy".

### 3. Acesse a Interface

- **SigNoz UI**: http://localhost:8080
- **cAdvisor**: http://localhost:8081 (monitoramento de containers)

## 📊 Monitoramento Configurado

Este setup já vem com monitoramento pré-configurado para:

### 🐳 Containers Docker
- **CPU**: Uso por container
- **Memória**: Consumo e limites
- **Rede**: Tráfego de entrada/saída
- **Disco**: I/O de leitura/escrita
- **Estado**: Containers em execução/parados

### 🗄️ PostgreSQL
- **Conexões**: Número de conexões ativas
- **Operações**: INSERT/UPDATE/DELETE/SELECT
- **Performance**: Tempo de resposta, locks
- **Recursos**: Uso de CPU/memória do PostgreSQL

### 🔧 OpenTelemetry Collector
- **Métricas internas**: Health do collector
- **Throughput**: Volume de dados processados

## 🎛️ Como Usar

### Comandos Básicos

```bash
# Iniciar todos os serviços
docker-compose up -d

# Parar todos os serviços
docker-compose down

# Ver logs de um serviço específico
docker-compose logs -f signoz-2
docker-compose logs -f otel-collector

# Reiniciar serviço após mudança de config
docker-compose restart otel-collector

# Ver status de todos os serviços
docker-compose ps

# Ver recursos sendo utilizados
docker stats
```

### Verificando Health dos Serviços

```bash
# SigNoz
curl http://localhost:8080/api/v1/health

# ClickHouse
docker-compose exec clickhouse clickhouse-client --query "SELECT 1"

# OpenTelemetry Collector
curl http://localhost:13133/
```

## 📈 Dashboards

### Dashboard de Containers Docker

Um dashboard pré-configurado está disponível em `dashboards/docker-containers-dashboard.json`.

**Para importar no SigNoz:**

1. Acesse http://localhost:8080
2. Vá para "Dashboards"
3. Clique em "New Dashboard"
4. Importe o JSON do arquivo
5. Configure as variáveis se necessário

**Painéis inclusos:**
- CPU Usage por container
- Memory Usage por container  
- Network I/O por container
- Disk I/O por container
- Container States (running/stopped)
- Top Containers (maiores consumidores)

### Criando Dashboards Personalizados

1. **Explore as métricas** em "Metrics" → "Query Builder"
2. **Métricas disponíveis**:
   - `container_*` (cAdvisor - containers)
   - `postgresql_*` (PostgreSQL)
   - `otelcol_*` (OpenTelemetry Collector)
3. **Crie visualizações** com diferentes tipos de gráficos
4. **Salve e organize** em dashboards temáticos

## 🔧 Troubleshooting

### Problemas Comuns

#### ClickHouse não inicia
```bash
# Verificar logs
docker-compose logs clickhouse

# Problemas comuns:
# - Memória insuficiente (precisa ~2GB)
# - Permissões de volume
# - Conflito de porta
```

#### OpenTelemetry Collector com erros
```bash
# Ver configuração atual
docker-compose exec otel-collector cat /etc/otel-collector-config.yaml

# Verificar conectividade com ClickHouse
docker-compose exec otel-collector wget --spider -q clickhouse:9000
```

#### PostgreSQL não conecta
```bash
# Verificar se PostgreSQL local está rodando
docker ps | grep postgres

# Testar conectividade
docker-compose exec otel-collector telnet host.docker.internal 5432
```

### Comandos Úteis para Debug

```bash
# Ver todos os containers
docker ps -a

# Verificar recursos do sistema
docker system df
docker system prune  # Limpar recursos não utilizados

# Logs detalhados
docker-compose logs --tail=50 -f [service-name]

# Executar comandos dentro dos containers
docker-compose exec clickhouse clickhouse-client
docker-compose exec signoz-2 /bin/sh
```

### Reset Completo

Se algo der errado, você pode resetar tudo:

```bash
# Parar e remover containers
docker-compose down

# Remover volumes (CUIDADO: apaga todos os dados!)
docker-compose down -v

# Remover imagens (força re-download)
docker-compose down --rmi all

# Recriar tudo
docker-compose up -d --force-recreate
```

## 📚 Conceitos Importantes

### OpenTelemetry
- **Padrão aberto** para coleta de telemetria
- **Vendor neutral**: Funciona com qualquer backend
- **Instrumentação automática**: Para muitas linguagens/frameworks
- **Collectors**: Recebem, processam e exportam dados

### ClickHouse
- **Database colunar**: Otimizado para analytics
- **Alta performance**: Para consultas agregadas
- **Compressão eficiente**: Armazena muito dados
- **Horizontal scaling**: Pode escalar com sharding

### Métricas vs Logs vs Traces

| Aspecto | Métricas | Logs | Traces |
|---------|----------|------|--------|
| **Tipo** | Numérico/Agregado | Textual/Eventos | Requisições distribuídas |
| **Volume** | Baixo | Médio/Alto | Médio |
| **Uso** | Alertas, dashboards | Debug, auditoria | Performance, debugging |
| **Exemplo** | CPU: 80% | "Error: file not found" | Request ID através de serviços |

### Quando usar cada um:

- **Métricas**: Monitoring contínuo, alertas, SLIs
- **Logs**: Debugging, auditoria, análise de erros
- **Traces**: Debugging de performance, microserviços

## 🔒 Segurança

### Dados Sensíveis
- Senhas estão hardcoded no compose (OK para desenvolvimento)
- **Produção**: Use Docker secrets ou variáveis de ambiente
- **Rede**: Services expostos apenas necessariamente

### Monitoramento Seguro
- PostgreSQL: Usuário com permissões mínimas
- ClickHouse: Configure usuários específicos
- OpenTelemetry: Filtre dados sensíveis nos processors

## 🚀 Próximos Passos

1. **Instrumentar sua aplicação** com OpenTelemetry
2. **Criar alertas** baseados nas métricas coletadas  
3. **Configurar retention** para otimizar armazenamento
4. **Adicionar mais receivers** (Redis, Nginx, etc.)
5. **Implementar service discovery** para ambientes dinâmicos

### Links Úteis

- [SigNoz Documentation](https://signoz.io/docs/)
- [OpenTelemetry Docs](https://opentelemetry.io/docs/)
- [ClickHouse Docs](https://clickhouse.com/docs/)
- [Docker Compose Reference](https://docs.docker.com/compose/)

---

💡 **Dica**: Mantenha este README atualizado conforme você adicionar novos receivers, dashboards ou configurações!