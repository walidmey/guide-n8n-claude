# Installer n8n en 15 minutes

Deux options pour démarrer avec n8n. Choisissez selon votre contexte.

---

## Option A — n8n Cloud (recommandée pour débuter)

La plus simple. Pas de serveur à gérer, pas d'installation technique.

{% stepper %}
{% step %}
### Créer un compte

Allez sur [n8n.io](https://n8n.io) et cliquez "Get started for free". Vous aurez accès à un essai gratuit de 14 jours, puis à un plan Starter à partir de ~20 €/mois.
{% endstep %}
{% step %}
### Accéder à votre instance

Après inscription, vous êtes redirigé vers votre espace de travail n8n. L'URL ressemble à `votre-nom.app.n8n.cloud`.
{% endstep %}
{% step %}
### Explorer l'interface

L'interface est divisée en 3 zones : le canvas (espace de travail visuel), le panneau de nœuds (à gauche, tous les nœuds disponibles), et le panneau de paramètres (à droite, pour configurer le nœud sélectionné).
{% endstep %}
{% endstepper %}

---

## Option B — Auto-hébergement avec Docker (pour les utilisateurs avancés)

Si vous avez un serveur disponible (VPS, Raspberry Pi, serveur d'entreprise) et que vous voulez la gratuité totale ou le contrôle des données.

```bash
# Créer un fichier docker-compose.yml
version: '3.8'
services:
  n8n:
    image: docker.n8n.io/n8nio/n8n
    restart: always
    ports:
      - "5678:5678"
    environment:
      - N8N_BASIC_AUTH_ACTIVE=true
      - N8N_BASIC_AUTH_USER=admin
      - N8N_BASIC_AUTH_PASSWORD=votre-mot-de-passe
      - N8N_HOST=localhost
      - N8N_PORT=5678
    volumes:
      - ~/.n8n:/home/node/.n8n
```

```bash
# Lancer n8n
docker-compose up -d
```

Accédez à n8n via `http://localhost:5678` (ou l'IP de votre serveur).

{% hint style="warning" %}
L'auto-hébergement nécessite que vous gériez vous-même les mises à jour, les sauvegardes, et la disponibilité. Pour un usage professionnel critique, pensez à mettre en place des sauvegardes régulières.
{% endhint %}

---

## Premiers repères dans l'interface

Une fois connecté, vous voyez le tableau de bord. Voici les sections importantes :

**Workflows** — Liste de tous vos workflows. C'est ici que vous créerez, éditerez, et activez vos automations.

**Credentials** — Section clé. C'est ici que vous stockez vos clés API (Gmail, Notion, Claude, etc.). Les credentials sont chiffrées et réutilisables entre les workflows.

**Executions** — L'historique d'exécution de vos workflows. Quand quelque chose ne fonctionne pas, c'est ici que vous diagnostiquez.

**Settings** — Configuration générale, gestion des utilisateurs si vous travaillez en équipe, et abonnement.

---

## Créer votre premier workflow vide

{% stepper %}
{% step %}
### Cliquer sur "New Workflow"

Dans le tableau de bord, cliquez le bouton "+ New Workflow" en haut à droite.
{% endstep %}
{% step %}
### Nommer le workflow

Cliquez sur "My Workflow" en haut à gauche et renommez-le. Exemple : "Test — Premier workflow Claude".
{% endstep %}
{% step %}
### Ajouter un nœud déclencheur

Cliquez sur le "+" dans le canvas ou sur "Add first step". Choisissez "Manual Trigger" pour démarrer — cela vous permet de déclencher le workflow manuellement, ce qui est parfait pour les tests.
{% endstep %}
{% endstepper %}

Vous avez maintenant un workflow avec un seul nœud. Dans le chapitre suivant, vous allez y connecter Claude.
