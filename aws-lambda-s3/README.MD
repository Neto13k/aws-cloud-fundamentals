# AWS Lambda + S3 — Integração e Automação

Estudo sobre integração entre AWS Lambda e Amazon S3, como parte do bootcamp na DIO. O objetivo é entender como funções serverless podem reagir a eventos de armazenamento e automatizar fluxos sem necessidade de servidores dedicados.

O bucket utilizado como referência neste estudo foi criado por mim via AWS SDK v3: [`aws-s3-bucket-creator`](https://github.com/Neto13k/aws-s3-bucket-creator).

---

## Índice

1. [O que é AWS Lambda](#1-o-que-é-aws-lambda)
2. [Modelo de execução serverless](#2-modelo-de-execução-serverless)
3. [Integração Lambda + S3](#3-integração-lambda--s3)
4. [Tipos de gatilho S3](#4-tipos-de-gatilho-s3)
5. [Anatomia de um handler Lambda](#5-anatomia-de-um-handler-lambda)
6. [Permissões: IAM Role para Lambda](#6-permissões-iam-role-para-lambda)
7. [Exemplo: processar arquivo ao fazer upload](#7-exemplo-processar-arquivo-ao-fazer-upload)
8. [Boas práticas](#8-boas-práticas)
9. [Limitações relevantes](#9-limitações-relevantes)
10. [Referências](#10-referências)

---

## 1. O que é AWS Lambda

Lambda é o serviço de computação serverless da AWS. Você sobe um trecho de código (função) e a AWS executa esse código em resposta a eventos — sem que você precise provisionar, configurar ou gerenciar servidores.

**O que você gerencia:**
- O código da função
- As permissões IAM associadas
- As configurações de memória, timeout e variáveis de ambiente

**O que a AWS gerencia:**
- Provisionamento de servidores
- Escalabilidade automática
- Alta disponibilidade
- Patches e manutenção da infraestrutura

---

## 2. Modelo de execução serverless

```
Evento ocorre → Lambda é invocada → Código executa → Resultado retornado → Recurso liberado
```

**Cobrança:** por número de requisições e por duração de execução (em ms), com 1 milhão de requisições e 400.000 GB-segundos gratuitos por mês no Free Tier.

**Runtimes disponíveis:** Node.js, Python, Java, Go, Ruby, .NET, e runtimes customizados via Lambda Layers.

| Característica | Valor |
|---|---|
| Timeout máximo | 15 minutos |
| Memória configurável | 128 MB a 10.240 MB |
| Tamanho do pacote (zip) | até 50 MB (250 MB descomprimido) |
| Armazenamento temporário (`/tmp`) | até 10 GB |

> **Ponto de atenção:** Lambda é stateless por design. Nenhum dado persiste entre execuções dentro da função. Para persistência, use S3, DynamoDB ou outro serviço externo.

---

## 3. Integração Lambda + S3

A integração entre Lambda e S3 funciona em dois sentidos:

### Lambda reage a eventos do S3 (trigger)

O S3 pode invocar automaticamente uma função Lambda quando algo acontece no bucket — um arquivo foi carregado, deletado ou modificado. É o padrão mais comum.

```
Upload de arquivo no S3
        ↓
S3 emite evento (s3:ObjectCreated)
        ↓
Lambda é invocada com os metadados do evento
        ↓
Função processa o arquivo (redimensiona imagem, extrai texto, valida formato, etc.)
        ↓
Resultado salvo de volta no S3 ou em outro serviço
```

### Lambda acessa o S3 diretamente (via SDK)

A função pode ler, escrever ou listar objetos no S3 usando o AWS SDK — sem necessidade de trigger. Útil quando Lambda é invocada por outro evento (API Gateway, SQS, etc.) e precisa interagir com armazenamento.

---

## 4. Tipos de gatilho S3

| Evento | Quando dispara |
|---|---|
| `s3:ObjectCreated:*` | Qualquer operação que crie um objeto (PUT, POST, COPY, multipart) |
| `s3:ObjectCreated:Put` | Apenas uploads via PUT |
| `s3:ObjectRemoved:*` | Qualquer deleção |
| `s3:ObjectRemoved:Delete` | Deleção permanente |
| `s3:ObjectRestore:*` | Restauração de objeto do Glacier |

**Configuração no console:**
1. Abrir o bucket S3 → aba **Properties**
2. Rolar até **Event notifications** → **Create event notification**
3. Selecionar o tipo de evento
4. Em **Destination**, escolher **Lambda function** e selecionar a função

> **Observação:** o S3 entrega o evento de forma assíncrona. A Lambda não retorna resposta diretamente para o S3 — ela processa e age por conta própria.

---

## 5. Anatomia de um handler Lambda

Todo código Lambda tem um ponto de entrada chamado **handler**. É a função que a AWS invoca a cada execução.

```javascript
// handler.js

export const handler = async (event, context) => {
  // `event`   → dados do gatilho (no caso do S3, contém bucket e chave do objeto)
  // `context` → metadados da execução (tempo restante, ID da requisição, etc.)

  console.log('Evento recebido:', JSON.stringify(event, null, 2));

  // Extraindo informações do evento S3
  const bucket = event.Records[0].s3.bucket.name;
  const key = decodeURIComponent(event.Records[0].s3.object.key.replace(/\+/g, ' '));

  console.log(`Arquivo recebido: ${key} no bucket: ${bucket}`);

  return {
    statusCode: 200,
    body: `Arquivo ${key} processado com sucesso.`
  };
};
```

**Estrutura do evento recebido pelo Lambda quando o S3 dispara:**

```json
{
  "Records": [
    {
      "eventSource": "aws:s3",
      "eventName": "ObjectCreated:Put",
      "s3": {
        "bucket": {
          "name": "neto13k-bucket-2026",
          "arn": "arn:aws:s3:::neto13k-bucket-2026"
        },
        "object": {
          "key": "uploads/documento.pdf",
          "size": 204800
        }
      }
    }
  ]
}
```

---

## 6. Permissões: IAM Role para Lambda

Lambda precisa de uma **IAM Role** para executar e acessar outros serviços. Sem a role correta, a função não consegue ler ou escrever no S3.

**Role mínima para Lambda ler e escrever no S3:**

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "s3:GetObject",
        "s3:PutObject"
      ],
      "Resource": "arn:aws:s3:::neto13k-bucket-2026/*"
    },
    {
      "Effect": "Allow",
      "Action": [
        "logs:CreateLogGroup",
        "logs:CreateLogStream",
        "logs:PutLogEvents"
      ],
      "Resource": "arn:aws:logs:*:*:*"
    }
  ]
}
```

> A permissão de logs (`CloudWatch Logs`) é necessária para que a função consiga registrar sua execução. Sem ela, você não enxerga nenhum output nem erro.

**Como criar a role:**
1. Console IAM → **Roles** → **Create role**
2. Em **Trusted entity**, selecionar **AWS service** → **Lambda**
3. Anexar a política acima (ou criar uma política customizada)
4. Na criação da função Lambda, selecionar essa role em **Execution role**

---

## 7. Exemplo: processar arquivo ao fazer upload

Cenário: toda vez que um arquivo `.json` for enviado ao bucket, a Lambda lê o conteúdo e loga as informações.

```javascript
// handler.js
import { S3Client, GetObjectCommand } from '@aws-sdk/client-s3';

const s3 = new S3Client({ region: process.env.AWS_REGION });

export const handler = async (event) => {
  const bucket = event.Records[0].s3.bucket.name;
  const key = decodeURIComponent(event.Records[0].s3.object.key.replace(/\+/g, ' '));

  // Só processa arquivos .json
  if (!key.endsWith('.json')) {
    console.log(`Arquivo ignorado (não é .json): ${key}`);
    return;
  }

  try {
    // Busca o objeto no S3
    const command = new GetObjectCommand({ Bucket: bucket, Key: key });
    const response = await s3.send(command);

    // Converte o stream para string
    const body = await response.Body.transformToString();
    const data = JSON.parse(body);

    console.log('Conteúdo do arquivo:', JSON.stringify(data, null, 2));
    console.log(`Arquivo ${key} processado com sucesso.`);

  } catch (error) {
    console.error(`Erro ao processar ${key}:`, error);
    throw error; // Re-lança para Lambda registrar como falha
  }
};
```

**Fluxo completo:**

```
Upload: uploads/dados.json → neto13k-bucket-2026
                ↓
     S3 emite evento ObjectCreated:Put
                ↓
        Lambda é invocada
                ↓
     GetObjectCommand busca o arquivo
                ↓
     Conteúdo logado no CloudWatch Logs
```

---

## 8. Boas práticas

**Permissões mínimas:** a role da Lambda deve ter acesso apenas ao bucket e às ações estritamente necessárias. Nunca usar `"Resource": "*"` em produção.

**Variáveis de ambiente:** não hardcode nomes de bucket ou configurações no código. Use variáveis de ambiente da Lambda:

```javascript
const bucket = process.env.BUCKET_NAME; // configurado nas env vars da função
```

**Tratamento de erros:** sempre re-lançar o erro após logar — isso permite que a AWS registre a falha e, se configurado, acione uma Dead Letter Queue (DLQ).

**Idempotência:** considere que a mesma função pode ser invocada mais de uma vez para o mesmo evento (entrega at-least-once). O código deve lidar com isso sem efeitos colaterais.

**Timeout adequado:** configurar o timeout com margem. Se a função normalmente termina em 3 segundos, configurar 10-15 segundos — não o máximo de 15 minutos.

---

## 9. Limitações relevantes

| Limitação | Detalhe |
|---|---|
| Execução stateless | Nenhum dado persiste entre invocações na memória da função |
| Cold start | Primeira invocação após inatividade tem latência adicional (pode ser centenas de ms) |
| Timeout máximo | 15 minutos — processos longos precisam de outra abordagem (ex: ECS, Step Functions) |
| Tamanho do evento | Payload máximo de 6 MB para invocação síncrona |
| Concorrência padrão | 1.000 execuções simultâneas por conta por região (limite aumentável via suporte) |
| Armazenamento `/tmp` | Temporário e não compartilhado entre instâncias — não use para estado persistente |

> **Cold start na prática:** funções Node.js e Python têm cold starts menores que Java ou .NET. Para workloads sensíveis à latência, Provisioned Concurrency mantém instâncias "quentes" — mas tem custo adicional.

---

## 10. Referências

- [Documentação AWS Lambda](https://docs.aws.amazon.com/lambda/)
- [Eventos S3 para Lambda](https://docs.aws.amazon.com/lambda/latest/dg/with-s3.html)
- [AWS SDK v3 — S3 Client](https://docs.aws.amazon.com/AWSJavaScriptSDK/v3/latest/client/s3/)
- [Boas práticas Lambda](https://docs.aws.amazon.com/lambda/latest/dg/best-practices.html)
- [Preços Lambda](https://aws.amazon.com/lambda/pricing/)
- [Lab relacionado — aws-s3-bucket-creator](https://github.com/Neto13k/aws-s3-bucket-creator)