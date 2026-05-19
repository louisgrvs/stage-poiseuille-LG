# Cylindre elliptique en écoulement de canal — Faibles nombres de Reynolds

Simulation et post-traitement du mouvement d'une particule elliptique dans un écoulement de Poiseuille en canal, dans le régime de Stokes. Le code calcule les positions d'équilibre, les plans de phase et le comportement de lubrification au voisinage de la paroi.

---

## Structure du dépôt

```
.
├── phase_plane_quarter_alpha2_betagoal.edp     # FreeFem++ — calcul du plan de phase
├── ellipse_poiseuille.edp                      # FreeFem++ — simulation temporelle de la trajectoire
├── lubrification_thetavar_alpha2_betagoal.edp  # FreeFem++ — analyse de lubrification en faisant varier l'angle
├── lubrification_thetafix_alpha2_betagoal.edp  # FreeFem++ — analyse de lubrification en faisant varier le centre
├── stokes_with_particle_torc.edp               # FreeFem++ — calcul du torque et des forces pour une particule à position fixée
├── affichage_plan_phase.ipynb                  # Python — visualisation du plan de phase
└── exposant_lubrification.ipynb                # Python — régression des exposants de lubrification
```

---

## Description des fichiers

### `phase_plane_quarter_alpha2_betagoal.edp`

Script FreeFem++ qui calcule le plan de phase `(Yc/d, θ/π)` pour une particule elliptique de rapport de forme `α = a/b = 2` et un paramètre `β = ab/d²` cible fixé via le paramètre `goal`.

Le script exploite les symétries du problème : seul le quart de domaine `Yc ∈ [0, d]`, `θ ∈ [0, π]` est calculé. Les trois autres quarts sont reconstruits analytiquement :

| Quadrant   | Transformation   |
|------------|------------------|
| (Y, −θ)    | (U, −V, +Ω)      |
| (−Y, θ)    | (U, +V, −Ω)      |
| (−Y, −θ)   | (U, −V, −Ω)      |

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

### `ellipse_poiseuille.edp`

Script FreeFem++ qui simule la dynamique temporelle d'une particule elliptique dans un écoulement de Poiseuille par intégration explicite d'Euler. À chaque pas de temps, le maillage est reconstruit autour de la position courante de la particule, puis quatre sous-problèmes de Stokes sont résolus (Poiseuille à corps fixe, translation x, translation y, rotation) pour construire la matrice de résistance et le second membre. La vitesse `(U, V, Ω)` est déduite par inversion du système 3×3, puis la position est mise à jour.

Les champs de vitesse sont visualisés à chaque pas avec une palette fixe et sauvegardés sous `poiseuille_plots/<step>.eps`. La trajectoire complète est écrite dans `trajectory.dat`.

**Paramètres clés :**

| Paramètre | Description |
|-----------|-------------|
| `Yc`      | Position verticale initiale (défaut : 0.2) |
| `theta`   | Angle d'orientation initial en radians (défaut : −π/2) |
| `a`, `b`  | Demi-axes de l'ellipse (défaut : 0.707·d et 0.354·d, soit α = 2) |
| `dt`      | Pas de temps (défaut : 0.2) |
| `Nstep`   | Nombre de pas de temps (défaut : 100) |
| `flow`    | Vitesse maximale du profil de Poiseuille (défaut : 1.0) |
| `ell`     | Demi-longueur du domaine de calcul (défaut : 12.0) |

**Fichiers de sortie :** `poiseuille_plots/<step>.eps`, `trajectory.dat`  
**Colonnes de `trajectory.dat` :** `t   Xc   Yc   theta`

---

### `lubrification_thetavar_alpha2_betagoal.edp`

Script FreeFem++ pour l'analyse de lubrification au voisinage de la paroi supérieure lorsque la particule tourne à centre `Yc` fixé. L'angle θ varie de 0 à π/2 selon une grille logarithmique sur le jeu `gapTop = d − Yc − yext(θ)`, concentrant les points près du contact. L'inversion de la relation `yext(θ)` se fait par dichotomie. Le maillage est raffiné automatiquement (ellipse et paroi supérieure) en fonction du gap courant.

Pour chaque configuration valide, les quatre sous-problèmes de Stokes sont résolus pour extraire la matrice de résistance complète et les vitesses libres `(U', V', Ω')`.

**Paramètres clés :**

| Paramètre | Description |
|-----------|-------------|
| `gap0`    | Jeu initial à θ = 0 (défaut : 0.3) |
| `Yc`      | Centre vertical, déduit de gap0 (= d − b − gap0) |
| `a`, `b`  | Demi-axes (défaut : a = 2b = 1.0, soit α = 2) |
| `marge`   | Jeu géométrique minimal (défaut : 0.015) |
| `NTheta`  | Nombre de points en θ dans la grille log (défaut : 40) |

**Fichier de sortie :** `lubrification_theta_rotation_Ycfixed.dat`  
**Colonnes de sortie :** `theta   gapTop   A11   A12   A13   A22   A23   A33   b1   b2   b3   d·Ω'/Umax   V'/Umax   U'/Umax`

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

### `stokes_with_particle_torc.edp`

Script FreeFem++ qui calcule le champ de vitesse et de pression autour d'une particule elliptique à position et orientation fixées, et en déduit la matrice de résistance complète ainsi que les vitesses libres `(U, V, Ω)`. Contrairement à `ellipse_poiseuille.edp`, ce script n'intègre pas en temps : il résout une seule configuration et produit des visualisations.

Les quatre sous-problèmes de Stokes sont résolus sur un domaine rectangulaire avec conditions de Poiseuille sur les bords latéraux et non-glissement sur les parois et l'ellipse. La matrice de résistance est assemblée par la formule de dissipation `Aij = ∫ 2μ D(uⁱ):D(uʲ) dx`.

**Paramètres clés :**

| Paramètre     | Description |
|---------------|-------------|
| `Yc`, `theta` | Position verticale et orientation de la particule (défaut : 0.0, π/2) |
| `a`, `b`      | Demi-axes de l'ellipse (défaut : 0.707 et 0.354, soit α ≈ 2) |
| `flow`        | Vitesse maximale du profil de Poiseuille (défaut : 1.0) |
| `d`           | Demi-largeur du canal (défaut : 1.0) |
| `ell`         | Demi-longueur du domaine de calcul (défaut : 4.0) |

**Fichiers de sortie :** `velocity_vectors.eps`, `velocity_magnitude.eps`, `velocity_field.eps`, `pressure.eps`

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

Les deux jeux de données (correspondant aux deux variantes de conditions aux bords) sont traités, chacun produisant une figure à 4 panneaux log-log avec les droites de régression superposées aux données brutes.

**Dépendances :** `numpy`, `matplotlib`, `scipy`

---

## Chaîne de traitement

```
FreeFem++ ellipse_poiseuille.edp
    └─► poiseuille_plots/<step>.eps   (visualisation de la dynamique)
    └─► trajectory.dat                (trajectoire t, Xc, Yc, θ)

FreeFem++ stokes_with_particle_torc.edp
    └─► velocity_vectors.eps / velocity_magnitude.eps / pressure.eps

FreeFem++ phase_plane_quarter_alpha2_betagoal.edp
    └─► phase_plane_quarter_alpha2_beta<XX>.dat
            └─► affichage_plan_phase.ipynb  (visualisation & détection des équilibres)

FreeFem++ lubrification_thetafix_alpha2_betagoal.edp
    └─► lubrification_alpha2_theta0pi.dat
    └─► lubrification_alpha2_theta0pi_Yrot0.dat
            └─► exposant_lubrification.ipynb

FreeFem++ lubrification_thetavar_alpha2_betagoal.edp
    └─► lubrification_theta_rotation_Ycfixed.dat
            └─► exposant_lubrification.ipynb
```

---

## Contexte physique

Le problème étudie le mouvement d'une particule elliptique rigide (demi-axes `a`, `b`, rapport de forme `α = a/b`) plongée dans un écoulement de Poiseuille dans un canal de demi-largeur `d`. Le paramètre adimensionnel `β = ab/d²` mesure la taille de la particule relativement au canal.

Dans la limite de Stokes (nombre de Reynolds nul), la vitesse de la particule `(U, V, Ω)` est reliée aux forces hydrodynamiques via une matrice de résistance linéaire. Les positions d'équilibre correspondent aux points du plan de phase `(Yc/d, θ/π)` où `V' = Ω' = 0`, c'est-à-dire où la particule se translate à la vitesse locale du fluide sans migration latérale ni rotation.

Au voisinage de la paroi, la théorie de lubrification prédit une divergence algébrique des coefficients de résistance en fonction du jeu `𝔡`. Les exposants sont retrouvés numériquement et comparés aux valeurs théoriques.

---

## Références

- Sugihara-Seki, M. — *The motion of an elliptical cylinder in channel flow at low Reynolds numbers*
- Bonheure, D., Hillairet, M., Patriarca, C., Sperone, G. — *Long-time behavior of an anisotropic rigid body interacting with a Poiseuille flow in an unbounded 2D channel*, arXiv:2406.01092 [math.AP], 2024. https://arxiv.org/abs/2406.01092