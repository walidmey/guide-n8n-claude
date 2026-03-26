# Workflow 1 — Veille et synthèse automatique

{% hint style="info" %}
Cette sous-partie décrit un workflow de veille qui collecte des articles, filtre les plus pertinents avec Claude, puis assemble un digest exploitable. Il résout le problème du trop-plein d'informations en ne gardant que ce qui mérite vraiment votre attention.
{% endhint %}

**Problème résolu :** Vous avez des dizaines de sources à surveiller (blogs, newsletters, alertes Google, flux RSS) mais pas le temps de tout lire. La majorité des articles ne vous intéresse pas, mais vous ne voulez pas rater les quelques informations vraiment pertinentes.

**Ce que fait ce workflow :** Chaque matin, il collecte les nouveaux articles de vos sources, filtre ceux qui correspondent à vos sujets d'intérêt grâce à Claude, et vous envoie un digest avec un résumé de chaque article retenu.

{% hint style="success" %}
À la fin de l'exécution, vous recevez un digest email avec uniquement les articles jugés pertinents, chacun accompagné d’un résumé court et de l’idée clé à retenir.
{% endhint %}

**Temps d'exécution estimé :** 1 à 3 minutes selon le volume de sources. **Fréquence recommandée :** Quotidienne (lundi à vendredi, à 7h30).

***

## Nœuds nécessaires

* **Schedule Trigger** — déclencheur horaire (natif n8n)
* **RSS Read** — lecture des flux RSS (natif n8n)
* **Split In Batches** — pour traiter les articles un par un (natif n8n)
* **Anthropic Claude** — filtrage et résumé
* **If** — pour ne garder que les articles pertinents (natif n8n)
* **Gmail / Email** — envoi du digest final

***

## Schéma du workflow

```
[Schedule 7h30] → [RSS Read × sources] → [Split In Batches]
    → [Claude : pertinent ?] → [If : oui/non]
        → [oui] → [Accumuler résumés]
        → [non] → [Ignorer]
    → [Assembler digest] → [Envoyer email]
```

***

## Configuration détaillée

### Nœud 1 — Schedule Trigger

* **Trigger at :** 07:30
* **Days :** Monday to Friday
* **Timezone :** Europe/Paris

### Nœud 2 — RSS Read

Ajoutez vos sources une par une. Pour chaque source :

* URL du flux RSS (ex. : `https://www.lesechos.fr/rss/rss_la_une.xml`)

{% hint style="info" %}
La plupart des blogs et médias proposent un flux RSS. Si vous ne trouvez pas l'URL, ajoutez `/feed` ou `/rss` à la fin de l'URL principale du site.
{% endhint %}

### Nœud 3 — Split In Batches

Ce nœud divise le tableau d'articles en éléments individuels pour les traiter un par un.

* **Batch Size :** 1

### Nœud 4 — Anthropic Claude (Filtre + Résumé)

**System Prompt :**

```
Tu es un assistant de veille spécialisé.
Tu analyses des articles et décides s'ils sont pertinents selon des critères précis.

Critères de pertinence : [INSÉREZ VOS SUJETS]
Exemples : transformation digitale, IA en entreprise, marketing B2B,
secteur [votre secteur], concurrent [nom du concurrent]

Tu renvoies UNIQUEMENT ce JSON (aucun autre texte) :
{
  "pertinent": true/false,
  "raison": "une phrase expliquant pourquoi",
  "resume": "2-3 phrases résumant l'article si pertinent, vide sinon",
  "angle": "l'information clé à retenir si pertinent, vide sinon"
}
```

**User Message :**

```
Titre : {{ $json.title }}
Source : {{ $json.link }}
Date : {{ $json.pubDate }}
Contenu : {{ $json.contentSnippet }}
```

**Temperature :** 0 **Max Tokens :** 300

### Nœud 5 — Code (parser JSON)

Claude renvoie du texte JSON. Ce nœud le transforme en objet utilisable :

```javascript
const response = $input.first().json.output[0].content[0].text;
const parsed = JSON.parse(response);
return [{ json: { ...parsed, title: $input.first().json.title, link: $input.first().json.link } }];
```

### Nœud 6 — If (filtre pertinence)

* **Condition :** `{{ $json.pertinent }}` equals `true`

### Nœud 7 — Aggregate (accumuler les résultats)

Ce nœud collecte tous les articles pertinents avant d'envoyer le digest.

* **Aggregate :** All item data into a single list

### Nœud 8 — Code (assembler le digest)

```javascript
const articles = $input.first().json.data;
if (!articles || articles.length === 0) {
  return [{ json: { digest: "Aucun article pertinent aujourd'hui.", count: 0 } }];
}

let html = `<h2>Votre veille du ${new Date().toLocaleDateString('fr-FR')}</h2>`;
html += `<p><em>${articles.length} article(s) sélectionné(s)</em></p><hr>`;

for (const article of articles) {
  html += `<h3><a href="${article.link}">${article.title}</a></h3>`;
  html += `<p><strong>Résumé :</strong> ${article.resume}</p>`;
  html += `<p><strong>À retenir :</strong> ${article.angle}</p>`;
  html += `<hr>`;
}

return [{ json: { digest: html, count: articles.length } }];
```

### Nœud 9 — Gmail (envoyer le digest)

* **To :** votre email
* **Subject :** `Veille du {{ new Date().toLocaleDateString('fr-FR') }} — {{ $json.count }} article(s)`
* **Email Type :** HTML
* **HTML :** `{{ $json.digest }}`

***

## Variantes

<details>

<summary>Publier le digest sur Notion plutôt que par email</summary>

Remplacez le nœud Gmail par un nœud Notion "Create Page". Créez une base de données dédiée à votre veille. Chaque digest devient une page datée, consultable et cherchable.

</details>

<details>

<summary>Ajouter un score de priorité 1-10</summary>

Ajoutez un champ "score" dans le JSON retourné par Claude, et modifiez votre prompt pour évaluer la pertinence sur 10. Triez les articles par score dans le nœud d'assemblage.

</details>
