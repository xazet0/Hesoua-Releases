# Hésoua

Mise en relation entre locataires et propriétaires pour la location immobilière en Côte d'Ivoire.
Annonces géolocalisées et vérifiées, pass de visite de 24 h payé en Mobile Money, prime au professionnel
dont l'annonce est validée.

Trois documents font autorité : `CAHIER_DES_CHARGES.md` (le quoi), `MVP.md` (le périmètre),
`PLAN_EXECUTION.md` (l'ordre). L'avancement réel est dans `docs/ETAT.md`.

## Pile

Next.js 16 (App Router) · TypeScript strict · Tailwind CSS 4 · Supabase (Postgres + RLS, Auth par
courriel, Storage) · Vitest · Playwright · déploiement Vercel + Supabase Cloud.

## Authentification

La connexion se fait par **adresse électronique et code à six chiffres**, via Supabase Auth. Le
téléphone reste obligatoire au profil : c'est le canal de contact entre locataire et propriétaire
(RM-01), mais il ne sert pas à ouvrir la session.

Deux réglages à faire une fois dans le tableau de bord Supabase, sans quoi personne ne peut se
connecter :

1. _Authentication → Emails → Magic Link_ : le gabarit par défaut n'envoie qu'un lien. Ajoutez-y
   `{{ .Token }}` pour que le code à six chiffres apparaisse dans le message.
2. _Project Settings → Authentication → SMTP Settings_ : le serveur d'envoi intégré de Supabase est
   limité à quelques messages par heure et réservé aux essais. Branchez votre propre SMTP (Resend,
   Brevo, SendGrid…) avant toute ouverture au public.

## Installation

Prérequis : Node.js 18 ou plus, Docker Desktop (pour la base locale), `jq` (utilisé par les hooks).

```bash
npm install
cp .env.example .env.local
```

Renseignez ensuite `.env.local`. Les valeurs locales apparaissent dans la sortie de `npx supabase start`.

```bash
npx supabase start          # démarre Postgres, Auth et Storage en local
npm run db:push             # applique les migrations de supabase/migrations/
npm run db:seed             # trois comptes et dix annonces de démonstration
npm run dev                 # http://localhost:3000
```

Sans base configurée, l'application démarre quand même et affiche un jeu d'annonces de démonstration :
utile pour travailler l'interface, insuffisant pour tester l'authentification ou le paiement.

## Commandes

| Commande            | Rôle                                                           |
| ------------------- | -------------------------------------------------------------- |
| `npm run dev`       | Serveur de développement                                       |
| `npm run build`     | Build de production                                            |
| `npm run lint`      | ESLint                                                         |
| `npm run typecheck` | `tsc --noEmit`                                                 |
| `npm run test`      | Tests unitaires Vitest                                         |
| `npm run test:e2e`  | Parcours Playwright                                            |
| `npm run db:push`   | Applique les migrations Supabase                               |
| `npm run db:seed`   | Jeu de démonstration (refuse de tourner sur une base distante) |

## Organisation

```
app/
  (public)/              liste, fiche annonce, recherche, favoris, profil
  hors-ligne/            page servie par le service worker sans réseau
  api/payments/webhook/  activation du pass (lot 4)
lib/
  domain/                règles métier pures et testées
  supabase/              clients navigateur / serveur / service
  format.ts              FCFA, dates, téléphones
components/
supabase/migrations/     SQL versionné
tests/unit  tests/e2e
```

## Points de sécurité structurants

Ces quatre règles conditionnent la valeur du produit. Elles sont détaillées dans `CLAUDE.md`.

1. **Le pass ne s'active que dans le webhook serveur**, après vérification de signature, de façon
   idempotente sur `payments.provider_ref`. Jamais depuis le client.
2. **RLS activée sur toutes les tables**, avec au moins une politique explicite par table.
3. **Le téléphone du propriétaire et le GPS exact ne quittent jamais le serveur sans pass actif.** Ils
   sont exclus par la vue `v_public_listings`, pas masqués en CSS.
4. **Les pièces d'identité vivent dans un bucket privé**, accessibles seulement par URL signée de cinq
   minutes générée côté serveur pour un administrateur.

Un test Playwright vérifie le point 3 en inspectant toutes les réponses réseau reçues par le navigateur.

## Licence

Projet privé.
