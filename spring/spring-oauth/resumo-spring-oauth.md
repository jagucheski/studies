# 🔐 OAuthStudy

Projeto de estudo para autenticação com **OAuth2** e **autenticação de dois fatores (2FA/TOTP)** utilizando Spring Boot.

---

## 🚀 Tecnologias

- Java 21
- Spring Boot 4.0.6
- Spring Security OAuth2 Client
- H2 Database (in-memory)
- TOTP (One-Time Password)
---

## ⚙️ Configuração

### Pré-requisitos

- Java 21+
- Maven 3.8+
- Conta no [Google Cloud Console](https://console.cloud.google.com) para obter as credenciais OAuth2

### Credenciais Google OAuth2

1. Acesse o [Google Cloud Console](https://console.cloud.google.com)
2. Crie um projeto
3. Vá em **APIs & Services → Credentials → Create Credentials → OAuth 2.0 Client ID**
4. Tipo: **Web application**
5. Authorized redirect URI: `http://localhost:8080/login/google/autorizado`
6. Copie o **Client ID** e **Client Secret**


## 📡 Endpoints - OAuth2 - Google

| Método | Endpoint                  | Descrição                                                                                |
|--------|---------------------------|------------------------------------------------------------------------------------------|
| `GET` | `/login/google`           | Gera a URL de autorização do Google com os scopes configurados                           |
| `GET` | `/login/google/autorizado`| Callback chamado pelo Google após o usuário autorizar o acesso. Retorna dados do usuario |

## 📡 Endpoints - 2FA/TOTP

| Método   | Endpoint | Descrição |
|----------|----------|-----------|
| `POST`   | `/verificar-a2f` | Valida o código TOTP do segundo fator |
| `PATCH`  | `/configurar-a2f/{email}` | Gera a URL OTPAuth para configurar o 2FA |

## 📡 Endpoints - H2
| Método   | Endpoint | Descrição |
|--------|---------------------------|-----------|
| `GET`    | `/h2-console` | Console do banco H2 |
---

## 🔄 Fluxo OAuth2

```
1. Usuário acessa GET /user
2. Spring redireciona para login do Google
3. Usuário autentica no Google
4. Google redireciona para /login/oauth2/code/google
5. Spring troca o code pelo token de acesso
6. GET /user retorna { name, email, picture }
```
---

## 🔒 Fluxo 2FA (TOTP)

```
1. PATCH /configurar-a2f/{email}  → retorna URL OTPAuth
2. Usuário escaneia o QR Code com Google Authenticator ou Authy
3. POST /verificar-a2f            → valida o código de 6 dígitos
```

### Exemplo de request `/verificar-a2f`:

```json
{
  "email": "usuario@email.com",
  "codigo": "123456"
}
```

### Exemplo de response:

```json
{
  "email": "usuario@email.com",
  "codigo": "123456",
  "valido": true
}
```

---

## 🗄️ Banco de Dados H2

Acesse o console em `http://localhost:8080/h2-console` com:

| Campo | Valor |
|-------|-------|
| JDBC URL | `jdbc:h2:mem:testdb` |
| User Name | `sa` |
| Password | `password` |

---

## 📁 Estrutura do Projeto

```
src/main/java/br/com/jagucheski/oauthstudy/
├── OauthstudyApplication.java
├── config/
│   └── SecurityConfig.java
├── controller/
│   ├── AutenticacaoController.java
│   └── GoogleController.java
├── model/
│   ├── DadosAutenticacao2F.java
│   ├── Usuario.java
│   ├── UsuarioAutenticacao2F.java
│   └── UsuarioLogiOauth.java
├── repository/
│   └── UsuarioRepository.java
└── service/
│   ├── LoginGoogleService.java
│   ├── UsuarioService.java
    └── TotpService.java

src/main/resources/
├── application.yml
├── schema.sql
└── data.sql
```

---
