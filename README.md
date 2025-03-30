# Projeto de Docker Swarm com Microserviços

## 📌 Objetivo
Este projeto tem como objetivo demonstrar a criação de um **cluster Docker Swarm** utilizando uma aplicação web em **PHP** e um banco de dados **MySQL**. A implementação foi realizada na plataforma **Google Cloud Platform (GCP)**.


## 🚀 Passo a Passo

### 1️⃣ **Criação de Instâncias no Google Cloud Platform (GCP)**

Crie três instâncias na plataforma GCP para configurar o ambiente. Utilize as imagens abaixo para acompanhar o processo visualmente.

<p align="center">
  <img src="/images/fig_1.png?raw=true" alt="Instances GCP" width="80%" height="50%" />
</p>


### 2️⃣ **Instalação do Docker e MySQL no Instance-1-manager**

Primeiro, instale o Docker na VM *Instance-1-manager*:

```bash
# Baixar e instalar o Docker
curl -fsSL https://get.docker.com -o get-docker.sh
sudo sh get-docker.sh
```

Crie volumes para o aplicativo e o banco de dados:

```bash
docker volume create data-mysql
docker volume create app
```

Crie o container MySQL com os parâmetros necessários:

```bash
docker run -e MYSQL_DATABASE=meubanco -e MYSQL_ROOT_PASSWORD=Senha123 --name mysql -d -p 3306:3306 --volume=data-mysql:/var/lib/mysql mysql:8.0
```

Resultado:

<p align="center">
  <img src="/images/fig_2.png?raw=true" alt="Container mysql" width="80%" height="50%" />
</p>


### 3️⃣ **Criação da Tabela no MySQL**

Utilize um cliente SQL, como o **DBeaver**, para criar a tabela no banco de dados MySQL:

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

Resultado:

<p align="center">
  <img src="/images/fig_3.png?raw=true" alt="Meu banco mysql" width="50%" height="30%" />
</p>


### 4️⃣ **Desenvolvimento do Microserviço em PHP**

Agora, crie o código PHP que será executado pelo servidor web. Crie o arquivo `index.php` no volume *app*:

```bash
cd /var/lib/docker/volumes/app/
nano index.php
```

Exemplo de código PHP para conectar ao banco de dados MySQL:

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

// Conexão com o banco de dados
$link = new mysqli($servername, $username, $password, $database);

if (mysqli_connect_errno()) {
    printf("Connect failed: %s\n", mysqli_connect_error());
    exit();
}

$valor_rand1 = rand(1, 999);
$valor_rand2 = strtoupper(substr(bin2hex(random_bytes(4)), 1));
$host_name = gethostname();

$query = "INSERT INTO dados (AlunoID, Nome, Sobrenome, Endereco, Cidade, Host) 
          VALUES ('$valor_rand1', '$valor_rand2', '$valor_rand2', '$valor_rand2', '$valor_rand2', '$host_name')";

if ($link->query($query) === TRUE) {
    echo "Novo registro criado com sucesso";
} else {
    echo "Erro: " . $link->error;
}
?>
</body>
</html>
```


### 5️⃣ **Criação do Container Web-Server PHP + Apache**

Agora, crie o container para o servidor web com o código PHP.

```bash
docker run --name web-server -dt -p 80:80 --mount type=volume,src=app,dst=/app webdevops/php-apache:alpine-php7
```

Resultado:

<p align="center">
  <img src="/images/fig_4.png?raw=true" alt="Web-server" width="80%" height="50%" />
</p>


### 6️⃣ **Testando a Conexão entre Containers**

Realize um teste de conexão entre o container **web-server** e o **mysql**.

<p align="center">
  <img src="/images/gif_1.gif?raw=true" alt="Web-server" width="80%" height="50%" />
</p>


### 7️⃣ **Iniciando o Docker Swarm Cluster**

Inicie o **Docker Swarm** no *manager node*:

```bash
docker swarm init
```

Após isso, adicione as instâncias *worker* ao cluster:

```bash
docker swarm join --token SWMTKN-1-1t6f4j6sdltce4ztek2va1ezovck8d0zhbc0glfa14j33axk37-b41dauvtua4602hdfpowwm77o 10.128.0.9:2377
```

Resultado:

<p align="center">
  <img src="/images/fig_6.png?raw=true" alt="Docker cluster swarm" width="80%" height="50%" />
</p>


### 8️⃣ **Criação de Serviço no Docker Swarm**

Crie um serviço com múltiplas réplicas para garantir alta disponibilidade:

```bash
docker service create --name web-server --replicas 10 -dt -p 80:80 --mount type=volume,src=app,dst=/app webdevops/php-apache:alpine-php7
docker service ps web-server
```

Resultado:

<p align="center">
  <img src="/images/fig_10.png?raw=true" alt="Docker cluster swarm" width="80%" height="50%" />
</p>


### 9️⃣ **Replicando o Volume no Cluster com NFS**

Para replicar o conteúdo entre os nós, configure o **NFS**:

No nó *manager*:

```bash
apt-get install nfs-server
nano /etc/exports
exportfs -ar
```

No nó *worker*:

```bash
apt-get install nfs-common
mount -o v3 10.128.0.9:/var/lib/docker/volumes/app/_data /var/lib/docker/volumes/app/_data
```


### 🔟 **Criação de Proxy com NGINX para Balanceamento de Carga**

Para distribuir a carga entre os containers, configure um **proxy NGINX**:

1. Crie o arquivo de configuração NGINX:

```bash
nano nginx.conf
```

2. Exemplo de configuração para balanceamento de carga:

```nginx
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
```

3. Crie o **Dockerfile**:

```dockerfile
FROM nginx
COPY nginx.conf /etc/nginx/nginx.conf
```

4. Build e run do container:

```bash
docker build -t proxy-app .
docker run --name my-proxy-app -dti -p 4500:4500 proxy-app
```

Resultado:

<p align="center">
  <img src="/images/fig_11.png?raw=true" alt="Docker cluster swarm" width="80%" height="50%" />
</p>


### 1️⃣1️⃣ **Estressando o Cluster**

Por fim, faça o teste de carga para verificar o balanceamento de carga e o funcionamento do cluster.

<p align="center">
  <img src="/images/gif_2.gif?raw=true" alt="Estressando o cluster" width="80%" height="50%" />
</p>


## ⚠️ Troubleshooting

### Problema: Erro de Conexão MySQL - `caching_sha2_password`

Este erro ocorre devido à mudança no método de autenticação do MySQL 8. Para resolver, crie um novo usuário com a autenticação **mysql_native_password**:

```bash
# Acesse o MySQL
mysql -u root -p --protocol=tcp

# Crie o novo usuário
CREATE USER newuser@"%" IDENTIFIED WITH mysql_native_password BY 'password';

# Conceda permissões ao novo usuário
GRANT ALL PRIVILEGES ON *.* TO "newuser"@"%";
FLUSH PRIVILEGES;
```

Referência: [MySQL 8 - Autenticação caching_sha2_password](https://dev.mysql.com/blog-archive/mysql-8-0-4-new-default-authentication-plugin-caching_sha2_password/)


## 📌 Conclusão

Este projeto demonstra a capacidade de gerenciar um cluster Docker Swarm com **PHP** e **MySQL**, incluindo a criação de microserviços, balanceamento de carga, e persistência de dados em múltiplos nós. A aplicação implementada demonstra como utilizar boas práticas de escalabilidade e alta disponibilidade, tornando a solução robusta e eficiente para ambientes de produção.

## Contribuições

Se você tiver sugestões de melhorias ou encontrar problemas com o script, sinta-se à vontade para abrir um **issue** ou submeter um **pull request**.

## Contatos e Network

- **LinkedIn**: [LinkedIn](https://www.linkedin.com/in/jacivaldocarvalho/) 👔
- **E-mail**: [E-mail](mailto:jacivaldocarvalho@gmail.com) 📧
- **GitHub**: [GitHub](https://github.com/jacivaldocarvalho) 🐙
- **Medium**: [Medium](https://medium.com/@jacivaldocarvalho) ✍️

Sempre aberto a novas conexões e oportunidades de aprendizado!

## Licença

Este projeto está licenciado sob a [MIT License](LICENSE).
