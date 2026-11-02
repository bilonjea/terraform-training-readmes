# Module IAM

Module Terraform responsable de la création des comptes AWS étudiants
pour la formation.

## Responsabilités

Le module :

-   crée les utilisateurs IAM étudiants ;
-   crée leurs access keys ;
-   crée leurs mots de passe de console ;
-   attache la policy permettant à chaque étudiant de changer son propre
    mot de passe ;
-   crée et attache la policy EC2 de la formation ;
-   limite l'utilisation EC2 aux régions autorisées ;
-   utilise le tag `Student` pour limiter la gestion des instances à
    celles appartenant à l'étudiant.

## Entrées

### `student_count`

Nombre de comptes étudiants à créer.

``` hcl
student_count = 10
```

Le module accepte entre 1 et 20 étudiants.

Les identifiants sont générés automatiquement :

``` text
student01
student02
...
student10
```

Les noms IAM correspondants sont :

``` text
tf-student01
tf-student02
...
tf-student10
```

### `allowed_aws_regions`

Liste des régions AWS dans lesquelles les étudiants peuvent utiliser les
ressources EC2.

Exemple :

``` hcl
allowed_aws_regions = [
  "eu-west-3",
  "eu-west-1",
  "eu-central-1",
  "us-east-1"
]
```

## Sorties

### `student_aws_credentials`

Map contenant, pour chaque étudiant :

``` text
username
console_password
access_key
secret_key
```

L'output est marqué comme `sensitive`.

Exemple d'utilisation depuis le root :

``` hcl
module.iam.student_aws_credentials
```

Le root transmet ensuite cette information au module workstation.

## Flux

``` text
student_count
      │
      ▼
   IAM module
      │
      ├── IAM users
      ├── Access keys
      ├── Console passwords
      └── EC2 policy
      │
      ▼
student_aws_credentials
```

## Utilisation directe du module

Le module peut être appelé depuis un autre projet Terraform :

``` hcl
module "iam" {
  source = "./modules/iam"

  student_count       = 10
  allowed_aws_regions = [
    "eu-west-3",
    "eu-west-1",
    "eu-central-1",
    "us-east-1"
  ]
}
```

Le provider AWS est fourni par le projet appelant.

## Sécurité

Les credentials sont des données sensibles et l'output est déclaré avec
:

``` hcl
sensitive = true
```

Dans ce projet de formation temporaire, ils sont utilisés directement
entre modules et ne sont pas exportés dans un fichier JSON
intermédiaire.
