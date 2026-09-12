# AWS Troubleshooting Challenge

[![CI](https://github.com/Julionores/aws-troubleshooting-challenge/actions/workflows/ci.yml/badge.svg)](https://github.com/Julionores/aws-troubleshooting-challenge/actions/workflows/ci.yml)

Une application web (VPC + Application Load Balancer + EC2) **volontairement cassée** de
trois façons distinctes, avec la démarche de diagnostic complète pour identifier et corriger
chaque cause racine -- comme un vrai incident de production.

> Projet réalisé par **Junior Tsafack Megnekeu** ([blog.jtmcloud.com](https://blog.jtmcloud.com) ·
> [GitHub](https://github.com/Julionores) ·
> [LinkedIn](https://www.linkedin.com/in/junior-tsafack-megnekeu-b673151b9)) — pièce d'un
> portfolio technique orienté Cloud/DevOps. Voir aussi
> [`dynamodb-streams-cdc-pipeline`](https://github.com/Julionores/dynamodb-streams-cdc-pipeline),
> [`s3-cross-region-replication`](https://github.com/Julionores/s3-cross-region-replication),
> [`aws-alb-deployment-patterns`](https://github.com/Julionores/aws-alb-deployment-patterns) et
> [`aws-vpc-connectivity-patterns`](https://github.com/Julionores/aws-vpc-connectivity-patterns).
> Côté DevSecOps/Full Stack, voir aussi
> [`devsecops-pipeline-reference`](https://github.com/Julionores/devsecops-pipeline-reference),
> [`securebank-api`](https://github.com/Julionores/securebank-api),
> [`postgresql-ha-repmgr`](https://github.com/Julionores/postgresql-ha-repmgr) et
> [`iso27001-isms-toolkit`](https://github.com/Julionores/iso27001-isms-toolkit).
> Côté Machine Learning, voir aussi [`gradientforge`](https://github.com/Julionores/gradientforge),
> [`radar-risque-impaye`](https://github.com/Julionores/radar-risque-impaye),
> [`collecte-agricole-planner`](https://github.com/Julionores/collecte-agricole-planner) et
> [`ticket-tide`](https://github.com/Julionores/ticket-tide), une prévision de série
> temporelle (famille ARMA).

## Pourquoi ce format

La plupart des projets de portfolio montrent « voici un système qui fonctionne ». Celui-ci
montre autre chose, tout aussi utile pour un poste Solutions Architect / Cloud / DevOps :
**la capacité à diagnostiquer méthodiquement une infrastructure qui ne fonctionne pas**, en
partant des symptômes observables (curl, health checks) pour remonter jusqu'aux causes
racines, sans deviner.

## Le défi

`cloudformation/template-broken.yaml` déploie un VPC, un Application Load Balancer et une
instance EC2 exécutant un serveur web. Une fois déployé, **le site est inaccessible**. Le
défi : diagnostiquer pourquoi, sans regarder tout de suite `template-fixed.yaml`.

## La démarche de diagnostic (réelle, testée sur AWS)

📄 **[docs/diagnostic.md](docs/diagnostic.md)** reproduit, avec les sorties de commandes
réelles obtenues lors d'un déploiement effectif sur AWS (région `eu-west-1`) :

1. Le symptôme observé (`curl` : timeout de connexion).
2. L'inspection de l'état de santé de la cible (`describe-target-health`).
3. L'identification des 3 causes racines (2 security groups sans règle d'entrée + un chemin
   de health check erroné).
4. Les correctifs appliqués, avec le diff exact entre les deux templates.
5. La vérification après correctif : `describe-target-health` repasse à `healthy`, et
   `curl` obtient une réponse `200 OK`.
6. Un bug annexe rencontré (et documenté) *pendant* la correction elle-même : une contrainte
   peu connue d'AWS sur le jeu de caractères autorisé dans la description d'une règle de
   security group.

## Structure du dépôt

```
cloudformation/
├── template-broken.yaml   # Version cassée (3 bugs volontaires, commentés dans le template)
└── template-fixed.yaml    # Version corrigée
docs/diagnostic.md         # Démarche de diagnostic complète, avec preuves réelles
```

## Reproduire le défi vous-même

```bash
aws cloudformation deploy \
  --template-file cloudformation/template-broken.yaml \
  --stack-name troubleshooting-challenge \
  --region eu-west-1

aws cloudformation describe-stacks --stack-name troubleshooting-challenge \
  --query "Stacks[0].Outputs"
# Tentez de charger l'URL obtenue -- ça ne fonctionnera pas. À vous de jouer.
```

Une fois votre diagnostic posé, comparez avec [docs/diagnostic.md](docs/diagnostic.md) et
[`template-fixed.yaml`](cloudformation/template-fixed.yaml).

Nettoyage :

```bash
aws cloudformation delete-stack --stack-name troubleshooting-challenge
```

## Validation

```bash
pip install cfn-lint
cfn-lint cloudformation/template-broken.yaml
cfn-lint cloudformation/template-fixed.yaml
```

Les deux templates sont syntaxiquement valides (le template cassé l'est délibérément *du
point de vue CloudFormation* -- ses bugs sont fonctionnels, pas syntaxiques).

## Limites assumées

- Une seule instance EC2 statique (pas d'Auto Scaling Group) : suffisant pour isoler les 3
  bugs cibles, mais pas représentatif d'une architecture haute disponibilité réelle (voir
  [`aws-alb-deployment-patterns`](https://github.com/Julionores/aws-alb-deployment-patterns)
  pour un exemple avec ASG).
- Les NACL du VPC sont volontairement permissives (autorisent tout le trafic) : elles ne
  font pas partie des bugs à trouver, uniquement des security groups.

## Licence

MIT — voir [`LICENSE`](LICENSE). Projet à but pédagogique et de démonstration.
