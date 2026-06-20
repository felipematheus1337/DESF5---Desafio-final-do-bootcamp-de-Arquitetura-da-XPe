# Desafio Final — Bootcamp Arquiteto(a) de Soluções

Arquitetura de e-commerce na AWS, projetada para alta disponibilidade, resiliência a falhas e escalabilidade dinâmica, atendendo aos requisitos do desafio final (Fundamentos de Cloud, Soluções de Rede, Soluções de Dados e Soluções Digitais — Escalabilidade e Elasticidade).

## Diagrama da Arquitetura

![Arquitetura da Solução](arquitetura.png)

---

## Como a requisição flui na arquitetura (passo a passo)

1. **Usuário → Internet Gateway**
   O cliente faz uma requisição HTTP/HTTPS pela internet pública. Ela entra na `VPC ECOMMERCE-VPC (10.0.0.0/16)` através do **Internet Gateway**, o único ponto de entrada/saída de tráfego da internet para a VPC.

2. **Internet Gateway → Application Load Balancer (ALB)**
   O tráfego é roteado para o **Application Load Balancer**, posicionado nas **subnets públicas** das zonas **AZ-A** e **AZ-B**. O ALB é protegido pelo **SG-ALB**, que libera apenas as portas **80/443** vindas da internet — nenhuma outra porta ou origem é aceita.

3. **ALB → Auto Scaling Group (EC2 Go-ecommerce-ms)**
   O ALB distribui (balanceia) as requisições entre as instâncias EC2 registradas no **Auto Scaling Group**, que ficam em **subnets privadas de aplicação**, replicadas em **AZ-A** e **AZ-B**. O ASG mantém **mínimo de 3 e máximo de 6 instâncias** Linux, escalando automaticamente conforme a demanda de CPU/tráfego.
   As instâncias só aceitam conexão do ALB: o **SG-APP** libera a porta da aplicação exclusivamente para origens do **SG-ALB**, bloqueando qualquer acesso direto da internet às VMs.

4. **EC2 → RDS (autenticação via IAM)**
   Cada instância EC2 possui uma **IAM Role** anexada (`rds-db:connect`), que concede permissão de leitura/escrita no banco de dados sem necessidade de credenciais fixas (usuário/senha) embutidas na aplicação — a autenticação é feita via IAM Database Authentication.

5. **EC2 → RDS Multi-AZ (Primary/Standby)**
   A aplicação acessa o banco de dados gerenciado (**Amazon RDS**), localizado em **subnets privadas de banco de dados**, isoladas da internet. O **SG-DB** libera as portas **5432/3306** apenas para origens do **SG-APP**, ou seja, somente as instâncias da aplicação alcançam o banco.
   O RDS está configurado em **Multi-AZ**: uma instância **Primary** ativa na **AZ-A** e uma instância **Standby** na **AZ-B**, com **replicação síncrona** entre elas. Em caso de falha da AZ-A (ou da instância primária), ocorre failover automático para a Standby, garantindo continuidade do serviço.

6. **Caminho de retorno**
   A resposta segue o caminho inverso: RDS → EC2 → ALB → Internet Gateway → Usuário, sempre dentro dos limites de rede definidos pelos Security Groups, sem expor o banco de dados ou as instâncias diretamente à internet.

---

## Checklist dos requisitos do PDF

### 1. (Obrigatório) Desenho da Arquitetura de Soluções

- [x] Uso de múltiplas zonas de disponibilidade (AZ-A e AZ-B) para garantir continuidade do serviço mesmo em caso de falha de uma zona.
- [x] Balanceamento de carga (Application Load Balancer) para distribuir o tráfego entre as instâncias.
- [x] Escalonamento automático das VMs (Auto Scaling Group), com mínimo de 3 e máximo de 6 instâncias, usando imagens Linux.
- [x] Provisão de um serviço de banco de dados gerenciado (PaaS) — Amazon RDS Multi-AZ — garantindo alta disponibilidade e segurança dos dados.
- [x] Configuração de controle de acesso (IAM Role `rds-db:connect`) para que as VMs tenham permissão de leitura e escrita no banco de dados.

### 2. (Opcional) Provisionamento das Máquinas Virtuais e Configuração do Banco de Dados

- [ ] Criação real das VMs nas múltiplas zonas de disponibilidade (Linux) na conta AWS.
- [ ] Configuração efetiva do Load Balancer na infraestrutura provisionada.
- [ ] Implementação prática do escalonamento automático (3 a 6 instâncias) na conta AWS.
- [ ] Restrição de acesso via firewall (Security Groups) aplicada na infraestrutura real — **já definida no desenho** (SG-ALB, SG-APP, SG-DB), pendente de provisionamento efetivo.
- [ ] Provisionamento real do RDS com replicação multi-regional e backups automáticos configurados na conta AWS.
- [ ] Políticas de IAM aplicadas e validadas na conta AWS — **já definida no desenho**, pendente de provisionamento efetivo.

> Este item é opcional segundo o enunciado. Nesta entrega, a infraestrutura foi **projetada e documentada** no diagrama, mas não foi efetivamente provisionada em uma conta cloud.

### 3. (Opcional) Implementação de Segurança e Monitoramento

- [ ] Políticas de IAM provisionadas controlando o acesso das VMs ao banco de dados — **desenhada na arquitetura**, não provisionada.
- [ ] Logs e monitoramento (ex.: AWS CloudWatch) habilitados para acompanhar desempenho, segurança e falhas.

### 4. Documentação e Entregáveis

- [x] **(Obrigatório)** Diagrama da Arquitetura, incluindo VMs, balanceador de carga, banco de dados e mecanismo de failover (RDS Multi-AZ Primary/Standby).
- [ ] **(Opcional)** Capturas de tela das configurações de serviços, permissões de IAM, segurança, escalabilidade e recuperação de desastres.

### Resumo dos Entregáveis

- [x] 1. Arquitetura da Solução (Diagrama).
- [ ] 2. (Opcional) Provisão da Infraestrutura (IaaS) para as máquinas.
- [ ] 3. (Opcional) Provisão da Infraestrutura (PaaS) para persistência.
- [ ] 4. (Opcional) Provisionamento dos Elementos de Segurança.

---

## Componentes da Arquitetura

| Componente | Função |
|---|---|
| VPC `ECOMMERCE-VPC` (10.0.0.0/16) | Isolamento de rede de toda a solução |
| Internet Gateway | Ponto de entrada/saída de tráfego da internet |
| Application Load Balancer (ALB) | Distribui requisições entre as instâncias da aplicação, em subnets públicas (AZ-A/AZ-B) |
| Auto Scaling Group | Escala as instâncias EC2 (Linux) de 3 a 6, em subnets privadas de aplicação (AZ-A/AZ-B) |
| EC2 Go-ecommerce-ms | Aplicação da loja virtual, executada nas instâncias gerenciadas pelo ASG |
| Amazon RDS Multi-AZ | Banco de dados gerenciado (PaaS) com Primary (AZ-A) e Standby (AZ-B) em replicação síncrona |
| SG-ALB | Libera 80/443 apenas da internet |
| SG-APP | Libera a porta da aplicação apenas para origem SG-ALB |
| SG-DB | Libera 5432/3306 apenas para origem SG-APP |
| IAM Role `rds-db:connect` | Concede às EC2 acesso de leitura/escrita ao RDS sem credenciais fixas |
