## Manual de Deploy de Aplicação Node.js com Docker

Este manual fornece um guia completo para dockerizar e implantar uma aplicação Node.js, cobrindo desde o ambiente de desenvolvimento local até a preparação para produção.

---

### Sumário

1.  **Introdução**
2.  **Pré-requisitos**
3.  **Estrutura da Aplicação Node.js (Exemplo)**
4.  **Dockerizando a Aplicação**
    *   4.1. Criando o `.dockerignore`
    *   4.2. Criando o `Dockerfile`
    *   4.3. Construindo a Imagem Docker
5.  **Desenvolvimento Local com Docker Compose**
    *   5.1. Criando o `docker-compose.yml`
    *   5.2. Executando e Gerenciando Serviços
6.  **Preparando para Produção**
    *   6.1. Variáveis de Ambiente
    *   6.2. Otimizações de Imagem (Multi-stage Build)
    *   6.3. Segurança
7.  **Deploy em Produção**
    *   7.1. Construindo a Imagem para Produção
    *   7.2. Executando em um Servidor (Múltiplas Abordagens)
        *   7.2.1. Docker Simples
        *   7.2.2. Usando Systemd (Recomendado para Single-Server)
        *   7.2.3. Com Reverse Proxy (Nginx/Traefik)
        *   7.2.4. Orquestradores (Docker Swarm/Kubernetes - Visão Geral)
8.  **Boas Práticas e Dicas**
9.  **Resolução de Problemas Comuns**
10. **Conclusão**

---

### 1. Introdução

Este manual tem como objetivo guiar você no processo de empacotar sua aplicação Node.js em contêineres Docker, facilitando o desenvolvimento, teste e implantação consistente em qualquer ambiente. O Docker garante que sua aplicação rode da mesma forma em sua máquina local, em servidores de staging e em produção, eliminando problemas de "funciona na minha máquina".

### 2. Pré-requisitos

Para seguir este manual, você precisará ter instalado em sua máquina:

*   **Docker Desktop (para Windows/macOS) ou Docker Engine (para Linux):** Inclui o Docker CLI e Docker Compose.
*   **Node.js e NPM/Yarn:** Para o desenvolvimento da sua aplicação Node.js.

### 3. Estrutura da Aplicação Node.js (Exemplo)

Vamos assumir uma estrutura de projeto Node.js básica:

```
my-nodejs-app/
├── node_modules/
├── src/
│   └── index.js
├── package.json
├── package-lock.json (ou yarn.lock)
└── .env (para variáveis de ambiente locais)
```

**Exemplo `src/index.js`:**

```javascript
const express = require('express');
const app = express();
const PORT = process.env.PORT || 3000;
const DB_HOST = process.env.DB_HOST || 'localhost';

app.get('/', (req, res) => {
  res.send(`Olá do Node.js na porta ${PORT}! Conectando ao DB em ${DB_HOST}`);
});

app.listen(PORT, () => {
  console.log(`Servidor rodando na porta ${PORT}`);
  console.log(`Variável DB_HOST: ${DB_HOST}`);
});
```

**Exemplo `package.json`:**

```json
{
  "name": "my-nodejs-app",
  "version": "1.0.0",
  "description": "Um app Node.js de exemplo para Docker.",
  "main": "src/index.js",
  "scripts": {
    "start": "node src/index.js",
    "dev": "nodemon src/index.js"
  },
  "keywords": [],
  "author": "",
  "license": "ISC",
  "dependencies": {
    "express": "^4.19.2"
  },
  "devDependencies": {
    "nodemon": "^3.1.0"
  }
}
```

**Exemplo `.env`:**

```
PORT=3000
DB_HOST=localhost
DB_USER=myuser
DB_PASSWORD=mypassword
DB_NAME=mydb
```

### 4. Dockerizando a Aplicação

Este é o coração do processo, onde criamos as instruções para o Docker construir sua imagem.

#### 4.1. Criando o `.dockerignore`

Assim como o `.gitignore`, o `.dockerignore` lista arquivos e diretórios que o Docker deve **ignorar** ao copiar o contexto para a construção da imagem. Isso acelera o processo de build e evita incluir arquivos desnecessários na imagem final.

Crie um arquivo chamado `.dockerignore` na raiz do seu projeto:

```
node_modules
npm-debug.log
.env
.git
.gitignore
Dockerfile
docker-compose.yml
README.md
*.log
tmp/
```

**Explicação:**
*   `node_modules`: Será instalado dentro do contêiner, não precisa ser copiado da máquina local.
*   `.env`: Contém variáveis de ambiente sensíveis e/ou locais. Não deve ir para a imagem.
*   `Dockerfile`, `docker-compose.yml`, `.git`, etc.: Arquivos de configuração ou controle de versão que não fazem parte da aplicação em tempo de execução.

#### 4.2. Criando o `Dockerfile`

O `Dockerfile` contém as instruções para o Docker construir a imagem da sua aplicação.

Crie um arquivo chamado `Dockerfile` na raiz do seu projeto:

```dockerfile
# Estágio de Build (para instalar dependências)
# Usamos uma imagem Node.js completa para garantir que o npm/yarn esteja disponível
FROM node:20-alpine AS builder

# Define o diretório de trabalho dentro do contêiner
WORKDIR /app

# Copia os arquivos de configuração de dependências primeiro
# Isso aproveita o cache do Docker: se package.json não mudar,
# npm install não será executado novamente.
COPY package.json package-lock.json ./

# Instala as dependências de produção
RUN npm install --only=production

# Estágio de Produção (imagem final, menor e mais segura)
FROM node:20-alpine

# Define o diretório de trabalho dentro do contêiner
WORKDIR /app

# Copia as dependências instaladas do estágio "builder"
COPY --from=builder /app/node_modules ./node_modules

# Copia o restante do código da aplicação
COPY . .

# Expõe a porta que sua aplicação Node.js escutará
# (Isto é apenas documentação, não publica a porta externamente)
EXPOSE 3000

# Comando para iniciar a aplicação quando o contêiner for executado
# Usa o script "start" definido no package.json
CMD ["npm", "start"]
```

**Explicação Detalhada do `Dockerfile`:**

*   **`FROM node:20-alpine AS builder`**: Define a imagem base para o primeiro estágio (`builder`). `node:20-alpine` é uma imagem oficial Node.js baseada em Alpine Linux, que é muito pequena e otimizada para contêineres. `AS builder` nomeia este estágio.
*   **`WORKDIR /app`**: Define `/app` como o diretório de trabalho padrão para quaisquer comandos subsequentes.
*   **`COPY package.json package-lock.json ./`**: Copia apenas os arquivos de dependência para o diretório de trabalho.
*   **`RUN npm install --only=production`**: Instala apenas as dependências de produção. Isso é crucial para manter a imagem final o menor possível.
*   **`FROM node:20-alpine`**: Inicia um **novo estágio de build** para a imagem final. Esta é a essência do "Multi-stage Build".
*   **`COPY --from=builder /app/node_modules ./node_modules`**: Copia as `node_modules` que foram instaladas no estágio `builder` para o estágio final. Isso evita ter que reinstalar as dependências e não inclui as ferramentas de build (npm, etc.) na imagem final.
*   **`COPY . .`**: Copia todo o restante do código da sua aplicação para o diretório de trabalho (`/app`).
*   **`EXPOSE 3000`**: Informa ao Docker que o contêiner escutará na porta 3000. Isso é mais uma documentação e não publica a porta automaticamente.
*   **`CMD ["npm", "start"]`**: Define o comando padrão para executar quando o contêiner é iniciado. Ele executa o script `start` do seu `package.json`.

#### 4.3. Construindo a Imagem Docker

Abra seu terminal na raiz do projeto (`my-nodejs-app/`) e execute:

```bash
docker build -t my-nodejs-app:latest .
```

*   `docker build`: Comando para construir uma imagem.
*   `-t my-nodejs-app:latest`: Atribui um nome (`my-nodejs-app`) e uma tag (`latest`) à imagem. Recomenda-se usar tags versionadas (ex: `my-nodejs-app:1.0.0`) em produção.
*   `.`: Indica que o Dockerfile está no diretório atual.

Você verá o Docker executando cada passo definido no `Dockerfile`. Ao final, sua imagem estará disponível:

```bash
docker images
```

### 5. Desenvolvimento Local com Docker Compose

O Docker Compose é uma ferramenta para definir e executar aplicações multi-contêiner Docker. É perfeito para configurar seu ambiente de desenvolvimento local, incluindo o banco de dados e outras dependências.

#### 5.1. Criando o `docker-compose.yml`

Crie um arquivo chamado `docker-compose.yml` na raiz do seu projeto:

```yaml
version: '3.8' # Versão do Docker Compose

services:
  app:
    build:
      context: . # Onde encontrar o Dockerfile (diretório atual)
      dockerfile: Dockerfile # Nome do Dockerfile
    ports:
      - "3000:3000" # Mapeia a porta 3000 do host para a porta 3000 do contêiner
    volumes:
      # Mapeia o diretório local do código para dentro do contêiner
      # Permite hot-reloading em desenvolvimento sem reconstruir a imagem
      - ./src:/app/src
      # Exclui o node_modules local para usar o do contêiner
      - /app/node_modules
    environment:
      # Variáveis de ambiente para o contêiner da aplicação
      # Podem ser carregadas de um arquivo .env para desenvolvimento
      # Ou definidas diretamente aqui
      DB_HOST: db # Nome do serviço de banco de dados, usado como hostname
      DB_USER: ${DB_USER:-myuser} # Usa variável de ambiente do host, ou default
      DB_PASSWORD: ${DB_PASSWORD:-mypassword}
      DB_NAME: ${DB_NAME:-mydb}
      PORT: 3000
    # O comando 'npm run dev' usa nodemon para hot-reloading
    # Garanta que 'nodemon' esteja no package.json como devDependency
    command: npm run dev
    depends_on:
      - db # Garante que o serviço 'db' inicie antes do 'app'
    networks:
      - app-network

  db:
    image: postgres:15-alpine # Imagem oficial do PostgreSQL
    ports:
      - "5432:5432" # Opcional: mapear porta do DB para o host
    environment:
      POSTGRES_USER: ${DB_USER:-myuser}
      POSTGRES_PASSWORD: ${DB_PASSWORD:-mypassword}
      POSTGRES_DB: ${DB_NAME:-mydb}
    volumes:
      - db-data:/var/lib/postgresql/data # Volume persistente para os dados do DB
    networks:
      - app-network

# Definição de volumes nomeados para persistência de dados
volumes:
  db-data:

# Definição de redes para isolamento de serviços
networks:
  app-network:
    driver: bridge # Rede padrão para comunicação entre os contêineres
```

**Explicação Detalhada do `docker-compose.yml`:**

*   **`services`**: Define os diferentes serviços (contêineres) da sua aplicação.
    *   **`app`**: O serviço da sua aplicação Node.js.
        *   `build: .`: Usa o Dockerfile no diretório atual para construir a imagem.
        *   `ports: - "3000:3000"`: Mapeia a porta 3000 do host para a porta 3000 do contêiner.
        *   `volumes: - ./src:/app/src`: Mapeia o código fonte local para dentro do contêiner. Qualquer mudança no seu código local reflete instantaneamente no contêiner (com `nodemon` ou similar).
        *   `environment`: Define variáveis de ambiente. `DB_HOST: db` permite que o `app` se conecte ao serviço `db` usando seu nome.
        *   `command: npm run dev`: Sobrescreve o `CMD` do Dockerfile para usar o `nodemon` em desenvolvimento.
        *   `depends_on: - db`: Garante que o serviço `db` seja iniciado antes do `app`.
    *   **`db`**: Um serviço de banco de dados PostgreSQL.
        *   `image: postgres:15-alpine`: Usa uma imagem oficial do PostgreSQL.
        *   `environment`: Define variáveis de ambiente específicas para o PostgreSQL (usuário, senha, banco de dados).
        *   `volumes: - db-data:/var/lib/postgresql/data`: Cria um volume persistente para os dados do banco de dados, garantindo que os dados não sejam perdidos ao parar/remover o contêiner.
*   **`volumes`**: Declara os volumes nomeados usados pelos serviços.
*   **`networks`**: Define redes personalizadas para os serviços, permitindo que eles se comuniquem de forma isolada.

#### 5.2. Executando e Gerenciando Serviços

Na raiz do seu projeto, execute:

```bash
docker compose up -d
```

*   `docker compose up`: Inicia os serviços definidos no `docker-compose.yml`.
*   `-d`: Executa os contêineres em modo "detached" (em segundo plano).

Você pode verificar o status dos contêineres com:

```bash
docker compose ps
```

Para ver os logs de todos os serviços (ou de um serviço específico):

```bash
docker compose logs -f         # Para todos os serviços
docker compose logs -f app     # Para o serviço 'app'
```

Para parar os serviços:

```bash
docker compose stop
```

Para parar e remover os contêineres (mas não os volumes persistentes):

```bash
docker compose down
```

Para parar e remover os contêineres E os volumes persistentes (cuidado, isso apaga seus dados do DB!):

```bash
docker compose down -v
```

Agora você pode acessar sua aplicação em `http://localhost:3000` e fazer alterações no código em `src/` que serão refletidas automaticamente (graças ao `nodemon` e ao volume).

### 6. Preparando para Produção

O ambiente de produção requer considerações adicionais para segurança, desempenho e gerenciamento de variáveis de ambiente.

#### 6.1. Variáveis de Ambiente

**NUNCA** coloque informações sensíveis (senhas, chaves de API) diretamente no seu `Dockerfile` ou no `docker-compose.yml` de produção.

*   **Para Docker Compose em produção (cenários simples):** Use o campo `environment` com valores reais ou carregue-os de um `.env` *não versionado* no servidor.
*   **Recomendado para Produção:**
    *   **Docker Secrets (Docker Swarm/Kubernetes):** Mecanismo seguro para gerenciar dados sensíveis.
    *   **Variáveis de Ambiente do Sistema:** Defina as variáveis de ambiente diretamente no servidor onde o Docker Engine está sendo executado.
    *   **Ferramentas de Gerenciamento de Segredos:** Vault (HashiCorp), AWS Secrets Manager, etc.

**Exemplo (definição no servidor antes de `docker run`):**

```bash
export PORT=8080
export DB_HOST=your-prod-db-host.com
export DB_USER=produser
export DB_PASSWORD=prodpassword
export DB_NAME=prod_db
```

#### 6.2. Otimizações de Imagem (Multi-stage Build)

Já implementamos o multi-stage build no `Dockerfile` na Seção 4.2. Esta é a principal otimização. Ela garante que sua imagem final contenha apenas o necessário para executar a aplicação, resultando em:

*   **Imagens Menores:** Reduz o tempo de download, uso de disco e superfície de ataque.
*   **Builds Mais Rápidos:** Aproveita o cache de camadas do Docker.
*   **Maior Segurança:** Não inclui ferramentas de build ou dependências de desenvolvimento.

#### 6.3. Segurança

*   **Não rodar como `root`:** Crie um usuário não-root dentro do contêiner e execute a aplicação com ele.
    ```dockerfile
    # Exemplo de Dockerfile com usuário não-root
    FROM node:20-alpine

    WORKDIR /app

    COPY package.json package-lock.json ./
    RUN npm install --only=production
    COPY . .

    # Cria um usuário não-root
    RUN addgroup --system appgroup && adduser --system --ingroup appgroup appuser
    # Define o usuário para executar a aplicação
    USER appuser

    EXPOSE 3000
    CMD ["npm", "start"]
    ```
*   **Imagens Base Mínimas:** `alpine` ou `slim` são preferíveis a imagens base completas.
*   **Atualize as Imagens Base:** Mantenha suas imagens base (`node:20-alpine`) atualizadas para receber patches de segurança.
*   **Escaneamento de Vulnerabilidades:** Use ferramentas como Snyk, Aqua Security ou o próprio Docker Scan para verificar vulnerabilidades em suas imagens.

### 7. Deploy em Produção

#### 7.1. Construindo a Imagem para Produção

Sempre reconstrua sua imagem explicitamente para produção, usando uma tag específica:

```bash
docker build -t my-nodejs-app:1.0.0 .
# Ou my-nodejs-app:$(git rev-parse HEAD) para usar o commit hash
```

Empurre a imagem para um Docker Registry (Docker Hub, AWS ECR, Google Container Registry, GitLab Registry, etc.):

```bash
docker login # Se ainda não estiver logado
docker tag my-nodejs-app:1.0.0 your-registry/my-nodejs-app:1.0.0
docker push your-registry/my-nodejs-app:1.0.0
```

No servidor de produção, você puxará essa imagem:

```bash
docker pull your-registry/my-nodejs-app:1.0.0
```

#### 7.2. Executando em um Servidor (Múltiplas Abordagens)

##### 7.2.1. Docker Simples (`docker run`)

Para um deploy rápido em um único servidor, sem orquestração:

```bash
# Defina as variáveis de ambiente no shell antes de executar
export PORT=8080
export DB_HOST=your-prod-db-host.com
export DB_USER=produser
export DB_PASSWORD=prodpassword
export DB_NAME=prod_db

docker run -d \
  --name my-nodejs-app-prod \
  -p 80:3000 \
  -e PORT=$PORT \
  -e DB_HOST=$DB_HOST \
  -e DB_USER=$DB_USER \
  -e DB_PASSWORD=$DB_PASSWORD \
  -e DB_NAME=$DB_NAME \
  your-registry/my-nodejs-app:1.0.0
```

*   `-d`: Executa em modo detached.
*   `--name`: Define um nome para o contêiner.
*   `-p 80:3000`: Mapeia a porta 80 do host (HTTP padrão) para a porta 3000 do contêiner.
*   `-e VAR=VAL`: Passa variáveis de ambiente para o contêiner.

**Desvantagens:** Não gerencia reinícios automáticos após falhas/reinicialização do servidor.

##### 7.2.2. Usando Systemd (Recomendado para Single-Server)

Systemd é um sistema de inicialização e gerenciador de serviços para Linux. Você pode usá-lo para gerenciar seu contêiner Docker como um serviço do sistema, garantindo que ele inicie no boot e seja reiniciado em caso de falha.

1.  Crie um arquivo `.env` no servidor (ex: `/etc/my-app/.env`) com suas variáveis de ambiente de produção:
    ```
    PORT=3000
    DB_HOST=your-prod-db-host.com
    DB_USER=produser
    DB_PASSWORD=prodpassword
    DB_NAME=prod_db
    ```

2.  Crie um arquivo de unidade Systemd (ex: `/etc/systemd/system/my-nodejs-app.service`):

    ```ini
    [Unit]
    Description=My Node.js App Docker Container
    After=docker.service
    Requires=docker.service

    [Service]
    Restart=always
    RestartSec=5s # Reinicia após 5 segundos em caso de falha
    # Carrega variáveis de ambiente do arquivo .env
    EnvironmentFile=/etc/my-app/.env
    ExecStartPre=-/usr/bin/docker stop my-nodejs-app-prod
    ExecStartPre=-/usr/bin/docker rm my-nodejs-app-prod
    ExecStart=/usr/bin/docker run --name my-nodejs-app-prod \
      -p 80:3000 \
      --env-file /etc/my-app/.env \
      your-registry/my-nodejs-app:1.0.0
    ExecStop=/usr/bin/docker stop my-nodejs-app-prod

    [Install]
    WantedBy=multi-user.target
    ```

3.  Recarregue o Systemd e ative/inicie o serviço:

    ```bash
    sudo systemctl daemon-reload
    sudo systemctl enable my-nodejs-app.service
    sudo systemctl start my-nodejs-app.service
    ```

4.  Verifique o status e logs:

    ```bash
    sudo systemctl status my-nodejs-app.service
    journalctl -u my-nodejs-app.service -f
    ```

##### 7.2.3. Com Reverse Proxy (Nginx/Traefik)

Para hospedar múltiplos aplicativos no mesmo servidor, lidar com SSL/TLS, e roteamento de tráfego, um reverse proxy é essencial.

**Exemplo com Nginx:**

1.  Execute seu contêiner Node.js internamente (sem `-p 80:3000` direto para o host):

    ```bash
    docker run -d \
      --name my-nodejs-app-prod \
      -e PORT=3000 \ # Exemplo: app escuta na porta 3000
      your-registry/my-nodejs-app:1.0.0
    ```
    *Crie uma rede Docker para Nginx e o app para comunicação interna.*

2.  Configure Nginx para rotear requisições para seu contêiner. Exemplo de configuração de Nginx (`/etc/nginx/sites-available/my-app.conf`):

    ```nginx
    server {
        listen 80;
        server_name myapp.example.com; # Seu domínio

        location / {
            # O nome do contêiner Docker pode ser usado como hostname na mesma rede
            proxy_pass http://my-nodejs-app-prod:3000;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;
        }

        # Para SSL (Let's Encrypt, por exemplo)
        # listen 443 ssl;
        # ssl_certificate /etc/letsencrypt/live/myapp.example.com/fullchain.pem;
        # ssl_certificate_key /etc/letsencrypt/live/myapp.example.com/privkey.pem;
    }
    ```

3.  Linke e recarregue Nginx:
    ```bash
    sudo ln -s /etc/nginx/sites-available/my-app.conf /etc/nginx/sites-enabled/
    sudo nginx -t
    sudo systemctl restart nginx
    ```

##### 7.2.4. Orquestradores (Docker Swarm/Kubernetes - Visão Geral)

Para aplicações em larga escala, com alta disponibilidade, escalabilidade e auto-cura, use plataformas de orquestração de contêineres:

*   **Docker Swarm:** Mais simples, integrado ao Docker Engine. Bom para começar com orquestração.
*   **Kubernetes:** Padrão da indústria para orquestração de contêineres. Complexo, mas extremamente poderoso e flexível. Ideal para ambientes de produção complexos.

Essas ferramentas gerenciam automaticamente a implantação, escalabilidade, rede e persistência, abstraindo a complexidade dos servidores individuais.

### 8. Boas Práticas e Dicas

*   **Use tags versionadas:** `my-nodejs-app:1.0.0` em vez de `my-nodejs-app:latest` para consistência e rollbacks fáceis.
*   **Mantenha as imagens pequenas:** Use multi-stage builds e imagens base Alpine.
*   **Logs para stdout/stderr:** Dentro do contêiner, escreva logs para a saída padrão. O Docker pode coletar e encaminhar esses logs para sistemas de logging centralizados (ELK Stack, Grafana Loki, CloudWatch Logs, etc.).
*   **Health Checks:** Adicione `HEALTHCHECK` ao seu `Dockerfile` para que o orquestrador (Docker Swarm, Kubernetes) saiba quando seu contêiner está realmente saudável e pronto para receber tráfego.
    ```dockerfile
    HEALTHCHECK --interval=30s --timeout=30s --start-period=5s --retries=3 \
      CMD curl -f http://localhost:3000/health || exit 1
    ```
*   **CI/CD:** Automatize o processo de build, teste e deploy de suas imagens Docker usando ferramentas como GitLab CI/CD, Jenkins, GitHub Actions, CircleCI.
*   **Cache de Dependências:** O `Dockerfile` exemplo já otimiza o cache copiando `package.json` antes de `npm install`.
*   **Evite o uso de volumes para código em produção:** Em produção, seu código deve estar *dentro* da imagem. Volumes para código são para desenvolvimento (hot-reloading).
*   **Monitoramento:** Integre suas aplicações Docker com soluções de monitoramento (Prometheus, Grafana, Datadog) para observar métricas e desempenho.

### 9. Resolução de Problemas Comuns

*   **Contêiner sai imediatamente (`Exited`)**:
    *   Verifique os logs: `docker logs <container_name_or_id>`.
    *   O comando `CMD` ou `ENTRYPOINT` está correto?
    *   A aplicação tem uma dependência que não foi instalada?
    *   Há erros de código na inicialização?
*   **Não consigo acessar a aplicação na porta mapeada**:
    *   A porta no `Dockerfile` (`EXPOSE`) está correta?
    *   A porta mapeada no `docker run -p` ou `docker-compose.yml` (`ports`) está correta?
    *   Há um firewall no host bloqueando a porta? (`ufw status`, `iptables -L`)
    *   A aplicação está escutando em `0.0.0.0` ou `localhost` dentro do contêiner? Deve ser `0.0.0.0` para que seja acessível externamente.
*   **Variáveis de ambiente não estão sendo carregadas**:
    *   Verifique se `process.env.MY_VAR` está sendo usado corretamente no Node.js.
    *   Confirme se as variáveis estão sendo passadas via `-e` no `docker run` ou `environment` no `docker-compose.yml`.
    *   Use `docker inspect <container_name_or_id>` e procure pela seção "Env" para verificar se as variáveis foram injetadas.
*   **`npm install` falha dentro do contêiner**:
    *   Verifique o `package.json` e `package-lock.json` para erros de sintaxe.
    *   A imagem base tem as ferramentas necessárias para compilar dependências nativas (se houver)? Imagens `alpine` podem precisar de pacotes adicionais (`build-base`, `python3`, etc.).
    *   Verifique as permissões de rede do Docker.
*   **Build da imagem lento/grande demais**:
    *   Verifique seu `.dockerignore`.
    *   Certifique-se de usar multi-stage builds.
    *   Use imagens base `alpine` ou `slim`.
    *   Execute `npm install --only=production`.

### 10. Conclusão

Parabéns! Você tem um manual completo para dockerizar e implantar sua aplicação Node.js. O Docker simplifica drasticamente o ciclo de vida do desenvolvimento ao deploy, garantindo ambientes consistentes. Continue explorando as funcionalidades do Docker e de ferramentas de orquestração para escalar suas aplicações com confiança.
