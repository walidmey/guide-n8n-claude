# Déployer en production

{% hint style="info" %}
Mettre un workflow n8n en production, ce n'est pas seulement l'exécuter plus souvent. Il faut le rendre résilient aux erreurs, visible via le monitoring, maîtrisé en coûts et sûr côté données.
{% endhint %}

Un workflow qui "fonctionne" en test et un workflow qui "tourne en production" sont deux choses différentes. Cette section couvre ce qui change quand vous passez de l'un à l'autre.

{% hint style="warning" %}
Point critique : ne partez jamais en production sans gestion d'erreurs, retries et validation des sorties Claude. Un seul JSON invalide ou une API indisponible peut casser tout le workflow en silence.
{% endhint %}

***

## Ce qui casse en production (et pas en test)

**Les données réelles sont désordonnées.** En test, vous utilisez des exemples propres. En production, vous rencontrerez des emails sans corps de texte, des flux RSS avec des entrées malformées, des champs Airtable vides, des sites web qui ne répondent pas. Vos workflows doivent survivre à ces cas.

**Les volumes changent le comportement.** Un workflow qui traite 5 emails par jour est différent d'un workflow qui en traite 200. Les rate limits API, les timeouts, les coûts — tout ça devient visible à l'échelle.

**Les services externes tombent.** Gmail peut être indisponible. L'API Anthropic peut retourner une erreur 503. Un webhook externe peut ne pas répondre. Si votre workflow n'est pas conçu pour gérer ça, il échoue silencieusement.

***

## Gestion des erreurs : les bases

### Activer les retries automatiques

Pour chaque nœud critique (Claude, Gmail, Notion, Airtable), activez les retries dans les paramètres du nœud :

* **Retry on Fail :** activer
* **Max Tries :** 3
* **Wait Between Tries :** 5000ms (5 secondes)

Cela suffit à gérer la majorité des erreurs transitoires (timeout, rate limit temporaire).

### Capturer les erreurs avec un nœud Error Trigger

n8n permet de connecter un flux d'erreur à chaque workflow. Créez un sous-workflow dédié à la gestion des erreurs :

1. Créez un nouveau workflow "Gestion erreurs"
2. Ajoutez un nœud **Error Trigger**
3. Connectez-le à une notification (Gmail, Slack, ou autre)

Dans votre workflow principal, allez dans Settings → Error Workflow et sélectionnez ce workflow.

**Notification d'erreur type :**

```
Subject : ⚠️ Erreur workflow n8n — {{ $json.workflow.name }}

Workflow : {{ $json.workflow.name }}
Nœud en erreur : {{ $json.execution.error.node.name }}
Message d'erreur : {{ $json.execution.error.message }}
Timestamp : {{ new Date().toISOString() }}
```

### Valider les sorties de Claude

Avant d'utiliser la réponse de Claude dans la suite du workflow, validez qu'elle est utilisable :

```javascript
const text = $input.first().json.output[0].content[0].text;

// Si Claude devait renvoyer du JSON
try {
  const parsed = JSON.parse(text);
  // Vérifier que les champs attendus sont présents
  if (!parsed.categorie || !parsed.urgence) {
    throw new Error('Champs manquants dans la réponse Claude');
  }
  return [{ json: parsed }];
} catch (e) {
  // Valeur de fallback plutôt qu'une erreur fatale
  return [{ json: {
    categorie: 'autre',
    urgence: 'normale',
    erreur_parsing: true,
    texte_brut: text
  }}];
}
```

***

## Monitoring : savoir ce qui se passe

### Tableau de bord n8n

n8n inclut un historique d'exécution natif. Consultez-le régulièrement (ou configurez une notification hebdomadaire) :

* **Executions :** liste toutes les exécutions avec statut (succès / erreur)
* **Durée :** identifiez les workflows qui ralentissent
* **Erreurs :** filtrez par statut "Error" pour voir les échecs récents

### Log personnalisé dans Airtable ou Notion

Pour un suivi plus précis, ajoutez un nœud à la fin de chaque workflow qui journalise l'exécution :

**Airtable — Create Record (table "Logs") :**

* **Workflow :** `{{ $workflow.name }}`
* **Statut :** `Succès`
* **Durée :** `{{ $execution.resumedAt - $execution.startedAt }}ms`
* **Items traités :** `{{ $input.all().length }}`
* **Timestamp :** `{{ new Date().toISOString() }}`

***

## Contrôle des coûts

### Estimer le coût d'un workflow

Chaque appel Claude consomme des tokens. Le coût dépend du modèle utilisé et du volume de tokens in/out.

Pour estimer :

1. Testez le workflow 3 à 5 fois avec des données réelles
2. Notez le nombre de tokens consommés dans les métadonnées de réponse Claude
3. Multipliez par votre volume quotidien / mensuel estimé

**Règles pratiques :**

* `claude-haiku` : \~10× moins cher que Sonnet — utilisez-le pour la classification simple, le filtrage, les tâches de routage
* `claude-sonnet` : rapport qualité/coût optimal pour la rédaction, l'analyse
* `claude-opus` : réservez-le pour les tâches critiques nécessitant le meilleur niveau de raisonnement

### Réduire les coûts sans sacrifier la qualité

**Filtrer en amont.** Avant d'envoyer à Claude, filtrez les éléments qui ne nécessitent clairement pas d'analyse. Un email avec l'objet "Newsletter Unsubscribe" n'a pas besoin d'être analysé par Claude.

**Tronquer les inputs longs.** Si vous envoyez le contenu d'un email de 5000 mots à Claude pour en extraire 3 champs, passez à 500 mots : `$json.body.substring(0, 2000)`.

**Choisir le bon modèle par tâche.** Classification → Haiku. Rédaction → Sonnet. Analyse complexe multi-sources → Sonnet ou Opus.

***

## Sécurité

### Ne stockez pas les clés API en clair

n8n dispose d'un système de Credentials dédié au stockage sécurisé des clés API. N'utilisez jamais de clés en clair dans les nœuds Code ou dans les expressions.

✅ Correct : créer un Credential "Anthropic API" et le référencer dans le nœud ❌ Incorrect : écrire `sk-ant-...` directement dans un champ

### RGPD et données personnelles

Si vos workflows traitent des données personnelles (emails de clients, fiches contacts), posez-vous ces questions :

* Ces données sont-elles envoyées à l'API Anthropic ? → Oui, par défaut
* Cela est-il compatible avec votre politique de confidentialité et vos obligations RGPD ?
* Pouvez-vous minimiser les données transmises (objet uniquement, pas le corps de l'email) ?

Pour les données sensibles, envisagez une instance n8n auto-hébergée avec chiffrement des données au repos.

### Accès aux credentials

Si plusieurs personnes ont accès à votre instance n8n, limitez les permissions. n8n Enterprise propose des rôles fins — sur les versions Community, limitez l'accès à l'interface d'administration.

***

## Checklist avant mise en production

Avant d'activer un workflow en production, vérifiez :

* [ ] Retries activés sur tous les nœuds critiques
* [ ] Error Workflow configuré avec notification
* [ ] Toutes les clés API stockées en Credentials (jamais en clair)
* [ ] Nœuds de validation JSON ajoutés après chaque appel Claude
* [ ] Test réalisé avec des données réelles (pas seulement des exemples)
* [ ] Estimation du coût mensuel calculée
* [ ] Logs d'exécution configurés
* [ ] Comportement en cas d'échec documenté (que se passe-t-il si Claude est indisponible ?)

***

## Instance cloud vs auto-hébergée

n8n propose deux modes de déploiement :

{% tabs %}
{% tab title="n8n Cloud" %}
**Avantages :**

* Zéro maintenance infrastructure
* Mises à jour automatiques
* Support disponible

**Inconvénients :**

* Coût mensuel (à partir de \~20€/mois)
* Données hébergées chez n8n
* Limites de volume selon le plan

**Recommandé pour :** individus et petites équipes qui veulent démarrer vite.
{% endtab %}

{% tab title="Auto-hébergé (Docker)" %}
**Avantages :**

* Contrôle total des données
* Coût d'infrastructure uniquement
* Pas de limites artificielles de volume

**Inconvénients :**

* Maintenance à votre charge
* Mises à jour manuelles
* Vous gérez la sécurité et les sauvegardes

**Recommandé pour :** équipes avec des contraintes RGPD strictes, fort volume d'exécutions, ou compétences techniques en interne.
{% endtab %}
{% endtabs %}

{% hint style="info" %}
Si vous débutez, commencez par n8n Cloud. Migrez vers une instance auto-hébergée quand vous avez validé vos workflows et identifié des contraintes de volume ou de confidentialité.
{% endhint %}
