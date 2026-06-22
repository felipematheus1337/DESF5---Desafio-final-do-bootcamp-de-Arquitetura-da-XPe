# Desafio Final — Bootcamp Arquiteto(a) de Soluções

Arquitetura de e-commerce na **AWS**, projetada e **provisionada** para alta disponibilidade, resiliência a falhas e escalabilidade dinâmica, atendendo aos requisitos do desafio final (Fundamentos de Cloud, Soluções de Rede, Soluções de Dados e Soluções Digitais — Escalabilidade e Elasticidade).

A aplicação é um microsserviço **Go** (CRUD de produtos), *stateless* e containerizado, distribuído via **Docker Hub** e executado em instâncias EC2 gerenciadas por Auto Scaling.

## Stack e decisões principais

| Item | Escolha |
|---|---|
| Cloud / Região | AWS — `sa-east-1` (São Paulo) |
| Aplicação | Microsserviço Go (CRUD), stateless, imagem no Docker Hub |
| Compute | EC2 Amazon Linux 2023 (`t3.micro`), gerenciadas por Auto Scaling Group |
| Balanceamento | Application Load Balancer (HTTP :80 → porta 8080 da app) |
| Persistência | Amazon RDS PostgreSQL **Multi-AZ** (PaaS) |
| Credencial do banco | **SSM Parameter Store** (SecureString) + IAM Role — sem senha no código |
| Provisionamento | Console AWS (sem IaC) |

> **Por que stateless:** todo o estado vive no RDS. Isso é o que permite o Auto Scaling criar/destruir instâncias de 3 a 6 sem perda de dados — qualquer instância atende qualquer requisição.

## Diagrama da Arquitetura

![Arquitetura da Solução](arquitetura.png)

---

## Como a requisição flui na arquitetura (passo a passo)

1. **Usuário → Internet Gateway**
   O cliente faz uma requisição HTTP/HTTPS pela internet pública. Ela entra na `VPC ECOMMERCE-VPC (10.0.0.0/16)` através do **Internet Gateway**, único ponto de entrada/saída de tráfego da internet para a VPC.

2. **Internet Gateway → Application Load Balancer (ALB)**
   O tráfego é roteado para o **ALB**, posicionado nas **subnets públicas** das zonas **AZ-A** e **AZ-B**. O ALB é protegido pelo **SG-ALB**, que libera apenas as portas **80/443** vindas da internet — nenhuma outra porta ou origem é aceita.

3. **ALB → Auto Scaling Group (EC2 Go-ecommerce-ms)**
   O ALB distribui as requisições entre as instâncias EC2 registradas no **Auto Scaling Group**, em **subnets privadas de aplicação**, replicadas em **AZ-A** e **AZ-B**. O ASG mantém **mínimo de 3 e máximo de 6 instâncias** Linux, escalando automaticamente por uma política de *target tracking* (CPU 50%).
   As instâncias só aceitam conexão do ALB: o **SG-APP** libera a porta da aplicação (8080) exclusivamente para origens do **SG-ALB**, bloqueando qualquer acesso direto da internet às VMs. O *health check* do ALB usa o endpoint `GET /health`.

4. **EC2 → credencial do banco (IAM + SSM)**
   A senha do banco **não fica embutida** na aplicação nem na imagem. Cada instância EC2 possui uma **IAM Role** anexada (`ecommerce-ec2-role`) com permissão `ssm:GetParameter` (+ `kms:Decrypt`) restrita ao parâmetro da senha. No boot, a aplicação usa a *identidade da role* para buscar a credencial no **SSM Parameter Store** — sem chave fixa no código.

5. **EC2 → RDS Multi-AZ (Primary/Standby)**
   A aplicação acessa o banco gerenciado (**Amazon RDS**), em **subnets privadas de banco**, isoladas da internet, conectando-se via SSL (`sslmode=require`). O **SG-DB** libera a porta **5432** apenas para origens do **SG-APP** — somente as instâncias da aplicação alcançam o banco.
   O RDS está em **Multi-AZ**: instância **Primary** na **AZ-A** e **Standby** na **AZ-B**, com **replicação síncrona**. Em caso de falha da AZ-A (ou da primária), há *failover* automático para a Standby, garantindo continuidade.

6. **Caminho de retorno**
   A resposta segue o caminho inverso: RDS → EC2 → ALB → Internet Gateway → Usuário, sempre dentro dos limites definidos pelos Security Groups, sem expor banco ou instâncias diretamente à internet.

---

## Controle de acesso ao banco (IAM, SSM e Security Groups)

O acesso seguro das VMs ao banco é garantido por **três camadas complementares**:

- **IAM (identidade):** a IAM Role anexada à EC2 define *se a instância consegue obter a credencial* do banco no SSM. Sem a role, não há acesso à senha.
- **SSM Parameter Store (segredo):** a senha vive como `SecureString` em `/go-ecommerce/db_password`, criptografada por KMS, fora do código e dos prints.
- **Security Groups (rede):** o encadeamento `SG-ALB → SG-APP → SG-DB` garante que só o ALB fala com a app, e só a app fala com o banco. É o "firewall para origens autorizadas" do enunciado.

O *read/write* propriamente dito é concedido pelo `GRANT` do PostgreSQL ao usuário da aplicação. Em conjunto, IAM + SSM + SG + GRANT entregam acesso de leitura e escrita controlado e sem credenciais fixas embutidas.

> **Alternativa não adotada:** *IAM Database Authentication* (`rds-db:connect`), que substitui a senha por um token temporário gerado via SDK. Optou-se pela senha em SSM por simplicidade operacional, mantendo o segredo fora do código.

---

## Provisionamento na AWS (prática)

A infraestrutura foi **efetivamente provisionada** no Console AWS (região `sa-east-1`) e documentada passo a passo, com capturas de tela em [`docs/prints/`](docs/prints/). Ordem de criação (cada recurso depende do anterior já existir):

1. **Rede** — VPC `ecommerce-vpc` (10.0.0.0/16) via *VPC and more*: 2 AZs, subnets públicas e privadas, Internet Gateway, NAT Gateway e route tables. Subnets dedicadas de banco (`ecommerce-db-subnet-az-a/-b`).
2. **Security Groups** — `SG-ALB` (80/443 da internet), `SG-APP` (8080 a partir do SG-ALB), `SG-DB` (5432 a partir do SG-APP).
3. **SSM Parameter Store** — senha do banco como `SecureString`.
4. **IAM Role** — `ecommerce-ec2-role` com `AmazonSSMManagedInstanceCore` + policy inline (`ssm:GetParameter`, `kms:Decrypt`).
5. **RDS PostgreSQL Multi-AZ** — `db.t3.micro`, Multi-AZ DB instance (failover), banco inicial `goecommerce`, backups automáticos habilitados, SG-DB, sem acesso público.
6. **Launch Template** — AMI Amazon Linux 2023, `t3.micro`, SG-APP, instance profile `ecommerce-ec2-role`, *user-data* que instala Docker, busca a senha no SSM e sobe o container do Docker Hub na porta 8080.
7. **Target Group + ALB** — TG HTTP:8080 com health check `/health`; ALB internet-facing nas subnets públicas, listener HTTP:80 → TG.
8. **Auto Scaling Group** — Launch Template nas subnets privadas, anexado ao TG, *health check type ELB*, desired 3 / min 3 / max 6, política de *target tracking* (CPU 50%).

> O roteiro detalhado de cada passo está no documento [`Subindo_na_AWS.pdf`](docs/Subindo_na_AWS.pdf).

### Validação (teste de ponta a ponta)

```bash
# Health check usado pelo ALB
curl http://<DNS-DO-ALB>/health           # {"status":"UP"}

# Escrita no RDS
curl -X POST http://<DNS-DO-ALB>/api/v1/products/ \
  -H "Content-Type: application/json" \
  -d '{"name":"Notebook","price":3500.50,"stock":10}'

# Leitura (prova persistência + balanceamento entre instâncias)
curl http://<DNS-DO-ALB>/api/v1/products/
```

---

## Recuperação de desastres (DR)

- **Multi-AZ:** réplica *Standby* síncrona em outra AZ, com *failover* automático em caso de falha de zona ou da instância primária.
- **Backups automáticos:** habilitados no RDS, permitindo *point-in-time recovery*.
- *Extensão possível (não implementada):* réplica de leitura *cross-region* para resiliência multi-regional.

---

## Checklist dos requisitos do PDF

### 1. (Obrigatório) Desenho da Arquitetura de Soluções

- [x] Múltiplas zonas de disponibilidade (AZ-A e AZ-B) para continuidade mesmo com falha de uma zona.
- [x] Balanceamento de carga (Application Load Balancer) distribuindo o tráfego entre as instâncias.
- [x] Escalonamento automático das VMs (Auto Scaling Group), mínimo de 3 e máximo de 6 instâncias, imagens Linux.
- [x] Banco de dados gerenciado (PaaS) — Amazon RDS Multi-AZ — com alta disponibilidade e segurança.
- [x] Controle de acesso (IAM Role + SSM) para que as VMs acessem o banco sem credenciais fixas.

### 2. (Opcional) Provisionamento das VMs e Configuração do Banco

- [x] VMs provisionadas em múltiplas AZs (Amazon Linux 2023) na conta AWS.
- [x] Load Balancer (ALB) configurado e distribuindo tráfego entre as instâncias.
- [x] Escalonamento automático (3 a 6 instâncias) implementado via ASG + target tracking.
- [x] Restrição de acesso via firewall (Security Groups SG-ALB / SG-APP / SG-DB) aplicada na infraestrutura real.
- [x] RDS Multi-AZ provisionado com backups automáticos. *(Multi-AZ, não multi-region — ver seção DR.)*
- [x] IAM aplicado e validado (role `ecommerce-ec2-role` anexada às EC2, lendo o segredo no SSM).

### 3. (Opcional) Implementação de Segurança e Monitoramento

- [x] IAM controlando o acesso das VMs à credencial do banco (role + SSM), provisionado e validado.
- [ ] Logs e monitoramento (CloudWatch) — alarmes de CPU são criados automaticamente pelo *target tracking* do ASG; marque ao anexar o print do CloudWatch.

### 4. Documentação e Entregáveis

- [x] **(Obrigatório)** Diagrama da Arquitetura, com VMs, balanceador, banco e mecanismo de failover (RDS Multi-AZ Primary/Standby).
- [x] **(Opcional)** Capturas de tela das configurações de serviços, IAM, segurança, escalabilidade e DR (em `docs/prints/`).

### Resumo dos Entregáveis

- [x] 1. Arquitetura da Solução (Diagrama).
- [x] 2. (Opcional) Provisão da Infraestrutura (IaaS) para as máquinas.
- [x] 3. (Opcional) Provisão da Infraestrutura (PaaS) para persistência.
- [x] 4. (Opcional) Provisionamento dos Elementos de Segurança.

---

## Componentes da Arquitetura

| Componente | Função |
|---|---|
| VPC `ECOMMERCE-VPC` (10.0.0.0/16) | Isolamento de rede de toda a solução |
| Internet Gateway | Ponto de entrada/saída de tráfego da internet |
| NAT Gateway | Saída de internet para as subnets privadas (ex.: puxar a imagem do Docker Hub) |
| Application Load Balancer (ALB) | Distribui requisições entre as instâncias, em subnets públicas (AZ-A/AZ-B) |
| Auto Scaling Group | Escala as EC2 (Linux) de 3 a 6, em subnets privadas de aplicação (AZ-A/AZ-B) |
| EC2 Go-ecommerce-ms | Microsserviço Go (CRUD), container do Docker Hub, porta 8080 |
| Amazon RDS Multi-AZ | Banco gerenciado (PaaS), Primary (AZ-A) + Standby (AZ-B) em replicação síncrona |
| SG-ALB | Libera 80/443 apenas da internet |
| SG-APP | Libera a porta da aplicação (8080) apenas para origem SG-ALB |
| SG-DB | Libera 5432 apenas para origem SG-APP |
| IAM Role `ecommerce-ec2-role` | Dá às EC2 acesso ao segredo do banco (SSM) sem credenciais fixas |
| SSM Parameter Store | Guarda a senha do banco como SecureString, fora do código |
