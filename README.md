# 🚀 WordPress com Docker na AWS
Este projeto configura um ambiente na AWS para rodar o **WordPress** utilizando **Docker**, **EFS**, **RDS MySQL** e **Load Balancer**, garantindo escalabilidade e persistência de dados.

---

## 📌 1. Arquitetura do Projeto
A infraestrutura é baseada em:
✅ **EC2** (Servidor de aplicação rodando WordPress em container Docker)  
✅ **EFS** (Armazenamento persistente para os arquivos do WordPress)  
✅ **RDS MySQL** (Banco de dados gerenciado para o WordPress)  
✅ **ALB (Load Balancer)** (Distribuição de tráfego entre múltiplas instâncias)  

---

## 📌 2. Criando a Infraestrutura na AWS

### 1️⃣ Criando a VPC e Sub-redes
1. Acesse o **AWS Console** > **VPC** > **Create VPC**.  
2. Crie uma **VPC com CIDR 10.0.0.0/16**.  
3. Crie **duas sub-redes públicas e duas privadas**.  
4. Associe um **Internet Gateway** à VPC para acesso externo.  

---

### 2️⃣ Criando a Instância EC2 para o WordPress
1. Vá para **EC2 > Launch Instance**.  
2. Escolha **Ubuntu 24.04 LTS** como AMI.  
3. Tipo de instância: **t2.micro** (grátis no Free Tier).  
4. **Configure o Security Group** com:
   - **SSH (22)** → Apenas para seu IP.  
   - **HTTP (80) e HTTPS (443)** → Para acesso ao site.  
5. Ative **"Auto-assign Public IP"** para permitir acesso externo.  
6. Faça o download da **chave `.pem`** para conectar via SSH.  
7. Clique em **Launch Instance**.  

---

## 📌 3. Configuração do Servidor e WordPress

### 1️⃣ Conectando via SSH
``bash
ssh -i "sua-chave.pem" ubuntu@SEU_IP_PUBLICO``

---

### 2️⃣ Instalando Docker e Dependências

``export DEBIAN_FRONTEND=noninteractive
sudo apt update -y && sudo apt upgrade -y
sudo apt install -y docker.io docker-compose git nfs-common mysql-server
sudo systemctl start docker
sudo systemctl enable docker
sudo usermod -aG docker ubuntu``

---

## 📌 4. Configuração do EFS (Armazenamento Persistente)

### 1️⃣ Criando o EFS

1.No AWS Console, vá para EFS > Create File System.
2.Selecione a VPC correta e sub-redes privadas.
3.Configure Segurança e Permissões.
4.Anote o ID do EFS 

---

### 2️⃣ Montando o EFS na EC2

``EFS_ID="fs-xxxxxxxxxx"
REGION="us-east-1"
sudo mkdir -p /mnt/efs/wordpress
sudo mount -t nfs4 -o nfsvers=4.1,rsize=1048576,wsize=1048576,hard,timeo=600,retrans=2 $EFS_ID.efs.$REGION.amazonaws.com:/ /mnt/efs
echo "$EFS_ID.efs.$REGION.amazonaws.com:/ /mnt/efs nfs4 defaults,_netdev 0 0" | sudo tee -a /etc/fstab``

---

## 📌 5. Configuração do Banco de Dados MySQL no RDS

Vá para RDS > Create Database.
Escolha MySQL e configure:
  ``Username: admin
  Password: SUA_SENHA_SEGURA
  Habilite acesso público (para teste).``
Finalize e copie o endpoint do banco de dados

---

## 📌 6. Criando o docker-compose.yml

Na criação do dokcer-compose.yml eu usei o siguinte script 
```
#!/bin/bash

sudo yum update -y
sudo yum install -y docker
sudo systemctl start docker
sudo systemctl enable docker
sudo usermod -a -G docker ec2-user

sudo curl -L "https://github.com/docker/compose/releases/download/1.27.4/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
sudo chmod +x /usr/local/bin/docker-compose

sudo yum install -y amazon-efs-utils
sudo systemctl start amazon-efs-utils
sudo systemctl enable amazon-efs-utils

sudo mkdir /mnt/efs

sudo echo "fs-01c2ceb042e154ac1.efs.us-east-1.amazonaws.com:/ /mnt/efs nfs4 nfsvers=4.1,rsize=1048576,wsize=1048576,hard,timeo=600,retrans=2,noresvport,_netdev 0 0" >> /etc/fstab
sudo mount -a

sudo mkdir /mnt/efs/wordpress


sudo cat <<EOF > /mnt/efs/docker-compose.yaml
version: '3.8'
services:
  wordpress:
    image: wordpress:latest
    restart: always
    ports:
      - 80:80
    environment:
      WORDPRESS_DB_HOST: bancocayo.cn0q06k0uoyw.us-east-1.rds.amazonaws.com
      WORDPRESS_DB_NAME: wordpressdb
      WORDPRESS_DB_USER: bancocayo
      WORDPRESS_DB_PASSWORD: 123456789F
    volumes:
      - /mnt/efs/wordpress:/var/www/html
EOF

docker-compose -f /mnt/efs/docker-compose.yaml up -d
```

---

## 📌 7. Configuração do Load Balancer (ALB)

1.No AWS Console, vá para EC2 > Load Balancers > Create Load Balancer.
2.Escolha Application Load Balancer.
3.Configure listeners para a porta 80 (HTTP).
4.Associe as instâncias EC2 ao Load Balancer.
5.Teste acessando o DNS do Load Balancer no navegador.

---

## 📌 10. Acessando o WordPress

E para acessar o WordPress basta apenas usar os seguintes comando abaixo.: 
```
http://SEU_IP_PUBLICO
```

---

## 📌 Conclusão do Projeto
Este projeto demonstrou a implementação completa de um ambiente escalável e seguro para rodar o WordPress na AWS, utilizando Docker, EFS, RDS MySQL e um Load Balancer.

Ao longo do processo, configuramos a infraestrutura necessária, garantindo persistência de dados, escalabilidade e alta disponibilidade. Utilizamos práticas recomendadas para a implantação de aplicações em nuvem, como:

✅ Automação da Instalação → Criamos um script user_data.sh para configurar o ambiente automaticamente.

✅ Segurança e Organização → Implementamos um VPC estruturado com sub-redes públicas e privadas.

✅ Armazenamento Persistente → Utilizamos EFS para garantir que os arquivos do WordPress sejam armazenados com segurança.

✅ Banco de Dados Gerenciado → Utilizamos RDS MySQL, separando a aplicação do banco de dados para melhor desempenho e gerenciamento.

✅ Balanceamento de Carga → Configuramos um Application Load Balancer (ALB) para distribuir o tráfego entre múltiplas instâncias EC2.

✅ Versionamento de Código → Criamos um repositório Git para manter o controle das versões e facilitar futuras melhorias.

Com essa abordagem, conseguimos criar um ambiente modular, escalável e de fácil manutenção, permitindo futuras expansões e adaptações conforme necessário. 🚀

Este projeto pode ser utilizado como base para outras aplicações web que precisam de alta disponibilidade e confiabilidade na AWS.
