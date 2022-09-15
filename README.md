# Implementação Conteiner Docker: Jenkins + SonarQube + Slack Notifications
<p>&nbsp;</p>

## **Indice**
- [Implementação Conteiner Docker: Jenkins + SonarQube + Slack Notifications](#implementação-conteiner-docker-jenkins--sonarqube--slack-notifications)
  - [**Indice**](#indice)
  - [**Introdução**](#introdução)
  - [**1. Ferramentas utilizadas**](#1-ferramentas-utilizadas)
  - [**2. Instalação e configuração do Jenkins**](#2-instalação-e-configuração-do-jenkins)
  - [**2.2 - Configuração da extensão do Docker**](#22---configuração-da-extensão-do-docker)
  - [**2.3 - Configuração do SonarQube Scanner for Jenkins**](#23---configuração-do-sonarqube-scanner-for-jenkins)
  - [**2.4 - Configuração do Slack Notifications**](#24---configuração-do-slack-notifications)
  - [**3 - Configuração do SonarQube**](#3---configuração-do-sonarqube)
  - [**4. Conclusão**](#4-conclusão)
<p>&nbsp;</p>

## **Introdução**

Este projeto nasceu da necessidade de de se montar um laboratório de CI/CD, onde optei por utilizar o Jenkins para a gestão do mesmo em conjunto com o SonarQube, para a checagem de vulnerabilidades em códigos e o Slack para envio de notificações informando o status da esteira.

Esta solução roda internamente um serviço docker para testes de builds, podendo ser utilizado para ambientes de homologação ou produção.

<p>&nbsp;</p>

## **1. Ferramentas utilizadas**

Para este laboratório, optei por utilizar as ferramentas abaixo:

* **Docker e Docker Compose =** Para o build da imagem e execução do container da aplicação **Aluraflix** e criação da stack de CI/CD com os serviços **Jenkins**, **Sonarqube** e **PostgreSQL**, que serão descritos logo abaixo.
* **Jenkins =** Ferramenta muito utilizada de integração contínua e entrega contínua. Contém vários plugins para as mais diversas aplicações e serviços, utiliza a linguagem **Groovy** para a criação de pipelines.
* **SonaQube =** Serviço para verificação e erros e vulnerabilidade em códigos, com regras para diversas linguagens de programação.
* **PostgreSQL =** Provê o banco de dados da aplicação **SonarQube**.
* **AWS CLI =** Ferramenta em linha de comando da Amazon para criação, deleção e consulta em sua infraestrutura em nuvem.
<p>&nbsp;</p>

## **2. Instalação e configuração do Jenkins**

Neste laboratório, optei por fazer uma instalação personalizada do Jenkins, onde adicionei alguns pacotes que não estão inclusos na instalação padrão, como o Docker e o AWS CLI. É nela que usamos para criar a imagem e rodar a aplicação em ambiente de desenvolvimento.

Abaixo o conteúdo do arquivo Dockerfile personalizado do Jenkins personalizado:

```yaml
# puxa a imagem oficial do jenkins
   2   │ FROM jenkins/jenkins:lts-jdk11
   3   │ # copia a pasta de configuracao do awscli
   4   │ COPY .aws /var/jenkins_home/.aws
   5   │ # instala os pacotes complementares
   6   │ USER root
   7   │ RUN apt-get update 
   8   │ RUN apt-get upgrade -y && apt-get install -y \ 
   9   │ net-tools build-essential bash-completion awscli \ 
  10   │ ca-certificates curl gnupg lsb-release
  11   │ # cria o diretorio e chaves GPG do docker
  12   │ RUN mkdir -p /etc/apt/keyrings && \ 
  13   │ curl -fsSL https://download.docker.com/linux/debian/gpg | gpg --dearmor -o /etc/apt/keyrings/docker.gpg
  14   │ # cria a lista de repositorio docker no sistema
  15   │ RUN echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/debian \
  16   │   $(lsb_release -cs) stable" | tee /etc/apt/sources.list.d/docker.list > /dev/null
  17   │ # instala o docker no sistema
  18   │ RUN apt-get update && apt-get install -y docker-ce docker-ce-cli containerd.io docker-compose-plugin
  19   │ # adiciona o usuario jenkins no grupo docker
  20   │ RUN usermod -aG docker jenkins
  21   │ # configura ps servicos do docker para iniciar automaticamente na inicializacao
  22   │ RUN systemctl enable docker.service && systemctl enable containerd.service 
  23   │ USER jenkins
```
Salvo o arquivo, o próximo passo é contruir a imagem personalizada do **Jenkins**, darei o nome de jenkins-zedek para esta imagem, para isso, execute o comando abaixo para a sua criação:

```zsh
docker build -t jenkins-zedek .
```

Acompanhe até o final a criação da imagem e verifique se a mesma foi criada corretamente com o comando:

```zsh
docker images
```

Será mostrado uma saida como abaixo:

```zsh
REPOSITORY                                                        TAG         IMAGE ID       CREATED        SIZE
jenkins-zedek                                                     latest      f3f092b89e6e   25 hours ago   1.51GB
jenkins/jenkins                                                   lts-jdk11   3c0cb3ef25cb   4 days ago     460MB
```

Com a imagem criada, vamos executar o **Docker Compose** utilizando o arquivo docker-compose.yml com o seguinte conteúdo abaixo:

```yaml
version: '3.3'

services:
  jenkins:
    image: jenkins-zedek:latest
    container_name: jenkins
    restart: unless-stopped
    
    volumes:
      - jenkins_home:/var/jenkins_home
      - /var/run/docker.sock:/var/run/docker.sock:rw
    
    ports:
      - 8080:8080
      - 50000:50000
    
    networks:
      - jenkins_net

 
  sonarqube:
    image: sonarqube:community
    depends_on:
      - db
    
    environment:
      SONAR_JDBC_URL: jdbc:postgresql://db:5432/sonar
      SONAR_JDBC_USERNAME: sonar
      SONAR_JDBC_PASSWORD: sonar
    
    volumes:
      - sonarqube_data:/opt/sonarqube/data
      - sonarqube_extensions:/opt/sonarqube/extensions
      - sonarqube_logs:/opt/sonarqube/logs
    
    ports:
      - "9000:9000"
    
    networks:
      - jenkins_net

  db:
    image: postgres:12
    
    environment:
      POSTGRES_USER: sonar
      POSTGRES_PASSWORD: sonar
    
    volumes:
      - postgresql:/var/lib/postgresql
      - postgresql_data:/var/lib/postgresql/data
    
    networks:
      - jenkins_net

volumes:
  jenkins_home:
  sonarqube_data:
  sonarqube_extensions:
  sonarqube_logs:
  postgresql:
  postgresql_data:

networks:
  jenkins_net:
    driver: bridge
```

O **Docker Compose** será executado via um shell script chamado **¨stack_jenkins.sh¨** para criar nossa stack do laboratório, segue abaixo o conteudo deste script:

```sh
#!/bin/bash

#Criado por: Melquis Marques
#Data: 11/04/22


echo "O que deseja fazer?"
echo "1. Criar a Stack Jenkins do Lab"
echo "2. Executar a Stack Jenkins do Lab"
echo "3. Parar a Stack Jenkins do Lab"
echo "4. Destruir a Stack Jenkins do Lab"
echo "5. Não executar a Stack Jenkins do Lab"
echo -n "Escolha uma das opções acima [1,2,3]: "
read VAR

if [ $VAR = 1 ] 
then
  #Setar valores requeridos para o SonarQube
  echo "*****************************************************************"
  echo "****** Setando parametros de host requeridos pelo SonarQube. ****"
  echo "***************## REQUER ACESSO ROOT DA MÁQUINA ##***************"
  echo "*****************************************************************"
  echo ""
  sudo sysctl -w vm.max_map_count=524288
  sudo sysctl -w fs.file-max=131072
  ulimit -n 131072
  ulimit -u 8192
  #Subir a Stack do Lab
  sleep 5
  clear
  echo ""
  echo "*****************************************************************"
  echo "**************** Criando a Stack Jenkins do Lab. *****************"
  echo "*****************************************************************"
  echo ""
  docker compose up -d
  sleep 45
  docker exec -it -u 0 jenkins chmod 666 /var/run/docker.sock
  clear
  echo ""
  echo "*****************************************************************"
  echo "************** Verificando se o ambiente subiu OK. **************"
  echo "*****************************************************************"
  echo ""
  docker compose ps
  echo ""
  sleep 5
  clear
  echo ""
  echo "*****************************************************************"
  echo "********** Stack criada e inicializada com sucesso! *************"
  echo "*****************************************************************" 
  echo ""

elif [ $VAR = 2 ] 
then
  #Setar valores requeridos para o SonarQube
  echo "*****************************************************************"
  echo "****** Setando parametros de host requeridos pelo SonarQube. ****"
  echo "***************## REQUER ACESSO ROOT DA MÁQUINA ##***************"
  echo "*****************************************************************"
  echo ""
  sudo sysctl -w vm.max_map_count=524288
  sudo sysctl -w fs.file-max=131072
  ulimit -n 131072
  ulimit -u 8192
  #Subir a Stack do Lab
  sleep 5
  clear
  echo ""
  echo "*****************************************************************"
  echo "**************** Subindo a Stack Jenkins do Lab. *****************"
  echo "*****************************************************************"
  echo ""
  docker compose start
  sleep 45
  clear
  echo ""
  echo "*****************************************************************"
  echo "************** Verificando se o ambiente subiu OK. **************"
  echo "*****************************************************************"
  echo ""
  docker compose ps
  echo ""
  sleep 5
  clear
  echo ""
  echo "*****************************************************************"
  echo "************** Stack inicializada com sucesso! ******************"
  echo "*****************************************************************" 
  echo ""

elif [ $VAR = 3 ] 
then
  #Parar a Stack do Lab
  echo ""
  echo "*****************************************************************"
  echo "**************** Parando a Stack Jenkins do Lab. *****************"
  echo "*****************************************************************"
  echo ""
  docker compose stop
  sleep 5
  clear
  echo ""
  echo "*****************************************************************"
  echo "************** Verificando se o ambiente parou OK. **************"
  echo "*****************************************************************"
  echo ""
  docker compose ps
  echo ""
  sleep 5
  clear
  echo ""
  echo "*****************************************************************"
  echo "***************** Stack parada com sucesso! *********************"
  echo "*****************************************************************" 
  echo ""

elif [ $VAR = 4 ] 
then
  #Parar a Stack do Lab
  echo ""
  echo "*****************************************************************"
  echo "************** Destruindo a Stack Jenkins do Lab. ****************"
  echo "*****************************************************************"
  echo ""
  docker compose down
  sleep 5
  clear
  echo ""
  echo "*****************************************************************"
  echo "************** Verificando se o ambiente parou OK. **************"
  echo "*****************************************************************"
  echo ""
  docker compose ps
  echo ""
  sleep 5
  clear
  echo ""
  echo "*****************************************************************"
  echo "***************** Stack parada com sucesso! *********************"
  echo "*****************************************************************" 
  echo ""

else
  echo "*****************************************************************"
  echo "**************** Nenhum comando foi executado!!! ****************"
  echo "*****************************************************************"
fi
```

Para a execução do script, rode o comando abaixo:

```zsh
./stack_jenkins.sh
```

Será perguntado o que você deseja fazer, conforme mostrado abaixo:

```zsh
❯ ./stack_jenkins.sh                                                                                                                          ─╯
O que deseja fazer?
1. Criar a Stack Jenkins do Lab
2. Executar a Stack Jenkins do Lab
3. Parar a Stack Jenkins do Lab
4. Destruir a Stack Jenkins do Lab
5. Não executar a Stack Jenkins do Lab
Escolha uma das opções acima [1,2,3]: 
```

Se for a primeira vez que esta executando o script, responda a pergunta com a opção **1**.

As próximas execuções, responda a pergunta com a opção **2**.

Para parar a stack, responda a pergunta com a opção **3**.

E para destruir a stack, responda a pergunta com a opção **4**.

O próximo passo é acessar o **Jenkins** pelo navegador, através do endereço http://localhost:8080, onde aparecerá a imagem abaixo para desbloquear o Jenkins:

![Desbloqueio do Jenkins](/imagens/unlock-jenkins.png)
<p>&nbsp;</p>

Para pegar esse password, execute o comando abaixo:

```zsh
docker exec -it jenkins cat /var/lib/jenkins/secrets/initialAdminPassword
```

Será mostrado uma código, copie ele e cole no campo **"Administrator password"** e clique no botão **"Continue"**.

A próxima tela, será perguntado se deseja instalar os plugins no Jenkis, nesta tela, selecione a opção desejada.


![Instalação de plugins do Jenkins](/imagens/select-plugins.png)
<p>&nbsp;</p>

Aguarde a finalização da instalação dos plugins e após isso, será mostrado uma janela para a criação do usuário e senha do sistema.

![Criação do usuário e senha do Jenkins](/imagens/create-first-user.png)
<p>&nbsp;</p>

Altere os campos com o nome e senha desejado e clique em **"Save and Continue"**

A próxima tela é a respeito da configuração da instância do Jenkis, nela, deixe a configuração padrão e clique no botão **"Save and Finish"**

![Configuração da instância do Jenkins](/imagens/instance-configuration.png)
<p>&nbsp;</p>

Pronto! Na próxima tela, clique no botão **"Start using Jenkins"** como mostrado abaixo:

![Tela de iniciar usando o Jenkins](/imagens/start-using-jenkins.png)
<p>&nbsp;</p>

Na tela inicial, você notará que o idioma estará em português, caso esteja usando seu sistema operacional em português. Nesta tela, clique na opção **"Gerenciar Jenkins"** conforme mostrado abaixo:

![Tela gerenciar o Jenkins](/imagens/manage_jenkins.png)
<p>&nbsp;</p>

Nesta tela, clique na opção **"Gerenciar extensões"**

![Tela gerenciar plugins](/imagens/manage_plugins.png)
<p>&nbsp;</p>

Procure por atualizações do Jenkins e das extensões  instaladas, clicando no botão **"Verificar agora"**.

![Tela verificar atualizações](/imagens/Check_updates.png)
<p>&nbsp;</p>

Se houver atualizações para serem instaladas, faça antes de prosseguir com a instalação dos demais plugins que serão utilizados neste laboratório.

![Tela instalação de atualizações](/imagens/update_extensions.png)
<p>&nbsp;</p>

As extensões que precisam ser instaladas para que os pipelines funcionem são:

* **SonarQube Scanner for Jenkins**;
* **Docker Plugin**;
* **Amazon ECR Plugin**;
* **CloudBees AWS Credentials**
* **Gitlab** (no meu caso, pois utilizo o repositório no Gitlab);
* **Slack Notification Plugin**.

Após instaladas essas extensões, iremos para as configurações de cada uma delas.
<p>&nbsp;</p>

## **2.2 - Configuração da extensão do Docker**

Para configurarmos a integração do Docker instalado na máquina local com o Jenkins, precisamos adicioná-lo as configurações do Jenkins, para isso, clique novamente em **"Gerenciar Jenkins"** e selecione a opção **"Gerenciar nós"**.

![Tela Gerenciar nós](/imagens/manage_nodes.png)
<p>&nbsp;</p>

Nesta tela, clique em **"Configurar nuvens"**.

![Tela configurar nuvens](/imagens/Configure_Clouds.png)
<p>&nbsp;</p>

Adicione uma nova nuvem, selecionando a opção **"Docker"** deixando como na imagem abaixo e clicando no botão **"Save"**.

![Tela configurar docker](/imagens/add_docker_cloud.png)
<p>&nbsp;</p>

Feito os passos acima, a integração do Docker com o Jenkins estará completa.
<p>&nbsp;</p>

## **2.3 - Configuração do SonarQube Scanner for Jenkins**

Para configurarmos a extensão, clique na opção **"Gerenciar Jenkins"** e selecione o item **"Ferramenta de configuração global"**.

![Tela configuração global](/imagens/Global_Configuration_Tool.png)
<p>&nbsp;</p>

Nesta tela, procure pelo item **"SonarQube Scanner"** e clique no botão **"SonarQube scanner instalações..."**.

![Tela SonarQube instalações](/imagens/Add_SonarQube_Scanner.png)
<p>&nbsp;</p>

Deixe configurado como mostrado na imagem abaixo e clique no botão **"Save"**:

![Tela SonarQube scanner configuração](/imagens/SonarQube_scanner_configuration.png)
<p>&nbsp;</p>

Com a configuração acima, a extensão estará pronta para se conectar ao servidor SonarQube e executar o scanner no código fonte da aplicação.

<p>&nbsp;</p>

## **2.4 - Configuração do Slack Notifications**

Para recebermos as notificações de execução, erro e sucesso nos deploys da aplicação **Aluraflix**, iremos utilizar um canal no **Slack**.
Para configurarmos, primeiramente teremos que criar um canal em sua Workspace.

Crie o canal com o nome de seu gosto e depois clique sobre a sua workspace e vá em **"Configurações e administração"** => **"Gerenciar apps"**.

![Tela Slack gerenciar apps](/imagens/slack_manage_apps.png)
<p>&nbsp;</p>

Na próxima tela, procure pelo app **"Jenkins CI"** e clique sobre ele para a instalação do mesmo.

![Tela Slack add Jenkins](/imagens/Slack_add_app_jenkins.png)
<p>&nbsp;</p>

Clique no botão **"Adicionar ao Slack"**.

![Tela Slack add Jenkins](/imagens/Slack_add_jenkis_2.png)
<p>&nbsp;</p>

Após adicionar o app do Jenkins no Slack, configure como mostrado na figura abaixo:

![Configuracao canal Slack Jenkins](/imagens/configure_slack.png)
<p>&nbsp;</p>

Nesta tela, escolha o canal que será usado no Slack e na opção **"Token"**, clique em **"Gerar"** e anote o token, pois será usado na configuração da extensão no Jenkins. Por fim, clique em **"Salvar configurações"**.

Agora, volte ao Jenkis e clique em **"Gerenciar Jenkins"** => **"Configurar o sistema"** e no final da página terá a opção **"Slack"**.

Nela, coloque o nome de sua Workspace e em **"Credential"**, clique no botão **"Add"** e crie uma credencial to tipo texto secreto e cole o token recebido pelo Slack feito anteriormente.

Salve a credencial com o nome de **"slack_auth"**, guarde esse nome, pois vai ser utilizado mais para frente nos scripts de pipeline. 

![Configuracao Slack Notification](/imagens/slack_notification_extension.png)
<p>&nbsp;</p>

Agora clique no botão **"Salvar"**.

<p>&nbsp;</p>

## **3 - Configuração do SonarQube**

O próximo serviço que iremos configurar é o **SonarQube**, para isso, no seu navegador digite http://localhost:9000.

No primeiro uso do serviço, coloque **admin** nos campos login e password e clique no botão **"Log in"**. Será pedido para trocar a senha, coloque uma senha to seu gosto.

Na tela principal, clique no botão **"Create Project"** e selecione a opção **"Manually"** para criar um novo projeto no SonarQube.


![SonarQube adicionar projeto](/imagens/SonarQube_create_project.png)
<p>&nbsp;</p>


Na próxima tela, dê um nome para o projeto e automaticamente será preenchido o campo **"Project Key"**. Clique no botão **"Set up"**.


![SonarQube configurar projeto](/imagens/SonarQube_configure_project.png)
<p>&nbsp;</p>

Na tela seguinte, clique sobre o item **"With Jenkins"** como mostrado na imagem abaixo.

![SonarQube projeto Jenkis](/imagens/SonarQube_add_Jenkins.png)
<p>&nbsp;</p>

Na tela a seguir, será perguntado qual a plataforma será utilizada, no meu caso, como utilizo o Gitlab, selecionao então ele.

![SonarQube projeto Jenkis gitlab](/imagens/SonarQube_Jenkins_Gitlab.png)
<p>&nbsp;</p>

Ao selecionar o item, será aberto um Wizard para a configuração do item, leia atentamente e clique no botão **"Configure Analysis"**.

![SonarQube projeto Jenkis gitlab wizard 1](/imagens/SonarQube_wizard_01.png)
<p>&nbsp;</p>

Conforme solicitado, crie um Job no Jenkis se atentando para o item **"Secret token"**, copie este token pois será usado mais para frente no projeto.

![SonarQube projeto Jenkis gitlab wizard 2](/imagens/SonarQube_wizard_02.png)
<p>&nbsp;</p>

Nesta parte do wizard, ele pede para criar um webhook no nosso repositório, crie o webhook passando o endereço do repositório e o token gerado no passo anterior.

![SonarQube projeto Jenkis gitlab wizard 3](/imagens/SonarQube_wizard_03.png)
<p>&nbsp;</p>

No próximo passo, é pedido para criar um arquivo **Jenkinsfile** utilizando um template que que melhor descreva seu projeto.

![SonarQube projeto Jenkis gitlab wizard 4](/imagens/SonarQube_wizard_04.png)
<p>&nbsp;</p>

Crie o arquivo **sonar-project.properties** como pedido e o arquivo Jenkisfile com o conteúdo sugerido.

![SonarQube projeto Jenkis gitlab wizard 5](/imagens/SonarQube_wizard_05.png)
<p>&nbsp;</p>

Clique no botão **"Finish this tutorial"** para finalizar esta configuração.

Um último passo a ser feito no SonarQube é gerar um token de acesso, utilizando para isso o usuário administrador. Para isso, clique em **"Administration"**.

![SonarQube projeto Jenkis gitlab wizard 8](/imagens/SonarQube_Administration_tab.png)
<p>&nbsp;</p>

Nesta tela, clique em **"Security"** => **"Users"**.

![SonarQube projeto Jenkis gitlab wizard 9](/imagens/SonarQube_Security_tab.png)
<p>&nbsp;</p>

No usuário **Administrator** clique em **"new token"**, copie o token gerado, pois será usado para configuração da integração do Jenkins com o SonarQube.

![SonarQube gerar token usuário administrator](/imagens/SonarQube_Administrator_token.png)
<p>&nbsp;</p>

O próximo passo será configurar a integração do servidor SonarQube no Jenkins. 

Entre no Jenkins novamente e vá em **"Gerenciar Jenkins"** => **"Configurar o sistema"** e procure pelo item **"SonarQube servers"** e deixe como a figura abaixo:

![Jenkins configuração do servidor SonarQube](/imagens/Jenkins_SonarQube_Configuration.png)
<p>&nbsp;</p>

Crie uma credencial do tipo texto secreto com o token do usuário **Administrator** gerado anteriormente no servidor **SonaQube** e clique em salvar.

<p>&nbsp;</p>

## **4. Conclusão**

Feito os passos acima, o ambiente estará pronto para ser usado conforme desejar.

Altere os dados das configurações do projeto para o nome do seu projeto atual.
