# Docker: Utilização prática no cenário de Microsserviços

Vamos configurar uma arquitetura de microsserviços usando Docker Swarm, replicando em 3 clusters na AWS. Utilizaremos um arquivo index.php e um banco MySQL.

## Passo 1: Configurar Ambiente AWS

### Criar 3 Instâncias EC2:

- Acesse o console da AWS e crie 3 instâncias EC2 com o sistema operacional de sua preferência (recomendo Amazon Linux 2 ou Ubuntu).

- Assegure-se de que as instâncias tenham segurança adequada, com as portas necessárias abertas (principalmente 22 para SSH e 3306 para MySQL).

## Passo 2: Instalar Docker nas Instâncias

### Conectar via SSH:

- Conecte-se a cada instância via SSH.

```bash
ssh -i "seu-arquivo-chave.pem" ec2-user@seu-endereco-publico-ec2
```

### Instalar Docker:

- Execute os seguintes comandos em cada instância para instalar o Docker:

```bash
sudo yum update -y
sudo yum install docker -y
sudo service docker start
sudo usermod -a -G docker ec2-user
```

## Passo 3: Configurar Docker Swarm

### Inicializar Docker Swarm:

- Em uma das instâncias, inicialize o Docker Swarm:

```bash
docker swarm init --advertise-addr seu-endereco-ip
```

### Adicionar Nós ao Swarm:

- Nas outras duas instâncias, junte-se ao Swarm com o comando fornecido pela inicialização (algo como docker swarm join --token <(TOKEN)> seu-endereco-ip:2377).

## Passo 4: Configurar o MySQL

### Criar Container MySQL:

- Crie um container MySQL em uma das instâncias:

```bash
docker run --name meu-mysql -e MYSQL_ROOT_PASSWORD=Senha123 -d mysql:latest
```

### Criar Banco de Dados e Tabela:

- Conecte-se ao container MySQL e crie o banco e a tabela necessários:

```bash
docker exec -it meu-mysql mysql -u root -p
```

- No shell MySQL:

```sql
CREATE DATABASE meubanco;
USE meubanco;
CREATE TABLE dados (
    AlunoID INT NOT NULL PRIMARY KEY,
    Nome VARCHAR(50),
    Sobrenome VARCHAR(50),
    Endereco VARCHAR(50),
    Cidade VARCHAR(50),
    Host VARCHAR(50)
);
```

## Passo 5: Configurar Serviço PHP com Docker

### Criar Dockerfile para PHP:

- Crie um arquivo Dockerfile no diretório do seu projeto:

```dockerfile
FROM php:7.4-apache
COPY index.php /var/www/html/
EXPOSE
```

### Criar index.php:

- Crie o arquivo index.php no mesmo diretório:

```php
<html><head><title>Exemplo PHP</title></head><body><?php
ini_set("display_errors", 1);
header('Content-Type: text/html; charset=iso-8859-1');
echo 'Versao Atual do PHP: ' . phpversion() . '<br>';
$servername = "54.234.153.24";
$username = "root";
$password = "Senha123";
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
?></body></html>
```

### Construir a Imagem Docker:

- No diretório do projeto, construa a imagem Docker:

```bash
docker build -t meu-php-app .
```

### Criar Serviço Docker Swarm:

- Crie o serviço PHP no Docker Swarm:

```bash
docker service create --name php-service --replicas 3 -p 80:80 meu-php-app
```

## Passo 6: Verificar e Testar

### Verificar Serviços:

- Verifique se os serviços estão rodando:

```bash
docker service ls
```

## Passo 7: Configurar Nginx

### Criar Configuração do Nginx:

- Crie um arquivo chamado nginx.conf com a seguinte configuração:

```dockerfile
events { }

http {
    upstream all {
        server 172.31.0.37:80;
        server 172.31.0.151:80;
        server 172.31.0.149:80;
    }
    server {
        listen 4500;
        location / {
            proxy_pass http://all/;
        }
    }
}
```

### Criar Dockerfile para o Nginx:

- No mesmo diretório, crie um arquivo chamado Dockerfile para o Nginx:

```dockerfile
FROM nginx:latest
COPY nginx.conf /etc/nginx/nginx.conf
EXPOSE 4500
```

### Construir a Imagem Docker do Nginx:

```bash
docker build -t meu-nginx .
```

### Criar Serviço Docker Swarm para o Nginx:

```bash
docker service create --name nginx-service --replicas 1 -p 4500:4500 meu-nginx
```

### Acessar Aplicação:

- Acesse a aplicação através do endereço IP de qualquer uma das instâncias (por exemplo, http://seu-endereco-publico-ec2).

# Estresse

Para os testes de estresse do contâiner e do serviço, foi utilizado o site: [loader.io](https://loader.io/)

![loader]()
