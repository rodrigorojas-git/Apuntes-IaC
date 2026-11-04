# Apuntes Terraform: Evaluación 2 (IaC I)

1. [Comandos y debugging](#1-comandos-y-debugging)
2. [Antes de empezar](#2-antes-de-empezar)
3. [Ejercicio resuelto](#3-ejercicio-resuelto)
4. [Qué cambiar según lo que pidan](#4-qué-cambiar-según-lo-que-pidan)
5. [Errores comunes](#5-errores-comunes)
6. [Código por recurso (Registry)](#6-código-por-recurso-registry)
7. [Repaso rápido](#7-repaso-rápido)

---

## 1. Comandos y debugging

Se ejecutan **dentro de `evaluacion2/`** (en VS Code: `cd evaluacion2`).

```bash
terraform init        # primero (y de nuevo si agregas provider o módulo)
terraform validate
terraform plan
terraform apply       # escribir: yes
terraform output
terraform destroy
```

**Debugging:** mira cómo empieza la línea de tu terminal en VS Code.

| Terminal | Activar | Desactivar |
|---|---|---|
| PowerShell (`PS C:\>`) | `$env:TF_LOG="DEBUG"`<br>`$env:TF_LOG_PATH="terraform.log"` | `Remove-Item Env:TF_LOG, Env:TF_LOG_PATH` |
| CMD (`C:\>`) | `set TF_LOG=DEBUG`<br>`set TF_LOG_PATH=terraform.log` | `set TF_LOG=`<br>`set TF_LOG_PATH=` |
| Git Bash (`$`) | `export TF_LOG=DEBUG`<br>`export TF_LOG_PATH=./terraform.log` | `unset TF_LOG TF_LOG_PATH` |

Después de activarlo, ejecuta `terraform plan`. Niveles: `TRACE` (máximo) → `DEBUG` → `INFO` → `WARN` → `ERROR`. Solo funciona en esa terminal.

---

## 2. Antes de empezar

- [ ] Learner Lab: **Start Lab** (círculo verde).
- [ ] **AWS Details → AWS CLI → Show**: copiar a `C:\Users\<usuario>\.aws\credentials` (crear la carpeta `.aws` si no existe). El archivo se llama `credentials` **sin extensión**: ojo con que el Bloc de notas no lo guarde como `credentials.txt`. Las usan Terraform y el AWS CLI del `local-exec` de S3. Si dice `ExpiredToken`, se vuelven a copiar.
- [ ] **AWS Details → Download PEM**: dejar `labsuser.pem` dentro de `evaluacion2/`. Solo lo usa el `remote-exec` de las EC2 (SSH); S3 no lo necesita.
- [ ] Verificar: `aws sts get-caller-identity`. Si muestra tu cuenta, está bien.
- [ ] Región `us-east-1`, key pair `vockey`.
- [ ] **Pegar el ID de la AMI** en `terraform.tfvars` → `ami_id = "ami-0..."`.

**Cómo sacar el ID de la AMI (Amazon Linux 2023):**
- Consola: EC2 → **Launch instance** → **Amazon Linux 2023** → copiar el `ami-0...` (64-bit x86) → cancelar.
- Terminal:
```bash
aws ec2 describe-images --owners amazon --filters "Name=name,Values=al2023-ami-2023*-x86_64" --query "sort_by(Images,&CreationDate)[-1].ImageId" --output text
```
El ID cambia según la región. Si usas **Ubuntu**, cambia en `modulos/ec2/main.tf` el `user = "ubuntu"` y usa `apt` en vez de `dnf`.

**Durante la prueba**
- Lee todo el enunciado y arma tu checklist con cada requisito; ajusta CIDR, cantidades y nombres de carpetas exactamente como los piden.
- Activa el debugging con `TF_LOG_PATH` al inicio: la terminal queda limpia y el `.log` queda como evidencia.
- Usa `terraform validate` seguido y lanza el `apply` temprano (los NLB tardan varios minutos).
- **Pantallazos a medida que avanzas:** `init`, `validate`, debugging, `plan`, `apply`, `output`, recursos en la consola de AWS y `destroy`.
- Pregunta antes de hacer `destroy`, por si quieren ver los recursos creados.
- Entiende lo que copias: pueden preguntarte qué hace cada parte.

---

## 3. Ejercicio resuelto

Carpeta **[`evaluacion2/`](evaluacion2/)**, lista para usar.

```
evaluacion2/
├── providers.tf, variables.tf, outputs.tf, main.tf
├── terraform.tfvars          ← lo que más se cambia
└── modulos/
    ├── vpc/  ec2/  nlb/  s3/  (main.tf, variables.tf, outputs.tf)
```

**Crea:** VPC `10.0.0.0/16`, 2 subredes públicas, IGW, route table, 2 EC2 (con `remote-exec`), 2 NLB y 2 S3 privados (con `local-exec`).

| Requisito | Dónde |
|---|---|
| Variables / outputs en raíz e hijos | `variables.tf` / `outputs.tf` de cada carpeta |
| `random` 6 caracteres en `locals` | `main.tf` raíz → `local.prefijo` |
| `locals` en todos los módulos | `local.nombre` en cada `modulos/*/main.tf` |
| `count` / `for_each` | `count`: subredes, EC2, NLB · `for_each`: S3 |
| `remote-exec` / `local-exec` | `modulos/ec2` / `modulos/s3` |
| AMI por variable (raíz → hijo → tfvars) | `ami_id` en `variables.tf`, `module "ec2"` y `terraform.tfvars` |
| Debugging | Sección 1 |

**Nombres de los recursos.** Cada nombre se usa solo dentro de su módulo, en su `main.tf` y su `outputs.tf`.

| Módulo | Recurso | Se referencia como |
|---|---|---|
| vpc | `aws_vpc "vpc_principal"` | `aws_vpc.vpc_principal.id` |
| vpc | `aws_internet_gateway "igw"` | `aws_internet_gateway.igw.id` |
| vpc | `aws_subnet "subred_publica"` | `aws_subnet.subred_publica[0].id` |
| vpc | `aws_route_table "tabla_publica"` | `aws_route_table.tabla_publica.id` |
| vpc | `aws_route_table_association "asociacion_publica"` | — |
| ec2 | `aws_security_group "sg_ssh"` | `aws_security_group.sg_ssh.id` |
| ec2 | `aws_instance "servidor"` | `aws_instance.servidor.public_ip` |
| nlb | `aws_lb "nlb"` | `aws_lb.nlb.dns_name` |
| s3 | `null_resource "bucket_privado"` | `null_resource.bucket_privado` |

Para renombrar: cambia el nombre y todas sus referencias del mismo módulo (`Ctrl + Shift + H` en VS Code). La raíz no se toca porque usa los outputs (`module.ec2[0].public_ip`).

> En Learner Lab `aws_s3_bucket` no funciona, por eso S3 se crea con AWS CLI vía `local-exec`.

---

## 4. Qué cambiar según lo que pidan

| Si piden... | Cambiar |
|---|---|
| Otros CIDR / más subredes | `terraform.tfvars` → `vpc_cidr`, `cidr_subredes` (EC2 y NLB se crean uno por subred) |
| Más o menos buckets | `terraform.tfvars` → `buckets` |
| Otro nombre | `terraform.tfvars` → `proyecto` (corto, el NLB admite máx. 32 caracteres) |
| Otra AMI | `terraform.tfvars` → `ami_id` (si es Ubuntu: `user = "ubuntu"` y `apt`) |
| Otro tipo de EC2 | En `module "ec2"`: `instance_type = "t3.small"` |
| Abrir puerto 80 | Copiar el bloque `ingress` en `modulos/ec2/main.tf` con `80` |
| Instalar algo en EC2 | `inline` de `remote-exec`: `"sudo dnf install -y httpd"`, `"sudo systemctl enable --now httpd"` |
| ALB en vez de NLB | Sección 6 → ALB (2 subredes + SG) |
| Conectar LB con EC2 | Sección 6 → target group + listener + attachment |
| Route table "principal" | En `modulos/vpc/main.tf`: `resource "aws_route_table" "principal" { vpc_id = aws_vpc.vpc_principal.id }` |
| `for_each` en módulos | Sección 7 → Módulos |

---

## 5. Errores comunes

| Error | Solución |
|---|---|
| `No configuration files` | Estás en otra carpeta → `cd evaluacion2` |
| `Module/provider not installed` | `terraform init` |
| `No valid credential sources` / `ExpiredToken` | Volver a copiar las credenciales del lab; revisar que el archivo no sea `credentials.txt` |
| `has no internet gateway` (NLB/ALB) | El LB se creó antes que el IGW; el ejercicio lo evita con `depends_on` en `modulos/vpc/outputs.tf`. Si te pasa, vuelve a hacer `apply` |
| `remote-exec` se queda en *Still creating...* | Revisar IGW, ruta `0.0.0.0/0`, IP pública y puerto 22 |
| `Invalid function argument` (pem) | `labsuser.pem` no está en `evaluacion2/` |
| `InvalidAMIID.Malformed` / `NotFound` | Falta pegar el ID real en `ami_id`, está mal copiado o es de otra región |
| `cannot be longer than 32 characters` | Acortar `proyecto` |
| `BucketAlreadyExists` / nombre inválido | Cambiar `proyecto`; solo minúsculas, números y guiones |
| `Reference to undeclared...` | Nombre mal escrito o variable/output no declarado en el módulo |

**Plan B:** si `remote-exec` falla, comenta `connection` y `provisioner` (`Ctrl + K, Ctrl + C`) y vuelve a hacer `apply`. Saca pantallazos a medida que avanzas.

---

## 6. Código por recurso (Registry)

### Providers
🔗 https://registry.terraform.io/providers/hashicorp/aws/latest/docs
```hcl
terraform {
  required_providers {
    aws    = { source = "hashicorp/aws", version = "~> 5.0" }
    random = { source = "hashicorp/random", version = "~> 3.1" }
  }
}

provider "aws" {
  region = "us-east-1"
}
```

### Random + locals
🔗 https://registry.terraform.io/providers/hashicorp/random/latest/docs/resources/string
```hcl
resource "random_string" "sufijo" {
  length  = 6
  upper   = false
  special = false
}

locals {
  prefijo = "${var.proyecto}-${random_string.sufijo.result}"
}
```

### VPC + zonas
🔗 https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/vpc
🔗 https://registry.terraform.io/providers/hashicorp/aws/latest/docs/data-sources/availability_zones
```hcl
resource "aws_vpc" "vpc_principal" {
  cidr_block = "10.0.0.0/16"
}

data "aws_availability_zones" "disponibles" {
  state = "available"
}
```

### Subnet
🔗 https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/subnet
```hcl
resource "aws_subnet" "subred_publica" {
  count                   = 2
  vpc_id                  = aws_vpc.vpc_principal.id
  cidr_block              = ["10.0.1.0/24", "10.0.2.0/24"][count.index]
  availability_zone       = data.aws_availability_zones.disponibles.names[count.index]
  map_public_ip_on_launch = true
}
```

### Internet Gateway + route table + asociación
🔗 https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/internet_gateway
🔗 https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/route_table
🔗 https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/route_table_association
```hcl
resource "aws_internet_gateway" "igw" {
  vpc_id = aws_vpc.vpc_principal.id
}

resource "aws_route_table" "tabla_publica" {
  vpc_id = aws_vpc.vpc_principal.id
  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_internet_gateway.igw.id
  }
}

resource "aws_route_table_association" "asociacion_publica" {
  count          = 2
  subnet_id      = aws_subnet.subred_publica[count.index].id
  route_table_id = aws_route_table.tabla_publica.id
}
```

### Security Group
🔗 https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/security_group
```hcl
resource "aws_security_group" "sg_web" {
  name   = "sg-web"
  vpc_id = aws_vpc.vpc_principal.id

  dynamic "ingress" {
    for_each = [22, 80]
    content {
      from_port   = ingress.value
      to_port     = ingress.value
      protocol    = "tcp"
      cidr_blocks = ["0.0.0.0/0"]
    }
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
}
```

### EC2 (AMI por variable)
🔗 https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/instance
```hcl
variable "ami_id" {
  type = string
}
# terraform.tfvars → ami_id = "ami-0..."

resource "aws_instance" "servidor" {
  count                  = 2
  ami                    = var.ami_id
  instance_type          = "t3.micro"
  subnet_id              = aws_subnet.subred_publica[count.index].id
  key_name               = "vockey"
  vpc_security_group_ids = [aws_security_group.sg_web.id]
}
```

**Alternativa: buscar la AMI automáticamente** (sin pegar el ID)
🔗 https://registry.terraform.io/providers/hashicorp/aws/latest/docs/data-sources/ami
```hcl
data "aws_ami" "al2023" {
  most_recent = true
  owners      = ["amazon"]
  filter {
    name   = "name"
    values = ["al2023-ami-2023*-x86_64"]
  }
}
# en la instancia: ami = data.aws_ami.al2023.id
```

### Network Load Balancer
🔗 https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/lb
```hcl
resource "aws_lb" "nlb" {
  name               = "mi-nlb"          # máx. 32 caracteres
  load_balancer_type = "network"
  subnets            = [aws_subnet.subred_publica[0].id]
}
```

**ALB:** `load_balancer_type = "application"`, `subnets = aws_subnet.subred_publica[*].id` (mínimo 2, en AZ distintas) y `security_groups = [aws_security_group.sg_web.id]`.

### Target group + listener + attachment
🔗 https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/lb_target_group
🔗 https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/lb_listener
🔗 https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/lb_target_group_attachment
```hcl
resource "aws_lb_target_group" "tg_web" {
  name     = "tg-web"
  port     = 80
  protocol = "TCP"                       # ALB: "HTTP"
  vpc_id   = aws_vpc.vpc_principal.id
}

resource "aws_lb_listener" "escucha_web" {
  load_balancer_arn = aws_lb.nlb.arn
  port              = 80
  protocol          = "TCP"              # ALB: "HTTP"
  default_action {
    type             = "forward"
    target_group_arn = aws_lb_target_group.tg_web.arn
  }
}

resource "aws_lb_target_group_attachment" "registro_web" {
  count            = 2
  target_group_arn = aws_lb_target_group.tg_web.arn
  target_id        = aws_instance.servidor[count.index].id
  port             = 80
}
```

### S3 privado (si el lab lo permite)
🔗 https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/s3_bucket
🔗 https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/s3_bucket_public_access_block
```hcl
resource "aws_s3_bucket" "bucket_privado" {
  for_each = toset(["datos", "logs"])
  bucket   = "${local.prefijo}-${each.key}"
}

resource "aws_s3_bucket_public_access_block" "bloqueo" {
  for_each                = aws_s3_bucket.bucket_privado
  bucket                  = each.value.id
  block_public_acls       = true
  block_public_policy     = true
  ignore_public_acls      = true
  restrict_public_buckets = true
}
```

### S3 con local-exec (Learner Lab)
```hcl
resource "null_resource" "bucket_privado" {
  for_each = toset(["datos", "logs"])
  triggers = { bucket = "${local.prefijo}-${each.key}" }

  provisioner "local-exec" {
    command = "aws s3api create-bucket --bucket ${self.triggers.bucket} --region us-east-1"
  }
  provisioner "local-exec" {
    when    = destroy
    command = "aws s3 rb s3://${self.triggers.bucket} --force"
  }
}
```

---

## 7. Repaso rápido

### count / for_each / for
🔗 https://developer.hashicorp.com/terraform/language/meta-arguments/count
🔗 https://developer.hashicorp.com/terraform/language/meta-arguments/for_each
```hcl
count    = 2                          # count.index → 0, 1 · ref: recurso[0], recurso[*].id
for_each = toset(["a", "b"])          # each.key    · ref: recurso["a"]
for_each = { a = "10.0.1.0/24" }      # each.key, each.value
lista    = [for x in var.lista : upper(x)]
```
`count` usa índices: si borras uno del medio, se recrean los siguientes. `for_each` usa claves: solo se borra ese.

### Variables y outputs
🔗 https://developer.hashicorp.com/terraform/language/values/variables
```hcl
variable "ambiente" {
  type    = string                    # string, number, bool, list(string), set(string), map(string)
  default = "dev"
  validation {
    condition     = contains(["dev", "prod"], var.ambiente)
    error_message = "Debe ser dev o prod."
  }
}

output "ips" {
  value     = module.ec2[*].public_ip
  sensitive = false                   # true = la oculta en pantalla (igual queda en el state)
}
```
Precedencia (gana el último): default → `TF_VAR_x` → `terraform.tfvars` → `*.auto.tfvars` → `-var`.

### Módulos
🔗 https://developer.hashicorp.com/terraform/language/modules/syntax
```hcl
module "vpc" {
  source   = "./modulos/vpc"
  vpc_cidr = var.vpc_cidr             # variable declarada en modulos/vpc/variables.tf
}
# leer un output del hijo: module.vpc.vpc_id

module "ec2" {                        # versión con for_each
  source    = "./modulos/ec2"
  for_each  = { web0 = 0, web1 = 1 }
  nombre    = each.key
  subnet_id = module.vpc.subnet_ids[each.value]
  # ...resto de argumentos
}
# output: { for k, m in module.ec2 : k => m.instance_id }
```

### Locals
```hcl
locals {
  nombre = "${var.prefijo}-${var.nombre}"
}
# uso: local.nombre
```
Variable = viene de afuera. Local = valor calculado interno.

### Provisioners
🔗 https://developer.hashicorp.com/terraform/language/provisioners
```hcl
resource "aws_instance" "servidor" {
  # ...
  connection {
    type        = "ssh"
    user        = "ec2-user"
    private_key = file("labsuser.pem")
    host        = self.public_ip
  }

  provisioner "remote-exec" {          # corre DENTRO de la instancia
    inline = ["sudo dnf install -y httpd", "sudo systemctl enable --now httpd"]
  }

  provisioner "local-exec" {           # corre en TU máquina
    command = "echo ${self.public_ip} >> ips.txt"
  }
}
```
- `remote-exec` necesita: IP pública, IGW + ruta, puerto 22 abierto, key pair y `connection`.
- `when = destroy`: se ejecuta al destruir.
- `on_failure = continue`: no se detiene si falla (por defecto falla y el recurso queda *tainted*).
