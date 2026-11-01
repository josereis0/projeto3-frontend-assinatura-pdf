# Projeto 3: Sistema Web de Assinatura Digital de PDFs
## Segurança da Informação - UAB00084 - Prof. Ygor Amaral

---
## Slide 1 - Capa
### Projeto 3: Assinatura Digital de PDFs
**Segurança como requisito central, não opcional**

- Disciplina: Segurança da Informação (UAB00084) | Engenharia de Computação UABJ/UFRPE
- Entrega: 20/11/2026 | Branch main com README executável
- Aluno: [Seu Nome] | Repositórios: backend + frontend desacoplados RNF-01

Visual: Capa com cadeado digital + documento PDF sendo selado, arquitetura em 3 camadas ao fundo

---
## Slide 2 - O Problema Real
### Por que assinar PDF é crítico?

**Contexto jurídico e LGPD:**
- PDFs com valor jurídico precisam garantir **autoria** (quem assinou) e **integridade** (não foi alterado depois)
- Selo visual falsificável sem criptografia é inútil
- LGPD Art. 6º: reter dados apenas pelo tempo necessário -> PDFs não podem ser armazenados RNF 9.2
- Tela de verificação pública é alvo fácil para bots -> precisa rate limiting REQ-53

**Ameaças:**
- Phishing de senha, credential stuffing, força bruta, enumeração de usuários
- Roubo de token, XSS, clickjacking, CSRF via cookie
- PDF malicioso com /JS explorando parser

---
## Slide 3 - Visão Geral da Arquitetura
### 3 partes: Frontend, API REST, PostgreSQL. PDF nunca chega ao banco.

**Fluxo (Figura 1 do PDF):**
- Navegador -> Frontend web (login, assinatura, verificação) Tailwind RNF-03
- Servidor -> API REST FastAPI RNF-02 -> Processamento PDF em memória, ambiente isolado REQ-62 (container sem rede)
- Dados -> PostgreSQL RNF-04: usuários, chaves, assinaturas, logs. PDFs processados e descartados

**Princípio:** Regras de segurança centralizadas em middlewares. Rotas recebem requisição já conferida.

Diagrama: Entrada -> Middlewares (CORS, headers, rate limit, CSRF, JWT) -> Rotas /api/v1 -> Serviços -> Infra (Criptografia, PDF isolado, Persistência)

---
## Slide 4 - Objetivos do Projeto
### 4 objetivos que guiam 66 requisitos

1. **Praticar controles de autenticação modernos:** senha Argon2id REQ-02, passkeys FIDO2/WebAuthn REQ-06, segundo fator TOTP REQ-11
2. **Manter sessão segura com JWT:** revogação no logout via denylist REQ-21 e rotação refresh token REQ-29 com detecção de reuso REQ-30
3. **Defender no navegador:** CORS REQ-31, 8 headers de segurança REQ-37..44, CSRF REQ-65
4. **Aplicar assinatura a caso real:** assinar PDF ISO 32000 REQ-46 com selo visual REQ-47 verificável por qualquer pessoa REQ-50..52

**Processo:** GitHub Project por aplicação, @ygoramaral colaborador, 1 issue por requisito com código REQ-01 no título, branch por issue, Ref #id no commit

---
## Slide 5 - Autenticação: Senha Forte
### REQ-01 a REQ-05 - Primeira barreira

**REQ-01 Política:** mínimo 12 caracteres + maiúsculas + minúsculas + números + símbolos. Checklist visual em tempo real no frontend.

**REQ-02 Armazenamento:** Argon2id obrigatório, parâmetros RFC 9106: memory=64MB, iterations=3, parallelism=4. Nunca bcrypt/scrypt/MD5/SHA-1. Código: `PasswordHasher(memory_cost=65536, time_cost=3, parallelism=4)`

**REQ-03 Bloqueio temporário:** após N falhas (ex: 5) -> bloqueio 15min. Campo `locked_until` + `failed_attempts` no User.

**REQ-04 Mensagem única:** login não diferencia usuário inexistente de senha errada -> evita enumeração de usuários.

**REQ-05 Rate limiting:** por IP e por conta, ex: 10 tentativas/min por IP. Técnica contra brute force e credential stuffing. Implementado em `middlewares/rate_limit.py` com buckets in-memory (Redis em produção).

---
## Slide 6 - Autenticação: Passkeys FIDO2
### REQ-06 a REQ-10 - Fim do phishing de senha

**Por que passkey?** Autenticação criptograficamente vinculada à origem. Chave privada nunca sai do autenticador (biometria/PIN). Nenhuma senha trafega.

**Fluxo (Figura 2):**
- Registro: `navigator.credentials.create()` -> servidor sorteia desafio -> autenticador assina com privada -> servidor verifica com pública guardada
- Auth: `navigator.credentials.get()`

**REQ-07 Dados no banco:** credentialId, publicKey, counter, aaguid (modelo do autenticador)

**REQ-08 Validação counter:** se ambos zero -> passkey sincronizada iCloud/Google (não usa contador). Se algum diferente de zero, recebido deve ser > armazenado, senão rejeita = possível clonagem.

**REQ-09 rpId:** domínio exato da aplicação configurado via env, evita phishing cross-domain

**REQ-10 Gerenciamento:** lista dispositivos, revoga, nomeia cada um

---
## Slide 7 - Autenticação: TOTP 2FA
### REQ-11 a REQ-16 - Segunda barreira sem SMS

**Motivação:** evita SIM swap. Baseado em tempo, não precisa infraestrutura SMS.

**REQ-11 Algoritmo:** TOTP RFC 6238, janela 30s, tolerância ±1 janela para compensar relógio.

**REQ-12 Segredo:** mínimo 160 bits entropia (`secrets.token_bytes(20)`), armazenado criptografado com Fernet + master key do .env. Funções `encrypt_secret/decrypt_secret`

**REQ-13 QR Code:** na ativação exibe QR compatível Google Authenticator/Authy. Gera URI `otpauth://` + imagem PNG base64

**REQ-14 Backup codes:** 8 códigos de uso único, armazenados como hash SHA-256 `BackupCode` table, exibidos uma vez só.

**REQ-15 TOTP antes do JWT:** depois de validar senha, exige código antes de emitir token. Token não sai na metade do fluxo (Figura 3)

**REQ-16 Alerta:** registra tentativas TOTP inválidas em `TOTPFailedAttempt` e alerta usuário quando se repetem

---
## Slide 8 - Fluxo Completo de Login com Senha
### Figura 3 do PDF - Nenhuma recusa diz se erro estava no login ou senha

Fluxograma:
1. Recebe login e senha
2. Dentro do limite de tentativas? Se não -> Recusa
3. Senha confere? Se não -> incrementa failed_attempts, log falha, Recusa genérica AUTH_001
4. 2FA ativado? Se sim -> Pede código TOTP -> Código válido? Se não -> log totp_invalido, Recusa
5. Emite tokens (access JWT 15min + refresh opaco 7 dias em cookie HttpOnly Secure SameSite=Strict)

**Pontos de auditoria:** cada falha gera `AuditLog` action `falha_autenticacao`, sucesso gera `login` + `emissao_token`

**Segurança em camadas:** mesmo com senha vazada, sem TOTP não entra; mesmo com TOTP roubado, expira em 30s

---
## Slide 9 - Tokens JWT: Fundamentos
### REQ-17 a REQ-20 - Conveniência sem perder revogação

**Problema do JWT padrão:** sem estado no servidor = sem revogação. Token interceptado pós-logout ainda válido até expirar.

**Solução do projeto: dois tokens**

| | Access Token | Refresh Token |
|---|---|---|
| O que é | JWT assinado RS256 | Valor opaco 256 bits CSPRNG |
| Validade | 15 min REQ-18 | 7 dias REQ-19 |
| Para que serve | Autenticar cada requisição | Obter novo par quando access expira |
| Como trafega | Header Authorization | Cookie HttpOnly Secure SameSite Strict REQ-20 nunca localStorage |
| O que servidor guarda | Só jti dos revogados na denylist | Hash de cada refresh emitido |

**REQ-17 Algoritmo:** RS256 ou ES256 assimétrico. Nunca `alg: none`

**Geração refresh:** `secrets.token_bytes(32)` -> hash SHA-256 armazenado, raw em hex no cookie

---
## Slide 10 - JWT: Denylist e Verificações Avançadas
### REQ-21 a REQ-28 - Assinatura válida é só o primeiro filtro

**REQ-21/22/23 Invalidação no logout:** jti armazenado em `denylist` com expiração = tempo restante do token. Consulta à denylist a cada requisição autenticada antes de processar. Refresh também invalidado no banco.

**Claims adicionais no payload (Figura 4):**

- REQ-24 jti: ID único do token, usado na denylist
- REQ-25 iat: data emissão. Tokens emitidos antes da última troca de senha são rejeitados -> força re-login pós troca senha
- REQ-26 ip: hash do IP no login. A cada requisição calcula hash do IP atual e compara. Se diferente -> gera alerta (não rejeita) para detectar roubo
- REQ-27 ua_hash: hash do User-Agent. Token rejeitado se UA da requisição != registrado na emissão
- REQ-28 session_id: identificador da sessão, permite revogar sessão específica e implementar "sair de todos dispositivos"

**Fluxo verificação (Figura 4):** assinatura e validade conferem? -> jti na denylist? -> emitido antes da troca de senha? -> UA confere? -> Processa, senão Rejeita. IP fica de fora porque só gera alerta.

---
## Slide 11 - JWT: Rotação e Detecção de Comprometimento
### REQ-29 e REQ-30 - Cada refresh vale para um uso só

**Fluxo normal (Figura 5 esquerda):**
- Frontend envia refresh token (cookie HttpOnly automático)
- Servidor valida no banco, verifica `used==False` e não expirado
- Marca `used=True`, emite novo par (access + refresh), novo `family_id` mesmo da família
- Novo refresh em cookie, access no body

**Ataque e detecção (Figura 5 direita):**
- Se mesmo refresh usado 2x -> assume comprometido REQ-30
- Invalida toda família de sessões daquele usuário: `DELETE WHERE family_id ==`
- Todas sessões caem, usuário precisa logar de novo, atacante perde acesso

**Por que rotação?** Refresh tem longa duração 7 dias = risco maior. Se nunca substituído, quem rouba gera access indefinidamente. Rotação reduz janela de exploração.

**Implementação:** `RefreshToken` com `family_id`, `used`, `expires_at`, `token_hash`. Endpoint `/api/v1/auth/refresh`

---
## Slide 12 - CORS e Preflight
### REQ-31 a REQ-36 - Bloqueando requisições de origens maliciosas

**Problema sem CORS:** requisições maliciosas de outros domínios interagindo com API em nome de usuário autenticado -> abre CSRF e requisições indevidas entre origens.

**REQ-31 Origens permitidas:** lista explícita via env `CORS_ALLOWED_ORIGINS=https://app.example.com,http://localhost:5173`. Nunca `*` em endpoints autenticados.

**REQ-32 Métodos:** restritos a GET, POST, PUT, DELETE, OPTIONS

**REQ-33 Headers:** restritos a Authorization, Content-Type, X-Request-ID

**REQ-34 Credentials:** `Access-Control-Allow-Credentials: true` só para origens confiáveis

**REQ-35 Preflight:** servidor responde corretamente a OPTIONS (Figura 6): navegador pergunta "posso fazer?" com métodos e headers -> servidor responde headers CORS -> só então navegador envia requisição real. Sem resposta certa, navegador bloqueia antes da API.

**REQ-36 Cache preflight:** `Access-Control-Max-Age: 600` para não repetir preflight a cada chamada

**Código:** `CORSMiddleware` FastAPI com config do `config.py`

---
## Slide 13 - Security Headers: 8 Escudos
### REQ-37 a REQ-44 - Instruindo o navegador a se defender

**4 ataques mitigados:** XSS injeção scripts, clickjacking iframe invisível, sniffing de conteúdo, injeção recursos externos.

- REQ-37 CSP: `default-src 'self'; script-src 'self'; object-src 'none'; frame-ancestors 'none'` -> impede scripts externos e injeção. Crítico pois interface manipula PDFs e renderiza selos.
- REQ-38 HSTS: `max-age=63072000; includeSubDomains; preload` -> força HTTPS, impede downgrade para HTTP onde tokens transitariam em claro
- REQ-39 X-Frame-Options: DENY -> impede ser embutido em iframe de terceiros, bloqueia clickjacking que faria usuário assinar sem perceber
- REQ-40 X-Content-Type-Options: nosniff -> PDF malicioso não interpretado como HTML/script
- REQ-41 Referrer-Policy: strict-origin-when-cross-origin -> URLs internas com IDs não vazam para externos
- REQ-42 Permissions-Policy: `camera=(), microphone=(), geolocation=()` -> restringe APIs sensíveis não usadas
- REQ-43 Cache-Control: no-store nas rotas autenticadas -> impede cache de documentos/dados auth em navegador/proxy
- REQ-44 X-XSS-Protection: 0 -> desativa header legado que pode ser explorado, proteção real vem do CSP

Middleware `SecurityHeadersMiddleware` adiciona em toda resposta

---
## Slide 14 - Assinatura Digital de PDFs
### REQ-45 a REQ-49 - Autoria e integridade com selo visual + criptografia

**Fluxo (Figura 7):**
Valida PDF -> Hash do original SHA-256 -> Selo e assinatura -> Hash do assinado -> Registra no banco -> Devolve e descarta (não armazena PDF)

**REQ-45 Par de chaves por usuário:** no cadastro gera RSA-2048 ou ECDSA P-256. Privada armazenada criptografada com senha mestra do sistema (MASTER_KEY env) ou HSM em produção. Pública em PEM.

**REQ-46 Assinatura embutida:** segue padrão PDF Signature ISO 32000, não apenas imagem sobreposta. Implementado via PyPDF2 metadata `/Signature /Signer /SignedHash` + merge de overlay reportlab.

**REQ-47 Conteúdo selo visual:** nome assinante, data/hora UTC, hash documento (primeiros 16 chars), identificador único assinatura. Retângulo com fundo claro na posição X,Y escolhida.

**REQ-48 Registro banco:** `signatures` table: original_hash SHA-256, signed_hash, userId, timestamp UTC, IP (geolocalização), page_number, x, y

**REQ-49 Posição selo:** coordenadas X Y e número página registradas junto com assinatura, permite reconstituir onde selo estava

---
## Slide 15 - Verificação Pública de Autenticidade
### REQ-50 a REQ-54 - Qualquer pessoa verifica sem login, mas com proteção

**Tela pública é alvo fácil:** robôs enviando milhares PDFs para sondar, consumir recursos, inferir info. Precisa rate limiting mesmo sendo pública.

**Fluxo (Figura 8):**
Recebe PDF -> Há assinatura embutida? Se não -> Assinatura não encontrada
-> A assinatura confere criptograficamente? Se não -> Documento adulterado
-> Hash bate com registrado? Se não -> Adulterado, se sim -> Documento íntegro

**REQ-50 Verificação criptográfica:** confere assinatura embutida no PDF, não apenas presença do selo visual (selo pode ser falsificado)

**REQ-51 Comparação hashes:** hash do enviado vs hash registrado no momento assinatura

**REQ-52 Resultado claro:** 3 formas com UI distinta: íntegro verde check, adulterado vermelho alerta, não encontrada amarelo

**REQ-53 Rate limiting verificação:** mesma técnica REQ-05, 20 req/min por IP

**REQ-54 Descarte:** PDFs de verificação não armazenados, processamento em memória e descarte imediato -> LGPD

---
## Slide 16 - Sessões, Auditoria e Proteção de Dados
### REQ-55 a REQ-58 e Seção 9.2 - Rastreabilidade completa para investigação e disputa jurídica

**Por que log imutável?** 3 propósitos: investigar incidentes (acesso indevido), evidências em disputa jurídica sobre autenticidade, detectar anomalias (usuário assinando dezenas docs em segundos)

**REQ-55 Log imutável append-only:** eventos: login, logout, falha autenticação, emissão token, assinatura documento, verificação documento, troca senha. Só acrescenta, nunca altera/exclui.

**REQ-56 6 campos por registro:**
- userId: Por quem?
- Timestamp UTC: Quando?
- IP e User-Agent: De onde?
- Ação: O que aconteceu?
- Resultado: Deu certo ou falhou?

**REQ-57 Sessões ativas:** usuário visualiza histórico sessões ativas (id, ip_hash, created_at), encerra cada individualmente, opção "sair de todos dispositivos" revoga todas `revoked=True`

**REQ-58 Atividade segurança:** registra e mostra dentro da app, destaque para não vistos no próximo login: novo login IP desconhecido, ativação/desativação 2FA, troca senha

**Proteção dados 9.2:** PDFs nunca armazenados nem assinatura nem verificação, só hashes e metadados no PostgreSQL. Decisão motivada LGPD: dados pessoais retidos apenas tempo necessário finalidade. PDFs podem conter info sensível terceiros.

---
## Slide 17 - Validação de Entradas e Isolamento
### REQ-60 a REQ-63 - PDF é vetor crítico

**PDF é formato complexo:** pode conter scripts, fontes, formulários, objetos comprimidos, refs externas. Parser vulnerável pode levar a RCE.

**Filtros (Figura 9):**
Até 20MB -> Começa com %PDF- -> Sem JavaScript -> Processa PDF

- REQ-60 Magic number: valida bytes iniciais `%PDF-` antes de qualquer parsing
- REQ-61 Tamanho máximo: 20MB, rejeita antes de carregar em memória
- REQ-62 Processamento isolado: PDF processado em ambiente isolado sandbox/container sem acesso rede, mitiga exploits bibliotecas. Em docker-compose: serviço `pdf-sandbox` com `network_mode: none`
- REQ-63 PDF com JS rejeitado: busca chaves `/JS` e `/JavaScript` no dicionário PDF

**REQ-64 Entradas texto sanitizadas:** todas entradas nome, email validadas e sanitizadas contra XSS (escapa HTML) e SQL injection (bloqueia SELECT DROP -- ; UNION). Função `sanitize_text()` + `validate_email()`

---
## Slide 18 - CSRF e Dependências
### REQ-65 e REQ-66 - Complementando segurança dos cookies

**CSRF (Figura 10):** site malicioso engana navegador para fazer requisição autenticada sem consentimento. Funciona porque navegador envia cookies automaticamente em toda requisição para domínio, independente de origem.

**Cenário concreto:** usuário autenticado no sistema, sem logout acessa site malicioso em outra aba. Site tem formulário oculto que dispara POST para `/api/v1/documents/sign`. Navegador anexa cookie refresh_token sozinho. Sem proteção CSRF, servidor vê cookie válido e processa, assinando doc sem usuário saber.

**Por que refresh em cookie é vulnerável?** HttpOnly protege de XSS (script não lê), mas não de envio automático em requisições forjadas.

**REQ-65 Proteção CSRF:** Double Submit Cookie (cookie csrf_token legível JS + header X-CSRF-Token ou X-Request-ID com mesmo valor, comparado com hmac) ou verificação header Origin/Referer. Middleware `csrf.py`

**REQ-66 Dependências atualizadas e auditadas:** bibliotecas desatualizadas = vetor mais comum. Uma dependência vulnerável que lida com PDF ou criptografia compromete tudo. Projeto roda verificações automáticas `npm audit`, Dependabot, Snyk. `requirements.txt` e `package.json` mantidos atualizados.

---
## Slide 19 - Requisitos Não Funcionais e Arquitetura Código
### RNF-01 a RNF-17 - Qualidade e manutenibilidade

**Arquitetura (RNF-01 a 04):**
- RNF-01 Backend e frontend desacoplados, comunicação exclusiva pela API
- RNF-02 Backend API REST FastAPI/Flask/NestJS etc
- RNF-03 Frontend web Tailwind CSS React/Vue/Angular
- RNF-04 PostgreSQL relacional

**Variáveis ambiente RNF-05..08:**
- RNF-05 Configuração em .env nunca hardcoded
- RNF-06 .env.example com todas vars documentadas sem valores reais
- RNF-07 .env no .gitignore jamais commitado
- RNF-08 Validação na inicialização, recusa subir se ausente -> `Settings()` checa `REQUIRED_VARS`

**Migrações RNF-09..11:**
- RNF-09 Schema gerenciado exclusivamente por migrações versionadas Alembic/Flyway/Liquibase
- RNF-10 Proibido criar/alterar tabelas manualmente direto no banco
- RNF-11 Cada migration timestamp/sequencial com upgrade e downgrade

**Erros RNF-12..15:**
- RNF-12 Objeto padronizado {code, message, timestamp}
- RNF-13 Códigos por domínio AUTH_001 DOC_001 VAL_001
- RNF-14 Validação lista campos inválidos sem expor stack trace/SQL
- RNF-15 Status HTTP corretos 400 cliente 401 não autenticado 403 não autorizado 404 não encontrado 500 interno

**Versionamento RNF-16/17:** todos endpoints sob /api/v1/, incrementa versão em breaking changes

**Organização código (Figura 11):** Entrada Middlewares -> Rotas /api/v1 validação erros -> Serviços autenticação/sessões/assinatura/verificação/auditoria -> Infra Criptografia Argon2id/JWT/TOTP/chaves + Processamento PDF isolado + Persistência PostgreSQL migrações

---
## Slide 20 - Demonstração, Resultados e Próximos Passos
### Entrega, demo ao vivo e lições aprendidas

**Entrega 20/11/2026:**
- 2 repositórios privados backend e frontend, @ygoramaral colaborador, GitHub Project por app, issue por requisito com código no título, branch por issue, Ref #id no commit, merge na main
- README de cada repo explicando como executar
- Artefato interativo demonstrando todos REQs visualmente + código backend completo em Python

**Demo ao vivo (roteiro 10 min):**
1. Registro com senha fraca -> checklist REQ-01 falha, senha forte Argon2id sucesso
2. Login com 5 falhas -> bloqueio REQ-03 + rate limit REQ-05
3. Ativação 2FA: mostra QR REQ-13, backup codes REQ-14, login com TOTP REQ-15
4. Passkey: registra dispositivo, mostra counter REQ-08, autentica sem senha
5. Assina PDF: drag drop, validação magic %PDF- REQ-60, rejeita JS REQ-63, posiciona selo X Y página REQ-49, gera hashes REQ-48, baixa assinado
6. Verificação pública: upload íntegro -> verde, altera 1 byte -> adulterado vermelho, sem assinatura -> não encontrada
7. Sessões: lista, encerra individual, sair todos REQ-57, mostra log append-only REQ-55 com 6 campos REQ-56 e alertas segurança REQ-58
8. Mostra headers segurança REQ-37..44 e CORS REQ-31..36 no painel

**Próximos passos produção:**
- HSM real para chaves privadas REQ-45, Redis para rate limit/denylist, testes penetração, WAF, monitoramento anomalias assinatura em massa

**Lições:** segurança em camadas, centralizar verificações em middlewares evita esquecer em uma rota, JWT sem revogação é risco, PDF é vetor crítico precisa sandbox

**Tenha uma boa diversão!**

---
## Roteiro de Apresentação (15 minutos)

**Tempo total: 15 min + 5 min perguntas**

**[0-1min] Slide 1 Capa:**
"Bom dia, professor Ygor e colegas. Hoje apresento o Projeto 3 de Segurança da Informação: Sistema Web de Assinatura Digital de PDFs. O diferencial deste projeto é que segurança não é feature opcional, é requisito central. Lido com documentos com valor jurídico e identidade de pessoas. Entrega dia 20/11 com dois repos desacoplados."

**[1-2min] Slide 2 Problema:**
"Por que isso importa? Um selo visual sem criptografia pode ser falsificado no Photoshop. Precisamos garantir autoria e integridade. Além disso LGPD exige que não armazenemos PDFs, apenas hashes. E a tela pública de verificação é alvo de bots, então mesmo pública precisa rate limiting."

**[2-3min] Slide 3 Arquitetura:**
"Arquitetura em 3 partes: frontend Tailwind, API REST FastAPI, PostgreSQL. O ponto crucial da Figura 1: PDF atravessa API e volta ao usuário, mas nunca chega ao banco. Processamento em memória em container isolado sem rede, para mitigar RCE via parser."

**[3-4min] Slide 4 Objetivos:**
"4 objetivos: praticar auth moderna, manter sessão segura JWT, defender no navegador, aplicar assinatura a caso real. Processo com GitHub Project, 1 issue por REQ com código no título, branch por issue, Ref #id no commit, @ygoramaral colaborador."

**[4-5min] Slide 5 Senha:**
"REQ-01 política 12 chars maiúscula minúscula número símbolo, implementada com checklist visual. REQ-02 Argon2id RFC 9106 memory 64MB iter 3 parallel 4, nunca bcrypt. REQ-03 bloqueio temporário 5 falhas 15min. REQ-04 mensagem única evita enumeração. REQ-05 rate limit por IP e conta contra brute force e credential stuffing."

**[5-6min] Slide 6 Passkeys:**
"Passkeys eliminam phishing. Chave privada nunca sai do autenticador. Implementei registro com create e auth com get, guardo 4 campos REQ-07: credentialId, publicKey, counter, aaguid. REQ-08 validação counter crucial: se zero em ambos é passkey sincronizada iCloud/Google, senão recebido deve ser maior, se não é clonagem. rpId configurado com domínio exato."

**[6-7min] Slide 7 TOTP:**
"TOTP evita SIM swap. RFC 6238 janela 30s tolerância ±1. Segredo 160 bits criptografado com Fernet. QR Code na ativação compatível Google Authenticator, backup codes hash uso único. TOTP antes do JWT, token não sai na metade do fluxo. Alerta de códigos inválidos repetidos."

**[7-8min] Slide 8 Fluxo Login:**
"Fluxo da Figura 3: recebe login senha, dentro limite? senha confere? Se falhar incrementa failed_attempts e log falha, mas sempre mensagem genérica. Se 2FA ativado pede TOTP, só após validar tudo emite tokens. Cada falha gera audit log."

**[8-9min] Slide 9 JWT Fundamentos:**
"JWT sem estado é conveniente mas sem revogação. Solução: dois tokens. Access JWT RS256 15min no header Authorization, refresh opaco 256 bits CSPRNG 7 dias hash no banco em cookie HttpOnly Secure SameSite Strict, nunca localStorage. Servidor guarda só jti revogados e hash refresh."

**[9-10min] Slide 10 JWT Verificações:**
"Assinatura válida é só primeiro filtro. Verifico denylist REQ-22, iat vs troca senha REQ-25 rejeita tokens antigos, ua_hash REQ-27 rejeita se User-Agent diferente, ip hash REQ-26 gera alerta mas não rejeita, session_id REQ-28 permite revogar sessão específica. Tudo centralizado em middleware, não em cada rota."

**[10-11min] Slide 11 Rotação:**
"Cada refresh vale um uso só REQ-29. Quando usa, invalida anterior e emite novo par mesma família. Se mesmo refresh usado duas vezes, assume comprometido REQ-30 e invalida família inteira, todas sessões caem. Reduz janela exploração."

**[11-12min] Slide 12 CORS:**
"Sem CORS, site malicioso faz requisição em nome do usuário autenticado. Origens explícitas nunca * em autenticados, métodos restritos, headers restritos, credentials só confiáveis, responde preflight OPTIONS corretamente, Max-Age cache."

**[12-13min] Slide 13 Headers:**
"8 headers obrigatórios: CSP impede scripts externos crítico pois manipula PDFs, HSTS força HTTPS, X-Frame-Options DENY bloqueia clickjacking que faria assinar sem perceber, nosniff impede PDF interpretado como HTML, Referrer-Policy evita vazamento URLs internas, Permissions-Policy restringe câmera/microfone/geolocation, no-store impede cache docs, X-XSS-Protection 0 desativa legado."

**[13-14min] Slide 14 e 15 Assinatura e Verificação:**
"Assinatura: gera RSA-2048 por usuário privada criptografada master key, embutida ISO 32000 não só imagem, selo visual nome data UTC hash ID, registra hashes original e assinado userId timestamp IP posição X Y. Verificação pública: 3 resultados claro íntegro/adulterado/não encontrada, verifica criptografia não só visual, compara hashes, rate limiting mesmo pública, descarta PDF."

**[14-15min] Slide 16 a 20 Conclusão:**
"Sessões ativas lista encerra individual e sair todos, log append-only 6 campos para investigação e disputa jurídica, eventos segurança destaque novo IP 2FA troca senha. Validação PDF magic %PDF- 20MB rejeita JS sandbox sem rede sanitização XSS SQL, CSRF Double Submit + Origin. RNFs backend frontend desacoplados Tailwind PostgreSQL .env validado migrações Alembic erros padronizados /api/v1/. Demo ao vivo agora: vou registrar, ativar 2FA, assinar PDF posicionando selo, verificar integridade, mostrar adulteração detectada."

**Perguntas esperadas:**
- Por que não armazenar PDF? LGPD, responsabilidade desnecessária, basta hashes.
- Por que refresh em cookie HttpOnly? Protege XSS, mas precisa CSRF.
- O que acontece se counter passkey falhar? Rejeita, indica possível clonagem.
- Como garantir log imutável? Tabela append-only sem update/delete, apenas insert.

