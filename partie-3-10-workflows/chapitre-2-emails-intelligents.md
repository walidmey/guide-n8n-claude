# Workflow 2 — Tri et réponse aux emails

{% hint style="info" %}
Cette sous-partie décrit un workflow qui analyse vos emails entrants, les classe par catégorie et urgence, puis déclenche l’action adaptée. Il résout le problème du tri manuel et vous aide à traiter plus vite les messages qui comptent vraiment.
{% endhint %}

**Problème résolu :** Votre boîte email est un mélange de demandes urgentes, de newsletters, de sollicitations commerciales, et de vraies questions clients. Vous passez trop de temps à trier avant même de commencer à répondre.

**Ce que fait ce workflow :** À intervalle régulier, il analyse vos emails non lus, les classe par type et urgence, crée des tâches pour ceux qui nécessitent une action, et prépare des brouillons de réponse pour les demandes standard.

{% hint style="success" %}
À la fin de l'exécution, vos emails non lus sont triés automatiquement : les messages importants deviennent des tâches ou des brouillons de réponse, tandis que les newsletters et messages sans intérêt sont écartés du flux principal.
{% endhint %}

***

## Nœuds nécessaires

* **Schedule Trigger** — toutes les 2 heures (natif n8n)
* **Gmail — Get Many Messages** — lecture des emails non lus
* **Anthropic Claude** — analyse et classification
* **Switch** — routage selon la catégorie
* **Gmail — Create Draft** — création de brouillons
* **Notion / Airtable** — création de tâches (optionnel)

***

## Schéma du workflow

```
[Schedule 2h] → [Gmail : emails non lus depuis 2h]
    → [Split] → [Claude : analyse email]
        → [Switch par catégorie]
            → [commercial] → [tâche CRM + brouillon réponse]
            → [support] → [tâche Notion + brouillon réponse]
            → [partenariat] → [tâche Notion + alerte]
            → [newsletter] → [marquer comme lu]
            → [spam] → [archiver]
```

***

## Configuration détaillée

### Nœud 1 — Gmail : Get Many Messages

* **Operation :** Get Many
* **Filters :** `is:unread newer_than:2h`
* **Max Results :** 50

### Nœud 2 — Anthropic Claude (Analyse)

**System Prompt :**

```
Tu es un assistant de gestion d'emails professionnel.
Analyse l'email fourni et renvoie UNIQUEMENT ce JSON :
{
  "categorie": "commercial | support | partenariat | newsletter | spam | autre",
  "urgence": "haute | normale | faible",
  "expediteur_type": "client | prospect | partenaire | inconnu | automatique",
  "objet_reel": "Ce que l'expéditeur veut vraiment en une phrase",
  "action_recommandee": "répondre | transférer | archiver | aucune",
  "reponse_suggeree": "Si action=répondre, suggestion de réponse en 2-3 phrases. Sinon vide.",
  "priorite_score": 1-10
}
Aucun autre texte que ce JSON.
```

**User Message :**

```
De : {{ $json.from }}
Objet : {{ $json.subject }}
Date : {{ $json.date }}
Corps :
{{ $json.text || $json.snippet }}
```

### Nœud 3 — Switch (routage par catégorie)

Créez 5 branches basées sur la valeur de `{{ $json.categorie }}` :

* `commercial` → branche 1
* `support` → branche 2
* `partenariat` → branche 3
* `newsletter` → branche 4
* Default (spam, autre) → branche 5

### Nœud 4a — Branche commercial : créer un brouillon

**Gmail — Create Draft :**

```
To : {{ $node["Gmail"].json.from }}
Subject : Re: {{ $node["Gmail"].json.subject }}
Body : {{ $node["Analyse Claude"].json.reponse_suggeree }}
```

{% hint style="info" %}
Le brouillon est créé mais pas envoyé. Vous le retrouvez dans vos brouillons Gmail pour révision avant envoi. C'est intentionnel — Claude propose, vous validez.
{% endhint %}

### Nœud 4b — Branche support : créer une tâche Notion

**Notion — Create Page :**

* **Database :** votre base de tâches support
* **Titre :** `[Support] {{ $node["Gmail"].json.subject }}`
* **Priorité :** `{{ $node["Analyse Claude"].json.urgence }}`
* **Description :** `{{ $node["Analyse Claude"].json.objet_reel }}`
* **Email de :** `{{ $node["Gmail"].json.from }}`

### Nœud 4d — Branche newsletter : marquer comme lu

**Gmail — Modify Message :**

* **Mark as read :** true
* **Remove Label :** INBOX (pour nettoyer votre boîte si vous souhaitez)

***

## Points d'attention

**La confidentialité des emails.** Vous envoyez le contenu de vos emails à l'API Anthropic. Pour les emails sensibles (données clients, contrats), évaluez si cette pratique est compatible avec vos obligations RGPD. Option : envoyer uniquement l'objet et les 200 premiers caractères du corps, pas le texte complet.

**Le risque d'erreur de classification.** Claude peut mal classifier un email. Ne laissez pas ce workflow archiver ou supprimer des emails automatiquement — limitez les actions automatiques à "marquer comme lu" ou "créer une tâche". Les actions irréversibles restent manuelles.

**Le volume.** Si vous recevez plus de 50 emails non lus par tranche de 2h, ajustez la fréquence ou le filtre. n8n peut gérer le volume, mais les appels Claude ont un coût.
