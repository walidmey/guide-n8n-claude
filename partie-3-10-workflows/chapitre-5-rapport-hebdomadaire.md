# Workflow 5 — Rapport hebdomadaire automatique

**Problème résolu :** Chaque semaine, vous passez 1 à 2 heures à rassembler des données éparpillées (métriques, tâches complétées, anomalies) pour produire un rapport de suivi. L'information existe — elle est juste dans cinq endroits différents.

**Ce que fait ce workflow :** Chaque lundi matin, il collecte les données de la semaine passée depuis vos sources (Google Analytics, Notion, Airtable, ou autre), demande à Claude d'identifier les tendances et anomalies, et envoie un rapport structuré prêt à partager.

**Temps d'exécution estimé :** 2 à 4 minutes.
**Fréquence recommandée :** Chaque lundi à 8h00.

---

## Nœuds nécessaires

- **Schedule Trigger** — lundi 8h00
- **HTTP Request** ou connecteurs natifs — collecte des données
- **Anthropic Claude** — analyse et rédaction
- **Gmail** — envoi du rapport
- **Notion** (optionnel) — archivage

---

## Schéma du workflow

```
[Schedule lundi 8h00]
    → [Collecter données semaine N-1]
        → [Métriques source 1]
        → [Métriques source 2]
        → [Tâches complétées Notion]
    → [Fusionner données]
    → [Claude : analyse + rédaction rapport]
    → [Gmail : envoyer rapport]
    → [Notion : archiver page rapport]
```

---

## Version de base : rapport depuis Airtable + Notion

Cette version fonctionne sans API externes complexes.

### Sources de données utilisées

1. **Airtable** — métriques ou KPIs saisis manuellement chaque semaine
2. **Notion** — tâches complétées dans une base de tâches
3. **Code** — calcul automatique des variations

### Configuration

#### Nœud 1 — Schedule Trigger

- **Trigger at :** 08:00
- **Days :** Monday
- **Timezone :** Europe/Paris

#### Nœud 2 — Code (calculer la plage de dates)

```javascript
const now = new Date();
const dayOfWeek = now.getDay();
const diffToMonday = dayOfWeek === 0 ? -6 : 1 - dayOfWeek;

const lastMonday = new Date(now);
lastMonday.setDate(now.getDate() + diffToMonday - 7);
lastMonday.setHours(0, 0, 0, 0);

const lastSunday = new Date(lastMonday);
lastSunday.setDate(lastMonday.getDate() + 6);
lastSunday.setHours(23, 59, 59, 999);

return [{
  json: {
    week_start: lastMonday.toISOString(),
    week_end: lastSunday.toISOString(),
    week_label: `Semaine du ${lastMonday.toLocaleDateString('fr-FR')} au ${lastSunday.toLocaleDateString('fr-FR')}`
  }
}];
```

#### Nœud 3 — Airtable (métriques de la semaine)

Configurez une table "Métriques hebdo" dans Airtable avec des colonnes pour vos KPIs principaux. Exemple : chiffre d'affaires, leads, conversion, satisfaction.

- **Operation :** Get Many Records
- **Filter :** `IS_AFTER({Date}, '{{ $json.week_start }}') AND IS_BEFORE({Date}, '{{ $json.week_end }}')`

#### Nœud 4 — Notion (tâches complétées)

- **Operation :** Get Database Items
- **Filter :**
  ```json
  {
    "and": [
      { "property": "Statut", "select": { "equals": "Terminé" } },
      { "property": "Date fin", "date": { "after": "{{ $node['Dates'].json.week_start }}" } }
    ]
  }
  ```

#### Nœud 5 — Code (assembler le contexte)

```javascript
const dates = $node["Dates"].json;
const metriques = $input.all().filter(item => item.json.source === 'airtable');
const taches = $input.all().filter(item => item.json.source === 'notion');

// Formater les métriques
let metriquesTxt = '';
for (const m of metriques) {
  metriquesTxt += `- ${m.json.Metrique}: ${m.json.Valeur} (vs ${m.json.Valeur_precedente || 'N/A'} semaine précédente)\n`;
}

// Formater les tâches
let tachesTxt = '';
for (const t of taches) {
  tachesTxt += `- ${t.json.Titre} [${t.json.Categorie || 'Général'}]\n`;
}

return [{
  json: {
    week_label: dates.week_label,
    metriques: metriquesTxt || 'Aucune métrique saisie',
    taches: tachesTxt || 'Aucune tâche terminée enregistrée'
  }
}];
```

---

## Nœud Claude — Analyse et rédaction

**System Prompt :**
```
Tu es un assistant d'analyse de performance.
Tu reçois des données de la semaine écoulée et tu rédiges un rapport
hebdomadaire structuré, destiné à un manager ou au fondateur d'une PME.

Ton rôle : identifier les tendances, signaler les anomalies,
formuler 2-3 recommandations actionnables.

Règles de rédaction :
- Ton professionnel mais direct, pas de jargon
- Commence par les faits saillants (1 paragraphe)
- Ensuite les détails organisés par section
- Termine par des recommandations concrètes
- Maximum 500 mots

Format de sortie obligatoire :
## Faits saillants
[1 paragraphe synthèse]

## Métriques
[analyse des chiffres avec variations]

## Réalisations
[liste des accomplissements principaux]

## Points d'attention
[anomalies, retards, écarts à surveiller]

## Recommandations pour la semaine prochaine
1. [action concrète]
2. [action concrète]
3. [action concrète]
```

**User Message :**
```
Période : {{ $json.week_label }}

MÉTRIQUES :
{{ $json.metriques }}

TÂCHES COMPLÉTÉES :
{{ $json.taches }}
```

---

## Envoi du rapport

### Gmail

- **To :** votre email (et ceux de vos collaborateurs si partagé)
- **Subject :** `Rapport hebdo — {{ $json.week_label }}`
- **Email Type :** HTML
- **HTML :**
```html
<div style="font-family: Arial, sans-serif; max-width: 700px; margin: 0 auto;">
  <h1 style="color: #333; border-bottom: 2px solid #eee; padding-bottom: 10px;">
    Rapport hebdomadaire
  </h1>
  <p style="color: #666; font-size: 14px;">{{ $node["Dates"].json.week_label }}</p>
  <div style="line-height: 1.6;">
    {{ $node["Claude"].json.output[0].content[0].text }}
  </div>
  <hr style="margin-top: 30px;">
  <p style="color: #999; font-size: 12px;">Généré automatiquement par n8n</p>
</div>
```

### Archivage Notion (optionnel)

Créez une base "Rapports hebdos" dans Notion. Chaque semaine, une nouvelle page est créée avec le rapport complet, consultable et cherchable.

**Notion — Create Page :**
- **Titre :** `Rapport — {{ $node["Dates"].json.week_label }}`
- **Contenu :** `{{ $node["Claude"].json.output[0].content[0].text }}`
- **Semaine :** `{{ $node["Dates"].json.week_start }}`

---

## Variante : rapport depuis Google Analytics

Si vous gérez un site web ou une application, remplacez ou complétez les données Airtable par des métriques Analytics.

### Nœud HTTP Request — Google Analytics Data API

```
Method : POST
URL : https://analyticsdata.googleapis.com/v1beta/properties/{{ votre_property_id }}:runReport
Authentication : OAuth2 (compte Google)
Body :
{
  "dateRanges": [
    {
      "startDate": "{{ $node['Dates'].json.week_start.split('T')[0] }}",
      "endDate": "{{ $node['Dates'].json.week_end.split('T')[0] }}"
    }
  ],
  "metrics": [
    { "name": "sessions" },
    { "name": "activeUsers" },
    { "name": "conversions" },
    { "name": "bounceRate" }
  ],
  "dimensions": [
    { "name": "date" }
  ]
}
```

Les données renvoyées alimentent directement le contexte de Claude pour une analyse plus précise.

---

## Personnaliser les sections du rapport

Le prompt définit la structure. Modifiez les sections selon votre contexte :

- **E-commerce :** CA, panier moyen, taux de conversion, retours
- **SaaS :** MRR, churn, nouvelles inscriptions, tickets support
- **Marketing :** impressions, clics, leads qualifiés, coût par lead
- **Opérations :** tâches livrées vs planifiées, délais, incidents

La force de ce workflow est que Claude adapte son analyse au contenu — vous n'avez pas à modifier la logique pour changer de métriques, juste le prompt et les sources de données.
