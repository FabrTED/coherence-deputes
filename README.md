# Coherence des deputes - Projet L2 MIASHS

Ce projet est un programme en langage C permettant d’analyser la coherence entre
les positions politiques affichees par des deputes (axes ideologiques declares)
et leurs votes reels sur plusieurs scrutins.

Le programme permet :
- le chargement de donnees depuis un fichier CSV,
- l’affichage d’un profil detaille pour chaque depute,
- la comparaison de deux deputes,
- le calcul d’un score de coherence (0 a 100),
- l’affichage de diagrammes ASCII 2D,
- la generation automatique de fichiers texte de resultats.

---

## Compilation

Le programme est concu pour etre compile en ligne de commande.
Le Fichier CSV doit être mis dans le même dossier que le programme.
Le Protocole pour créer son propre CSV est dispo dans le fichier Protocole CSV.txt disponible dans la branche donnees et dans le rapport.

### Sous Windows (Code::Blocks ou MinGW)

```bash
gcc src/main.c -o coherence -lm


