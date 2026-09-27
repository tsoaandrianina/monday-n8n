# Automatisation monday.com + n8n — Exercice Ethanolle

Automatisation qui crée chaque jour, dans un tableau **TODO** monday.com, la tâche correspondant au jour de la semaine actuel, à partir d'un tableau **Template de tâches**. Le système évite les doublons et notifie par email en cas d'échec.

## Sommaire

- [Architecture](#architecture)
- [Structure monday.com](#structure-mondaycom)
- [Workflow principal](#workflow-principal)
- [Workflow de notification d'erreur](#workflow-de-notification-derreur)
- [Installation](#installation)
- [Choix techniques](#choix-techniques)
- [Limites connues et pistes d'amélioration](#limites-connues-et-pistes-damélioration)

## Architecture

Deux workflows n8n distincts :

1. **`workflow-principal.json`** — tourne chaque jour à 8h, lit le tableau template, vérifie si la tâche du jour existe déjà dans TODO, et la crée si nécessaire.
2. **`workflow-notification-erreur.json`** — déclenché automatiquement par n8n si le workflow principal échoue à une étape quelconque ; envoie un email d'alerte.

```
┌─────────────────────┐        en cas d'échec       ┌──────────────────────────┐
│  Workflow principal  │ ───────────────────────────▶│ Workflow notification erreur │
└─────────────────────┘                              └──────────────────────────┘
```

## Structure monday.com

Deux tableaux sont nécessaires côté monday.com :

| Tableau | Groupe | Colonnes | Rôle |
|---|---|---|---|
| **Template de tâches** | Jours de la semaine | Jours (Name), Tache, Statut | Contient une ligne par jour (Lundi → Dimanche), chacune avec une tâche générique |
| **TODO** | Aujourd'hui | Tache, Statut | Reçoit automatiquement la tâche du jour |

## Workflow principal

Étapes exécutées dans l'ordre :

1. **Déclencheur quotidien (8h)** — `Schedule Trigger`, se déclenche tous les jours à l'heure configurée.
2. **Calculer le jour actuel** — nœud `Code` (JavaScript) qui déduit le jour de la semaine en français à partir de la date système.
3. **Récupérer tâches existantes dans TODO** — nœud `monday.com` (Get Many), liste le contenu actuel du groupe "Aujourd'hui".
4. **Récupérer tâche du jour (Template)** — nœud `monday.com` (Get by column value), recherche dans le tableau Template la ligne correspondant au jour calculé à l'étape 2.
5. **Vérifier si tâche déjà présente** — nœud `If`, compare le nom de la tâche du template à la liste récupérée à l'étape 3.
6. **Créer tâche dans TODO** — nœud `monday.com` (Create Item), exécuté uniquement si la tâche n'existe pas encore (branche `true` du IF).

Ce fonctionnement garantit qu'une même tâche n'est jamais créée deux fois, même si le workflow est relancé manuellement plusieurs fois le même jour.

## Workflow de notification d'erreur

- **Error Trigger** — nœud spécial de n8n qui s'active automatiquement quand le workflow principal échoue (à condition d'être référencé dans ses Settings → "Error Workflow").
- **Send an Email** — envoie un email (SMTP Gmail) contenant le nom du workflow en échec, le message d'erreur et le nœud concerné.

## Installation

1. Créer un compte monday.com et les deux tableaux décrits ci-dessus (voir capture `screenshots/` si fournie).
2. Récupérer un jeton API monday.com (Centre de développeurs → Clé API).
3. Importer les deux fichiers `.json` dans n8n (`Import from File`).
4. Configurer les credentials :
   - **monday.com** : coller le jeton API.
   - **SMTP** : identifiants de l'adresse email utilisée pour les alertes (avec mot de passe d'application si Gmail).
5. Dans le workflow principal, aller dans **Settings → Error Workflow** et sélectionner le workflow de notification.
6. Adapter les noms de tableaux/groupes dans les nœuds monday.com si les tiens diffèrent.
7. Publier (activer) le workflow principal.

## Choix techniques

- La colonne native **"Name"** du tableau Template est utilisée pour stocker le jour de la semaine, ce qui simplifie la recherche par valeur de colonne (`Get By Column Value`) sans avoir besoin d'une colonne dédiée.
- La vérification anti-doublon se fait par **comparaison de noms** plutôt que par suppression systématique, pour éviter de perdre une tâche éventuellement modifiée manuellement dans TODO.
- Le paramètre **"Continue (using error output)"** est activé sur certains nœuds pour éviter qu'une absence de résultat (ex: tableau TODO vide) ne bloque l'exécution du reste du flux.

## Limites connues et pistes d'amélioration

- Le flux suppose que les libellés de jours dans le template sont orthographiés exactement comme calculés par le nœud `Code` (sensible à la casse et aux accents) — une validation supplémentaire pourrait être ajoutée.
- Le workflow ne gère actuellement qu'**une seule tâche par jour** ; il pourrait être étendu pour supporter plusieurs tâches par jour via une boucle (`Split in Batches`).
- Une notification de succès (pas seulement d'échec) pourrait être ajoutée pour confirmer visuellement que la tâche du jour a bien été créée.
