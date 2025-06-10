# Guide de test et débogage du système de portes

## Tests essentiels

### 1. Test de positionnement
```javascript
// Ajouter dans le rendu pour visualiser les portes
if (DEBUG_MODE) {
    // Dessiner une ligne rouge au centre des cellules avec portes
    for (const door of doorSystem.doors.values()) {
        if (door.type === CELL_DOOR_NS) {
            // Ligne verticale à x = door.x + 0.5
        } else {
            // Ligne horizontale à y = door.y + 0.5
        }
    }
}
```

### 2. Test de collision
- **Approche frontale** : Le joueur doit être bloqué
- **Approche latérale** : Le joueur doit glisser le long
- **Porte ouverte** : Le joueur doit passer librement
- **Porte en mouvement** : Le joueur doit être bloqué jusqu'à ouverture suffisante

### 3. Test d'animation
```javascript
// Vérifier la cohérence du timer
console.log(`Porte ${x},${y}: état=${door.state}, timer=${door.timer.toFixed(2)}`);

// Le timer doit:
// - Rester entre 0 et 1
// - Augmenter/diminuer de manière fluide
// - S'arrêter aux extrémités
```

## Points de débogage

### 1. Console de débogage
```javascript
// Ajouter après castRayDDA
if (hit && hit.isDoor) {
    console.log(`Ray ${ray}: Porte touchée à distance ${hit.perpDist}`);
}
```

### 2. Visualisation des rayons
```javascript
// Mode debug : dessiner les rayons qui touchent des portes
if (DEBUG_RAYS && hit.isDoor) {
    // Dessiner une ligne du joueur au point d'impact
    debugDrawLine(player.x, player.y, hitX, hitY, 'yellow');
}
```

### 3. État des portes en temps réel
```javascript
// Afficher dans l'interface
document.getElementById('debug-info').innerHTML = `
    Portes: ${doorSystem.doors.size}<br>
    Actives: ${activeDoors}<br>
    Devant: ${frontDoor ? `${frontDoor.x},${frontDoor.y}` : 'Aucune'}
`;
```

## Problèmes courants et solutions

### Problème 1 : La porte ne s'ouvre pas
**Symptômes** : Appuyer sur ESPACE ne fait rien

**Vérifications** :
1. La cellule devant le joueur contient-elle une porte ?
```javascript
const checkX = Math.floor(player.x + Math.cos(player.angle));
const checkY = Math.floor(player.y + Math.sin(player.angle));
console.log(`Cellule devant: ${checkX},${checkY} = ${worldMap[checkX][checkY]}`);
```

2. La porte est-elle initialisée ?
```javascript
console.log(doorSystem.getDoor(checkX, checkY));
```

### Problème 2 : Le joueur traverse la porte fermée
**Symptômes** : La collision ne fonctionne pas

**Vérifications** :
```javascript
// Dans isDoorBlocking, ajouter des logs
console.log(`Collision check: cellX=${cellX}, cellY=${cellY}`);
console.log(`Distance au centre: ${Math.abs(cellX - 0.5)}`);
console.log(`Bloque: ${Math.abs(cellX - 0.5) < PLAYER_RADIUS}`);
```

### Problème 3 : La porte n'est pas visible
**Symptômes** : L'espace s'ouvre mais rien ne s'affiche

**Vérifications** :
1. Le raycasting détecte-t-il la porte ?
2. La texture est-elle correctement chargée ?
3. Les coordonnées de texture sont-elles valides ?

```javascript
if (hit.isDoor) {
    console.log(`Texture X: ${hit.textureX}`);
    console.log(`Wall height: ${wallHeight}`);
}
```

### Problème 4 : Animation saccadée
**Symptômes** : La porte s'ouvre par à-coups

**Solution** :
```javascript
// Vérifier deltaTime
console.log(`DeltaTime: ${deltaTime}, FPS: ${1/deltaTime}`);

// S'assurer que l'animation est basée sur le temps
door.timer += deltaTime * DOOR_SPEED; // Pas += 0.1
```

## Tests de performance

### Mesurer l'impact des portes
```javascript
// Profiler le raycasting
const t0 = performance.now();
const hit = castRayDDA(rayDirX, rayDirY);
const t1 = performance.now();
raycastTimes.push(t1 - t0);

// Afficher les statistiques
if (frameCount % 60 === 0) {
    const avg = raycastTimes.reduce((a,b) => a+b) / raycastTimes.length;
    console.log(`Raycast moyen: ${avg.toFixed(3)}ms`);
    raycastTimes = [];
}
```

## Configuration de test recommandée

### Map de test
```javascript
const testMap = [
    [1,1,1,1,1,1,1],
    [1,0,0,0,0,0,1],
    [1,0,0,0,0,0,1],
    [1,1,10,1,1,0,1], // Porte NS
    [1,0,0,0,1,0,1],
    [1,0,11,11,11,0,1], // 3 portes EW alignées
    [1,0,0,0,1,0,1],
    [1,1,1,1,1,1,1]
];
```

### Scénarios de test
1. **Traversée simple** : Ouvrir et passer une porte
2. **Portes multiples** : Ouvrir plusieurs portes en séquence
3. **Angles variés** : Approcher sous différents angles
4. **Stress test** : Beaucoup de portes qui s'ouvrent/ferment

## Outils de débogage visuels

### Mini-carte avec portes
```javascript
function drawMinimap() {
    const scale = 10;
    for (let x = 0; x < MAP_WIDTH; x++) {
        for (let y = 0; y < MAP_HEIGHT; y++) {
            const cell = worldMap[x][y];
            
            if (cell === CELL_DOOR_NS || cell === CELL_DOOR_EW) {
                const door = doorSystem.getDoor(x, y);
                // Couleur selon l'état
                ctx.fillStyle = door.state === DOOR_OPEN ? 'green' : 
                               door.state === DOOR_CLOSED ? 'red' : 'yellow';
            }
        }
    }
}
```

## Checklist de validation

- [ ] Les portes bloquent le joueur quand fermées
- [ ] Les portes s'ouvrent avec ESPACE
- [ ] L'animation est fluide (60 FPS maintenu)
- [ ] Les textures s'affichent correctement
- [ ] Les portes se referment correctement
- [ ] Pas de glitches visuels aux angles
- [ ] La collision fonctionne de tous les côtés
- [ ] Les performances restent acceptables
- [ ] Plusieurs portes peuvent être actives
- [ ] Le son fonctionne (si implémenté)