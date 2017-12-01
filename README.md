# Auto Deploy

Visto que o processo de deploy atualmente é trabalhoso e inseguro, já que FTP não é uma ferramenta ideal para trabalhar com o manuseio e armazenamento de aplicações de produção
sugerimos a utilização de um repositório onde essas aplicações sejam armazenadas de forma segura e de fácil acesso e o mais importante, somente para pessoas autorizadas.

Então para isso, vamos criar um bucket no S3 na Amazon que poderá ser montado em qualquer servidor utilizando credenciais IAM. Após a montagem, o mesmo poderá ser acessível
em diretório, vamos utilizar como exemplo nesse documento o diretório "/mnt/s3app".

E criação de um bucket S3 nos traz algumas vantagens principais como:

* Maior disponibilidade, visto que o S3 é replicado para diversas zonas dentro da AWS;
* Maior velocidade para download e upload dos arquivos;
* Maior segurança, pois o acesso no S3 é feito somente via credentials ou usuário e senha no console AWS;
* Redução de custo, pois o S3 só cobra por GB armazenado;

Nesse documento não vamos abordar a configuração de um bucket S3 em um servidor Linux, mas pode ser facilmente configurado via awscli.

Tendo em vista que o bucket já está montado em /mnt/s3app, nos servidores Jenkins da fábrica de software. Quando um deploy é finalizado e liberado para homologação ou produção
pelo Jenkins ele já é armazenado automaticamente via plugin s3 em diretórios pré-definidos.

Esses buckets também estão montados nos servidores de aplicação e também recebem automaticamente os novos deploys.

Analisando o cenário descrito acima, definimos duas formas de automatizar esse processo de deploy: via ansible ou via docker





