# 📦 batchstudy

Projeto de estudo com **Spring Batch** para importação de dados via arquivo CSV para um banco PostgreSQL.

---

## 🚀 Tecnologias

- Java 21
- Spring Boot 4.x
- Spring Batch 5.x
- Spring Data JPA
- PostgreSQL
- Maven

---

## 📋 O que o projeto faz

Ao ser executado, o job realiza as seguintes etapas em sequência:

```
importacaoJob
  ├── Step 1: lerTransacoesStep    → lê o arquivo dados.csv e persiste os registros no banco
  └── Step 2: moverArquivosStep   → move os CSVs processados para a pasta imported-files/
                                    renomeando com data e hora (ex: dados_20250501_143022.csv)
```

---

## 🗂️ Estrutura esperada de pastas

```
batchstudy/
├── files/                  ← coloque os arquivos .csv aqui antes de rodar
├── imported-files/         ← criada automaticamente após a execução
├── src/
└── pom.xml
```

---

## 📄 Formato do CSV

O arquivo `dados.csv` deve seguir o formato abaixo, com `;` como delimitador:

```
cpf;cliente;nascimento;evento;data;tipoIngresso;valor
123.456.789-00;João Silva;1990-05-01;Show de Rock;2025-04-15;INTEIRA;150.00
```

Linhas iniciadas com `--` são ignoradas (comentários).

---

## ⚙️ Configuração

### `application.yaml`

```yaml
spring:
  application:
    name: batchstudy

  datasource:
    url: jdbc:postgresql://localhost:5432/codetickets_db
    username: postgres
    password: postgres
    driver-class-name: org.postgresql.Driver

  batch:
    jdbc:
      initialize-schema: always
    job:
      enabled: false  # o job não roda automaticamente; use o endpoint REST
```

### Banco de dados

Certifique-se de ter o PostgreSQL rodando e o banco criado:

```sql
CREATE DATABASE codetickets_db;
```

As tabelas de metadados do Spring Batch são criadas automaticamente pelo `initialize-schema: always`.

---

## ▶️ Como executar

### Via endpoint REST

Suba a aplicação e dispare o job com uma requisição POST:

```bash
curl -X POST http://localhost:8080/batch/importar
```

### Via linha de comando

```bash
./mvnw spring-boot:run
```

---

## 🗄️ Tabela de destino

```sql
CREATE TABLE importacao (
    id              BIGSERIAL PRIMARY KEY,
    cpf             VARCHAR(14),
    cliente         VARCHAR(255),
    evento          VARCHAR(255),
    nascimento      DATE,
    tipo_ingresso   VARCHAR(50),
    valor           NUMERIC(10,2),
    hora_importacao TIMESTAMP
);
```

---

## 📌 Dependências principais

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-batch</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-batch-jdbc</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-jpa</artifactId>
</dependency>
<dependency>
    <groupId>org.postgresql</groupId>
    <artifactId>postgresql</artifactId>
    <scope>runtime</scope>
</dependency>
```

---

## 🔭 Possíveis melhorias

Veja a seção [MELHORIAS.md](MELHORIAS.md) para um detalhamento técnico das evoluções sugeridas.
