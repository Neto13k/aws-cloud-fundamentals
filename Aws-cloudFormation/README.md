# aws-cloudformation-fundamentals

Documentação prática do laboratório de introdução ao AWS CloudFormation, realizado como parte do bootcamp na DIO. O objetivo foi entender como automatizar a criação de recursos na AWS por meio de templates.

---

## O que é AWS CloudFormation

CloudFormation é o serviço de **Infraestrutura como Código (IaC)** da AWS. Em vez de criar recursos manualmente pelo console, você descreve o que quer em um arquivo de template (YAML ou JSON) e a AWS provisiona tudo automaticamente.

**Vantagem principal:** o processo é repetível, versionável e auditável — você consegue recriar toda uma infraestrutura a partir de um arquivo.

---

## Conceitos principais

### Template

Arquivo de configuração que descreve os recursos a serem criados. Pode ser escrito em **YAML** (mais legível) ou **JSON**.

Estrutura básica de um template YAML:

```yaml
AWSTemplateFormatVersion: "2010-09-09"
Description: Minha primeira stack

Resources:
  MinhaInstancia:
    Type: AWS::EC2::Instance
    Properties:
      InstanceType: t2.micro
      ImageId: ami-xxxxxxxxxxxxxxxxx
```

### Stack

Uma stack é o conjunto de recursos criados a partir de um template. Quando você faz o deploy de um template, a AWS cria uma stack com todos os recursos definidos nele.

- Criar o template → fazer deploy → AWS cria a stack
- Deletar a stack → AWS remove todos os recursos associados automaticamente

### Stack de Firewall

No laboratório foi criada uma stack responsável por configurar múltiplas instâncias EC2 interligadas, com regras de firewall (Security Groups) aplicadas em conjunto. Isso demonstra como o CloudFormation gerencia dependências entre recursos — ele sabe a ordem certa de criação sem que você precise definir manualmente.

---

## YAML vs JSON

| | YAML | JSON |
|---|---|---|
| Legibilidade | Alta | Média |
| Suporte a comentários | Sim | Não |
| Uso mais comum | Templates CloudFormation | APIs e integrações |

Para templates CloudFormation, YAML é o formato preferido por ser mais fácil de ler e manter.

---

## Fluxo de trabalho

```
Escrever template (.yaml) → Upload no CloudFormation → Criar Stack → AWS provisiona recursos
                                                                    ↓
                                                         Monitorar eventos na aba Events
                                                                    ↓
                                                         Stack com status CREATE_COMPLETE
```

---

## Boas práticas observadas

- Sempre adicionar uma `Description` no template para documentar o propósito da stack
- Deletar stacks de teste após o uso — todos os recursos são removidos juntos, sem risco de esquecer algo rodando
- Versionar os arquivos de template no GitHub

---

## Referências

- [Documentação AWS CloudFormation](https://docs.aws.amazon.com/cloudformation/)
- [Referência de tipos de recursos](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-template-resource-type-ref.html)
- [Exemplos de templates](https://github.com/awslabs/aws-cloudformation-templates)