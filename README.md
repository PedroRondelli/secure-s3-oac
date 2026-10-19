# S3 + CloudFront com Origin Access Control (OAC)

## Sumário

- [Introdução](#introdução)
- [Caso de Uso](#caso-de-uso)
- [Arquitetura](#arquitetura)
- [Boas Práticas Implementadas](#boas-práticas-implementadas)
- [Principais Propriedades dos Recursos](#principais-propriedades-dos-recursos)
- [Como Fazer o Deploy](#como-fazer-o-deploy)
- [Outputs](#outputs)

---

## Introdução

A adoção de **Infrastructure as Code (IaC)** tornou-se uma prática essencial no desenvolvimento moderno de software. Gerenciar infraestrutura manualmente — via console ou CLI ad hoc — gera ambientes difíceis de reproduzir, auditar e escalar. IaC resolve isso ao tratar a infraestrutura como código-fonte: versionável, revisável, testável e compartilhável entre times.

O **AWS CloudFormation** é o serviço nativo da AWS para IaC. Com ele, você descreve todos os recursos em um template (YAML ou JSON) e o CloudFormation se encarrega de criar, atualizar e destruir esses recursos de forma orquestrada, respeitando dependências e mantendo o estado desejado. Os principais ganhos são:

- **Reprodutibilidade**: o mesmo template gera ambientes idênticos em contas e regiões distintas.
- **Rastreabilidade**: cada mudança na infraestrutura passa pelo controle de versão do template.
- **Rollback automático**: em caso de falha durante um deploy, o CloudFormation reverte para o estado anterior sem intervenção manual.
- **Segurança auditável**: políticas de acesso e configurações de segurança ficam explícitas no código, facilitando revisões e conformidade.

---

## Caso de Uso

Este template provisiona a infraestrutura necessária para hospedar e expor uma **Single Page Application (SPA)** de forma segura e performática na AWS, utilizando:

- Um **bucket S3 privado** como origem dos arquivos estáticos da aplicação.
- Uma **distribuição CloudFront** como CDN global, responsável por entregar o conteúdo com baixa latência a usuários em qualquer região.
- Um **Origin Access Control (OAC)** para garantir que o bucket S3 seja acessível *somente* pelo CloudFront, nunca diretamente pela internet.
- Um **bucket de logs dedicado** para armazenar os logs de acesso tanto do S3 quanto do CloudFront.

O resultado é uma arquitetura serverless para frontends onde:

1. O usuário acessa a aplicação via HTTPS pelo domínio da distribuição CloudFront.
2. O CloudFront autentica a requisição na origem usando OAC (assinatura SigV4).
3. O S3 valida a assinatura e serve o arquivo. Sem OAC, o bucket rejeita qualquer acesso direto.
4. Erros 403 (acesso negado) são tratados retornando o `index.html`, garantindo que o roteamento do lado do cliente funcione corretamente em SPAs.

---

## Arquitetura

```
Usuário
   │
   │ HTTPS
   ▼
┌──────────────────────────────┐
│     CloudFront Distribution  │  ← CDN global, TLS 1.2+, redirect HTTP→HTTPS
│     (CFDistributionSPA)      │
└──────────┬───────────────────┘
           │ SigV4 (OAC)
           ▼
┌──────────────────────────────┐
│        S3 Bucket (Privado)   │  ← Arquivos da SPA, acesso público bloqueado
│        (S3Bucket)            │
└──────────┬───────────────────┘
           │ Logs
           ▼
┌──────────────────────────────┐
│      S3 Logging Bucket       │  ← Logs de acesso do S3 e do CloudFront
│      (LoggingBucket)         │
└──────────────────────────────┘
```

---

## Boas Práticas Implementadas

### Segurança

| Prática | Como está implementada |
|---|---|
| **Bloqueio total de acesso público ao S3** | `PublicAccessBlockConfiguration` com todas as flags em `true` nos dois buckets |
| **Acesso à origem exclusivo via OAC** | `BucketPolicy` permite `s3:GetObject` somente quando `AWS:SourceArn` corresponde à distribuição CloudFront específica |
| **Autenticação da origem com SigV4** | `CloudFrontOriginAccessControl` com `SigningBehavior: always` e `SigningProtocol: sigv4` — padrão mais moderno e seguro em substituição ao OAI |
| **Criptografia em repouso** | Ambos os buckets usam `SSE-S3 (AES-256)` via `BucketEncryption` |
| **Forçar HTTPS** | `ViewerProtocolPolicy: redirect-to-https` redireciona conexões HTTP para HTTPS |
| **Versão mínima de TLS** | `MinimumProtocolVersion: TLSv1.2_2021` impede uso de protocolos obsoletos |
| **Logs de acesso centralizados** | CloudFront e S3 enviam logs para um bucket dedicado, permitindo auditoria e análise de acessos |
| **Política de logging restrita** | `BucketPolicyForLogging` permite escrita no bucket de logs somente pelo serviço `logging.s3.amazonaws.com` da mesma conta |

### Reuso e Manutenibilidade

| Prática | Como está implementada |
|---|---|
| **Nome dinâmico via pseudo-parâmetro** | `!Sub 's3-oac-${AWS::AccountId}'` garante nomes únicos por conta, evitando conflitos ao reutilizar o template |
| **Referências internas (`!Ref` e `!GetAtt`)** | Recursos se referenciam entre si sem hardcode de ARNs ou nomes, facilitando mudanças e redeployments |
| **Uso de `!Sub` com partição AWS** | `arn:${AWS::Partition}:cloudfront::...` torna o template compatível com regiões GovCloud e China |
| **`DependsOn` explícito** | O OAC declara dependência do `S3Bucket`, garantindo a ordem correta de criação |
| **Cache Policy gerenciada** | Uso da `CachingOptimized` policy (ID fixo da AWS) evita criação de recursos extras e já aplica as melhores configurações de cache para arquivos estáticos |

---

## Principais Propriedades dos Recursos

### `S3Bucket` — Bucket principal da aplicação

| Propriedade | Descrição |
|---|---|
| `BucketName` | Nome gerado dinamicamente com o Account ID para garantir unicidade global |
| `PublicAccessBlockConfiguration` | Bloqueia todas as formas de acesso público, seja via ACL ou política de bucket |
| `LoggingConfiguration` | Define o bucket de destino e o prefixo dos logs de acesso do S3 |
| `BucketEncryption` | Habilita criptografia server-side com AES-256 para todos os objetos armazenados |
| `VersioningConfiguration` | Versionamento suspenso — pode ser habilitado para rollback de arquivos |

---

### `BucketPolicy` — Política de acesso do bucket principal

| Propriedade | Descrição |
|---|---|
| `Principal.Service` | Restringe o acesso ao serviço `cloudfront.amazonaws.com` |
| `Condition / AWS:SourceArn` | Limita ainda mais o acesso à distribuição CloudFront específica deste stack, evitando que outras distribuições na mesma conta acessem o bucket |
| `Action` | Permite apenas `s3:GetObject*` — somente leitura de objetos |

---

### `LoggingBucket` — Bucket de logs

| Propriedade | Descrição |
|---|---|
| `OwnershipControls / BucketOwnerPreferred` | Garante que o proprietário do bucket tenha controle total sobre os objetos de log escritos pelo serviço de logging do S3 |
| `DeletionPolicy: Delete` | Remove o bucket ao deletar o stack — adequado para ambientes de desenvolvimento |
| `BucketEncryption` | Os logs também são criptografados em repouso |

---

### `CFDistributionSPA` — Distribuição CloudFront

| Propriedade | Descrição |
|---|---|
| `Origins / S3OriginConfig / OriginAccessIdentity: ""` | Campo deixado vazio intencionalmente: ao usar OAC, o OAI (mecanismo legado) deve ser desativado |
| `OriginAccessControlId` | Vincula o OAC criado neste stack à origem S3 |
| `DefaultRootObject` | Define `index.html` como arquivo padrão ao acessar a raiz da distribuição |
| `CachePolicyId` | Referencia a política `CachingOptimized` gerenciada pela AWS, ideal para assets estáticos |
| `ViewerProtocolPolicy` | Redireciona HTTP para HTTPS automaticamente |
| `CustomErrorResponses` | Retorna `index.html` com status 200 para erros 403 — necessário para SPAs com roteamento client-side |
| `PriceClass` | `PriceClass_All` usa todos os edge locations do CloudFront globalmente |
| `ViewerCertificate / MinimumProtocolVersion` | Força TLS 1.2 como protocolo mínimo de comunicação |

---

### `CloudFrontOriginAccessControl` — OAC

| Propriedade | Descrição |
|---|---|
| `OriginAccessControlOriginType` | Especifica `s3` como tipo de origem — o OAC suporta outros tipos como `mediastore` e `lambda` |
| `SigningBehavior: always` | O CloudFront sempre assina as requisições para a origem, independentemente do cabeçalho de autorização do viewer |
| `SigningProtocol: sigv4` | Utiliza o protocolo de assinatura SigV4 da AWS, padrão atual para autenticação de requisições na AWS |

---

## Como Fazer o Deploy

Pré-requisito: AWS CLI configurada com permissões para criar os recursos do template.

```bash
# Criar o stack
aws cloudformation create-stack \
  --template-file stack-template.yaml \
  --stack-name secury-s3-oac \
  --capabilities CAPABILITY_NAMED_IAM

# Acompanhar eventos do stack
aws cloudformation describe-stack-events \
  --stack-name secury-s3-oac \
  --query 'StackEvents[*].[LogicalResourceId,ResourceStatus,ResourceStatusReason]' \
  --output table

# Remover o stack
aws cloudformation delete-stack \
  --stack-name secury-s3-oac
```

---

## Outputs

Após o deploy bem-sucedido, o stack expõe os seguintes valores:

| Output | Descrição |
|---|---|
| `BucketName` | Nome do bucket S3 onde os arquivos da SPA devem ser enviados |
| `SPADomain` | Domínio da distribuição CloudFront para acessar a aplicação no browser |

Para consultar os outputs via CLI:

```bash
aws cloudformation describe-stacks \
  --stack-name secury-s3-oac \
  --query 'Stacks[0].Outputs'
```
