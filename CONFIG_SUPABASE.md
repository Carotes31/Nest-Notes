# Config Supabase pour NestNotes Ultimate

L'app utilise **Supabase** (Auth + base de données Postgres + Realtime) pour
synchroniser tes notes entre appareils. Aucun compte Google nécessaire —
juste un email. Gratuit dans les limites du tier Free.

## 1. Créer le compte et le projet

1. Va sur https://supabase.com
2. **Start your project** → crée un compte avec **ton email** (ou GitHub si t'en as un)
3. **New project** :
   - Nom : `nestnotes`
   - Mot de passe de la base de données : génère-en un solide et garde-le de côté
   - Région : `eu-west-1` ou `eu-central-1` (proche de Toulouse)
4. Attends ~2 min que le projet se provisionne

## 2. Créer la table

Dans le dashboard du projet, va dans **SQL Editor** (icône `</>` dans le menu
de gauche) → **New query**, colle ceci puis **Run** :

```sql
create table nestnotes (
  user_id uuid references auth.users(id) on delete cascade primary key,
  notes jsonb default '[]'::jsonb,
  settings jsonb default '{"theme":"default"}'::jsonb,
  updated_at timestamptz default now()
);

-- Active la sécurité au niveau des lignes (RLS)
alter table nestnotes enable row level security;

-- Chaque utilisateur ne peut lire/écrire QUE sa propre ligne
create policy "Users can view own data"
  on nestnotes for select
  using (auth.uid() = user_id);

create policy "Users can insert own data"
  on nestnotes for insert
  with check (auth.uid() = user_id);

create policy "Users can update own data"
  on nestnotes for update
  using (auth.uid() = user_id);
```

Ces règles **RLS** (Row Level Security) sont essentielles : sans elles,
n'importe qui pourrait lire ou écrire les notes de tout le monde.

## 3. Activer le Realtime sur la table

1. Menu **Database** > **Replication**
2. Trouve la table `nestnotes` dans la liste, active le toggle pour qu'elle
   soit incluse dans `supabase_realtime`

(Sans ça, la sync fonctionne quand même au chargement/à la sauvegarde, mais
pas en "live" instantané entre appareils ouverts en même temps.)

## 4. Configurer l'email de confirmation (recommandé : le désactiver)

Par défaut Supabase exige une confirmation par email à l'inscription. Pour un
usage perso/famille, c'est plus simple de la désactiver :

1. Menu **Authentication** > **Providers** > **Email**
2. Désactive **Confirm email**
3. **Save**

Si tu préfères garder la confirmation par email, laisse activé — l'app gère
déjà ce cas (message "vérifie ta boîte mail").

## 5. Récupérer les clés API

1. Menu **Project Settings** (icône engrenage) > **API**
2. Note :
   - **Project URL** (ex: `https://xxxxxxxxxxxx.supabase.co`)
   - **anon public** key (clé longue commençant par `eyJ...`)

⚠️ La clé **anon public** est faite pour être exposée côté client (c'est le
fonctionnement normal de Supabase, protégé par les règles RLS qu'on a mises
en place). Ne prends jamais la clé `service_role`, celle-là doit rester secrète.

## 6. Coller dans index.html

Ouvre `index.html`, cherche tout en haut (vers la ligne 17) :

```js
const SUPABASE_URL = "REPLACE_ME";
const SUPABASE_ANON_KEY = "REPLACE_ME";
```

Remplace par tes vraies valeurs de l'étape 5.

## 7. Déployer

Push/déploie `index.html` (+ les autres fichiers) sur Netlify comme d'habitude.
Au premier chargement tu verras l'écran de connexion. Crée un compte avec
ton email, et tes notes locales existantes seront automatiquement uploadées
vers le cloud au premier login (migration auto, rien à faire).

---

## Comment ça marche côté code

- **Auth** : email/mot de passe via `supabase.auth.signUp` /
  `signInWithPassword`, pas de dépendance Google.
- **Stockage** : une seule ligne par utilisateur dans la table `nestnotes` :
  `{ user_id, notes: [...], settings: {...}, updated_at }`.
- **Sync au login** : au chargement, on récupère la ligne et on remplace
  l'état local par l'état cloud.
- **Sync live (Realtime)** : abonnement Postgres Changes sur la ligne de
  l'utilisateur — si tu modifies une note sur ton PC pendant que ton
  téléphone est ouvert, il reçoit la mise à jour quasi instantanément.
- **Offline-first** : `localStorage` reste utilisé comme cache. Si le réseau
  coupe, tu peux continuer à éditer ; ça se resynchronisera à la reconnexion
  (debounce de 500ms pour ne pas spammer la base à chaque frappe).
- **Indicateur de sync** (point coloré en haut à droite) :
  - 🟢 vert = synchronisé
  - 🟣 violet clignotant = sync en cours
  - 🔴 rouge = hors-ligne / erreur

## Limites du tier gratuit (largement suffisant ici)

- 500 Mo de base de données, 5 Go de bande passante/mois
- 50 000 utilisateurs actifs/mois
- Le projet se met en pause après 1 semaine d'inactivité totale (se réveille
  automatiquement à la prochaine requête, juste un peu plus lent la première fois)

Pour un usage perso ça ne sera jamais un problème.
