# Deploy automatizado com GitHub Actions, AWS e Docker

## AWS IAM

### Introdução
O IAM faz o gerenciamento da parte de identificação e acesso aos serviços da AWS. Para o correto funcionamento de todas as funcionalidades do deploy, vamos criar duas funções do IAM, uma para o CodeDeploy acessar o EC2, e outra para o EC2 utilizar o SSM (que concede acesso ao System Manager) e possibilitar a comunicação com o CodeDeploy. Pot último, vamos criar um usuário do IAM, para o GitHub acessar posteriormente o CodeDeploy.

<img src="https://user-images.githubusercontent.com/18728222/126000552-9a1e66c3-c485-4ae2-b603-8f6494f5b71b.png"/>

### Função para o CodeDeploy
Acesse o IAM, clique em Funções e em sequência Criar Função. Agora vamos executar as etapas:

- **Etapa 1 - Confiança**
  * **Selecionar tipo de entidade confiável**: Serviço da AWS
  * **Escolha um caso de uso**: EC2

- **Etapa 2 - Permissões**
  * **Attach políticas de permissões**: devem ser selecionadas as seguintes políticas:
    - *AdministratorAccess*
    - *AmazonEC2FullAccess*
    - *AmazonEC2RoleforAWSCodeDeploy*
    - *AWSCodeDeployRole*
  * **Definir limite de permissões**: Criar role sem limite de permissões   

- **Etapa 3 - Tags**
  * **Adicionar tags**: não é obrigatório, mas pode ser utilizado para facilitar a identificação e agrupamento da função na AWS.

- **Etapa 4 - Revisar**
  * **Nome da função**: CodeDeployRole
  * **Descrição da função**: Permite o acesso à EC2 a partir do CodeDeploy para execucao de todas etapas do deploy
  * **Entidades confiáveis**: Serviço da AWS: ec2.amazonaws.com
  * **Políticas**: *AdministratorAccess*, *AmazonEC2FullAccess*, *AmazonEC2RoleforAWSCodeDeploy*, *AWSCodeDeployRole*
  * **Limite de permissões**: Limite de permissões não definido

Por ultimo clique em *Criar a função*.

Na sequência abra a função criada, clique na aba *Relações de confiança* e em *Editar Relações de confiança*. No campo documento da política, insira o seguinte código:

```
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Service": "codedeploy.amazonaws.com"
      },
      "Action": "sts:AssumeRole"
    }
  ]
}
```
E por último, *Atualizar política de confiança*.

### Função para o EC2
Acesse o IAM, clique em Funções e em sequência Criar Função. Agora vamos executar as etapas:

- Etapa 1 - Confiança
  * **Selecionar tipo de entidade confiável**: Serviço da AWS
  * **Escolha um caso de uso**: EC2

- Etapa 2 - Permissões
  * **Attach políticas de permissões**: devem ser selecionadas as seguintes políticas:
    - *AmazonEC2RoleforAWSCodeDeploy*
    - *AmazonEC2RoleforSSM*
  * **Definir limite de permissões**: Criar role sem limite de permissões   

- Etapa 3 - Tags
  * **Adicionar tags**: não é obrigatório, mas pode ser utilizado para facilitar a identificação e agrupamento da função na AWS.

- Etapa 4 - Revisar
  * **Nome da função**: EC2CodeDeployRole
  * **Descrição da função**: Permite o acesso ao CodeDeploy e ao agente SSM a partir da EC2
  * **Entidades confiáveis**: Serviço da AWS: ec2.amazonaws.com
  * **Políticas**: *AmazonEC2RoleforAWSCodeDeploy*, *AmazonEC2RoleforSSM*
  * **Limite de permissões**: Limite de permissões não definido

Por ultimo clique em *Criar a função*.

Na sequência abra a função criada, clique na aba *Relações de confiança* e em *Editar Relações de confiança*. No campo documento da política, insira o seguinte código:

```
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Service": "ec2.amazonaws.com"
      },
      "Action": "sts:AssumeRole"
    }
  ]
}
```
E por último, *Atualizar política de confiança*.

### Usuário para o GitHub
Acesse o IAM, clique em Usuários e em sequência Criar Usuário. Agora vamos executar as etapas:

- Etapa 1 - Detalhes
  * **Nome do usuário**: GitHubCodeDeployUser
  * **Tipo de acesso**: Acesso programático

- Etapa 2 - Permissões
  * **Definir permissões**: Anexar políticas existentes de forma direta, e selecionar a política *AWSCodeDeployFullAccess*.
  * **Definir limite de permissões**: Criar user sem limite de permissões

- Etapa 3 - Tags
  * **Adicionar tags**: não é obrigatório, mas pode ser utilizado para facilitar a identificação e agrupamento da função na AWS.

- Etapa 4 - Revisar
  * **Detalhes do usuário**:
    - **Nome de usuário**: GitHubCodeDeployUser
    - **Tipo de acesso AWS**: Acesso programático: com uma chave de acesso
    - **Limite de permissões**: Limite de permissões não definido
  * **Resumo de permissões**:
    - **Política gerenciada**: AWSCodeDeployFullAccess

Por ultimo clique em *Criar o usuário*.

Na tela que segue, anote o ID da chave de acesso e a Chave de acesso secreta, para ser utilizado posteriormente no GitHub. Pode ser utilizado o botão *Fazer download .csv* para salvar os dados.

## AWS Parameter Store

### Introdução
Utilizaremos essa ferramenta para guardar os parâmetros da aplicação, para serem injetados no .env da aplicação durante o deploy, evitando que qualquer dado sensível fiquei exposto no código fonte da aplicação.

### Criando os parâmetros
Busque por Parameter Store na AWS, ele é um subserviço do AWS System Manager. Ao abrir clique em criar parâmetro.

Na criação do parâmetro existem vários campos para serem informados:

- **Nome**: informe o nome do parâmetro. Deixe ele igual a como seria no seu arquivo .env.
- **Descrição**: aqui pode ser informada uma descrição para o parâmetro.
- **Nível**: utilize o padrão para não gerar cobranças.
- **Tipo**: 
  * *String*: um valor qualquer.
  * *StringList*: vários valores que podem ser separados por vírgula.
  * *SecureString*: um valor qualquer que fica criptografado dentro do sistema usando chave KMS. Este tipo pode ser usado em dados muito sensíveis, como senhas e tokens.
- **Tipo de Dados**: para escolher o tipo de dados. Pode deixar selecionado text, ele serve para qualquer valor, seja texto ou numérico.
- **Valor**: o valor do parâmetro.
- **Tags**: pode ser informado tags para facilitar a localização e agrupamento dos parâmetros dentro da AWS.

## AWS EC2
### Introdução
Utilizaremos essa ferramenta para criação da uma instância Linux que será o servidor da nossa aplicação.

### Criando uma Instância
Acesse a guia Instâncias do EC2, e na tela de listagem que abriu, selecione *Executar instâncias*. Na tela que se segue, teremos um passo a passo para criação da Instância:

- **Etapa 1 - Selecione a AMI**
  * Selecione uma imagem Linux.
- **Etapa 2 - Escolher tipo de instância**
  * Escolha um tipo de instância adequada para o seu servidor.
- **Etapa 3 - Configurar instância**
  * **Número de instâncias**: 1
  * **Opção de compra**: deixar desabilitado
  * **Rede**: selecione uma VPC
  * **Sub-rede**: selecione uma Subnet ou deixe sem preferência
  * **Auto-assign Public IP**: Usar configuração de sub-rede (Habilitar)
  * **Grupo de posicionamento**: deixar desabilitado
  * **Reserva de capacidade**: Nenhuma
  * **Diretório de ingresso em domínio**: Nenhum diretório
  * **Função do IAM**: EC2CodeDeployRole
  * **Comportamento de desligamento**: Encerrar
- **Etapa 4 - Adicionar armazenamento**
  * Esolha um tipo/quantidade de armazenamento adequado para o seu servidor.
- **Etapa 5 - Adicionar Tags**
  * Adicione uma Tag para identificar a instância poteriormente, no CodeDeploy. Coloque como chave EC2CodeDeploy.
- **Etapa 6 - Configure o security group**
  * **Atribuir um grupo de segurança**: selecione um existente ou crie um novo.
- **Etapa 7 - Análise**
  * Revise os dados da sua instância, garantindo que seja criada a tag EC2CodeDeploy, para que encontremos essa instância posteriormente no CodeDeploy.
- **Etapa 8 - Executar**
  * Será solicitado para utilizar um par de chaves existente ou criar um novo. Escolha qual é a melhor opção para você.

### Configuração Básica
Assim que criada a instância, precisamos fazer algumas configurações na mesma para funcionar o deploy. Conecte a sua instância via SHH e executando o seguinte comando para atualizar o repositório de pacotes:

```
sudo apt update
```

E na sequência para executar o upgrade dos pacotes no sistema:

```
sudo apt upgrade -y
```

### Instalando o CodeDeploy Agent
Agora iremos instalar o agente do CodeDeploy. Para o mesmo funcionar, precisaremos instalar o Ruby na nossa instância:

```
sudo apt install ruby -y
```

Em seguida, caso não esteja instalando, o WGET:

```
sudo apt install wget -y
```

Agora vamos copiar o instalador do agente. Ele fica armazenado no bucket S3 da região da sua instância. Vamos usar como exemplo a região de São Paulo (sa-east-1):

```
wget https://aws-codedeploy-sa-east-1.s3.sa-east-1.amazonaws.com/latest/install
```

Torne o arquivo baixado executável:

```
chmod +x ./install
```

Execute o instalador:

```
sudo ./install auto
```

Em seguida, configure o serviço para sempre iniciar com o sistema:

```
sudo systemctl enable codedeploy-agent
```

E inicie o serviço:

```
sudo systemctl start codedeploy-agent
```

### Instalando o AWS Cli
Precisaremos também do AWS Cli para acessar o S3 e outros serviços da AWS.

Baixe a versão mais recente:

```
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
```

Descompacte:

```
unzip awscliv2.zip
```

E por fim instale:

```
sudo ./aws/install
```

### Instalando o Docker
Agora iremos instalar o Docker. Após a instalação, iremos configurar um grupo de usuário dele para poder executar o Docker sem o uso de super usuário.

Instale as ferramentas necessárias para rodar o Docker:

```
sudo apt install \
     apt-transport-https \
     ca-certificates \
     curl \
     gnupg \
     lsb-release -y
```

Adicione a chave GPG oficial:

```
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg
```

Agora vamos configurar um repositório para atualização estável:

```
echo \
  "deb [arch=amd64 signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/ubuntu \
  $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
```

Atualize o repositório de pacotes:

```
sudo apt update
```

E por ultimo, instale o Docker:

```
sudo apt install docker-ce docker-ce-cli containerd.io
```

Agora vamos alterar as configurações de usuário para executar o Docker sem ser super usuário:

Vamos criar o grupo docker:

```
sudo groupadd docker
```

E adicionaremos o usuário atual ao grupo:
```
sudo usermod -aG docker $USER
```

Agora, vamos aplicar as alterações:

```
newgrp docker 
```

Por ultimo, vamos configurar para o Docker sempre iniciar com o sistema:

```
sudo systemctl enable docker.service
```

### Instalando o Docker Compose
Para a instalação do Docker Compose, só precisamos baixar o binário do mesmo na versão desejada (aqui usaremos a 1.29.2) e alterar suas permissões de acesso.

Para baixar o binário, execute:

```
sudo curl -L "https://github.com/docker/compose/releases/download/1.29.2/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
```

E alterar suas permissões de acesso:

```
sudo chmod +x /usr/local/bin/docker-compose
```
## AWS Code Deploy

### Introdução
Utilizaremos essa ferramenta para fazer a implantação da aplicação, depois de ser feito algum commit no GitHub.

Podemos observar o workflow do CodeDeploy a seguir:

<img src="https://user-images.githubusercontent.com/18728222/126000662-7d9facd6-13ce-4b64-94c5-d5f7c5745d71.png"/>

Vamos fazer a configuração de cada etapa.

### Aplicativo
Acessa o CodeDeploy e clique em *Criar aplicativo*. Feito isso será abert uma tela de configuração de aplicativo, onde deverá ser preenchido os seguintes campos:

- **Nome do aplicativo**: GitDeployAutomated
- **Plataforma de computação**: EC2/On-premises

### Grupo de Implantação
Para criação de um grupo de implantação, abra o aplicativo criado na etapa anterior e clique em *Criar grupo de implantação*. Será aberta uma tela de criação do grupo, onde deverá ser preenchido os seguintes campos:

- **Nome do grupo de implantação**: GitDeployAutomatedGroup
- **Função de serviço**: CodeDeployRole
- **Tipo de implantação**: No local
- **Configuração do ambiente**: Instâncias do Amazon EC2
  * **Grupo de tags 1**: informe na chave, a tag da instância que configuramos anteriormente, *EC2CodeDeploy*.
- Configuração do agente com o AWS Systems Manager  
  * **Instalar o agente do AWS CodeDeploy**: *Agora e programar atualizações*.
  * **Programador Básico**: configure 10 dias.
- **Configurações de implantação**: selecione *CodeDeployDefault.OneAtATime*.
- **Load balancer**: não selecione *Habilitar balanceamento de carga*.

Agora clique em *Avançado - opcional*, e preencha a seguinte opção:
- **Reversões**: habilite Reverter quando uma implantação falhar

Finalize clicando em *Criar grupo de implantação*.

### Implantação
**ATENÇÃO**: execute esta etapa somente após executar a próxima etapa, *Configuração do Projeto*.

Agora vamos criar uma implantação que é a etapa que efitivamente copia o código fonte para a EC2 e executa as operações do deploy. Para tal, vamos acessar o grupo de implantação *GitDeployAutomatedGroup* recém criado. Ao abrir ele, vamos clicar em *Criar implantação*, onde iremos validar e preencher os campos:

- **Aplicativo**: GitDeployAutomated.
- **Grupo de implantação**: GitDeployAutomatedGroup.
- **Plataforma de computação**: EC2/On-premises.
- **Tipo de implantação**: No local.
- **Tipo de revisão**: selecione *Meu aplicativo está armazenado no GitHub*.
- **Nome do token do GitHub**: preencha com um nome para identificar seu repositório. preencha com GitHubActionsDeploy e clique em *Conectar ao GitHub*. Na tela que abre, confirme a ação.
- **Nome do repositório**: o nome do seu repositório no GitHub.
- **ID de confirmação**: preencha com o hash gerado no ultimo commit do seu repositório. Ele pode ser encontrado no seu histórico de commit, ao lado direito do seu último commit (tem um ícone de "copiar", pode clicar nele para copiar o hash completo).
- **Descrição da implantação**: informe uma descrição para essa implantação.

O restante dos campos deve ser deixado como está. Algumas configurações já vem pro padrão configuradas a partir do próprio grupo. Ao final, clique em *Criar implantação* para criar a primeira implantação manual.

Onde podemos verificar que o deploy está funcionando corretamente

## Configuração do Projeto
Agora iremos criar um projeto e criar os arquivos e scripts necessários para integração com o *CodeDeploy*.

O projeto terá a seguinte estrura:

<p align="center">
  <img src="https://user-images.githubusercontent.com/18728222/126000706-ddd3e936-b7f9-4d42-994c-eede270f55dd.png" />
<p/>

- **api**: pasta onde vai estar alocado o código da aplicação juntamente com o Dockerfile.
- **deploy**: pasta que contém os scripts do deploy da aplicação.
- **appspec.yml**: arquivo com as configurações do passo a passo para o *CodeDeploy* executar.

### AppSpec
Neste arquivo ficam as definições do deploy, definindo versão, sistema operacional, arquivos a serem copiados e hooks para disparar scripts em cada etapa da implantação:

<p align="center">
  <img src="https://user-images.githubusercontent.com/18728222/126000763-1e9e74eb-dac3-480a-898f-846cf1de2372.png" />
<p/>

- **version**: pode ser informado um número identificando a versão da configuração de deploy.
- **os**: sistema operacional para o deploy.
- **files**: possui dois campos para serem informado valores:
  * **source**: local de origem para cópia dos arquivos para deploy.
  * **destination**: local de destino na instância para salvar os arquivos do deploy.
- **hooks**: possui 4 campos para informar scripts em cada passo do deploy:
  * **BeforeInstall**: script executado antes da cópia dos arquivos para a máquina, onde fazemos operações como: instalação de aplicativos, criação de pastas e etc.
  * **AfterInstall**: script executado após cópia dos arquivos para a máquina, onde fazemos operações como: alteração de permissões de arquivos e/ou pastas, cópia de arquivos externos necessários para a aplicação, build da aplicação e etc.
  * **ApplicationStart**: script executado para início da aplicação, onde executamos uma operação para iniciar a aplicação.
  * **ApplicationStop**: script executado para parar a aplicação, caso for necessário. Se for informado, ele sempre será executado antes de iniciar um novo deploy.

Vamos utilizar o exemplo abaixo, sem o **ApplicationStop** pois não será necessária essa etapa ao utilizar Docker.

**appspec.yml**:

```
version: 1.0
os: linux
files:
  - source: /
    destination: /home/ubuntu/app
hooks:
  BeforeInstall:
    - location: deploy/before_install.sh
      timeout: 300
      runas: root
  AfterInstall:
    - location: deploy/after_install.sh
      timeout: 600
      runas: root
  ApplicationStart:
    - location: deploy/application_start.sh
      timeout: 300
      runas: root
```

Em cada script informamos um *timeout*, que é o tempo limite da operação, e um *runas*, que é usuário que executará as operações.

Os scripts serão criado agora no próximo passo.

### Scripts de Deploy
Como visto no passo anterior, podemos informar scripts em cada passo da operação. Vamos criar os scripts utilizados no exemplo anterior.

**before_install.sh**:

```
#!/bin/bash

cd /home/ubuntu

# Criamos a pasta para copiar a nossa aplicação

sudo mkdir -p app
```

**after_install.sh**:

```
#!/bin/bash

cd /home/ubuntu/app

# Removemos o arquivo .env antigo

sudo rm -f .env

# Injetamos o parâmetro salvo no AWS Parameter Store (some-env) na variável .env
# Repita o código para cada parâmetro

echo some-env=$(aws ssm get-parameters --output text --region sa-east-1 --names some-env --with-decryption --query Parameters[0].Value) >> .env

# Executamos o build da nossa aplicação com o docker-compose

docker-compose build --no-cache
```

**application_start.sh**:

```
#!/bin/bash

cd /home/ubuntu/app

# Subimos a nossa aplicação com o docker-compose

docker-compose up -d
```

Todos devem estar dentro da pasta deploy. Por fim, altere as permissões de acesso dos arquivos:

```
chmod +x before_install.sh
chmod +x after_install.sh
chmod +x application_start.sh
```

## GitHub Actions

### Introdução

Primeiramente iremos criar um workflow do *GitHub Actions* para gerar uma build da aplicação com o *Maven* e testá-la. Após esse processo, caso obtenha sucesso, será executado o deploy do código para o *CodeDeploy*.

### Configurando os Secrets
Antes de criarmos propriamente o workflow, vamos configurar alguns Secrets, variáveis do ambiente do repositório, com as credencias da AWS que cadastramos no *AWS AMI*.

No seu repositório, acesse *Settings*, e selecione a guia *Secrets*. Use a opção *New repository secret* para criar as variáveis, conforme a lista (Name/Value) abaixo:

- **AWS_REGION**: região da *AWS* que está sendo utilizada. Por exemplo, a região de São Paulo é sa-east-1
- **AWS_ACCESS_KEY_ID**: *Access key ID* do usuário GitHub cadastrado no *AWS AMI*
- **AWS_SECRET_ACCESS_KEY**: *Secret access key* do usuário GitHub cadastrado no *AWS AMI*

### Configurando um Workflow

Acessando a guia Actions do seu repositório no GitHub será exibido vários workflows prontos para serem utilizados. Vamos ignorar e clicar em *set up a workflow yourself*.

Será aberta uma nova tela como uma estrutura básica de workflow, e uma sugestão de nome para o arquivo.

Para configuração inicial do workflow, coloque o seguinte código:

```
name: CI/CD Pipeline

on:
  push:
    branches: [ main ]
```

- **name**: é o nome do workflow
- **on**: define quando vai ser disparada a action. No caso, no momento do *push* para a *branch* main.

Agora, vamos criar o código que vai executar a build do projeto e os testes dele, o *CI* do processo:

```
jobs:
  continuous-integration:
    runs-on: ubuntu-latest
    steps:
    - name: Checkout Repository to Workspace 🛒
      uses: actions/checkout@v2
    - name: Setup Java ☕
      uses: actions/setup-java@v2
      with:
        java-version: '11'
        distribution: 'adopt'
    - name: Caching Maven Packages 📦
      uses: actions/cache@v2
      with:
        path: ~/.m2
        key: ${{ runner.os }}-m2-${{ hashFiles('**/pom.xml') }}
        restore-keys: ${{ runner.os }}-m2  
    - name: Build and Test 🧪
      run: mvn -B test --file api/pom.xml
```

- **jobs**: declara que a partir desse ponto vai ser iniciada a declaraçãos dos serviços a serem executados.
- **continuous-integration**: pode ser informado qualquer nome, é a definição do nome do serviço.
- **runs-on**: versão do sistema operacional onde será executado o serviço.
- **steps**: declara que a partir desse ponto vai ser iniciado os passos do serviço a serem executados.
- **name**: indica o nome do passo do serviço.
- **uses**: faz a chamada de uma ação a ser executada. O formato da descrição é **owner**/**repository**@**version**.
- **with**: usado para informar parâmetros necessários da ação. Esses dados são informados no repositório dela.
- **run**: executa um comando dentro do container.

Os campos listados acima são padrões para tudo que for declarado dentro do workflow do GitHub Actions, como pode ser visto no código de deploy.

Por ultimo, vamos criar o código que vai executar a deploy do projeto, o *CD* do processo:

```
continuous-deployment:
    runs-on: ubuntu-latest
    needs: [continuous-integration]
    if: github.ref == 'refs/heads/main'
    steps:
      - name: Checkout Repository to Workspace 🛒
        uses: actions/checkout@v2
      - name: Configure AWS Credentials 🔐
        uses: aws-actions/configure-aws-credentials@v1
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: ${{ secrets.AWS_REGION }}
      - name: Create a Deployment to CodeDeploy 🚚
        run: |
          aws deploy create-deployment \
            --application-name GitDeployAutomated \
            --deployment-group-name GitDeployAutomatedGroup \
            --deployment-config-name CodeDeployDefault.OneAtATime \
            --github-location repository=${{ github.repository }},commitId=${{ github.sha }}
```

O código acima configura as credencias da *AWS*, com os *Secrets* criados anteriormente. Na sequência, cria um deploy para o *CodeDeploy*.
Três linhas do código você precisa informar os dados conforme a sua configuração no *CodeDeploy*:

- **--application-name**: nome do aplicativo cadastrado no *CodeDeploy*, no exemplo, GitDeployAutomated
- **--deployment-group-name**: nome do grupo de implantação, no exemplo, GitDeployAutomatedGroup
- **--deployment-config-name**: a configuração de implantação, no exemplo, CodeDeployDefault.OneAtATime

Ao final do processo, faça um commit, e acompanhe o processo na aba *Actions* do seu repositório.
