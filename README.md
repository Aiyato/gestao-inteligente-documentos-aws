# ☁️ Gestão Inteligente de Documentos — AWS Serverless

Arquitetura **serverless na AWS** para ingestão, armazenamento, recuperação e retenção histórica de documentos PDF, utilizando **Amazon API Gateway, AWS Lambda, Amazon S3, IAM e Amazon CloudWatch**.

O projeto foi desenvolvido para o cenário de uma startup de IA que recebe aproximadamente **50.000 novos arquivos por mês** e precisa manter todo o histórico de documentos sem tornar os custos de armazenamento inviáveis.

## 📌 Visão geral

A solução utiliza duas camadas de armazenamento:

- 🔥 **Camada quente:** Amazon S3 Standard durante os primeiros 12 meses, oferecendo acesso rápido aos documentos.
- ❄️ **Camada fria:** Amazon S3 Glacier Deep Archive após 365 dias, reduzindo significativamente o custo de armazenamento.
- 🔒 **Retenção histórica:** nenhum documento é excluído, preservando a base para futuros processos de treinamento e análise de IA.

A arquitetura segue o modelo **pay-per-use**, evitando servidores provisionados permanentemente.

## 🎯 Objetivos

- Permitir upload de arquivos PDF por meio de uma API.
- Validar tipo e tamanho dos arquivos antes do processamento.
- Isolar documentos por usuário.
- Gerar URLs pré-assinadas para acesso temporário.
- Manter o bucket S3 privado.
- Automatizar a transição de documentos antigos para armazenamento de baixo custo.
- Garantir observabilidade por meio do CloudWatch.
- Criar uma arquitetura escalável, segura e financeiramente sustentável.

## 🏗️ Arquitetura

```text
                    ┌─────────────────┐
                    │     Cliente     │
                    └────────┬────────┘
                             │
                             ▼
                    ┌─────────────────┐
                    │  API Gateway    │
                    │   HTTP / API     │
                    └────────┬────────┘
                             │
                             ▼
                    ┌─────────────────┐
                    │   AWS Lambda    │
                    │ Lógica de negócio│
                    └──────┬──────┬───┘
                           │      │
                  ┌────────┘      └───────────┐
                  ▼                           ▼
          ┌───────────────┐           ┌───────────────┐
          │   Amazon S3   │           │  CloudWatch   │
          │                │           │ Logs/Métricas │
          │  S3 Standard  │           └───────────────┘
          │      ↓         │
          │  Lifecycle     │
          │      ↓         │
          │ Glacier Deep   │
          │    Archive     │
          └───────────────┘
                  ▲
                  │
          ┌───────────────┐
          │      IAM      │
          │ Segurança e   │
          │ permissões    │
          └───────────────┘
```

## ☁️ Serviços AWS utilizados

### Amazon API Gateway

Responsável por funcionar como porta de entrada da aplicação, disponibilizando um endpoint HTTP gerenciado e escalável para receber as requisições.

**Modelo de cobrança:** baseado principalmente no número de requisições e, quando aplicável, no volume de dados transferidos.

### AWS Lambda

Executa a lógica de negócio sob demanda, sem necessidade de manter servidores ativos continuamente.

Utilizada para:

- Processamento das requisições.
- Validação dos arquivos.
- Controle de acesso.
- Geração de URLs pré-assinadas.
- Integração com o S3.

**Modelo de cobrança:** número de execuções e duração das funções, considerando a memória alocada.

### Amazon S3

Responsável pelo armazenamento dos documentos.

Estrutura sugerida para as chaves:

```text
usuario-123/documento.pdf
usuario-456/documento.pdf
```

A organização por prefixo facilita o isolamento dos documentos entre usuários.

O S3 também é utilizado para implementar o **Lifecycle Management**, movendo automaticamente documentos antigos para uma classe de armazenamento mais econômica.

### AWS IAM

Responsável pelo controle de permissões e pelo princípio do **menor privilégio**.

A Lambda recebe permissões específicas para trabalhar somente com os recursos necessários, reduzindo a possibilidade de acesso indevido aos documentos.

### Amazon CloudWatch

Utilizado para:

- Logs das funções Lambda.
- Monitoramento de requisições.
- Identificação de erros.
- Acompanhamento de desempenho.
- Auditoria das operações críticas.

## 🔄 S3 Lifecycle

A principal estratégia de otimização de custos é o uso do **S3 Lifecycle**.

```text
Upload
  │
  ▼
S3 Standard
  │
  │ 0 — 365 dias
  │
  ▼
S3 Glacier Deep Archive
  │
  │ 365+ dias
  │
  ▼
Retenção histórica
```

Após **365 dias**, os objetos são automaticamente movidos para o **S3 Glacier Deep Archive**.

Nenhum arquivo é excluído, garantindo a retenção do histórico necessário para futuras aplicações de IA.

## 🔐 Segurança

A arquitetura considera os seguintes mecanismos:

- **S3 Block Public Access** habilitado.
- Bucket privado.
- URLs pré-assinadas para acesso temporário.
- Controle de permissões utilizando IAM.
- Princípio do menor privilégio.
- Isolamento dos documentos por usuário.
- Criptografia em trânsito utilizando HTTPS/TLS.
- Criptografia em repouso utilizando SSE no S3.
- Logs e auditoria por meio do CloudWatch.

### Isolamento por usuário

Cada documento utiliza o ID do usuário como prefixo:

```text
usuario-123/documento.pdf
```

Dessa forma, a aplicação pode restringir o acesso de cada usuário aos seus próprios objetos.

## 📋 Requisitos funcionais

| ID | Requisito | Prioridade |
|---|---|---|
| RF01 | Permitir upload de arquivos PDF através do API Gateway | Alta |
| RF02 | Validar tipo e tamanho do arquivo | Alta |
| RF03 | Registrar logs de upload, acesso e erros | Alta |
| RF04 | Armazenar documentos utilizando o ID do usuário como prefixo | Alta |
| RF05 | Permitir recuperação dos documentos recentes | Alta |
| RF06 | Gerar URLs pré-assinadas para acesso temporário | Alta |
| RF07 | Impedir acesso a documentos de outros usuários | Alta |
| RF08 | Mover automaticamente arquivos com mais de 365 dias para armazenamento frio | Alta |
| RF09 | Manter todos os documentos sem exclusão automática | Alta |
| RF10 | Permitir consulta de métricas e logs através do CloudWatch | Média |

## ⚙️ Requisitos não funcionais

### Segurança

- S3 Block Public Access habilitado.
- Controle de acesso via IAM.
- Criptografia em trânsito e em repouso.

### Custo

A arquitetura utiliza serviços serverless e armazenamento em camadas para reduzir custos conforme o volume de dados cresce.

### Escalabilidade

API Gateway e Lambda aproveitam o escalonamento automático da AWS para lidar com picos de requisições.

### Disponibilidade

A solução utiliza serviços gerenciados da AWS, reduzindo a necessidade de administração de infraestrutura própria.

### Observabilidade

Operações críticas são registradas no CloudWatch para facilitar auditoria e troubleshooting.

## 🏛️ AWS Well-Architected Framework

A solução foi analisada considerando os seis pilares do AWS Well-Architected Framework:

| Pilar | Aplicação |
|---|---|
| 🛠️ Operational Excellence | CloudWatch Logs e monitoramento |
| 🔐 Security | IAM, criptografia e Block Public Access |
| 🛡️ Reliability | S3 gerenciado e arquitetura escalável |
| ⚡ Performance Efficiency | Serverless e auto scaling |
| 💰 Cost Optimization | Pay-per-use e S3 Lifecycle |
| 🌱 Sustainability | Uso eficiente dos recursos |

## 💰 Estimativa de custos

As estimativas foram baseadas nas seguintes premissas:

- **50.000 novos arquivos/mês**
- Tamanho médio: **800 KB por arquivo**
- Aproximadamente **100.000 requisições de leitura/mês**
- Operação considerada em estado estável após 3 anos
- Uma única região AWS
- Retenção de logs no CloudWatch: 30 dias

### Estimativa mensal

| Serviço | Custo estimado |
|---|---:|
| Amazon API Gateway | US$ 0,00 |
| AWS Lambda | US$ 0,20 |
| Amazon S3 Standard | US$ 10,53 |
| S3 Glacier Deep Archive | US$ 0,91 |
| Data Transfer | US$ 6,84 |
| Amazon CloudWatch | US$ 3,75 |
| **Total estimado** | **US$ 22,41/mês** |

> ⚠️ Os valores são estimativas e podem variar de acordo com a região AWS, volume de utilização, comportamento dos usuários e alterações na tabela de preços.

## 📊 Análise de custos

O armazenamento representa o principal componente do custo da arquitetura.

A estratégia de mover documentos com mais de 365 dias para o Glacier Deep Archive permite manter o histórico completo enquanto reduz o custo por GB armazenado.

O processamento com API Gateway e Lambda permanece praticamente gratuito nesse volume de utilização, reforçando a vantagem do modelo serverless para uma aplicação com crescimento variável.

## 📈 Escalabilidade

O cenário considera aproximadamente:

```text
50.000 arquivos novos / mês
        ↓
   600.000 / ano
        ↓
~1,4 TB acumulados em 3 anos
```

A arquitetura foi projetada para acompanhar esse crescimento sem exigir o provisionamento manual de servidores.

## 🚀 Benefícios da solução

- ✅ Arquitetura 100% serverless.
- ✅ Escalabilidade automática.
- ✅ Pagamento conforme utilização.
- ✅ Armazenamento altamente durável.
- ✅ Retenção histórica sem exclusão dos documentos.
- ✅ Otimização automática de custos.
- ✅ Controle granular de acesso.
- ✅ Monitoramento e auditoria.
- ✅ Redução da necessidade de gerenciamento de infraestrutura.

## 📁 Estrutura conceitual do armazenamento

```text
S3 Bucket
│
├── usuario-001/
│   ├── documento-001.pdf
│   └── documento-002.pdf
│
├── usuario-002/
│   ├── documento-003.pdf
│   └── documento-004.pdf
│
└── usuario-003/
    └── documento-005.pdf
```

## 🧰 Tecnologias

- ☁️ Amazon Web Services (AWS)
- 🪣 Amazon S3
- ⚡ AWS Lambda
- 🌐 Amazon API Gateway
- 🔐 AWS IAM
- 📊 Amazon CloudWatch
- ♻️ S3 Lifecycle
- ❄️ S3 Glacier Deep Archive
- 🏗️ Serverless Architecture

## 🎓 Objetivo do projeto

Este projeto foi desenvolvido como um **case de arquitetura em Cloud Computing**, com foco na criação de uma solução AWS escalável, segura e otimizada para custos.

O projeto demonstra conhecimentos em:

- Arquitetura Serverless
- Amazon S3
- AWS Lambda
- API Gateway
- IAM
- CloudWatch
- S3 Lifecycle
- Otimização de custos na AWS
- Segurança em Cloud
- AWS Well-Architected Framework

## 👥 Autores

- Andrea Alvares Duran
- **Brenan Ulisses Araújo**
- João Marcos Lopes de Oliveira
- Julia Kethilyn da Silva Vieira
- Mayk Ferreira
- Yuri Santiago Borges

---

⭐ **Projeto desenvolvido com foco em Cloud Computing, AWS e arquitetura Serverless.**
