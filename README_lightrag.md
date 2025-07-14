# 🐳 Setup PostgreSQL + AGE + LightRAG via Portainer

## 1. Criar Estrutura de Diretórios

```bash
# Criar pasta na raiz do servidor
mkdir -p /root/postgres_custom
cd /root/postgres_custom
```

## 2. Criar Dockerfile Customizado

```bash
# Criar arquivo Dockerfile
nano Dockerfile
```

**Cole este conteúdo no Dockerfile:**

```dockerfile
FROM apache/age:release_PG16_1.5.0

# Instalar pgvector
RUN apt-get update && \
    apt-get install -y git build-essential postgresql-server-dev-16 && \
    cd /tmp && \
    git clone --branch v0.8.0 https://github.com/pgvector/pgvector.git && \
    cd pgvector && \
    make clean && \
    make PG_CONFIG=/usr/bin/pg_config && \
    make install PG_CONFIG=/usr/bin/pg_config && \
    apt-get remove -y git build-essential postgresql-server-dev-16 && \
    apt-get autoremove -y && \
    apt-get clean && \
    rm -rf /var/lib/apt/lists/* /tmp/pgvector

# Configurar shared_preload_libraries
RUN echo "shared_preload_libraries = 'age,vector'" >> /usr/share/postgresql/postgresql.conf.sample

# Adicionar label para identificação
LABEL description="PostgreSQL 16 with Apache AGE and pgvector"
LABEL version="1.0"
```

## 3. Build da Imagem Docker

```bash
# Construir a imagem customizada
docker build -t seuusuario/pg16:v1.5.0 .
```

## 4. Deploy PostgreSQL via Portainer

### 4.1. Acessar Portainer
- Ir para **Portainer → Stacks → Add Stack**
- **Nome**: `postgres-age-stack`

### 4.2. Stack PostgreSQL (postgres_age.yml)


### 5.2. Conectar no Database
```bash
psql -U app_user -d lightrag
```

### 5.3. Executar Comandos SQL
```sql
-- Instalar extensões básicas
CREATE EXTENSION IF NOT EXISTS vector;
CREATE EXTENSION IF NOT EXISTS "uuid-ossp";
CREATE EXTENSION IF NOT EXISTS pg_trgm;

-- Instalar e configurar AGE
CREATE EXTENSION IF NOT EXISTS age;
LOAD 'age';
SET search_path = ag_catalog, "$user", public;

-- Criar grafo para LightRAG
SELECT create_graph('chunk_entity_relation');

-- Verificar se tudo foi criado
SELECT 'Extensão instalada: ' || extname || ' v' || extversion as status
FROM pg_extension 
WHERE extname IN ('vector', 'uuid-ossp', 'pg_trgm', 'age');

-- Verificar grafo AGE
SELECT * FROM ag_catalog.ag_graph;

-- Sair do psql
\q
```

## 6. Configurar Acesso Externo via Portainer

### 6.1. Baixar pg_hba.conf
- **Portainer → Volumes**
- **Localizar** volume `postgres-age-stack_postgres_data`
- **Clicar** em "Browse"
- **Navegar** para pasta raiz do volume
- **Localizar** arquivo `pg_hba.conf`
- **Clicar** no ícone de "Download" ao lado do arquivo

### 6.2. Editar arquivo pg_hba.conf
- **Abrir** arquivo baixado em editor de texto
- **Localizar** a linha:
```
host    all             all             127.0.0.1/32            md5
```
- **Alterar** para:
```
host    all             all             0.0.0.0/0                md5
```
- **Salvar** o arquivo

### 6.3. Subir arquivo modificado
- **Voltar** para o volume no Portainer
- **Clicar** em "Upload"
- **Selecionar** o arquivo pg_hba.conf editado
- **Confirmar** o upload (sobrescrever o arquivo existente)

### 6.4. Reiniciar Container
- **Portainer → Containers**
- **Localizar** container PostgreSQL
- **Clicar** em "Restart"
- **Aguardar** reinicialização (30 segundos)

## 7. Verificar Conectividade

### 7.1. Via Console do Container
- **Acessar console** do container PostgreSQL
- **Executar**:
```bash
psql -U app_user -d lightrag -c "SELECT 1;"
```

### 7.2. Via Host (se tiver psql instalado)
```bash
psql -h localhost -p 5432 -U app_user -d lightrag -c "SELECT 1;"
```

## 8. Deploy LightRAG via Portainer

### 8.1. Criar Nova Stack
- **Portainer → Stacks → Add Stack**
- **Nome**: `lightrag-stack`

### 8.2. Stack LightRAG (lightrag.yml)


### 8.3. Deploy da Stack LightRAG
- **Substituir** `SUA_CHAVE_OPENAI_AQUI` pela sua chave OpenAI real
   - **Substituir** `lightrag2.seudominio.com` pelo seu dominio real 
- **Clicar** em "Deploy the stack"
- **Aguardar** inicialização (2-3 minutos)

## 9. Verificação via Portainer

### 9.1. Verificar Status das Stacks
- **Portainer → Stacks**
- **Verificar** se ambas as stacks estão "running"
- **postgres-age-stack**: 1/1 services running
- **lightrag-stack**: 1/1 services running

### 9.2. Verificar Logs
- **Portainer → Containers**
- **Localizar** container LightRAG
- **Clicar** em "Logs"
- **Verificar** se não há erros de conexão com banco

### 9.3. Testar WebUI
- **Abrir navegador**: `http://seu-servidor:9621/webui`
- **Fazer login**: `admin` / `teste2025!`
- **Verificar** se interface carrega corretamente

## 10. Troubleshooting via Portainer

### 10.1. Container não inicia
- **Portainer → Containers**
- **Verificar logs** do container com problema
- **Verificar resources** (memória/CPU suficientes)

### 10.2. LightRAG não conecta no banco
- **Verificar** se nome do serviço está correto: `postgres-age-stack_postgres`
- **Verificar logs** do LightRAG para erros de conexão
- **Testar conectividade** via console entre containers

### 10.3. Erro de extensões AGE
- **Acessar console** do PostgreSQL
- **Verificar** se extensões estão instaladas:
```sql
SELECT extname FROM pg_extension;
```

## 11. Checklist Final

### ✅ PostgreSQL
- [ ] Stack `postgres-age-stack` running
- [ ] Container PostgreSQL acessível na porta 5432
- [ ] Extensões instaladas: vector, uuid-ossp, pg_trgm, age
- [ ] Grafo `chunk_entity_relation` criado
- [ ] pg_hba.conf configurado para acesso externo

### ✅ LightRAG
- [ ] Stack `lightrag-stack` running
- [ ] Container LightRAG acessível na porta 9621
- [ ] Conectividade com PostgreSQL funcionando
- [ ] WebUI acessível: `http://servidor:9621/webui`
- [ ] Login funcionando com credenciais configuradas

### ✅ Funcionalidades
- [ ] Upload de documentos funcionando
- [ ] Processamento de documentos sem erros
- [ ] Queries retornando respostas
- [ ] Grafos sendo gerados no AGE

## 12. Próximos Passos

1. **Acessar WebUI**: `http://seu-servidor:9621/webui`
2. **Login**: admin / pmpr2025!
3. **Upload de documentos** de teste (.txt simples)
4. **Testar queries** nos modos: naive, local, global, hybrid
5. **Verificar grafos** gerados via AGE

🎉 **Setup completo via Portainer! PostgreSQL + AGE + pgvector + LightRAG funcionando!**