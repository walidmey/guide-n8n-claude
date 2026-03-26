# Qu'est-ce que n8n ?

n8n (prononcé "n-eight-n") est un outil d'automatisation de workflows. Son principe : relier des applications entre elles et définir des actions qui se déclenchent automatiquement selon des conditions.

La formule de base ressemble à ceci : **"Quand X se produit dans l'application A, fais Y dans l'application B."**

Exemples concrets :
- Quand un nouveau formulaire est soumis sur votre site → créez une entrée dans votre CRM et envoyez un email de bienvenue.
- Quand un client envoie un email avec le mot "urgent" → créez une tâche prioritaire dans Notion et alertez votre équipe sur Slack.
- Chaque lundi à 8h → extrayez les données de la semaine depuis Google Sheets et envoyez un rapport formaté par email.

---

## n8n vs les autres outils d'automatisation

Vous avez peut-être déjà utilisé Zapier, Make (ex-Integromat), ou même des automatisations natives dans votre CRM. Voici ce qui distingue n8n :

{% tabs %}
{% tab title="Zapier" %}
**Zapier** est plus simple à prendre en main, dispose de plus d'intégrations natives (5000+), et est la référence grand public. Mais il devient très cher dès qu'on augmente le volume ou la complexité. Ses "Zaps" sont linéaires — difficile de construire des logiques conditionnelles complexes.

n8n est plus flexible, gratuit en auto-hébergement, et nettement plus puissant pour les logiques complexes.
{% endtab %}
{% tab title="Make (Integromat)" %}
**Make** est le concurrent direct. Interface visuelle, bon équilibre puissance/accessibilité, pricing basé sur les opérations. Pour des utilisateurs avancés sans compétences techniques, c'est souvent un bon choix.

n8n est plus adapté si vous voulez héberger vous-même pour des raisons de coût ou de données sensibles, ou si vous avez des besoins très spécifiques que les outils SaaS ne couvrent pas.
{% endtab %}
{% tab title="n8n" %}
**n8n** est open-source. Vous pouvez l'héberger sur votre propre serveur, payer une version cloud, ou utiliser la version communauté gratuitement. Son modèle de données est plus riche que Zapier, sa gestion des erreurs est plus fine, et son nœud Code vous permet d'injecter du JavaScript si un jour vous en avez besoin.

Principal inconvénient : la courbe d'apprentissage est légèrement plus élevée que Zapier, et la bibliothèque d'intégrations natives est plus petite (mais en croissance rapide, et l'API générique couvre la plupart des besoins).
{% endtab %}
{% endtabs %}

---

## Les concepts de base de n8n

Pour utiliser n8n, vous avez besoin de comprendre trois concepts. Pas plus.

### 1. Le nœud (node)

Un nœud est un bloc dans votre workflow. Chaque nœud fait une chose : déclencher le workflow, lire des données depuis une source, transformer des données, appeler une API, écrire dans une application.

Exemples de nœuds : Gmail (lire/envoyer des emails), Notion (créer/modifier des pages), HTTP Request (appeler n'importe quelle API), Code (transformer des données en JavaScript), Claude (envoyer une requête à Claude et récupérer la réponse).

### 2. La connexion

Les nœuds sont reliés entre eux par des connexions. Les données "coulent" de gauche à droite. La sortie d'un nœud devient l'entrée du suivant.

### 3. Le déclencheur (trigger)

Chaque workflow commence par un déclencheur. Ce peut être : un événement (un email reçu, un formulaire soumis, une ligne ajoutée dans un Google Sheet), un horaire (chaque matin à 7h, chaque lundi), ou un webhook (un appel HTTP d'une autre application).

---

## Ce que n8n ne fait pas

n8n orchestre et connecte. Il ne pense pas, ne comprend pas le contenu, ne prend pas de décisions contextuelles.

Si votre workflow doit "lire cet email et décider si c'est une opportunité commerciale ou une demande support", n8n seul ne peut pas. Il faut un modèle de langage pour ça — c'est là qu'intervient Claude.

{% hint style="success" %}
La règle simple : n8n gère **quand** et **comment** les données circulent. Claude gère **ce que les données signifient** et **quelle action décider en conséquence**.
{% endhint %}
