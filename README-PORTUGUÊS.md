# HealthProof — Infraestrutura Verificável de Dados de Saúde

HealthProof é uma **infraestrutura Web3 de verificação e interoperabilidade médica** que permite a pacientes, laboratórios e prestadores de saúde no Chile trocar documentos médicos de forma segura, preservando a **soberania do paciente, a confidencialidade dos dados e o cumprimento legal**. 

O protocolo combina **criptografia híbrida do lado do cliente (ECDH + AES-GCM), infraestrutura blockchain soberana (Avalanche L1) e armazenamento distribuído** para garantir que os registros médicos permaneçam privados, portáteis e interoperáveis entre instituições.

**Contexto legal:** HealthProof está projetado para cumprir com a **Lei 21.668** do Chile (publicada em 28 de maio de 2024), que modifica a Lei 20.584 para estabelecer a **interoperabilidade das fichas clínicas** entre prestadores públicos e privados. A lei exige que os prestadores adotem medidas que habilitem a interoperabilidade, garantam a continuidade do atendimento ao paciente e assegurem a proteção e conservação adequada dos registros por pelo menos 15 anos.

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

## 4. Compartilhar (Re-wrap)

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
- **Soberania de dados** em HealthProof se aplica na **camada de criptografia ECDH**, não na camada de wallet. Mesmo que a Privy desaparecesse, um paciente com sua chave privada ECDH (backup ou recuperada) ainda pode descriptografar seus documentos médicos.

## Futuro: Rota de Migração de Wallet

Uma característica planejada permitirá que os usuários que começaram com uma wallet embutida **vinculem e migrem para uma wallet externa** enquanto preservam sua identidade, permissões on-chain e histórico de dados criptografados.

---

# Smart Contracts

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

