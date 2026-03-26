# Votre premier workflow IA

Construisons votre premier workflow complet et fonctionnel. Le cas d'usage : **résumer automatiquement un texte et l'envoyer par email**.

Simple en apparence, ce workflow vous apprendra les patterns fondamentaux réutilisables dans tous les workflows plus complexes.

---

## Ce que va faire ce workflow

1. Vous le déclenchez manuellement avec un texte en entrée
2. n8n envoie ce texte à Claude avec l'instruction de le résumer
3. Claude renvoie un résumé structuré
4. n8n envoie ce résumé par email à l'adresse que vous spécifiez

---

## Construction pas à pas

{% stepper %}
{% step %}
### Créer le workflow et configurer le déclencheur

Créez un nouveau workflow. Nommez-le "Résumé automatique + Email".

Ajoutez un nœud "Manual Trigger". Dans ses paramètres, ajoutez un champ de données :
- Nom du champ : `texte`
- Type : String

Ce champ vous permettra de passer du texte au workflow au moment du déclenchement.
{% endstep %}
{% step %}
### Ajouter le nœud Claude

Cliquez "+" après le trigger. Ajoutez le nœud "Anthropic Claude".

Configurez-le ainsi :

**System Prompt :**
```
Tu es un assistant de synthèse professionnelle.
Tu reçois un texte et tu produis un résumé structuré.
Format de sortie obligatoire :
**RÉSUMÉ** (2-3 phrases max)
**POINTS CLÉS** (3 à 5 points, format liste)
**À RETENIR** (une seule phrase)
Ne rajoute aucun commentaire en dehors de ce format.
```

**User Message :**
```
{{ $json.texte }}
```

(Cette expression référence la valeur du champ "texte" du nœud précédent)

**Model :** `claude-sonnet-4-5`
**Temperature :** 0.3 (pour un résumé factuel, on veut peu de créativité)
{% endstep %}
{% step %}
### Ajouter le nœud Email

Cliquez "+" après le nœud Claude. Ajoutez le nœud "Send Email" (ou Gmail, si vous utilisez Gmail).

Vous devrez d'abord créer un credential Gmail (OAuth) ou un credential SMTP selon votre configuration email.

Configurez le nœud :
- **To :** votre adresse email
- **Subject :** `Résumé automatique — {{ new Date().toLocaleDateString('fr-FR') }}`
- **Email Type :** HTML
- **HTML :**
```html
<h2>Résumé automatique</h2>
{{ $json.output[0].content[0].text }}
```
{% endstep %}
{% step %}
### Tester le workflow

Cliquez "Test Workflow". Dans le panneau qui s'ouvre, renseignez le champ "texte" avec un article ou un texte de votre choix (au moins 200 mots pour que le résumé soit pertinent).

Cliquez "Run Test".

Vous verrez l'exécution en temps réel : le déclencheur, puis Claude (2 à 5 secondes), puis l'envoi de l'email.
{% endstep %}
{% step %}
### Activer le workflow

Si le test est concluant, cliquez "Save" et activez le workflow avec le toggle en haut à droite. Il est maintenant actif et peut être déclenché à la demande.
{% endstep %}
{% endstepper %}

---

## Ce que vous venez d'apprendre

Ce workflow simple contient les patterns fondamentaux de tous les workflows n8n + Claude :

**Pattern 1 — Le nœud Claude comme transformateur.** Vous lui donnez du texte brut, il renvoie du texte structuré que vous pouvez utiliser dans la suite du workflow.

**Pattern 2 — Les expressions `{{ }}`.** C'est la colle qui relie les nœuds entre eux. La sortie d'un nœud devient l'entrée du suivant via ces expressions.

**Pattern 3 — Le System Prompt comme contrat.** Définir clairement le format de sortie attendu est ce qui rend la réponse de Claude prévisible et utilisable par les nœuds suivants.

---

## Variations possibles

<details>
<summary>Variation : déclencher automatiquement sur un email entrant</summary>

Remplacez le "Manual Trigger" par un nœud "Gmail — On New Email". Configurez-le pour surveiller un label ou une adresse spécifique. Le workflow se déclenchera automatiquement à chaque nouvel email correspondant.
</details>

<details>
<summary>Variation : stocker le résumé dans Notion plutôt que l'envoyer par email</summary>

Remplacez le nœud Email par un nœud "Notion — Create Page". Créez une nouvelle page dans une base de données Notion avec le résumé en contenu. Idéal pour constituer une bibliothèque de synthèses.
</details>

<details>
<summary>Variation : déclencher sur un webhook externe</summary>

Remplacez le "Manual Trigger" par un nœud "Webhook". Vous obtenez une URL que vous pouvez appeler depuis n'importe quelle autre application, script, ou bouton Zapier. Cela transforme votre workflow en API accessible.
</details>

---

Vous avez maintenant tous les éléments pour construire des workflows plus complexes. La Partie 3 vous donne 5 workflows complets, prêts à déployer.
