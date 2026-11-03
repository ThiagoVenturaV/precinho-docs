# Capítulo 7: Guia de Deploy, Infraestrutura e Operações

> **Tempo estimado de leitura:** 15 minutos  
> **Objetivo:** Compreender a configuração de produção, variáveis de ambiente, comandos de manutenção, deploy no Vercel e orquestração Docker Compose na AWS EC2.

---

## 1. Topologia de Servidores e URLs de Produção

| Serviço | Provedor | URL Oficial / Host | Função |
| :--- | :--- | :--- | :--- |
| **Frontend Web** | Vercel | `https://precinho-xi.vercel.app/chat` | Interface SPA/PWA Angular para usuários finais |
| **Backend API** | AWS EC2 (Docker) | `18.219.118.83` (`us-east-2`) | Spring Boot 4.1.1 + Java 25 (porta 8080) |
| **Banco de Dados** | AWS EC2 (Docker) | `18.219.118.83:5432` | PostgreSQL 17 + pgvector (catálogo e transações) |
| **Túnel Seguro** | Cloudflare | `https://finest-mattress-assists-takes.trycloudflare.com` | Exposição segura sem IP público nem portas abertas |
| **Aceleração IA** | Groq LPU | `https://api.groq.com/openai/v1` | Modelo Qwen 3.8-27B com inferência ultrarrápida |

---

## 2. Orquestração Docker na AWS EC2 (`docker-compose.yml`)

Na instância EC2, a aplicação roda sob Docker Compose v2 no diretório `/home/ec2-user/precinho`:

```yaml
services:
  api:
    build:
      context: ./backend/ai-commerce-api
      dockerfile: Dockerfile
    container_name: ai-commerce-api
    restart: unless-stopped
    ports:
      - "8080:8080"
    environment:
      - DB_URL=jdbc:postgresql://db:5432/ai_commerce_db
      - DB_USERNAME=postgres
      - DB_PASSWORD=postgres
      - SPRING_PROFILES_ACTIVE=dev
      - SERVER_PORT=8080
      - CORS_ALLOWED_ORIGINS=https://precinho-xi.vercel.app,http://localhost:4200,http://localhost:4000
      - GROQ_ENABLED=true
      - GROQ_API_KEY=${GROQ_API_KEY}
      - GROQ_MODEL=qwen/qwen3.8-27b
      - AI_PROVIDER=groq
      - VECTOR_ENABLED=false
    depends_on:
      db:
        condition: service_healthy
    networks:
      - ai-commerce-net

  db:
    image: pgvector/pgvector:pg17
    container_name: ai-commerce-db
    restart: unless-stopped
    environment:
      - POSTGRES_DB=ai_commerce_db
      - POSTGRES_USER=postgres
      - POSTGRES_PASSWORD=postgres
    ports:
      - "5432:5432"
    volumes:
      - postgres_data:/var/lib/postgresql/data
    networks:
      - ai-commerce-net

volumes:
  postgres_data:

networks:
  ai-commerce-net:
    driver: bridge
```

---

## 3. O Serviço Cloudflare Tunnel (`systemd`)

Para garantir que o túnel HTTPS permaneça ativo mesmo após reinicializações da instância EC2, ele é gerenciado como um serviço do sistema operacional:

- **Arquivo do serviço:** `/etc/systemd/system/precinho-tunnel.service`
- **Comandos operacionais:**
  ```bash
  # Verificar status do túnel
  sudo systemctl status precinho-tunnel

  # Reiniciar serviço do túnel
  sudo systemctl restart precinho-tunnel

  # Acompanhar logs em tempo real
  sudo journalctl -u precinho-tunnel -f
  ```

---

## 4. Gerenciamento de Memória Swap em Instâncias EC2

Em instâncias do tipo `t3.small` ou `t3.medium` (2GB a 4GB de RAM), compilações pesadas do Maven (`mvnw clean package`) podem sofrer *Out-Of-Memory* (OOM).

Para evitar lentidão e saturação de disco EBS de 8GB:
- **Tamanho ideal de Swap:** **1,5 GB** (alocado via `/swapfile`).
- **Comando de verificação:** `free -m` e `df -h`.
- Isso garante 2.8GB de espaço livre em disco para imagens Docker e estabilidade de memória durante compilações.

---

## 5. Como Fazer Deploy de Novas Versões

### 5.1. Atualizando o Backend na EC2:
Conecte-se via SSH (`plink` ou cliente SSH tradicional) e execute:
```bash
# 1. Acesse o diretório do projeto
cd /home/ec2-user/precinho

# 2. Puxe as atualizações mais recentes do Git
git pull origin main

# 3. Recompile e reinicie o contêiner da API
docker compose build api
docker compose up -d --no-deps api

# 4. Acompanhe os logs de inicialização do Spring Boot
docker logs -f ai-commerce-api
```

### 5.2. Atualizando o Frontend no Vercel:
O repositório do frontend está integrado à Vercel com **Continuous Deployment (CI/CD)**:
1. Ao realizar `git push origin main` no repositório `Precinho-Frontend`, a Vercel dispara o build automaticamente;
2. O script `"prebuild": "node scripts/set-env.mjs"` gera as variáveis de ambiente necessárias em `src/environments/`;
3. A nova versão entra em produção em menos de 60 segundos com zero downtime.

---

## 6. Runbook de Diagnóstico e Troubleshooting

| Sintoma | Causa Mais Provável | Como Resolver |
| :--- | :--- | :--- |
| **Chat exibe erro de conexão** | Túnel Cloudflare desconectado | Execute `sudo systemctl restart precinho-tunnel` na EC2. |
| **Erro HTTP 403 Forbidden no chat** | Bloqueio de CORS | Verifique se a origem `https://precinho-xi.vercel.app` consta em `CORS_ALLOWED_ORIGINS` no `.env`. |
| **Erro "No space left on device"** | Cache do Docker ou swap inflado | Execute `docker builder prune -af` e ajuste o swap para 1.5GB. |
| **Erro HTTP 429 na IA** | Cota de tokens por minuto atingida | O sistema possui retry automático de 1.5s; mantenha o prompt compacto. |
| **Banco não conecta** | Contêiner `ai-commerce-db` reiniciando | Execute `docker logs ai-commerce-db` e verifique a saúde do volume `postgres_data`. |

---

## 📌 Principais Aprendizados do Capítulo 7

- A infraestrutura é **resiliente e econômica**, operando em contêineres Docker com túnel Cloudflare seguro.
- O túnel roda como **serviço nativo systemd**, subindo automaticamente em caso de reboot da máquina.
- O swap balanceado de **1.5GB** garante estabilidade para compilações Maven sem esgotar o disco EBS de 8GB.
- O fluxo de deploy é ágil e desacoplado: frontend via Vercel Edge CI/CD e backend via Docker Compose.

---

👉 **Siga para o [Capítulo 8: Roteiro de Pitch e FAQ Executivo](./08-roteiro-de-pitch-e-perguntas-respostas.md)**
