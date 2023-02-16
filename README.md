# Projeto Prático de Docker Dwarm com Microserviços

## Objetivo
Criar um cluster docker com uma aplicação web em php que consome um banco de dados mysql. O projeto foi reaizado no Google Cloud Platform.


## Passo 1: Cria três instâncias no GCP.

<p>
 <img src="/images/fig_1.png?raw=true" alt="Instances GCP" width="80%" height="50%" />
</p>

## Passo 2: Instala o Docker e o container Mysql na VM Instance-1-manager.

```shell
# Instala o docker
curl -fsSL https://get.docker.com -o get-docker.sh
sudo sh get-docker.sh
```

```shell

#cria os volumes App e Data
docker volume create data-mysql
docker volume create app

```

```shell
# cria o container mysql
docker run -e MYSQL_DATABASE=meubanco -e MYSQL_ROOT_PASSWORD=Senha123 --name mysql -d -p 3306:3306 --volume=data-mysql:/var/lib/mysql mysql:8.0
```

Resultados
<p>
 <img src="/images/fig_2.png?raw=true" alt="Container mysql" width="80%" height="50%" />
</p>


###Passo 3:

Cria a tabela no container mysql pelo DBeaver ou outro cliente sql.

```sql
CREATE TABLE dados (
    AlunoID int,
    Nome varchar(50),
    Sobrenome varchar(50),
    Endereco varchar(150),
    Cidade varchar(50),
    Host varchar(50)
);
```
Resultado

<p>
 <img src="/images/fig_3.png?raw=true" alt="Meu banco mysql" width="50%" height="30%" />
</p>

### Passo 4: Cria o código web-server em PHP

Criar o microserviço em PHP. No volume app criado anteriormente, cria-se o arquivo index.php 

```shell

cd /var/lib/docker/volumes/app/
nano index.php
```

O código da aplicação.

```php
<html>

<head>
<title>Exemplo PHP</title>
</head>
<body>

<?php
ini_set("display_errors", 1);
header('Content-Type: text/html; charset=iso-8859-1');



echo 'Versao Atual do PHP: ' . phpversion() . '<br>';

$servername = "35.222.89.31";
$username = "jacivaldocarvalho";
$password = "secret-password";
$database = "meubanco";

// Criar conexão


$link = new mysqli($servername, $username, $password, $database);

/* check connection */
if (mysqli_connect_errno()) {
    printf("Connect failed: %s\n", mysqli_connect_error());
    exit();
}

$valor_rand1 =  rand(1, 999);
$valor_rand2 = strtoupper(substr(bin2hex(random_bytes(4)), 1));
$host_name = gethostname();


$query = "INSERT INTO dados (AlunoID, Nome, Sobrenome, Endereco, Cidade, Host) VALUES ('$valor_rand1' , '$valor_rand2', '$valor_rand2', '$valor_rand2', '$valor_rand2','$host_name')";


if ($link->query($query) === TRUE) {
  echo "New record created successfully";
} else {
  echo "Error: " . $link->error;
}

?>
</body>
</html>
```

### Passo 5: Cria o container web-server

Cria o container php-apache para comportar o index.php.

```shell
docker run --name web-server -dt -p 80:80 --mount type=volume,src=app,dst=/app webdevops/php-apache:alpine-php7
```

Resultado

<p>
 <img src="/images/fig_4.png?raw=true" alt="Web-server" width="80%" height="50%" />
</p>

### Passo 6: Estressando o container
Teste de conexão dos containers web-service e mysql, utilizando uma instância apenas.
<p>
 <img src="/images/gif_1.gif?raw=true" alt="Web-server" width="80%" height="50%" />
</p>

### Passo 7: Iniciando um cluster docker swarm

```shell
# remover o container web-server do manager para realizar a replicação
docker stop web-service
docker rm webservice

# inicia o cluster swarm
docker swarm init
```
Resultado na instance-1-manager
<p>
 <img src="/images/fig_6.png?raw=true" alt="Docker cluster swarm" width="80%" height="50%" />
</p>

Inserindo as instâncias 1 e 2 worked ao cluster:

```shell
docker swarm join --token SWMTKN-1-1t6f4j6sdltce4ztek2va1ezovck8d0zhbc0glfa14j33axk37-b41dauvtua4602hdfpowwm77o 10.128.0.9:2377
```

<p>
 <img src="/images/fig_7.png?raw=true" alt="Docker cluster swarm" width="80%" height="50%" />
</p>

<p>
 <img src="/images/fig_8.png?raw=true" alt="Docker cluster swarm" width="80%" height="50%" />
</p>

Em resumo 
<p>
 <img src="/images/fig_9.png?raw=true" alt="Docker cluster swarm" width="80%" height="50%" />
</p>

### Passo 8: Cria serviço no cluster

```shell
# Cria o serviço com 10 réplicas
docker service create --name web-server --replicas 10 -dt -p 80:80 --mount type=volume,src=app,dst=/app webdevops/php-apache:alpine-php7

# Verifica o resultado
docker service ps web-server
```
Resultado
<p>
 <img src="/images/fig_10.png?raw=true" alt="Docker cluster swarm" width="80%" height="50%" />
</p>

### Passo 9: Replica o volume dentro do cluster
Como o conteúdo da instância manager não é replicado para os outros nós, há a necessidade de um servidor de arquivo, para esse caso, utilizamos o NFS.

```shell
# No nó manager fazemos
apt-get install nfs-server
```
```shell
# No nó worked 2 e 3 fazemos
apt-get install nfs-common
```

```shell
# configura a pasta que será disponibilizada aos demais nós
nano /etc/exports

# insere no arquivo. Obs. Está liberado para todos, porém o ideal é setar para cada nó.
/var/lib/docker/volumes/app/_data *(rw,sync,subtree_check)

# Para exportar a pasta
exportfs -ar

# Para conferir o compartilhamento
root@instance-1-manager:/var/lib/docker/volumes/app/_data# showmount -e
Export list for instance-1-manager:
/var/lib/docker/volumes/app/_data *

```
```shell
# Monta a pasta compartilhada nos dois nós worked

mount -o v3 10.128.0.9:/var/lib/docker/volumes/app/_data /var/lib/docker/volumes/app/_data

```

### Passo 10: Cria um proxy NGINX

```shell
# cria a pasta proxy
mkdir proxy
cd proxy

# cria o arquivo
nano nginx.conf
```

```shell
# Insere no arquivo nginx.conf 

http {

    upstream all {
        server 35.222.89.31:80;
        server 104.197.43.115:80;
        server 34.27.42.237:80;
    }

    server {
         listen 4500;
         location / {
              proxy_pass http://all/;
         }
    }

}


events { }
```

```dockerfile
# Para facilitar, cria um dockerfile com o conteúdo
FROM nginx
COPY nginx.conf /etc/nginx/nginx.conf
```

```shell
#Cria a imagem
docker build -t proxy-app .

# Sobe o container criado
docker run --name my-proxy-app -dti -p 4500:4500 proxy-app
```

Resultado

<p>
 <img src="/images/fig_11.png?raw=true" alt="Docker cluster swarm" width="80%" height="50%" />
</p>

### Passo 11: Estressando o cluster
Podemos observar na coluna host que o proxy está realizando o balanceamento entre os nós em cada reqisição.

<p>
 <img src="/images/gif_2.gif?raw=true" alt="Estressando o cluster" width="80%" height="50%" />
</p>


### troubleshooting

#### Mysql erro: caching_sha2_password

O MySQL 8 utiliza uma autenticação diferente ao de seus antecessores, que até o momento não é reconhecido pelo PHP 7, o que gera o erro "The server requested authentication method unknown to the client". 


Solução: Criar um novo usuário, pois como o usuário atual (roor) do MySQL por padrão utiliza o tipo de autenticação caching2_sha2_password a conexão não será realizada utilizando ele.


Criando novo usuário:
```shell
#acesse o mysql
mysql -u root -p --protocol=tcp

# Criar usuário com configuração antiga (mysql_native_password)
CREATE USER newuser@"%" IDENTIFIED WITH mysql_native_password BY 'password';

# Garante pemissões ao novo usuário
GRANT ALL PRIVILEGES ON *.* TO "newuser"@"%";

# A nova permissão concedida ao usuário será ativada após a execução do comando a seguir
FLUSH PRIVILEGES;
```

Referência: https://dev.mysql.com/blog-archive/mysql-8-0-4-new-default-authentication-plugin-caching_sha2_password/