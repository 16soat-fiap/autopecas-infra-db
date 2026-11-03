# autopecas-infra-db

## Propósito

Infraestrutura do banco de dados gerenciado do Autopeças, separada do
repositório da Lambda e da aplicação. Usa PostgreSQL 16 em Amazon RDS,
Terraform, AWS provider e GitHub Actions. O alvo inicial é o AWS Academy Learner Lab.

## Tecnologias utilizadas

| Componente | Tecnologia |
| --- | --- |
| Banco gerenciado | Amazon RDS PostgreSQL 16 |
| Infraestrutura | Terraform e AWS provider |
| Rede e proteção | VPC, subnets, security groups e TLS |
| Estado da infraestrutura | S3 com chaves por ambiente e lockfile |
| Administração e permissões | SQL e psql |
| Validação | Testes Terraform simulados, Python 3 e PostgreSQL em Docker |
| Integração contínua | GitHub Actions para validação |

## Arquitetura

~~~mermaid
flowchart LR
  L[Lambda de autenticação] --> LS[Security group cliente Lambda]
  LS -->|TCP 5432| DB[(RDS PostgreSQL privado)]
  K[Aplicação Kubernetes] --> KS[Security group da aplicação]
  KS -->|TCP 5432| DB
  T[Terraform] --> S[State S3 separado por ambiente]
~~~

Este repositório cria a instância e o banco vazio, subnet group, parameter
group com TLS obrigatório e security groups. 

## Configuração

A rede existente deve fornecer sub-redes na mesma VPC em ao menos duas zonas.
Os exemplos incluem os IDs consultados no Lab em us-east-1; confirme-os antes
de usar, pois não são portáveis para outras contas. A instância RDS fica sem acesso público. Para integrar com o EKS, use os
outputs vpc_id e private_subnet_ids do repositório Kubernetes/rede, junto
com o SG real dos nós. Os IDs históricos do Lab não descrevem necessariamente
a rede atualmente provisionada.

O perfil de laboratório usa uma instância db.t3.micro, sem Multi-AZ,
com 20 GiB de armazenamento gp3 criptografado e sem crescimento automático.
Backups automáticos e snapshot final estão desativados.

A senha do administrador é fornecida por TF_VAR_db_password. Ela é sensível
na saída, mas fica no state e nos planos salvos. Não use essa conta na Lambda.
Não há criação de roles IAM nem integração obrigatória com Secrets Manager.

## Validação local

Requer Terraform >= 1.10 e < 2.0:

~~~bash
terraform -chdir=terraform init -backend=false
terraform -chdir=terraform fmt -check -recursive
terraform -chdir=terraform validate
terraform -chdir=terraform test
~~~

Os sete testes Terraform usam AWS simulada e não provisionam recursos.
Para verificar também os usuários e as permissões em PostgreSQL 16 real:

~~~bash
python3 scripts/test-permissoes.py
~~~

Requer Python 3 e Docker com containers Linux. Os sete testes SQL criam um
container descartável sem portas publicadas, sem acesso à rede externa e
sem usar credenciais AWS ou do RDS. O container é removido ao terminar.
O workflow validate.yml executa Terraform e os testes SQL em Pull Requests.
O deploy automático será configurado pelo integrante responsável pela pipeline.

## Preparar o plano de homologação

Renove as credenciais temporárias do Learner Lab no ambiente da AWS CLI.
O bucket autopecas-tfstate-376721682670 já existe, com criptografia, mas seu
versionamento foi habilitado no primeiro deploy em 22/10/2026. Este repositório
não assume a propriedade do bucket compartilhado e não altera sua configuração.
As chaves de state são db/homologacao/terraform.tfstate e
db/producao/terraform.tfstate; não se misturam com lambda/.

~~~bash
cp terraform/environments/homologacao.tfvars.example terraform/environments/homologacao.tfvars
cp terraform/environments/homologacao.s3.tfbackend.example terraform/environments/homologacao.s3.tfbackend
read -rs -p "Senha do administrador do banco: " TF_VAR_db_password; echo
export TF_VAR_db_password
export TF_DATA_DIR="$PWD/terraform/.terraform/homologacao"
terraform -chdir=terraform init -reconfigure -backend-config=environments/homologacao.s3.tfbackend
terraform -chdir=terraform plan -var-file=environments/homologacao.tfvars -out=homologacao.tfplan
~~~

Revise o plano antes de provisionar. Para criar os recursos na AWS:

~~~bash
terraform -chdir=terraform apply homologacao.tfplan
terraform -chdir=terraform output
~~~

Repita separadamente para produção, com sua configuração, state e senha.
O perfil atual exclui sem snapshot final. Para exigir snapshot, configure
skip_final_snapshot=false e um final_snapshot_identifier único.
Antes de remover o banco, desabilite deletion_protection por um apply prévio.
Sem backup nem snapshot, excluir o banco perde seus dados permanentemente.

## Produção preparada para o Lab

O ambiente `producao` tem arquivos e state separados. Ele usa o mesmo perfil
econômico da homologação: `db.t3.micro`, 20 GiB, Single-AZ, sem backups
automáticos ou crescimento do disco, com criptografia, TLS obrigatório e
proteção contra exclusão. Esse perfil é uma concessão do Lab.

| Item | Homologação | Produção |
| --- | --- | --- |
| Identificador RDS | autopecas-db-homologacao | autopecas-db-producao |
| State S3 | db/homologacao/terraform.tfstate | db/producao/terraform.tfstate |
| Diretório local Terraform | .terraform/homologacao | .terraform/producao |
| Senha administrativa | Exclusiva do ambiente | Outra senha, independente |

Na raiz do repositório, prepare produção sem sobrescrever arquivos existentes:

~~~bash
cp -n terraform/environments/producao.tfvars.example terraform/environments/producao.tfvars
cp -n terraform/environments/producao.s3.tfbackend.example terraform/environments/producao.s3.tfbackend
export TF_DATA_DIR="$PWD/terraform/.terraform/producao"
read -rs -p "Senha administrativa de producao: " TF_VAR_db_password; echo
export TF_VAR_db_password
terraform -chdir=terraform init -reconfigure -backend-config=environments/producao.s3.tfbackend
terraform -chdir=terraform plan -var-file=environments/producao.tfvars -out=producao.tfplan
~~~

A senha também pode vir de um arquivo local `producao.secrets.tfvars.json`,
com permissão 0600 e fora do Git, passado em um segundo `-var-file`.
Os arquivos locais e planos são ignorados pelo repositório; planos e state
contêm segredos e não devem ser publicados.

Em 25/10/2026, o plano real de produção foi validado: **7 recursos a criar,
0 a alterar e 0 a destruir**. Não foi executado apply de produção.
A preparação preserva a homologação e não cria um segundo RDS.
Os exemplos deixam o SG da aplicação vazio; informe a origem Kubernetes real;
quando informado, o Terraform verifica se ele pertence à VPC do banco.

## Documentação das APIs

Este repositório provisiona infraestrutura e não oferece API HTTP própria. As APIs de negócio pertencem à [aplicação principal](https://github.com/16soat-fiap/16SOAT-TechChallenge1).

- [Swagger UI da aplicação](http://localhost:8080/swagger-ui.html).
- [Especificação OpenAPI](http://localhost:8080/v3/api-docs).
- [Autenticação e coleção Postman da Lambda](https://github.com/16soat-fiap/autopecas-lambda-auth#readme).

Os endereços localhost exigem a aplicação em execução na porta 8080. Para acessar o Swagger de uma aplicação já publicada no EKS, com kubectl e o contexto do cluster configurados:

~~~bash
kubectl -n homologacao port-forward service/autopecas-api 8080:8080
~~~

Mantenha o comando aberto e acesse o Swagger pelo link local. Para produção, substitua o namespace por producao.

O catálogo atual do Gateway não publica /swagger-ui.html nem /v3/api-docs; por isso não se anuncia um endereço externo de Swagger que não esteja roteado.

## Integração com a Lambda e a aplicação

Outputs para o repositório da Lambda:

| Output | Variável na Lambda |
| --- | --- |
| db_url | db_url |
| lambda_subnet_ids | lambda_subnet_ids |
| lambda_security_group_ids | lambda_security_group_ids |

A role IAM da Lambda continua sendo configurada em seu próprio repositório.
A URL exige TLS com sslmode=require. A Lambda não precisa de acesso público
nem NAT para consultar o banco na mesma rede privada.
application_security_group_ids deve receber os SGs reais de origem do tráfego
Kubernetes, conforme a configuração de rede dos pods/nós.

Após o deploy, a aplicação principal deve executar suas migrations.
Depois, conecte via psql a partir de uma máquina com acesso à VPC e permissão
no SG. O computador local não tem acesso direto ao banco privado por padrão.
Use o proprietário das tabelas para executar:

~~~bash
psql "host=<endpoint> dbname=autopecas user=<proprietario> sslmode=require" -f scripts/criar-usuario-lambda.sql
~~~

O script solicita uma senha para lambda_auth e concede SELECT somente nas
colunas usadas pela autenticação. Execute em terminal interativo.
Forneça essa senha e db_user=lambda_auth à Lambda, sem usar o administrador.

## Usuário da aplicação e migrations

O usuário `autopecas_app` é separado do administrador e de `lambda_auth`.
Ele recebe SELECT, INSERT, UPDATE e DELETE nas tabelas de negócio e
USAGE/SELECT nas sequências. Não recebe DDL, acesso à tabela de roles
ou ao histórico do Flyway. Conceder roles de funcionários continua sendo
um procedimento administrativo, sem autopromoção pela aplicação.

Execute, em terminal interativo com acesso à VPC, depois das migrations e
da criação de `lambda_auth`:

~~~bash
psql "host=<endpoint> dbname=autopecas user=<proprietario> sslmode=require" -f scripts/criar-usuario-aplicacao.sql
psql "host=<endpoint> dbname=autopecas user=<proprietario> sslmode=require" -f scripts/verificar-permissoes.sql
~~~

O primeiro script usa `\password`: o psql solicita e cifra a senha, sem
colocá-la no arquivo SQL. O verificador rejeita roles administrativas,
propriedade herdada, DDL, permissões excessivas e falta de acesso aos dados.
Ele não revoga privilégios existentes silenciosamente; corrija explicitamente
qualquer permissão indevida identificada antes de usar o usuário.

| Configuração da aplicação | Valor |
| --- | --- |
| DB_URL | Output db_url, com TLS |
| DB_USERNAME | autopecas_app |
| DB_PASSWORD | Senha definida para esse usuário |
| SPRING_FLYWAY_ENABLED | false, após migrations executadas pelo proprietário |

As migrations continuam no repositório da aplicação e devem executar antes
do serviço, com uma identidade administrativa separada. O usuário de
execução não pode executar migrations. A configuração da aplicação e a
pipeline de deploy serão ajustadas pelos respectivos responsáveis.

Após novas migrations, revise a lista explícita de tabelas e sequências em
`permissoes-aplicacao.sql` e execute novamente esse arquivo e o verificador.
Isso não redefine a senha. Não há concessão automática de acesso a novas
tabelas, evitando expor tabelas sensíveis criadas no futuro.

Os scripts foram preparados e testados localmente. Sua presença no repositório
não significa que tenham sido executados em um banco; confirme os usuários e
privilégios no ambiente antes do deploy. As migrations pertencem à aplicação.

## Primeiro teste de homologação

Em 22/10/2026, o RDS privado foi provisionado, as migrations V1/V2 da aplicação
foram aplicadas com Flyway 11.14.1 e o usuário lambda_auth foi validado com
permissões restritas. A função temporária de migrations foi removida.
O perfil utilizado não tem Multi-AZ, backups, snapshot final ou crescimento
automático do disco. Os recursos provisionados consomem créditos do Lab.

## Registro histórico da homologação

Em 25/10/2026, a autenticação por CPF foi validada na AWS, com V1/V2/V4
aplicadas e `lambda_auth` somente para leitura. O RDS de homologação foi
reativado a pedido do usuário para testes manuais. Esse registro descreve a validação de 25/10/2026; não comprova o estado
atual dos recursos. Consulte a conta AWS e o state de cada ambiente para
verificar a infraestrutura existente.
