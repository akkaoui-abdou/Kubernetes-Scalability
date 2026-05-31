# Kubernetes Scalabilité : Verticale et Horizontale

## Introduction

La scalabilité (*scaling*) dans Kubernetes permet d'adapter automatiquement ou manuellement les ressources d'une application en fonction de la charge.

Il existe deux principaux types de scalabilité :

- **Scalabilité verticale (Vertical Scaling)** : augmentation des ressources d'un Pod existant.
- **Scalabilité horizontale (Horizontal Scaling)** : augmentation du nombre de Pods.

---

# 1. Scalabilité Verticale (Vertical Scaling)

## Définition

La scalabilité verticale consiste à augmenter les ressources CPU et/ou mémoire d'un Pod existant.

### Exemple

Avant :

```text
Pod
├── CPU : 500m
└── RAM : 1Gi
```

Après :

```text
Pod
├── CPU : 2
└── RAM : 4Gi
```

## Avantages

- Facile à mettre en place.
- Ne nécessite pas de modification de l'application.
- Adapté aux applications monolithiques.

## Inconvénients

- Limité par la capacité maximale du nœud.
- Peut nécessiter un redémarrage du Pod.
- Point unique de défaillance.

---

## Exemple Deployment pour VPA

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-vpa
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nginx-vpa
  template:
    metadata:
      labels:
        app: nginx-vpa
    spec:
      containers:
      - name: nginx
        image: nginx:latest
        resources:
          requests:
            cpu: 100m
            memory: 128Mi
```

## Configuration Vertical Pod Autoscaler (VPA)

```yaml
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: nginx-vpa
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: nginx-vpa

  updatePolicy:
    updateMode: Auto
```

### Modes VPA

| Mode | Description |
|--------|-------------|
| Off | Recommandations uniquement |
| Initial | Applique uniquement à la création du Pod |
| Auto | Mise à jour automatique |
| Recreate | Redémarre les Pods pour appliquer les changements |

---

## Vérification VPA

```bash
kubectl get vpa
kubectl describe vpa nginx-vpa
```

---

# 2. Scalabilité Horizontale (Horizontal Scaling)

## Définition

La scalabilité horizontale consiste à ajouter ou supprimer des Pods en fonction de la charge.

### Exemple

Avant :

```text
[Pod1] [Pod2]
```

Après :

```text
[Pod1] [Pod2] [Pod3] [Pod4] [Pod5]
```

## Avantages

- Très haute disponibilité.
- Tolérance aux pannes.
- Adaptée aux architectures cloud-native.
- Quasiment sans limite de croissance.

## Inconvénients

- Nécessite des applications stateless ou distribuées.
- Gestion plus complexe des sessions et du stockage.

---

## Deployment pour HPA

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-app

spec:
  replicas: 2

  selector:
    matchLabels:
      app: nginx-app

  template:
    metadata:
      labels:
        app: nginx-app

    spec:
      containers:
      - name: nginx
        image: nginx:latest

        resources:
          requests:
            cpu: 200m
            memory: 128Mi

          limits:
            cpu: 500m
            memory: 256Mi
```

---

## Configuration Horizontal Pod Autoscaler (HPA)

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler

metadata:
  name: nginx-app-hpa

spec:

  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: nginx-app

  minReplicas: 2
  maxReplicas: 10

  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
```

### Fonctionnement

```text
CPU > 70%
       │
       ▼
HPA augmente le nombre de Pods
       │
       ▼
CPU redescend
       │
       ▼
HPA réduit le nombre de Pods
```

---

## Vérification HPA

```bash
kubectl get hpa

kubectl describe hpa nginx-app-hpa
```

---

# 3. Cluster Autoscaler

## Définition

Le Cluster Autoscaler augmente ou réduit automatiquement le nombre de nœuds Kubernetes.

### Cas d'utilisation

Lorsque le HPA crée de nouveaux Pods mais qu'aucun nœud ne dispose des ressources nécessaires :

```text
HPA crée de nouveaux Pods
            │
            ▼
Pas assez de CPU/RAM
            │
            ▼
Cluster Autoscaler
            │
            ▼
Ajout d'un nouveau Node
```

---

## Architecture complète

```text
                    Utilisateurs
                          │
                          ▼
                   Load Balancer
                          │
                          ▼
                  Service Kubernetes
                          │
                          ▼

            ┌─────────────────────┐
            │ Deployment          │
            │ HPA                 │
            └─────────────────────┘
                     │
        ┌────────────┼────────────┐
        ▼            ▼            ▼

      Pod1         Pod2         Pod3

                     │
                     ▼

            Cluster Autoscaler

                     │

        ┌────────────┼────────────┐
        ▼            ▼            ▼

      Node1        Node2        Node3
```

---

# Comparaison VPA vs HPA

| Critère | VPA | HPA |
|----------|-----|-----|
| Action | Augmente CPU/RAM | Ajoute des Pods |
| Objet impacté | Pod existant | Nombre de Pods |
| Disponibilité | Moyenne | Élevée |
| Tolérance aux pannes | Faible | Forte |
| Limite | Taille du Node | Très élevée |
| Cas d'usage | Applications monolithiques | Microservices |

---

# Bonnes Pratiques

## Utiliser HPA lorsque :

- Application stateless.
- Architecture microservices.
- Forte variation de trafic.
- Besoin de haute disponibilité.

## Utiliser VPA lorsque :

- Application difficilement réplicable.
- Base de données.
- Services monolithiques.

## Production

La plupart des clusters Kubernetes utilisent :

```text
HPA
 +
Cluster Autoscaler
```

pour obtenir une scalabilité automatique complète.

---

# Commandes Utiles

```bash
# Vérifier les Pods
kubectl get pods

# Vérifier les métriques
kubectl top pods

# Vérifier les Nodes
kubectl top nodes

# Vérifier HPA
kubectl get hpa

# Vérifier VPA
kubectl get vpa

# Description HPA
kubectl describe hpa

# Description VPA
kubectl describe vpa
```

---

# Conclusion

Kubernetes propose deux mécanismes complémentaires :

- **Vertical Pod Autoscaler (VPA)** : augmente les ressources CPU/Mémoire d'un Pod.
- **Horizontal Pod Autoscaler (HPA)** : augmente ou réduit le nombre de Pods.

En environnement de production, la combinaison :

```text
HPA + Cluster Autoscaler
```

est généralement la stratégie la plus utilisée pour garantir performance, disponibilité et optimisation des coûts.



---

# 4. Topology Spread Constraints

## Définition

Les `topologySpreadConstraints` permettent de contrôler la répartition des Pods dans le cluster Kubernetes.

L'objectif est d'éviter que tous les Pods d'une même application soient placés sur :

- Le même Node
- La même Zone de disponibilité (Availability Zone)
- Le même Rack
- Ou toute autre topologie définie par un label

Cela améliore :

- La haute disponibilité (HA)
- La résilience aux pannes
- L'équilibrage de charge
- La tolérance aux défaillances d'infrastructure

---

## Pourquoi utiliser topologySpreadConstraints ?

### Sans topologySpreadConstraints

```text
Node-A
├── Pod-1
├── Pod-2
├── Pod-3
└── Pod-4

Node-B
└── Aucun Pod

Node-C
└── Aucun Pod
```

Si le Node-A tombe :

```text
❌ Tous les Pods sont perdus
❌ Service indisponible
```

---

### Avec topologySpreadConstraints

```text
Node-A
├── Pod-1
└── Pod-2

Node-B
├── Pod-3
└── Pod-4

Node-C
└── Pod-5
```

Si un Node tombe :

```text
✔ Les autres Pods continuent de servir le trafic
✔ Haute disponibilité
```

---

## Exemple de configuration

```yaml
apiVersion: apps/v1
kind: Deployment

metadata:
  name: nginx-ha

spec:
  replicas: 6

  selector:
    matchLabels:
      app: nginx-ha

  template:
    metadata:
      labels:
        app: nginx-ha

    spec:

      topologySpreadConstraints:

      - maxSkew: 1

        topologyKey: kubernetes.io/hostname

        whenUnsatisfiable: DoNotSchedule

        labelSelector:
          matchLabels:
            app: nginx-ha

      containers:
      - name: nginx
        image: nginx:latest
```

---

## Explication des paramètres

### maxSkew

Définit l'écart maximal autorisé entre les domaines de topologie.

```yaml
maxSkew: 1
```

Exemple valide :

```text
Node-A : 3 Pods
Node-B : 2 Pods
Node-C : 2 Pods
```

Écart maximal = 1

---

### topologyKey

Détermine le critère de répartition.

```yaml
topologyKey: kubernetes.io/hostname
```

Répartition par Node.

Autres valeurs fréquentes :

```yaml
topology.kubernetes.io/zone
```

Répartition par Zone AWS/Azure/GCP.

```yaml
topology.kubernetes.io/region
```

Répartition par Région.

---

### whenUnsatisfiable

Détermine le comportement lorsque la contrainte ne peut pas être respectée.

#### DoNotSchedule

```yaml
whenUnsatisfiable: DoNotSchedule
```

Le Pod ne sera pas créé tant qu'un placement équilibré n'est pas possible.

#### ScheduleAnyway

```yaml
whenUnsatisfiable: ScheduleAnyway
```

Le Scheduler tente d'équilibrer mais accepte un déséquilibre si nécessaire.

---

### labelSelector

Indique quels Pods doivent être pris en compte pour le calcul.

```yaml
labelSelector:
  matchLabels:
    app: nginx-ha
```

---

## Répartition par Zone de disponibilité

Exemple pour un cluster multi-zones :

```yaml
topologySpreadConstraints:

- maxSkew: 1

  topologyKey: topology.kubernetes.io/zone

  whenUnsatisfiable: DoNotSchedule

  labelSelector:
    matchLabels:
      app: nginx-ha
```

Résultat :

```text
Zone-A : 2 Pods
Zone-B : 2 Pods
Zone-C : 2 Pods
```

---

## Association avec HPA

Lorsque le HPA augmente le nombre de Pods :

```text
HPA
 │
 ▼
Création de nouveaux Pods
 │
 ▼
TopologySpreadConstraints
 │
 ▼
Répartition équilibrée des Pods
```

Exemple :

```text
Avant HPA

Node-A : 1 Pod
Node-B : 1 Pod

Après Scale Out

Node-A : 3 Pods
Node-B : 3 Pods
Node-C : 2 Pods
```

Le Scheduler respecte les contraintes définies.

---

## Différence avec Pod Anti-Affinity

### Pod Anti-Affinity

Empêche deux Pods similaires d'être placés ensemble.

```yaml
podAntiAffinity:
```

Exemple :

```text
Node-A : Pod-1
Node-B : Pod-2
Node-C : Pod-3
```

---

### Topology Spread Constraints

Cherche à répartir équitablement les Pods.

```yaml
topologySpreadConstraints:
```

Exemple :

```text
Node-A : 2 Pods
Node-B : 2 Pods
Node-C : 2 Pods
```

---

## Bonnes Pratiques

Pour une application critique :

```yaml
replicas: 6
```

Combiner :

- HPA
- Cluster Autoscaler
- topologySpreadConstraints
- PodDisruptionBudget

Exemple d'architecture :

```text
                 Ingress
                    │
                    ▼

              Deployment
                    │

                    ▼

                  HPA
                    │

                    ▼

     TopologySpreadConstraints

                    │

        ┌───────────┼───────────┐
        ▼           ▼           ▼

      Node-A      Node-B      Node-C
       2 Pods      2 Pods      2 Pods
```

---

## Conclusion

Les `topologySpreadConstraints` permettent de garantir une répartition équilibrée des Pods dans le cluster.

Ils complètent parfaitement :

- HPA (Horizontal Pod Autoscaler)
- Cluster Autoscaler
- Pod Anti-Affinity
- PodDisruptionBudget

et constituent aujourd'hui une bonne pratique pour toute application Kubernetes nécessitant une haute disponibilité.

