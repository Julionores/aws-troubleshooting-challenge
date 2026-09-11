# Démarche de diagnostic

Ce document reproduit la démarche réelle suivie pour diagnostiquer et corriger
`cloudformation/template-broken.yaml`, avec les sorties de commandes obtenues lors du test
réel sur AWS (région `eu-west-1`).

## 1. Symptôme observé

Après déploiement, la sortie `LoadBalancerDNSName` de la stack pointe vers un load
balancer... qui ne répond pas :

```bash
$ curl -sv --max-time 10 http://ChallengeElb-1465618653.eu-west-1.elb.amazonaws.com/
*   Trying 52.211.224.24:80...
* Connection timed out after 10000 milliseconds
* Closing connection
```

Un **timeout de connexion** (et non un refus de connexion, ni une erreur HTTP) est un premier
indice important : la requête TCP elle-même n'aboutit pas. Cela oriente d'emblée le
diagnostic vers la couche réseau (security groups, NACL) plutôt que vers l'application.

## 2. Étape 1 -- l'état de la cible dans le Target Group

```bash
$ aws elbv2 describe-target-health --target-group-arn <arn> --region eu-west-1
```

```json
{
  "Target": { "Id": "i-0cad87040d98ed797", "Port": 80 },
  "TargetHealth": {
    "State": "unhealthy",
    "Reason": "Target.Timeout",
    "Description": "Request timed out"
  }
}
```

Le champ `Reason: Target.Timeout` confirme l'hypothèse de l'étape 1 : ce n'est pas un
code HTTP d'erreur (ce qui pointerait vers l'application ou une mauvaise configuration
du health check), mais une absence totale de réponse -- symptomatique d'un blocage réseau.

## 3. Étape 2 -- inspection des security groups

```bash
$ aws ec2 describe-security-groups --filters "Name=group-name,Values=*ChallengeSGELB*"
{ "GroupId": "sg-0e8bbc6d47a465b76", "IngressRules": [] }

$ aws ec2 describe-security-groups --filters "Name=group-name,Values=*ChallengeSGapp*"
{ "GroupId": "sg-0c8e6b8bb84043add", "IngressRules": [] }
```

**Cause racine n°1 et n°2 confirmées** : ni le security group de l'ALB (`ChallengeSGELB`),
ni celui de l'instance applicative (`ChallengeSGapp`), n'ont la moindre règle d'entrée.
Aucun trafic ne peut atteindre l'ALB depuis Internet, et même s'il le pouvait, l'ALB ne
pourrait pas relayer le trafic vers l'instance.

## 4. Étape 3 -- configuration du health check

```bash
$ aws elbv2 describe-target-groups --target-group-arns <arn> \
    --query "TargetGroups[0].HealthCheckPath"
"/toto.html"
```

**Cause racine n°3 confirmée** : le chemin de health check (`/toto.html`) ne correspond à
aucune page réellement servie par l'instance (qui sert `/index.html` à la racine `/`).
Ce bug est masqué par les deux premiers : tant que le trafic réseau est bloqué, le health
check échoue de toute façon par timeout, avant même d'atteindre l'application pour recevoir
une erreur 404. **Il faut corriger les trois causes pour restaurer le service**, pas
seulement la plus visible.

## 5. Correctifs appliqués

| # | Ressource | Avant | Après |
|---|---|---|---|
| 1 | `ChallengeSGELB` | Aucune règle d'entrée | Autorise `tcp/80` depuis `0.0.0.0/0` |
| 2 | `ChallengeSGapp` | Aucune règle d'entrée | Autorise `tcp/80` **uniquement** depuis `ChallengeSGELB` (pas Internet direct) |
| 3 | `ChallengeTargetGroup` | `HealthCheckPath: /toto.html` | `HealthCheckPath: /` |

Voir le diff complet entre
[`template-broken.yaml`](../cloudformation/template-broken.yaml) et
[`template-fixed.yaml`](../cloudformation/template-fixed.yaml).

> 💡 **Un bug rencontré en corrigeant le bug** : la première tentative de correction a
> échoué au déploiement avec l'erreur AWS `Invalid rule description` -- la description
> d'une règle de security group n'accepte **pas l'apostrophe** parmi ses caractères
> valides (`a-zA-Z0-9. _-:/()#,@[]+=&;{}!$*`). Une description comme *"Trafic HTTP en
> provenance de l'ALB uniquement"* est donc rejetée par l'API EC2. Détail à connaître
> avant d'écrire des descriptions de règles en français.

## 6. Vérification après correctif

```bash
$ aws elbv2 describe-target-health --target-group-arn <arn> \
    --query "TargetHealthDescriptions[0].TargetHealth.State"
"healthy"

$ curl -sv --max-time 10 http://ChallengeElb-1465618653.eu-west-1.elb.amazonaws.com/
< HTTP/1.1 200 OK
< Server: Apache/2.4.68 (Amazon Linux)
<
<html><body><h1>Le serveur applicatif repond correctement.</h1></body></html>
```

La cible passe `healthy` dès le premier contrôle après déploiement du correctif, et le
load balancer répond `200 OK` avec le contenu attendu.

## Méthode générale à retenir

Face à une application web inaccessible derrière un ALB, l'ordre de diagnostic efficace est :

1. **`curl` direct** sur le DNS de l'ALB -- distinguer un timeout (réseau) d'un refus de
   connexion (rien n'écoute) ou d'une erreur HTTP (l'application répond, mais mal).
2. **`describe-target-health`** -- le champ `Reason` indique précisément la nature du
   problème (`Target.Timeout`, `Target.ResponseCodeMismatch`, `Target.FailedHealthChecks`...).
3. **Security groups** de l'ALB puis de la cible, dans cet ordre (le trafic les traverse
   dans cet ordre).
4. **Configuration du health check** (chemin, port, codes de succès attendus) -- seulement
   une fois le réseau écarté, pour éviter de corriger un symptôme masqué par un problème
   plus fondamental.
