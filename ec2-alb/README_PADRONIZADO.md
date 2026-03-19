# Terraform AWS EC2 + ALB Module

## 📌 Visão Geral
Este repositório contém um **módulo Terraform altamente parametrizável** para provisionamento de infraestrutura AWS utilizando:

- EC2 / Auto Scaling Group (ASG)
- Application Load Balancer (ALB)
- Target Groups, Listeners e Rules
- HTTPS com ACM (SNI)
- Health Checks, Stickiness
- Security Groups dinâmicos
- Discos EBS (root e extras)
- Auto Scaling baseado em métricas (CPU, memória e customizadas)
- Backend remoto S3

Todo o conteúdo original foi **mantido**, apenas **organizado, padronizado e documentado**.

---

## 📂 Estrutura do Repositório
```text
terraform-aws-ec2-lb/
├── modules/
│   └── ec2-lb/
│       ├── main.tf
│       ├── variables.tf
│       ├── outputs.tf
├── main.tf
├── variables.tf
├── outputs.tf
├── backend.tf
└── README.md
```

---

## 🔐 Backend Remoto (S3)

### Criar bucket para o tfstate
```bash
aws s3 mb s3://meu-tfstate-bucket
```

### backend.tf
```hcl
terraform {
  backend "s3" {
    bucket = "meu-tfstate-bucket"
    key    = "terraform/ec2-lb/terraform.tfstate"
    region = "us-east-1"
  }
}
```

📌 Recomenda-se usar **DynamoDB** para lock em produção.

---

## 🧩 Módulo `ec2-lb`

### Recursos suportados
- EC2 ou ASG
- ALB com múltiplos listeners
- HTTP → HTTPS Redirect
- Certificados ACM (SNI)
- Target Groups múltiplos
- Health checks customizados
- Sticky sessions
- Security Groups dinâmicos
- Volumes EBS (root + extras)
- User Data para formatação/mount
- Auto Scaling Policies (CPU, Memória, Custom)

---

## ⚙️ Exemplo de Uso do Módulo

```hcl
provider "aws" {
  region = "us-east-1"
}

module "ec2_lb" {
  source = "github.com/<SEU_USUARIO>/<SEU_REPOSITORIO>//modules/ec2-lb?ref=v1.0.0"

  vpc_id   = "vpc-123456"
  subnets = ["subnet-abc123", "subnet-def456"]

  ami_id        = "ami-0c55b159cbfafe1f0"
  instance_type = "t2.micro"
  key_name      = "minha-key"

  asg_min_size         = 2
  asg_max_size         = 5
  asg_desired_capacity = 3

  volume_size = 50
  volume_type = "gp3"

  certificates = [
    "arn:aws:acm:us-east-1:123456789012:certificate/abcd",
    "arn:aws:acm:us-east-1:123456789012:certificate/efgh"
  ]

  target_groups = [
    {
      name     = "ec2-tg"
      port     = 80
      protocol = "HTTP"
      health_check = {
        path                = "/health"
        port                = 80
        protocol            = "HTTP"
        interval            = 20
        timeout             = 5
        healthy_threshold   = 2
        unhealthy_threshold = 2
      }
      stickiness = {
        enabled  = true
        type     = "lb_cookie"
        duration = 300
      }
    }
  ]

  listeners = [
    { port = 80,  protocol = "HTTP",  target_group = "ec2-tg" },
    { port = 443, protocol = "HTTPS", target_group = "ec2-tg" }
  ]
}
```

---

## 🔄 Inicialização
```bash
terraform init
terraform plan
terraform apply -auto-approve
```

---

## 📤 Outputs
- IPs públicos / DNS
- DNS do Load Balancer
- Informações de Target Groups

---

## 🏷️ Versionamento com Git Tags

```bash
git checkout main
git pull origin main

git tag -a v1.0.0 -m "Primeira versão estável do módulo"

git push origin v1.0.0
```

---

## ✅ Resultado Final
- Infraestrutura escalável
- HTTPS com múltiplos domínios
- Balanceamento inteligente
- Totalmente reutilizável
- Pronto para produção

---

📌 **Observação**: Este README foi padronizado sem remoção de código Terraform.
