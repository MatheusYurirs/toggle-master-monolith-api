
raw
Readme · MD
# ToggleMaster — Fase 01
 
![Python](https://img.shields.io/badge/Python-3776AB?style=for-the-badge&logo=python&logoColor=white)
![Flask](https://img.shields.io/badge/Flask-000000?style=for-the-badge&logo=flask&logoColor=white)
![AWS](https://img.shields.io/badge/AWS-FF9900?style=for-the-badge&logo=amazonaws&logoColor=white)
![PostgreSQL](https://img.shields.io/badge/PostgreSQL-4169E1?style=for-the-badge&logo=postgresql&logoColor=white)
 
> **FIAP POSTECH — DevOps e Arquitetura Cloud**  
> Tech Challenge · Fase 01
 
---
 
## Sobre o Projeto
 
O **ToggleMaster** é uma plataforma centralizada para gerenciamento de **Feature Flags** (Feature Toggles), que permite ativar ou desativar funcionalidades em produção de forma controlada, sem necessidade de novos deploys.
 
Nesta primeira fase, implementei um **MVP monolítico** (Flask + PostgreSQL) com deploy manual em infraestrutura AWS, aplicando princípios de segurança, isolamento de rede e boas práticas de arquitetura cloud.
 
---
 
## Arquitetura AWS
 
A infraestrutura foi projetada com foco em **isolamento por camadas**, **menor privilégio** e **separação clara entre recursos públicos e privados**.
 
```
                          Internet
                              │
                              ▼
                    [ Internet Gateway ]
                              │
        ╔═════════════════════╪═════════════════════╗
        ║        VPC  10.0.0.0/16                   ║
        ║                     │                     ║
        ║   ┌─────────────────▼──────────────────┐  ║
        ║   │   SUBNET PÚBLICA  10.0.1.0/24      │  ║
        ║   │   us-east-1a                       │  ║
        ║   │                                    │  ║
        ║   │        ┌──────────────────┐        │  ║
        ║   │        │  EC2  t3.micro   │        │  ║
        ║   │        │  Flask + Gunicorn│        │  ║
        ║   │        │  porta 5000      │        │  ║
        ║   │        └────────┬─────────┘        │  ║
        ║   │   SG-EC2:       │                  │  ║
        ║   │   SSH :22 ──► IP fixo              │  ║
        ║   │   HTTP :5000 ──► 0.0.0.0/0         │  ║
        ║   └─────────────────┼──────────────────┘  ║
        ║                     │  (tráfego interno)  ║
        ║   ┌─────────────────▼──────────────────┐  ║
        ║   │   SUBNETS PRIVADAS                 │  ║
        ║   │   10.0.2.0/24 (us-east-1a)         │  ║
        ║   │   10.0.3.0/24 (us-east-1b)         │  ║
        ║   │                                    │  ║
        ║   │        ┌──────────────────┐        │  ║
        ║   │        │  RDS db.t3.micro │        │  ║
        ║   │        │  PostgreSQL      │        │  ║
        ║   │        │  Single-AZ       │        │  ║
        ║   │        └──────────────────┘        │  ║
        ║   │   SG-RDS:                          │  ║
        ║   │   :5432 ──► somente SG-EC2         │  ║
        ║   └────────────────────────────────────┘  ║
        ╚═══════════════════════════════════════════╝
```
 
> As subnets privadas **não possuem rota para a internet** — o RDS é inacessível externamente por design.
 
### Recursos Provisionados
 
| Recurso | Configuração |
|---------|-------------|
| VPC | CIDR `10.0.0.0/16` |
| Subnet pública | `10.0.1.0/24` — us-east-1a |
| Subnet privada 1 | `10.0.2.0/24` — us-east-1a |
| Subnet privada 2 | `10.0.3.0/24` — us-east-1b |
| Internet Gateway | Associado à subnet pública |
| Route Table pública | `0.0.0.0/0 → IGW` |
| Route Table privada | Sem rota para internet |
| EC2 | `t3.micro` · Amazon Linux 2 · IP público habilitado |
| RDS | `db.t3.micro` · PostgreSQL · Single-AZ · DB Subnet Group em 2 AZs |
| Security Group EC2 | SSH :22 (IP fixo) · HTTP :5000 (0.0.0.0/0) |
| Security Group RDS | PostgreSQL :5432 somente via SG-EC2 |
 
---
 
## Provisionamento da Infraestrutura
 
### Ordem de criação no Console AWS
 
```
VPC
 └─► Subnets (1 pública + 2 privadas em AZs distintas)
      └─► Internet Gateway (associar à VPC)
           └─► Route Tables (pública com 0.0.0.0/0 → IGW; privada sem saída)
                └─► Security Groups (SG-EC2 e SG-RDS)
                     └─► DB Subnet Group (ambas as subnets privadas)
                          └─► RDS PostgreSQL
                               └─► EC2 (subnet pública, IP público habilitado)
```
 
### Configuração da EC2 via SSH
 
```bash
# Conectar à instância
ssh -i sua-chave.pem ec2-user@<IP_PUBLICO_EC2>
 
# Instalar dependências do sistema
sudo yum update -y
sudo yum install python3 python3-pip git -y
 
# Clonar o repositório
git clone https://github.com/MatheusYurirs/toggle-master-monolith-api.git
cd toggle-master-monolith-api
 
# Criar ambiente virtual e instalar dependências Python
python3 -m venv venv
source venv/bin/activate
pip install -r requirements.txt
```
 
### Variáveis de Ambiente — Persistência para o Gunicorn
 
> ⚠️ `export` simples no terminal **não propaga** para processos iniciados com `sudo`.  
> A solução é persistir em `/etc/environment` e usar o flag `-E` no Gunicorn:
 
```bash
# Persistir variáveis no sistema
sudo tee -a /etc/environment <<EOF
DB_HOST=<ENDPOINT_RDS>
DB_PORT=5432
DB_NAME=togglemaster
DB_USER=<SEU_USUARIO>
DB_PASSWORD=<SUA_SENHA>
EOF
 
# Recarregar no shell atual
source /etc/environment
```
 
### Inicialização e Execução
 
```bash
# Inicializar o schema do banco de dados
flask init-db
 
# Subir a aplicação com Gunicorn (herda o ambiente completo)
sudo -E venv/bin/gunicorn -w 2 -b 0.0.0.0:5000 app:app
```
 
Acesse em: `http://<IP_PUBLICO_EC2>:5000`
 
---
 
## Segurança
 
A arquitetura segue o princípio de **defesa em profundidade**: cada camada acessa apenas o que precisa, sem exposição desnecessária.
 
| Decisão | Justificativa |
|---------|--------------|
| RDS em subnets privadas | Banco inacessível diretamente da internet |
| SG-RDS referencia SG-EC2 (não IP) | Acesso por identidade de recurso — resiliente a mudanças de IP |
| SSH restrito a IP fixo | Reduz superfície de ataque ao acesso administrativo |
| Tráfego EC2 ↔ RDS exclusivamente dentro da VPC | Dados do banco nunca trafegam pela internet |
| Credenciais via variáveis de ambiente | Nenhuma credencial hardcoded ou versionada no repositório |
| Subnets privadas sem rota para internet | Camada de dados completamente isolada por roteamento |
 
### Fluxo de Rede
 
```
Usuário
   │
   ▼ HTTPS/HTTP
Internet Gateway
   │
   ▼
EC2 (Subnet Pública) — SG libera :5000
   │
   │ Tráfego interno VPC
   ▼
RDS (Subnet Privada) — SG aceita :5432 somente do SG-EC2
```
 
---
 
## Análise 12-Factor App
 
Avaliação da aderência da aplicação aos [12 fatores](https://12factor.net/pt_br/) no contexto desta fase.
 
### ✅ Fatores Atendidos
 
| Fator | Como atende |
|-------|------------|
| **F1 · Codebase** | Repositório único no GitHub; mesmo código para dev e produção, sem bifurcações |
| **F2 · Dependências** | `requirements.txt` com versões fixas; Dockerfile garante ambiente reproduzível e isolado |
| **F4 · Backing Services** | PostgreSQL consumido via variáveis de ambiente (`DB_HOST`, `DB_PORT`...); troca entre Docker local e RDS sem alterar nenhuma linha de código |
| **F6 · Processos** | Aplicação completamente stateless — todo estado persiste exclusivamente no banco; reiniciar a instância não perde dados de negócio |
| **F7 · Port Binding** | Flask expõe o serviço via binding direto da porta 5000, sem dependência de servidor externo como Apache ou Tomcat |
 
### ⚠️ Fatores Parcialmente Atendidos
 
| Fator | O que já funciona | O que falta |
|-------|-------------------|-------------|
| **F3 · Config** | Variáveis de ambiente usadas corretamente na aplicação | Credenciais ainda expostas no `docker-compose.yaml`; corrigir com `.env` + `.gitignore` |
| **F8 · Concorrência** | App e banco em containers separados, escalonáveis de forma independente | Escalar horizontalmente a camada de aplicação ainda exige um ALB à frente de múltiplas instâncias EC2 |
 
### 🔧 Fatores a Evoluir nas Próximas Fases
 
| Fator | Gap atual | Evolução prevista |
|-------|-----------|-------------------|
| **F5 · Build/Release/Run** | Build e run integrados via docker-compose; sem pipeline formal nem artefatos imutáveis | Pipeline CI/CD com GitHub Actions — build → tag de release → deploy (Fase 3) |
| **F9 · Descartabilidade** | Infraestrutura criada manualmente; não pode ser recriada automaticamente | IaC com Terraform — provisionar toda a stack do zero em minutos (Fase 2) |
| **F10 · Paridade Dev/Prod** | Dev usa Docker local; prod usa EC2 + RDS sem padronização formal entre ambientes | Padronizar com containers + variáveis controladas por pipeline de CI/CD |
| **F11 · Logs** | Logs não direcionados ao stdout de forma estruturada | Integração com CloudWatch Logs via agente ou driver de container |
| **F12 · Admin Processes** | `flask init-db` executado manualmente no terminal | Tornar step isolado e automatizado no pipeline de deploy |
 
> **Placar desta fase:** 5 atendidos · 2 parciais · 5 a evoluir.  
> Os gaps não são falhas — são o roteiro natural das Fases 2 e 3, onde IaC, CI/CD e observabilidade entram em cena.
 
---
 
## Estimativa de Custos AWS
 
Região: **us-east-1 (N. Virginia)** — escolhida por ser a mais econômica da AWS com maior cobertura de serviços. Para produção com usuários brasileiros, `sa-east-1 (São Paulo)` é recomendada para reduzir latência.
 
| Serviço | On-Demand / mês |
|---------|:--------------:|
| EC2 `t3.micro` | $7,59 |
| RDS `db.t3.micro` PostgreSQL | $15,44 |
| IP Público | $3,65 |
| Transferência de dados (~10 GB) | $0,90 |
| VPC, Subnets, IGW, Route Tables, SGs | $0,00 |
| **Total estimado** | **$27,58** |
 
> 💡 **Com AWS Free Tier (primeiro ano):** EC2 e RDS cobertos pelas 750h/mês gratuitas.  
> Custo efetivo: **$4,55/mês** — apenas IP público e transferência de dados.
 
---
 
## Desafios e Decisões Técnicas
 
### DB Subnet Group rejeitado com zona única
A AWS exige que o DB Subnet Group contenha subnets em **pelo menos duas Availability Zones**. A configuração inicial com apenas `us-east-1a` foi recusada pelo console. Solução: criar a segunda subnet privada em `us-east-1b` e incluir ambas no grupo — o que também prepara a arquitetura para futura alta disponibilidade com Multi-AZ.
 
### Porta 5000 inacessível externamente
A aplicação estava rodando normalmente na EC2, mas inacessível pelo navegador. Causa: ausência da regra de ingress `:5000 → 0.0.0.0/0` no Security Group da EC2. Regra adicionada, acesso normalizado imediatamente.
 
### RDS criado sem `DB Name`
A instância RDS foi criada sem preencher o campo `DB Name`. A aplicação inicializava sem erros, mas falhava silenciosamente na conexão. Solução: recriar a instância com o nome do banco definido — detalhe pequeno, impacto total.
 
### Variáveis de ambiente não propagando para o Gunicorn
`export` no terminal não propaga para processos filhos iniciados com `sudo`. O Gunicorn subia sem as variáveis de banco configuradas. Solução: persistir em `/etc/environment` e executar com `sudo -E venv/bin/gunicorn` para herdar o ambiente completo do sistema.
 
### Username `user` reservado no PostgreSQL
`user` é palavra reservada no PostgreSQL e não pode ser nome de usuário de uma instância RDS. Identificado na criação; resolvido escolhendo um username alternativo.
 
---
 
## Execução Local
 
```bash
git clone https://github.com/MatheusYurirs/toggle-master-monolith-api.git
cd toggle-master-monolith-api
 
# Criar .env (nunca versionar)
cp .env.example .env  # ajuste as variáveis conforme necessário
 
# Subir aplicação + banco via Docker Compose
docker compose up --build
 
# Acesse: http://localhost:5000
```
 
---

*FIAP POSTECH — Pós-Graduação em DevOps e Arquitetura Cloud · Matheus Yuri Rodrigues da Silva · 2025*
