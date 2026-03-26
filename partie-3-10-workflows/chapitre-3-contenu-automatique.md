# Workflow 3 — Création de contenu automatisée

{% hint style="info" %}
Cette sous-partie décrit un workflow qui transforme un brief minimal en plusieurs drafts de contenu prêts à relire. Il résout le problème du temps de rédaction en automatisant le premier jet, sans vous laisser partir d’une page blanche.
{% endhint %}

**Problème résolu :** Vous devez produire régulièrement du contenu (posts LinkedIn, newsletters, descriptions de produits, articles de blog) mais la production manuelle est chronophage. Vous avez les idées et les données — il vous manque le temps pour la rédaction.

**Ce que fait ce workflow :** À partir d'un brief minimal (titre, angle, mots-clés), il génère un premier jet de contenu formaté selon vos spécifications, dans votre ton habituel. Vous relisez et affinez — vous ne partez plus d'une page blanche.

{% hint style="success" %}
À la fin de l’exécution, vous obtenez automatiquement un draft LinkedIn et une version newsletter du même sujet, stockés dans Notion et prêts pour relecture ou publication.
{% endhint %}

***

## Nœuds nécessaires

* **Webhook** ou **Airtable Trigger** — déclencheur sur nouvelle entrée brief
* **Anthropic Claude** — génération du contenu
* **Notion** ou **Google Docs** — stockage du draft
* **Gmail** — notification optionnelle

***

## Schéma du workflow

```
[Airtable : nouvelle ligne "brief"]
    → [Claude : génère post LinkedIn]
    → [Claude : génère variation email]
    → [Notion : crée page draft]
    → [Gmail : notification "nouveau draft prêt"]
```

***

## Mise en place de la base Airtable pour les briefs

Créez une table "Briefs" avec ces champs :

* **Titre** (texte) — ex. "Pourquoi l'automatisation IA fait peur aux managers"
* **Type** (select) — LinkedIn / Newsletter / Article / Description produit
* **Angle** (texte) — ex. "Angle provocateur : la vraie raison c'est la peur de perdre le contrôle"
* **Mots-clés** (texte) — ex. "IA, manager, automatisation, résistance au changement"
* **Longueur** (select) — Court (300 mots) / Moyen (600 mots) / Long (1200 mots)
* **Statut** (select) — À générer / En révision / Validé / Publié

***

## Configuration du nœud Airtable Trigger

* **Event :** New Record
* **Table :** Briefs
* **Filter :** Statut = "À générer"

Quand vous ajoutez une ligne avec le statut "À générer", le workflow se déclenche automatiquement.

***

## Configuration du nœud Claude — Post LinkedIn

**System Prompt :**

```
Tu es un expert en personal branding LinkedIn pour les professionnels du digital et de l'IA.
Tu rédiges des posts LinkedIn engageants, dans un style direct et humain.

Règles de rédaction :
- Commence par une accroche qui crée une tension ou surprise
- Phrases courtes, une idée par ligne
- Pas de bullet points standards — préfère les formats narratifs
- Maximum 1300 caractères
- Termine par une question ouverte qui invite à commenter
- Ton : professionnel mais accessible, jamais corporate
- Pas d'emojis en excès (maximum 3-4 si vraiment pertinents)

Format de sortie : le texte du post uniquement, prêt à copier-coller.
```

**User Message :**

```
Sujet : {{ $json.Titre }}
Angle : {{ $json.Angle }}
Mots-clés à intégrer : {{ $json['Mots-clés'] }}
Longueur : {{ $json.Longueur }}
```

***

## Configuration du nœud Claude — Newsletter (variation)

Créez un second nœud Claude pour générer une variation email/newsletter du même contenu.

**System Prompt :**

```
Tu es un rédacteur de newsletters B2B.
À partir d'un brief, tu rédiges un email de newsletter professionnel.

Structure obligatoire :
1. Ligne d'objet (5-8 mots, crée de la curiosité)
2. Corps : introduction + développement + conclusion avec appel à l'action
3. Signature : "Bonne lecture, [Prénom]"

Ton : direct, utile, pas de jargon inutile.
Format de sortie :
OBJET : [ligne d'objet]
---
[corps de l'email]
```

**User Message :** identique au nœud précédent.

***

## Stockage dans Notion

**Notion — Create Page :**

* **Database :** Base "Drafts Contenu"
* **Titre :** `Draft — {{ $json.Titre }}`
* **Type :** `{{ $json.Type }}`
* **Contenu LinkedIn :** `{{ $node["Claude LinkedIn"].json.output[0].content[0].text }}`
* **Contenu Email :** `{{ $node["Claude Newsletter"].json.output[0].content[0].text }}`
* **Statut :** "En révision"
* **Date :** `{{ new Date().toISOString() }}`

***

## Personnaliser le ton

La qualité du contenu généré dépend directement de la description de votre ton dans le System Prompt. Investissez 30 minutes pour définir votre style personnel :

1. Prenez 5 de vos meilleurs posts ou emails passés
2. Copiez-les dans Claude et demandez : "Décris le style et le ton de ces textes en 150 mots maximum"
3. Utilisez cette description dans votre System Prompt

Un exemple de description de ton personnalisé :

```
Style d'écriture : direct et sans fioritures. Préfère l'exemple concret
à l'explication théorique. N'hésite pas à prendre position.
Évite le vocabulaire "transformation", "disruption", "synergies".
Humour sec occasionnel. Conclut souvent par une inversion ou un retournement
de la prémisse initiale.
```

Ce niveau de précision dans le prompt transforme la sortie.
