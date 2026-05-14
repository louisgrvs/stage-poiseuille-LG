# Cylindre elliptique en écoulement de canal — Faibles nombres de Reynolds

Simulation et post-traitement du mouvement d'une particule elliptique dans un écoulement de Poiseuille en canal, dans le régime de Stokes. Le code calcule les positions d'équilibre, les plans de phase et le comportement de lubrification au voisinage de la paroi.

---

## Structure du dépôt

```
.
├── phase_plane_quarter_alpha2_betagoal.edp     # FreeFem++ — calcul du plan de phase
├── lubrification_thetafix_alpha2_betagoal.edp  # FreeFem++ — analyse de lubrification
├── affichage_plan_phase.ipynb                  # Python — visualisation du plan de phase
└── exposant_lubrification.ipynb                # Python — régression des exposants de lubrification
```

---

## Description des fichiers

### `phase_plane_quarter_alpha2_betagoal.edp`

Script FreeFem++ qui calcule le plan de phase `(Yc/d, θ/π)` pour une particule elliptique de rapport de forme `α = a/b = 2` et un paramètre `β = ab/d²` cible fixé via le paramètre `goal`.

Le script exploite les symétries du problème : seul le quart de domaine `Yc ∈ [0, d]`, `θ ∈ [0, π]` est calculé. Les trois autres quarts sont reconstruits analytiquement :

| Quadrant         | Transformation                        |
|------------------|---------------------------------------|
| (Y, −θ)          | (U, −V, +Ω)                           |
| (−Y, θ)          | (U, +V, −Ω)                           |
| (−Y, −θ)         | (U, −V, −Ω)                           |

Pour chaque point de grille `(Yc, θ)`, trois problèmes de Stokes sont résolus (translation en X, translation en Y, rotation) afin de construire la matrice de résistance et d'extraire la vitesse de la particule `(V', Ω')`. Les résultats sont écrits dans `phase_plane_quarter_alpha2_beta<XX>.dat`.

**Paramètres clés :**

| Paramètre | Description |
|-----------|-------------|
| `goal`    | β cible = ab/d² (contrôle la taille de la particule) |
| `d`       | Demi-largeur du canal (défaut : 6.0) |
| `Ntheta`  | Nombre de points en θ dans [0, π] (défaut : 25) |
| `NYc`     | Nombre de points en Yc/d dans [0, 1] (défaut : 21) |

**Colonnes de sortie :** `θ/π   Yc/d   V/Umax   d·Ω/Umax   U/Umax`

---

### `lubrification_thetafix_alpha2_betagoal.edp`

Script FreeFem++ pour l'analyse de lubrification au voisinage de la paroi, à `θ = 0` fixé. La particule est placée à des distances croissantes de la paroi supérieure selon une grille logarithmique, et les coefficients de résistance `A11`, `A13`, `A22`, `A33` sont calculés pour chaque position.

Deux variantes de conditions aux bords sont implémentées pour le sous-problème de rotation :
- **Corps rigide complet :** `v|∂B = (−(y − VB), x − UB)` (rotation référencée au centre de la particule)
- **Simplifiée :** `v|∂B = (−y, x − UB)` (rotation référencée à l'origine, résultats dans `*_Yrot0.dat`)

**Paramètres clés :**

| Paramètre | Description |
|-----------|-------------|
| `goal`    | β cible (contrôle la taille de la particule) |
| `d`       | Demi-largeur du canal (défaut : 6.0) |
| `NYclub`  | Nombre de points en distance à la paroi (défaut : 20, espacement log) |
| `marge`   | Jeu géométrique minimal (défaut : 0.02) |

**Fichiers de sortie :** `lubrification_alpha2_theta0pi.dat`, `lubrification_alpha2_theta0pi_Yrot0.dat`  
**Colonnes de sortie :** `1 − Yc/d   ...   A11   ...   A13   ...   A22   A33   ...`

---

### `affichage_plan_phase.ipynb`

Notebook Python pour la visualisation du plan de phase calculé par le script FreeFem++.

**Première partie — profils de vitesse :** charge les fichiers `.dat` pour plusieurs valeurs de β (`0.125`, `0.25`, `0.32`) et plusieurs décalages latéraux `Yd` (`0`, `0.15`, `0.3`). Trace les composantes de vitesse `U1/Umax` à `U4/Umax` en fonction de θ dans une grille de figures (lignes = composantes, colonnes = valeurs de β). Reproduit la figure 6 de Sugihara-Seki à titre de référence.

**Deuxième partie — plan de phase :** pour une valeur de β donnée, charge le quart calculé, reconstruit le plan complet par symétrie, et produit :
- Un fond `pcolormesh` montrant la magnitude `‖(V', Ω')‖`
- Un champ `quiver` de vecteurs vitesse normalisés
- Les iso-contours `V' = 0` (cyan) et `Ω' = 0` (orange) pour localiser les points d'équilibre
- Une détection automatique des points d'équilibre (changements de signe + seuil sur la magnitude)
- Une figure de vérification montrant les quatre quarts reconstruits

**Dépendances :** `numpy`, `matplotlib`

---

### `exposant_lubrification.ipynb`

Notebook Python pour extraire les exposants de loi de puissance des coefficients de résistance lorsque la particule s'approche de la paroi.

Effectue une régression linéaire log-log de `|Aij|` en fonction de `𝔡` (jeu entre la particule et la paroi) pour plusieurs seuils de distance (`0.05`, `0.10`, …, `0.30`). Les résultats sont comparés aux prédictions théoriques :

| Coefficient | Exposant théorique |
|-------------|-------------------|
| A11         | −1/2              |
| A13         | (?)               |
| A22         | −3/2              |
| A33         | −1/2              |

Les deux jeux de données (correspondant aux deux variantes de conditions aux bords du fichier `.edp`) sont traités, chacun produisant une figure à 4 panneaux log-log avec les droites de régression superposées aux données brutes.

**Dépendances :** `numpy`, `matplotlib`, `scipy`

---

## Chaîne de traitement

```
FreeFem++ phase_plane_quarter_alpha2_betagoal.edp
    └─► phase_plane_quarter_alpha2_beta<XX>.dat
            └─► affichage_plan_phase.ipynb  (visualisation & détection des équilibres)

FreeFem++ lubrification_thetafix_alpha2_betagoal.edp
    └─► lubrification_alpha2_theta0pi.dat
    └─► lubrification_alpha2_theta0pi_Yrot0.dat
            └─► exposant_lubrification.ipynb  (régression log-log & comparaison des exposants)
```

---

## Contexte physique

Le problème étudie le mouvement d'une particule elliptique rigide (demi-axes `a`, `b`, rapport de forme `α = a/b`) plongée dans un écoulement de Poiseuille dans un canal de demi-largeur `d`. Le paramètre adimensionnel `β = ab/d²` mesure la taille de la particule relativement au canal.

Dans la limite de Stokes (nombre de Reynolds nul), la vitesse de la particule `(U, V, Ω)` est reliée aux forces hydrodynamiques via une matrice de résistance linéaire. Les positions d'équilibre correspondent aux points du plan de phase `(Yc/d, θ/π)` où `V' = Ω' = 0`, c'est-à-dire où la particule se translate à la vitesse locale du fluide sans migration latérale ni rotation.

Au voisinage de la paroi, la théorie de lubrification prédit une divergence algébrique des coefficients de résistance en fonction du jeu `𝔡`. Les exposants sont retrouvés numériquement et comparés aux valeurs théoriques.

---

## Référence

- Sugihara-Seki, M. — *The motion of an elliptical cylinder in channel flow at low Reynolds numbers* (référencé dans `affichage_plan_phase.ipynb`)
Codes éléments finis méthode numérique pour analyse et compréhension d'un plan de phase d'une particule elliptique dans un liquide de Poiseuille
- Bonheure, D., Hillairet, M., Patriarca, C., Sperone, G. — Long-time behavior of an anisotropic rigid body interacting with a Poiseuille flow in an unbounded 2D channel, arXiv:2406.01092 [math.AP], 2024. https://arxiv.org/abs/2406.01092