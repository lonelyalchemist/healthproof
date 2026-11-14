# HealthProof — Infraestrutura Verificável de Dados de Saúde

HealthProof é uma **infraestrutura Web3 de verificação e interoperabilidade médica** que permite a pacientes, laboratórios e prestadores de saúde no Chile trocar documentos médicos de forma segura, preservando a **soberania do paciente, a confidencialidade dos dados e o cumprimento legal**. 

O protocolo combina **criptografia híbrida do lado do cliente (ECDH + AES-GCM), infraestrutura blockchain soberana (Avalanche L1) e armazenamento distribuído** para garantir que os registros médicos permaneçam privados, portáteis e interoperáveis entre instituições.

**Contexto legal:** HealthProof está projetado para cumprir com a **Lei 21.668** do Chile (publicada em 28 de maio de 2024), que modifica a Lei 20.584 para estabelecer a **interoperabilidade das fichas clínicas (prontuários médicos)** entre prestadores públicos e privados. A lei exige que os prestadores adotem medidas que habilitem a interoperabilidade, garantam a continuidade do atendimento ao paciente e assegurem a proteção e conservação adequada dos registros por pelo menos 15 anos.

---

# Tabela de Conteúdos

- [Visão](#visão)
- [Conformidade Legal (Chile)](#conformidade-legal-chile)
- [Resumo da Arquitetura](#resumo-da-arquitetura)
  - [Camada Cliente](#camada-cliente)
  - [Camada Protocolo](#camada-protocolo)
  - [Camada de Segurança e Criptografia](#camada-de-segurança-e-criptografia)
  - [Camada de Armazenamento](#camada-de-armazenamento)
  - [Camada Blockchain (Hygieia L1)](#camada-blockchain-hygieia-l1)
  - [Aprofundamento na Criptografia](#aprofundamento-na-criptografia)
- [Autenticação (Privy)](#autenticação-privy)
- [Contratos Inteligentes](#contratos-inteligentes)
- [Esquema do Banco de Dados](#esquema-de-banco-de-dados)
- [Meta-Transações (EIP-2771)](#meta-transações-eip-2771)
- [Guardião e Recuperação de Chaves](#guardião-e-recuperação-de-chaves)
- [Interoperabilidade e Nós Distribuídos (Roadmap)](#interoperabilidade-e-nós-distribuídos-roadmap)
- [Stack Tecnológico](#stack-tecnológico)
- [Estrutura do Monorepo](#estrutura-do-monorepo)
- [Variáveis de Ambiente](#variáveis-de-ambiente)
- [Início Rápido](#início-rápido)
- [Endereços de Contratos](#endereços-de-contratos)
- [Roadmap](#roadmap)

  ---

# Visão

Os sistemas de saúde atuais sofrem de problemas estruturais críticos:

- **Históricos clínicos fragmentados**: Um paciente com doença crônica tem seu histórico disperso em clínicas, laboratórios e hospitais.
- **Falta de interoperabilidade**: As instituições não podem compartilhar dados essenciais para garantir a continuidade do cuidado.
- **Controle limitado do paciente**: Os pacientes não decidem quem acessa seus dados sensíveis nem por quanto tempo.
- **Brechas de residência de dados**: Os dados médicos frequentemente saem do território nacional, violando a soberania e os requisitos legais.

HealthProof apresenta um novo paradigma:

> **Os pacientes se tornam os controladores soberanos de seus registros médicos por meio do domínio criptográfico de suas chaves, enquanto os prestadores de saúde podem verificar documentos médicos e interoperar sem depender de intermediários centralizados ou infraestrutura estrangeira.**

---

# Conformidade Legal (Chile)

## Lei 21.668 — Interoperabilidade das Fichas Clínicas (Prontuários Médicos)

Publicada no *Diario Oficial* em **28 de maio de 2024**, a Lei 21.668 modifica a Lei 20.584 (Direitos e Deveres das Pessoas em relação à Atenção à Saúde) para estabelecer: 

### Mandatos Legais-Chave

| Mandato | Implementação na HealthProof |
|---------|------------------------------|
| **Interoperabilidade entre prestadores** | Rede de nós distribuídos com replicação criptografada; cada prestador opera um nó que replica os dados aos quais está autorizado a acessar. |
| **Continuidade do atendimento ao paciente** | Concessão de permissões em tempo real através do `PermissionManager` on-chain; acesso de emergência para cenários críticos. |
| **Conservação de dados (15 anos)** | Cada nó se compromete com a replicação a longo prazo; `document_secrets` inclui o mapeamento de `storage_node` para rastreabilidade. |
| **Proteção e confidencialidade adequadas** | Criptografia ECDH + AES-GCM do lado do cliente; os documentos são ilegíveis para qualquer nó ou atacante sem a chave privada do paciente. |
| **Custódia por cada prestador** | Cada prestador (laboratório, clínica) armazena apenas os documentos que emite, cumprindo o requisito legal de que cada um custodia seus próprios registros. |
| **Acesso a dados essenciais para continuidade de atendimento** | O sistema de permissões permite aos médicos solicitar e aos pacientes conceder acesso a documentos específicos ou episódios clínicos completos. |

### Lacuna Regulatória Pendente 

O Ministério da Saúde do Chile está no processo de publicar a regulação técnica que detalha a "forma e condições" da interoperabilidade. A HealthProof se posiciona como uma  **implementação de referência** para um padrão distribuído, que preserva a privacidade e a soberania dos dados.

---

# Resumo da Arquitetura

HealthProof segue uma **arquitetura de protocolo em múltiplas camadas**, separando a interação clínica, a segurança criptográfica, o armazenamento distribuído e a verificação blockchain.

```
┌───────────────────────────────────────────────────────────────────────────┐
│                                CAMADA CLIENTE                             │
│  Next.js 16 App · Privy Auth · i18n (es/en) · Neumorphism UI              │
└───────────────────────────────────────────────────────────────────────────┘
                                      │
┌───────────────────────────────────────────────────────────────────────────┐
│                               CAMADA PROTOCOLO                            │
│  Identidade · Permissões · Documentos · Episódios · Ordens · Auditoria    │
└───────────────────────────────────────────────────────────────────────────┘
                                      │
┌───────────────────────────────────────────────────────────────────────────┐
│                     CAMADA DE SEGURANÇA E CRIPTOGRAFIA                    │
│  ECDH P-256 · AES-GCM · HKDF · IndexedDB · Backup/Recuperação de Chaves   │
└───────────────────────────────────────────────────────────────────────────┘
                                      │
┌───────────────────────────────────────────────────────────────────────────┐
│                         CAMADA DE ARMAZENAMENTO                           │
│  Blobs criptografados · IPFS/Pinata (atual) · Nós privados (futuro)       │
└───────────────────────────────────────────────────────────────────────────┘
                                      │
┌───────────────────────────────────────────────────────────────────────────┐
│                     CAMADA BLOCKCHAIN (Hygieia L1)                        │
│  IdentityRegistry · PermissionManager · GuardianRegistry ·                │
│  UUPS Proxies · EIP-2771 Meta-TX · Moeda nativa: HVE                      │
└───────────────────────────────────────────────────────────────────────────┘
```

## Camada Cliente

O **Cliente Clínico HealthProof** (`apps/frontend/healthproof-frontend`) fornece interfaces baseadas em identidades para:

- **Pacientes**: Ver documentos, conceder/revogar permissões, gerenciar tutores, escanear códigos QR.
- **Médicos**: Emitir ordens médicas, solicitar acesso a documentos, verificar permissões.
- **Laboratórios**: Receber ordens, enviar resultados de exames, compartilhar via QR.
- **Instituições / Centros Médicos**: Gerenciar pessoal, verificar participantes da rede.

Os usuários se autenticam por meio do **Privy** (OTP por e-mail, Google OAuth, ou wallets externas como MetaMask). O sistema gera automaticamente um par de chaves ECDH P-256 no primeiro login e armazena a chave privada no IndexedDB do navegador.

## Camada Protocolo

Os principais módulos de domínio residem em `src/features/` e `src/actions/`:

| Módulo | Responsabilidade |
|--------|-----------------|
| `features/permissions` | Geração de QR, construção de payloads de permissão, concessão/revogação on-chain |
| `features/documents` | Listagem de documentos, recuperação de metadados |
| `features/medical-orders` | Criar ordem, listar ordens, obter resultados |
| `actions/*-onchain.ts` | Ações do servidor que interagem com Hygieia via Wagmi + Viem |
| `actions/permissions/*` | Fluxo de convite (enviar/aceitar/rejeitar/cancelar), chaves de permissão |
| `actions/auth/*` | Busca de usuário, recuperação de chave pública, gestão de backups |

## Camada de Segurança e Criptografia

Veja [Aprofundamento na Criptografia](#aprofundamento-na-criptografia).

## Camada de Armazenamento

Atualmente, os documentos médicos criptografados são enviados para **Pinata IPFS** (rede IPFS pública). Essa é uma **arquitetura transicional**.

**Migração planejada (Fase 1):** Substituir Pinata com **armazenamento de objetos privado** (compatível S3 ou MinIO) hospedado no Chile para cumprir com a residência de dados.

**Visão a longo prazo (Fase 3):** Cada prestador de saúde opera seu próprio **nodo IPFS private-swarm / libp2p**. Os documentos são replicados através da rede nacional de saúde, mas apenas os blobs criptografados viajam; as chaves de descriptografia nunca saem do controle do paciente.

## Camada Blockchain (Hygieia L1)

**Hygieia** é a camada 1 própria da HealthProof sobre Avalanche (Subnet), implantada através da `avalanche-cli`.

| Parâmetro | Valor |
|-----------|-------|
| **ID da Chain** | `21668` |
| **Moeda nativa** | `HVE` |
| **VM** | Subnet-EVM | 
| **implantação** | Nó hospedado na AWS, RPC no Chile |
| **Consenso** | Avalanche (validadores de subnet proof-of-stake) |

### Por que uma L1 dedicada?

1. **Soberania**: O histórico de transações, o registro de identidades e a rastreabilidade de auditoria permanecem em infraestrutura controlada pela rede de saúde chilena.
2. **Desempenho**: Finalização rápida (~1s) e tarifas baixas, adequadas para o alto volume de ordens médicas e permissões.
3. **Conformidade**: Sem dependência de L1s estrangeiras; as auditorias regulatórias podem inspecionar a cadeia diretamente.
4. **Fundamento da interoperabilidade**: Os contratos inteligentes governam quais nós estão autorizados a se juntar à rede (`HealthcareNetworkRegistry`).

---

# Aprofundamento na Criptografia

HealthProof utiliza um **esquema de criptografia híbrida** que combina a **Web Crypto API** com identidade on-chain: 

## 1. Geração de Chaves (ECDH P-256)

Na primeira autenticação, o navegador gera um par de chaves ECDH P-256 não extraível:

```
Chave Pública  → exportada como JWK → guardada em DB (users.public_key)
Chave Privada  → CryptoKey não extraível → guardada em IndexedDB
```

A chave privada está marcada como `non-extractable` (Web Crypto), o que significa que nenhum JavaScript (incluindo XSS malicioso) pode exportá-la em formato raw.

## 2. Criptografar Documentos (AES-GCM)

Quando um laboratório envia um documento:

1. **Gerar chave de sessão AES-256-GCM** (aleatória, uma por documento).
2. **Criptografar o arquivo** com AES-GCM → produz ciphertext + IV.
3. **Encapsulamento da chave de sessão** para cada destinatário autorizado usando ECDH:
   - Derivar segredo compartilhado entre a chave privada do laboratório e a chave pública do paciente (ECDH).
   - Derivar chave de encapsulamento via HKDF-SHA-256.
   - Encapsulamento da chave de sessão AES → `encrypted_key`.
4. **Armazenamento**:
   - Blob criptografado → camada de armazenamento (IPFS / nó futuro).
   - Linha `document_secrets` → `document_id`, `uploader_wallet`, `patient_wallet`, `iv`, `encrypted_keys` JSONB.
  
## 3. Descriptografar Documentos

Quando um paciente (ou médico autorizado) baixa:

1. Obter o blob criptografado do armazenamento.
2. Obter a linha `document_secrets` → obter `iv` e `encrypted_keys[myWallet]`.
3. Usar a **chave privada ECDH do paciente** (da IndexedDB) para derivar o segredo compartilhado com a chave pública do uploader.
4. HKDF → desencapsular a chave de sessão AES.
5. AES-GCM descriptografa → documento original.

## 4. Compartilhar

Quando um paciente compartilha com um médico:

1. O paciente usa sua chave privada + a chave pública do médico para re-encapsular (re-wrap) a chave de sessão AES.
2. A nova `encrypted_key` é armazenada na tabela `permission_keys` e/ou é incluída em um código QR.
3. `PermissionManager.grantPermission` on-chain registra a autorização.

## 5. Respaldo e Recuperação de Chaves

Para prevenir o bloqueio ao trocar de dispositivo:

- No primeiro login, a chave privada ECDH é **exportada temporariamente** (como JWK), é criptografada com PBKDF2+AES-GCM usando uma chave derivada de `userId + SHAMIR_ENCRYPTION_KEY`, e é armazenada no DB como `users.encrypted_private_key`.
- Em um novo navegador/dispositivo: detectar IndexedDB vazio → tentar recuperação automática a partir do backup.
- **Prevenção de Conflito de Chaves**: Se o usuário tiver documentos criptografados (`document_secrets`), mas as chaves de IndexedDB não coincidirem, um `KeyConflictBanner` global avisa e bloqueia uploads/compartilhamentos até que seja resolvido.

> **Nota:** O verdadeiro **Shamir Secret Sharing** (criptografia de limiar dividindo a chave entre múltiplos guardiões) é uma **melhoria futura** planejada para integrar com `GuardianRegistry`. O backup atual é um armazenamento criptografado de uma única parte.

 ---
 
 # Autenticação (Privy)

A HealthProof migrou de Supabase Auth para **Privy** (`@privy-io/react-auth`).

## Métodos de Início de Sessão Disponíveis 

- **E-mail + OTP**: O usuário recebe um código único por e-mail. Privy cria automaticamente uma wallet Ethereum embutida baseada em MPC.
- **Google OAuth**: Início de sessão social com um clique; wallet embutida criada automaticamente.
- **Wallet Externa**: O usuário conecta uma wallet existente (MetaMask, Rainbow, etc.) via o modal de wallets do Privy.

## Wallets Embutidas (MPC)

```tsx
// providers.tsx
embeddedWallets: {
  ethereum: {
    createOnLogin: "users-without-wallets",
  },
}
```

As wallets embutidas de Privy usam **Multi-Party Computation (MPC)**: a chave privada é dividida em shards distribuídos entre Privy, o dispositivo do usuário e (opcionalmente) um serviço de recuperação. Nenhuma parte única possui a chave completa.

### Realidade Custodial vs. Não Custodial

- **Wallets embutidas**: Oferecem uma ótima UX (sem frase semente) mas o usuário não possui a chave privada completa.
- **Wallets externas** (MetaMask): São completamente não custodiais: o usuário possui a frase semente e a chave nunca toca os servidores da Privy.
- A **soberania de dados** da HealthProof se aplica na **camada de criptografia ECDH**, não na camada de wallet. Mesmo que a Privy desaparecesse, um paciente com sua chave privada ECDH (backup ou recuperada) ainda pode descriptografar seus documentos médicos.

## Futuro: Rota de Migração de Wallet

Uma característica planejada permitirá que os usuários que começaram com uma wallet embutida **vinculem e migrem para uma wallet externa** enquanto preservam sua identidade, permissões on-chain e histórico de dados criptografados.

---

# Contratos Inteligentes 

Todos os contratos são **UUPS Upgradeable Proxies** implantados em Hygieia (Avalanche L1, `chainId 21668`).

| Contrato | Função |
|----------|-----|
| `IdentityRegistry` | Identidade baseada em papéis (PATIENT, DOCTOR, LAB, INSTITUTION, ADMIN). `hasRole()`, `grantRole()`, `revokeRole()`. |
| `GuardianRegistry` | Designação de guardiões para pacientes. Usado para futura recuperação de chaves e delegação de acesso de emergência. |
| `PermissionManager` | Consultas de permissão O(1). `grantPermission()`, `revokePermission()`, `hasAccess()`. |
| `ClinicalEpisodeRegistry` | Agrupamento de ordens médicas e documentos sob um único episódio clínico. |
| `MedicalOrderRegistry` | Emissão de ordens médicas por médicos, atribuição a laboratórios. |
| `MedicalDocumentRegistry` | Registro de metadados de documentos (hashes, URIs). |
| `HealthcareNetworkRegistry` | Registro de instituições de saúde autorizadas e seus nós. |
| `AuditTrail` | Registro de eventos imutável: criação de ordem, upload de documento, concessão/revogação de permissões. |
| `HealthProofKernel` | Coordenador central do protocolo. |
| `HealthProofGateway` | Ponto de entrada proxy para meta-transações sem gás e ações com verificação de papel. |
| `HealthProofProtocol` | Orquestração de alto nível do protocolo. |
| `TrustedForwarder` | Forwarder de confiança EIP-2771 para meta-transações. |

### Meta-Transações (EIP-2771)

Os usuários assinam transações off-chain (sem gás). Um relayer (implantador ou serviço dedicado) as envia via `TrustedForwarder`. O contrato de destino extrai o endereço do signatário original da assinatura anexada.

Isso é essencial para pacientes que usam wallets embutidas e podem não possuir tokens HVE de gás.

---

# Esquema de Base de Dados

HealthProof usa **Supabase (PostgreSQL)** para dados estruturados. Papéis, permissões e eventos de auditoria vivem on-chain; o banco de dados armazena metadados de criptografia e perfis de usuário.

## Tabelas (Atual — 3 Tabelas)

### 1. `users`

```sql
id TEXT PRIMARY KEY,           -- Privy DID (did:privy:...)
wallet_address TEXT UNIQUE,   -- Endereço Ethereum
email TEXT,
full_name TEXT,
public_key TEXT,              -- JWK da chave pública ECDH
encrypted_private_key TEXT,   -- Backup criptografado PBKDF2+AES-GCM
created_at TIMESTAMPTZ
```

> `role` foi removido do DB e agora está **on-chain** em `IdentityRegistry`.

### 2. `document_secrets`

```sql
id UUID PRIMARY KEY,
document_id VARCHAR UNIQUE,       -- CID ou caminho de armazenamento
file_name TEXT,                   -- Nome legível por humanos
uploader_wallet TEXT REFERENCES users(wallet_address),
patient_wallet TEXT REFERENCES users(wallet_address),
iv TEXT,                          -- Vetor de inicialização AES-GCM
encrypted_keys JSONB,             -- { wallet_address: wrapped_key }
uploader_public_key TEXT,         -- JWK da chave pública do uploader no momento do upload
created_at TIMESTAMPTZ
```

### 3. `permission_keys`

```sql
id UUID PRIMARY KEY,
document_id TEXT REFERENCES document_secrets(document_id),
patient_wallet TEXT REFERENCES users(wallet_address),
grantee_wallet TEXT REFERENCES users(wallet_address),
encrypted_key TEXT,               -- Chave de sessão AES reencapsulada para o grantee
created_at TIMESTAMPTZ,
UNIQUE(document_id, grantee_wallet)
```

### Adições Futuras

- **Coluna `storage_node`** em `document_secrets`: rastreia qual nó físico (endpoint IPFS/libp2p) hospeda o blob criptografado.
- **Tabela `invitations`**: rastreia convites de permissão pendentes antes da execução on-chain (já implementada via server actions).

---

# Meta-Transações (EIP-2771)

```typescript
// 1. Usuário assina off-chain
const signed = await signMetaTransaction(
  walletClient,
  CONTRACT_ADDRESSES.PermissionManager,
  "grantPermission",
  [patientWallet, granteeWallet, scope, resourceId, expiry],
  PermissionManagerAbi
);

// 2. Servidor faz o relay  
const result = await grantPermissionOnChain({ request, ...payload });
```

**Benefícios:**
- Os pacientes não precisam de tokens HVE.
- Os usuários de wallet embebida experimentam UX sem gás.
- O relayer paga o gás, subsidiado pela rede ou pela instituição.

---

# Guardiões e Recuperação de Chaves

## GuardianRegistry (On-Chain)

Os pacientes podem designar **guardiões** (familiares de confiança ou médicos) que obtêm autoridade limitada:

- **Futuro**: Participar na **recuperação por umbral Shamir Secret Sharing** (ex. 2-de-3 guardiões necessários para reconstruir a chave privada ECDH do paciente, se ele perder todos os seus dispositivos).
- **Acesso de emergência**: Um guardião pode ativar uma solicitação de permissão de emergência com time-lock, registrada imutavelmente em `AuditTrail`.

## Prevenção de Conflito de Chaves

O sistema previne a regeneração acidental de chaves ECDH quando existem dados criptografados:

```
Chaves IndexedDB coincidem com DB public_key → OK
IndexedDB vazio + DB tem chave + documentos existem → conflito (missing_local_keys)
Chaves IndexedDB diferem + documentos existem → conflito (key_mismatch)
```

O `KeyConflictBanner` global bloqueia uploads/shares até que o usuário resolva o conflito via `KeyRecoveryModal`.

---

# Interoperabilidade e Nós Distribuídos (Roadmap)

Esta é a **visão da Fase 3-4** de HealthProof e aborda diretamente a **Lei 21.668**.

## Estado Atual (Fase 0)

- Documentos criptografados armazenados em **IPFS público via Pinata**.
- Todos os nós (frontend + backend) se conectam a um único provedor de armazenamento.
- Os dados saem do território chileno (DHT global de IPFS).

## Fase 1 — Armazenamento Privado no Chile

Substituir Pinata por **buckets compatíveis S3 ou MinIO** hospedados em servidores chilenos. Os blobs criptografados nunca saem do território nacional. Isso satisfaz o cumprimento imediato de residência de dados.

## Fase 2 — Endpoints de Propriedade Institucional

Cada clínica ou laboratório configura seu próprio endpoint de armazenamento:

```sql
ALTER TABLE document_secrets ADD COLUMN storage_node TEXT;
```

- O Laboratório A armazena seus documentos em `node-a.healthproof.cl`.
- O paciente acessa-os via o frontend de HealthProof, que roteia a solicitação de download para o nó correto.
- O nó apenas serve o **blob criptografado**; não pode descriptografá-lo.

## Fase 3 — Rede P2P Privada de Saúde

Implementar um **IPFS private swarm** ou **rede permisionada libp2p**:

- Apenas nós registrados em `HealthcareNetworkRegistry` podem se juntar.
- A identidade do nó está vinculada a um papel `INSTITUTION` on-chain.
- Os documentos criptografados são replicados através da rede (ex. 3 cópias: nó emissor + cache do paciente + arquivo regional).
- O conteúdo nunca é anunciado ao DHT público de IPFS.

## Fase 4 — Rede Nacional de Interoperabilidade

- Cada prestador de saúde certificado no Chile opera um nó HealthProof.
- O nó valida seus próprios dados (assinaturas digitais do emissor) e replica dados de pares sob SLAs codificados on-chain.
- Os pacientes têm uma **visão unificada única** de seu histórico clínico, agregada de todos os nós, descriptografada localmente com sua chave privada.
- O Ministério da Saúde pode auditar a rede via `AuditTrail` em Hygieia sem acessar os dados do paciente.

### Interoperabilidade sem Centralização

| Abordagem Tradicional | Abordagem Distribuída HealthProof |
|-----------------------|-----------------------------------|
| Banco de dados nacional único (monopólio, honeypot) | Cada prestador custodia seus próprios dados (cumpre Lei 21.668) |
| Exportações não criptografadas ou fracamente criptografadas | Criptografia end-to-end ECDH; os nós apenas armazenam ciphertext |
| Integrações ponto a ponto complexas HL7/FHIR | Protocolo padronizado via contratos inteligentes + replicação de blobs criptografados |
| O paciente não tem rastreabilidade de auditoria | Cada acesso registrado imutavelmente on-chain |

---

# Stack Tecnológico

## Frontend

| Tecnologia | Propósito |
|------------|-----------|
| Next.js 16 (App Router) | Framework React com Server Actions |
| TypeScript | Segurança de tipos |
| TailwindCSS + neumorfismo personalizado | Estilos de UI |
| Sileo | Notificações toast |
| `next-intl` | i18n (Espanhol / Inglês) |
| Zustand | Estado global (auth, permissões, UI, conflitos de chave) |
| React Query | Cache de estado do servidor |

## Web3

| Tecnologia | Propósito |
|------------|-----------|
| Privy (`@privy-io/react-auth`) | Autenticação + wallets embutidas/externas |
| Wagmi | Hooks React para Ethereum |
| Viem | Biblioteca moderna TypeScript para Ethereum (cliente de wallet, codificação) |
| `qrcode.react` | Geração de códigos QR para compartilhar permissões |

## Criptografia

| Tecnologia | Propósito |
|------------|-----------|
| Web Crypto API (`crypto.subtle`) | ECDH P-256, AES-GCM, HKDF, PBKDF2 |
| IndexedDB | Armazenamento de chave privada não extraível |

## Blockchain

| Tecnologia | Propósito |
|------------|-----------|
| Hardhat | Desenvolvimento de contratos inteligentes, testes, implantação |
| OpenZeppelin UUPS | Padrão de proxy atualizável |
| EIP-2771 | Padrão de meta-transação |
| Avalanche-CLI | Implantação de L1 / Subnet |

## Backend e Banco de Dados

| Tecnologia | Propósito |
|------------|-----------|
| Supabase (PostgreSQL) | Dados estruturados: users, document_secrets, permission_keys |
| Pinata (transicional) | IPFS pinning para blobs criptografados |
| Server Actions (Next.js) | Lógica de backend segura sem camada de API separada |

---

# Estrutura do Monorepo

```
healthproof/
├── apps/
│   ├── backend/              # (futuro) serviços backend dedicados
│   └── frontend/
│       └── healthproof-frontend/   # Aplicação Next.js 16
│           ├── src/
│           │   ├── app/[locale]/      # Roteamento i18n (es, en)
│           │   │   ├── auth/
│           │   │   ├── dashboard/
│           │   │   │   ├── documents/
│           │   │   │   ├── guardians/
│           │   │   │   ├── kernel/
│           │   │   │   ├── permissions/
│           │   │   │   ├── profile/
│           │   │   │   ├── share/
│           │   │   │   └── upload/
│           │   │   └── landing/
│           │   ├── actions/         # Server Actions (on-chain + DB)
│           │   ├── components/
│           │   ├── features/        # Módulos de domínio
│           │   ├── hooks/           # Hooks do React (auth, keys, sync)
│           │   ├── lib/             # Utilidades, env, contracts, metatx
│           │   ├── services/        # Criptografia, armazenamento, API
│           │   ├── state/           # Stores Zustand
│           │   └── types/           # Tipos de domínio e API
│           ├── messages/en.json
│           ├── messages/es.json
│           └── public/
├── infra/
│   └── avalanche/
│       ├── contracts/           # Smart contracts Solidity
│       │   ├── src/
│       │   └── test/
│       └── network/             # Configuração de subnet
├── packages/
│   ├── blockchain-sdk/          # Utilidades blockchain compartilhadas
│   ├── contracts/               # ABIs de contratos e scripts de despliegue
│   ├── did/                     # Utilidades DID
│   └── encryption/              # Helpers de criptografia compartilhados
├── docs/
│   └── architecture-debt-v1.2.md
└── supabase_migrations/         # Migrações SQL
```


---

# Variáveis de Ambiente

Crie um arquivo `.env` em `apps/frontend/healthproof-frontend/`:

```bash
# Autenticação Privy
NEXT_PUBLIC_PRIVY_APP_ID=seu-id-do-app-privy

# Supabase (apenas banco de dados — autenticação gerenciada pelo Privy)
NEXT_PUBLIC_SUPABASE_URL=https://seu-projeto.supabase.co
SUPABASE_SERVICE_ROLE_KEY=sua-chave-de-funcao-de-serviço

# Pinata / Armazenamento (Fase 0 — migrará para armazenamento privado)
PINATA_JWT_SECRET=seu-jwt-pinata
NEXT_PUBLIC_PINATA_GATEWAY=https://seu-gateway.mypinata.cloud

# Blockchain
NEXT_PUBLIC_RPC_URL=http://seu-nó-hygieia:9654/ext/bc/.../rpc
NEXT_PUBLIC_CHAIN_ID=21668

# Endereços de Contratos (Hygieia L1)
NEXT_PUBLIC_IDENTITY_REGISTRY=0x...
NEXT_PUBLIC_GUARDIAN_REGISTRY=0x...
NEXT_PUBLIC_PERMISSION_MANAGER=0x...
NEXT_PUBLIC_CLINICAL_EPISODE_REGISTRY=0x...
NEXT_PUBLIC_MEDICAL_ORDER_REGISTRY=0x...
NEXT_PUBLIC_MEDICAL_DOCUMENT_REGISTRY=0x...
NEXT_PUBLIC_HEALTHCARE_NETWORK_REGISTRY=0x...
NEXT_PUBLIC_AUDIT_TRAIL=0x...
NEXT_PUBLIC_HEALTHPROOF_KERNEL=0x...
NEXT_PUBLIC_HEALTHPROOF_GATEWAY=0x...
NEXT_PUBLIC_HEALTHPROOF_PROTOCOL=0x...
NEXT_PUBLIC_TRUSTED_FORWARDER=0x...

# Segredos do servidor
DEPLOYER_PRIVATE_KEY=0x...
SHAMIR_ENCRYPTION_KEY=sua-chave-shamir
```

---

# Início Rápido

## Pré-requisitos

- Node.js 20+
- pnpm (recomendado) ou npm
- Git

## Instalação

```bash
git clone https://github.com/jpgsChile/healthproof
cd healthproof
pnpm install
```

## Configurar Ambiente

```bash
cd apps/frontend/healthproof-frontend
cp .env.example .env
# Complete todas as variáveis necessárias
```

## Executar Servidor de Desenvolvimento

```bash
pnpm dev
# ou
npm run dev
```

Abra [http://localhost:3000](http://localhost:3000).

## Build

```bash
pnpm build
```

## Desenvolvimento de Contratos Inteligentes

```bash
cd infra/avalanche/contracts
npx hardhat compile
npx hardhat test
npx hardhat run scripts/deployHealthProofUUPS.ts --network hygieia
```

---

# Endereços de Contratos

## Hygieia L1 (Produção / Staging)

**RPC:** `http://3.141.110.34:9654/ext/bc/2qXqVm6f7B8LeMt4Gxa7V39LW8YVQiRuhzqH57Vaik9dD4VPRq/rpc`  
**Chain ID:** `21668`  
**Implantador:** `0x8db97C7cEcE249c2b98bDC0226Cc4C2A57BF52FC`

| Contrato | Endereço Proxy |
|----------|-----------------|
| IdentityRegistry | `0x68EA48917a3f9416613A48788BCe54578395a315` |
| GuardianRegistry | `0xE742a4b5F98453027fA3A9b0de106e237B6746B1` |
| PermissionManager | `0xb91b7959e715c059cE10eBEbe3288dA9d8012961` |
| ClinicalEpisodeRegistry | `0x207Ac23cf698ce54ad2AE2391be5df4b8c66430F` |
| MedicalOrderRegistry | `0x9E1222D98DBc740bbD406b5945084D363888CeA0` |
| MedicalDocumentRegistry | `0x5b190A85fb41D7C1d173a4501f12b81c28F59824` |
| HealthcareNetworkRegistry | `0x724820e539ae50f5fe7434fc9a6Cd2B17F832D31` |
| AuditTrail | `0x1AA001Cd20F35F3F4EF1A945053CeE4Acc24aDb4` |
| HealthProofKernel | `0xcad00692aa206527F64Fc683dB0f711dc49CB176` |
| HealthProofGateway | `0x68adE62397958E78A728313D27d3B1227Abf93d3` |
| HealthProofProtocol | `0x0D6F31D23704631417D8C55360aF2D493A578BF2` |
| TrustedForwarder | `0xC76413e3c098DC67cfdE4C2E92351792EC6924bf` |

> **Nota:** Os contratos são UUPS atualizáveis. Os endereços de implementação são rastreados separadamente.

---

# Roadmap

## Concluído

- [x] Sistema de criptografia híbrida ECDH P-256
- [x] Backup e recuperação de chaves (PBKDF2 + IndexedDB)
- [x] Prevenção de conflito de chaves (armazenamento Zustand + banner)
- [x] Autenticação Privy (e-mail, Google, carteira)
- [x] Implantação do Hygieia Avalanche L1
- [x] Conjunto de contratos inteligentes UUPS (Identidade, Permissão, Guardião, Episódio, Ordem, Registros de Documentos)
- [x] Meta-transações EIP-2771 (UX sem gás)
- [x] Dashboard baseado em identidades (Paciente, Médico, Laboratório, Instituição, Admin)
- [x] Fluxo de convite de permissões (enviar / aceitar / rejeitar / cancelar)
- [x] Compartilhar permissões por meio de código QR
- [x] Sistema de design Neumorfismo
- [x] i18n (Espanhol / Inglês)

## Em Progresso

- [ ] Corrigir registro on-chain de e-mail/OTP (atualmente apenas o login com Google registra corretamente)
- [ ] Migrar armazenamento de Pinata/IPFS público → armazenamento privado hospedado no Chile
- [ ] Adicionar coluna `storage_node` a `document_secrets`
- [ ] Implementar acesso de emergência (cumprimento com continuidade de atendimento urgente da Lei 21.668)

## Curto Prazo

- [ ] Vinculação e migração de carteiras (embutida → externa)
- [ ] Concessão de permissões multi-documento (seleção múltipla na UI)
- [ ] Notificações push para convites de permissão
- [ ] Fluxo de versionamento e correção de documentos
- [ ] Dashboard de auditoria aprimorado para instituições

## Médio Prazo

- [ ] Nós de armazenamento de propriedade institucional (Fase 2)
- [ ] IPFS private swarm / rede permisionada libp2p (Fase 3)
- [ ] Recuperação por limiar Shamir Secret Sharing usando `GuardianRegistry`
- [ ] App mobile (React Native) com proteção biométrica de chaves (WebAuthn / Secure Enclave)
- [ ] Integração com a regulamentação do Ministério da Saúde do Chile (uma vez publicada a norma técnica)

## Longo Prazo

- [ ] Onboarding nacional de prestadores de saúde (cada clínica/laboratório certificado opera um nó)
- [ ] Acordos de SLA on-chain para replicação de dados e conservação de 15 anos
- [ ] Aprendizado federado sobre dados médicos criptografados (diagnóstico com IA preservando a privacidade)
- [ ] Acordos de interoperabilidade transfronteiriça (corredores de dados de saúde MERCOSUL / Aliança do Pacífico)

---

# Contribuir

Damos as boas-vindas a contribuições de engenheiros de saúde, criptógrafos e desenvolvedores blockchain focados em **infraestrutura médica que preserve a privacidade de dados médicos**.

Por favor, abra um issue ou pull request em:  
https://github.com/jpgsChile/healthproof

---

# Licença

[Licença a definir]

---

> **Aviso legal:** HealthProof é um projeto de infraestrutura experimental. Ainda não está certificado para uso médico em produção no Chile. a Implantação clínica requer revisão pelo **Ministério da Saúde** e pela **Superintendência de Saúde** sob a futura regulamentação técnica da Lei 21.668.

> **Nota do Tradutor:** Esse repositório foi traduzido para português para servir como *sandbox* e referência técnica para desenvolvedores brasileiros interessados em testar soluções e arquiteturas compatíveis com o ecossistema de **Open Health** no Brasil.
