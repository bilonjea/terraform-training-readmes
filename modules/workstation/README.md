# Module Workstation

Module Terraform responsable de la création d'un workstation AWS de
secours pour les sessions de formation Terraform.

Le workstation permet aux étudiants de travailler depuis un
environnement AWS préconfiguré lorsque leurs ordinateurs locaux ne sont
pas disponibles ou ne disposent pas de la configuration nécessaire.

## Activation

Le module est optionnel.

Depuis le projet racine :

``` hcl
create_workstation = true
```

crée le workstation.

``` hcl
create_workstation = false
```

n'en crée pas.

## Entrées

### `instance_type`

Type d'instance EC2 utilisé pour le workstation.

Exemple :

``` hcl
instance_type = "t3.medium"
```

### `student_count`

Nombre d'étudiants utilisant le workstation.

Le module crée les comptes Linux correspondants :

``` text
student01
student02
...
```

### `student_credentials`

Credentials produits par le module IAM.

Le module reçoit directement :

``` hcl
module.iam.student_aws_credentials
```

Aucun fichier JSON intermédiaire n'est utilisé.

### `git_repositories`

Liste des dépôts Git à cloner dans les environnements étudiants.

### `aws_account_id`

Identifiant du compte AWS utilisé pour la formation.

### `allowed_aws_regions`

Liste des régions AWS autorisées pour les étudiants. Cette information
est également utilisée dans la configuration fournie aux étudiants.

## Architecture réseau

Le workstation est déployé dans un VPC avec un subnet public.

Le subnet n'attribue pas automatiquement de Public IPv4 :

``` hcl
map_public_ip_on_launch = false
```

L'instance ne demande pas non plus de Public IPv4 automatique :

``` hcl
associate_public_ip_address = false
```

L'accès public du workstation utilise explicitement une Elastic IP.

``` text
Internet
   │
   ▼
Elastic IP
   │
   ▼
EC2 Workstation
   │
   ▼
ENI / IP privée
   │
   ▼
Subnet
```

Le choix de l'Availability Zone est déterministe afin d'éviter des
changements permanents de plan.

## Configuration du workstation

Le module configure notamment :

-   Ubuntu ;
-   Terraform ;
-   AWS CLI ;
-   code-server ;
-   NGINX ;
-   Certbot ;
-   les comptes Linux étudiants ;
-   les credentials AWS étudiants ;
-   les dépôts Git de formation.

Les mots de passe injectés dans le `user_data` sont encodés en Base64
avant d'être transmis au script afin d'éviter que des caractères
spéciaux des secrets soient interprétés par Bash.

## Elastic IP

Le workstation reçoit une Elastic IP dédiée :

``` hcl
resource "aws_eip" "workstation" {
  ...
}
```

Puis elle est associée à l'instance :

``` hcl
resource "aws_eip_association" "workstation" {
  ...
}
```

## Outputs

Le module expose notamment :

``` text
instance_id
public_ip
public_dns
workstation_public_ip
student_password
workstation_availability_zone
```

Les outputs sensibles sont marqués `sensitive`.

Le projet racine peut choisir lesquels exposer à l'utilisateur final.

## Stabilité Terraform

L'attribut :

``` hcl
associate_public_ip_address = false
```

est volontaire.

Une règle `lifecycle.ignore_changes` est utilisée pour cet attribut afin
d'éviter un faux drift lié au comportement du provider AWS lors de
l'association de l'Elastic IP.

## Utilisation depuis le root

``` hcl
module "workstation" {
  count = var.create_workstation ? 1 : 0

  source = "./modules/workstation"

  instance_type       = var.instance_type
  student_count       = var.student_count
  student_credentials = module.iam.student_aws_credentials
  git_repositories    = var.git_repositories
  aws_account_id      = var.aws_account_id
  allowed_aws_regions = var.allowed_aws_regions
}
```

Lorsque `create_workstation = false`, aucune ressource de ce module
n'est créée.

## Destruction

Le workstation est conçu pour être temporaire.

Pour supprimer l'environnement :

``` bash
terraform destroy
```
