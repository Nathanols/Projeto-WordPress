# WordPress em Alta Disponibilidade na AWS

![Image](https://github.com/user-attachments/assets/6cd13a6b-efb8-41fe-82cf-6082621750f5)

## 📌 Resumo do projeto

Este projeto demonstra como provisionar uma arquitetura de WordPress pronta para estudo/experimentação que prioriza **alta disponibilidade** e **resiliência**, usando serviços gerenciados da AWS. O WordPress roda dentro de **containers Docker** em instâncias EC2 que pertencem a um **Auto Scaling Group**. O tráfego externo entra via **ALB** e os arquivos de mídia são armazenados em **EFS**. Os dados relacionais ficam no **Amazon RDS**.

---

```markdown
## 🏗️ Arquitetura (visão geral)

```mermaid
flowchart LR
  Internet --> ALB[ALB (public subnets)]
  ALB --> TG[Target Group]
  TG --> ASG[Auto Scaling Group]
  ASG --> EC2[EC2 (Docker + WP)]
  EC2 -->|NFS| EFS[(EFS)]
  EC2 -->|MySQL| RDS[(RDS MySQL)]
```

---

## ✅ O que foi feito

* Criada uma **VPC personalizada** com 2 Availability Zones (AZs), 2 subnets públicas (ALB) e 4 subnets privadas (EC2 + RDS).
* Provisionado **Application Load Balancer** em subnets públicas com Target Group apontando para instâncias EC2 em subnets privadas.
* Implementado **Auto Scaling Group** com Launch Template que usa `user-data` para provisionar Docker, Docker Compose, montar EFS e inicializar WordPress via `docker-compose`.
* Criado **Amazon EFS** e configurados mount targets nas subnets privadas; EFS usado para persistência de `/var/www/html` do WordPress.
* Criado **Amazon RDS** (MySQL compatível) em subnets privadas, com acesso restrito por Security Group apenas às EC2.
* Configurado Security Groups para isolar tráfego e garantir que apenas o ALB fale com as instâncias na porta HTTP.

---

## 🧰 Tecnologias e versões (usadas/validadas)

* **AMI:** Amazon Linux 2 (AMI mínima recomendada)
* **Docker:** instalado via `amazon-linux-extras` (varia conforme AMI; validado com Docker Engine 24.x em testes)
* **Docker Compose:** `v2.20.2` (binário instalado em `/usr/local/bin/docker-compose`)
* **WordPress:** `wordpress:latest` (imagem oficial Docker — usar tag específica em produção)
* **RDS:** MySQL-compatible (testado com MySQL 8.x / db.t3g.micro) — **sem Multi-AZ**
* **EFS:** v4.1 mount targets, performance mode `generalPurpose`
* **ALB:** Application Load Balancer (HTTP listener 80)
* **Auto Scaling Group:** mínima 2 instâncias, máxima 4 (ajustável)

---

## 📁 Arquivos importantes

* `user-data.sh` — script usado no Launch Template / EC2 para preparar a instância (instalar Docker, Docker Compose, montar EFS, criar `docker-compose.yml` e subir containers).
* `docker-compose.yml` — compose que roda o container WordPress conectado ao RDS e montando EFS em `/var/www/html`.
* `README.md` — este documento.

---

## ⚙️ Conteúdos principais (exemplos)

### docker-compose.yml (exemplo)

```yaml
version: '3.8'

services:
  wordpress:
    image: wordpress:latest
    container_name: wordpress
    restart: always
    ports:
      - "80:80"
    environment:
      WORDPRESS_DB_HOST: SEU ENDPOINT RDS
      WORDPRESS_DB_USER: admin
      WORDPRESS_DB_PASSWORD: ***
      WORDPRESS_DB_NAME: wordpress
    volumes:
      - /mnt/efs/wordpress:/var/www/html
```

## 🧭 Passo-a-passo (Console AWS)

> **Pré-requisito:** acesso com permissão suficiente para criar VPC, EC2, RDS, EFS, ALB, IAM (caso necessário) e Elastic IP para NAT.

1. **Criar VPC** (`10.0.0.0/16`) e selecionar 2 AZs.
2. **Criar Subnets**:

   * Públicas (2): para ALB e NAT Gateway.
   * Privadas (4): para EC2 e RDS (2 AZs × 2 subnets cada).
3. **Criar Internet Gateway** e associar à VPC.
4. **Criar NAT Gateway** em uma das subnets públicas (atribuir Elastic IP).
5. **Configurar Route Tables**:

   * Public RT: rota 0.0.0.0/0 → IGW; associar subnets públicas.
   * Private RT: rota 0.0.0.0/0 → NAT Gateway; associar subnets privadas.
6. **Criar Security Groups** (exemplos):

   * `SG-ALB`:

     * Inbound: HTTP 80 from `0.0.0.0/0` (ou CIDR limitado)
     * Outbound: all
   * `SG-EC2`:

     * Inbound: HTTP 80 from `SG-ALB` (referencie o id do SG-ALB)
     * Inbound: SSH 22 from seu IP (apenas para debug)
     * Outbound: all (ou permitir 3306 para `SG-RDS`)
   * `SG-RDS`:

     * Inbound: MySQL/Aurora 3306 from `SG-EC2`
     * Outbound: none / all conforme necessidade
   * `SG-EFS`:

     * Inbound: NFS 2049 from `SG-EC2`
7. **Criar EFS**: criar file system, criar mount targets nas subnets privadas e associar `SG-EFS`.
8. **Criar RDS** (MySQL-compatible):

   * Engine: MySQL (ou MariaDB)
   * Instance class: `db.t3g.micro` (conforme restrição didática)
   * Subnet group: subnets privadas
   * Public access: NO
   * Security group: `SG-RDS`
9. **Criar Launch Template** para EC2:

   * AMI: Amazon Linux 2
   * Instance type: `t3.micro` (ou `t3.small` conforme necessidade)
   * Associate SG: `SG-EC2`
   * Disable Auto-assign public IPv4 (instâncias ficam em privates)
   * User data: cole o `user-data.sh` completo
10. **Criar Target Group (ALB)**:

    * Target type: instance (ou ip se preferir)
    * Protocol: HTTP, Port: 80
    * Health check path: `/`
11. **Criar ALB** em subnets públicas:

    * Listener HTTP 80 → forward para Target Group criado
    * Security Group: `SG-ALB`
12. **Criar Auto Scaling Group**:

    * Launch template: selecionado
    * Network: selecione as subnets privadas
    * Attach to Target Group: selecione o TG
    * Set min=2 desired=2 max=4 (ajuste conforme testes)
13. **Aguardar** instâncias nascerem e `docker-compose` rodar via user-data.
14. **Verificar** Target Group → targets deverão ficar `healthy`.
15. **Acessar** o DNS do ALB no navegador e completar o setup do WordPress.

---

## 🔎 Validação & testes úteis (comandos)

> Execute estes comandos via SSH em uma instância EC2 do ASG (ou use SSM Session Manager):

### Estado do Docker / containers

```bash
sudo systemctl status docker
/usr/local/bin/docker-compose --version
cd /home/ec2-user
/usr/local/bin/docker-compose up -d
docker ps --format 'table {{.Names}}\t{{.Status}}\t{{.Ports}}'
docker logs -f wordpress
```

### Testes HTTP

```bash
curl -i http://localhost/
curl -i http://<EC2-PRIVATE-IP>/
aws elbv2 describe-target-health --target-group-arn <TARGET_GROUP_ARN>
```

### Verificar EFS

```bash
mount | grep efs
ls -la /mnt/efs/wordpress
df -h /mnt/efs
```

### Verificar RDS

```bash
mysql -h <RDS-ENDPOINT> -u admin -p
```

---

## 🛠️ Troubleshooting — problemas comuns

* **502 Bad Gateway no ALB:** targets `unhealthy`, SG errado, container não iniciado, conflito.
* **docker-compose não encontrado:** use `/usr/local/bin/docker-compose` ou instale plugin oficial `docker-compose-plugin` e rode `docker compose`.
* **Problemas de permissão no EFS:** use `sudo chown -R 1000:1000 /mnt/efs/wordpress`.
* **Erro no RDS:** revise SGs (`SG-EC2` → `SG-RDS`), credenciais e endpoint.
* **Health Check ALB falhando:** ajuste caminho para `/` ou crie `healthcheck.php` retornando `200 OK`.

---

## Tela do Projeto funcionando
<img width="1917" height="1077" alt="Image" src="https://github.com/user-attachments/assets/62114b11-e42e-44ae-b057-025d54155ebe" />

## 🧹 Limpeza de recursos (evitar cobranças)

1. Delete Auto Scaling Group e Launch Template.
2. Delete Load Balancer e Target Group.
3. Delete EC2 instances/volumes.
4. Delete RDS (snapshot opcional).
5. Delete EFS.
6. Delete NAT Gateway + Elastic IP.
7. Delete IGW, route tables, subnets, VPC.

---

## ⚠️ Considerações finais

* Fixar versões de imagens e pacotes.
* Usar RDS Multi-AZ, backups, escalabilidade.

---

👨‍💻 **Autor:** Nathan Oliveira (projeto de estudo em AWS)

