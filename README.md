# 1. Auto Deploy

Visando a diminuição do tempo de deploy das aplicações recebidas pela fábrica de software, definimos duas alternativas para subir essas aplicações para produção, uma utilizando o servidor ftp já existente
e automatizando o processo final, e a outra utilizando um repositório onde essas aplicações sejam armazenadas de forma segura, de fácil acesso e o mais importante, somente para pessoas autorizadas e sem alterar a 
esteira de build. As alternativas utilizarão tecnologias distintas, a primeira utiliza Ansible e a segunda Docker.

A segunda alternativa vamos criar um bucket no S3 na Amazon que poderá ser montado em qualquer servidor utilizando credenciais IAM. Após a montagem, o mesmo poderá ser acessível
em um diretório configurável, vamos utilizar como exemplo nesse documento o diretório "/mnt/s3app".

A criação de um bucket S3 nos traz algumas vantagens principais como:

* Maior disponibilidade, visto que o S3 é replicado para diversas zonas dentro da AWS;
* Maior velocidade para download e upload dos arquivos;
* Maior segurança, pois o acesso no S3 é feito somente via credentials ou usuário e senha no console AWS;
* Redução de custo, pois o S3 só cobra por GB armazenado;

Nesse documento não vamos abordar a configuração de um bucket S3 em um servidor Linux, mas pode ser facilmente configurado via awscli.

Tendo em vista que o bucket já está montado em /mnt/s3app, nos servidores Jenkins da fábrica de software. Quando um deploy é finalizado e liberado para homologação ou produção
pelo Jenkins ele já é armazenado automaticamente via plugin s3 em diretórios pré-definidos.

Esses buckets também estão montados nos servidores de aplicação e também recebem automaticamente os novos deploys.

Abaixo vamos descrever os passos necessários em cada uma das alternativas.


## Ansible

O Ansible é uma ferramenta para facilitar o provisionamento de servidores, gerenciamento de configurações e deploy de aplicações. Como não precisa de um agent para fazer as tarefas nos servidores remotos
ele é amplamente utilizado pela sua facilidade de instalação e utilização. Com ele é possível utilizar pequenos comandos, como criar vários procedimentos e executá-los de uma só vez (playbook). Em nosso caso
vamos utilizar um playbook para automatizar o processo de deploy da nossa aplicação Java. O Ansible não necessita de agentes para executar seus jobs, tudo é feito via SSH.

Para instalar o Ansible em servidores RedHat like:
```
yum install epel-release
yum install ansible
```

Para instalar o Ansible em servidores Debian like:
```
echo 'deb http://http.debian.net/debian jessie-backports main' > /etc/apt/sources.list.d/backports.list
apt-get update
apt-get -t jessie-backports install "ansible"
```

Após a instalação do Ansible, configure os servidores que rodarão a aplicação Java no arquivo /etc/ansible/hosts

```
[app-servers]
servidor1 ansible_ssh_host=192.168.10.100
servidor2 ansible_ssh_host=192.168.10.200
```
Configure o usuário e senha com permissões e acesso a máquina via SSH
```
mkdir -p /etc/ansible/group_vars
touch /etc/ansible/group_vars/servers
echo "---" > /etc/ansible/group_vars/servers
echo "ansible_ssh_user: usuario" >> /etc/ansible/group_vars/servers
echo "ansible_ssh_pass: senha" >> /etc/ansible/group_vars/servers
```
Crie o arquivo /etc/ansible/deployJava.yml (ou outro nome de sua escolha) e adicione as seguintes linhas

```
---
- hosts: servers
  vars:
  - warName: app.war
  - warRemotePath: /app
  - logFile: /var/log/application.log

  tasks:

  - name: Delete existent war file
    file: path={{ warRemotePath }}/{{ warName }} state=absent

  - name: Download WAR to server
    command: wget ftp://jenkinsaplications.mycompany.com/application.war -O {{ warRemotePath }}/{{ warName }}
  
  - name: Status of Java Process
    shell: ps -ef |  grep jar | grep -v grep
    register: process_list
    changed_when: false  

  - name: Kill "Java" if process running
    command: pkill -f 'java -jar'
    when: "process_list.stdout.find('java -jar') == 1"  
  
  - name: Start Application and save output to logfile
    command: /usr/bin/java -jar {{ warRemotePath }}/{{ warName }} > {{ logFile }}
```
E execute o Ansible para fazer o deploy da aplicação

```
ansible-playbook -l appserver javaDeploy.yml
```

## Docker

Para rodar uma aplicação java via docker é bem simples, basta instalarmos e configurarmos o docker e o docker-compose em seu servidor que rodará a aplicação com os comandos abaixo:

```
curl -fsSL https://get.docker.com/ | sh
systemctl start docker
systemctl enable docker
usermod -aG docker $(whoami)
usermod -aG docker yourusername
```

```
pip install docker-compose
```

Após isso crie um arquivo com o nome "docker-compose.yml" com o seguinte conteúdo
```
appjava:
 image: bankmonitor/spring-boot:latest-war
 container_name: appjava
 hostname: appjava
 volumes:
 - /app:/app
 env_file:
  - ./java.env
```

Lembrando que a sua aplicação dentro do container tem que estar dentro do diretório "/app" com o nome "app.war", interessante é ter esse diretório criado no host para não confundir.

Crie um arquivo com as variáveis de ambiente da sua aplicação com o nome "java.env"

```
SPRING_PROFILES_ACTIVE=prod
```

Após isso basta rodar o container com os comandos abaixo, no mesmo diretório do seu docker-compose-yml e java.env

```
docker-compose build
docker-compose up -d
```

Essa tarefa de rodar os comandos para inicializar o container pode ser facilmente feita do seu servidor Ansible.


______________________________________________________________________________________________________________________________________________________________________________________________



# 2. Minishift

OpenShift é uma plataforma PAAS desenvolvida pela RedHat onde podemos criar nossa nuvem privada, com a finalidade de fazer deploy, teste e gerenciamento de aplicações, tanto para homologação
quanto para produção. Trabalha com containers docker, facilitando assim a portabilidade das suas aplicações. O Minishift é uma plataforma que utiliza máquinas virtuais para prover uma arquitetura
semelhante ao OpenShift localmente em qualquer servidor Linux ou Windows.

Nesse documento vamos abordar a instalação do Minishit em uma distribuição Centos 7.

Faça o download dos pré-requisitos utilizados pelo minishift

```
yum install -y qemu-kvm
yum install -y kvm
curl -L https://github.com/dhiltgen/docker-machine-kvm/releases/download/v0.7.0/docker-machine-driver-kvm -o /usr/local/bin/docker-machine-driver-kvm
chmod +x /usr/local/bin/docker-machine-driver-kvm
systemctl start libvirtd
```

Faça o download do Minishift

```
wget https://github.com/minishift/minishift/releases/download/v1.9.0/minishift-1.9.0-linux-amd64.tgz
tar xvf minishift-1.9.0-linux-amd64.tgz
cp minishift /usr/bin/
```

Após o processo acima, o minishift já deverá estar disponível para ser executado

```
[root@server ~]# minishift
Minishift is a command-line tool that provisions and manages single-node OpenShift clusters optimized for development workflows.

Usage:
  minishift [command]

Available Commands:
  addons      Manages Minishift add-ons.
  config      Modifies Minishift configuration properties.
  console     Opens or displays the OpenShift Web Console URL.
  delete      Deletes the Minishift VM.
  docker-env  Sets Docker environment variables.
  help        Help about any command
  hostfolder  Manages host folders for the OpenShift cluster.
  ip          Gets the IP address of the running cluster.
  logs        Gets the logs of the running OpenShift cluster.
  oc-env      Sets the path of the 'oc' binary.
  openshift   Interacts with your local OpenShift cluster.
  profile     Manages Minishift profiles.
  ssh         Log in to or run a command on a Minishift VM with SSH.
  start       Starts a local OpenShift cluster.
  status      Gets the status of the local OpenShift cluster.
  stop        Stops the running local OpenShift cluster.
  update      Updates Minishift to the latest version.
  version     Gets the version of Minishift.

Flags:
      --alsologtostderr                  log to standard error as well as files
  -h, --help                             help for minishift
      --log_backtrace_at traceLocation   when logging hits line file:N, emit a stack trace (default :0)
      --log_dir string                   If non-empty, write log files in this directory (default "")
      --logtostderr                      log to standard error instead of files
      --profile string                   Profile name (default "minishift")
      --show-libmachine-logs             Show logs from libmachine.
      --stderrthreshold severity         logs at or above this threshold go to stderr (default 2)
  -v, --v Level                          log level for V logs
      --vmodule moduleSpec               comma-separated list of pattern=N settings for file-filtered logging

Use "minishift [command] --help" for more information about a command.
```

O Minishift utiliza o KVM para instanciar seus containers, desta forma o Sistema Operacional irá criar interfaces de redes específicas as maquinas virtuais KVM.

Faça o start do ambiente Minishift

```
minishift start
```

Você deverá receber no final uma mensagem informando a interface que o web console estará disponível

- The server is accessible via web console at:
    https://192.168.42.7:8443

Por padrão não existe um portforward encaminhando as requisições do seu servidor Centos para a interface do Minishift, para corrigir isso vamos instalar um
Proxy reverso, fazendo o encaminhamento dessas requisições.

```
yum install -y nginx
```

Criar o arquivo /etc/nginx/conf.d/default.conf com o seguinte conteúdo, alterando somente o IP da sua interface web do minishift

```
server {
        listen 8443;
        server_name _;

      location / {
	  proxy_set_header Host $host;
	  proxy_set_header X-Real-IP $remote_addr;
          proxy_pass https://192.168.42.7:8443/;
	  proxy_set_header Connection "";
 	  proxy_read_timeout 180s;
        }
     }
```

Após isso faça o reload das configurações no nginx e seu Minishift já estará funcionando no endereço http://iphostserver:8443

```
nginx -t
systemctl reload nginx
```




