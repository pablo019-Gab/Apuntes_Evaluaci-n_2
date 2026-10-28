# Apuntes de Terraform

Resumen práctico de: `count` / `for_each`, variables y outputs, módulos, `locals` + `random`, provisioners y debugging.

---

## 0. Comandos básicos

```bash
terraform init                 # Descarga providers y módulos (siempre primero)
terraform fmt                  # Ordena el código
terraform validate             # Revisa la sintaxis
terraform plan                 # Muestra qué va a cambiar
terraform apply                # Aplica los cambios
terraform destroy              # Elimina todo
terraform output               # Muestra los outputs
terraform state list           # Lista los recursos creados
terraform console              # Consola para probar expresiones

terraform apply -var="nombre=prod"      # Pasar una variable
terraform apply -var-file="prod.tfvars" # Pasar un archivo de variables

terraform taint aws_instance.web        # Marcar recurso para recrearlo
terraform untaint aws_instance.web      # Quitar la marca

export TF_LOG=DEBUG                     # Activar logs
export TF_LOG_PATH=./terraform.log      # Guardar logs en archivo
unset TF_LOG TF_LOG_PATH                # Desactivar logs
```

---

## 1. `count` y `for_each`

Sirven para crear **varios recursos** con un solo bloque.

### `count` (recibe un número)

```hcl
variable "nombres" {
  type    = list(string)
  default = ["web", "api", "db"]
}

resource "aws_instance" "servidor" {
  count         = length(var.nombres)          # Crea 3 instancias
  ami           = "ami-0c02fb55956c7d316"
  instance_type = "t2.micro"

  tags = {
    Name = var.nombres[count.index]            # count.index = 0, 1, 2
  }
}

# Cómo acceder:
# aws_instance.servidor[0].id     -> una instancia
# aws_instance.servidor[*].id     -> todas (lista)
```

### `for_each` (recibe un map o set)

```hcl
variable "instancias" {
  type = map(string)
  default = {
    web = "t2.micro"
    api = "t2.small"
  }
}

resource "aws_instance" "servidor" {
  for_each      = var.instancias
  ami           = "ami-0c02fb55956c7d316"
  instance_type = each.value                   # el valor: "t2.micro", "t2.small"

  tags = {
    Name = each.key                            # la llave: "web", "api"
  }
}

# Cómo acceder:
# aws_instance.servidor["web"].id
```

> Si tienes una **lista** y quieres usar `for_each`, conviértela con `toset(var.lista)`. En un set, `each.key` y `each.value` son iguales.

### Diferencias

| | `count` | `for_each` |
|---|---|---|
| Recibe | Número | Map o set |
| Identifica por | Índice `[0]` | Llave `["web"]` |
| Variables | `count.index` | `each.key`, `each.value` |
| Si borras uno del medio | Los índices se corren y se recrean recursos | Solo se borra ese |

- `count` sirve para copias iguales o para condicionales (`count = var.crear ? 1 : 0`).
- `for_each` sirve cuando cada recurso es distinto.
- No se pueden usar los dos en el mismo bloque.

---

## 2. Variables y Outputs

### Variables (`variables.tf`)

```hcl
variable "region" {            # string
  type    = string
  default = "us-east-1"        # valor por defecto
}

variable "zonas" {             # list: se accede con índice
  type    = list(string)
  default = ["us-east-1a", "us-east-1b"]
}

variable "tags" {              # map: se accede con llave
  type = map(string)
  default = {
    Proyecto = "Evaluacion"
    Entorno  = "dev"
  }
}

variable "nombre_proyecto" {   # sin default: Terraform lo pide
  type = string
}
```

Uso: `var.region`, `var.zonas[0]`, `var.tags["Entorno"]`.

### Outputs (`outputs.tf`)

```hcl
output "ip_publica" {
  description = "IP pública de la instancia"
  value       = aws_instance.web.public_ip
}

output "ids" {
  value = aws_instance.servidor[*].id      # todos los ids (con count)
}
```

### Outputs entre módulo hijo y raíz

El módulo raíz **solo puede ver lo que el hijo exporta con `output`**.

```hcl
# modulos/ec2/outputs.tf  (HIJO)
output "instance_id" {
  value = aws_instance.this.id
}

# main.tf  (RAÍZ)
output "id_servidor" {
  value = module.web.instance_id           # module.<nombre_del_módulo>.<output>
}
```

---

## 3. Módulos

Un módulo es una carpeta con archivos `.tf`. Estructura estándar:

```
proyecto/
├── main.tf            # Raíz: llama a los módulos
├── variables.tf
├── outputs.tf
└── modulos/
    └── ec2/
        ├── main.tf        # recursos
        ├── variables.tf   # entradas
        └── outputs.tf     # salidas
```

### Módulo local

```hcl
# modulos/ec2/variables.tf
variable "ami"    { type = string }
variable "nombre" { type = string }

# modulos/ec2/main.tf
resource "aws_instance" "this" {
  ami           = var.ami
  instance_type = "t2.micro"
  tags          = { Name = var.nombre }
}

# modulos/ec2/outputs.tf
output "instance_id" {
  value = aws_instance.this.id
}
```

```hcl
# main.tf (raíz) - llamar al módulo
module "web" {
  source = "./modulos/ec2"           # ruta local

  ami    = "ami-0c02fb55956c7d316"   # estos son las variables del módulo
  nombre = "servidor-web"
}
```

### Módulo del Registry público

```hcl
module "vpc" {
  source  = "terraform-aws-modules/vpc/aws"
  version = "~> 5.0"                 # version solo va en módulos del Registry

  name = "mi-vpc"
  cidr = "10.0.0.0/16"
}

# Usar un output del módulo:
# module.vpc.vpc_id
```

> Cada vez que agregues o cambies un módulo, ejecuta `terraform init` de nuevo.

---

## 4. Locals y provider Random

### `locals`

Sirve para **reutilizar valores** (nombres, tags). Se declara con `locals` y se usa con `local.`

```hcl
locals {
  prefijo = "${var.proyecto}-${var.entorno}"

  tags_comunes = {
    Proyecto = var.proyecto
    Entorno  = var.entorno
  }
}

resource "aws_instance" "web" {
  ami           = "ami-0c02fb55956c7d316"
  instance_type = "t2.micro"

  # merge() junta dos maps
  tags = merge(local.tags_comunes, { Name = "${local.prefijo}-web" })
}
```

### `random_string` (sufijo único)

Genera un texto aleatorio que se **guarda en el estado**, así que no cambia en cada apply.

```hcl
resource "random_string" "sufijo" {
  length  = 6
  special = false     # sin símbolos
  upper   = false     # sin mayúsculas (S3 no las acepta)
}

locals {
  nombre_unico = "${var.proyecto}-${random_string.sufijo.result}"   # .result es el valor
}
```

---

## 5. Provisioners

Ejecutan comandos cuando se **crea** un recurso (o cuando se destruye con `when = destroy`).

### `local-exec` (corre en TU computador)

```hcl
resource "aws_instance" "web" {
  ami           = "ami-0c02fb55956c7d316"
  instance_type = "t2.micro"

  provisioner "local-exec" {
    command = "echo ${self.public_ip} >> ips.txt"    # self = este mismo recurso
  }
}
```

### `remote-exec` (corre DENTRO del servidor, necesita `connection`)

```hcl
resource "aws_instance" "web" {
  ami           = "ami-0c02fb55956c7d316"
  instance_type = "t2.micro"
  key_name      = "mi-llave"

  connection {
    type        = "ssh"
    user        = "ec2-user"                       # usuario de la AMI
    private_key = file("~/.ssh/mi-llave.pem")      # llave privada
    host        = self.public_ip
  }

  provisioner "remote-exec" {
    inline = [
      "sudo yum install -y httpd",
      "sudo systemctl start httpd",
    ]
  }
}
```

| | `local-exec` | `remote-exec` |
|---|---|---|
| Dónde corre | Tu máquina | El servidor remoto |
| `connection` | No | Sí (SSH) |

> Si un provisioner falla, el recurso queda **tainted** y se recrea en el próximo apply.

### Caso AWS Academy: crear el S3 con provisioner

En Academy no deja crear S3 con `aws_s3_bucket`, así que se crea con el **AWS CLI** desde un `null_resource`:

```hcl
resource "random_string" "sufijo" {
  length  = 6
  special = false
  upper   = false
}

locals {
  nombre_bucket = "evaluacion-${random_string.sufijo.result}"
}

resource "null_resource" "bucket_s3" {
  # Se guardan los datos aquí para poder usarlos al destruir
  triggers = {
    bucket = local.nombre_bucket
    region = "us-east-1"
  }

  # Al hacer apply: crea el bucket
  provisioner "local-exec" {
    command = "aws s3api create-bucket --bucket ${self.triggers.bucket} --region ${self.triggers.region}"
  }

  # Al hacer destroy: borra el bucket
  provisioner "local-exec" {
    when    = destroy
    command = "aws s3 rb s3://${self.triggers.bucket} --force"
  }
}
```

Puntos clave:

- Las credenciales de Academy **expiran**. Si sale `ExpiredToken`, vuelve a copiarlas.
- En `us-east-1` no se usa `--create-bucket-configuration`. En otras regiones sí:
  `--create-bucket-configuration LocationConstraint=<región>`
- Con `when = destroy` solo se puede usar `self`, por eso el nombre va en `triggers`.
- Con `remote-exec` se haría igual, pero la EC2 necesita `iam_instance_profile = "LabInstanceProfile"`.

---

## 6. Debugging y Taint

### Logs

Niveles: `TRACE` (más detalle), `DEBUG`, `INFO`, `WARN`, `ERROR`.

```bash
# Activar
export TF_LOG=DEBUG
export TF_LOG_PATH=./terraform.log
terraform plan
cat terraform.log

# Desactivar
unset TF_LOG
unset TF_LOG_PATH
```

> Necesitas **las dos** variables: sin `TF_LOG` no se escribe nada aunque exista `TF_LOG_PATH`.

### Taint / Untaint

**Taint** marca un recurso como dañado para que en el próximo `apply` se **destruya y se vuelva a crear**.

```bash
terraform state list                         # 1. ver los recursos
terraform taint aws_instance.web             # 2. marcar
terraform plan                               # 3. verás "-/+" (destroy and recreate)
terraform apply                              # 4. lo recrea
terraform untaint aws_instance.web           # (si te arrepientes)
```

Sintaxis según el caso:

```bash
terraform taint 'aws_instance.servidor[1]'         # con count
terraform taint 'aws_instance.servidor["web"]'     # con for_each
terraform taint module.web.aws_instance.this       # dentro de un módulo
```

Se usa para repetir un provisioner o forzar que un recurso se cree de nuevo.

> `taint` está deprecado. Lo moderno es `terraform apply -replace="aws_instance.web"`.

---

## 7. Errores comunes y tips

| Error | Solución |
|---|---|
| Olvidar `terraform init` tras agregar provider o módulo | Ejecutar `terraform init` |
| `for_each` con una lista | Usar `toset(var.lista)` |
| `count` y `for_each` en el mismo bloque | Solo se puede uno |
| `aws_instance.servidor.id` con `count` | Usar `aws_instance.servidor[0].id` |
| `aws_instance.srv.id` con `for_each` | Usar `aws_instance.srv["web"].id` |
| `random_string.sufijo` sin `.result` | Usar `random_string.sufijo.result` |
| Acceder a un recurso interno del módulo | El módulo debe exportarlo con `output` |
| `version` en un módulo local | `version` solo sirve en módulos del Registry |
| `taint` con índice sin comillas | Usar comillas simples: `'aws_instance.x[0]'` |
| `host = aws_instance.x.public_ip` dentro de su propio `connection` | Usar `self.public_ip` |
| `remote-exec` da timeout | Revisar Security Group (puerto 22), IP pública y usuario correcto |

Tips para la prueba:

1. Flujo: `init` → `validate` → `plan` → `apply`. Lee siempre el mensaje de error, indica archivo y línea.
2. En el plan: `+` crea, `-` destruye, `~` modifica, `-/+` recrea.
3. Accesos rápidos: `var.x`, `local.x`, `module.nombre.output`, `recurso.nombre[0]`, `recurso.nombre["llave"]`.
4. Si practicas en una cuenta real, haz `terraform destroy` al final.
5. No subas a GitHub: `*.tfstate`, `.terraform/` ni archivos con credenciales.
