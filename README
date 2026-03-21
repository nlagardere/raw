# RAW

> *Post raw. Nothing else.*

Un réseau social pour lycéens basé sur une question quotidienne. Pas de likes, pas de followers, pas d'algo. Juste une photo, une émotion, et une question imposée à tous.

---

## Pour les utilisateurs 

### C'est quoi RAW ?

Chaque jour, tout le monde reçoit la même question bizarre et inattendue. Tu y réponds avec une photo et une émotion. C'est tout. Tes posts sont anonymes — les autres voient ta photo mais ne savent pas que c'est toi. Si quelqu'un est touché par ta photo, il peut te envoyer une demande de connexion. C'est toi qui choisis ce que tu révèles : ton Instagram, ton Snap, ton numéro, ou rien du tout.

À minuit tout disparaît. Demain, nouvelle question.

### Comment utiliser l'appli

**Créer un compte**
Ouvre l'appli dans ton navigateur. Choisis un pseudo, entre le nom de ton lycée, et crée un mot de passe. C'est tout, pas d'email nécessaire.

**Poster**
Va dans l'onglet POSTER. Lis la question du jour, prends ou choisis une photo qui y répond, sélectionne l'émotion qui colle le mieux, et appuie sur "Poster anonymement". Tu ne peux poster qu'une seule fois par jour.

**Explorer le feed**
L'onglet FEED affiche toutes les photos du jour dans l'ordre chronologique. Filtre par émotion pour voir uniquement les posts "joie", "rage", "calme", "tristesse", "bizarre" ou "hype".

**Connecter avec quelqu'un**
Si une photo te touche, appuie sur CONNECT. La personne reçoit une demande anonyme. Elle choisit si elle accepte, et via quelle plateforme (Instagram, Snapchat, numéro ou message anonyme).

**Voter pour demain**
Dans l'onglet CONNECT, tu peux voter pour la question de demain parmi 3 options. La plus votée gagne.

**Supprimer ton post**
Si tu veux retirer ton post, il apparaît dans le feed avec un bouton "SUPPRIMER" en rouge. La suppression est immédiate et définitive.

### Les 6 émotions

| Émotion | Couleur | Ce que ça veut dire |
|---------|---------|---------------------|
| JOIE | Jaune | T'es bien, léger, content |
| RAGE | Rouge | Quelque chose t'énerve ou t'indigne |
| CALME | Bleu | Posé, serein, dans ta bulle |
| TRISTESSE | Violet | Mélancolie, nostalgie, quelque chose pèse |
| BIZARRE | Vert | Décalé, inattendu, pas de case |
| HYPE | Orange | Excité, speed, à fond |

---

## Pour les développeurs

### Stack technique

- **Frontend** : HTML / CSS / JavaScript vanilla — un seul fichier `raw-app.html`
- **Backend** : [Supabase](https://supabase.com) (PostgreSQL + Storage + Realtime)
- **Fonts** : Bebas Neue + Space Mono via Google Fonts
- **Auth** : Système maison simplifié (pas Supabase Auth) — les users sont stockés dans la table `users`, le mot de passe est encodé en base64 dans le localStorage

### Structure du fichier HTML

```
raw-app.html
├── <style>          CSS complet (variables, composants, screens)
├── Screens HTML
│   ├── #loader      Écran de chargement initial
│   ├── #s-auth      Login + inscription
│   ├── #s-feed      Feed principal
│   ├── #s-post      Formulaire de post
│   └── #s-connect   Liste des posts + vote
├── .bnav            Navigation bas de page
├── #modal-connect   Modal de connexion avec quelqu'un
├── #modal-vote      Modal de vote pour demain
└── <script>
    ├── Config       SUPABASE_URL + SUPABASE_KEY
    ├── Constants    EMO, BGS, SYM
    ├── State        currentUser, currentQuestion, feedPosts...
    ├── init()       Point d'entrée — vérifie session localStorage
    ├── Auth         doLogin(), doRegister(), showRegister(), showLogin()
    ├── loadApp()    Charge question + posts + realtime après login
    ├── Feed         loadFeed(), renderFeed()
    ├── Realtime     setupRealtime() — écoute les nouveaux posts
    ├── Upload       FileReader + Canvas compression + Supabase Storage
    ├── submitPost() Upload photo → Storage → insert posts table
    ├── deletePost() Delete Storage + delete posts table
    ├── Connect      openConnectModal(), doConnect(), loadConnectList()
    ├── Vote         openVoteModal(), castVote()
    ├── Nav          goTo(), goScreen(), closeModal()
    ├── Countdown    startCountdown() — compte à rebours jusqu'à minuit
    └── Utils        timeSince(), showToast()
```

### Base de données Supabase

#### Table `users`
```sql
id          uuid  PRIMARY KEY
created_at  timestamptz
username    text  UNIQUE NOT NULL
school      text  NOT NULL
```

#### Table `questions`
```sql
id          uuid  PRIMARY KEY
created_at  timestamptz
text        text  NOT NULL
date        date  UNIQUE NOT NULL
```
Une question par jour. La date correspond à `current_date`. Le système charge la question dont `date = aujourd'hui`.

#### Table `posts`
```sql
id           uuid  PRIMARY KEY
created_at   timestamptz
user_id      uuid  FK → users.id
question_id  uuid  FK → questions.id
photo_url    text  NOT NULL
emotion      text  CHECK IN (joy, rage, calm, sad, weird, hype)
expires_at   timestamptz  DEFAULT now() + 48h
```
Les posts expirent au bout de 48h (colonne `expires_at`). La RLS policy filtre automatiquement les posts expirés via `expires_at > now()`.

#### Table `votes`
```sql
id             uuid  PRIMARY KEY
created_at     timestamptz
user_id        uuid  FK → users.id
question_text  text  NOT NULL
vote_date      date  DEFAULT current_date
UNIQUE(user_id, vote_date)
```
Un seul vote par user par jour, garanti par la contrainte UNIQUE.

#### Table `connections`
```sql
id            uuid  PRIMARY KEY
created_at    timestamptz
from_user_id  uuid  FK → users.id
to_post_id    uuid  FK → posts.id
platform      text  (instagram, snapchat, numero, message)
status        text  DEFAULT 'pending' CHECK IN (pending, accepted, declined)
UNIQUE(from_user_id, to_post_id)
```

### Storage Supabase

Bucket : `photos` (public)

Les fichiers sont nommés `{user_id}_{timestamp}.jpg`. Avant l'upload, les images sont compressées côté client via Canvas API : redimensionnement à max 800px, qualité JPEG 75%. Cela réduit la taille d'environ 80% et maintient le bucket dans les limites du free tier (1GB).

### Row Level Security (RLS)

Toutes les tables ont RLS activé. Les policies actuelles sont permissives (anon peut tout lire et insérer) ce qui est adapté pour un proto. En production il faudrait basculer sur Supabase Auth et restreindre les writes à `auth.uid() = user_id`.

### Realtime

Le feed écoute les `INSERT` sur la table `posts` via Supabase Realtime :

```javascript
sb.channel('posts-feed')
  .on('postgres_changes', {event: 'INSERT', schema: 'public', table: 'posts'}, payload => {
    feedPosts.unshift(payload.new);
    renderFeed();
  })
  .subscribe();
```

Les nouveaux posts apparaissent instantanément pour tous les utilisateurs connectés sans reload.

### Limites du free tier Supabase

| Ressource | Limite | Usage estimé |
|-----------|--------|--------------|
| Database | 500 MB | ~1M posts texte |
| Storage | 1 GB | ~6000 photos (compressées 150kb) |
| Bandwidth | 5 GB/mois | ~33 000 téléchargements de photos |
| MAU | 50 000 | Largement suffisant au lancement |

Avec la suppression automatique des posts à 48h, le storage reste constant et ne s'accumule jamais.

### Ajouter des questions

Via le SQL Editor de Supabase :

```sql
insert into questions (text, date) values
  ('Ta question ici', '2026-03-21');
```

Ou via l'interface Table Editor de Supabase directement.

### Déploiement

Le fichier `raw-app.html` est autonome — il suffit de l'héberger n'importe où :

- **GitHub Pages** : push le fichier dans un repo public, active Pages
- **Netlify** : drag & drop du fichier sur netlify.com/drop
- **Vercel** : `vercel deploy`

Aucun serveur backend nécessaire, tout passe par Supabase.

---

## Roadmap

- [ ] Vraie auth Supabase (email/password) pour plus de sécurité
- [ ] Système de révélation côté destinataire (accepter/refuser une connexion)
- [ ] Suppression automatique des photos expirées via Edge Function
- [ ] Notifications push quand quelqu'un veut te connecter
- [ ] Page lycée — voir les questions des jours passés (sans les photos)
- [ ] Mode PWA pour installer l'appli sur l'écran d'accueil

---

*Fait par Nathan, lycée Camille Jullian, Bordeaux — 2026*
