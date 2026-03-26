# Workflow 4 — CRM enrichi par l'IA

**Problème résolu :** Votre CRM contient des contacts avec des informations incomplètes. Avant chaque appel ou email, vous passez 15 à 20 minutes à rechercher manuellement le contexte sur LinkedIn, le site de l'entreprise, les actualités récentes. Ce temps s'accumule sur des dizaines de contacts par semaine.

**Ce que fait ce workflow :** Quand un nouveau contact est ajouté dans votre CRM (HubSpot, Airtable, Notion), il collecte automatiquement des informations publiques sur la personne et l'entreprise, génère un résumé d'enrichissement structuré, et met à jour la fiche contact.

---

## Nœuds nécessaires

- **HubSpot Trigger** ou **Airtable Trigger** — nouveau contact
- **HTTP Request** — recherche d'informations (LinkedIn, site web)
- **Anthropic Claude** — synthèse et structuration
- **HubSpot / Airtable** — mise à jour de la fiche
- **Gmail** — notification optionnelle

---

## Schéma du workflow

```
[HubSpot : nouveau contact]
    → [HTTP : scrape profil LinkedIn/site web]
    → [Claude : synthèse enrichissement]
    → [HubSpot : mise à jour contact]
    → [Gmail : notification "contact enrichi"]
```

---

## Variante Airtable (recommandée pour commencer)

Si vous n'avez pas HubSpot, cette variante utilise Airtable comme CRM léger.

### Structure de la table Airtable "Contacts"

Créez une table avec ces champs :
- **Nom** (texte)
- **Email** (email)
- **Entreprise** (texte)
- **Site web** (URL)
- **Poste** (texte)
- **Statut enrichissement** (select) — À enrichir / En cours / Enrichi
- **Résumé IA** (texte long)
- **Contexte appel** (texte long)
- **Dernière mise à jour** (date)

---

## Configuration détaillée

### Nœud 1 — Airtable Trigger

- **Event :** New Record
- **Table :** Contacts
- **Filter :** Statut enrichissement = "À enrichir"

### Nœud 2 — HTTP Request (site web de l'entreprise)

Ce nœud récupère le contenu de la page d'accueil pour extraire le contexte métier.

- **Method :** GET
- **URL :** `{{ $json['Site web'] }}`
- **Options > Response Format :** Text

{% hint style="warning" %}
Certains sites bloquent les requêtes automatiques. Si la requête échoue, ajoutez un nœud "If" pour gérer les erreurs et continuer le workflow même sans cette donnée.
{% endhint %}

### Nœud 3 — Anthropic Claude (enrichissement)

**System Prompt :**
```
Tu es un assistant de recherche commerciale.
À partir des informations fournies sur un contact et son entreprise,
tu génères deux blocs de texte structurés :

1. RÉSUMÉ ENTREPRISE : 3-4 phrases sur l'activité, la taille estimée,
   les enjeux probables, le positionnement marché.

2. CONTEXTE APPEL : 2-3 points d'accroche pour personnaliser
   un premier contact. Formule comme des notes préparatoires,
   pas comme un discours commercial.

Sois factuel. Si une information est manquante, indique-le
plutôt qu'inventer.

Format de sortie :
RÉSUMÉ ENTREPRISE :
[texte]

CONTEXTE APPEL :
- [point 1]
- [point 2]
- [point 3]
```

**User Message :**
```
Contact : {{ $json.Nom }}
Poste : {{ $json.Poste }}
Entreprise : {{ $json.Entreprise }}
Site web : {{ $json['Site web'] }}

Contenu de la page d'accueil du site :
{{ $node["HTTP Request"].json.data || "Non disponible" }}
```

### Nœud 4 — Code (extraire les deux blocs)

```javascript
const text = $input.first().json.output[0].content[0].text;

const resumeMatch = text.match(/RÉSUMÉ ENTREPRISE :\n([\s\S]*?)(?=\nCONTEXTE APPEL|$)/);
const contexteMatch = text.match(/CONTEXTE APPEL :\n([\s\S]*?)$/);

const resume = resumeMatch ? resumeMatch[1].trim() : '';
const contexte = contexteMatch ? contexteMatch[1].trim() : '';

return [{
  json: {
    ...($input.first().json),
    resume_entreprise: resume,
    contexte_appel: contexte
  }
}];
```

### Nœud 5 — Airtable (mise à jour)

**Operation :** Update Record

- **Record ID :** `{{ $json.id }}` (l'ID du record Airtable)
- **Résumé IA :** `{{ $json.resume_entreprise }}`
- **Contexte appel :** `{{ $json.contexte_appel }}`
- **Statut enrichissement :** `Enrichi`
- **Dernière mise à jour :** `{{ new Date().toISOString() }}`

---

## Enrichissement avancé avec l'API LinkedIn (option)

Pour les équipes qui ont accès à l'API LinkedIn ou à un outil comme Proxycurl ou Apollo, vous pouvez enrichir les données directement depuis le profil LinkedIn.

### Nœud HTTP Request — Proxycurl (si disponible)

```
Method : GET
URL : https://nubela.co/proxycurl/api/v2/linkedin
Headers :
  Authorization : Bearer {{ $credentials.proxycurl_api_key }}
Query Parameters :
  linkedin_profile_url : {{ $json.linkedin_url }}
  extra : include
```

Ajoutez les données renvoyées (expériences, formations, posts récents) au User Message du nœud Claude pour un enrichissement plus précis.

---

## Notification email optionnelle

Ajoutez un nœud Gmail après la mise à jour Airtable pour être notifié quand un contact est enrichi.

**Subject :** `Contact enrichi — {{ $json.Nom }} ({{ $json.Entreprise }})`

**Body :**
```
Nouveau contact enrichi dans votre CRM.

Nom : {{ $json.Nom }}
Entreprise : {{ $json.Entreprise }}
Poste : {{ $json.Poste }}

Résumé entreprise :
{{ $json.resume_entreprise }}

Contexte appel :
{{ $json.contexte_appel }}
```

---

## Variante : enrichissement en lot

Si vous avez déjà une base de contacts à enrichir (pas seulement les nouveaux), modifiez le déclencheur :

1. Utilisez un **Schedule Trigger** (ex. : chaque nuit à 2h)
2. Remplacez l'Airtable Trigger par **Airtable — Get Many Records** avec filtre `Statut enrichissement = "À enrichir"`
3. Ajoutez un nœud **Split In Batches** (batch size : 5) pour éviter de surcharger l'API
4. Le reste du workflow reste identique

{% hint style="info" %}
Traitez par lots de 5 maximum si vous utilisez le mode lot. Les appels Claude prennent 2 à 5 secondes chacun — un lot de 50 contacts = 3 à 5 minutes d'exécution, et vous évitez les rate limits de l'API Anthropic.
{% endhint %}
