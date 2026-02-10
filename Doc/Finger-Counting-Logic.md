# Analyse de la logique de comptage de doigts

## Vue d'ensemble

L'application **x-cube-n6-ai-hand-landmarks** implémente un comptage de doigts levés (0 à 5) en exploitant les 21 landmarks (points de repère) détectés par le modèle de hand landmark. La logique est entièrement contenue dans la fonction `count_raised_fingers()` du fichier `Src/app.c`.

---

## Pipeline de traitement

```
Caméra → Palm Detector → Hand Landmark → Post-processing → Comptage de doigts → Affichage
```

### Étapes détaillées

1. **Palm Detector** : détecte la paume de la main et produit une ROI (Region Of Interest).
2. **Hand Landmark** : à partir de la ROI, le modèle infère 21 landmarks en coordonnées normalisées (x, y) dans le repère de la ROI.
3. **`ld_post_process()`** (`Src/ld.c`) : normalise les 63 floats de sortie (21 × 3 coordonnées) en 21 points `ld_point_t` avec x et y entre 0 et 1.
4. **`count_raised_fingers()`** (`Src/app.c`) : analyse les landmarks décodés en coordonnées écran pour déterminer combien de doigts sont levés.
5. **Affichage** : le nombre de doigts (0–5) est affiché en haut à gauche de l'écran.

---

## Convention des landmarks (MediaPipe Hand)

Le modèle utilise la convention MediaPipe avec 21 landmarks indexés comme suit :

```
          ┌─ TIP (4)
          │
          ├─ IP  (3)         ┌─ TIP (8)    ┌─ TIP (12)   ┌─ TIP (16)   ┌─ TIP (20)
          │                  │              │              │              │
          ├─ MCP (2)         ├─ DIP (7)     ├─ DIP (11)   ├─ DIP (15)   ├─ DIP (19)
          │                  │              │              │              │
          ├─ CMC (1)         ├─ PIP (6)     ├─ PIP (10)   ├─ PIP (14)   ├─ PIP (18)
          │                  │              │              │              │
          │                  ├─ MCP (5)     ├─ MCP (9)    ├─ MCP (13)   ├─ MCP (17)
          │                  │              │              │              │
          └──────────────────┴──────────────┴──────────────┴──────────────┘
                                            │
                                        Wrist (0)
```

| Doigt   | CMC/MCP | PIP | DIP | TIP |
|---------|---------|-----|-----|-----|
| Pouce   | 1 (CMC), 2 (MCP) | — | 3 (IP) | 4 |
| Index   | 5       | 6   | 7   | 8   |
| Majeur  | 9       | 10  | 11  | 12  |
| Annulaire | 13    | 14  | 15  | 16  |
| Auriculaire | 17  | 18  | 19  | 20  |

---

## Algorithme de comptage

### Étape 0 : Décodage des landmarks en coordonnées écran

Avant le comptage, chaque landmark brut (normalisé 0–1 dans le repère de la ROI) est converti en coordonnées écran par `decode_ld_landmark()` :

```c
decoded->x = roi->cx + (lm->x - 0.5) * w * cos(rotation) - (lm->y - 0.5) * h * sin(rotation);
decoded->y = roi->cy + (lm->x - 0.5) * w * sin(rotation) + (lm->y - 0.5) * h * cos(rotation);
```

Cette transformation applique :
- **Translation** : centrage sur le centre de la ROI (`roi->cx`, `roi->cy`)
- **Rotation** : compensation de l'angle de rotation de la main (`roi->rotation`)
- **Mise à l'échelle** : adaptation à la taille de la ROI (`w`, `h`)

### Étape 1 : Estimation du centre de la paume

```c
palm_cx = (lm[0].x + lm[9].x) / 2.0f;
```

Le centre horizontal de la paume est approximé comme le **milieu** entre :
- `lm[0]` = **Wrist** (poignet)
- `lm[9]` = **MCP du majeur** (base du majeur)

Ce point sert de **référence** uniquement pour la détection du pouce.

### Étape 2 : Détection du pouce (logique spéciale)

```c
tip_dist = |lm[4].x - palm_cx|;   // Distance TIP du pouce au centre paume
ip_dist  = |lm[3].x - palm_cx|;   // Distance IP du pouce au centre paume

if (tip_dist > ip_dist)  → Pouce levé ✋
```

**Pourquoi une logique différente ?**
Le pouce ne se plie pas verticalement comme les autres doigts : il s'ouvre **latéralement**. On compare donc la **distance horizontale** (en valeur absolue) du bout du pouce (TIP, index 4) au centre de la paume avec celle de l'articulation IP (index 3).

- **Pouce levé** : le TIP est **plus éloigné** du centre de la paume que l'IP → le pouce est déployé vers l'extérieur.
- **Pouce replié** : le TIP est **plus proche** du centre que l'IP → le pouce est replié vers la paume.

> ℹ️ Cette méthode fonctionne indépendamment de l'orientation gauche/droite de la main grâce à l'utilisation de la valeur absolue.

### Étape 3 : Détection des 4 autres doigts (index, majeur, annulaire, auriculaire)

Pour ces 4 doigts, la logique est **identique et simple** :

```c
if (TIP.y < PIP.y)  → Doigt levé ✋
```

| Doigt        | Condition                     |
|--------------|-------------------------------|
| Index        | `lm[8].y < lm[6].y`          |
| Majeur       | `lm[12].y < lm[10].y`        |
| Annulaire    | `lm[16].y < lm[14].y`        |
| Auriculaire  | `lm[20].y < lm[18].y`        |

**Principe** : En coordonnées écran, **Y augmente vers le bas**. Si le bout du doigt (TIP) a une coordonnée Y **plus petite** que l'articulation PIP, cela signifie que le TIP est **plus haut** que le PIP → le doigt est levé.

```
    TIP ● (y petit = haut)     Le doigt est LEVÉ ✅
         |
    DIP  ●
         |
    PIP  ● (y grand = bas)     Référence de comparaison
         |
    MCP  ●

    vs.

    PIP  ● (y petit)           Le doigt est REPLIÉ ❌
         |
    MCP  ●
        / 
    DIP ●
       |
    TIP ● (y grand = bas)
```

> ℹ️ On compare TIP avec PIP (et non MCP) pour avoir une détection plus **tolérante** : un doigt partiellement fléchi mais encore « levé » sera compté.

---

## Intégration dans la boucle principale

```c
// Src/app.c — Boucle d'inférence (nn_thread)

if (is_tracking)
    disp.info.finger_count = count_raised_fingers(&rois[0], ld_landmarks[0]);
else
    disp.info.finger_count = -1;   // -1 = pas de main détectée
```

Le comptage est effectué **à chaque frame** où une main est trackée. La valeur est stockée dans `disp.info.finger_count` :
- **-1** : aucune main détectée
- **0–5** : nombre de doigts levés

### Affichage

```c
// Src/app.c — display_info()

if (info->finger_count >= 0) {
    UTIL_LCD_SetFont(&GESTURE_FONT);        // Font24 sur DK
    UTIL_LCDEx_PrintfAt(10, 10, LEFT_MODE, "%d", info->finger_count);
    UTIL_LCD_SetFont(&LCD_FONT);
}
```

Le nombre est affiché en **gros caractères** (Font24) dans le coin supérieur gauche de l'écran LCD.

---

## Résumé visuel de l'algorithme

```
┌─────────────────────────────────────────────┐
│          count_raised_fingers()             │
│                                             │
│  1. Décoder 21 landmarks → coords écran    │
│                                             │
│  2. palm_cx = (Wrist.x + MCP9.x) / 2      │
│                                             │
│  3. POUCE:                                  │
│     |TIP(4).x - palm_cx| > |IP(3).x - ..| │
│     → OUI = pouce levé (+1)                │
│                                             │
│  4. INDEX:    TIP(8).y  < PIP(6).y  → +1  │
│  5. MAJEUR:   TIP(12).y < PIP(10).y → +1  │
│  6. ANNULAIRE:TIP(16).y < PIP(14).y → +1  │
│  7. AURICULAIRE:TIP(20).y < PIP(18).y → +1│
│                                             │
│  Retourne : count (0 à 5)                  │
└─────────────────────────────────────────────┘
```

---

## Limitations connues

| Limitation | Détail |
|------------|--------|
| **Orientation** | L'algorithme fonctionne principalement avec la main orientée **doigts vers le haut**. Si la main est retournée (doigts vers le bas), les comparaisons Y sont inversées et le comptage sera erroné. La rotation de la ROI compense partiellement ce problème. |
| **Pouce** | La détection du pouce par distance horizontale peut échouer si le pouce est plié vers l'intérieur à un angle inhabituel. |
| **Pas de handedness** | L'algorithme ne distingue pas main gauche/main droite, ce qui peut affecter la précision du pouce dans certaines positions. |
| **Seuil binaire** | Chaque doigt est soit levé soit replié — il n'y a pas de notion de doigt « partiellement levé ». |
| **Une seule main** | Le système ne gère qu'une seule main à la fois (`PD_MAX_HAND_NB` utilisé avec index 0 uniquement pour le comptage). |
