# AWS VPC Terraform module

[![Help Contribute to Open Source](https://www.codetriage.com/terraform-aws-modules/terraform-aws-vpc/badges/users.svg)](https://www.codetriage.com/terraform-aws-modules/terraform-aws-vpc)

Modulo Terraform que cria resources de VPC na AWS

Esses são os tipos de resources suportados:

* [VPC](https://www.terraform.io/docs/providers/aws/r/vpc.html)
* [Subnet](https://www.terraform.io/docs/providers/aws/r/subnet.html)
* [Route](https://www.terraform.io/docs/providers/aws/r/route.html)
* [Route table](https://www.terraform.io/docs/providers/aws/r/route_table.html)
* [Internet Gateway](https://www.terraform.io/docs/providers/aws/r/internet_gateway.html)
* [NAT Gateway](https://www.terraform.io/docs/providers/aws/r/nat_gateway.html)
* [VPN Gateway](https://www.terraform.io/docs/providers/aws/r/vpn_gateway.html)
* [VPC Endpoint](https://www.terraform.io/docs/providers/aws/r/vpc_endpoint.html) (Gateway: S3, DynamoDB; Interface: EC2, SSM)
* [RDS DB Subnet Group](https://www.terraform.io/docs/providers/aws/r/db_subnet_group.html)
* [ElastiCache Subnet Group](https://www.terraform.io/docs/providers/aws/r/elasticache_subnet_group.html)
* [Redshift Subnet Group](https://www.terraform.io/docs/providers/aws/r/redshift_subnet_group.html)
* [DHCP Options Set](https://www.terraform.io/docs/providers/aws/r/vpc_dhcp_options.html)
* [Default VPC](https://www.terraform.io/docs/providers/aws/r/default_vpc.html)

## Modo de usar

```hcl
module "vpc" {
  source = "terraform-aws-modules/vpc/aws"

  name = "my-vpc"
  cidr = "10.0.0.0/16"

  azs             = ["eu-west-1a", "eu-west-1b", "eu-west-1c"]
  private_subnets = ["10.0.1.0/24", "10.0.2.0/24", "10.0.3.0/24"]
  public_subnets  = ["10.0.101.0/24", "10.0.102.0/24", "10.0.103.0/24"]

  enable_nat_gateway = true
  enable_vpn_gateway = true

  tags = {
    Terraform = "true"
    Environment = "dev"
  }
}
```
## IPs Externos NAT Gateway

Por padrão esse modo provisiona um novo Elastic IP para cada VPC NAT Gateways.
Isso significa que quando criar uma nova VPC, novos IPs são alocados e quando esta VPC for destruida esses IPs serão descartados.
Algumas vezes é bom mmanter o mesmo IP se você destroi e recria a VPC
Por conta disso é possível definir IPs existentes no NAT Gateway.
Isso previne que a destruição da VPC descarte esses IPs possibilitando o uso dos mesmo IPs na recriação da VPC.

Para isso aloque IPs fora da declaração dos módulos de VPC.
```hcl
resource "aws_eip" "nat" {
  count = 3

  vpc = true
}
```

E passe os IPs reservados como parametro para esse módulo.
```hcl
module "vpc" {
  source = "terraform-aws-modules/vpc/aws"

  # The rest of arguments are omitted for brevity

  enable_nat_gateway  = true
  single_nat_gateway  = false
  reuse_nat_ips       = true                      # <= Skip creation of EIPs for the NAT Gateways
  external_nat_ip_ids = ["${aws_eip.nat.*.id}"]   # <= IPs specified here as input to the module
}
```

Observe que neste exemplo nós alocamos 3 IPs porque nós vamos provisionar 3 NAT Gateways (devido a `single_nat_gateway = false` e ter 3 subnets)
Se por outro lado `single_nat_gateway = true` então `aws_eip.nat` precisa só alocar 1 IP.
Passando os IPs para o módulo basta configurar duas variáveis `reuse_nat_ips = true` e `external_nat_ip_ids = ["${aws_eip.nat.*.id}"]`

## Cenários de uso do NAT Gateway

Este modulo suporta 3 modos para criar NAT Gateway. Cada um será explicado detalhadamente na sua sessão correspondente.

* Um NAT Gateway por subnet é a melhor opção e a default se não for redefinido.
    * `enable_nat_gateway = true`
    * `single_nat_gateway = false`
    * `one_nat_gateway_per_az = false`
* Apenas 1 NAT Gateway
    * `enable_nat_gateway = true`
    * `single_nat_gateway = true`
    * `one_nat_gateway_per_az = false`
* Apenas 1 NAT Gateway por AZ
    * `enable_nat_gateway = true`
    * `single_nat_gateway = false`
    * `one_nat_gateway_per_az = true`

Se `single_nat_gateway` e `one_nat_gateway_per_az` forem definidos como `true`, o `single_nat_gateway` prevalece.

### Um NAT Gateway por subnet (default)

Por padrão, o modulo irá determinar o numero de NAT Gateways que serão criados baseado no `max()` de itens da lista de subnet privada (`database_subnets`, `elasticache_subnets`, `private_subnets` e `redshift_subnets`). O modulo **não** considera o o número de `intra_subnets` pois esses são definidos para não ter acesso a Internet via NAT Gateway.
Por exemplo, se sua configuração for parecida com exemplo abaixo:

```hcl
database_subnets    = ["10.0.21.0/24", "10.0.22.0/24"]
elasticache_subnets = ["10.0.31.0/24", "10.0.32.0/24"]
private_subnets     = ["10.0.1.0/24", "10.0.2.0/24", "10.0.3.0/24", "10.0.4.0/24", "10.0.5.0/24"]
redshift_subnets    = ["10.0.41.0/24", "10.0.42.0/24"]
intra_subnets       = ["10.0.51.0/24", "10.0.52.0/24", "10.0.53.0/24"]
```

Então `5` NAT Gateways serão criados conforme os `5` CDIR das subnet privadas especificadas.

### Single NAT Gateway

Se `single_nat_gateway = true`, então todas as subnet privadas terão seu tráfego de Internet roteada por apenas um único NAT Gateway. O NAT Gateway será configurado em apenas uma única
subnet publica dos seus ranges de `public_subnets`.

### 1 NAT Gateway por AZ

Se `one_nat_gateway_per_az = true` e `single_nat_gateway = false`, então o modulo irá colocar 1 NAT Gateway em cada AZ especificada no `var.azs`. Existem alguns recursos relacionados a
essa flag que devem ser definidos:

* As variáveis `var.azs` **devem** ser especificadas.
* O número de CDIR block de subnet public no `public_subnets` **deve** ser maior ou igual ao número de AZ definidas no `var.azs`. Esta é a garantia que cada NAT Gateway tem uma subnet
publica para ser publicado.

## "private" vs "intra" subnets

Por padrão, se NAT Gateway estiver habilitado, as subnets privadas serão configuradas com rotas para o trafego de Internet que apontam para o NAT Gateway configurado usando as
opções acima.

Se você precisar de subnet privada que não devem ter rota para Internet (conforme [RFC1918 Category 1 subnets](https://tools.ietf.org/html/rfc1918)), você deve especificar em
`intra_subnet`. Um exemplo de caso de uso é a configuração de função Lambda em VPC onde os Lambdas precisam apenas acesso a recursos internos ou VPC endpoints para serviços da AWS.

Desde quando o Lambda pode alocar ENI proporcionalmente ao trafego recebido ([leia mais](https://docs.aws.amazon.com/lambda/latest/dg/vpc.html)), isso pode ser utilizado para
reservar uma grande quantidade de subnet privadas mantendo o o trafego interno apenas a VPC.


## Conditional creation

Algumas vezes você precisa ter uma forma de criar recursos da VPC mas não uma VPC, e não é permitido usar `count` dentro do bloco de `module`, então a solução é especificar o
argumento `create_vpc`.

```hcl
# This VPC will not be created
module "vpc" {
  source = "terraform-aws-modules/vpc/aws"

  create_vpc = false
  # ... omitted
}
```

## Acesso publico para instancias RDS

As vezes é necessário permitir o acesso publico para instancias RDS (isso não é recomendado para produção) especificando esses argumentos:

```hcl
  create_database_subnet_group           = true
  create_database_subnet_route_table     = true
  create_database_internet_gateway_route = true

  enable_dns_hostnames = true
  enable_dns_support   = true
```

## Versão Terraform

Versão terraform mínima requerida é 0.10.3 para que esse módulo funcione.


## Inputs

| Nome | Descrição | Tipo | Default | Obrigatório |
|------|-------------|:----:|:-----:|:-----:|
| amazon\_side\_asn | The Autonomous System Number (ASN) for the Amazon side of the gateway. By default the virtual private gateway is created with the current default Amazon ASN. | string | `"64512"` | no |
| assign\_generated\_ipv6\_cidr\_block | Requests an Amazon-provided IPv6 CIDR block with a /56 prefix length for the VPC. You cannot specify the range of IP addresses, or the size of the CIDR block | string | `"false"` | no |
| azs | A list of availability zones in the region | list | `[]` | no |
| cidr | The CIDR block for the VPC. Default value is a valid CIDR, but not acceptable by AWS and should be overridden | string | `"0.0.0.0/0"` | no |
| create\_database\_internet\_gateway\_route | Controls if an internet gateway route for public database access should be created | string | `"false"` | no |
| create\_database\_nat\_gateway\_route | Controls if a nat gateway route should be created to give internet access to the database subnets | string | `"false"` | no |
| create\_database\_subnet\_group | Controls if database subnet group should be created | string | `"true"` | no |
| create\_database\_subnet\_route\_table | Controls if separate route table for database should be created | string | `"false"` | no |
| create\_elasticache\_subnet\_group | Controls if elasticache subnet group should be created | string | `"true"` | no |
| create\_elasticache\_subnet\_route\_table | Controls if separate route table for elasticache should be created | string | `"false"` | no |
| create\_redshift\_subnet\_group | Controls if redshift subnet group should be created | string | `"true"` | no |
| create\_redshift\_subnet\_route\_table | Controls if separate route table for redshift should be created | string | `"false"` | no |
| create\_vpc | Controls if VPC should be created (it affects almost all resources) | string | `"true"` | no |
| database\_route\_table\_tags | Additional tags for the database route tables | map | `{}` | no |
| database\_subnet\_group\_tags | Additional tags for the database subnet group | map | `{}` | no |
| database\_subnet\_suffix | Suffix to append to database subnets name | string | `"db"` | no |
| database\_subnet\_tags | Additional tags for the database subnets | map | `{}` | no |
| database\_subnets | A list of database subnets | list | `[]` | no |
| default\_vpc\_enable\_classiclink | Should be true to enable ClassicLink in the Default VPC | string | `"false"` | no |
| default\_vpc\_enable\_dns\_hostnames | Should be true to enable DNS hostnames in the Default VPC | string | `"false"` | no |
| default\_vpc\_enable\_dns\_support | Should be true to enable DNS support in the Default VPC | string | `"true"` | no |
| default\_vpc\_name | Name to be used on the Default VPC | string | `""` | no |
| default\_vpc\_tags | Additional tags for the Default VPC | map | `{}` | no |
| dhcp\_options\_domain\_name | Specifies DNS name for DHCP options set | string | `""` | no |
| dhcp\_options\_domain\_name\_servers | Specify a list of DNS server addresses for DHCP options set, default to AWS provided | list | `[ "AmazonProvidedDNS" ]` | no |
| dhcp\_options\_netbios\_name\_servers | Specify a list of netbios servers for DHCP options set | list | `[]` | no |
| dhcp\_options\_netbios\_node\_type | Specify netbios node_type for DHCP options set | string | `""` | no |
| dhcp\_options\_ntp\_servers | Specify a list of NTP servers for DHCP options set | list | `[]` | no |
| dhcp\_options\_tags | Additional tags for the DHCP option set | map | `{}` | no |
| ec2\_endpoint\_private\_dns\_enabled | Whether or not to associate a private hosted zone with the specified VPC for EC2 endpoint | string | `"false"` | no |
| ec2\_endpoint\_security\_group\_ids | The ID of one or more security groups to associate with the network interface for EC2 endpoint | list | `[]` | no |
| ec2\_endpoint\_subnet\_ids | The ID of one or more subnets in which to create a network interface for EC2 endpoint. Only a single subnet within an AZ is supported. If omitted, private subnets will be used. | list | `[]` | no |
| elasticache\_route\_table\_tags | Additional tags for the elasticache route tables | map | `{}` | no |
| elasticache\_subnet\_suffix | Suffix to append to elasticache subnets name | string | `"elasticache"` | no |
| elasticache\_subnet\_tags | Additional tags for the elasticache subnets | map | `{}` | no |
| elasticache\_subnets | A list of elasticache subnets | list | `[]` | no |
| enable\_dhcp\_options | Should be true if you want to specify a DHCP options set with a custom domain name, DNS servers, NTP servers, netbios servers, and/or netbios server type | string | `"false"` | no |
| enable\_dns\_hostnames | Should be true to enable DNS hostnames in the VPC | string | `"false"` | no |
| enable\_dns\_support | Should be true to enable DNS support in the VPC | string | `"true"` | no |
| enable\_dynamodb\_endpoint | Should be true if you want to provision a DynamoDB endpoint to the VPC | string | `"false"` | no |
| enable\_ec2\_endpoint | Should be true if you want to provision an EC2 endpoint to the VPC | string | `"false"` | no |
| enable\_nat\_gateway | Should be true if you want to provision NAT Gateways for each of your private networks | string | `"false"` | no |
| enable\_s3\_endpoint | Should be true if you want to provision an S3 endpoint to the VPC | string | `"false"` | no |
| enable\_ssm\_endpoint | Should be true if you want to provision an SSM endpoint to the VPC | string | `"false"` | no |
| enable\_vpn\_gateway | Should be true if you want to create a new VPN Gateway resource and attach it to the VPC | string | `"false"` | no |
| external\_nat\_ip\_ids | List of EIP IDs to be assigned to the NAT Gateways (used in combination with reuse_nat_ips) | list | `[]` | no |
| igw\_tags | Additional tags for the internet gateway | map | `{}` | no |
| instance\_tenancy | A tenancy option for instances launched into the VPC | string | `"default"` | no |
| intra\_route\_table\_tags | Additional tags for the intra route tables | map | `{}` | no |
| intra\_subnet\_tags | Additional tags for the intra subnets | map | `{}` | no |
| intra\_subnets | A list of intra subnets | list | `[]` | no |
| manage\_default\_vpc | Should be true to adopt and manage Default VPC | string | `"false"` | no |
| map\_public\_ip\_on\_launch | Should be false if you do not want to auto-assign public IP on launch | string | `"true"` | no |
| name | Name to be used on all the resources as identifier | string | `""` | no |
| nat\_eip\_tags | Additional tags for the NAT EIP | map | `{}` | no |
| nat\_gateway\_tags | Additional tags for the NAT gateways | map | `{}` | no |
| one\_nat\_gateway\_per\_az | Should be true if you want only one NAT Gateway per availability zone. Requires `var.azs` to be set, and the number of `public_subnets` created to be greater than or equal to the number of availability zones specified in `var.azs`. | string | `"false"` | no |
| private\_route\_table\_tags | Additional tags for the private route tables | map | `{}` | no |
| private\_subnet\_suffix | Suffix to append to private subnets name | string | `"private"` | no |
| private\_subnet\_tags | Additional tags for the private subnets | map | `{}` | no |
| private\_subnets | A list of private subnets inside the VPC | list | `[]` | no |
| propagate\_private\_route\_tables\_vgw | Should be true if you want route table propagation | string | `"false"` | no |
| propagate\_public\_route\_tables\_vgw | Should be true if you want route table propagation | string | `"false"` | no |
| public\_route\_table\_tags | Additional tags for the public route tables | map | `{}` | no |
| public\_subnet\_suffix | Suffix to append to public subnets name | string | `"public"` | no |
| public\_subnet\_tags | Additional tags for the public subnets | map | `{}` | no |
| public\_subnets | A list of public subnets inside the VPC | list | `[]` | no |
| redshift\_route\_table\_tags | Additional tags for the redshift route tables | map | `{}` | no |
| redshift\_subnet\_group\_tags | Additional tags for the redshift subnet group | map | `{}` | no |
| redshift\_subnet\_suffix | Suffix to append to redshift subnets name | string | `"redshift"` | no |
| redshift\_subnet\_tags | Additional tags for the redshift subnets | map | `{}` | no |
| redshift\_subnets | A list of redshift subnets | list | `[]` | no |
| reuse\_nat\_ips | Should be true if you don't want EIPs to be created for your NAT Gateways and will instead pass them in via the 'external_nat_ip_ids' variable | string | `"false"` | no |
| secondary\_cidr\_blocks | List of secondary CIDR blocks to associate with the VPC to extend the IP Address pool | list | `[]` | no |
| single\_nat\_gateway | Should be true if you want to provision a single shared NAT Gateway across all of your private networks | string | `"false"` | no |
| ssm\_endpoint\_private\_dns\_enabled | Whether or not to associate a private hosted zone with the specified VPC for SSM endpoint | string | `"false"` | no |
| ssm\_endpoint\_security\_group\_ids | The ID of one or more security groups to associate with the network interface for SSM endpoint | list | `[]` | no |
| ssm\_endpoint\_subnet\_ids | The ID of one or more subnets in which to create a network interface for SSM endpoint. Only a single subnet within an AZ is supported. If omitted, private subnets will be used. | list | `[]` | no |
| tags | A map of tags to add to all resources | map | `{}` | no |
| vpc\_tags | Additional tags for the VPC | map | `{}` | no |
| vpn\_gateway\_id | ID of VPN Gateway to attach to the VPC | string | `""` | no |
| vpn\_gateway\_tags | Additional tags for the VPN gateway | map | `{}` | no |
