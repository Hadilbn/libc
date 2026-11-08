# TP note "Return-to-libc" pour CyberRangeCZ Platform

Construit sur le modele de `library-clickjacking-cupcakes` (lui-meme derive de
`library-junior-hacker`), avec le contenu du SEED Labs *Return-to-libc Attack Lab*.
Note **sur 20** (la plateforme affiche x10 = 200 points), pensee pour une classe
d'une trentaine d'etudiants.

## Recapitulatif des niveaux

| # | Niveau | Type | Points (x10) | = /20 | Flag obtenu par |
|---|---|---|---|---|---|
| 0 | Introduction et bareme | info | - | - | - |
| 1 | Acces au sandbox | access | - | - | passkey `start` |
| 2 | Tache 1 - Adresses system()/exit() | training | 30 | 3 | `sudo r2llab-verify 1` |
| 3 | Tache 2 - Adresse variable d'environnement | training | 30 | 3 | `sudo r2llab-verify 2` |
| 4 | Tache 3 - Shell root (system/exit) | training | 60 | 6 | `sudo r2llab-verify 3` |
| 5 | Tache 3 bis - Questions de reflexion | training | 20 | 2 | reponse `oui` (manuel) |
| 6 | Tache 4 - Contournement dash (execv) | training | 60 | 6 | `sudo r2llab-verify 4` |
| 7 | Tache 5 (BONUS) - ROP, foo() x10 | training | 30 | 3 bonus | `sudo r2llab-verify 5` |
| 8 | Fin du TP | info | - | - | - |
| | **Total** | | **200** | **20** | |

**Penalites indices :** 5, 10, 15 ou 20 points = **-0.5 a -2 /20** (nombres entiers
car la plateforme n'accepte pas les decimaux). Montrer la solution complete donne
0 au niveau (`solution_penalized: true`).

## Topologie

| Hote | Image | Role |
|---|---|---|
| `seed` | `seed-ubuntu-20.04` (**a verifier/adapter**) | poste etudiant : compile et exploite `retlib` localement |
| `router` | `debian-12-x86_64` | donne acces Internet au sandbox (apt, PEDA) |

Reseau `seed-switch` 10.1.27.0/24 (`seed` = 10.1.27.10, router = 10.1.27.1) + WAN.

### A verifier absolument avant la premiere utilisation
Lors d'un essai precedent sur votre cloud avec un nom d'image invalide, OpenStack
a automatiquement substitue une image nommee litteralement **"seed"** - mais la
VM n'a alors pas atteint l'etat `ACTIVE` (echec Terraform). Avant de lancer ce TP :
1. Demandez a l'administrateur de la plateforme le nom EXACT de l'image SEED
   Ubuntu enregistree dans votre catalogue OpenStack/Glance, et le `mgmt_user`
   correspondant (souvent `ubuntu`, `seed`, ou `debian` selon l'image).
2. Mettez ce nom dans `topology.yml` (`hosts: seed: base_box: image:`).
3. Si aucune image SEED pre-equipee (gcc-multilib, gdb) n'existe chez vous,
   une image `ubuntu-20.04-x86_64` generique fonctionne aussi : le role
   `seed` installe lui-meme tous les paquets necessaires (gcc-multilib, gdb,
   PEDA, zsh) au provisioning.

## Comment chaque etudiant obtient des flags differents (anti-triche)

1. `variables.yml` declare 5 variables (`flag_t1_addrs` ... `flag_t5_rop`) :
   la plateforme genere une valeur aleatoire **differente par sandbox** (APG).
2. `training.json` a `variant_sandboxes: true`, et chaque niveau "training"
   utilise `variant_answers: true` + `answer_variable_name: <variable>`.
3. Le role `seed` ecrit ces valeurs dans `/opt/r2llab/secrets.json`
   (root-only) avec le verificateur `/usr/local/bin/r2llab-verify`.
4. L'etudiant lance `sudo r2llab-verify <1-5>`. L'outil **execute reellement**
   `./retlib` avec le `badfile` de l'etudiant et verifie :
   - Tache 1 : les adresses system()/exit() qu'il a notees correspondent aux
     vraies valeurs (recalculees en direct via gdb sur SA machine, puisque
     l'ASLR est desactivee -> c'est reproductible).
   - Tache 2 : le format de l'adresse notee (verification allegee, voir
     limites ci-dessous).
   - Tache 3 : l'execution de `retlib` + son `badfile` donne bien un shell
     avec `uid=0`.
   - Tache 4 : meme chose, MAIS seulement si `/bin/sh` pointe deja vers
     `dash` - impossible de recycler le badfile de la Tache 3.
   - Tache 5 : `foo()` est invoquee exactement 10 fois ET un shell root
     est obtenu ensuite.
   Le flag personnel n'est affiche QUE si tout est correct.
5. L'etudiant est retire du groupe `sudo` general (seul `r2llab-verify` est
   autorise via `sudoers.d`), pour qu'il ne lise pas `secrets.json` directement.

### Limites de la verification automatique
- La Tache 2 ne verifie que le format de l'adresse notee par l'etudiant (pas
  sa valeur exacte), car reproduire fidelement l'environnement du processus
  `prtenv` depuis le contexte `sudo` du verificateur n'est pas fiable. Son
  exactitude est confirmee indirectement par la reussite de la Tache 3, qui
  reutilise la meme adresse dans l'exploit.
- Les Taches 1, 3, 4 et 5 sont verifiees dynamiquement et fiables (le
  verificateur rejoue gdb ou `retlib` lui-meme sur la machine de l'etudiant).

## Importer dans CyberRangeCZ et lancer pour N etudiants

1. **Poussez ce depot sur GitHub** (prive recommande, avec deploy key ou
   token scope au repo - voir conversation precedente).
2. *Sandbox definitions* -> **Create** -> URL Git du depot (branche `main`).
   La plateforme lit `topology.yml`, `variables.yml`, `provisioning/`.
3. *Training definitions* -> **Upload** `training.json` (etat `UNRELEASED`,
   "variant sandboxes" deja active) -> publiez quand vous etes pret.
4. *Pools* -> creez un pool a partir de la sandbox definition, taille
   N + 2-3 de reserve.
5. *Training instances* -> creez une instance a partir de la training
   definition + le pool, donnez le lien/token aux etudiants.
6. Le tableau de scores affiche des points x10 ; divisez par 10 pour la
   note sur 20.

## Contenu du depot

```
topology.yml                 topologie (VM seed + routeur)
variables.yml                variables APG = les 5 flags par etudiant
training.json                definition du training (niveaux, indices, bareme)
provisioning/
  playbook.yml                point d'entree Ansible
  requirements.yml            roles externes (user-access, chrony)
  host_vars/seed.yml
  roles/seed/                 deploiement du lab : paquets, compilation,
                               flags, verificateur, regles sudo
    files/                    retlib.c, prtenv.c, Makefile, squelettes
                               d'exploit, r2llab-verify
```

## Depannage

- **`r2llab-verify: command not found`** -> ouvrez un nouveau terminal (alias
  dans `.bashrc`) ou lancez `sudo /usr/local/bin/r2llab-verify <n>`.
- **L'etudiant a encore `sudo`** -> certaines images accordent un sudo
  NOPASSWD via `/etc/sudoers.d/` qui pourrait entrer en conflit ; verifiez
  qu'aucune autre regle ne donne acces a `secrets.json`.
- **La Tache 4 echoue alors que la Tache 3 marchait** -> c'est le
  comportement attendu : `/bin/sh` doit pointer vers `dash` pour la Tache 4,
  et l'exploit `system()`-based de la Tache 3 ne fonctionne plus dans ce cas.
- **Erreur "routers" a la creation de la Sandbox Definition** -> deja gere
  ici (`routers:` + `router_mappings:` presents dans `topology.yml`).
