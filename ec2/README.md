
# Documentação de umm modulo EC2 completo.

---
## **Estrutura do Projeto**

```bash
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

---

### **Dando inicio ao provisionamento dos requisitos**
<details>
<sumary>1º Criar Bucket S3 para o tfstate remoto e configurar:</sumary>

No console da AWS ou via CLI:
```bash
aws s3 mb s3://meu-tfstate-bucket
```

Configurar o backend remoto (backend.tf).
```bash
terraform {
  backend "s3" {
    bucket = "meu-tfstate-bucket"
    key    = "terraform/ec2-lb/terraform.tfstate"
    region = "us-east-1"
  }
}
```
</details>
##### 3º Criar o módulo modules/ec2-lb/main.tf #####
####################################################

resource "aws_security_group" "ec2_sg" {
  name        = "ec2-sg"
  description = "Allow HTTP and SSH"
  vpc_id      = var.vpc_id

  ingress {
    from_port   = 22
    to_port     = 22
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  ingress {
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
}

resource "aws_instance" "ec2" {
  ami           = var.ami_id
  instance_type = var.instance_type
  security_groups = [aws_security_group.ec2_sg.name]

  tags = {
    Name = "Terraform-EC2"
  }
}

resource "aws_lb" "app_lb" {
  name               = "app-lb"
  internal           = false
  load_balancer_type = "application"
  security_groups    = [aws_security_group.ec2_sg.id]
  subnets            = var.subnets

  tags = {
    Name = "App-LB"
  }
}

##### 4º Definir variáveis (modules/ec2-lb/variables.tf) #####
##############################################################

variable "vpc_id" {}
variable "subnets" {
  type = list(string)
}
variable "ami_id" {}
variable "instance_type" {
  default = "t2.micro"
}

##### 5º Outputs (modules/ec2-lb/outputs.tf) #####
##################################################

output "ec2_public_ip" {
  value = aws_instance.ec2.public_ip
}

output "lb_dns_name" {
  value = aws_lb.app_lb.dns_name
}


#######################################
##### 6º Usar o módulo no main.tf #####
#######################################

provider "aws" {
  region = "us-east-1"
}

module "ec2_lb" {
  source        = "./modules/ec2-lb"
  vpc_id        = "vpc-123456"
  subnets       = ["subnet-abc123", "subnet-def456"]
  ami_id        = "ami-0c55b159cbfafe1f0"
  instance_type = "t2.micro"
}

##### 7º Inicializar e aplicar #####
####################################

terraform init
terraform plan
terraform apply -auto-approve

##### 8º Resultado esperado #####
#################################

- EC2 criada com IP público.
- Security Group permitindo SSH (22) e HTTP (80).
- Load Balancer criado e associado à EC2.
- tfstate armazenado remotamente no S3.

📌 Observações
- Não utilizamos DynamoDB para lock do state, apenas o bucket S3.
- É altamente recomendado usar DynamoDB em ambientes de produção para evitar concorrência.
- Este exemplo é simplificado para fins de aprendizado.




* Estrutura do projeto consumidor

terraform-ec2-lb-deploy/
├── main.tf
├── variables.tf
├── outputs.tf
├── backend.tf

##### 1ª Configurar o backend remoto (backend.tf) #####
#######################################################

terraform {
  backend "s3" {
    bucket = "meu-tfstate-bucket"
    key    = "terraform/ec2-lb/terraform.tfstate"
    region = "us-east-1"
  }
}

##### 2ª Usar o módulo diretamente do GitHub (main.tf) #####
############################################################

provider "aws" {
  region = "us-east-1"
}

module "ec2_lb" {
  source        = "github.com/<SEU_USUARIO>/<SEU_REPOSITORIO>//modules/ec2-lb"
  vpc_id        = "vpc-123456"
  subnets       = ["subnet-abc123", "subnet-def456"]
  ami_id        = "ami-0c55b159cbfafe1f0"
  instance_type = "t2.micro"
}

OBS:  Note que o SOURCE aponta para o repositório GitHub.
Se quiser travar em uma versão/tag específica, basta adicionar "?ref=v1.0.0" ao final:
Exemplo: "github.com/<SEU_USUARIO>/<SEU_REPOSITORIO>//modules/ec2-lb?ref=v1.0.0"


##### 3ª Variávels (variables.tf) #####
#######################################

variable "vpc_id" {}
variable "subnets" {
  type = list(string)
}
variable "ami_id" {}
variable "instance_type" {
  default = "t2.micro"
}

##### 4ª Outputs (outputs.tf) #####
###################################

output "ec2_public_ip" {
  value = module.ec2_lb.ec2_public_ip
}

output "lb_dns_name" {
  value = module.ec2_lb.lb_dns_name
}


* Comandos para executar


# Inicializar o projeto e baixar o módulo do GitHub
terraform init

# Validar o plano
terraform plan

# Aplicar a infraestrutura
terraform apply -auto-approve


📌 Resultado
- O Terraform buscará o módulo diretamente no GitHub.
- Criará a EC2, o Security Group e o Load Balancer.
- O tfstate será armazenado remotamente no bucket S3.

Quer que eu te mostre também como publicar o módulo no GitHub com versionamento (tags e releases) para que você possa consumir versões específicas de forma mais organizada?


###############
###############

Para provisionar 3 instâncias EC2 usando o módulo que você publicou no GitHub, basta ajustar o módulo para suportar múltiplas instâncias e passar o input adequado.
Existem duas formas comuns de fazer isso


* Opção 1: Usando "count" dentro do módulo
No seu módulo (modules/ec2-lb/main.tf), altere o recurso da EC2:

resource "aws_instance" "ec2" {
  count         = var.instance_count
  ami           = var.ami_id
  instance_type = var.instance_type
  security_groups = [aws_security_group.ec2_sg.name]

  tags = {
    Name = "Terraform-EC2-${count.index}"
  }
}


E no "variables.tf" do módulo:

variable "instance_count" {
  type    = number
  default = 1
}


* Opção 2: Passando lista de instâncias (mais flexível)

resource "aws_instance" "ec2" {
  for_each      = toset(var.instances)
  ami           = var.ami_id
  instance_type = var.instance_type
  security_groups = [aws_security_group.ec2_sg.name]

  tags = {
    Name = "Terraform-EC2-${each.key}"
  }
}

E nno Variables.tf:

variable "instances" {
  type = list(string)
}


Projeto consumidor (buscando módulo no GitHub)
No seu "main.tf":

provider "aws" {
  region = "us-east-1"
}

module "ec2_lb" {
  source         = "github.com/<SEU_USUARIO>/<SEU_REPOSITORIO>//modules/ec2-lb?ref=v1.0.0"
  vpc_id         = "vpc-123456"
  subnets        = ["subnet-abc123", "subnet-def456"]
  ami_id         = "ami-0c55b159cbfafe1f0"
  instance_type  = "t2.micro"
  instance_count = 3   # 👈 Aqui você define quantas instâncias quer
}


Resultado
• 	Serão criadas 3 instâncias EC2.
• 	Todas estarão associadas ao Security Group e ao Load Balancer.
• 	O tfstate continuará armazenado no bucket S3 remoto.


###############
###############

Para passar inputs de key pair e adicionar um disco extra de 30 GB na sua instância EC2 via Terraform, você precisa ajustar o módulo para aceitar essas variáveis e depois fornecer os valores no main.tf do projeto consumidor.


1º Alterar o módulo para suportar key_name e disco extra 
No arquivo modules/ec2-lb/main.tf

resource "aws_instance" "ec2" {
  count         = var.instance_count
  ami           = var.ami_id
  instance_type = var.instance_type
  key_name      = var.key_name
  security_groups = [aws_security_group.ec2_sg.name]

  root_block_device {
    volume_size = 30   # 👈 disco extra de 30 GB
    volume_type = "gp2"
  }

  tags = {
    Name = "Terraform-EC2-${count.index}"
  }
}


2º  Definir variáveis no módulo (variables.tf)

variable "key_name" {
  description = "Nome da key pair existente na AWS"
  type        = string
}

variable "instance_count" {
  description = "Número de instâncias EC2"
  type        = number
  default     = 1
}


3º Consumir o módulo no projeto (main.tf)

provider "aws" {
  region = "us-east-1"
}

module "ec2_lb" {
  source         = "github.com/<SEU_USUARIO>/<SEU_REPOSITORIO>//modules/ec2-lb?ref=v1.0.0"
  vpc_id         = "vpc-123456"
  subnets        = ["subnet-abc123", "subnet-def456"]
  ami_id         = "ami-0c55b159cbfafe1f0"
  instance_type  = "t2.micro"
  instance_count = 3          # 👈 provisiona 3 instâncias
  key_name       = "minha-key" # 👈 key pair já criada na AWS
}


###############
###############


Para que o consumidor do módulo () consiga passar como input o tamanho do disco e o tipo de disco, você precisa expor essas variáveis no módulo e depois referenciá-las no projeto.


1º Ajustar o módulo (modules/ec2-lb/variables.tf)
Adicione variáveis para o disco:

variable "volume_size" {
  description = "Tamanho do disco root em GB"
  type        = number
  default     = 30
}

variable "volume_type" {
  description = "Tipo do disco root (gp2, gp3, io1, etc.)"
  type        = string
  default     = "gp2"
}


2º  Usar as variáveis no recurso EC2 (modules/ec2-lb/main.tf)

resource "aws_instance" "ec2" {
  count         = var.instance_count
  ami           = var.ami_id
  instance_type = var.instance_type
  key_name      = var.key_name
  security_groups = [aws_security_group.ec2_sg.name]

  root_block_device {
    volume_size = var.volume_size
    volume_type = var.volume_type
  }

  tags = {
    Name = "Terraform-EC2-${count.index}"
  }
}


3º Consumidor do módulo (main.tf)

Agora, no projeto que consome o módulo, você passa os inputs:

provider "aws" {
  region = "us-east-1"
}

module "ec2_lb" {
  source         = "github.com/<SEU_USUARIO>/<SEU_REPOSITORIO>//modules/ec2-lb?ref=v1.0.0"
  vpc_id         = "vpc-123456"
  subnets        = ["subnet-abc123", "subnet-def456"]
  ami_id         = "ami-0c55b159cbfafe1f0"
  instance_type  = "t2.micro"
  instance_count = 3
  key_name       = "minha-key"

  # 👇 Inputs para disco
  volume_size    = 50   # tamanho do disco root em GB
  volume_type    = "gp3" # tipo do disco root
}

📌 Resultado
- Serão criadas 3 instâncias EC2.
- Cada instância terá um disco root de 50 GB do tipo gp3.
- Todas usarão a key pair informada para acesso SSH.
- O tfstate continuará armazenado no bucket S3 remoto.


###############
###############

Vamos estender o módulo para suportar volumes EBS adicionais além do disco root. 
Assim você poderá passar inputs no consumidor () para definir tamanho, tipo e até quantos discos extras quiser.


1º Ajustar o módulo (modules/ec2-lb/variables.tf)
Adicione uma variável para lista de volumes extras:

variable "extra_volumes" {
  description = "Lista de volumes EBS adicionais"
  type = list(object({
    size = number
    type = string
  }))
  default = []
}


2º Usar a variável no recurso EC2 (modules/ec2-lb/main.tf)

resource "aws_instance" "ec2" {
  count         = var.instance_count
  ami           = var.ami_id
  instance_type = var.instance_type
  key_name      = var.key_name
  security_groups = [aws_security_group.ec2_sg.name]

  root_block_device {
    volume_size = var.volume_size
    volume_type = var.volume_type
  }

  dynamic "ebs_block_device" {
    for_each = var.extra_volumes
    content {
      device_name = "/dev/sd${chr(98 + each.key)}" # gera nomes /dev/sdb, /dev/sdc...
      volume_size = each.value.size
      volume_type = each.value.type
    }
  }

  tags = {
    Name = "Terraform-EC2-${count.index}"
  }
}

3º Consumidor do módulo (main.tf)
Agora você pode passar os inputs no projeto que consome o módulo:

provider "aws" {
  region = "us-east-1"
}

module "ec2_lb" {
  source         = "github.com/<SEU_USUARIO>/<SEU_REPOSITORIO>//modules/ec2-lb?ref=v1.0.0"
  vpc_id         = "vpc-123456"
  subnets        = ["subnet-abc123", "subnet-def456"]
  ami_id         = "ami-0c55b159cbfafe1f0"
  instance_type  = "t2.micro"
  instance_count = 3
  key_name       = "minha-key"

  # Disco root
  volume_size    = 50
  volume_type    = "gp3"

  # Volumes extras
  extra_volumes = [
    { size = 30, type = "gp2" },
    { size = 100, type = "gp3" }
  ]
}


Resultado
- Cada instância terá um disco root de 50 GB gp3.
- Além disso, cada instância terá 2 volumes extras:
- Um de 30 GB gp2.
- Outro de 100 GB gp3.
- Todos os volumes serão anexados automaticamente às instâncias



###############
###############

Para que os volumes EBS adicionais sejam automaticamente formatados e montados dentro da instância, você pode usar o recurso "user_data" do Terraform. 
Esse campo permite passar um script que será executado na inicialização da EC2

1º Ajustar o recurso EC2 no módulo (modules/ec2-lb/main.tf)
Adicione o user_data com um script de inicialização:

resource "aws_instance" "ec2" {
  count         = var.instance_count
  ami           = var.ami_id
  instance_type = var.instance_type
  key_name      = var.key_name
  security_groups = [aws_security_group.ec2_sg.name]

  root_block_device {
    volume_size = var.volume_size
    volume_type = var.volume_type
  }

  dynamic "ebs_block_device" {
    for_each = var.extra_volumes
    content {
      device_name = "/dev/sd${chr(98 + each.key)}" # /dev/sdb, /dev/sdc...
      volume_size = each.value.size
      volume_type = each.value.type
    }
  }

  user_data = <<-EOF
              #!/bin/bash
              # Atualiza pacotes
              yum update -y

              # Formata e monta volumes extras
              for dev in /dev/sdb /dev/sdc; do
                mkfs -t ext4 $dev
                mkdir -p /mnt/$(basename $dev)
                mount $dev /mnt/$(basename $dev)
                echo "$dev /mnt/$(basename $dev) ext4 defaults,nofail 0 2" >> /etc/fstab
              done
              EOF

  tags = {
    Name = "Terraform-EC2-${count.index}"
  }
}resource "aws_instance" "ec2" {
  count         = var.instance_count
  ami           = var.ami_id
  instance_type = var.instance_type
  key_name      = var.key_name
  security_groups = [aws_security_group.ec2_sg.name]

  root_block_device {
    volume_size = var.volume_size
    volume_type = var.volume_type
  }

  dynamic "ebs_block_device" {
    for_each = var.extra_volumes
    content {
      device_name = "/dev/sd${chr(98 + each.key)}" # /dev/sdb, /dev/sdc...
      volume_size = each.value.size
      volume_type = each.value.type
    }
  }

  user_data = <<-EOF
              #!/bin/bash
              # Atualiza pacotes
              yum update -y

              # Formata e monta volumes extras
              for dev in /dev/sdb /dev/sdc; do
                mkfs -t ext4 $dev
                mkdir -p /mnt/$(basename $dev)
                mount $dev /mnt/$(basename $dev)
                echo "$dev /mnt/$(basename $dev) ext4 defaults,nofail 0 2" >> /etc/fstab
              done
              EOF

  tags = {
    Name = "Terraform-EC2-${count.index}"
  }
}



2º Consumidor do módulo (main.tf)

provider "aws" {
  region = "us-east-1"
}

module "ec2_lb" {
  source         = "github.com/<SEU_USUARIO>/<SEU_REPOSITORIO>//modules/ec2-lb?ref=v1.0.0"
  vpc_id         = "vpc-123456"
  subnets        = ["subnet-abc123", "subnet-def456"]
  ami_id         = "ami-0c55b159cbfafe1f0"
  instance_type  = "t2.micro"
  instance_count = 3
  key_name       = "minha-key"

  # Disco root
  volume_size    = 50
  volume_type    = "gp3"

  # Volumes extras
  extra_volumes = [
    { size = 30, type = "gp2" },
    { size = 100, type = "gp3" }
  ]
}


📌 Resultado
- Cada instância terá:
- Disco root de 50 GB gp3.
- Dois volumes extras: 30 GB gp2 e 100 GB gp3.
- O script user_data irá:
- Formatar os volumes extras em ext4.
- Criar diretórios (/mnt/sdb, /mnt/sdc).
- Montar os volumes automaticamente.
- Registrar no /etc/fstab para que sejam montados em cada reboot.


###############
###############

Vamos parametrizar os device names para que você possa controlar diretamente no "main.tf" do consumidor quais volumes serão anexados como /dev/sdb, /dev/sdc, etc.


1º Ajustar o módulo (modules/ec2-lb/variables.tf)
Defina a variável extra_volumes incluindo o campo device_name:

variable "extra_volumes" {
  description = "Lista de volumes EBS adicionais com device_name"
  type = list(object({
    device_name = string
    size        = number
    type        = string
  }))
  default = []
}


2º Usar a variável no recurso EC2 (modules/ec2-lb/main.tf)

resource "aws_instance" "ec2" {
  count         = var.instance_count
  ami           = var.ami_id
  instance_type = var.instance_type
  key_name      = var.key_name
  security_groups = [aws_security_group.ec2_sg.name]

  root_block_device {
    volume_size = var.volume_size
    volume_type = var.volume_type
  }

  dynamic "ebs_block_device" {
    for_each = var.extra_volumes
    content {
      device_name = each.value.device_name
      volume_size = each.value.size
      volume_type = each.value.type
    }
  }

  user_data = <<-EOF
              #!/bin/bash
              yum update -y

              # Formata e monta volumes extras
              for dev in ${join(" ", [for v in var.extra_volumes : v.device_name])}; do
                mkfs -t ext4 $dev
                mkdir -p /mnt/$(basename $dev)
                mount $dev /mnt/$(basename $dev)
                echo "$dev /mnt/$(basename $dev) ext4 defaults,nofail 0 2" >> /etc/fstab
              done
              EOF

  tags = {
    Name = "Terraform-EC2-${count.index}"
  }
}


Note que o "user_data" agora monta dinamicamente todos os devices passados no input.

3º Consumidor do módulo (main.tf)
Agora você controla os nomes dos devices diretamente:

provider "aws" {
  region = "us-east-1"
}

module "ec2_lb" {
  source         = "github.com/<SEU_USUARIO>/<SEU_REPOSITORIO>//modules/ec2-lb?ref=v1.0.0"
  vpc_id         = "vpc-123456"
  subnets        = ["subnet-abc123", "subnet-def456"]
  ami_id         = "ami-0c55b159cbfafe1f0"
  instance_type  = "t2.micro"
  instance_count = 3
  key_name       = "minha-key"

  # Disco root
  volume_size    = 50
  volume_type    = "gp3"

  # Volumes extras com device_name explícito
  extra_volumes = [
    { device_name = "/dev/sdb", size = 30, type = "gp2" },
    { device_name = "/dev/sdc", size = 100, type = "gp3" }
  ]
}


Resultado
- Cada instância terá um disco root de 50 GB gp3.
- Dois volumes extras serão anexados:
- /dev/sdb com 30 GB gp2.
- /dev/sdc com 100 GB gp3.
- O user_data formatará e montará automaticamente os volumes em /mnt/sdb e /mnt/sdc.
- Os volumes estarão configurados no /etc/fstab para serem montados em cada reboot.



###############
###############

Parametrizar também os pontos de montagem deixa o módulo muito mais flexível. 
Assim você pode dizer, por exemplo, que o /dev/sdb deve ser montado em /data e o /dev/sdc em /logs.


1º Ajustar o módulo (modules/ec2-lb/variables.tf)
Agora cada volume extra terá também o campo mount_point:

variable "extra_volumes" {
  description = "Lista de volumes EBS adicionais com device_name e mount_point"
  type = list(object({
    device_name  = string
    size         = number
    type         = string
    mount_point  = string
  }))
  default = []
}


2º sar a variável no recurso EC2 (modules/ec2-lb/main.tf)

resource "aws_instance" "ec2" {
  count         = var.instance_count
  ami           = var.ami_id
  instance_type = var.instance_type
  key_name      = var.key_name
  security_groups = [aws_security_group.ec2_sg.name]

  root_block_device {
    volume_size = var.volume_size
    volume_type = var.volume_type
  }

  dynamic "ebs_block_device" {
    for_each = var.extra_volumes
    content {
      device_name = each.value.device_name
      volume_size = each.value.size
      volume_type = each.value.type
    }
  }

  user_data = <<-EOF
              #!/bin/bash
              yum update -y

              # Formata e monta volumes extras
              %{ for vol in extra_volumes ~}
              mkfs -t ext4 ${vol.device_name}
              mkdir -p ${vol.mount_point}
              mount ${vol.device_name} ${vol.mount_point}
              echo "${vol.device_name} ${vol.mount_point} ext4 defaults,nofail 0 2" >> /etc/fstab
              %{ endfor ~}
              EOF

  tags = {
    Name = "Terraform-EC2-${count.index}"
  }
}


OBS: Aqui usamos template interpolation no  para iterar sobre os volumes extras e aplicar formatação/montagem conforme os inputs.



3º Consumidor do módulo (main.tf)
gora você controla device_name e mount_point diretamente:


provider "aws" {
  region = "us-east-1"
}

module "ec2_lb" {
  source         = "github.com/<SEU_USUARIO>/<SEU_REPOSITORIO>//modules/ec2-lb?ref=v1.0.0"
  vpc_id         = "vpc-123456"
  subnets        = ["subnet-abc123", "subnet-def456"]
  ami_id         = "ami-0c55b159cbfafe1f0"
  instance_type  = "t2.micro"
  instance_count = 3
  key_name       = "minha-key"

  # Disco root
  volume_size    = 50
  volume_type    = "gp3"

  # Volumes extras com device_name e mount_point
  extra_volumes = [
    { device_name = "/dev/sdb", size = 30, type = "gp2", mount_point = "/data" },
    { device_name = "/dev/sdc", size = 100, type = "gp3", mount_point = "/logs" }
  ]
}


Resultado
- Cada instância terá um disco root de 50 GB gp3.
- Dois volumes extras:
- /dev/sdb (30 GB gp2) montado em /data.
- /dev/sdc (100 GB gp3) montado em /logs.
- O user_data formatará e montará automaticamente os volumes nos pontos de montagem definidos.
- O /etc/fstab será atualizado para garantir que os volumes sejam montados em cada reboot.


###############
###############


vamos tornar o Security Group (SG) totalmente parametrizável via input, permitindo que você defina as regras de entrada (ingress) e saída (egress) diretamente no main.tf do consumidor.


1º Ajustar o módulo (modules/ec2-lb/variables.tf)
Defina variáveis para ingress e egress:

variable "ingress_rules" {
  description = "Lista de regras de entrada para o SG"
  type = list(object({
    from_port   = number
    to_port     = number
    protocol    = string
    cidr_blocks = list(string)
  }))
  default = []
}

variable "egress_rules" {
  description = "Lista de regras de saída para o SG"
  type = list(object({
    from_port   = number
    to_port     = number
    protocol    = string
    cidr_blocks = list(string)
  }))
  default = []
}


2º Usar as variáveis no recurso SG (modules/ec2-lb/main.tf)

resource "aws_security_group" "ec2_sg" {
  name        = "ec2-sg"
  description = "Security Group parametrizado"
  vpc_id      = var.vpc_id

  dynamic "ingress" {
    for_each = var.ingress_rules
    content {
      from_port   = ingress.value.from_port
      to_port     = ingress.value.to_port
      protocol    = ingress.value.protocol
      cidr_blocks = ingress.value.cidr_blocks
    }
  }

  dynamic "egress" {
    for_each = var.egress_rules
    content {
      from_port   = egress.value.from_port
      to_port     = egress.value.to_port
      protocol    = egress.value.protocol
      cidr_blocks = egress.value.cidr_blocks
    }
  }
}


3º Consumidor do módulo (main.tf)
Agora você pode passar as regras diretamente:

provider "aws" {
  region = "us-east-1"
}

module "ec2_lb" {
  source         = "github.com/<SEU_USUARIO>/<SEU_REPOSITORIO>//modules/ec2-lb?ref=v1.0.0"
  vpc_id         = "vpc-123456"
  subnets        = ["subnet-abc123", "subnet-def456"]
  ami_id         = "ami-0c55b159cbfafe1f0"
  instance_type  = "t2.micro"
  instance_count = 3
  key_name       = "minha-key"

  # Disco root
  volume_size    = 50
  volume_type    = "gp3"

  # Volumes extras
  extra_volumes = [
    { device_name = "/dev/sdb", size = 30, type = "gp2", mount_point = "/data" },
    { device_name = "/dev/sdc", size = 100, type = "gp3", mount_point = "/logs" }
  ]

  # Regras de entrada
  ingress_rules = [
    { from_port = 22, to_port = 22, protocol = "tcp", cidr_blocks = ["0.0.0.0/0"] },
    { from_port = 80, to_port = 80, protocol = "tcp", cidr_blocks = ["0.0.0.0/0"] }
  ]

  # Regras de saída
  egress_rules = [
    { from_port = 0, to_port = 0, protocol = "-1", cidr_blocks = ["0.0.0.0/0"] }
  ]
}



Resultado
- O Security Group será criado com regras de entrada e saída definidas via input.
- Você pode adicionar/remover portas e protocolos sem alterar o módulo, apenas ajustando o main.tf.
- Isso deixa o módulo genérico e reutilizável para diferentes cenários.


###############
###############

vamos deixar o módulo ainda mais flexível criando Security Groups adicionais via input — por exemplo, um SG para as instâncias e outro para o Load Balancer.


1º Ajustar variáveis (modules/ec2-lb/variables.tf)
Defina uma lista de SGs, cada um com suas regras

variable "security_groups" {
  description = "Lista de Security Groups adicionais"
  type = list(object({
    name          = string
    description   = string
    ingress_rules = list(object({
      from_port   = number
      to_port     = number
      protocol    = string
      cidr_blocks = list(string)
    }))
    egress_rules = list(object({
      from_port   = number
      to_port     = number
      protocol    = string
      cidr_blocks = list(string)
    }))
  }))
  default = []
}


2º Criar SGs dinamicamente (modules/ec2-lb/main.tf)

resource "aws_security_group" "sgs" {
  for_each    = { for sg in var.security_groups : sg.name => sg }
  name        = each.value.name
  description = each.value.description
  vpc_id      = var.vpc_id

  dynamic "ingress" {
    for_each = each.value.ingress_rules
    content {
      from_port   = ingress.value.from_port
      to_port     = ingress.value.to_port
      protocol    = ingress.value.protocol
      cidr_blocks = ingress.value.cidr_blocks
    }
  }

  dynamic "egress" {
    for_each = each.value.egress_rules
    content {
      from_port   = egress.value.from_port
      to_port     = egress.value.to_port
      protocol    = egress.value.protocol
      cidr_blocks = egress.value.cidr_blocks
    }
  }
}


3º Associar SGs às instâncias e ao Load Balancer

resource "aws_instance" "ec2" {
  count         = var.instance_count
  ami           = var.ami_id
  instance_type = var.instance_type
  key_name      = var.key_name

  # associa SG específico das instâncias
  vpc_security_group_ids = [aws_security_group.sgs["ec2-sg"].id]

  root_block_device {
    volume_size = var.volume_size
    volume_type = var.volume_type
  }

  tags = {
    Name = "Terraform-EC2-${count.index}"
  }
}

resource "aws_lb" "app_lb" {
  name               = "app-lb"
  internal           = false
  load_balancer_type = "application"
  subnets            = var.subnets

  # associa SG específico do LB
  security_groups    = [aws_security_group.sgs["lb-sg"].id]

  tags = {
    Name = "App-LB"
  }
}


4º Consumidor do módulo (main.tf)
Agora você define os SGs diretamente:

module "ec2_lb" {
  source         = "github.com/<SEU_USUARIO>/<SEU_REPOSITORIO>//modules/ec2-lb?ref=v1.0.0"
  vpc_id         = "vpc-123456"
  subnets        = ["subnet-abc123", "subnet-def456"]
  ami_id         = "ami-0c55b159cbfafe1f0"
  instance_type  = "t2.micro"
  instance_count = 3
  key_name       = "minha-key"
  volume_size    = 50
  volume_type    = "gp3"

  security_groups = [
    {
      name        = "ec2-sg"
      description = "SG para instâncias EC2"
      ingress_rules = [
        { from_port = 22, to_port = 22, protocol = "tcp", cidr_blocks = ["0.0.0.0/0"] },
        { from_port = 80, to_port = 80, protocol = "tcp", cidr_blocks = ["0.0.0.0/0"] }
      ]
      egress_rules = [
        { from_port = 0, to_port = 0, protocol = "-1", cidr_blocks = ["0.0.0.0/0"] }
      ]
    },
    {
      name        = "lb-sg"
      description = "SG para Load Balancer"
      ingress_rules = [
        { from_port = 80, to_port = 80, protocol = "tcp", cidr_blocks = ["0.0.0.0/0"] }
      ]
      egress_rules = [
        { from_port = 0, to_port = 0, protocol = "-1", cidr_blocks = ["0.0.0.0/0"] }
      ]
    }
  ]
}


Resultado
- O módulo cria dois Security Groups:
- ec2-sg para as instâncias EC2 (SSH + HTTP).
- lb-sg para o Load Balancer (HTTP).
- Cada recurso (EC2 e LB) é associado ao SG correto.
- Você pode adicionar quantos SGs quiser, apenas ajustando o input no consumidor.


###############
###############

amos parametrizar também os Target Groups e Listeners do Load Balancer via input, para que o tráfego seja automaticamente distribuído entre as instâncias criadas.


1º Variáveis para Target Groups e Listeners (modules/ec2-lb/variables.tf)

variable "target_groups" {
  description = "Lista de Target Groups para o Load Balancer"
  type = list(object({
    name     = string
    port     = number
    protocol = string
  }))
  default = []
}

variable "listeners" {
  description = "Lista de Listeners para o Load Balancer"
  type = list(object({
    port            = number
    protocol        = string
    target_group    = string
  }))
  default = []
}



2º Criar Target Groups e Listeners (modules/ec2-lb/main.tf)

resource "aws_lb_target_group" "tg" {
  for_each = { for tg in var.target_groups : tg.name => tg }
  name     = each.value.name
  port     = each.value.port
  protocol = each.value.protocol
  vpc_id   = var.vpc_id
}

resource "aws_lb_listener" "listener" {
  for_each          = { for l in var.listeners : l.port => l }
  load_balancer_arn = aws_lb.app_lb.arn
  port              = each.value.port
  protocol          = each.value.protocol

  default_action {
    type             = "forward"
    target_group_arn = aws_lb_target_group.tg[each.value.target_group].arn
  }
}

# Registrar instâncias EC2 nos Target Groups
resource "aws_lb_target_group_attachment" "tg_attachment" {
  for_each = { for idx, inst in aws_instance.ec2 : idx => inst }
  target_group_arn = aws_lb_target_group.tg["ec2-tg"].arn
  target_id        = each.value.id
  port             = 80
}



3º Consumidor do módulo (main.tf)
Agora você define os Target Groups e Listeners diretamente:

module "ec2_lb" {
  source         = "github.com/<SEU_USUARIO>/<SEU_REPOSITORIO>//modules/ec2-lb?ref=v1.0.0"
  vpc_id         = "vpc-123456"
  subnets        = ["subnet-abc123", "subnet-def456"]
  ami_id         = "ami-0c55b159cbfafe1f0"
  instance_type  = "t2.micro"
  instance_count = 3
  key_name       = "minha-key"
  volume_size    = 50
  volume_type    = "gp3"

  # Security Groups
  security_groups = [
    {
      name        = "ec2-sg"
      description = "SG para instâncias EC2"
      ingress_rules = [
        { from_port = 22, to_port = 22, protocol = "tcp", cidr_blocks = ["0.0.0.0/0"] },
        { from_port = 80, to_port = 80, protocol = "tcp", cidr_blocks = ["0.0.0.0/0"] }
      ]
      egress_rules = [
        { from_port = 0, to_port = 0, protocol = "-1", cidr_blocks = ["0.0.0.0/0"] }
      ]
    },
    {
      name        = "lb-sg"
      description = "SG para Load Balancer"
      ingress_rules = [
        { from_port = 80, to_port = 80, protocol = "tcp", cidr_blocks = ["0.0.0.0/0"] }
      ]
      egress_rules = [
        { from_port = 0, to_port = 0, protocol = "-1", cidr_blocks = ["0.0.0.0/0"] }
      ]
    }
  ]

  # Target Groups
  target_groups = [
    { name = "ec2-tg", port = 80, protocol = "HTTP" }
  ]

  # Listeners
  listeners = [
    { port = 80, protocol = "HTTP", target_group = "ec2-tg" }
  ]
}



Resultado
- O módulo cria:
- Target Group (ec2-tg) para as instâncias EC2.
- Listener na porta 80 do Load Balancer, que encaminha tráfego para o Target Group.
- As instâncias EC2 são automaticamente registradas no Target Group.
- O tráfego recebido pelo Load Balancer será distribuído entre as instâncias EC2.


###############
###############

Vamos parametrizar também os health checks do Target Group via input, para que você tenha controle total sobre como o Load Balancer verifica se as instâncias estão saudáveism.


1º Variáveis para health checks (modules/ec2-lb/variables.tf)
Adicione os parâmetros de health check:

variable "health_check" {
  description = "Configuração de health check para Target Groups"
  type = object({
    path                = string
    port                = number
    protocol            = string
    interval            = number
    timeout             = number
    healthy_threshold   = number
    unhealthy_threshold = number
  })
  default = {
    path                = "/"
    port                = 80
    protocol            = "HTTP"
    interval            = 30
    timeout             = 5
    healthy_threshold   = 3
    unhealthy_threshold = 2
  }
}

2º Usar health check no Target Group (modules/ec2-lb/main.tf)

resource "aws_lb_target_group" "tg" {
  for_each = { for tg in var.target_groups : tg.name => tg }
  name     = each.value.name
  port     = each.value.port
  protocol = each.value.protocol
  vpc_id   = var.vpc_id

  health_check {
    path                = var.health_check.path
    port                = var.health_check.port
    protocol            = var.health_check.protocol
    interval            = var.health_check.interval
    timeout             = var.health_check.timeout
    healthy_threshold   = var.health_check.healthy_threshold
    unhealthy_threshold = var.health_check.unhealthy_threshold
  }
}



3º Consumidor do módulo (main.tf)
gora você pode passar os parâmetros de health check diretamente:

module "ec2_lb" {
  source         = "github.com/<SEU_USUARIO>/<SEU_REPOSITORIO>//modules/ec2-lb?ref=v1.0.0"
  vpc_id         = "vpc-123456"
  subnets        = ["subnet-abc123", "subnet-def456"]
  ami_id         = "ami-0c55b159cbfafe1f0"
  instance_type  = "t2.micro"
  instance_count = 3
  key_name       = "minha-key"
  volume_size    = 50
  volume_type    = "gp3"

  # Target Groups
  target_groups = [
    { name = "ec2-tg", port = 80, protocol = "HTTP" }
  ]

  # Listeners
  listeners = [
    { port = 80, protocol = "HTTP", target_group = "ec2-tg" }
  ]

  # Health check customizado
  health_check = {
    path                = "/health"
    port                = 80
    protocol            = "HTTP"
    interval            = 20
    timeout             = 5
    healthy_threshold   = 2
    unhealthy_threshold = 2
  }
}



Resultado
- O Target Group terá health check configurado conforme os inputs.
- O Load Balancer verificará a saúde das instâncias acessando http://<instancia>:80/health.
- Intervalo de checagem: 20 segundos.
- Timeout: 5 segundos.
- Instância será considerada saudável após 2 respostas positivas consecutivas e não saudável após 2 falhas consecutivas.


###############
###############

Vamos deixar o módulo capaz de lidar com múltiplos Target Groups, cada um com seu próprio health check. Assim você pode ter, por exemplo, um TG verificando  e outro verificando .

1º ariáveis (modules/ec2-lb/variables.tf)
Agora cada Target Group terá também os parâmetros de health check:

variable "target_groups" {
  description = "Lista de Target Groups com health checks"
  type = list(object({
    name                = string
    port                = number
    protocol            = string
    health_check = object({
      path                = string
      port                = number
      protocol            = string
      interval            = number
      timeout             = number
      healthy_threshold   = number
      unhealthy_threshold = number
    })
  }))
  default = []
}

variable "listeners" {
  description = "Lista de Listeners para o Load Balancer"
  type = list(object({
    port         = number
    protocol     = string
    target_group = string
  }))
  default = []
}



2º arget Groups com health checks (modules/ec2-lb/main.tf)

resource "aws_lb_target_group" "tg" {
  for_each = { for tg in var.target_groups : tg.name => tg }
  name     = each.value.name
  port     = each.value.port
  protocol = each.value.protocol
  vpc_id   = var.vpc_id

  health_check {
    path                = each.value.health_check.path
    port                = each.value.health_check.port
    protocol            = each.value.health_check.protocol
    interval            = each.value.health_check.interval
    timeout             = each.value.health_check.timeout
    healthy_threshold   = each.value.health_check.healthy_threshold
    unhealthy_threshold = each.value.health_check.unhealthy_threshold
  }
}

resource "aws_lb_listener" "listener" {
  for_each          = { for l in var.listeners : l.port => l }
  load_balancer_arn = aws_lb.app_lb.arn
  port              = each.value.port
  protocol          = each.value.protocol

  default_action {
    type             = "forward"
    target_group_arn = aws_lb_target_group.tg[each.value.target_group].arn
  }
}

# Registrar instâncias EC2 em um TG específico (exemplo: ec2-tg)
resource "aws_lb_target_group_attachment" "tg_attachment" {
  for_each = { for idx, inst in aws_instance.ec2 : idx => inst }
  target_group_arn = aws_lb_target_group.tg["ec2-tg"].arn
  target_id        = each.value.id
  port             = 80
}



3º Consumidor do módulo (main.tf)
Agora você pode passar múltiplos Target Groups com health checks diferentes:

module "ec2_lb" {
  source         = "github.com/<SEU_USUARIO>/<SEU_REPOSITORIO>//modules/ec2-lb?ref=v1.0.0"
  vpc_id         = "vpc-123456"
  subnets        = ["subnet-abc123", "subnet-def456"]
  ami_id         = "ami-0c55b159cbfafe1f0"
  instance_type  = "t2.micro"
  instance_count = 3
  key_name       = "minha-key"
  volume_size    = 50
  volume_type    = "gp3"

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
    },
    {
      name     = "status-tg"
      port     = 8080
      protocol = "HTTP"
      health_check = {
        path                = "/status"
        port                = 8080
        protocol            = "HTTP"
        interval            = 30
        timeout             = 5
        healthy_threshold   = 3
        unhealthy_threshold = 2
      }
    }
  ]

  listeners = [
    { port = 80, protocol = "HTTP", target_group = "ec2-tg" },
    { port = 8080, protocol = "HTTP", target_group = "status-tg" }
  ]
}



Resultado
- O módulo cria dois Target Groups:
- ec2-tg verificando /health na porta 80.
- status-tg verificando /status na porta 8080.
- O Load Balancer terá dois listeners:
- Porta 80 encaminhando para ec2-tg.
- Porta 8080 encaminhando para status-tg.
- As instâncias EC2 serão registradas no TG principal (ec2-tg), e você pode expandir para registrar em múltiplos TGs se necessário.



###############
###############

 vamos deixar o módulo capaz de registrar automaticamente as mesmas instâncias em múltiplos Target Groups, cada um com seu próprio health check. Assim, uma instância pode servir tanto  quanto .


 1º ariáveis (modules/ec2-lb/variables.tf)
Já temos a lista de Target Groups com health check. Vamos mantê-la:

variable "target_groups" {
  description = "Lista de Target Groups com health checks"
  type = list(object({
    name     = string
    port     = number
    protocol = string
    health_check = object({
      path                = string
      port                = number
      protocol            = string
      interval            = number
      timeout             = number
      healthy_threshold   = number
      unhealthy_threshold = number
    })
  }))
  default = []
}



2º Criar Target Groups e registrar instâncias em todos (modules/ec2-lb/main.tf)

resource "aws_lb_target_group" "tg" {
  for_each = { for tg in var.target_groups : tg.name => tg }
  name     = each.value.name
  port     = each.value.port
  protocol = each.value.protocol
  vpc_id   = var.vpc_id

  health_check {
    path                = each.value.health_check.path
    port                = each.value.health_check.port
    protocol            = each.value.health_check.protocol
    interval            = each.value.health_check.interval
    timeout             = each.value.health_check.timeout
    healthy_threshold   = each.value.health_check.healthy_threshold
    unhealthy_threshold = each.value.health_check.unhealthy_threshold
  }
}

resource "aws_lb_listener" "listener" {
  for_each          = { for l in var.listeners : l.port => l }
  load_balancer_arn = aws_lb.app_lb.arn
  port              = each.value.port
  protocol          = each.value.protocol

  default_action {
    type             = "forward"
    target_group_arn = aws_lb_target_group.tg[each.value.target_group].arn
  }
}

# Registrar instâncias em TODOS os Target Groups
resource "aws_lb_target_group_attachment" "tg_attachment" {
  for_each = {
    for tg_name, tg in aws_lb_target_group.tg :
    tg_name => tg
  }

  target_group_arn = each.value.arn
  target_id        = aws_instance.ec2[0].id
  port             = each.value.port
}

# Se quiser registrar todas as instâncias em todos os TGs:
resource "aws_lb_target_group_attachment" "tg_attachment_all" {
  for_each = {
    for tg_name, tg in aws_lb_target_group.tg :
    tg_name => tg
  }

  target_group_arn = each.value.arn
  target_id        = aws_instance.ec2[each.key].id
  port             = each.value.port
}

OBS: Aqui criamos attachments para cada Target Group e associamos todas as instâncias EC2 a eles



3º Consumidor do módulo (main.tf)

module "ec2_lb" {
  source         = "github.com/<SEU_USUARIO>/<SEU_REPOSITORIO>//modules/ec2-lb?ref=v1.0.0"
  vpc_id         = "vpc-123456"
  subnets        = ["subnet-abc123", "subnet-def456"]
  ami_id         = "ami-0c55b159cbfafe1f0"
  instance_type  = "t2.micro"
  instance_count = 3
  key_name       = "minha-key"
  volume_size    = 50
  volume_type    = "gp3"

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
    },
    {
      name     = "status-tg"
      port     = 8080
      protocol = "HTTP"
      health_check = {
        path                = "/status"
        port                = 8080
        protocol            = "HTTP"
        interval            = 30
        timeout             = 5
        healthy_threshold   = 3
        unhealthy_threshold = 2
      }
    }
  ]

  listeners = [
    { port = 80, protocol = "HTTP", target_group = "ec2-tg" },
    { port = 8080, protocol = "HTTP", target_group = "status-tg" }
  ]
}


Resultado
- O módulo cria dois Target Groups (ec2-tg e status-tg), cada um com seu próprio health check.
- O Load Balancer terá dois listeners (80 e 8080).
- Todas as instâncias EC2 serão registradas em ambos os Target Groups.
- O tráfego recebido na porta 80 será distribuído entre as instâncias servindo /health.
- O tráfego recebido na porta 8080 será distribuído entre as mesmas instâncias servindo /status.


###############
###############

vamos deixar o módulo capaz de configurar sticky sessions (afinidade de sessão) nos Target Groups, para que um cliente seja sempre direcionado à mesma instância durante a sua sessão.


1º - Variáveis (modules/ec2-lb/variables.tf)
Adicione o campo stickiness dentro da definição de Target Groups

variable "target_groups" {
  description = "Lista de Target Groups com health checks e stickiness"
  type = list(object({
    name     = string
    port     = number
    protocol = string
    health_check = object({
      path                = string
      port                = number
      protocol            = string
      interval            = number
      timeout             = number
      healthy_threshold   = number
      unhealthy_threshold = number
    })
    stickiness = object({
      enabled  = bool
      type     = string
      duration = number
    })
  }))
  default = []
}



2º Target Groups com stickiness (modules/ec2-lb/main.tf)

resource "aws_lb_target_group" "tg" {
  for_each = { for tg in var.target_groups : tg.name => tg }
  name     = each.value.name
  port     = each.value.port
  protocol = each.value.protocol
  vpc_id   = var.vpc_id

  health_check {
    path                = each.value.health_check.path
    port                = each.value.health_check.port
    protocol            = each.value.health_check.protocol
    interval            = each.value.health_check.interval
    timeout             = each.value.health_check.timeout
    healthy_threshold   = each.value.health_check.healthy_threshold
    unhealthy_threshold = each.value.health_check.unhealthy_threshold
  }

  stickiness {
    enabled  = each.value.stickiness.enabled
    type     = each.value.stickiness.type
    cookie_duration = each.value.stickiness.duration
  }
}


3º Consumidor do módulo (main.tf)
Agora você pode passar os parâmetros de stickiness junto com os health checks:

module "ec2_lb" {
  source         = "github.com/<SEU_USUARIO>/<SEU_REPOSITORIO>//modules/ec2-lb?ref=v1.0.0"
  vpc_id         = "vpc-123456"
  subnets        = ["subnet-abc123", "subnet-def456"]
  ami_id         = "ami-0c55b159cbfafe1f0"
  instance_type  = "t2.micro"
  instance_count = 3
  key_name       = "minha-key"
  volume_size    = 50
  volume_type    = "gp3"

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
    },
    {
      name     = "status-tg"
      port     = 8080
      protocol = "HTTP"
      health_check = {
        path                = "/status"
        port                = 8080
        protocol            = "HTTP"
        interval            = 30
        timeout             = 5
        healthy_threshold   = 3
        unhealthy_threshold = 2
      }
      stickiness = {
        enabled  = true
        type     = "lb_cookie"
        duration = 600
      }
    }
  ]

  listeners = [
    { port = 80, protocol = "HTTP", target_group = "ec2-tg" },
    { port = 8080, protocol = "HTTP", target_group = "status-tg" }
  ]
}



Resultado
- O Target Group ec2-tg terá stickiness habilitado com cookie de 300 segundos.
- O Target Group status-tg terá stickiness habilitado com cookie de 600 segundos.
- O Load Balancer criará cookies de afinidade e manterá o cliente sempre na mesma instância durante o tempo configurado.
- Isso garante consistência em sessões que dependem de estado (ex.: login, carrinho de compras)



###############
###############


Vamos evoluir o módulo para que, em vez de criar instâncias EC2 diretamente, ele crie um Auto Scaling Group (ASG). 
Assim você pode escalar horizontalmente suas instâncias de forma automática, mantendo a integração com o Load Balancer e os Target Groups.


1º Variáveis para ASG (modules/ec2-lb/variables.tf)

variable "asg_min_size" {
  description = "Número mínimo de instâncias no ASG"
  type        = number
  default     = 1
}

variable "asg_max_size" {
  description = "Número máximo de instâncias no ASG"
  type        = number
  default     = 3
}

variable "asg_desired_capacity" {
  description = "Capacidade desejada de instâncias no ASG"
  type        = number
  default     = 2
}


2º Launch Template para instâncias (modules/ec2-lb/main.tf)
O ASG precisa de um Launch Template para definir como as instâncias serão criadas:

resource "aws_launch_template" "ec2_lt" {
  name_prefix   = "ec2-lt-"
  image_id      = var.ami_id
  instance_type = var.instance_type
  key_name      = var.key_name

  block_device_mappings {
    device_name = "/dev/sda1"
    ebs {
      volume_size = var.volume_size
      volume_type = var.volume_type
    }
  }

  tag_specifications {
    resource_type = "instance"
    tags = {
      Name = "Terraform-ASG-Instance"
    }
  }
}


3º Auto Scaling Group (modules/ec2-lb/main.tf)

resource "aws_autoscaling_group" "ec2_asg" {
  name                      = "ec2-asg"
  min_size                  = var.asg_min_size
  max_size                  = var.asg_max_size
  desired_capacity           = var.asg_desired_capacity
  vpc_zone_identifier        = var.subnets
  launch_template {
    id      = aws_launch_template.ec2_lt.id
    version = "$Latest"
  }

  target_group_arns = [for tg in aws_lb_target_group.tg : tg.arn]

  tag {
    key                 = "Name"
    value               = "Terraform-ASG"
    propagate_at_launch = true
  }
}


4º Consumidor do módulo (main.tf)
Agora você pode parametrizar o ASG diretamente:

module "ec2_lb" {
  source         = "github.com/<SEU_USUARIO>/<SEU_REPOSITORIO>//modules/ec2-lb?ref=v1.0.0"
  vpc_id         = "vpc-123456"
  subnets        = ["subnet-abc123", "subnet-def456"]
  ami_id         = "ami-0c55b159cbfafe1f0"
  instance_type  = "t2.micro"
  key_name       = "minha-key"
  volume_size    = 50
  volume_type    = "gp3"

  # Configuração do Auto Scaling Group
  asg_min_size        = 2
  asg_max_size        = 5
  asg_desired_capacity = 3

  # Target Groups e Listeners
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
    { port = 80, protocol = "HTTP", target_group = "ec2-tg" }
  ]
}


Resultado
- O módulo cria um Launch Template com a configuração da instância.
- O Auto Scaling Group gerencia automaticamente o número de instâncias entre asg_min_size e asg_max_size.
- As instâncias são registradas automaticamente nos Target Groups do Load Balancer.
- O tráfego é distribuído entre todas as instâncias ativas.
- Você pode escalar horizontalmente de forma automática ou manual ajustando os parâmetros


###############
###############

vamos deixar o módulo capaz de criar políticas de scaling automático para o Auto Scaling Group (ASG), baseadas em métricas como CPU > 70% ou Memória > 85%.

1º Variáveis para políticas de scaling (modules/ec2-lb/variables.tf)

variable "scaling_policies" {
  description = "Lista de políticas de scaling automático para o ASG"
  type = list(object({
    name           = string
    adjustment_type = string
    scaling_adjustment = number
    metric_type    = string   # "CPU" ou "Memory"
    threshold      = number
    comparison     = string   # "GreaterThanOrEqualToThreshold", etc.
    period         = number
    evaluation     = number
    statistic      = string   # "Average", "Maximum", etc.
  }))
  default = []
}


2º CloudWatch Alarms + Scaling Policies (modules/ec2-lb/main.tf)

resource "aws_autoscaling_policy" "asg_policy" {
  for_each = { for p in var.scaling_policies : p.name => p }
  name                   = each.value.name
  autoscaling_group_name = aws_autoscaling_group.ec2_asg.name
  adjustment_type        = each.value.adjustment_type
  scaling_adjustment     = each.value.scaling_adjustment
}

resource "aws_cloudwatch_metric_alarm" "asg_alarm" {
  for_each = { for p in var.scaling_policies : p.name => p }
  alarm_name          = "${each.value.name}-alarm"
  comparison_operator = each.value.comparison
  evaluation_periods  = each.value.evaluation
  metric_name         = each.value.metric_type == "CPU" ? "CPUUtilization" : "MemoryUtilization"
  namespace           = "AWS/EC2"
  period              = each.value.period
  statistic           = each.value.statistic
  threshold           = each.value.threshold
  alarm_description   = "Alarm para ${each.value.metric_type} no ASG"
  dimensions = {
    AutoScalingGroupName = aws_autoscaling_group.ec2_asg.name
  }
  alarm_actions = [aws_autoscaling_policy.asg_policy[each.key].arn]
}


3º Consumidor do módulo (main.tf)
Agora você pode passar as políticas diretamente:

module "ec2_lb" {
  source         = "github.com/<SEU_USUARIO>/<SEU_REPOSITORIO>//modules/ec2-lb?ref=v1.0.0"
  vpc_id         = "vpc-123456"
  subnets        = ["subnet-abc123", "subnet-def456"]
  ami_id         = "ami-0c55b159cbfafe1f0"
  instance_type  = "t2.micro"
  key_name       = "minha-key"
  volume_size    = 50
  volume_type    = "gp3"

  # Configuração do Auto Scaling Group
  asg_min_size         = 2
  asg_max_size         = 5
  asg_desired_capacity = 3

  # Políticas de scaling automático
  scaling_policies = [
    {
      name               = "cpu-scale-up"
      adjustment_type    = "ChangeInCapacity"
      scaling_adjustment = 1
      metric_type        = "CPU"
      threshold          = 70
      comparison         = "GreaterThanOrEqualToThreshold"
      period             = 60
      evaluation         = 2
      statistic          = "Average"
    },
    {
      name               = "memory-scale-up"
      adjustment_type    = "ChangeInCapacity"
      scaling_adjustment = 1
      metric_type        = "Memory"
      threshold          = 85
      comparison         = "GreaterThanOrEqualToThreshold"
      period             = 60
      evaluation         = 2
      statistic          = "Average"
    }
  ]

  # Target Groups e Listeners
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
    { port = 80, protocol = "HTTP", target_group = "ec2-tg" }
  ]
}



Resultado
- O ASG começa com 3 instâncias e pode escalar até 5.
- Se a CPU média ≥ 70% por 2 períodos consecutivos, o ASG adiciona +1 instância.
- Se a Memória média ≥ 85% por 2 períodos consecutivos, o ASG também adiciona +1 instância.
- As instâncias são automaticamente registradas nos Target Groups e recebem tráfego do Load Balancer


###############
###############

vamos completar o ciclo do Auto Scaling Group (ASG) parametrizando também as políticas de scale-down, para reduzir instâncias quando o uso de CPU ou memória estiver baixo, otimizando custos automaticamente.


1º Variáveis para scale-down (modules/ec2-lb/variables.tf)
Você pode usar a mesma estrutura de scaling_policies, mas agora incluir políticas de redução

variable "scaling_policies" {
  description = "Lista de políticas de scaling automático (up e down) para o ASG"
  type = list(object({
    name               = string
    adjustment_type    = string
    scaling_adjustment = number
    metric_type        = string   # "CPU" ou "Memory"
    threshold          = number
    comparison         = string   # "GreaterThanOrEqualToThreshold" ou "LessThanOrEqualToThreshold"
    period             = number
    evaluation         = number
    statistic          = string   # "Average", "Maximum", etc.
  }))
  default = []
}



2º CloudWatch Alarms + Policies (modules/ec2-lb/main.tf)
O mesmo recurso que usamos para scale-up serve para scale-down, basta mudar o comparison e scaling_adjustment:

resource "aws_autoscaling_policy" "asg_policy" {
  for_each = { for p in var.scaling_policies : p.name => p }
  name                   = each.value.name
  autoscaling_group_name = aws_autoscaling_group.ec2_asg.name
  adjustment_type        = each.value.adjustment_type
  scaling_adjustment     = each.value.scaling_adjustment
}

resource "aws_cloudwatch_metric_alarm" "asg_alarm" {
  for_each = { for p in var.scaling_policies : p.name => p }
  alarm_name          = "${each.value.name}-alarm"
  comparison_operator = each.value.comparison
  evaluation_periods  = each.value.evaluation
  metric_name         = each.value.metric_type == "CPU" ? "CPUUtilization" : "MemoryUtilization"
  namespace           = "AWS/EC2"
  period              = each.value.period
  statistic           = each.value.statistic
  threshold           = each.value.threshold
  alarm_description   = "Alarm para ${each.value.metric_type} no ASG"
  dimensions = {
    AutoScalingGroupName = aws_autoscaling_group.ec2_asg.name
  }
  alarm_actions = [aws_autoscaling_policy.asg_policy[each.key].arn]
}


3º Consumidor do módulo (main.tf)
Agora você pode passar scale-up e scale-down juntos:

module "ec2_lb" {
  source         = "github.com/<SEU_USUARIO>/<SEU_REPOSITORIO>//modules/ec2-lb?ref=v1.0.0"
  vpc_id         = "vpc-123456"
  subnets        = ["subnet-abc123", "subnet-def456"]
  ami_id         = "ami-0c55b159cbfafe1f0"
  instance_type  = "t2.micro"
  key_name       = "minha-key"
  volume_size    = 50
  volume_type    = "gp3"

  # Configuração do Auto Scaling Group
  asg_min_size         = 2
  asg_max_size         = 5
  asg_desired_capacity = 3

  # Políticas de scaling automático
  scaling_policies = [
    # Scale-up CPU
    {
      name               = "cpu-scale-up"
      adjustment_type    = "ChangeInCapacity"
      scaling_adjustment = 1
      metric_type        = "CPU"
      threshold          = 70
      comparison         = "GreaterThanOrEqualToThreshold"
      period             = 60
      evaluation         = 2
      statistic          = "Average"
    },
    # Scale-down CPU
    {
      name               = "cpu-scale-down"
      adjustment_type    = "ChangeInCapacity"
      scaling_adjustment = -1
      metric_type        = "CPU"
      threshold          = 30
      comparison         = "LessThanOrEqualToThreshold"
      period             = 60
      evaluation         = 2
      statistic          = "Average"
    },
    # Scale-up Memory
    {
      name               = "memory-scale-up"
      adjustment_type    = "ChangeInCapacity"
      scaling_adjustment = 1
      metric_type        = "Memory"
      threshold          = 85
      comparison         = "GreaterThanOrEqualToThreshold"
      period             = 60
      evaluation         = 2
      statistic          = "Average"
    },
    # Scale-down Memory
    {
      name               = "memory-scale-down"
      adjustment_type    = "ChangeInCapacity"
      scaling_adjustment = -1
      metric_type        = "Memory"
      threshold          = 40
      comparison         = "LessThanOrEqualToThreshold"
      period             = 60
      evaluation         = 2
      statistic          = "Average"
    }
  ]

  # Target Groups e Listeners
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
    { port = 80, protocol = "HTTP", target_group = "ec2-tg" }
  ]
}


Resultado
- O ASG escalará para cima quando CPU ≥ 70% ou Memória ≥ 85%.
- O ASG escalará para baixo quando CPU ≤ 30% ou Memória ≤ 40%.
- Isso garante elasticidade: mais instâncias quando há carga alta e menos instâncias quando há carga baixa, otimizando custos.
- As instâncias continuam sendo registradas automaticamente nos Target Groups e recebendo tráfego do Load Balancer.


###############
###############

Vamos expandir o padrão para suportar métricas customizadas além de CPU/Memória, como número de requisições por segundo (RPS) ou tamanho de fila de mensagens. 
Isso é muito útil quando a carga da aplicação não está diretamente ligada ao uso de CPU, mas sim ao throughput ou backlog.


1º  Variáveis (modules/ec2-lb/variables.tf)
Aqui adicionamos a possibilidade de definir métricas customizadas, parametrizando namespace e metric_name

variable "scaling_policies" {
  description = "Lista de políticas de scaling automático (up e down) para o ASG"
  type = list(object({
    name               = string
    adjustment_type    = string
    scaling_adjustment = number
    metric_type        = string   # "CPU", "Memory" ou "Custom"
    namespace          = optional(string) # Ex: "AWS/ApplicationELB" ou "MyApp/Metrics"
    metric_name        = optional(string) # Ex: "RequestCount" ou "QueueLength"
    threshold          = number
    comparison         = string   # "GreaterThanOrEqualToThreshold" ou "LessThanOrEqualToThreshold"
    period             = number
    evaluation         = number
    statistic          = string   # "Average", "Maximum", etc.
  }))
  default = []
}


2º CloudWatch Alarms + Policies (modules/ec2-lb/main.tf)
Agora o metric_name e namespace são dinâmicos: se for CPU/Memory usamos defaults, se for Custom usamos os valores passados.

resource "aws_autoscaling_policy" "asg_policy" {
  for_each = { for p in var.scaling_policies : p.name => p }

  name                   = each.value.name
  autoscaling_group_name = aws_autoscaling_group.ec2_asg.name
  adjustment_type        = each.value.adjustment_type
  scaling_adjustment     = each.value.scaling_adjustment
}

resource "aws_cloudwatch_metric_alarm" "asg_alarm" {
  for_each = { for p in var.scaling_policies : p.name => p }

  alarm_name          = "${each.value.name}-alarm"
  comparison_operator = each.value.comparison
  evaluation_periods  = each.value.evaluation

  metric_name = (
    each.value.metric_type == "CPU"    ? "CPUUtilization" :
    each.value.metric_type == "Memory" ? "MemoryUtilization" :
    each.value.metric_name
  )

  namespace = (
    each.value.metric_type == "CPU" || each.value.metric_type == "Memory"
    ? "AWS/EC2"
    : each.value.namespace
  )

  period            = each.value.period
  statistic         = each.value.statistic
  threshold         = each.value.threshold
  alarm_description = "Alarm para ${each.value.metric_type} no ASG"

  dimensions = {
    AutoScalingGroupName = aws_autoscaling_group.ec2_asg.name
  }

  alarm_actions = [aws_autoscaling_policy.asg_policy[each.key].arn]
}



3º  Consumidor do módulo (main.tf)
Agora você pode passar métricas customizadas junto com CPU/Memory:

scaling_policies = [
  # Scale-up CPU
  {
    name               = "cpu-scale-up"
    adjustment_type    = "ChangeInCapacity"
    scaling_adjustment = 1
    metric_type        = "CPU"
    threshold          = 70
    comparison         = "GreaterThanOrEqualToThreshold"
    period             = 60
    evaluation         = 2
    statistic          = "Average"
  },
  # Scale-down CPU
  {
    name               = "cpu-scale-down"
    adjustment_type    = "ChangeInCapacity"
    scaling_adjustment = -1
    metric_type        = "CPU"
    threshold          = 30
    comparison         = "LessThanOrEqualToThreshold"
    period             = 60
    evaluation         = 2
    statistic          = "Average"
  },
  # Scale-up baseado em requisições por segundo (RPS)
  {
    name               = "rps-scale-up"
    adjustment_type    = "ChangeInCapacity"
    scaling_adjustment = 2
    metric_type        = "Custom"
    namespace          = "AWS/ApplicationELB"
    metric_name        = "RequestCount"
    threshold          = 1000
    comparison         = "GreaterThanOrEqualToThreshold"
    period             = 60
    evaluation         = 2
    statistic          = "Sum"
  },
  # Scale-down baseado em fila de mensagens
  {
    name               = "queue-scale-down"
    adjustment_type    = "ChangeInCapacity"
    scaling_adjustment = -1
    metric_type        = "Custom"
    namespace          = "MyApp/Metrics"
    metric_name        = "QueueLength"
    threshold          = 50
    comparison         = "LessThanOrEqualToThreshold"
    period             = 60
    evaluation         = 2
    statistic          = "Average"
  }
]



Resultado
- Escala para cima com base em CPU, memória ou número de requisições.
- Escala para baixo quando CPU/memória estão baixas ou quando a fila de mensagens está pequena.
- O módulo agora suporta qualquer métrica customizada registrada no CloudWatch, mantendo o mesmo padrão de parametrização.



###############
###############


 vamos configurar o Load Balancer para fazer o redirect automático de HTTP → HTTPS e usar o certificado já cadastrado no AWS Certificate Manager (ACM), no caso o teste.pactual.net

1º Variáveis para certificados (modules/ec2-lb/variables.tf).

variable "certificate_arn" {
  description = "ARN do certificado SSL/TLS no ACM"
  type        = string
}



2º Listeners com redirect HTTP → HTTPS (modules/ec2-lb/main.tf)

# Listener HTTP (porta 80) que redireciona para HTTPS
resource "aws_lb_listener" "http_redirect" {
  load_balancer_arn = aws_lb.app_lb.arn
  port              = 80
  protocol          = "HTTP"

  default_action {
    type = "redirect"
    redirect {
      port        = "443"
      protocol    = "HTTPS"
      status_code = "HTTP_301"
    }
  }
}

# Listener HTTPS (porta 443) que usa o certificado ACM
resource "aws_lb_listener" "https_listener" {
  load_balancer_arn = aws_lb.app_lb.arn
  port              = 443
  protocol          = "HTTPS"
  ssl_policy        = "ELBSecurityPolicy-2016-08"
  certificate_arn   = var.certificate_arn

  default_action {
    type             = "forward"
    target_group_arn = aws_lb_target_group.tg["ec2-tg"].arn
  }
}


3º Consumidor do módulo (main.tf)
Aqui você passa o ARN do certificado já cadastrado no ACM:

module "ec2_lb" {
  source         = "github.com/<SEU_USUARIO>/<SEU_REPOSITORIO>//modules/ec2-lb?ref=v1.0.0"
  vpc_id         = "vpc-123456"
  subnets        = ["subnet-abc123", "subnet-def456"]
  ami_id         = "ami-0c55b159cbfafe1f0"
  instance_type  = "t2.micro"
  key_name       = "minha-key"
  volume_size    = 50
  volume_type    = "gp3"

  # Configuração do Auto Scaling Group
  asg_min_size         = 2
  asg_max_size         = 5
  asg_desired_capacity = 3

  # Certificado ACM
  certificate_arn = "arn:aws:acm:us-east-1:123456789012:certificate/abcd-efgh-ijkl-mnop"

  # Target Groups
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

  # Listeners (HTTP → HTTPS + HTTPS)
  listeners = [
    { port = 80, protocol = "HTTP", target_group = "ec2-tg" },
    { port = 443, protocol = "HTTPS", target_group = "ec2-tg" }
  ]
}


Resultado
- O Load Balancer terá dois listeners:
- Porta 80 (HTTP) → redireciona automaticamente para HTTPS (443).
- Porta 443 (HTTPS) → usa o certificado teste.pactual.net do ACM.
- Todo tráfego será servido de forma segura via HTTPS.
- O Target Group continua recebendo tráfego das instâncias EC2, mas agora apenas via HTTPS.



###############
###############


Perfeito — vamos configurar o Application Load Balancer (ALB) para suportar múltiplos certificados via SNI (Server Name Indication). Assim, o mesmo LB pode servir diferentes domínios, como teste.pactual.net e api.pactual.net.


1º Variáveis para múltiplos certificados (modules/ec2-lb/variables.tf)

variable "certificates" {
  description = "Lista de certificados ACM para o Load Balancer"
  type        = list(string)
  default     = []
}


2º Listener HTTPS com SNI (modules/ec2-lb/main.tf)

# Listener HTTPS principal
resource "aws_lb_listener" "https_listener" {
  load_balancer_arn = aws_lb.app_lb.arn
  port              = 443
  protocol          = "HTTPS"
  ssl_policy        = "ELBSecurityPolicy-2016-08"

  # Primeiro certificado obrigatório
  certificate_arn   = var.certificates[0]

  default_action {
    type             = "forward"
    target_group_arn = aws_lb_target_group.tg["ec2-tg"].arn
  }
}

# Certificados adicionais via SNI
resource "aws_lb_listener_certificate" "extra_certs" {
  for_each        = toset(slice(var.certificates, 1, length(var.certificates)))
  listener_arn    = aws_lb_listener.https_listener.arn
  certificate_arn = each.value
}


3º Listener HTTP com redirect para HTTPS

resource "aws_lb_listener" "http_redirect" {
  load_balancer_arn = aws_lb.app_lb.arn
  port              = 80
  protocol          = "HTTP"

  default_action {
    type = "redirect"
    redirect {
      port        = "443"
      protocol    = "HTTPS"
      status_code = "HTTP_301"
    }
  }
}


4º Consumidor do módulo (main.tf)
Agora você passa os certificados diretamente:

module "ec2_lb" {
  source         = "github.com/<SEU_USUARIO>/<SEU_REPOSITORIO>//modules/ec2-lb?ref=v1.0.0"
  vpc_id         = "vpc-123456"
  subnets        = ["subnet-abc123", "subnet-def456"]
  ami_id         = "ami-0c55b159cbfafe1f0"
  instance_type  = "t2.micro"
  key_name       = "minha-key"
  volume_size    = 50
  volume_type    = "gp3"

  # Auto Scaling Group
  asg_min_size         = 2
  asg_max_size         = 5
  asg_desired_capacity = 3

  # Certificados ACM (SNI)
  certificates = [
    "arn:aws:acm:us-east-1:123456789012:certificate/abcd-efgh-ijkl-mnop", # teste.pactual.net
    "arn:aws:acm:us-east-1:123456789012:certificate/qrst-uvwx-yzab-cdef"  # api.pactual.net
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
    { port = 80, protocol = "HTTP", target_group = "ec2-tg" },
    { port = 443, protocol = "HTTPS", target_group = "ec2-tg" }
  ]
}


Resultado
- O Load Balancer terá:
- Listener HTTP (80) → redireciona para HTTPS.
- Listener HTTPS (443) → usa múltiplos certificados ACM via SNI.
- Clientes acessando teste.pactual.net ou api.pactual.net receberão o certificado correto automaticamente.
- O tráfego será encaminhado para o Target Group ec2-tg, com health checks e stickiness configurados.



###############
###############

Para parametrizar regras de listener com roteamento baseado em path e/ou host, você pode estruturar o módulo de forma que receba essas regras como input e aplique dinamicamente no recurso de listener. 

1º Cadastraremos a variavel necessario em (modules/ec2-lb/variables.tf).

variable "listener_rules" {
  type = list(object({
    priority = number
    host     = string
    path     = string
    target_group_arn = string
  }))
}



2º Agora cadastre a entrada no modulo(modules/ec2-lb/main.tf).

resource "aws_lb_listener_rule" "this" {
  for_each = { for rule in var.listener_rules : "${rule.host}-${rule.path}" => rule }

  listener_arn = aws_lb_listener.front_end.arn
  priority     = each.value.priority

  action {
    type             = "forward"
    target_group_arn = each.value.target_group_arn
  }

  condition {
    host_header {
      values = [each.value.host]
    }
  }

  condition {
    path_pattern {
      values = [each.value.path]
    }
  }
}

3º Consumidor do módulo (main.tf).

listener_rules = [
  {
    priority        = 10
    host            = "api.meuservico.com"
    path            = "/api/*"
    target_group_arn = aws_lb_target_group.api.arn
  },
  {
    priority        = 20
    host            = "app.meuservico.com"
    path            = "/app/*"
    target_group_arn = aws_lb_target_group.app.arn
  }
]

Assim, o módulo fica parametrizável: você só precisa passar a lista de regras como input, e ele cria dinamicamente os listener rules para cada combinação de host/path → target group.

###############
###############

 Para deixar o input mais flexível e evitar duplicação de regras quando você tem múltiplos paths para o mesmo host, você pode modelar o input como uma lista de objetos onde cada host tem um conjunto de paths. Assim, o módulo gera uma única regra de listener por host, mas com múltiplos .

1º Cadastraremos a variavel necessario em (modules/ec2-lb/variables.tf).


variable "listener_rules" {
  type = list(object({
    priority        = number
    host            = string
    paths           = list(string)
    target_group_arn = string
  }))
}


2º Agora cadastre a entrada no modulo(modules/ec2-lb/main.tf).

resource "aws_lb_listener_rule" "this" {
  for_each = { for rule in var.listener_rules : rule.host => rule }

  listener_arn = aws_lb_listener.front_end.arn
  priority     = each.value.priority

  action {
    type             = "forward"
    target_group_arn = each.value.target_group_arn
  }

  condition {
    host_header {
      values = [each.value.host]
    }
  }

  condition {
    path_pattern {
      values = each.value.paths
    }
  }
}


3º Consumidor do módulo (main.tf).

listener_rules = [
  {
    priority        = 10
    host            = "api.meuservico.com"
    paths           = ["/api/*", "/v1/*", "/v2/*"]
    target_group_arn = aws_lb_target_group.api.arn
  },
  {
    priority        = 20
    host            = "app.meuservico.com"
    paths           = ["/app/*", "/dashboard/*"]
    target_group_arn = aws_lb_target_group.app.arn
  }
]


Com isso, você consegue:
• 	Definir múltiplos paths para o mesmo host sem duplicar regras.
• 	Manter o input mais limpo e fácil de expandir.
• 	Garantir que cada regra de listener seja única por host, mas abrangente nos paths.


###############
###############

Para passar essas tags obrigatórias via input no seu , você pode usar variáveis no Terraform e depois aplicar no recurso com  ou diretamente em . Aqui vai um exemplo prático:


Definição das variáveis (variables.tf)

variable "tags" {
  description = "Tags obrigatórias para os recursos"
  type = map(string)
  default = {
    Name        = "WEB-APP"
    Enviroment  = "Dev"
    Owner       = "Wanderson"
    Tecnologia  = "EC2"
    Application = "risk-manager"
    Team        = "prevenção a Fraude"
    Region      = "us-east-1"
    Critical    = "Low"
  }
}


 Uso no (main.tf)

 provider "aws" {
  region = var.tags["Region"]

  default_tags {
    tags = var.tags
  }
}

resource "aws_instance" "web" {
  ami           = "ami-1234567890abcdef0"
  instance_type = "t2.micro"

  tags = var.tags
}

###############
###############


* Passo a Passo para criar e enviar uma tag no GIT.

1. Verifique se está no branch correto
Normalmente você cria tags a partir do branch principal ( ou ):

git checkout main
git pull origin main


2. Crie a tag localmente
Existem dois tipos de tags:
• 	Tag leve (lightweight): apenas um marcador simples.
• 	Tag anotada (annotated): contém mensagem, autor e data (recomendado para versionamento).
👉 Exemplo de tag anotada:

git tag -a v1.0.0 -m "Primeira versão estável do módulo EC2-LB"


3. Liste as tags criadas
Para confirmar que a tag foi criada

git tag


4. Envie a tag para o GitHub

git push origin v1.0.0


Se quiser enviar todas as tags de uma vez:
git push origin --tags
