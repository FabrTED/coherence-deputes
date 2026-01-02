/* ============================================================================
   PROJET L2 MIASHS (UPN) — "Coherence deputes"
   Langage : C (console / Code::Blocks / Windows)
   Objectif : comparer ce qu'un depute "affiche" (axes programme) vs ce qu'il fait
              (axes deduits de ses votes), calculer un score de coherence, afficher
              profils, comparaison, et Top 5 (meilleurs / pires).

   DEMANDES UTILISATEUR (prises en compte ici) :
   1) Affichage centre (au mieux) + interface "page" (efface ecran, separateurs).
   2) PLUS AERE : espaces entre blocs / sections.
   3) EN CAS D'ERREUR : on NE retourne PAS directement au menu.
      -> message d'erreur + possibilite de recommencer.
      -> 0 = annuler (retour menu) dans les saisies mp_id.
   4) Top 5 : afficher le symbole % et exporter un fichier Top/Bottom (sans diagrammes).
   5) Sortie console et protocole : EVITER accents et caracteres speciaux (ASCII).
      -> On precise dans le protocole que le CSV doit eviter accents/UTF-8 si besoin.
   6) Diagrammes conserves pour PROFIL et COMPARAISON (ecran + fichiers).
   7) COMPARAISON : 1 seul diagramme avec 4 points : P1 V1 P2 V2 (dans le meme diagramme).

   IMPORTANT :
   - Le centrage depend de la largeur reelle de ta console.
     Modifie CONSOLE_WIDTH si besoin (80, 100, 120...).
   - Le CSV doit etre dans le dossier du .exe (ou donner le chemin).

   Compilation (si besoin) : gcc main.c -o projet -lm
   ============================================================================ */

#include <stdio.h>      // printf, fopen, fgets, fprintf
#include <string.h>     // strlen, strncpy, memset, strchr, memmove
#include <stdlib.h>     // malloc, realloc, free, strtol, qsort, bsearch, system
#include <ctype.h>      // isspace
#include <math.h>       // sqrtf, lroundf
#include <time.h>       // time, localtime, strftime
#include <stdarg.h>     // va_list, va_start, vsnprintf

/* =========================
   REGLAGES GENERAUX
   ========================= */

// Largeur supposee de la console pour centrer les lignes
#define CONSOLE_WIDTH 100

// Nombre de votes V001..V008 dans le CSV
#define NB_VOTES 8

// Taille max d'une ligne CSV lue
#define MAX_LINE 1024

// Capacite initiale du tableau dynamique
#define DEFAULT_CAPACITY 16

/* =========================
   STRUCTURES DE DONNEES
   ========================= */

// 1 ligne CSV -> 1 depute
typedef struct {
    int  mp_id;                      // identifiant unique
    char nom[80];                    // nom complet (ASCII conseille)
    char parti[32];                  // sigle parti
    int  axe_economie;               // -5..+5 (programme)
    int  axe_societal;               // -5..+5 (programme)
    char patrimoine_classe;          // A..E
    char votes[NB_VOTES];            // P/C/A/N
} Depute;

// Liste dynamique (tableau realloc)
typedef struct {
    Depute *data;                    // tableau de deputes
    int size;                        // nb d'elements utilises
    int capacity;                    // nb d'elements alloues
} ListeDeputes;

/* =========================
   CONSOLE : "PAGES"
   ========================= */

void clear_screen(void) {
    // Windows (Code::Blocks)
    system("cls");
}

void pause_enter(void) {
    // Attendre une ligne "Entree" (utile pour lire l'ecran)
    int c;
    while ((c = getchar()) != '\n' && c != EOF) {}
}

/* =========================
   CENTRAGE + AFFICHAGE
   ========================= */

void print_padding(int pad) {
    for (int i = 0; i < pad; i++) putchar(' ');
}

void print_center_line(const char *s) {
    // Affiche une ligne centree (si possible)
    if (!s) return;

    int len = (int)strlen(s);

    // Si trop long, pas de centrage
    if (len >= CONSOLE_WIDTH) {
        printf("%s\n", s);
        return;
    }

    int pad = (CONSOLE_WIDTH - len) / 2;
    print_padding(pad);
    printf("%s\n", s);
}

void cprintf_center(const char *fmt, ...) {
    // printf centre (une ou plusieurs lignes)
    char buf[2048];

    va_list ap;
    va_start(ap, fmt);
    vsnprintf(buf, sizeof(buf), fmt, ap);
    va_end(ap);

    // Si plusieurs lignes -> centrer ligne par ligne
    char *p = buf;
    while (*p) {
        char *nl = strchr(p, '\n');
        if (!nl) {
            print_center_line(p);
            break;
        }
        *nl = '\0';
        print_center_line(p);
        p = nl + 1;
    }
}

void sep_center(void) {
    cprintf_center("============================================================");
}

void subsep_center(void) {
    cprintf_center("------------------------------");
}

/* =========================
   OUTILS : nettoyage saisie
   ========================= */

void rstrip_newline(char *s) {
    // Supprime \n et \r en fin de chaine
    if (!s) return;

    size_t n = strlen(s);
    while (n > 0 && (s[n - 1] == '\n' || s[n - 1] == '\r')) {
        s[n - 1] = '\0';
        n--;
    }
}

void trim_inplace(char *s) {
    // Trim espaces gauche + droite
    if (!s) return;

    // Trim gauche
    size_t i = 0;
    while (s[i] && isspace((unsigned char)s[i])) i++;
    if (i > 0) memmove(s, s + i, strlen(s + i) + 1);

    // Trim droite
    size_t n = strlen(s);
    while (n > 0 && isspace((unsigned char)s[n - 1])) {
        s[n - 1] = '\0';
        n--;
    }
}

/* =========================
   SAISIE CENTREE
   ========================= */

void ask_center_line(char *out, int n, const char *prompt) {
    // Afficher un prompt centre (autant que possible), puis lire une ligne
    int len = (int)strlen(prompt);
    if (len < CONSOLE_WIDTH) {
        int pad = (CONSOLE_WIDTH - len) / 2;
        print_padding(pad);
    }
    printf("%s", prompt);

    if (!fgets(out, n, stdin)) {
        out[0] = '\0';
        return;
    }

    // Nettoyer fin de ligne + espaces
    rstrip_newline(out);
    trim_inplace(out);
}

int lire_int_strict_from_buf(const char *buf, int *out_value) {
    // Convertit buf en entier strict :
    //  - accepte uniquement si toute la chaine est un nombre
    if (!buf || buf[0] == '\0') return 0;

    char *endptr = NULL;
    long v = strtol(buf, &endptr, 10);

    if (!endptr || *endptr != '\0') return 0;

    *out_value = (int)v;
    return 1;
}

int ask_center_int(int *out_value, const char *prompt) {
    // Demande un entier via prompt centre
    char buf[64];
    ask_center_line(buf, sizeof(buf), prompt);
    return lire_int_strict_from_buf(buf, out_value);
}

/* =========================
   LISTE DYNAMIQUE : init/free/push
   ========================= */

void liste_init(ListeDeputes *L) {
    L->data = NULL;
    L->size = 0;
    L->capacity = 0;
}

void liste_free(ListeDeputes *L) {
    free(L->data);
    L->data = NULL;
    L->size = 0;
    L->capacity = 0;
}

int liste_reserve(ListeDeputes *L, int new_capacity) {
    // Assure une capacite >= new_capacity
    if (new_capacity <= L->capacity) return 1;

    Depute *new_data = (Depute*)realloc(L->data, (size_t)new_capacity * sizeof(Depute));
    if (!new_data) return 0;

    L->data = new_data;
    L->capacity = new_capacity;
    return 1;
}

int liste_push_back(ListeDeputes *L, const Depute *d) {
    // Ajoute un element a la fin
    if (L->size >= L->capacity) {
        int new_cap = (L->capacity == 0) ? DEFAULT_CAPACITY : (L->capacity * 2);
        if (!liste_reserve(L, new_cap)) return 0;
    }

    L->data[L->size] = *d;
    L->size++;
    return 1;
}

/* =========================
   PATRIMOINE : categories
   ========================= */

const char* patrimoine_intervalle(char c) {
    // Grille "equilibree" : pas 100% population, pas 100% deconnecte deputes
    switch (c) {
        case 'A': return "< 20k";
        case 'B': return "20k-100k";
        case 'C': return "100k-500k";
        case 'D': return "500k-2M";
        case 'E': return "> 2M";
        default:  return "inconnu";
    }
}

/* =========================
   PROTOCOLE (dans les fichiers)
   ========================= */

void protocole_write(FILE *out) {
    fprintf(out, "\n============================================================\n");
    fprintf(out, "AIDE / PROTOCOLE\n");
    fprintf(out, "------------------------------\n\n");

    fprintf(out, "IMPORTANT (console) : eviter accents / caracteres speciaux.\n");
    fprintf(out, "Si ta console affiche mal : mettre uniquement ASCII dans CSV.\n\n");

    fprintf(out, "Chaque ligne CSV = 1 depute.\n");
    fprintf(out, "Selection par mp_id (identifiant unique).\n\n");

    fprintf(out, "Axes (programme) :\n");
    fprintf(out, "X economie  (-5..+5) : -5 gauche eco (Etat/redistribution) | +5 droite eco (marche)\n");
    fprintf(out, "Y societal  (-5..+5) : -5 progressiste (libertes)         | +5 conservateur/ordre\n\n");

    fprintf(out, "Patrimoine (classe A..E) :\n");
    fprintf(out, "A (%s)\n", patrimoine_intervalle('A'));
    fprintf(out, "B (%s)\n", patrimoine_intervalle('B'));
    fprintf(out, "C (%s)\n", patrimoine_intervalle('C'));
    fprintf(out, "D (%s)\n", patrimoine_intervalle('D'));
    fprintf(out, "E (%s)\n\n", patrimoine_intervalle('E'));

    fprintf(out, "Votes : P=Pour, C=Contre, A=Abstention, N=Absent\n\n");

    fprintf(out, "Themes :\n");
    fprintf(out, "V001 Budget/rigueur\n");
    fprintf(out, "V002 Fiscalite/redistribution\n");
    fprintf(out, "V003 Immigration/securite\n");
    fprintf(out, "V004 Energie/industrie\n");
    fprintf(out, "V005 Social/chomage\n");
    fprintf(out, "V006 Ecole/autorite\n");
    fprintf(out, "V007 Institutions/autorite executive\n");
    fprintf(out, "V008 Environnement/transports\n\n");

    // IMPORTANT : nouvelle legende comparaison
    fprintf(out, "Diagramme comparaison (un seul diagramme) :\n");
    fprintf(out, "P1/V1 = programme/votes du depute 1 ; P2/V2 = programme/votes du depute 2\n");
    fprintf(out, "Si plusieurs points se superposent : **\n");

    fprintf(out, "============================================================\n");
}

/* =========================
   CSV : split avec guillemets
   ========================= */

int split_csv_quoted(char *line, char *fields[], int max_fields) {
    // Decoupe CSV en gerant "..."
    int count = 0;
    char *p = line;

    while (*p && count < max_fields) {
        // Sauter espaces
        while (*p && (*p == ' ' || *p == '\t')) p++;

        // Champ entre guillemets
        if (*p == '"') {
            p++;
            fields[count++] = p;

            while (*p && *p != '"') p++;
            if (*p == '"') { *p = '\0'; p++; }

            while (*p && *p != ',') p++;
            if (*p == ',') { *p = '\0'; p++; }
        } else {
            // Champ normal
            fields[count++] = p;

            while (*p && *p != ',') p++;
            if (*p == ',') { *p = '\0'; p++; }
        }
    }

    // Nettoyer champs
    for (int i = 0; i < count; i++) trim_inplace(fields[i]);
    return count;
}

/* =========================
   TRI + BSEARCH (mp_id)
   ========================= */

// Tri croissant par mp_id
int cmp_depute_by_mpid(const void *a, const void *b) {
    const Depute *da = (const Depute*)a;
    const Depute *db = (const Depute*)b;
    if (da->mp_id < db->mp_id) return -1;
    if (da->mp_id > db->mp_id) return  1;
    return 0;
}

// Comparateur bsearch : cle int vs element Depute
int cmp_key_mpid_to_depute(const void *key, const void *elem) {
    const int *k = (const int*)key;
    const Depute *d = (const Depute*)elem;
    if (*k < d->mp_id) return -1;
    if (*k > d->mp_id) return  1;
    return 0;
}

// Recherche dichotomique dans un tableau trie
int trouver_depute_par_id(const ListeDeputes *L, int mp_id) {
    if (!L || L->size <= 0) return -1;

    Depute *found = (Depute*)bsearch(
        &mp_id,
        L->data,
        (size_t)L->size,
        sizeof(Depute),
        cmp_key_mpid_to_depute
    );

    if (!found) return -1;

    // convertir pointeur -> index
    return (int)(found - L->data);
}

/* =========================
   CHARGEMENT CSV (puis tri)
   ========================= */

int charger_deputes_csv(const char *fichier, ListeDeputes *L) {
    FILE *fp = fopen(fichier, "r");
    if (!fp) {
        perror("Erreur fopen");
        return 0;
    }

    // Repartir sur liste vide
    liste_free(L);
    liste_init(L);

    char line[MAX_LINE];

    // Lire et ignorer l'en-tete
    if (!fgets(line, sizeof(line), fp)) {
        fclose(fp);
        return 1;
    }

    // Lire les deputes
    while (fgets(line, sizeof(line), fp)) {
        rstrip_newline(line);
        trim_inplace(line);
        if (line[0] == '\0') continue;

        char *fields[32];
        int nfields = split_csv_quoted(line, fields, 32);

        // Champs attendus : 6 + 8 = 14
        if (nfields < 14) continue;

        Depute d;
        memset(&d, 0, sizeof(d));

        d.mp_id = atoi(fields[0]);
        strncpy(d.nom, fields[1], sizeof(d.nom) - 1);
        strncpy(d.parti, fields[2], sizeof(d.parti) - 1);
        d.axe_economie = atoi(fields[3]);
        d.axe_societal = atoi(fields[4]);
        d.patrimoine_classe = fields[5][0];

        for (int i = 0; i < NB_VOTES; i++) {
            char v = fields[6 + i][0];
            if (v != 'P' && v != 'C' && v != 'A' && v != 'N') v = 'N';
            d.votes[i] = v;
        }

        if (!liste_push_back(L, &d)) {
            fclose(fp);
            return 0;
        }
    }

    fclose(fp);

    // Tri par mp_id pour permettre bsearch
    qsort(L->data, (size_t)L->size, sizeof(Depute), cmp_depute_by_mpid);

    return 1;
}

/* =========================
   CALCULS : votes -> axes + coherence
   ========================= */

int vote_char_to_int(char v) {
    // P=+1, C=-1, A/N=0
    if (v == 'P') return  1;
    if (v == 'C') return -1;
    return 0;
}

float clampf(float x, float lo, float hi) {
    // Clamp simple
    if (x < lo) return lo;
    if (x > hi) return hi;
    return x;
}

void calculer_axes_votes(const Depute *d, float *eco, float *soc) {
    // Poids simples : cree des axes "votes" plausibles
    const float w_eco[NB_VOTES] = { 2.0f, 2.0f, 0.5f, 1.0f, 1.5f, 0.0f, 0.0f, 1.0f };
    const float w_soc[NB_VOTES] = { 0.0f, 0.0f, 2.0f, 0.5f, 0.0f, 1.5f, 2.0f, 0.5f };

    float sum_eco = 0.0f, sum_soc = 0.0f;
    float norm_eco = 0.0f, norm_soc = 0.0f;

    for (int i = 0; i < NB_VOTES; i++) {
        int vi = vote_char_to_int(d->votes[i]);
        sum_eco += vi * w_eco[i];
        sum_soc += vi * w_soc[i];
        norm_eco += w_eco[i];
        norm_soc += w_soc[i];
    }

    if (norm_eco < 1e-6f) norm_eco = 1.0f;
    if (norm_soc < 1e-6f) norm_soc = 1.0f;

    *eco = clampf(5.0f * (sum_eco / norm_eco), -5.0f, 5.0f);
    *soc = clampf(5.0f * (sum_soc / norm_soc), -5.0f, 5.0f);
}

float calculer_coherence(const Depute *d) {
    // Compare distance entre (axes programme) et (axes votes)
    float eco_vote = 0.0f, soc_vote = 0.0f;
    calculer_axes_votes(d, &eco_vote, &soc_vote);

    float dx = (float)d->axe_economie - eco_vote;
    float dy = (float)d->axe_societal - soc_vote;

    float dist = sqrtf(dx * dx + dy * dy);

    // diagonale du carre [-5,5]x[-5,5] = sqrt(200) ~ 14.1421
    const float maxDist = 14.1421356f;

    float score = 100.0f * (1.0f - dist / maxDist);
    return clampf(score, 0.0f, 100.0f);
}

/* =========================
   AFFICHAGES : liste + themes
   ========================= */

void afficher_themes_votes_center(void) {
    // Ligne de rappel compacte des themes
    cprintf_center("Themes : V001 V002 V003 V004 V005 V006 V007 V008");
    cprintf_center("         Budg Fisc Immi Ener Soci Ecol Inst Envi");
}

void afficher_liste_center(const ListeDeputes *L) {
    sep_center();
    cprintf_center("LISTE DES DEPUTES (%d)", L->size);
    sep_center();

    printf("\n"); // aeration
    cprintf_center("%-6s | %-28s | %-10s", "mp_id", "Nom", "Parti");
    cprintf_center("-------+------------------------------+-----------");

    for (int i = 0; i < L->size; i++) {
        cprintf_center("%-6d | %-28s | %-10s",
                       L->data[i].mp_id, L->data[i].nom, L->data[i].parti);
    }

    printf("\n"); // aeration
    sep_center();
}

/* =========================
   DIAGRAMMES ASCII 2D
   - Profil : P (programme) et v (votes)
   - Comparaison : 1 seul diagramme avec P1 V1 P2 V2
   ========================= */

int map_axis_to_grid(float v, int size) {
    // map [-5,5] -> [0,size-1]
    float t = (v + 5.0f) / 10.0f;
    int idx = (int)lroundf(t * (float)(size - 1));
    if (idx < 0) idx = 0;
    if (idx >= size) idx = size - 1;
    return idx;
}

/* --- Grille a tokens 2 caracteres (ex: "P1", "V2", "..") --- */
void token_set(char grid[21][21][3], int y, int x, const char tok[3]) {
    // Place un token 2 chars dans la cellule (x,y)
    // Si collision (cellule deja occupee par autre chose) -> "**"
    if (x < 0 || x >= 21 || y < 0 || y >= 21) return;

    // Si deja vide
    if (strcmp(grid[y][x], "..") == 0 ||
        strcmp(grid[y][x], "--") == 0 ||
        strcmp(grid[y][x], "||") == 0 ||
        strcmp(grid[y][x], "+ ") == 0) {
        strncpy(grid[y][x], tok, 2);
        grid[y][x][2] = '\0';
        return;
    }

    // Si deja un token different -> collision
    if (strcmp(grid[y][x], tok) != 0) {
        strcpy(grid[y][x], "**");
    }
}

void diagramme_print_grid_center(char grid[21][21][3], FILE *log) {
    // Affiche la grille (haut -> bas) en tokens 2 chars
    // Sur console : centrer chaque ligne
    // Dans fichier : pas de centrage
    const int size = 21;

    for (int y = size - 1; y >= 0; y--) {
        char linebuf[512];
        int pos = 0;

        // Construire une ligne : "tok tok tok ..."
        for (int x = 0; x < size; x++) {
            linebuf[pos++] = grid[y][x][0];
            linebuf[pos++] = grid[y][x][1];
            linebuf[pos++] = ' ';
        }
        linebuf[pos] = '\0';

        // Console (centre)
        print_center_line(linebuf);

        // Fichier (brut)
        if (log) {
            for (int x = 0; x < size; x++) {
                fprintf(log, "%c%c ", grid[y][x][0], grid[y][x][1]);
            }
            fprintf(log, "\n");
        }
    }
}

void afficher_diagramme_profil(FILE *log, const Depute *d) {
    // Diagramme profil : P (programme) et v (votes)
    // Ici on utilise tokens 2 chars : "P " et "v "
    const int size = 21;
    char grid[21][21][3];

    // Init : ".."
    for (int y = 0; y < size; y++)
        for (int x = 0; x < size; x++)
            strcpy(grid[y][x], "..");

    // Axes : "--" et "||" et centre "+ "
    int mid = size / 2;
    for (int x = 0; x < size; x++) strcpy(grid[mid][x], "--");
    for (int y = 0; y < size; y++) strcpy(grid[y][mid], "||");
    strcpy(grid[mid][mid], "+ ");

    // Programme (axes CSV)
    int px = map_axis_to_grid((float)d->axe_economie, size);
    int py = map_axis_to_grid((float)d->axe_societal, size);

    // Votes (axes calcules)
    float eco_vote = 0.0f, soc_vote = 0.0f;
    calculer_axes_votes(d, &eco_vote, &soc_vote);
    int vx = map_axis_to_grid(eco_vote, size);
    int vy = map_axis_to_grid(soc_vote, size);

    // Placer tokens
    token_set(grid, py, px, "P ");
    token_set(grid, vy, vx, "v ");

    // Si meme case, token_set aura mis "**" (collision)
    // Mais ici, si exactement meme case, on prefere "* "
    if (px == vx && py == vy) strcpy(grid[py][px], "* ");

    printf("\n"); // aeration
    subsep_center();
    cprintf_center("DIAGRAMME 2D (PROFIL)  X=economie  Y=societal");
    cprintf_center("P=programme  v=votes  *=meme position");
    cprintf_center("X : -5 gauche eco ... +5 droite eco");
    cprintf_center("Y : -5 progressiste ... +5 conservateur/ordre");
    cprintf_center("P(eco=%d, soc=%d) | v(eco=%.2f, soc=%.2f)",
                  d->axe_economie, d->axe_societal, eco_vote, soc_vote);
    subsep_center();

    if (log) {
        fprintf(log, "\n------------------------------\n");
        fprintf(log, "DIAGRAMME 2D (PROFIL)  X=economie  Y=societal\n");
        fprintf(log, "P=programme  v=votes  *=meme position\n");
        fprintf(log, "X : -5 gauche eco ... +5 droite eco\n");
        fprintf(log, "Y : -5 progressiste ... +5 conservateur/ordre\n");
        fprintf(log, "P(eco=%d, soc=%d) | v(eco=%.2f, soc=%.2f)\n\n",
                d->axe_economie, d->axe_societal, eco_vote, soc_vote);
    }

    // Afficher la grille
    diagramme_print_grid_center(grid, log);

    printf("\n"); // aeration
}

void afficher_diagramme_comparaison(FILE *log, const Depute *a, const Depute *b) {
    // Diagramme comparaison : un seul diagramme avec 4 points
    // P1 V1 = depute A ; P2 V2 = depute B
    // Tokens 2 chars : "P1", "V1", "P2", "V2"
    const int size = 21;
    char grid[21][21][3];

    // Init : ".."
    for (int y = 0; y < size; y++)
        for (int x = 0; x < size; x++)
            strcpy(grid[y][x], "..");

    // Axes
    int mid = size / 2;
    for (int x = 0; x < size; x++) strcpy(grid[mid][x], "--");
    for (int y = 0; y < size; y++) strcpy(grid[y][mid], "||");
    strcpy(grid[mid][mid], "+ ");

    // Calcul points de A
    int p1x = map_axis_to_grid((float)a->axe_economie, size);
    int p1y = map_axis_to_grid((float)a->axe_societal, size);

    float v1eco = 0.0f, v1soc = 0.0f;
    calculer_axes_votes(a, &v1eco, &v1soc);
    int v1x = map_axis_to_grid(v1eco, size);
    int v1y = map_axis_to_grid(v1soc, size);

    // Calcul points de B
    int p2x = map_axis_to_grid((float)b->axe_economie, size);
    int p2y = map_axis_to_grid((float)b->axe_societal, size);

    float v2eco = 0.0f, v2soc = 0.0f;
    calculer_axes_votes(b, &v2eco, &v2soc);
    int v2x = map_axis_to_grid(v2eco, size);
    int v2y = map_axis_to_grid(v2soc, size);

    // Placer tokens (collision -> "**")
    token_set(grid, p1y, p1x, "P1");
    token_set(grid, v1y, v1x, "V1");
    token_set(grid, p2y, p2x, "P2");
    token_set(grid, v2y, v2x, "V2");

    printf("\n"); // aeration
    subsep_center();
    cprintf_center("DIAGRAMME 2D (COMPARAISON)  X=economie  Y=societal");
    cprintf_center("P1/V1 = depute 1 (programme/votes) | P2/V2 = depute 2 (programme/votes)");
    cprintf_center("Si plusieurs points superposes : **");
    cprintf_center("X : -5 gauche eco ... +5 droite eco");
    cprintf_center("Y : -5 progressiste ... +5 conservateur/ordre");
    subsep_center();

    // Rappel numerique (utile)
    cprintf_center("Depute1 P1(%d,%d) V1(%.2f,%.2f) | Depute2 P2(%d,%d) V2(%.2f,%.2f)",
                  a->axe_economie, a->axe_societal, v1eco, v1soc,
                  b->axe_economie, b->axe_societal, v2eco, v2soc);

    if (log) {
        fprintf(log, "\n------------------------------\n");
        fprintf(log, "DIAGRAMME 2D (COMPARAISON)  X=economie  Y=societal\n");
        fprintf(log, "P1/V1 = depute 1 (programme/votes) | P2/V2 = depute 2 (programme/votes)\n");
        fprintf(log, "Si plusieurs points superposes : **\n");
        fprintf(log, "X : -5 gauche eco ... +5 droite eco\n");
        fprintf(log, "Y : -5 progressiste ... +5 conservateur/ordre\n");
        fprintf(log, "Depute1 P1(%d,%d) V1(%.2f,%.2f) | Depute2 P2(%d,%d) V2(%.2f,%.2f)\n\n",
                a->axe_economie, a->axe_societal, v1eco, v1soc,
                b->axe_economie, b->axe_societal, v2eco, v2soc);
    }

    // Grille
    diagramme_print_grid_center(grid, log);

    printf("\n"); // aeration
}

/* =========================
   PROFIL COMPLET : affichage + fichier
   ========================= */

void afficher_fiche_complete(FILE *log, const Depute *d) {
    // Bloc identite (aere)
    cprintf_center("mp_id       : %d", d->mp_id);
    cprintf_center("Nom         : %s", d->nom);
    cprintf_center("Parti       : %s", d->parti);

    printf("\n"); // aeration
    cprintf_center("Axes (programme) : eco=%d  soc=%d", d->axe_economie, d->axe_societal);
    cprintf_center("Patrimoine       : %c (%s)", d->patrimoine_classe, patrimoine_intervalle(d->patrimoine_classe));

    printf("\n"); // aeration

    // Votes (ligne)
    char votesbuf[128];
    int p = 0;
    p += snprintf(votesbuf + p, sizeof(votesbuf) - (size_t)p, "Votes : ");
    for (int i = 0; i < NB_VOTES; i++) {
        p += snprintf(votesbuf + p, sizeof(votesbuf) - (size_t)p, "%c", d->votes[i]);
        if (i < NB_VOTES - 1) p += snprintf(votesbuf + p, sizeof(votesbuf) - (size_t)p, " ");
    }
    cprintf_center("%s", votesbuf);
    afficher_themes_votes_center();

    // Ecriture fichier (non centre)
    if (log) {
        fprintf(log, "mp_id       : %d\n", d->mp_id);
        fprintf(log, "Nom         : %s\n", d->nom);
        fprintf(log, "Parti       : %s\n\n", d->parti);

        fprintf(log, "Axes (programme) : eco=%d  soc=%d\n", d->axe_economie, d->axe_societal);
        fprintf(log, "Patrimoine       : %c (%s)\n\n", d->patrimoine_classe, patrimoine_intervalle(d->patrimoine_classe));

        fprintf(log, "Votes : ");
        for (int i = 0; i < NB_VOTES; i++) {
            fprintf(log, "%c", d->votes[i]);
            if (i < NB_VOTES - 1) fprintf(log, " ");
        }
        fprintf(log, "\n");
        fprintf(log, "Themes : V001 V002 V003 V004 V005 V006 V007 V008\n");
        fprintf(log, "         Budg Fisc Immi Ener Soci Ecol Inst Envi\n");
    }
}

void print_profil_complet(FILE *log, const Depute *d) {
    sep_center();
    cprintf_center("PROFIL COMPLET");
    sep_center();

    printf("\n"); // aeration

    if (log) {
        fprintf(log, "\n============================================================\n");
        fprintf(log, "PROFIL COMPLET\n");
        fprintf(log, "============================================================\n\n");
    }

    // Fiche de base
    afficher_fiche_complete(log, d);

    printf("\n"); // aeration

    // Calculs
    float eco_vote = 0.0f, soc_vote = 0.0f;
    calculer_axes_votes(d, &eco_vote, &soc_vote);
    float coh = calculer_coherence(d);

    subsep_center();
    cprintf_center("Axes (votes)      : eco=%.2f  soc=%.2f", eco_vote, soc_vote);
    cprintf_center("Coherence (0-100) : %.1f %%", coh);
    subsep_center();

    if (log) {
        fprintf(log, "\n------------------------------\n");
        fprintf(log, "Axes (votes)      : eco=%.2f  soc=%.2f\n", eco_vote, soc_vote);
        fprintf(log, "Coherence (0-100) : %.1f %%\n", coh);
        fprintf(log, "------------------------------\n");
    }

    // Diagramme profil (P vs v)
    afficher_diagramme_profil(log, d);

    sep_center();
}

/* =========================
   EXPORTS : noms de fichiers
   ========================= */

void make_timestamp(char *buf, int n) {
    // Timestamp simple pour nom de fichier unique
    time_t t = time(NULL);
    struct tm *tm_info = localtime(&t);
    strftime(buf, n, "%Y%m%d_%H%M%S", tm_info);
}

/* =========================
   PROFIL : export
   ========================= */

void export_profil(const Depute *d) {
    char ts[32];
    make_timestamp(ts, sizeof(ts));

    char filename[220];
    snprintf(filename, sizeof(filename), "profil_%d_%s.txt", d->mp_id, ts);

    FILE *log = fopen(filename, "w");
    if (!log) {
        sep_center();
        cprintf_center("Erreur : impossible de creer le fichier '%s'.", filename);
        cprintf_center("Le profil est affiche uniquement a l'ecran.");
        sep_center();

        // Affichage ecran uniquement
        print_profil_complet(NULL, d);
        return;
    }

    // Protocole dans le fichier
    protocole_write(log);

    // Profil complet dans fichier + ecran
    print_profil_complet(log, d);

    fclose(log);

    printf("\n");
    subsep_center();
    cprintf_center("Fichier enregistre : %s", filename);
}

/* =========================
   COMPARAISON : similarite votes + export
   ========================= */

float similarite_votes(const Depute *a, const Depute *b) {
    // Similarite sur votes connus (pas N)
    int considered = 0, same = 0;

    for (int i = 0; i < NB_VOTES; i++) {
        if (a->votes[i] == 'N' || b->votes[i] == 'N') continue;
        considered++;
        if (a->votes[i] == b->votes[i]) same++;
    }

    if (considered == 0) return 0.0f;
    return (float)same / (float)considered;
}

void export_comparaison(const Depute *a, const Depute *b) {
    char ts[32];
    make_timestamp(ts, sizeof(ts));

    char filename[240];
    snprintf(filename, sizeof(filename), "compare_%d_%d_%s.txt", a->mp_id, b->mp_id, ts);

    FILE *log = fopen(filename, "w");
    if (!log) {
        sep_center();
        cprintf_center("Erreur : impossible de creer le fichier '%s'.", filename);
        cprintf_center("La comparaison est affichee uniquement a l'ecran.");
        sep_center();

        // Ecran uniquement : resume + diagramme comparaison
        sep_center();
        cprintf_center("COMPARAISON");
        sep_center();

        printf("\n");
        cprintf_center("Depute 1 : mp_id=%d | %s (%s)", a->mp_id, a->nom, a->parti);
        cprintf_center("Depute 2 : mp_id=%d | %s (%s)", b->mp_id, b->nom, b->parti);

        printf("\n");
        float sim = 100.0f * similarite_votes(a, b);
        cprintf_center("Similarite votes : %.1f %%", sim);

        // Diagramme unique P1 V1 P2 V2 (ecran)
        afficher_diagramme_comparaison(NULL, a, b);

        // Option : on peut afficher ensuite les 2 profils complets si tu veux
        // (mais tu voulais surtout un seul diagramme en comparaison)
        return;
    }

    // Protocole dans fichier
    protocole_write(log);

    // Affichage ecran
    sep_center();
    cprintf_center("COMPARAISON");
    sep_center();

    printf("\n");
    cprintf_center("Depute 1 : mp_id=%d | %s (%s)", a->mp_id, a->nom, a->parti);
    cprintf_center("Depute 2 : mp_id=%d | %s (%s)", b->mp_id, b->nom, b->parti);

    printf("\n");
    float sim = 100.0f * similarite_votes(a, b);
    cprintf_center("Similarite votes : %.1f %%", sim);

    // Ecriture fichier (non centree)
    fprintf(log, "\n============================================================\n");
    fprintf(log, "COMPARAISON\n");
    fprintf(log, "============================================================\n\n");
    fprintf(log, "Depute 1 : mp_id=%d | %s (%s)\n", a->mp_id, a->nom, a->parti);
    fprintf(log, "Depute 2 : mp_id=%d | %s (%s)\n\n", b->mp_id, b->nom, b->parti);
    fprintf(log, "Similarite votes : %.1f %%\n\n", sim);

    // IMPORTANT : diagramme unique P1 V1 P2 V2 (ecran + fichier)
    afficher_diagramme_comparaison(log, a, b);

    // Bonus utile : conserver aussi les chiffres de coherence individuels
    // (petit + pour le rapport, sans surcharger)
    float cohA = calculer_coherence(a);
    float cohB = calculer_coherence(b);

    printf("\n");
    cprintf_center("Coherence depute 1 : %.1f %% | depute 2 : %.1f %%", cohA, cohB);

    fprintf(log, "\nCoherence depute 1 : %.1f %%\n", cohA);
    fprintf(log, "Coherence depute 2 : %.1f %%\n", cohB);

    fclose(log);

    printf("\n");
    subsep_center();
    cprintf_center("Fichier enregistre : %s", filename);
}

/* =========================
   TOP 5 : affichage + export (sans diagrammes)
   ========================= */

typedef struct {
    int index;        // index dans L->data
    float score;      // coherence
} ScoreItem;

void sort_scores_desc(ScoreItem *arr, int n) {
    // Tri selection (simple, suffisant pour 20 deputes)
    for (int i = 0; i < n; i++) {
        int best = i;
        for (int j = i + 1; j < n; j++) {
            if (arr[j].score > arr[best].score) best = j;
        }
        if (best != i) {
            ScoreItem tmp = arr[i];
            arr[i] = arr[best];
            arr[best] = tmp;
        }
    }
}

void export_top5_file(const ListeDeputes *L, const ScoreItem *items, int k) {
    // Export Top/Bottom dans un fichier (sans diagrammes)
    char ts[32];
    make_timestamp(ts, sizeof(ts));

    char filename[240];
    snprintf(filename, sizeof(filename), "top5_%s.txt", ts);

    FILE *log = fopen(filename, "w");
    if (!log) {
        sep_center();
        cprintf_center("Erreur : impossible de creer le fichier '%s'.", filename);
        sep_center();
        return;
    }

    // protocole obligatoire dans le fichier
    protocole_write(log);

    fprintf(log, "\n============================================================\n");
    fprintf(log, "CLASSEMENT COHERENCE (Top %d / Bottom %d)\n", k, k);
    fprintf(log, "============================================================\n\n");

    fprintf(log, "Top %d : plus coherents\n", k);
    fprintf(log, "------------------------------\n");
    for (int i = 0; i < k; i++) {
        const Depute *d = &L->data[items[i].index];
        fprintf(log, "#%d  mp_id=%d  |  %s  |  %.1f %%\n",
                i + 1, d->mp_id, d->nom, items[i].score);
    }

    fprintf(log, "\nTop %d : moins coherents\n", k);
    fprintf(log, "------------------------------\n");
    for (int i = 0; i < k; i++) {
        int idx = L->size - 1 - i;
        const Depute *d = &L->data[items[idx].index];
        fprintf(log, "#%d  mp_id=%d  |  %s  |  %.1f %%\n",
                i + 1, d->mp_id, d->nom, items[idx].score);
    }

    fclose(log);

    printf("\n");
    subsep_center();
    cprintf_center("Fichier enregistre : %s", filename);
}

void afficher_top5_et_export(const ListeDeputes *L) {
    if (!L || L->size <= 0) {
        sep_center();
        cprintf_center("Aucun depute charge.");
        sep_center();
        return;
    }

    ScoreItem *items = (ScoreItem*)malloc((size_t)L->size * sizeof(ScoreItem));
    if (!items) {
        sep_center();
        cprintf_center("Erreur memoire : impossible de calculer le classement.");
        sep_center();
        return;
    }

    // Calcul scores
    for (int i = 0; i < L->size; i++) {
        items[i].index = i;
        items[i].score = calculer_coherence(&L->data[i]);
    }

    // Tri decroissant (meilleurs d'abord)
    sort_scores_desc(items, L->size);

    int k = (L->size < 5) ? L->size : 5;

    sep_center();
    cprintf_center("CLASSEMENT COHERENCE");
    sep_center();

    printf("\n");
    cprintf_center("Top %d : plus coherents", k);
    subsep_center();

    for (int i = 0; i < k; i++) {
        const Depute *d = &L->data[items[i].index];
        cprintf_center("#%d  mp_id=%d  |  %s  |  %.1f %%",
                      i + 1, d->mp_id, d->nom, items[i].score);
    }

    printf("\n");
    cprintf_center("Top %d : moins coherents", k);
    subsep_center();

    for (int i = 0; i < k; i++) {
        int idx = L->size - 1 - i;
        const Depute *d = &L->data[items[idx].index];
        cprintf_center("#%d  mp_id=%d  |  %s  |  %.1f %%",
                      i + 1, d->mp_id, d->nom, items[idx].score);
    }

    printf("\n");
    sep_center();

    // Export fichier (sans diagrammes)
    export_top5_file(L, items, k);

    free(items);
}

/* =========================
   AIDE ECRAN (centre)
   ========================= */

void afficher_protocole_center(void) {
    sep_center();
    cprintf_center("AIDE / PROTOCOLE");
    sep_center();

    printf("\n");
    cprintf_center("IMPORTANT : eviter accents / caracteres speciaux (ASCII conseille).");
    cprintf_center("Si la console affiche mal, le CSV doit eviter UTF-8/accents.");

    printf("\n");
    cprintf_center("Selection : utiliser mp_id (identifiant unique).");
    cprintf_center("Axes programme : X=economie, Y=societal (valeurs -5..+5).");
    cprintf_center("X : -5 gauche eco ... +5 droite eco");
    cprintf_center("Y : -5 progressiste ... +5 conservateur/ordre");

    printf("\n");
    cprintf_center("Patrimoine : A(%s) B(%s) C(%s) D(%s) E(%s)",
                  patrimoine_intervalle('A'),
                  patrimoine_intervalle('B'),
                  patrimoine_intervalle('C'),
                  patrimoine_intervalle('D'),
                  patrimoine_intervalle('E'));

    printf("\n");
    cprintf_center("Votes : P=Pour | C=Contre | A=Abstention | N=Absent");
    cprintf_center("Themes : V001 Budg | V002 Fisc | V003 Immi | V004 Ener");
    cprintf_center("         V005 Soci | V006 Ecol | V007 Inst | V008 Envi");

    printf("\n");
    cprintf_center("Comparaison : un seul diagramme avec P1 V1 P2 V2 (positions des 2 deputes).");
    cprintf_center("Si plusieurs points superposes : **");

    printf("\n");
    sep_center();
}

/* =========================
   MENU (aere)
   ========================= */

void afficher_menu_center(void) {
    sep_center();
    cprintf_center("MENU");
    sep_center();

    printf("\n");
    cprintf_center("0. Aide / protocole");
    cprintf_center("1. Charger CSV");

    printf("\n");
    cprintf_center("2. Afficher liste des deputes");
    cprintf_center("3. Afficher profil complet (mp_id)");
    cprintf_center("4. Comparer deux deputes (mp_id, mp_id)");

    printf("\n");
    cprintf_center("5. Top 5 plus coherents / moins coherents");

    printf("\n");
    cprintf_center("9. Quitter");
    printf("\n");

    subsep_center();
}

/* =========================
   OUTILS : saisie mp_id avec "recommencer"
   ========================= */

int demander_mpid_avec_reessai(int *out_id, const char *prompt) {
    // Demande un mp_id.
    // - si saisie invalide -> message -> re-demande
    // - 0 = annuler et retourner au menu (retour 0)
    // - si OK -> retourne 1
    while (1) {
        int id = -1;

        if (!ask_center_int(&id, prompt)) {
            printf("\n");
            cprintf_center("Mauvaise entree. Reessayez (ex: 12). 0 = annuler.");
            printf("\n");
            continue;
        }

        if (id == 0) {
            // Annulation
            return 0;
        }

        if (id < 0) {
            printf("\n");
            cprintf_center("mp_id negatif interdit. Reessayez. 0 = annuler.");
            printf("\n");
            continue;
        }

        *out_id = id;
        return 1;
    }
}

/* =========================
   MAIN
   ========================= */

int main(void) {
    // Initialiser la liste
    ListeDeputes L;
    liste_init(&L);

    // CSV par defaut
    char fichier[256] = "deputes.csv";

    // Indique si on a charge
    int charge = 0;

    while (1) {
        clear_screen();
        afficher_menu_center();

        int choix = -1;

        // Lire choix menu
        if (!ask_center_int(&choix, "Choix : ")) {
            printf("\n");
            cprintf_center("Mauvaise entree : choisissez un numero du menu.");
            cprintf_center("Conseil : tapez 0 pour afficher l'aide.");
            printf("\n");
            cprintf_center("Appuyez sur Entree pour continuer...");
            pause_enter();
            continue;
        }

        // Quitter
        if (choix == 9) {
            clear_screen();
            sep_center();
            cprintf_center("Fin du programme.");
            sep_center();
            break;
        }

        // Aide
        if (choix == 0) {
            clear_screen();
            afficher_protocole_center();
            cprintf_center("Appuyez sur Entree pour revenir au menu...");
            pause_enter();
            continue;
        }

        // Charger CSV (avec reessai sur chemin si besoin)
        if (choix == 1) {
            clear_screen();
            sep_center();
            cprintf_center("CHARGEMENT CSV");
            sep_center();

            printf("\n");
            cprintf_center("Conseil : place le CSV dans le meme dossier que le programme.");
            printf("\n");

            while (1) {
                char buf[256];
                char prompt[256];

                snprintf(prompt, sizeof(prompt), "Nom du fichier CSV (Entree pour '%s') : ", fichier);
                ask_center_line(buf, sizeof(buf), prompt);

                // si vide -> garder fichier actuel
                if (buf[0] != '\0') {
                    strncpy(fichier, buf, sizeof(fichier) - 1);
                    fichier[sizeof(fichier) - 1] = '\0';
                }

                // tenter chargement
                if (!charger_deputes_csv(fichier, &L)) {
                    printf("\n");
                    cprintf_center("Echec chargement. Verifiez nom/chemin + format CSV.");
                    cprintf_center("Reessayez ou tapez 0 au menu pour aide.");

                    printf("\n");
                    // demander : recommencer ou annuler
                    char rep[8];
                    ask_center_line(rep, sizeof(rep), "Reessayer ? (O/N) : ");
                    if (rep[0] == 'N' || rep[0] == 'n') {
                        charge = 0;
                        break; // retour menu
                    }
                    // sinon on boucle et on redemande fichier
                } else {
                    printf("\n");
                    cprintf_center("Succes : %d deputes charges depuis '%s'.", L.size, fichier);
                    cprintf_center("Les deputes ont ete tries par mp_id (recherche rapide).");
                    charge = 1;
                    break;
                }
            }

            printf("\n");
            cprintf_center("Appuyez sur Entree pour revenir au menu...");
            pause_enter();
            continue;
        }

        // Si pas charge, on bloque proprement
        if (!charge) {
            clear_screen();
            sep_center();
            cprintf_center("Action impossible : aucun CSV charge.");
            cprintf_center("Chargez d'abord un CSV (option 1).");
            sep_center();

            printf("\n");
            cprintf_center("Appuyez sur Entree pour revenir au menu...");
            pause_enter();
            continue;
        }

        // Option 2 : liste
        if (choix == 2) {
            clear_screen();
            afficher_liste_center(&L);

            cprintf_center("Appuyez sur Entree pour revenir au menu...");
            pause_enter();
            continue;
        }

        // Option 3 : profil complet (reessai mp_id)
        if (choix == 3) {
            while (1) {
                clear_screen();
                sep_center();
                cprintf_center("PROFIL COMPLET");
                sep_center();

                printf("\n");
                cprintf_center("Entrez mp_id (0 = annuler).");
                printf("\n");

                int id = 0;
                if (!demander_mpid_avec_reessai(&id, "mp_id : ")) {
                    // annule -> menu
                    break;
                }

                int idx = trouver_depute_par_id(&L, id);
                if (idx < 0) {
                    printf("\n");
                    cprintf_center("Erreur : mp_id introuvable. Reessayez. (0 = annuler)");
                    printf("\n");
                    cprintf_center("Appuyez sur Entree pour continuer...");
                    pause_enter();
                    continue; // rester dans option 3
                }

                // Afficher + exporter
                clear_screen();
                export_profil(&L.data[idx]);

                printf("\n");
                cprintf_center("Appuyez sur Entree pour revenir a la saisie (ou 0 pour annuler).");
                pause_enter();
            }

            continue; // retour menu
        }

        // Option 4 : comparaison (reessai mp_id A et B)
        if (choix == 4) {
            while (1) {
                clear_screen();
                sep_center();
                cprintf_center("COMPARAISON DE 2 DEPUTES");
                sep_center();

                printf("\n");
                cprintf_center("Entrez mp_id (0 = annuler).");
                printf("\n");

                int idA = 0;
                if (!demander_mpid_avec_reessai(&idA, "mp_id 1 : ")) {
                    break; // annule
                }

                int idB = 0;
                if (!demander_mpid_avec_reessai(&idB, "mp_id 2 : ")) {
                    break; // annule
                }

                // Recherche
                int idxA = trouver_depute_par_id(&L, idA);
                int idxB = trouver_depute_par_id(&L, idB);

                if (idxA < 0 || idxB < 0) {
                    printf("\n");
                    cprintf_center("Erreur : un des mp_id est introuvable.");
                    cprintf_center("Reessayez. (0 = annuler)");
                    printf("\n");
                    cprintf_center("Appuyez sur Entree pour continuer...");
                    pause_enter();
                    continue; // rester dans option 4
                }

                // Afficher + exporter
                clear_screen();
                export_comparaison(&L.data[idxA], &L.data[idxB]);

                printf("\n");
                cprintf_center("Appuyez sur Entree pour refaire une comparaison (ou 0 pour annuler).");
                pause_enter();
            }

            continue; // retour menu
        }

        // Option 5 : Top 5 + export
        if (choix == 5) {
            clear_screen();
            afficher_top5_et_export(&L);

            printf("\n");
            cprintf_center("Appuyez sur Entree pour revenir au menu...");
            pause_enter();
            continue;
        }

        // Choix invalide
        clear_screen();
        sep_center();
        cprintf_center("Choix invalide. Reessayez.");
        sep_center();

        printf("\n");
        cprintf_center("Appuyez sur Entree pour revenir au menu...");
        pause_enter();
    }

    // Liberer la memoire
    liste_free(&L);

    return 0;
}
