# Ce que Claude apporte dans un workflow

La plupart des gens utilisent Claude comme un outil de réponse : vous posez une question, vous obtenez une réponse. C'est utile. Ce n'est pas de l'automatisation.

Dans un workflow n8n, Claude joue un rôle différent. Il est appelé programmatiquement, reçoit des données structurées, et renvoie une réponse que le workflow peut utiliser pour prendre des décisions ou produire un livrable.

---

## Les quatre rôles de Claude dans un workflow

### 1. Analyseur de contenu

Claude lit un contenu (email, article, document, message) et en extrait ce dont vous avez besoin : catégorie, sentiment, entités nommées, points clés, niveau d'urgence.

**Exemple :** Un email arrive dans votre boîte. n8n l'envoie à Claude avec l'instruction : "Classe cet email parmi : demande commerciale, réclamation, partenariat, spam, autre. Renvoie uniquement la catégorie." n8n utilise la réponse pour router l'email vers la bonne équipe ou créer la bonne tâche.

### 2. Rédacteur structuré

Claude rédige du contenu selon un format précis que vous lui spécifiez dans le prompt : résumé en 3 points, email de réponse, fiche client, description produit, synthèse hebdomadaire.

**Exemple :** Chaque vendredi, n8n collecte toutes les notes de la semaine depuis Notion, les envoie à Claude avec l'instruction : "Rédige un rapport hebdomadaire de 300 mots maximum, structuré en 3 sections : Avancées, Blocages, Priorités semaine prochaine." Le rapport arrive dans votre email le vendredi soir.

### 3. Décideur conditionnel

Claude évalue une situation et renvoie une décision binaire ou une valeur discrète que n8n utilise pour brancher le workflow dans une direction ou une autre.

**Exemple :** Un nouveau prospect entre dans votre CRM. Claude évalue son profil (secteur, taille, poste du contact, message) et renvoie "A", "B" ou "C" selon un score de priorité. n8n utilise cette valeur pour décider du séquençage commercial.

### 4. Transformateur de données

Claude convertit des données d'un format à un autre, extrait des informations structurées depuis du texte libre, ou normalise des données hétérogènes.

**Exemple :** Des prospects remplissent un formulaire en texte libre ("Je dirige une PME de 50 personnes dans le bâtiment et je cherche à améliorer ma gestion de chantier"). Claude extrait : secteur = bâtiment, taille = 50 personnes, besoin = gestion de chantier. Ces champs structurés vont directement dans votre CRM.

---

## Ce que Claude ne fait pas bien dans un workflow

Être honnête ici évite des déceptions.

**Claude est lent.** Un appel API à Claude prend 2 à 15 secondes selon la longueur de la réponse. Ce n'est pas un problème pour des workflows asynchrones (qui tournent en fond). C'est rédhibitoire pour des workflows qui doivent répondre en temps réel à un utilisateur.

**Claude n'a pas de mémoire entre les appels.** Chaque appel est indépendant. Si votre workflow appelle Claude 10 fois, il ne sait pas ce qui s'est passé dans les 9 appels précédents — sauf si vous lui envoyez explicitement le contexte dans chaque requête.

**Claude peut se tromper.** Sur des classifications simples avec un bon prompt, il est très fiable (>95% dans la plupart des cas). Sur des jugements complexes ou ambigus, il peut se tromper. Concevez vos workflows en conséquence : prévoyez un chemin de sortie pour les cas incertains, et ne laissez pas Claude prendre des décisions irréversibles sans validation humaine.

{% hint style="warning" %}
**Règle d'or :** Claude est un assistant, pas un décideur final. Utilisez-le pour enrichir votre processus de décision, pas pour remplacer le jugement humain sur des sujets critiques.
{% endhint %}

---

## Le prompt est votre code

Dans un workflow n8n + Claude, la qualité du prompt est la qualité du workflow.

Un prompt flou donne des réponses floues. Un prompt précis — avec un rôle défini, un format de sortie spécifié, des exemples si nécessaire — donne des réponses cohérentes et réutilisables.

Investissez du temps dans la rédaction et le test de vos prompts. C'est là que se joue 80% de la valeur du workflow.

```
Bon prompt pour un workflow :

"Tu es un assistant de classification d'emails B2B.
Analyse l'email suivant et renvoie UNIQUEMENT un JSON avec :
{
  "categorie": "commercial | support | partenariat | spam | autre",
  "urgence": "haute | normale | faible",
  "resume": "une phrase maximum"
}
Ne renvoie rien d'autre que ce JSON."
```

Ce format de prompt — rôle défini, sortie structurée, instruction explicite — est réutilisable pour presque tous vos cas d'usage d'analyse.
