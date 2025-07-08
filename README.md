1. Rodar container temporário para migração

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

  2. Aguardar migração terminar

docker service logs -f chatwoot-migrate

docker service rm chatwoot-migrate

# Conectar no PostgreSQL para confirmar
docker exec $(docker ps -q -f name=postgres) psql -U app_user -d chatwoot_database -c "\dt"

4. Subir stack do Chatwoot
