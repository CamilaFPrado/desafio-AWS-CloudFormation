# AWS CloudFormation - Infraestrutura Automatizada

Este repositório documenta minha experiência prática com AWS CloudFormation, implementando infraestrutura como código (IaC) para automatizar a criação e gerenciamento de recursos na AWS.

## Índice
Visão Geral do Projeto
Arquitetura Implementada
Template CloudFormation
Implementação Passo a Passo
Anotações e Insights
Recursos Úteis

## Visão Geral do Projeto
Este laboratório prático teve como objetivo criar uma infraestrutura completa usando AWS CloudFormation, demonstrando o poder da infraestrutura como código para provisionamento automatizado e consistente de recursos.

## Arquitetura Implementada
A stack CloudFormation implementada inclui:

CloudFormation Stack
├── VPC com Subnets Públicas e Privadas
├── Internet Gateway
├── NAT Gateway
├── Security Groups
├── Instâncias EC2 (Auto Scaling Group)
├── Application Load Balancer
├── Banco de dados RDS (MySQL)
└── Bucket S3 para logs

## Template CloudFormation
Estrutura Principal do Template
```
AWSTemplateFormatVersion: '2010-09-09'
Description: Infraestrutura automatizada para aplicação web

Parameters:
  EnvironmentType:
    Description: Ambiente de deploy
    Type: String
    Default: dev
    AllowedValues:
      - dev
      - staging
      - prod

  InstanceType:
    Description: Tipo de instância EC2
    Type: String
    Default: t3.micro
    AllowedValues:
      - t3.micro
      - t3.small
      - t3.medium

Mappings:
  RegionMap:
    us-east-1:
      AMI: ami-0c02fb55956c7d316
    us-west-2:
      AMI: ami-0cd3dfa4e37921605

Conditions:
  IsProduction: !Equals [!Ref EnvironmentType, prod]

Resources:
  # VPC e Networking
  VPC:
    Type: AWS::EC2::VPC
    Properties:
      CidrBlock: 10.0.0.0/16
      EnableDnsSupport: true
      EnableDnsHostnames: true
      Tags:
        - Key: Name
          Value: !Sub ${AWS::StackName}-vpc

  # Subnets Públicas
  PublicSubnet1:
    Type: AWS::EC2::Subnet
    Properties:
      VpcId: !Ref VPC
      CidrBlock: 10.0.1.0/24
      AvailabilityZone: !Select [0, !GetAZs '']
      MapPublicIpOnLaunch: true

  # Internet Gateway
  InternetGateway:
    Type: AWS::EC2::InternetGateway

  # Security Group para Instâncias EC2
  WebServerSecurityGroup:
    Type: AWS::EC2::SecurityGroup
    Properties:
      GroupDescription: Security Group para servidores web
      VpcId: !Ref VPC
      SecurityGroupIngress:
        - IpProtocol: tcp
          FromPort: 80
          ToPort: 80
          CidrIp: 0.0.0.0/0
        - IpProtocol: tcp
          FromPort: 22
          ToPort: 22
          CidrIp: 0.0.0.0/0

  # Auto Scaling Group
  WebServerAutoScalingGroup:
    Type: AWS::AutoScaling::AutoScalingGroup
    Properties:
      VPCZoneIdentifier:
        - !Ref PublicSubnet1
      LaunchConfigurationName: !Ref WebServerLaunchConfig
      MinSize: 2
      MaxSize: !If [IsProduction, 6, 4]
      DesiredCapacity: 2
      TargetGroupARNs:
        - !Ref WebTargetGroup

  # Load Balancer
  WebLoadBalancer:
    Type: AWS::ElasticLoadBalancingV2::LoadBalancer
    Properties:
      Subnets:
        - !Ref PublicSubnet1
      SecurityGroups:
        - !Ref WebServerSecurityGroup

Outputs:
  WebsiteURL:
    Description: URL do Load Balancer
    Value: !Sub http://${WebLoadBalancer.DNSName}
  DatabaseEndpoint:
    Description: Endpoint do banco de dados RDS
    Value: !GetAtt Database.Endpoint.Address
```

## Implementação Passo a Passo
1. Preparação do Ambiente
```
# Configurar credenciais AWS
aws configure

# Validar template CloudFormation
aws cloudformation validate-template --template-body file://infrastructure.yaml
```

2. Implantação da Stack
```
# Criar stack
aws cloudformation create-stack \
  --stack-name my-infrastructure \
  --template-body file://infrastructure.yaml \
  --parameters ParameterKey=EnvironmentType,ParameterValue=dev \
  --capabilities CAPABILITY_IAM

# Monitorar criação
aws cloudformation describe-stacks --stack-name my-infrastructure

# Ver eventos da stack
aws cloudformation describe-stack-events --stack-name my-infrastructure
```

3. Testes e Validação
```
# Testar endpoints
curl $(aws cloudformation describe-stacks --stack-name my-infrastructure --query "Stacks[0].Outputs[?OutputKey=='WebsiteURL'].OutputValue" --output text)

# Verificar recursos criados
aws ec2 describe-instances --filters Name=tag:aws:cloudformation:stack-name,Values=my-infrastructure
```

4. Atualização e Gerenciamento
```
# Criar change set para atualização
aws cloudformation create-change-set \
  --stack-name my-infrastructure \
  --change-set-name my-change-set \
  --template-body file://infrastructure-v2.yaml

# Executar change set
aws cloudformation execute-change-set \
  --change-set-name my-change-set \
  --stack-name my-infrastructure

# Excluir stack
aws cloudformation delete-stack --stack-name my-infrastructure
```

## Anotações e Insights

Lições Aprendidas:
Vantagens do CloudFormation: Consistência: Infraestrutura idêntica em diferentes ambientes
Versionamento: Controle de versão dos templates
Replicabilidade: Implantação rápida em múltiplas regiões
Documentação: Template como documentação da infraestrutura


Melhores Práticas Implementadas:
Uso de Parameters para customização
Conditions para comportamentos diferentes por ambiente
Outputs para exportar informações importantes
Tags consistentes em todos os recursos


Desafios Superados:
Gerenciamento de dependências entre recursos
Depuração de erros de template
Limites de recursos por stack
Rollback automático em caso de falha


Padrões de Design:
Separação de concerns em nested stacks
Uso de custom resources para extensibilidade
Implementação de wait conditions para coordenação


Integrações Avançadas:
CloudFormation + CodePipeline para CI/CD
Integração com AWS Config para compliance
Uso do CloudFormation Registry para recursos de terceiros


## Próximos Passos
Para continuar evoluindo com AWS CloudFormation, planejo:

Explorar Nested Stacks para templates complexos
Implementar Custom Resources para recursos personalizados
Integrar com AWS CDK para desenvolvimento em linguagens de programação
Explorar CloudFormation Macros para transformações de template
Implementar Drift Detection para monitoramento de configuração




Este repositório será mantido como referência para futuros projetos e como material de estudo para certificações AWS.
