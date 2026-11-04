# Posibles agregados para la prueba

Cambios **sobre el ejercicio de `evaluacion2/`** por si piden algo extra. Cada uno indica qué archivo tocar y el bloque a copiar. Están validados sobre el código del repo.

| # | Agregado | Probabilidad |
|---|---|---|
| 1 | [Apache con `remote-exec` + puerto 80](#1-apache-con-remote-exec--puerto-80) | Alta |
| 2 | [Guardar IPs en un archivo con `local-exec`](#2-guardar-ips-en-un-archivo-con-local-exec) | Alta |
| 3 | [Puertos con `dynamic`](#3-puertos-con-dynamic) | Alta |
| 4 | [Conectar NLB con las EC2](#4-conectar-nlb-con-las-ec2) | Media |
| 5 | [`for_each` en vez de `count` (EC2)](#5-for_each-en-vez-de-count-ec2) | Media |
| 6 | [Validaciones y más outputs](#6-validaciones-y-más-outputs) | Media |
| 7 | [Módulo `sg` separado](#7-módulo-sg-separado) | Baja |
| 8 | [Route table "principal"](#8-route-table-principal) | Baja |
| 9 | [Cambiar cantidades](#9-cambiar-cantidades) | Baja |

Después de cualquier cambio: `terraform init` (si agregaste un módulo) → `terraform validate` → `plan` → `apply`.

---

## 1. Apache con remote-exec + puerto 80

**`modulos/ec2/main.tf`**: en el security group, agrega este bloque junto al `ingress` del 22. Si usas el agregado 3, no lo agregues: basta con poner `80` en la lista de puertos.
```hcl
  ingress {
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }
```
Reemplaza el `provisioner "remote-exec"` de la instancia por:
```hcl
  provisioner "remote-exec" {
    inline = [
      "sudo dnf install -y httpd",
      "sudo systemctl enable --now httpd",
      "echo '<h1>${local.nombre}</h1>' | sudo tee /var/www/html/index.html",
    ]
  }
```

**`outputs.tf` (raíz)**, opcional, para tener el link directo:
```hcl
output "urls_web" {
  value = [for ip in module.ec2[*].public_ip : "http://${ip}"]
}
```

**Probar:** abre la URL en el navegador (con `http://`, no `https://`) y saca un pantallazo con el nombre de la instancia.

---

## 2. Guardar IPs en un archivo con local-exec

**`modulos/ec2/main.tf`**: dentro de `resource "aws_instance" "servidor"`, debajo del `remote-exec`:
```hcl
  provisioner "local-exec" {
    command = "echo ${local.nombre} ${self.public_ip} >> ips.txt"
  }
```

**Probar:** después del `apply` aparece `ips.txt` en `evaluacion2/`, con una línea por instancia. Bórralo antes de repetir el `apply`, porque `>>` agrega líneas al final.

---

## 3. Puertos con dynamic

Reemplaza el `ingress` fijo por una lista de puertos que viene del tfvars.

**`modulos/ec2/variables.tf`**, agregar:
```hcl
variable "puertos" {
  type = list(number)
}
```

**`modulos/ec2/main.tf`**: en el security group, reemplaza el bloque `ingress { ... 22 ... }` por:
```hcl
  dynamic "ingress" {
    for_each = var.puertos
    content {
      from_port   = ingress.value
      to_port     = ingress.value
      protocol    = "tcp"
      cidr_blocks = ["0.0.0.0/0"]
    }
  }
```

**`variables.tf` (raíz)**, agregar:
```hcl
variable "puertos" {
  type = list(number)
}
```

**`main.tf` (raíz)**: dentro de `module "ec2"`, agrega:
```hcl
  puertos   = var.puertos
```

**`terraform.tfvars`**, agregar:
```hcl
puertos       = [22, 80]
```

`ingress.value` es cada puerto de la lista. **El 22 siempre tiene que estar**, porque lo necesita el `remote-exec`.

---

## 4. Conectar NLB con las EC2

Cada NLB reenvía el puerto 80 a la EC2 de su misma subred. **Requiere el agregado 1** (Apache + puerto 80); sin un servidor web, el target queda *unhealthy*.

**`modulos/nlb/variables.tf`**, agregar:
```hcl
variable "vpc_id" {
  type = string
}

variable "instance_id" {
  type = string
}
```

**`modulos/nlb/main.tf`**, agregar debajo del `aws_lb`:
```hcl
resource "aws_lb_target_group" "tg_web" {
  name     = "${local.nombre}-tg"
  port     = 80
  protocol = "TCP"
  vpc_id   = var.vpc_id
}

resource "aws_lb_listener" "escucha_web" {
  load_balancer_arn = aws_lb.nlb.arn
  port              = 80
  protocol          = "TCP"

  default_action {
    type             = "forward"
    target_group_arn = aws_lb_target_group.tg_web.arn
  }
}

resource "aws_lb_target_group_attachment" "registro_web" {
  target_group_arn = aws_lb_target_group.tg_web.arn
  target_id        = var.instance_id
  port             = 80
}
```

**`main.tf` (raíz)**: `module "nlb"` queda así:
```hcl
module "nlb" {
  source = "./modulos/nlb"
  count  = length(var.cidr_subredes)

  prefijo     = local.prefijo
  nombre      = "nlb-${count.index}"
  subnet_id   = module.vpc.subnet_ids[count.index]
  vpc_id      = module.vpc.vpc_id
  instance_id = module.ec2[count.index].instance_id
}
```

**Probar:** abre `http://<nlb_dns>` (sale en `terraform output`). El target tarda 1 a 3 minutos en quedar *healthy* (EC2 → Target Groups → Targets).

---

## 5. for_each en vez de count (EC2)

Sirve si piden `for_each`, instancias con nombres propios o de distinto tamaño.

**`variables.tf` (raíz)**, agregar:
```hcl
variable "instancias" {
  type = map(object({
    subred = number
    tipo   = string
  }))
}
```

**`terraform.tfvars`**, agregar:
```hcl
instancias = {
  web = { subred = 0, tipo = "t3.micro" }
  app = { subred = 1, tipo = "t3.small" }
}
```

**`main.tf` (raíz)**: reemplaza `module "ec2"` completo por:
```hcl
module "ec2" {
  source   = "./modulos/ec2"
  for_each = var.instancias

  prefijo       = local.prefijo
  nombre        = each.key
  instance_type = each.value.tipo
  vpc_id        = module.vpc.vpc_id
  subnet_id     = module.vpc.subnet_ids[each.value.subred]
  ami_id        = var.ami_id
}
```

**`outputs.tf` (raíz)**: con `for_each` el splat `[*]` no funciona. Cambia el output de las IPs por:
```hcl
output "ec2_ips_publicas" {
  value = { for k, m in module.ec2 : k => m.public_ip }
}
```

`each.key` es `web` / `app`; `each.value.tipo` y `each.value.subred` vienen del map. Las referencias pasan a ser `module.ec2["web"]`.

> No lo combines con el agregado 4: el NLB está pensado para `count` (una EC2 por subred). Si piden ambos, deja `count` en EC2.

---

## 6. Validaciones y más outputs

**`variables.tf` (raíz)**: reemplaza las variables `proyecto` y `vpc_cidr` por:
```hcl
variable "proyecto" {
  type = string

  validation {
    condition     = length(var.proyecto) <= 10
    error_message = "El proyecto debe tener 10 caracteres o menos (nombre del NLB máx. 32)."
  }
}

variable "vpc_cidr" {
  type = string

  validation {
    condition     = can(cidrhost(var.vpc_cidr, 0))
    error_message = "vpc_cidr debe ser un CIDR válido, ej: 10.0.0.0/16."
  }
}
```

**IP privada como output sensible.** En `modulos/ec2/outputs.tf`, agregar:
```hcl
output "private_ip" {
  value = aws_instance.servidor.private_ip
}
```
En `outputs.tf` (raíz), agregar:
```hcl
output "ec2_ips_privadas" {
  value     = module.ec2[*].private_ip
  sensitive = true
}
```

**Probar:** `terraform output` muestra `<sensitive>`, y `terraform output ec2_ips_privadas` muestra el valor. Para probar la validación, pon `proyecto = "nombremuylargo"` y ejecuta `plan`: debe aparecer tu `error_message`.

---

## 7. Módulo sg separado

Saca el security group de EC2 a su propio módulo. Se crea **1 SG compartido** por las 2 instancias.

**Crear `modulos/sg/variables.tf`:**
```hcl
variable "prefijo" {
  type = string
}

variable "vpc_id" {
  type = string
}

variable "puertos" {
  type = list(number)
}
```

**Crear `modulos/sg/main.tf`:**
```hcl
locals {
  nombre = "${var.prefijo}-sg"
}

resource "aws_security_group" "sg_web" {
  name   = local.nombre
  vpc_id = var.vpc_id

  dynamic "ingress" {
    for_each = var.puertos
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

**Crear `modulos/sg/outputs.tf`:**
```hcl
output "sg_id" {
  value = aws_security_group.sg_web.id
}
```

**`modulos/ec2/main.tf`**: borra el `resource "aws_security_group" "sg_ssh"` completo y en la instancia cambia:
```hcl
  vpc_security_group_ids = [var.sg_id]
```

**`modulos/ec2/variables.tf`**: cambia `variable "vpc_id"` por:
```hcl
variable "sg_id" {
  type = string
}
```

**`main.tf` (raíz)**: agrega el módulo antes de `module "ec2"`:
```hcl
module "sg" {
  source = "./modulos/sg"

  prefijo = local.prefijo
  vpc_id  = module.vpc.vpc_id
  puertos = var.puertos
}
```
Dentro de `module "ec2"`, cambia la línea `vpc_id = module.vpc.vpc_id` por:
```hcl
  sg_id     = module.sg.sg_id
```

**`variables.tf` (raíz)** y **`terraform.tfvars`**: agrega `puertos`, igual que en el agregado 3.

Al ser un módulo nuevo, **`terraform init`** antes del `validate`.

---

## 8. Route table "principal"

**`modulos/vpc/main.tf`**, agregar al final:
```hcl
resource "aws_route_table" "principal" {
  vpc_id = aws_vpc.vpc_principal.id
}
```
No se asocia a ninguna subred: las subredes públicas siguen usando `tabla_publica`.

---

## 9. Cambiar cantidades

Solo se cambia **`terraform.tfvars`**:
```hcl
cidr_subredes = ["10.0.1.0/24", "10.0.2.0/24", "10.0.3.0/24"]   # 3 subredes → 3 EC2 y 3 NLB
buckets       = ["datos", "logs", "backup"]                       # 3 buckets
```
Hasta 4 subredes funciona sin problema. Desde la 5ª, la subred cae en `us-east-1e`, donde `t3.micro` no está disponible; en ese caso usa `instance_type = "t2.micro"`.
