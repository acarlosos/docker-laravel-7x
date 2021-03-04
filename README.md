## Como configurar Laravel, Nginx, e MySQL com Docker Compose
## Passo 1 — Fazendo download do Projeto e instalando dependências

Primeiramente, verifique se você está no seu diretório home e faça uma cópia da versão mais recente do Projeto para um diretório chamado docker-laravel-7x:

    $ cd ~
    $ git clone https://github.com/acarlosos/docker-laravel-7x docker-laravel-7x

Vá até o diretório docker-laravel-7x:

    $ cd ~/docker-laravel-7x

Em seguida, utilize a imagem do composer para montar os diretórios que você precisará para seu projeto:

    $ docker run --rm -it --volume $(pwd):/app prooph/composer:7.3 install

Como passo final, defina as permissões no diretório do projeto para que ele seja propriedade do seu usuário não root:

    sudo chown -R $USER:$USER ~/laravel-app

## Passo 2 - Modificando as configurações do ambiente e executando os contêineres

Como passo final, porém, vamos fazer uma cópia do arquivo .env.example que o Laravel inclui por padrão e nomear a copia .env, que é o arquivo que o Laravel espera para definir seu ambiente:

    $ cp .env.example .env

Você pode agora modificar o arquivo .env no contêiner app para incluir detalhes específicos sobre sua configuração.

    $ nano .env

Encontre o bloco que especifica o DB_CONNECTION e atualize-o para refletir as especificidades da sua configuração. Você modificará os seguintes campos:

O DB_HOST será seu contêiner de banco de dados db.
O DB_DATABASE será o banco de dados laravel.
O DB_USERNAME será o nome de usuário que você usará para o seu banco de dados. Neste caso, vamos usar laraveluser.
O DB_PASSWORD será a senha segura que você gostaria de usar para esta conta de usuário, vamos usar secret.

```/var/www/.env```
```
DB_CONNECTION=mysql
DB_HOST=db
DB_PORT=3306
DB_DATABASE=laravel
DB_USERNAME=laraveluser
DB_PASSWORD=secret
```

Salve suas alterações e saia do seu editor.

Com todos os seus serviços definidos no seu arquivo docker-compose, você precisa emitir um único comando para iniciar todos os contêineres, criar os volumes e configurar e conectar as redes:

    $ docker-compose up -d

Assim que o processo for concluído, utilize o comando a seguir para listar todos os contêineres em execução:
    $ docker ps

Você verá o seguinte resultado com detalhes sobre seus contêineres do app, webserver e db:

```Output```

```
CONTAINER ID        NAMES               IMAGE                             STATUS              PORTS
c31b7b3251e0        db                  mysql:5.7.22                      Up 2 seconds        0.0.0.0:3306->3306/tcp
ed5a69704580        app                 digitalocean.com/php              Up 2 seconds        9000/tcp
5ce4ee31d7c0        webserver           nginx:alpine                      Up 2 seconds
```

Usaremos agora o docker-compose exec para definir a chave do aplicativo para o aplicativo Laravel.
Este comando gerará uma chave e a copiará para seu arquivo .env, garantindo que as sessões do seu usuário e os dados criptografados permaneçam seguros:

    $ docker-compose exec app php artisan key:generate

Para colocar essas configurações em um arquivo de cache, que irá aumentar a velocidade de carregamento do seu aplicativo, execute:

    $ docker-compose exec app php artisan config:cache

Suas definições da configuração serão carregadas em /var/www/bootstrap/cache/config.php no contêiner.

Como passo final, visite http://localhost no navegador.
Você verá a seguinte página inicial para seu aplicativo Laravel:

<img src="https://assets.digitalocean.com/articles/laravel_docker/laravel_home.png" >


## Passo 3 - Criando um usuário para o MySQL

Para criar um novo usuário, execute uma bash shell interativa no contêiner db com o docker-compose exec:

    $ docker-compose exec db bash

Dentro do contêiner, logue na conta administrativa root do MySQL:

    root@6efb373db53c:/# mysql -u root -p

Você será solicitado a inserir a senha para a conta root do MySQL ( secret ).

    mysql> show databases;

Você verá o banco de dados laravel listado no resultado:

```
Output
+--------------------+
| Database           |
+--------------------+
| information_schema |
| laravel            |
| mysql              |
| performance_schema |
| sys                |
+--------------------+
5 rows in set (0.00 sec)
```

Em seguida, crie a conta de usuário que terá permissão para acessar esse banco de dados.

    mysql> GRANT ALL ON laravel.* TO 'laraveluser'@'%' IDENTIFIED BY 'secret';

Reinicie os privilégios para notificar o servidor MySQL das alterações:

    mysql> FLUSH PRIVILEGES;

Saia do MySQL:

    mysql>exit;

Por fim, saia do contêiner:

    root@6efb373db53c:/# exit

## Passo 4 - Migrando dados e teste com o console Tinker

Primeiramente, teste a conexão com o MySQL executando o comando Laravel artisan migrate, que cria uma tabela migrations no banco de dados de dentro do contêiner:

    $ docker-compose exec app php artisan migrate

Este comando irá migrar as tabelas padrão do Laravel. O resultado que confirma a migração será como este:

```
Output

Migration table created successfully.
Migrating: 2014_10_12_000000_create_users_table
Migrated:  2014_10_12_000000_create_users_table
Migrating: 2014_10_12_100000_create_password_resets_table
Migrated:  2014_10_12_100000_create_password_resets_table
```

Assim que a migração for concluída, você pode fazer uma consulta para verificar se está devidamente conectado ao banco de dados usando o comando tinker:

    $ docker-compose exec app php artisan tinker

Teste a conexão do MySQL obtendo os dados que acabou de migrar:

    >>> \DB::table('migrations')->get();

Você verá um resultado que se parece com este:

```
Output
=> Illuminate\Support\Collection {#3147
     all: [
       {#3145
         +"id": 1,
         +"migration": "2014_10_12_000000_create_users_table",
         +"batch": 1,
       },
       {#3154
         +"id": 2,
         +"migration": "2014_10_12_100000_create_password_resets_table",
         +"batch": 1,
       },
       {#3155
         +"id": 3,
         +"migration": "2019_08_19_000000_create_failed_jobs_table",
         +"batch": 1,
       },
     ],
   }
>>>
```

## O arquivo do Docker Compose

No arquivo docker-compose, você tem três serviços: app, webserver e db.  Certifique-se de substituir a senha root para o MYSQL_ROOT_PASSWORD, definida como uma variável de ambiente sob o serviço db, por uma senha forte da sua escolha:

```
~/laravel-app/docker-compose.yml

version: '3'
services:

  #PHP Service
  app:
    build:
      context: .
      dockerfile: Dockerfile
    image: digitalocean.com/php
    container_name: app
    restart: unless-stopped
    tty: true
    environment:
      SERVICE_NAME: app
      SERVICE_TAGS: dev
    working_dir: /var/www
    volumes:
      - ./:/var/www
      - ./php/local.ini:/usr/local/etc/php/conf.d/local.ini
    networks:
      - app-backend

  #Nginx Service
  webserver:
    image: nginx:alpine
    container_name: webserver
    restart: unless-stopped
    tty: true
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./:/var/www
      - ./nginx/conf.d/:/etc/nginx/conf.d/
    networks:
      - app-backend

  #MySQL Service
  db:
    image: mysql:5.7.22
    container_name: db
    restart: unless-stopped
    tty: true
    ports:
      - "3306:3306"
    environment:
      MYSQL_DATABASE: laravel
      MYSQL_ROOT_PASSWORD: secret
      SERVICE_TAGS: dev
      SERVICE_NAME: mysql
    volumes:
      - dbdata:/var/lib/mysql
      - ./mysql/my.cnf:/etc/mysql/my.cnf
    networks:
      - app-backend

#Docker Networks
networks:
  app-backend:
    driver: bridge

#Volumes
volumes:
  dbdata:
    driver: local

```
Os serviços aqui definidos incluem:

- app: esta definição de serviço contém o aplicativo Laravel e executa uma imagem personalizada do Docker, digitalocean.com/php, que você definirá no Passo 4. Ela também define o working_dir no contêiner para /var/www.
- webserver: esta definição de serviço extrai a imagem nginx:alpine do Docker e expõe as portas 80 e 443.
- db: esta definição de serviço extrai a imagem mysql:5.7.22 do Docker e define algumas variáveis de ambiente, incluindo um banco de dados chamado laravel para o seu aplicativo e a senha da** root** do banco de dados. Você pode dar o nome que quiser ao banco de dados e deve substituir o your_mysql_root_password pela senha forte escolhida. Esta definição de serviço também mapeia a porta 3306 no host para a porta 3306 no contêiner.

Cada propriedade container_name define um nome para o contêiner, que corresponde ao nome do serviço.

Para facilitar a comunicação entre contêineres, os serviços estão conectados a uma rede bridge chamada app-backend. Uma rede bridge utiliza um software bridge que permite que os contêineres conectados à mesma rede bridge se comuniquem uns com os outros. O driver da bridge instala automaticamente regras na máquina do host para que contêineres em redes bridge diferentes não possam se comunicar diretamente entre eles. Isso cria um nível de segurança mais elevado para os aplicativos, garantindo que apenas serviços relacionados possam se comunicar uns com os outros. Isso também significa que você pode definir várias redes e serviços que se conectam a funções relacionadas: os serviços de aplicativo front-end podem usar uma rede frontend, por exemplo, e os serviços back-end podem usar uma rede backend.

## Persistindo os dados

O Docker tem recursos poderosos e convenientes para persistir os dados. No nosso aplicativo, vamos usar volumes e bind mounts para persistir o banco de dados, o aplicativo e os arquivos de configuração. Os volumes oferecem flexibilidade para backups e persistência além do ciclo de vida de um contêiner, enquanto os bind mounts facilitam alterações no código durante o desenvolvimento, fazendo alterações nos arquivos do host ou diretórios imediatamente disponíveis nos seus contêineres. Nossa configuração usa ambos.

```
~/laravel-app/docker-compose.yml
...
#MySQL Service
db:
  ...
    volumes:
      - dbdata:/var/lib/mysql
    networks:
      - app-backend
  ...

```
