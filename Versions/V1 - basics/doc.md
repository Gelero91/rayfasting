# Documentation Technique - Raycaster Wolfenstein 3D Optimisé

## Table des matières
1. [Vue d'ensemble](#vue-densemble)
2. [Architecture générale](#architecture-générale)
3. [Concepts mathématiques](#concepts-mathématiques)
4. [Structure des données](#structure-des-données)
5. [Système de rendu](#système-de-rendu)
6. [Optimisations implémentées](#optimisations-implémentées)
7. [Guide des fonctions](#guide-des-fonctions)
8. [Extension du code](#extension-du-code)

## Vue d'ensemble

Ce raycaster est une implémentation optimisée d'un moteur de rendu 3D pseudo-3D inspiré de Wolfenstein 3D. Il utilise la technique du ray casting pour créer une vue en perspective 3D à partir d'une carte 2D.

### Caractéristiques principales
- Rendu temps réel à 60+ FPS
- Textures pour murs, sol et plafond
- Système de brouillard pour la profondeur
- Collisions avec glissement le long des murs
- Optimisations avancées pour les performances

## Architecture générale

```
┌─────────────────┐
│   Entrée        │
│   Clavier       │
└────────┬────────┘
         │
┌────────▼────────┐
│  Mise à jour    │
│  Joueur         │
└────────┬────────┘
         │
┌────────▼────────┐     ┌──────────────┐
│ Rendu Sol/      │────►│ Buffer Pixels│
│ Plafond         │     └──────┬───────┘
└─────────────────┘            │
                               │
┌─────────────────┐            │
│ Rendu Murs      │────────────┤
└─────────────────┘            │
                               │
                     ┌─────────▼───────┐
                     │ Canvas Context  │
                     │ putImageData()  │
                     └─────────────────┘
```

## Concepts mathématiques

### 1. Ray Casting
Le ray casting consiste à lancer des rayons depuis la position du joueur à travers chaque colonne de l'écran pour détecter les murs.

```
Joueur (P) → Direction (D)
     │
     ├── Plan de caméra (perpendiculaire à D)
     │
     └── Rayons lancés à travers le plan
```

### 2. Plan de caméra
Le plan de caméra est perpendiculaire à la direction du joueur et définit le champ de vision (FOV).

```javascript
// Vecteurs du plan de caméra
planeX = -dy * 0.66  // Perpendiculaire à la direction
planeY = dx * 0.66   // Le facteur 0.66 donne un FOV de ~66°
```

### 3. Projection perspective
La hauteur d'un mur à l'écran est inversement proportionnelle à sa distance :
```
hauteurÉcran = hauteurMonde * distanceÉcran / distancePerpendiculaire
```

### 4. DDA (Digital Differential Analyzer)
Algorithme pour traverser efficacement la grille 2D :
```javascript
// Distance entre deux intersections de grille
deltaDistX = |1 / rayDirX|
deltaDistY = |1 / rayDirY|
```

## Structure des données

### Configuration globale
```javascript
const SCREEN_WIDTH = 640;        // Largeur de l'écran
const SCREEN_HEIGHT = 400;       // Hauteur de l'écran
const TEXTURE_SIZE = 64;         // Taille des textures (64x64)
const MAP_WIDTH = 24;            // Largeur de la carte
const MAP_HEIGHT = 24;           // Hauteur de la carte
const FOV = Math.PI / 3;         // Champ de vision (60°)
const NUM_RAYS = SCREEN_WIDTH/2; // Nombre de rayons (optimisation)
```

### État du joueur
```javascript
const player = {
    x: 3.5,        // Position X dans la carte
    y: 3.5,        // Position Y dans la carte
    angle: 0,      // Angle de rotation (radians)
    dx: 1,         // Direction X (cos(angle))
    dy: 0,         // Direction Y (sin(angle))
    planeX: 0,     // Plan caméra X
    planeY: 0.66,  // Plan caméra Y
    moveSpeed: 0,  // Vitesse actuelle
    rotSpeed: 0    // Vitesse de rotation
};
```

### Buffer de pixels
```javascript
let imageData = ctx.createImageData(SCREEN_WIDTH, SCREEN_HEIGHT);
let pixelBuffer = new Uint32Array(imageData.data.buffer);
```
- Format ABGR (32 bits par pixel)
- Manipulation directe pour les performances

### Carte du monde
```javascript
const worldMap = [
    [1,1,1,1,1,...],  // 1-4 : différents types de murs
    [1,0,0,0,0,...],  // 0 : espace vide
    ...
];
```

## Système de rendu

### 1. Pipeline de rendu
```
clearScreen() → renderFloorCeiling() → renderWalls() → putImageData()
```

### 2. Rendu du sol et plafond
Le sol et le plafond sont rendus par scanlines horizontales :

```javascript
for (let y = SCREEN_HEIGHT/2 + 1; y < SCREEN_HEIGHT; y++) {
    // Calculer la distance pour cette ligne
    const p = y - SCREEN_HEIGHT/2;
    const rowDistance = posZ / p;
    
    // Interpoler entre les rayons gauche et droit
    for (let x = 0; x < SCREEN_WIDTH; x++) {
        // Calculer position dans le monde
        // Récupérer texture
        // Appliquer brouillard
        // Dessiner pixel
    }
}
```

### 3. Rendu des murs
Les murs sont rendus colonne par colonne :

```javascript
for (let ray = 0; ray < NUM_RAYS; ray++) {
    // Calculer direction du rayon
    const cameraX = 2 * ray / NUM_RAYS - 1;
    const rayDirX = player.dx + player.planeX * cameraX;
    const rayDirY = player.dy + player.planeY * cameraX;
    
    // Lancer le rayon (DDA)
    const hit = castRayDDA(rayDirX, rayDirY);
    
    // Calculer hauteur du mur
    const wallHeight = SCREEN_HEIGHT / perpWallDist;
    
    // Dessiner la colonne
}
```

### 4. Système de brouillard
```javascript
if (distance > FOG_START) {
    fogFactor = (distance - FOG_START) / (FOG_END - FOG_START);
    color = blendColors(originalColor, fogColor, fogFactor);
}
```

## Optimisations implémentées

### 1. Manipulation directe des pixels
- Utilisation de `Uint32Array` au lieu de `fillRect()`
- Une seule écriture pour RGBA
- ~3x plus rapide

### 2. Tables de précalcul
```javascript
const sinTable = new Float32Array(360);
const cosTable = new Float32Array(360);
// Évite des milliers d'appels à Math.sin/cos
```

### 3. Stockage vertical des textures
```javascript
// Transposition pour améliorer la localité cache
for (let y = 0; y < TEXTURE_SIZE; y++) {
    for (let x = 0; x < TEXTURE_SIZE; x++) {
        transposed[x * TEXTURE_SIZE + y] = texture[y * TEXTURE_SIZE + x];
    }
}
```

### 4. Réduction du nombre de rayons
- `NUM_RAYS = SCREEN_WIDTH / 2`
- Utilisation de bandes plus larges
- Division par 2 des calculs

### 5. DDA optimisé
- Pas de boucle avec petits incréments
- Saut direct aux intersections de grille
- Calcul exact de la distance perpendiculaire

### 6. Rendu conditionnel
- Pas de redraw si le joueur n'a pas bougé
- Skip des pixels dans le brouillard complet

## Guide des fonctions

### updatePlayer(deltaTime)
Met à jour la position et rotation du joueur.
```javascript
// Entrées : deltaTime (temps écoulé)
// Sorties : modifie l'état du joueur
// Gère : rotation, déplacement, collisions
```

### checkCollision(x, y)
Vérifie si une position est valide (pas dans un mur).
```javascript
// Entrées : x, y (position à tester)
// Sorties : true si collision, false sinon
// Utilise : PLAYER_RADIUS pour collision circulaire
```

### renderFloorCeiling()
Rend le sol et le plafond par scanlines horizontales.
```javascript
// Entrées : état du joueur, textures
// Sorties : pixels dans pixelBuffer
// Méthode : scanline horizontale avec interpolation
```

### renderWalls()
Rend les murs colonne par colonne.
```javascript
// Entrées : état du joueur, carte, textures
// Sorties : pixels dans pixelBuffer, z-buffer
// Méthode : ray casting avec DDA
```

### castRayDDA(rayDirX, rayDirY)
Lance un rayon et trouve l'intersection avec un mur.
```javascript
// Entrées : direction du rayon (rayDirX, rayDirY)
// Sorties : {perpDist, tile, textureX, side}
// Algorithme : DDA (Digital Differential Analyzer)
```

### blendColors(color1, color2, factor)
Mélange deux couleurs (pour le brouillard).
```javascript
// Entrées : deux couleurs ABGR, facteur [0,1]
// Sorties : couleur mélangée
// Utilise : interpolation linéaire par canal
```

### TextureGenerator
Classe pour générer des textures procédurales.
```javascript
generateWallTexture(baseColor)   // Texture de mur avec briques
generateFloorTexture(pattern)    // Texture de sol
generateCeilingTexture()         // Texture de plafond
```

## Extension du code

### Ajouter des sprites
```javascript
// Structure sprite
const sprite = {
    x: 10.5,      // Position dans le monde
    y: 10.5,
    texture: 0,   // Index de texture
    scale: 1.0    // Échelle
};

// Rendu (après les murs)
// 1. Calculer distance et angle
// 2. Projeter sur l'écran
// 3. Vérifier z-buffer
// 4. Dessiner colonnes visibles
```

### Ajouter des portes
```javascript
// Dans la carte : valeurs spéciales (ex: 10-19)
// État de porte
const doors = {
    "10,5": { open: 0.0, opening: false } // 0=fermé, 1=ouvert
};

// Dans DDA : vérifier position dans la cellule
if (isDoor(tile)) {
    const doorOpen = doors[`${mapX},${mapY}`].open;
    // Ajuster la position d'intersection
}
```

### Améliorer l'éclairage
```javascript
// Éclairage dynamique simple
const lightSources = [
    { x: 5, y: 5, radius: 5, intensity: 1.0 }
];

// Dans le rendu
const distToLight = distance(x, y, light.x, light.y);
const lightFactor = Math.max(0, 1 - distToLight / light.radius);
color = applyLighting(color, lightFactor * light.intensity);
```

### Optimisations supplémentaires

#### 1. Web Workers
```javascript
// Diviser le rendu en sections
const worker = new Worker('render-worker.js');
worker.postMessage({
    section: 0,
    player: player,
    map: worldMap
});
```

#### 2. Occlusion culling
```javascript
// Marquer les colonnes déjà rendues
const columnRendered = new Array(SCREEN_WIDTH).fill(false);
// Skip si déjà couvert par un mur plus proche
```

#### 3. LOD (Level of Detail)
```javascript
// Réduire la qualité pour les objets lointains
if (distance > LOD_THRESHOLD) {
    textureSize = TEXTURE_SIZE / 2;
    skipLines = 2;
}
```

## Considérations de performance

### Profiling
```javascript
// Mesurer les sections critiques
const t0 = performance.now();
renderFloorCeiling();
const t1 = performance.now();
console.log(`Sol/plafond: ${t1 - t0}ms`);
```

### Goulots d'étranglement typiques
1. **Rendu sol/plafond** : Le plus coûteux (~40% du temps)
2. **Texture mapping** : Accès mémoire (~25%)
3. **Ray casting** : Calculs mathématiques (~20%)
4. **Manipulation pixels** : Écriture buffer (~15%)

### Cibles de performance
- 60 FPS sur matériel moderne (< 16.67ms/frame)
- 30 FPS sur matériel ancien (< 33.33ms/frame)
- Utilisation mémoire < 50MB

## Ressources et références

- [Lodev Raycasting Tutorial](https://lodev.org/cgtutor/raycasting.html)
- [Permadi Ray Casting Tutorial](https://permadi.com/1996/05/ray-casting-tutorial/)
- [Game Engine Black Book: Wolfenstein 3D](http://fabiensanglard.net/gebbwolf3d/)
- [Digital Differential Analyzer](https://en.wikipedia.org/wiki/Digital_differential_analyzer)

## Licence et crédits

Ce code est fourni à des fins éducatives. Les techniques de raycasting sont basées sur les travaux originaux d'id Software pour Wolfenstein 3D (1992).