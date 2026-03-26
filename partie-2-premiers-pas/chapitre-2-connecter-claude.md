# Connecter Claude à n8n

Pour appeler Claude depuis n8n, il vous faut deux choses : une clé API Anthropic, et un credential configuré dans n8n.

---

## Étape 1 — Obtenir une clé API Anthropic

{% stepper %}
{% step %}
### Créer un compte Anthropic

Rendez-vous sur [console.anthropic.com](https://console.anthropic.com) et créez un compte.
{% endstep %}
{% step %}
### Ajouter des crédits

Dans la section Billing, ajoutez un moyen de paiement et des crédits (commencez par 10 €, largement suffisant pour les tests). La facturation est à l'usage — pas d'abonnement mensuel obligatoire.
{% endstep %}
{% step %}
### Générer une clé API

Dans la section "API Keys", cliquez "Create Key". Donnez-lui un nom explicite (ex. "n8n-workflows"). Copiez la clé immédiatement — elle ne sera plus affichée après fermeture de la fenêtre.

La clé ressemble à : `sk-ant-api03-xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx`
{% endstep %}
{% endstepper %}

{% hint style="danger" %}
**Ne partagez jamais votre clé API.** Elle donne accès à votre compte Anthropic et peut générer des coûts. Stockez-la dans n8n Credentials et nulle part ailleurs (pas dans un Google Doc, pas dans un email).
{% endhint %}

---

## Étape 2 — Configurer le credential dans n8n

{% stepper %}
{% step %}
### Accéder à Credentials

Dans n8n, cliquez sur "Credentials" dans le menu gauche, puis "+ Add Credential".
{% endstep %}
{% step %}
### Chercher "Anthropic"

Dans la barre de recherche, tapez "Anthropic". Sélectionnez "Anthropic API".
{% endstep %}
{% step %}
### Coller la clé API

Dans le champ "API Key", collez votre clé. Donnez un nom au credential (ex. "Claude — Production"). Cliquez "Save".
{% endstep %}
{% step %}
### Tester la connexion

n8n va tester automatiquement la connexion. Si tout est correct, vous verrez un indicateur vert "Connection tested successfully".
{% endstep %}
{% endstepper %}

---

## Étape 3 — Ajouter le nœud Claude dans un workflow

Dans votre workflow de test (celui avec le Manual Trigger), cliquez sur le "+" après le nœud trigger.

Dans le panneau de recherche de nœuds, tapez "Claude" ou "Anthropic". Sélectionnez **"Anthropic Claude"**.

Configurez le nœud :

| Paramètre | Valeur recommandée |
|-----------|-------------------|
| Credential | Sélectionnez le credential créé à l'étape 2 |
| Operation | Message |
| Model | claude-sonnet-4-5 (bon équilibre vitesse/qualité) |
| System Prompt | Laissez vide pour l'instant |
| User Message | "Dis-moi bonjour en 5 langues différentes." |

Cliquez "Test step" pour exécuter uniquement ce nœud. Si la connexion fonctionne, vous verrez la réponse de Claude dans le panneau de résultats à droite.

---

## Comprendre les sorties du nœud Claude

Quand Claude répond, n8n reçoit un objet JSON. La sortie qui vous intéresse est dans `output[0].content[0].text`.

Pour les workflows pratiques, vous utilisez cette valeur dans les nœuds suivants. Par exemple, si vous voulez envoyer la réponse de Claude par email, le champ "Email body" contiendra l'expression : `{{ $json.output[0].content[0].text }}`

{% hint style="info" %}
Cette expression `{{ ... }}` est la syntaxe n8n pour référencer des données depuis un nœud précédent. Dans le panneau de configuration de n'importe quel nœud, vous pouvez glisser des champs depuis le panneau de résultats pour les insérer automatiquement avec la bonne expression.
{% endhint %}

---

## Les paramètres clés du nœud Claude

**Model** — Le modèle à utiliser. Pour la plupart des cas d'usage : `claude-sonnet-4-5` (rapide, économique, performant). Pour des tâches complexes nécessitant un raisonnement approfondi : `claude-opus-4-5`. Pour du texte court et simple : `claude-haiku-4-5-20251001`.

**Temperature** — Contrôle la créativité de la réponse. 0 = déterministe (même prompt → même réponse, idéal pour les classifications). 1 = créatif (idéal pour la rédaction). Défaut : 1.

**Max Tokens** — Longueur maximale de la réponse. Limitez ce paramètre pour les réponses courtes (classifications, scores) — ça accélère les appels et réduit les coûts.

**System Prompt** — Instructions permanentes que Claude suit pour tous les échanges dans ce nœud. C'est ici que vous définissez le rôle, le format de sortie, et les contraintes. C'est votre levier principal de contrôle.
