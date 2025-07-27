# Contexto

Cria um ambiente integrado, configurado para que, no momento em que um registro for adicionado
à tabela "usuarios" do banco "testdb", no Postgres, seja enviada uma mensagem para o tópico
"postgres-usuarios" no kafka.

## Configurações após a execução dos containers

Após a criação dos containers é necessário executar os seguintes passos:

### 1. Verificar se todos os containers estão rodando

```bash
docker ps
```

Confirme que os seguintes containers estão em estado "Up":
- `broker` (Kafka)
- `kafka-connect`
- `postgres`
- `kafka-ui`
- `pgadmin4`

### 2. Aguardar inicialização completa

- Aguarde 2-3 minutos, ou mais, para que todos os serviços inicializem completamente, especialmente o Kafka Connect.
- Após isso, talvez seja ainda necessário reiniciar o container "pgadmin4" algumas vezes até que o serviço suba corretamente

### 3. Verificar se o Kafka Connect está funcionando

```bash
curl http://localhost:8083/
```

Se retornar informações sobre o Kafka Connect, prossiga para o próximo passo.


### 4. Acessar o base de dados através do pgadmin

- Digitar no browser 'http://localhost:5051' para acessar o pgadmin (senha = admin)
 
- Criar uma nova conexão 

![screenshot](/images/config-postgres-connection.png)

### 5. Criar o banco de dados e tabela no PostgreSQL

```
# Criar a tabela usuarios

CREATE TABLE usuarios (
    id SERIAL PRIMARY KEY,
    nome VARCHAR(100),
    email VARCHAR(100),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
); 
```

### 6. Configurar o Kafka Connect JDBC Source
Execute o comando abaixo (bash)
```bash
curl -X POST http://localhost:8083/connectors \
  -H "Content-Type: application/json" \
  -d '{
    "name": "postgres-source-connector",
    "config": {
      "connector.class": "io.confluent.connect.jdbc.JdbcSourceConnector",
      "connection.url": "jdbc:postgresql://postgres:5432/testdb",
      "connection.user": "postgres",
      "connection.password": "postgres",
      "table.whitelist": "usuarios",
      "mode": "incrementing",
      "incrementing.column.name": "id",
      "topic.prefix": "postgres-",
      "poll.interval.ms": 5000,
      "tasks.max": 1
    }
  }'
```

### 7. Verificar se o conector foi criado com sucesso

```bash
# Listar conectores
curl http://localhost:8083/connectors

# Verificar status do conector
curl http://localhost:8083/connectors/postgres-source-connector/status
```

### 8. Testar o fluxo de dados

#### 8.1. Inserir dados de teste na tabela

```bash
# Inserir registros de teste
INSERT INTO usuarios (nome, email) VALUES 
    ('João Silva', 'joao@email.com'),
    ('Maria Santos', 'maria@email.com');

# Verificar os dados inseridos
SELECT * FROM usuarios;
```

#### 8.2. Verificar se o tópico foi criado no Kafka

```bash
# Listar tópicos
docker exec -it kafka-tools /opt/kafka/bin/kafka-topics.sh --bootstrap-server broker:29092 --list
```

Deve aparecer o tópico: `postgres-usuarios`
 
### 9. Verificar funcionamento em tempo real

#### 9.1. Acessar o terminal do kafka-tools
```bash
  docker exec -it kafka-tools bash
```

#### 9.2. No terminal, deixe o consumer rodando:

```bash 
kafka-console-consumer.sh \
  --bootstrap-server broker:29092 \
  --topic postgres-usuarios \
  --from-beginning
```

#### 9.2. Use o pgadmin para inserir novos registros:

```bash
INSERT INTO usuarios (nome, email) VALUES ('Novo Usuario', 'novo@email.com');
```

Você deve ver a nova mensagem aparecer imediatamente no consumer.

## Interfaces Web Disponíveis

- **Kafka UI**: http://localhost:8084
  - Visualizar tópicos, partições e mensagens
  
- **pgAdmin**: http://localhost:5051
  - Login: `admin@admin.com` / `admin`
  - Para conectar ao PostgreSQL use hostname: `postgres`

## Portas dos Serviços

| Serviço | Porta | URL |
|---------|-------|-----|
| Kafka Broker | 9092 | localhost:9092 |
| Kafka Connect | 8083 | http://localhost:8083 |
| Kafka UI | 8084 | http://localhost:8084 |
| PostgreSQL | 5432 | localhost:5432 |
| pgAdmin | 5051 | http://localhost:5051 |

## Troubleshooting

### Problema: pgadmin não executa

```bash

# Reiniciar o serviço
docker restart pgadmin4
```

### Problema: Conector com erro

```bash
# Verificar status detalhado
curl http://localhost:8083/connectors/postgres-source-connector/status

# Deletar e recriar o conector
curl -X DELETE http://localhost:8083/connectors/postgres-source-connector
# Executar novamente o comando do passo 5
```
