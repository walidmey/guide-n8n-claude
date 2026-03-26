# Agents chaînés : quand n8n orchestre plusieurs IA

Les workflows des parties précédentes suivent un schéma linéaire : déclencher → collecter → Claude analyse → agir. La sortie de Claude alimente la prochaine étape, mais Claude lui-même n'a pas d'influence sur le chemin parcouru.

Les agents chaînés changent ça. La réponse d'un premier Claude décide quelle branche du workflow s'active, ce que fait le deuxième Claude, et parfois si le workflow continue ou s'arrête.

---

## La différence entre workflow et agent

Un **workflow** suit toujours le même chemin. Un **agent** décide son chemin en fonction du contexte.

Dans un workflow classique :
```
Email reçu → Claude analyse → Draft créé
```

Dans un système à agents :
```
Email reçu → Claude analyse
    → Si devis demandé → Claude 2 génère le chiffrage
    → Si réclamation → Claude 2 prépare une réponse d'escalade
    → Si question technique → Claude 2 consulte la documentation + répond
    → Si spam → Archiver sans action
```

La différence : le premier Claude prend une décision, le second agit en conséquence avec un contexte et des instructions différents.

---

## Pattern fondamental : le routeur

Le routeur est le pattern de base des agents chaînés. Un premier Claude classifie et route, un ou plusieurs Claudes spécialisés traitent.

### Structure

```
[Déclencheur]
    → [Claude Routeur : classifie + décide]
    → [Switch : lit la décision]
        → [Branche A] → [Claude Spécialiste A]
        → [Branche B] → [Claude Spécialiste B]
        → [Branche C] → [Claude Spécialiste C]
```

### Prompt du Claude Routeur

Le routeur doit renvoyer une décision en JSON, claire et exploitable par le nœud Switch :

```
Tu es un routeur de requêtes.
Tu reçois une demande et tu décides quel spécialiste doit la traiter.

Renvoie UNIQUEMENT ce JSON :
{
  "type": "devis | support | info | hors_scope",
  "complexite": "simple | complexe",
  "contexte_cle": "ce que le spécialiste doit absolument savoir en une phrase"
}
```

### Prompt du Claude Spécialiste (exemple : devis)

```
Tu es un assistant spécialisé en chiffrage de prestations.
Tu reçois une demande qualifiée par le routeur, avec son contexte.

Contexte transmis : {{ $node["Routeur"].json.contexte_cle }}
Complexité : {{ $node["Routeur"].json.complexite }}

À partir de la demande originale, génère un email de réponse
proposant une estimation de délai et de budget.
```

---

## Pattern avancé : la boucle avec condition d'arrêt

Ce pattern permet à Claude d'affiner son travail de manière itérative — il produit une première version, l'évalue, et recommence si la qualité n'est pas satisfaisante.

### Cas d'usage : génération de contenu avec contrôle qualité

```
[Brief entrant]
    → [Claude : génère le contenu]
    → [Claude Évaluateur : note la qualité 1-10 + raison]
    → [If : score >= 7]
        → [oui] → [Stocker + Notifier]
        → [non] → [Claude : régénère avec feedback]
                    → [retour au Claude Évaluateur] (max 3 itérations)
```

### Implémentation dans n8n

n8n ne supporte pas nativement les boucles infinies, mais vous pouvez simuler jusqu'à N itérations avec un nœud Code qui maintient un compteur :

```javascript
const iteration = $input.first().json.iteration || 0;
const score = $input.first().json.score || 0;
const maxIterations = 3;

if (score >= 7 || iteration >= maxIterations) {
  return [{ json: { ...($input.first().json), done: true } }];
} else {
  return [{ json: { ...($input.first().json), done: false, iteration: iteration + 1 } }];
}
```

Puis un nœud If sur `done === true` pour sortir de la boucle.

---

## Pattern de mémoire : transmettre le contexte entre agents

Dans un système multi-agents, chaque Claude reçoit uniquement ce qu'on lui donne. Si le Claude 2 a besoin du contexte analysé par le Claude 1, vous devez le transmettre explicitement.

### Approche recommandée : objet contexte cumulatif

À chaque étape, enrichissez un objet JSON partagé :

```javascript
// Après Claude 1
const contexteGlobal = {
  input_original: $node["Déclencheur"].json.texte,
  analyse_claude1: $node["Claude1"].json.output[0].content[0].text,
  timestamp: new Date().toISOString()
};

// Passez cet objet au Claude 2 dans son User Message
```

```
// User Message du Claude 2
Contexte de l'analyse précédente :
{{ $json.analyse_claude1 }}

Demande originale :
{{ $json.input_original }}

Sur cette base, effectue [ta tâche spécifique].
```

---

## Exemple complet : traitement automatique de devis entrants

Voici un workflow de 4 nœuds Claude enchaînés pour traiter une demande de devis par email.

### Schéma

```
[Gmail : email entrant]
    → [Claude 1 — Qualification]
          Décide : est-ce une vraie demande de devis ?
          Extrait : besoin principal, budget mentionné, urgence
    → [Switch : qualifié / non qualifié]
    → [qualifié] → [Claude 2 — Analyse besoin]
                       Reformule le besoin, identifie les ambiguïtés
    → [Claude 3 — Rédaction réponse]
               Rédige un email de réponse personnalisé
               avec questions de qualification et prochaine étape
    → [Gmail : créer brouillon]
    → [non qualifié] → [Gmail : archiver + marquer lu]
```

### Ce que ce workflow remplace

Sans automatisation : lire l'email (2 min), décider si c'est sérieux (1 min), rédiger une première réponse (10 min). Soit 13 minutes par demande.

Avec le workflow : vous recevez un brouillon prêt à envoyer. Vous relisez en 2 minutes et envoyez.

---

## Limites à connaître

**La latence s'accumule.** Chaque appel Claude prend 2 à 5 secondes. Trois Claudes chaînés = 6 à 15 secondes de traitement. Pour des workflows déclenchés à la demande, c'est acceptable. Pour des workflows à très fort volume (centaines d'exécutions par heure), c'est à surveiller.

**Les erreurs se propagent.** Si Claude 1 renvoie un JSON malformé, Claude 2 reçoit des données incorrectes et produit une réponse sans valeur. Ajoutez des nœuds de validation entre chaque étape, notamment un nœud Code pour vérifier que le JSON est valide avant de continuer.

**Les coûts s'additionnent.** Chaque appel Claude consomme des tokens. Un workflow avec 3 Claudes consomme 3× plus qu'un workflow avec 1 Claude. Estimez le coût unitaire avant de déployer à grande échelle.

{% hint style="info" %}
Commencez avec 2 Claudes maximum avant d'en chaîner davantage. Maîtrisez d'abord le pattern routeur + spécialiste, puis ajoutez de la complexité seulement si les cas d'usage le justifient.
{% endhint %}
