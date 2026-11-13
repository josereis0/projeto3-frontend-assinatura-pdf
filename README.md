# SignPDF Secure — Frontend Web (Projeto 3 · Segurança da Informação)

Aplicação web desacoplada para **Assinatura Digital de Documentos PDF** e **Verificação Pública de Autenticidade**, construída com **React 18** e **Tailwind CSS**, comunicando-se com a API REST em `/api/v1`.

---

## 1. Arquitetura do Frontend

A aplicação é organizada em módulos independentes por domínio de segurança:

```text
Camada de Apresentação (React + Tailwind CSS)
 ├── Módulo de Autenticação (Em)
 │    ├── Login com Senha (política de 12+ caracteres, bloqueio temporário e rate limit)
 │    ├── Segundo Fator TOTP (RFC 6238, janela de 30s ±1, Backup Codes de uso único)
 │    └── Passkeys WebAuthn / FIDO2 (navigator.credentials.create / .get, rpId, counter)
 ├── Módulo de Assinatura Digital de PDFs (pm / nf)
 │    ├── Validação de entrada (%PDF- magic number, limite de 20 MB, bloqueio de /JS e /JavaScript)
 │    ├── Posicionamento interativo do selo visual (coordenadas X, Y e página)
 │    └── Assinatura criptográfica RSA-2048 (WebCrypto / ISO 32000 incremental update)
 ├── Módulo de Verificação Pública (yv)
 │    ├── Acesso público sem necessidade de login e com rate limiting (20 req/min)
 │    └── Recomputação do SHA-256 original + verificação da assinatura RSA-2048 embutida
 └── Módulo de Governança, Sessões e Segurança (mm / gm / i1)
      ├── Inspeção e simulação do pipeline JWT RS256 (jti denylist, iat, ip_hash, ua_hash)
      ├── Rotação de Refresh Token opaco (HttpOnly, Secure, SameSite=Strict) e detecção de reuso
      ├── Gerenciamento de Sessões Ativas (encerramento individual e global)
      ├── Trilha de Auditoria Imutável (append-only) e Alertas de Segurança da Conta
      └── Painel de Configurações (2FA TOTP, Dispositivos Passkey, Troca de Senha, CSRF e Headers HTTP)
```

> **Proteção de Dados (LGPD):** Os arquivos PDF enviados para assinatura ou verificação são processados exclusivamente em memória e descartados imediatamente após a operação. Apenas hashes SHA-256 e metadados de auditoria são registrados.

---

## 2. Como Subir o Sistema Completo (Backend + Frontend)

### Passo 1 — Subir o Backend (Docker + PostgreSQL)

1. Abra um terminal na pasta do backend (`projeto3-backend-assinatura-pdf/projeto-assinatura-pdf/backend`):
   ```powershell
   cd "projeto3-backend-assinatura-pdf/projeto-assinatura-pdf/backend"
   ```
2. Caso o arquivo `.env` ainda não exista, crie-o a partir do exemplo:
   ```powershell
   copy .env.example .env
   ```
3. Inicie os containers do PostgreSQL 15 e da API FastAPI (porta `8000`):
   ```powershell
   docker compose up -d --build
   ```
4. Confirme que a API está respondendo em `http://localhost:8000/docs`.

---

### Passo 2 — Subir o Frontend Web (Porta 5173)

1. Abra **outro terminal** na pasta deste repositório (`projeto3-frontend-assinatura-pdf`):
   ```powershell
   cd "projeto3-frontend-assinatura-pdf"
   ```
2. Inicie o servidor HTTP na porta `5173` (origem permitida pelo CORS do backend):
   ```powershell
   python -m http.server 5173
   ```
3. Acesse no navegador:
   ```text
   http://localhost:5173
   ```

---

## 3. Repositórios Relacionados
- **Backend (API FastAPI + PostgreSQL + Docker):** `projeto3-backend-assinatura-pdf`
