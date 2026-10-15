# Centreon + InfluxDB + n8n — Automatisation des incidents avec IA locale

Workflow n8n pour automatiser l'analyse et le suivi des incidents Centreon. Les métriques historiques sont interrogées dans **InfluxDB**, complétées par **Prometheus** (système, processus et conteneurs) et **Loki** (logs).

Un modèle d'IA local peut corréler ces informations et proposer un diagnostic. L'objectif est de **séparer la supervision de l'analyse**, sans multiplier les appels à l'API Centreon pour consulter des historiques de métriques.

## Architecture

```text
Centreon (alerte + output)
          |
          v
         n8n
          |
          +-- Identification de la ressource concernée
          |
          +-- InfluxDB   : historique des métriques
          +-- Prometheus : métriques système / processus / conteneurs
          +-- Loki       : journaux
          |
          v
     Gemma 4 local
          |
          v
         GLPI (ticket et diagnostic)
          |
          v
 Centreon (acquittement et lien GLPI, si incident encore actif)
```

## Stack utilisée

| Outil | Rôle |
|---|---|
| Centreon | Supervision, alertes et acquittements |
| InfluxDB | Historique des métriques |
| Prometheus | Métriques système, processus et conteneurs |
| Loki | Centralisation des logs |
| n8n | Orchestration |
| Gemma 4 26B A4B | Diagnostic IA local |
| LM Studio | Exécution du modèle local |
| GLPI | Tickets et suivi des incidents |
| Qdrant | Mémoire vectorielle, selon les branches configurées |

## Fonctionnement

Lorsqu'une notification Centreon arrive dans n8n :

1. Le workflow lit l'hôte, le service, l'état et le message `output`.
2. Il récupère les instances connues dans InfluxDB et cherche une correspondance **exacte** avec le message d'alerte.
3. Si une seule instance correspond, il cible ses métriques et ses logs, tout en conservant un contexte hôte. Sinon, il utilise la collecte générale de l'hôte.
4. Il rassemble l'historique InfluxDB, les métriques Prometheus et les logs Loki.
5. Le LLM local propose un diagnostic fondé sur les données disponibles.
6. Un ticket GLPI est créé avec le diagnostic.
7. Si le service Centreon est toujours en incident et non acquitté, le workflow l'acquitte en ajoutant le lien du ticket GLPI en commentaire.

**Aucune remédiation technique automatique n'est exécutée.**

## Scénarios testés

### Saturation CPU Linux

Une charge créée avec `stress-ng` déclenche une alerte Centreon. InfluxDB confirme le pic CPU, Prometheus identifie le processus consommateur et Loki fournit la trace de la commande exécutée.

### Arrêt d'un conteneur Docker

Le message Centreon signale l'arrêt du conteneur `splunk`. Le workflow retrouve dynamiquement son nom parmi les instances InfluxDB et interroge les métriques et logs correspondants. Les éléments collectés permettent un diagnostic, **sans garantir à eux seuls l'identification de la cause racine**.

Ces noms servent uniquement à décrire les tests du lab ; ils ne sont pas codés en dur dans les règles de sélection.

## Prérequis

- n8n 2.x (testé avec **2.42.3**).
- Centreon avec notifications webhook et API REST (testé avec **25.10.16**).
- InfluxDB recevant les métriques Centreon.
- Prometheus et les exporters nécessaires.
- Loki avec des labels utilisables pour cibler les logs.
- GLPI avec API REST activée.
- Un endpoint compatible avec l'API OpenAI pour le LLM local (LM Studio dans le lab).
- Qdrant et un service d'embeddings pour les branches de mémoire, si utilisées.

## Import et configuration

1. Importer le fichier JSON nettoyé dans n8n.
2. Remplacer les **URL d'exemple** par celles de votre environnement.
3. Configurer les **credentials n8n** et régénérer/renseigner les tokens nécessaires (Centreon, InfluxDB, GLPI, etc.).
4. Adapter le bucket et l'organisation InfluxDB, ainsi que les noms de labels utilisés dans InfluxDB, Prometheus et Loki.
5. Configurer le modèle local et les éventuels services Qdrant/embeddings.
6. Paramétrer la notification Centreon pour transmettre le champ `output` au webhook.
7. Tester les nœuds un par un avant d'activer le workflow.

**Important :** le JSON publié est une **copie expurgée**. Il ne fonctionnera pas sans configuration de vos endpoints et authentifications.

## Sécurité

- Ne publiez jamais de tokens API, secrets ou exports d'exécution contenant des données sensibles.
- Utilisez les **credentials n8n**, plutôt que des tokens en clair dans les en-têtes des nœuds.
- Utilisez HTTPS et limitez les droits des comptes techniques en environnement de production.
- Vérifiez les URL, IP, identifiants et données des tickets avant toute publication.
- Révoquez et renouvelez tout token ayant été exposé dans un ancien export.

## Limites et points à adapter

- L'identification automatique dépend de la présence du nom de ressource dans le `output` Centreon **et** dans les labels InfluxDB. Une absence de correspondance entraîne un repli sur l'hôte.
- Les labels et noms d'instances peuvent différer entre InfluxDB, Prometheus et Loki : adaptez les sélecteurs à votre environnement.
- La branche historique de clôture GLPI repose encore sur le champ Zabbix `event_id` et nécessite une adaptation pour les notifications Centreon.
- Le prompt système fourni contient une ancienne consigne sur l'absence d'acquittement automatique ; harmonisez-la avec le workflow si vous utilisez cette fonctionnalité.
- L'acquittement automatique ne constitue **pas** une résolution de l'incident.
- Les diagnostics du modèle sont des aides à l'analyse ; les causes non démontrées doivent rester des hypothèses.

## Réutilisation

Cette architecture peut être adaptée à d'autres outils de supervision en normalisant leurs alertes en entrée et en adaptant les actions finales. **L'IA est optionnelle dans une architecture de ce type** : collecte, enrichissement, reporting et création de tickets peuvent aussi être automatisés sans LLM.

Projet développé et testé en **lab** : adaptez les contrôles, permissions et règles de traitement avant tout usage en production.
