
# Guide Complet: Setup des Instances Test PM2

## Vue d'ensemble

Ce guide explique comment mettre en place des instances de test parallèles aux instances de production, permettant de tester les modifications avant de les déployer en production.

**Architecture finale:**
- `vms-yd` → Production YD (PORT: 3000)
- `vms-cn` → Production CN (PORT: 6002)
- `vms-yd-test` → Test YD (PORT: 6003)
- `vms-cn-test` → Test CN (PORT: 6001)

---

## ÉTAPE 1: Vérifier les scripts npm

### Objectif
S'assurer que les scripts npm `prod-test:yd` et `prod-test:cn` existent dans `package.json`

### Vérification
```bash
npm run prod-test:yd    # Doit démarrer l'instance test YD en local
npm run prod-test:cn    # Doit démarrer l'instance test CN en local
```

### État actuel
Les scripts existent déjà dans `package.json`:
```json
"prod-test:yd": "cross-env NODE_ENV=prod-yd-test node src/app.js",
"prod-test:cn": "cross-env NODE_ENV=prod-cn-test node src/app.js",
```

---

## ÉTAPE 2: Configurer le fichier ecosystem.config.js

### Objectif
Ajouter les instances de test `vms-yd-test` et `vms-cn-test` au fichier PM2

### Fichier à modifier
**`/ecosystem.config.js`**

### Contenu actuel
```javascript
module.exports = {
  apps: [
    {
      name: "vms-yd",
      script: "npm",
      args: "run prod:yd",
      env: {
        NODE_ENV: "prod:yd",
      },
    },
    {
      name: "vms-cn",
      script: "npm",
      args: "run prod:cn",
      env: {
        NODE_ENV: "prod:cn",
      },
    },
  ],
};
```

### Nouvelle configuration à ajouter
Ajouter les deux configurations de test:

```javascript
module.exports = {
  apps: [
    {
      name: "vms-yd",
      script: "npm",
      args: "run prod:yd",
      env: {
        NODE_ENV: "prod:yd",
      },
    },
    {
      name: "vms-cn",
      script: "npm",
      args: "run prod:cn",
      env: {
        NODE_ENV: "prod:cn",
      },
    },
    {
      name: "vms-yd-test",
      script: "npm",
      args: "run prod-test:yd",
      env: {
        NODE_ENV: "prod-yd-test",
      },
    },
    {
      name: "vms-cn-test",
      script: "npm",
      args: "run prod-test:cn",
      env: {
        NODE_ENV: "prod-cn-test",
      },
    },
  ],
};
```

### Explication des modifications
- **`name`**: Identifiant unique pour chaque instance PM2
- **`script`**: Commande à exécuter (`npm`)
- **`args`**: Arguments passés à npm (`run prod-test:yd` ou `run prod-test:cn`)
- **`env`**: Variables d'environnement (NODE_ENV pour charger le bon fichier .env)

---

## ÉTAPE 3: Créer la route /status

### Objectif
Créer un endpoint `/status` qui retourne les informations de l'environnement actuel

### Fichier à créer
**`/src/routes/statusRoutes.js`**

### Contenu
```javascript
const express = require('express');
const router = express.Router();
const { getEnv, loadEnvironment } = require('../../config/env');

/**
 * Route GET /status
 * Retourne les informations sur l'environnement et l'instance actuelle
 */
router.get('/', (req, res) => {
  try {
    loadEnvironment();
    
    const nodeEnv = process.env.NODE_ENV || 'unknown';
    const port = process.env.PORT || 'undefined';
    const dbName = process.env.DB_NAME || 'undefined';
    const apiVersion = process.env.API_VERSION || 'v1';
    const isProd = nodeEnv.startsWith('prod');
    const isTest = nodeEnv.includes('test');
    
    // Déterminer la région
    let region = 'unknown';
    if (nodeEnv.includes('yd')) region = 'YD (Youdao)';
    if (nodeEnv.includes('cn')) region = 'CN (China)';
    
    const statusInfo = {
      status: 'online',
      timestamp: new Date().toISOString(),
      environment: {
        NODE_ENV: nodeEnv,
        region: region,
        type: isTest ? 'test' : isProd ? 'production' : 'development',
        isProduction: isProd,
        isTest: isTest,
        isDevelopment: !isProd && !isTest,
      },
      server: {
        port: port,
        apiVersion: apiVersion,
      },
      database: {
        name: dbName,
        host: process.env.DB_HOST || 'undefined',
      },
      uptime: process.uptime(),
      memory: process.memoryUsage(),
    };

    res.status(200).json(statusInfo);
  } catch (error) {
    res.status(500).json({
      status: 'error',
      message: error.message,
      timestamp: new Date().toISOString(),
    });
  }
});

module.exports = router;
```

### Fichier à modifier
**`/src/app.js`**

### Modification requise
Ajouter l'enregistrement de la route status (après les autres routes):

```javascript
// Importer la route
const statusRoutes = require('./routes/statusRoutes');

// Enregistrer la route (ajouter cette ligne avec les autres app.use())
app.use('/status', statusRoutes);
```

### Exemple de réponse
```json
{
  "status": "online",
  "timestamp": "2026-04-13T10:30:45.123Z",
  "environment": {
    "NODE_ENV": "prod-cn-test",
    "region": "CN (China)",
    "type": "test",
    "isProduction": false,
    "isTest": true,
    "isDevelopment": false
  },
  "server": {
    "port": "3003",
    "apiVersion": "v1"
  },
  "database": {
    "name": "vms_db_cn_test",
    "host": "localhost"
  },
  "uptime": 12345.67,
  "memory": { ... }
}
```

---

## ÉTAPE 4: Vérifier les ports dans le fichiers de configuration .env.prod-test*

- `vms-yd-prod`: Port 3000
- `vms-cn-prod`: Port 6002
- `vms-yd-test`: Port 6003 ← Nouveau
- `vms-cn-test`: Port 6001 ← Nouveau

---

## ÉTAPE 5: Déployer sur le serveur aPanel

### Objectif
Mettre à jour la configuration PM2 sur le serveur de production

### Étapes de déploiement

#### A. Push les modifications localement
```bash
# Depuis /Users/md/Desktop/vms-backend (votre machine locale)
git add ecosystem.config.js src/routes/statusRoutes.js src/app.js
git commit -m "feat: add test instances for PM2 with status endpoint"
git push origin main

npm run deploy:yd
npm run deploy:cn
```

#### B. Sur le serveur aPanel
```bash
# Accédez au répertoire du backend
cd /www/wwwroot/cn-vms-backend.com/

# Testez les nouveaux scripts npm (optionnel)
npm run prod-test:yd    # Pour vérifier que ça fonctionne
# Ctrl+C pour arrêter après quelques secondes
```

---

## ÉTAPE 6: Recharger l'écosystème PM2

### Objectif
Appliquer la nouvelle configuration PM2 et démarrer les instances de test

### Commandes sur le serveur

#### 1- Vérifier l'état actuel
```bash
pm2 list
# Affiche les instances actuelles
```

#### 2- Recharger la configuration ecosystem
```bash
pm2 startOrRestart ecosystem.config.js
pm2 reload ecosystem.config.js
```

**Explication:**
- Cette commande lit le fichier `ecosystem.config.js`
- Elle démarre les instances qui ne sont pas en cours d'exécution
- Elle redémarre les instances qui existent déjà
- Les nouvelles instances `vms-yd-test` et `vms-cn-test` vont être démarrées

#### 3- Sauvegarder la configuration PM2
```bash
pm2 save
```

**Importance:**
- Sauvegarde l'état actuel de PM2
- Permet à PM2 de redémarrer automatiquement après un reboot du serveur

#### 4- Générer le script de démarrage
```bash
pm2 startup
```

---

## ÉTAPE 7: Vérification

### Objectif
Vérifier que toutes les instances sont en fonctionnement

### 1- Lister les instances PM2
```bash
pm2 list
```

**Résultat attendu:**
```
┌─────┬──────────────┬─────────┬─────────┬─────────┬──────────┐
│ id  │ name         │ version │ mode    │ status  │ restart  │
├─────┼──────────────┼─────────┼─────────┼─────────┼──────────┤
│ 0   │ vms-yd       │ 1.0.0   │ cluster │ online  │ 0        │
│ 1   │ vms-cn       │ 1.0.0   │ cluster │ online  │ 0        │
│ 2   │ vms-yd-test  │ 1.0.0   │ cluster │ online  │ 0        │
│ 3   │ vms-cn-test  │ 1.0.0   │ cluster │ online  │ 0        │
└─────┴──────────────┴─────────┴─────────┴─────────┴──────────┘
```

**Critères de succès:**
- ✓ Au moins 4 items affichés
- ✓ `vms-yd-test` présent avec status `online`
- ✓ `vms-cn-test` présent avec status `online`
- ✓ Colonne `status` affiche `online` pour tous

### 1- Vérifier les logs
```bash
# Logs de vms-yd-test
pm2 logs vms-yd-test

# Logs de vms-cn-test
pm2 logs vms-cn-test
```

### 2- Tester les endpoints /status

#### Test instance CN Test
```bash
curl http://192.168.1.97:6001/status
```

**Réponse attendue:**
```json
{
  "status": "online",
  "environment": {
    "NODE_ENV": "prod-cn-test",
    "region": "CN (China)",
    "type": "test"
  },
  "server": {
    "port": "6001"
  }
}
```

#### Test instance YD Test
```bash
curl http://192.168.1.97:6003/status
```

**Réponse attendue:**
```json
{
  "status": "online",
  "environment": {
    "NODE_ENV": "prod-yd-test",
    "region": "YD",
    "type": "test"
  },
  "server": {
    "port": "6003"
  }
}
```

### 4- Vérifier l'uptime
```bash
pm2 info vms-yd-test
pm2 info vms-cn-test
```

---

## 🔧 Cas de probleme possible

### Problème: Instance ne démarre pas (status: stopped)
```bash
# Vérifier les logs d'erreur
pm2 logs vms-cn-test --err

# Redémarrer manuelment
pm2 restart vms-cn-test

# Si ça ne marche toujours pas
pm2 delete vms-cn-test
pm2 start ecosystem.config.js --only vms-cn-test
```

### Problème: Port déjà utilisé
```bash
# Trouver le processus qui utilise le port (ex: 3003)
lsof -i :3003

# Arrêter l'instance et relancer
pm2 stop vms-yd-test
pm2 start vms-yd-test
```

### Problème: Erreur de base de données
```bash
# Vérifier que le fichier .env.prod-cn-test existe
ls -la .env.prod-cn-test
ls -la .env.prod-yd-test

# Vérifier les identifiants dans les fichiers
grep DB_NAME .env.prod-cn-test
grep DB_HOST .env.prod-cn-test
```

---

## Vue d'ensemble des fichiers modifiés

| Fichier | Action | Raison |
|---------|--------|--------|
| `ecosystem.config.js` | Modifier | Ajouter 2 nouvelles instances test |
| `src/routes/statusRoutes.js` | Créer | Nouvel endpoint /status |
| `src/app.js` | Modifier | Enregistrer la route /status |
| `.env.prod-yd-test` | Créer | Configuration pour instance test YD |
| `.env.prod-cn-test` | Créer | Configuration pour instance test CN |

---

## Résumé des commandes

### Déploiement
```bash
# Local
git push origin main
npm run deploy:cn
npm run deploy:yd
```

### Activation
```bash
# Sur le serveur
pm2 startOrRestart ecosystem.config.js
pm2 reload ecosystem.config.js
pm2 save
pm2 startup
```

### Vérification
```bash
# Sur le serveur
pm2 list
pm2 logs vms-cn-test
pm2 logs vms-yd-test
curl http://192.168.1.97:6003/status
curl http://192.168.1.97:6001/status
```

---

## Notes

- Les instances de test utilisent des bases de données **séparées** des instances de production
- Les ports de test sont différents
- Les fichiers `.env.prod-*-test` doivent être créés avec des identifiants de test
- L'endpoint `/status` aide à vérifier rapidement quel environnement répond
- PM2 redémarrera automatiquement les instances après un reboot du serveur (grâce à `pm2 startup`)

