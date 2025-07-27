# Contexto

Cria um ambiente integrado, configurado para que, no momento em que um registro for adicionado
à tabela "usuarios" do banco "testdb", no Postgres, seja enviada uma mensagem para o tópico
"postgres-usuarios" no kafka.

## docker-compose

```
services:
  kafka:
    image: apache/kafka:latest
    hostname: broker
    container_name: broker
    ports:
      - "9092:9092"
    environment:
      KAFKA_BROKER_ID: 1
      KAFKA_LISTENER_SECURITY_PROTOCOL_MAP: PLAINTEXT:PLAINTEXT,PLAINTEXT_HOST:PLAINTEXT,CONTROLLER:PLAINTEXT
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://broker:29092,PLAINTEXT_HOST://localhost:9092
      KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 1
      KAFKA_GROUP_INITIAL_REBALANCE_DELAY_MS: 0
      KAFKA_TRANSACTION_STATE_LOG_MIN_ISR: 1
      KAFKA_TRANSACTION_STATE_LOG_REPLICATION_FACTOR: 1
      KAFKA_PROCESS_ROLES: broker,controller
      KAFKA_NODE_ID: 1
      KAFKA_CONTROLLER_QUORUM_VOTERS: 1@broker:29093
      KAFKA_LISTENERS: PLAINTEXT://broker:29092,CONTROLLER://broker:29093,PLAINTEXT_HOST://0.0.0.0:9092
      KAFKA_INTER_BROKER_LISTENER_NAME: PLAINTEXT
      KAFKA_CONTROLLER_LISTENER_NAMES: CONTROLLER
      KAFKA_LOG_DIRS: /tmp/kraft-combined-logs
      CLUSTER_ID: MkU3OEVBNTcwNTJENDM2Qk
    restart: always

  kafka-ui:
    image: provectuslabs/kafka-ui:latest
    container_name: kafka-ui
    ports:
      - "8084:8080"
    environment:
      KAFKA_CLUSTERS_0_NAME: local
      KAFKA_CLUSTERS_0_BOOTSTRAP_SERVERS: broker:29092
    depends_on:
      - kafka
    restart: always

  # PostgreSQL database
  postgres:
    image: postgres:15
    container_name: postgres
    environment:
      POSTGRES_DB: testdb
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: postgres
    ports:
      - "5432:5432"
    volumes:
      - postgres_data:/var/lib/postgresql/data
    restart: always

  # pgAdmin for PostgreSQL management
  pgadmin:
    image: dpage/pgadmin4:latest
    container_name: pgadmin4
    environment:
      PGADMIN_DEFAULT_EMAIL: admin@admin.com
      PGADMIN_DEFAULT_PASSWORD: admin
      PGADMIN_CONFIG_SERVER_MODE: 'False'
    ports:
      - "5051:80"
    volumes:
      - pgadmin_data:/var/lib/pgadmin
    depends_on:
      - postgres
    restart: always
    user: "5050:5050"  # Adicionar para resolver problemas de permissão

  # Kafka Connect with JDBC connector
  kafka-connect:
    image: confluentinc/cp-kafka-connect:7.5.0  # Use a specific stable version
    container_name: kafka-connect
    depends_on:
      - kafka
      - postgres
    ports:
      - "8083:8083"
    environment:
      CONNECT_BOOTSTRAP_SERVERS: broker:29092
      CONNECT_REST_ADVERTISED_HOST_NAME: kafka-connect
      CONNECT_REST_PORT: 8083
      CONNECT_GROUP_ID: connect-cluster-group
      CONNECT_CONFIG_STORAGE_TOPIC: docker-connect-configs
      CONNECT_OFFSET_STORAGE_TOPIC: docker-connect-offsets
      CONNECT_STATUS_STORAGE_TOPIC: docker-connect-status
      CONNECT_CONFIG_STORAGE_REPLICATION_FACTOR: 1
      CONNECT_OFFSET_STORAGE_REPLICATION_FACTOR: 1
      CONNECT_STATUS_STORAGE_REPLICATION_FACTOR: 1
      CONNECT_KEY_CONVERTER: org.apache.kafka.connect.json.JsonConverter
      CONNECT_VALUE_CONVERTER: org.apache.kafka.connect.json.JsonConverter
      CONNECT_KEY_CONVERTER_SCHEMAS_ENABLE: "false"
      CONNECT_VALUE_CONVERTER_SCHEMAS_ENABLE: "false"
      CONNECT_PLUGIN_PATH: "/usr/share/java,/usr/share/confluent-hub-components"
      CONNECT_INTERNAL_KEY_CONVERTER: org.apache.kafka.connect.json.JsonConverter
      CONNECT_INTERNAL_VALUE_CONVERTER: org.apache.kafka.connect.json.JsonConverter
      CONNECT_INTERNAL_KEY_CONVERTER_SCHEMAS_ENABLE: "false"
      CONNECT_INTERNAL_VALUE_CONVERTER_SCHEMAS_ENABLE: "false"
      CONNECT_LOG4J_ROOT_LOGLEVEL: INFO
      CONNECT_LOG4J_LOGGERS: org.apache.kafka.connect.runtime.rest=WARN,org.reflections=ERROR
      # Add these configurations to help with timing issues
      CONNECT_HEARTBEAT_INTERVAL_MS: 3000
      CONNECT_SESSION_TIMEOUT_MS: 30000
      CONNECT_REBALANCE_TIMEOUT_MS: 300000
    volumes:
      - ./connectors:/usr/share/confluent-hub-components
    command:
      - bash
      - -c
      - |
        echo "Waiting for Kafka to be ready..."
        sleep 30
        echo "Installing JDBC connector..."
        confluent-hub install --no-prompt confluentinc/kafka-connect-jdbc:10.7.4
        echo "Starting Kafka Connect..."
        /etc/confluent/docker/run
    restart: always
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8083/"]
      interval: 30s
      timeout: 10s
      retries: 10
      start_period: 120s

  # roda um container linux configurado para executar os comandos do kafka
  kafka-tools:
    image: apache/kafka:latest
    container_name: kafka-tools
    depends_on:
      - kafka
    command: sleep infinity
    environment:
      KAFKA_BOOTSTRAP_SERVERS: broker:29092
      PATH: "/opt/kafka/bin:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin"
    volumes:
      - ./scripts:/scripts

volumes:
  postgres_data:
  pgadmin_data:

networks:
  default:
    name: kafka-network
    driver: bridge
```

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

### 8.2. Verificar se o tópico foi criado no Kafka

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
