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
