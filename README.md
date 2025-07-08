# 🚀 Stack Completa Docker Swarm - Produção

## 📋 Visão Geral

Stack profissional para produção contendo todas as ferramentas necessárias para automação, atendimento ao cliente e integração WhatsApp Business.

### 🛠️ Aplicações Incluídas

- **Portainer** - Gerenciamento visual Docker Swarm
- **Traefik v3** - Reverse proxy com SSL automático
- **PostgreSQL 16** - Banco principal com pgvector (IA)
- **Redis 7** - Cache e message broker
- **MinIO** - Object storage S3-compatible
- **N8N** - Automação workflows (4 serviços)
- **Chatwoot** - Plataforma de atendimento
- **Evolution API** - Integração WhatsApp Business

---

## ⚙️ Instalação Inicial

### 1. Instalação Docker
```bash
curl -fsSL https://get.docker.com -o get-docker.sh
sudo sh get-docker.sh
```

### 2. Configurar Docker Swarm
```bash
# Substituir pelo IP do seu servidor
docker swarm init --advertise-addr=SEU_IP_AQUI
```

### 3. Criar Redes Docker
```bash
# Rede pública (Traefik)
docker network create --driver=overlay traefik_public

# Rede interna (aplicações)
docker network create --driver=overlay app_network
```

### 4. Criar Diretório MinIO
```bash
sudo mkdir -p /var/data/minio
sudo chown -R $USER:docker /var/data/minio
```

---

## 📦 Ordem de Deploy

### 1. Infraestrutura Base
```bash
# 1. Portainer (gerenciamento)
docker stack deploy -c portainer.yml portainer

# 2. Traefik (reverse proxy)
docker stack deploy -c traefik.yml traefik
```

### 2. Bancos de Dados
```bash
# 3. PostgreSQL
docker stack deploy -c postgres.yml postgres

# 4. Redis
docker stack deploy -c redis.yml redis
```

### 3. Storage
```bash
# 5. MinIO
docker stack deploy -c minio.yml minio
```

### 4. Aplicações
```bash
# 6. N8N (4 serviços)
docker stack deploy -c n8n_editor.yml n8n-editor
docker stack deploy -c n8n_webhook.yml n8n-webhook
docker stack deploy -c n8n_worker.yml n8n-worker
docker stack deploy -c n8n_mcp.yml n8n-mcp

# 7. Evolution API
docker stack deploy -c evolution.yml evolution

# 8. Chatwoot (requer migração primeiro)
# Ver seção "Migração Chatwoot" abaixo
```

---

## 🔄 Migração Chatwoot

**⚠️ IMPORTANTE**: Execute a migração antes de subir o Chatwoot:

### 1. Container Temporário para Migração
```bash
docker service create --name chatwoot-migrate \
  --network app_network \
  --restart-condition none \
  -e NODE_ENV=production \
  -e RAILS_ENV=production \
  -e SECRET_KEY_BASE=194F83A5E420E2908213782FE1E64C2E7C07B5C3F7409BA90138E2D1E658BD77 \
  -e POSTGRES_HOST=postgres \
  -e POSTGRES_USERNAME=app_user \
  -e POSTGRES_PASSWORD="4hwM-An0+14dC=z" \
  -e POSTGRES_DATABASE=chatwoot_database \
  -e POSTGRES_PORT=5432 \
  -e REDIS_URL=redis://:J40geW0tC8VoaUqoZ@redis:6379 \
  chatwoot/chatwoot:v4.2.0 \
  bundle exec rails db:chatwoot_prepare
```

### 2. Monitorar Migração
```bash
# Acompanhar logs
docker service logs -f chatwoot-migrate
```

### 3. Limpar e Verificar
```bash
# Remover serviço temporário
docker service rm chatwoot-migrate

# Verificar tabelas criadas
docker exec $(docker ps -q -f name=postgres) psql -U app_user -d chatwoot_database -c "\dt"
```

### 4. Deploy Chatwoot
```bash
docker stack deploy -c chatwoot.yml chatwoot
```

---

## 🌐 Domínios Configurados

Ajuste os domínios nos arquivos YML conforme sua necessidade:

- **Portainer**: `painel.seudominio.com.br`
- **Traefik**: `traefik.seudominio.com.br` (opcional)
- **MinIO API**: `files.seudominio.com.br`
- **MinIO Console**: `minio.seudominio.com.br`
- **N8N Editor**: `editor.seudominio.com.br`
- **N8N Webhook**: `workflow.seudominio.com.br`
- **Chatwoot**: `chat.seudominio.com.br`
- **Evolution API**: `evo.seudominio.com.br`

---

## 🔧 Configurações Importantes

### Variáveis para Alterar

Antes do deploy, altere estas variáveis nos arquivos:

#### PostgreSQL (`postgres.yml`)
```yaml
POSTGRES_PASSWORD: "SUA_SENHA_POSTGRES"
```

#### Redis (`redis.yml`)
```yaml
--requirepass "SUA_SENHA_REDIS"
```

#### MinIO (`minio.yml`)
```yaml
MINIO_ROOT_USER: SEU_USUARIO_MINIO
MINIO_ROOT_PASSWORD: SUA_SENHA_MINIO
```

#### Chatwoot (`chatwoot.yml`)
```yaml
SECRET_KEY_BASE: SUA_CHAVE_SECRETA_64_CHARS
```

#### N8N (todos os arquivos)
```yaml
N8N_ENCRYPTION_KEY: SUA_CHAVE_N8N_32_CHARS
```

#### Evolution API (`evolution.yml`)
```yaml
AUTHENTICATION_API_KEY: SUA_CHAVE_EVOLUTION
```

### Gerar Chaves Seguras
```bash
# SECRET_KEY_BASE (64 chars)
openssl rand -hex 32

# N8N_ENCRYPTION_KEY (32 chars)
openssl rand -hex 16

# Senhas seguras
openssl rand -base64 32
```

---

## 📊 Monitoramento

### Verificar Serviços
```bash
# Listar todos os serviços
docker service ls

# Ver logs específicos
docker service logs -f NOME_DO_SERVICO

# Ver detalhes do serviço
docker service inspect NOME_DO_SERVICO
```

### Verificar Saúde
```bash
# Containers rodando
docker ps

# Uso de recursos
docker stats

# Espaço em disco
docker system df
```

---

## 🔒 Segurança

### Configurações SSL
- Certificados automáticos via Let's Encrypt
- Redirecionamento HTTP → HTTPS
- Headers de segurança configurados

### Acesso Restrito
- PostgreSQL: Apenas rede interna
- Redis: Apenas rede interna + senha
- MinIO: CORS configurado
- Todas aplicações: Atrás do Traefik

---

## 🆘 Troubleshooting

### Problemas Comuns

#### Serviço não sobe
```bash
# Ver logs de erro
docker service logs NOME_SERVICO

# Ver eventos do swarm
docker events

# Verificar constraints
docker service inspect NOME_SERVICO --format '{{.Spec.TaskTemplate.Placement}}'
```

#### SSL não funciona
```bash
# Ver logs do Traefik
docker service logs traefik

# Verificar DNS
nslookup seudominio.com

# Testar portas
curl -I http://seudominio.com
```

#### Banco não conecta
```bash
# Testar PostgreSQL
docker exec CONTAINER_POSTGRES psql -U app_user -d app_database -c "SELECT 1"

# Testar Redis
docker exec CONTAINER_REDIS redis-cli ping
```

---

## 📚 Recursos Adicionais

### Documentação
- [Docker Swarm Docs](https://docs.docker.com/engine/swarm/)
- [Traefik v3 Docs](https://doc.traefik.io/traefik/)
- [PostgreSQL Docs](https://www.postgresql.org/docs/)

### Backup
```bash
# PostgreSQL
docker exec postgres pg_dump -U app_user app_database > backup.sql

# MinIO
mc mirror minio/bucket /backup/minio/

# Volumes Docker
docker run --rm -v volume_name:/data -v /backup:/backup alpine tar czf /backup/volume.tar.gz -C /data .
```

---

## 📞 Suporte

### Links Úteis
- **GitHub**: https://github.com/andersonlemesc/stacks_producao
- **Issues**: Reporte problemas na seção Issues do GitHub
- **Documentação**: Wiki do repositório

### Requisitos Mínimos
- **CPU**: 4 vCPUs (mínimo 2)
- **RAM**: 8GB (mínimo 6GB)
- **Storage**: 80GB SSD
- **OS**: Ubuntu 22.04 LTS

---

## 📝 Licença

Este projeto está sob licença MIT. Veja o arquivo [LICENSE](LICENSE) para mais detalhes.

---

## ⭐ Contribuições

Contribuições são bem-vindas! Por favor:

1. Fork o projeto
2. Crie uma branch para sua feature
3. Commit suas mudanças
4. Push para a branch
5. Abra um Pull Request

---

**🚀 Desenvolvido por Anderson Lemes**  
**📺 YouTube**: [AndersonLemes](https://www.youtube.com/@andersonslemes) 
**💼 LinkedIn**: [AndersonLemes](https://www.linkedin.com/in/anderson-lemes-713335250/)
