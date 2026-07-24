# Framework d'Audit de Sécurité Automatisé

Framework d'audit et de test d'intrusion **automatisé** pour applications web, orchestrant une douzaine d'outils Kali Linux dans un pipeline unique, avec une interface terminal et un **rapport final JSON + PDF**.

> ⚠️ **Usage strictement autorisé.** N'utilisez ce framework que sur des systèmes que vous êtes **explicitement autorisé** à tester (votre propre lab, une cible d'entraînement type OWASP Juice Shop, ou un périmètre de pentest contractualisé). Tout scan non autorisé est illégal.
>
> Deux garde-fous techniques bloquent tout audit sur une cible non autorisée : une **allowlist obligatoire** ([`cibles_autorisees.txt`](pentest_framework/cibles_autorisees.txt)) et une **attestation utilisateur bloquante** au démarrage. Voir la section [Garde-fous](#-garde-fous-autorisation-des-cibles).

---

## 🚀 Lancement en 1 commande

```bash
cd pentest_framework
bash ./lanceur.sh <cible>
```

Exemples :

```bash
bash ./lanceur.sh localhost:3000        # cible locale avec port
bash ./lanceur.sh http://127.0.0.1:3000 # URL complète
bash ./lanceur.sh exemple.com           # domaine (énumération de sous-domaines)
```

Le lanceur s'occupe de **tout** : installation des dépendances, création de l'environnement Python, permissions, puis exécution. Aucune étape manuelle.

> 💡 Pour OWASP Juice Shop, visez son adresse réelle (`localhost:3000`), pas un simple nom d'hôte non résoluble.

---

## 📋 Prérequis

- **Kali Linux** (ou distribution disposant des outils de pentest via `apt`).
- Droits **`sudo`** (installation des paquets).
- **Docker** démarré pour le scan ZAP : `sudo systemctl start docker`.
- Accès Internet (installation des outils + templates Nuclei).

---

## 🏗️ Architecture

Le framework est organisé en **3 couches** qui communiquent par fichiers (`.txt` pour les cibles/URLs, `.json` pour les résultats) :

```
lanceur.sh ──► orchestrateur.py ──► wrap_*.sh ──► outils Kali
  (setup)       (cerveau + UI)      (adaptateurs)   (nmap, nuclei…)
                     ▲                    │
                     └──── fichiers JSON ─┘
```

- **`lanceur.sh`** — point d'entrée : installe les dépendances (apt + venv Python), fixe les permissions, lance l'orchestrateur.
- **`orchestrateur.py`** — le chef d'orchestre : enchaîne les phases, **parallélise** les outils indépendants (pour tenir sous ~5 min), appelle les wrappers, parse les résultats et génère les rapports. Interface terminal via **Rich**.
- **`wrap_*.sh`** — un adaptateur par outil : chacun vérifie/installe son outil, le lance avec des options bornées par `timeout`, et produit une **sortie JSON normalisée**.

---

## 🔬 Pipeline d'analyse

1. **Reconnaissance** — `subfinder` (sous-domaines) → `nmap` (ports/services) → `httpx` (URLs réellement actives).
2. **Sélection interactive** — choix des services à auditer + confirmation.
3. **Analyse parallèle** — `whatweb` (technos), `testssl` (TLS), chaîne `katana → ffuf → arjun` (crawl, endpoints cachés, paramètres), `nuclei` (vulnérabilités), `zap` (DAST).
4. **Audits ciblés** — `sqlmap` (injection SQL), `hydra` (brute-force d'authentification).
5. **Restitution** — normalisation, synthèse terminal, **rapport JSON + PDF**.

---

## 🧰 Outils intégrés

| Phase | Outils |
|------|--------|
| Reconnaissance | subfinder, nmap, httpx |
| Cartographie web | whatweb, testssl, katana, ffuf, arjun |
| Vulnérabilités | nuclei, OWASP ZAP |
| Exploitation ciblée | sqlmap, hydra |

Tous sont open-source et installés automatiquement par le lanceur.

---

## 📄 Rapports générés

À la fin du scan, dans le dossier `pentest_framework/` :

- **`rapport_final.json`** — rapport structuré : métadonnées (cible, date), résumé par sévérité, et le détail de chaque résultat (`Outil_Source`, `Nom_Vuln`, `Risque`, `Cible_URL`, `Description`).
- **`rapport_final.pdf`** — rapport lisible : synthèse par sévérité et détails groupés (généré via `fpdf2`).

Chaque résultat est classé par niveau de risque : **Critical / High / Medium / Low / Info**.

---

## 📁 Structure du projet

```
pentest_framework/
├── lanceur.sh           # Point d'entrée (1 commande)
├── orchestrateur.py     # Orchestration + UI + parsing + rapports
├── wrap_subfinder.sh    # Énumération de sous-domaines
├── wrap_nmap.sh         # Scan de ports/services
├── wrap_httpx.sh        # Probe HTTP/HTTPS
├── wrap_whatweb.sh      # Fingerprint technologique
├── wrap_testssl.sh      # Audit TLS/SSL
├── wrap_katana.sh       # Crawl JS-aware (SPA)
├── wrap_ffuf.sh         # Brute-force de répertoires
├── wrap_arjun.sh        # Découverte de paramètres HTTP
├── wrap_nuclei.sh       # Scan de vulnérabilités par templates
├── wrap_zap.sh          # DAST dynamique (OWASP ZAP)
├── wrap_sqlmap.sh       # Injection SQL
└── wrap_hydra.sh        # Brute-force d'authentification
```

---

## 🛡️ Garde-fous (autorisation des cibles)

Le framework refuse de scanner une cible qui n'a pas été explicitement autorisée. Deux vérifications sont effectuées **avant** tout appel réseau (subfinder, nmap, etc.) :

### 1. Allowlist — `pentest_framework/cibles_autorisees.txt`

Le programme sort avec le code `2` si la cible n'est pas listée dans ce fichier. Format :

```
# Cibles exactes
testphp.vulnweb.com
localhost

# Wildcard sous-domaines
*.vulnweb.com
```

- Une cible par ligne, `#` pour les commentaires.
- La normalisation ignore le schéma, le port et le chemin : `https://testphp.vulnweb.com:8080/foo` matche l'entrée `testphp.vulnweb.com`.
- Wildcards de sous-domaine supportés via `*.domaine.tld`.

### 2. Attestation utilisateur bloquante

Avant tout scan actif, l'utilisateur doit **taper `oui`** explicitement pour attester disposer d'une autorisation écrite. Toute autre réponse (y compris Enter à vide) annule l'audit.

Ces protections ne remplacent pas une autorisation légale — elles empêchent seulement les usages accidentels ou par erreur de saisie.

---

## ⚠️ Notes & limitations

- **ZAP nécessite Docker** : si le démon n'est pas démarré, la phase DAST est ignorée (`sudo systemctl start docker`).
- **Applications SPA** (Angular/React comme Juice Shop) : le crawl et le brute-force de répertoires peuvent remonter peu de résultats si le rendu JavaScript headless n'aboutit pas.
- **Cible non HTTPS** : `testssl` n'a rien à auditer sur une cible en HTTP pur — c'est normal.
- Certains outils (Katana, httpx) peuvent être installés via `go install` en secours si absents des dépôts apt.
