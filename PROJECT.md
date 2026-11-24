
# NETSCAN : Application de bureau portable de détection des appareils réseau

> Ce fichier est la source de vérité du projet. L'IA doit le relire au début de chaque session et le respecter strictement.

---

## 1. RÔLE

Tu es un ingénieur logiciel senior (Python, réseau, UI desktop, packaging). Tu construis avec moi, phase par phase, une application de bureau PORTABLE qui détecte et affiche les appareils connectés à mon réseau local.

## 2. OBJECTIF

Une application desktop 100 % locale, livrée en un seul exécutable portable.

- AUCUN backend distant, AUCUN compte, AUCUNE télémétrie
- AUCUN téléchargement à l'exécution : tout est embarqué (base OUI, polices, icônes)
- AUCUNE dépendance externe à installer (pas de nmap requis)
- Le dossier `data/` (base, config, logs) est stocké à côté de l'exécutable : l'app est copiable sur clé USB

Cibles : **Windows 10/11 et Linux** (macOS hors périmètre pour l'instant).
Langues de l'interface : **français et anglais** (français par défaut).

## 3. STACK IMPOSÉE

| Domaine          | Choix                                                                                                |
| ---------------- | ---------------------------------------------------------------------------------------------------- |
| Langage          | Python 3.11+                                                                                         |
| Interface        | PySide6 (Qt6)                                                                                        |
| Base de données | SQLite (module `sqlite3` standard)                                                                 |
| Réseau          | modules standard (`socket`, `subprocess`, `ipaddress`) + `zeroconf` (mDNS) ; Scapy OPTIONNEL |
| Concurrence      | `concurrent.futures.ThreadPoolExecutor` + `QThread`/workers Qt                                   |
| Empaquetage      | PyInstaller (onefile) ou Nuitka                                                                      |
| Tests            | pytest, pytest-qt                                                                                    |
| Qualité         | ruff, mypy (strict), type hints partout                                                              |
| CI               | GitHub Actions (build Windows + Linux)                                                               |

Si tu veux changer la stack, DEMANDE-MOI d'abord.

## 4. ARCHITECTURE (séparation stricte)

```
netscan/
  PROJECT.md
  README.md
  LICENSE
  CHANGELOG.md
  pyproject.toml
  requirements.txt
  main.py
  app/
    __init__.py
    version.py
    core/                 # moteur réseau, AUCUN import Qt
      interfaces.py       # détection interfaces / sous-réseau
      scanner.py          # orchestration du scan
      probes.py           # sonde portable (UDP/TCP) + lecture table ARP
      arp_raw.py          # mode avancé Scapy (optionnel)
      resolvers.py        # DNS inverse, mDNS, NetBIOS
      oui.py              # fabricant à partir de la MAC
      identity.py         # identité d'appareil (voir section 7)
      anomalies.py        # détection d'anomalies simples
    storage/
      db.py               # connexion, migrations
      migrations/         # 001_init.sql, 002_...
      repository.py       # accès aux données (requêtes paramétrées)
    services/
      scan_service.py     # scan manuel, annulation, progression
      scheduler.py        # scan périodique (V2)
      notifier.py         # notifications locales (V2)
      export_service.py   # CSV / JSON
      settings_service.py
    ui/
      main_window.py
      widgets/
      dialogs/            # avertissement, paramètres, diagnostic
      theme/              # tokens couleurs, QSS sombre/clair
      i18n/               # fichiers de traduction fr/en
    assets/
      oui.csv             # base IEEE embarquée
      fonts/
      icons/
    paths.py              # chemins portables (data/ à côté de l'exe)
    logging_setup.py
  data/                   # créé au premier lancement
    netscan.db
    config.json
    logs/
  tests/
    unit/
    integration/
  .github/workflows/build.yml
```

Règles :

- `ui` n'appelle que `services`. `services` appelle `core` et `storage`. `core` ne connaît ni `ui` ni `services`.
- Les chemins passent TOUS par `paths.py` (gère le mode PyInstaller via `sys._MEIPASS` pour les assets et le dossier de l'exécutable pour `data/`).

## 5. FONCTIONNALITÉS

### V1 (obligatoire)

1. Détection automatique de l'interface active et du sous-réseau (choix manuel si plusieurs interfaces)
2. Scan de tous les appareils : IP, MAC, fabricant (OUI), nom d'hôte
3. **Mode portable (sans droits admin, sans pilote)** : envoi d'un paquet UDP/TCP léger à chaque IP du sous-réseau, puis lecture de la table ARP de l'OS (`/proc/net/arp` ou `ip neigh` sous Linux ; `arp -a` sous Windows)
4. **Mode avancé (optionnel)** : vrai scan ARP via Scapy si privilèges (Linux) ou Npcap (Windows) présents ; bascule automatique vers le mode portable sinon
5. Résolution de noms : DNS inverse, mDNS (zeroconf), NetBIOS si possible
6. Scan concurrent avec barre de progression et bouton Annuler (annulation réelle, immédiate)
7. Tableau avec tri, recherche, filtres (en ligne / hors ligne / nouveaux / connus)
8. Export CSV et JSON
9. Historique SQLite : première apparition, dernière apparition, statut
10. Écran d'avertissement légal au premier lancement (voir section 10)
11. Écran Paramètres et écran Diagnostic (voir sections 8 et 9)

### V2 (après validation explicite de la V1)

1. Scan périodique en arrière-plan avec icône dans la barre système
2. Notification locale à l'apparition d'un appareil INCONNU
3. Liste de confiance : nommer et marquer un appareil « connu »
4. Détection d'anomalies simples (conflit d'IP, MAC qui change d'IP très souvent, deux MAC pour la même IP)
5. Mode ligne de commande qui réutilise le même moteur (`python main.py --scan --json`)

### Hors périmètre / à discuter plus tard (NE PAS CODER sans mon accord)

- Détection du type d'appareil, ports ouverts
- Carte visuelle du réseau
- Rapports PDF/HTML
- API locale optionnelle sur 127.0.0.1
- Système de plugins
- Détection d'anomalies par machine learning
- Alertes externes (Telegram, e-mail)
- Support macOS

## 6. MODÈLE DE DONNÉES (SQLite)

Utiliser des migrations versionnées (`PRAGMA user_version`). `PRAGMA foreign_keys = ON`, mode `WAL`.

```sql
-- 001_init.sql
CREATE TABLE devices (
  id            INTEGER PRIMARY KEY AUTOINCREMENT,
  identity_key  TEXT    NOT NULL UNIQUE,   -- voir section 7
  display_name  TEXT,                      -- nom donné par l'utilisateur
  hostname      TEXT,
  vendor        TEXT,
  is_trusted    INTEGER NOT NULL DEFAULT 0 CHECK (is_trusted IN (0,1)),
  is_random_mac INTEGER NOT NULL DEFAULT 0 CHECK (is_random_mac IN (0,1)),
  notes         TEXT,
  first_seen    TEXT    NOT NULL,          -- ISO 8601 UTC
  last_seen     TEXT    NOT NULL
);

CREATE TABLE device_addresses (
  id         INTEGER PRIMARY KEY AUTOINCREMENT,
  device_id  INTEGER NOT NULL REFERENCES devices(id) ON DELETE CASCADE,
  ip         TEXT    NOT NULL,
  mac        TEXT    NOT NULL,
  first_seen TEXT    NOT NULL,
  last_seen  TEXT    NOT NULL,
  UNIQUE (device_id, ip, mac)
);

CREATE TABLE scans (
  id           INTEGER PRIMARY KEY AUTOINCREMENT,
  started_at   TEXT NOT NULL,
  finished_at  TEXT,
  subnet       TEXT NOT NULL,
  mode         TEXT NOT NULL CHECK (mode IN ('portable','arp_raw')),
  status       TEXT NOT NULL CHECK (status IN ('running','completed','cancelled','failed')),
  devices_found INTEGER NOT NULL DEFAULT 0,
  error        TEXT
);

CREATE TABLE events (
  id         INTEGER PRIMARY KEY AUTOINCREMENT,
  device_id  INTEGER REFERENCES devices(id) ON DELETE SET NULL,
  scan_id    INTEGER REFERENCES scans(id) ON DELETE SET NULL,
  type       TEXT NOT NULL CHECK (type IN
             ('appeared','disappeared','ip_changed','mac_changed','ip_conflict','new_unknown')),
  details    TEXT,
  created_at TEXT NOT NULL
);

CREATE TABLE settings (
  key   TEXT PRIMARY KEY,
  value TEXT NOT NULL
);

CREATE INDEX idx_addr_ip   ON device_addresses(ip);
CREATE INDEX idx_addr_mac  ON device_addresses(mac);
CREATE INDEX idx_events_dt ON events(created_at);
```

Règles : requêtes paramétrées uniquement, transactions pour toute écriture multiple, dates en UTC ISO 8601, sauvegarde automatique de la base avant toute migration.

## 7. IDENTITÉ DES APPAREILS (MAC aléatoires)

Les téléphones modernes utilisent des MAC aléatoires (bit « localement administré » = 1, c'est-à-dire le 2e chiffre hexadécimal du premier octet vaut 2, 6, A ou E). Un même téléphone peut donc changer de MAC.

Algorithme de `identity_key` :

1. Si la MAC est **globale** (bit localement administré = 0) : `identity_key = "mac:<MAC normalisée>"`
2. Si la MAC est **aléatoire** : marquer `is_random_mac = 1` et tenter de relier à un appareil existant dans cet ordre :
   a. même nom mDNS/NetBIOS/hostname non vide
   b. même IP stable (bail DHCP) ET même nom d'hôte
   c. sinon : `identity_key = "rnd:<MAC>"` (nouvel appareil)
3. Ne JAMAIS fusionner deux appareils sans critère fort. En cas de doute, créer un nouvel appareil et laisser l'utilisateur les fusionner manuellement.
4. Normalisation MAC : majuscules, séparateur `:`.

À tester avec des cas unitaires : MAC globale, MAC aléatoire avec nom identique, MAC aléatoire sans nom, changement d'IP, deux appareils au même hostname.

## 8. PARAMÈTRES (écran Paramètres)

| Clé                       | Défaut   | Description                           |
| -------------------------- | --------- | ------------------------------------- |
| `language`               | `fr`    | `fr` ou `en`                      |
| `theme`                  | `dark`  | `dark` ou `light`                 |
| `interface`              | `auto`  | interface réseau à scanner          |
| `subnet_override`        | vide      | CIDR manuel (validé)                 |
| `scan_mode`              | `auto`  | `auto`, `portable`, `arp_raw`   |
| `scan_timeout_ms`        | `800`   | timeout par hôte                     |
| `max_concurrency`        | `64`    | threads simultanés (borné 8 à 256) |
| `auto_scan_enabled`      | `false` | scan périodique (V2)                 |
| `auto_scan_interval_min` | `10`    | intervalle (V2), min 1                |
| `history_retention_days` | `90`    | purge automatique                     |
| `notify_unknown`         | `true`  | notification appareil inconnu (V2)    |
| `legal_accepted`         | `false` | avertissement accepté                |

Toute valeur est validée et bornée. Une valeur invalide revient au défaut avec un avertissement dans les logs.

## 9. ÉCRAN DIAGNOSTIC

Affiche : version de l'app, OS, version Python/Qt, interface et sous-réseau détectés, mode de scan réellement utilisé (et pourquoi), présence de privilèges / Npcap, chemin du dossier `data/`, taille de la base, dernier scan et sa durée. Boutons : « Copier le diagnostic » et « Exporter les logs (zip) ».

## 10. ÉTHIQUE ET SÉCURITÉ

- L'app ne scanne QUE le sous-réseau local de la machine (refuser tout CIDR public ou hors des plages privées RFC 1918 / lien local ; refuser aussi un masque plus large que /16)
- Au premier lancement : écran obligatoire avec case à cocher : « J'utilise cet outil uniquement sur un réseau dont je suis propriétaire ou pour lequel j'ai une autorisation. » Tant que non accepté, aucun scan n'est possible
- Aucune donnée envoyée sur Internet. Aucune télémétrie. Aucun appel réseau sortant hors du sous-réseau local
- Aucune exécution de commande construite à partir d'une entrée non validée (`subprocess` avec liste d'arguments, jamais `shell=True`)
- Aucun secret en clair ; pas de mot de passe stocké

## 11. DESIGN (attractif et moderne)

- Thème sombre par défaut + thème clair, bascule en un clic. Couleurs centralisées dans des tokens (un seul fichier), générant le QSS
- Style sobre et professionnel : coins arrondis (8 à 12 px), espacement généreux, une couleur d'accent, police Inter (ou équivalent) embarquée
- Tableau clair : lignes aérées, survol, sélection visible, colonnes redimensionnables et mémorisées
- Pastilles de statut : vert = en ligne, gris = hors ligne, orange = nouvel appareil, rouge = anomalie
- Barre latérale (Appareils, Historique, Paramètres, Diagnostic) + barre d'état avec résumé du dernier scan
- États soignés : vide (« Aucun scan pour l'instant »), chargement, erreur lisible par un humain avec action proposée
- JAMAIS de fenêtre figée : tout travail réseau et base de données lourd dans des workers
- Fenêtre redimensionnable, minimum 900x600, support des écrans HiDPI
- Accessibilité : contraste suffisant, navigation clavier, raccourcis (Ctrl+R scan, Ctrl+F recherche, Échap annule)

## 12. PERFORMANCE

- Un sous-réseau /24 scanné en **moins de 10 secondes** en mode portable sur un réseau normal
- Concurrence bornée (`max_concurrency`) pour ne pas saturer un petit routeur
- Démarrage de l'app en moins de 3 secondes
- Tableau fluide jusqu'à 1000 appareils (modèle/vue Qt, pas de widgets par ligne)
- Mémoire stable lors de scans répétés (pas de fuite de threads)

## 13. SOLIDITÉ ET GESTION D'ERREURS

- Aucun `except:` nu : exceptions précises, journalisées (`logging`, fichier dans `data/logs/`, rotation 5 x 1 Mo)
- Validation de toutes les entrées (CIDR, timeouts, valeurs de paramètres)
- Timeout sur toute opération réseau, aucune boucle infinie
- Échec partiel d'un scan : afficher les résultats obtenus + avertissement, jamais de crash
- Écritures de base atomiques ; aucune perte de données à la fermeture brutale
- Arrêt propre : threads stoppés, connexions fermées, scan annulé
- Cas à gérer explicitement : pas de réseau, plusieurs interfaces, VPN actif, droits insuffisants, Npcap absent, pare-feu qui bloque, base verrouillée ou corrompue (sauvegarde + recréation avec confirmation), disque plein, OS non supporté
- Gestionnaire global d'exceptions Qt : message clair à l'utilisateur + trace dans les logs, jamais de fermeture silencieuse

## 14. SPÉCIFICITÉS PAR OS

**Linux**

- Mode portable : lecture de `/proc/net/arp` (sinon `ip neigh`)
- Mode avancé : nécessite `cap_net_raw` ; ne JAMAIS lancer toute l'app en root. Documenter `setcap` dans le README
- Détection d'interface via `/sys/class/net` ou `ip -4 addr`

**Windows**

- Mode portable : `arp -a` (analyse robuste, sortie localisée selon la langue de l'OS : ne pas dépendre de mots, seulement du format IP/MAC)
- Mode avancé : nécessite Npcap installé ; si absent, rester en mode portable et l'indiquer dans Diagnostic
- Gérer les caractères/encodages de la console Windows (`subprocess` avec `encoding` explicite)
- Signature/SmartScreen : documenter l'avertissement possible à l'exécution de l'exe non signé

## 15. TESTS ET QUALITÉ

- Tests unitaires pour `core`, `storage`, `identity`, `oui`, `paths`, `settings`
- Appels réseau et `subprocess` MOCKÉS dans les tests
- Tests d'intégration : scan contre un faux réseau simulé (réponses ARP simulées), migrations sur une base vide et sur une base d'une ancienne version
- Tests UI avec pytest-qt (lancement, thème, annulation d'un scan)
- `ruff check`, `ruff format --check`, `mypy --strict` sans erreur
- Couverture visée : plus de 80 % sur `core`, `storage` et `services`
- Test de l'exécutable final sur une machine propre (sans Python installé)

## 16. INTERNATIONALISATION

- Toutes les chaînes de l'interface passent par le système de traduction Qt (`tr`) ; aucune chaîne en dur
- Fichiers `fr` et `en` fournis ; changement de langue sans réinstallation (redémarrage de l'app accepté)
- Dates et heures affichées dans le fuseau local, stockées en UTC

## 17. PROCESSUS DE TRAVAIL (TRÈS IMPORTANT)

Tu travailles par PHASES. À la fin de chaque phase, tu t'ARRÊTES et tu attends mon « OK » explicite.

| Phase | Contenu                                                                                           |
| ----- | ------------------------------------------------------------------------------------------------- |
| 0     | Plan : résume ta compréhension, risques, questions ouvertes                                     |
| 1     | Structure du projet, venv, requirements, config ruff/mypy/pytest, `paths.py`, logs              |
| 2     | Moteur de scan (`core`) en mode portable + tests                                                |
| 3     | Stockage SQLite, migrations, repository, identité des appareils + tests                          |
| 4     | Services (scan, annulation, historique, export, paramètres)                                      |
| 5     | Interface V1 (thèmes, tableau, progression, paramètres, diagnostic, avertissement légal, i18n) |
| 6     | Mode avancé Scapy optionnel + bascule automatique                                                |
| 7     | Empaquetage portable + CI Windows/Linux + test sur machine propre                                 |
| 8     | Revue V1 : checklist qualité, README, limites connues. STOP : validation avant la V2             |
| 9     | V2 (scan périodique, notifications, liste de confiance, anomalies, CLI)                          |
| 10    | Revue finale, CHANGELOG, version 1.0                                                              |

## 18. CONFIRMATIONS OBLIGATOIRES

Avant toute action importante, tu m'expliques en 2 ou 3 lignes CE QUE tu vas faire et POURQUOI, puis tu attends « oui ». Sont importantes :

- installer un paquet ou ajouter une dépendance
- créer, supprimer ou écraser un fichier existant
- modifier le schéma de la base (nouvelle migration)
- lancer un scan réel sur le réseau
- exécuter une commande système ou nécessitant des privilèges
- changer l'architecture, la stack ou le périmètre
- toute action irréversible (suppression de données, purge, `git reset`, etc.)

En cas de doute, DEMANDE.

## 19. RÈGLES ANTI-ERREUR

- Ne jamais inventer une API, une option ou une fonction de bibliothèque. Si tu n'es pas sûr, dis-le et propose de vérifier la documentation officielle
- Donne des fichiers COMPLETS et exécutables, jamais de « ... » ni de morceaux à compléter
- Indique à chaque fois comment lancer et tester ce que tu viens d'écrire
- Après chaque phase, exécute (ou donne) : `ruff check`, `mypy`, `pytest`
- En cas d'erreur : explique la cause, propose la correction, attends mon OK, puis corrige
- Ne réécris pas du code qui fonctionne sans raison ; modifications minimales, signalées
- Reste cohérent avec les noms, types et structures déjà définis dans ce fichier
- Relis ce `PROJECT.md` au début de chaque session et ne contredis jamais ses règles
- N'ajoute aucune fonctionnalité de la section « Hors périmètre » sans mon accord

## 20. FORMAT DE RÉPONSE

- Réponds en français ; code, noms de variables et commentaires en anglais
- Pour chaque phase : (1) ce qui a été fait, (2) fichiers créés/modifiés, (3) comment tester, (4) points d'attention, (5) la confirmation demandée pour continuer

## 21. CRITÈRES D'ACCEPTATION FINALE

- [ ] L'exécutable se lance sur une machine propre, sans rien installer
- [ ] Le scan trouve les appareils de mon réseau sans droits admin (mode portable)
- [ ] L'interface ne se fige jamais pendant un scan ; l'annulation est immédiate
- [ ] Un /24 est scanné en moins de 10 secondes
- [ ] Un même téléphone à MAC aléatoire n'apparaît pas en doublon dans les cas couverts
- [ ] L'écran d'avertissement bloque le scan tant qu'il n'est pas accepté
- [ ] Aucun appel réseau sortant hors du sous-réseau local
- [ ] Tous les tests passent ; ruff et mypy sans erreur
- [ ] Le dossier `data/` suit l'exécutable (copiable sur clé USB)
- [ ] README, LICENSE, CHANGELOG et diagnostic exportable présents
- [ ] Interface complète en français et en anglais

---

**Commence par la Phase 0 : résume ta compréhension, liste tes risques et tes questions, puis attends mon OK.**
