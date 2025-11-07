# 📋 Documentation Complète - TP Application UFHB
## Système de Gestion des Contacts avec Upload d'Images

---

## 📋 **Table des Matières**
1. [Architecture Globale](#-architecture-globale)
2. [Configuration AWS](#-configuration-aws)
3. [Implémentation Backend](#-implémentation-backend)
4. [API Gateway](#-api-gateway)
5. [Frontend](#-frontend)
6. [Dépannage](#-dépannage)
7. [Annexes](#-annexes)

---

## 🏗️ **Architecture Globale**

### **Schéma d'Architecture AWS**
```
[Utilisateur] 
    ↓
[Frontend S3] → [API Gateway] → [Lambda] → [DynamoDB]
                            ↓
                         [S3 Uploads]
```

### **Composants AWS Utilisés**
- **Amazon S3** : Hébergement frontend + stockage images
- **AWS Lambda** : Logique métier backend
- **Amazon DynamoDB** : Base de données NoSQL
- **API Gateway** : Interface REST API
- **IAM** : Gestion des permissions

---

## ⚙️ **Configuration AWS**

### **1. Création des Buckets S3**

#### **Bucket Frontend : `ufhb-frontend-jjk`**
```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "PublicReadGetObject",
            "Effect": "Allow",
            "Principal": "*",
            "Action": "s3:GetObject",
            "Resource": "arn:aws:s3:::ufhb-frontend-jjk/*"
        }
    ]
}
```
**Configuration** : Site web statique activé

#### **Bucket Uploads : `ufhb-uploads-jjk`**
```json
[
    {
        "AllowedHeaders": ["*"],
        "AllowedMethods": ["PUT", "POST"],
        "AllowedOrigins": ["*"],
        "ExposeHeaders": []
    }
]
```

### **2. Configuration DynamoDB**
- **Table** : `ContactForms`
- **Clé primaire** : `id` (String)
- **Région** : `eu-north-1`

### **3. Fonction Lambda**
- **Nom** : `ufhb-api-handler`
- **Runtime** : Node.js 22.x
- **Rôle IAM** : Permissions DynamoDB + S3

---

## 🔧 **Implémentation Backend**

### **Structure de la Fonction Lambda**

#### **Dépendances AWS SDK**
```javascript
import { DynamoDBClient } from "@aws-sdk/client-dynamodb";
import { DynamoDBDocumentClient, PutCommand, ScanCommand } from "@aws-sdk/lib-dynamodb";
import { S3Client, PutObjectCommand } from "@aws-sdk/client-s3";
import { getSignedUrl } from "@aws-sdk/s3-request-presigner";
import { randomUUID } from "crypto";
```

#### **Configuration**
```javascript
const TABLE_NAME = "ContactForms";
const UPLOAD_BUCKET = "ufhb-uploads-jjk";
const FRONTEND_URL = "http://ufhb-frontend-jjk.s3-website.eu-north-1.amazonaws.com";

const corsHeaders = {
    'Access-Control-Allow-Origin': FRONTEND_URL,
    'Access-Control-Allow-Headers': 'Content-Type,X-Amz-Date,Authorization,X-Api-Key,X-Amz-Security-Token',
    'Access-Control-Allow-Methods': 'GET,POST,PUT,DELETE,OPTIONS',
    'Access-Control-Allow-Credentials': true,
    'Content-Type': 'application/json'
};
```

### **Gestion des Routes**

#### **Handler Principal**
```javascript
export const handler = async (event) => {
    // Gestion CORS preflight
    if (event.httpMethod === 'OPTIONS') {
        return { statusCode: 200, headers: corsHeaders, body: JSON.stringify({ message: 'CORS preflight' }) };
    }
    
    // Routage des requêtes
    if (event.resource === '/contact' && event.httpMethod === 'POST') {
        return await handleContact(event);
    }
    else if (event.resource === '/submissions' && event.httpMethod === 'GET') {
        return await handleSubmissions();
    }
    else if (event.resource === '/upload-url' && event.httpMethod === 'POST') {
        return await handleUploadUrl(event);
    }
    else {
        return { statusCode: 404, headers: corsHeaders, body: JSON.stringify({ error: 'Route non trouvée' }) };
    }
};
```

#### **Fonctions Spécifiques**

**1. Gestion des Contacts (`handleContact`)**
- Validation des champs requis (nom, email)
- Génération d'UUID unique
- Sauvegarde DynamoDB

**2. Récupération des Soumissions (`handleSubmissions`)**
- Scan de la table DynamoDB
- Tri par timestamp décroissant
- Limite de 100 résultats

**3. Génération d'URL d'Upload (`handleUploadUrl`)**
- Création de signed URLs S3
- Nom de fichier sécurisé
- Expiration après 1 heure

---

## 🌐 **API Gateway**

### **Configuration de l'API**
- **Nom** : `ufhb-api`
- **Type** : REST API
- **URL de base** : `https://wcbe3ec637.execute-api.eu-north-1.amazonaws.com/prod`

### **Endpoints Configurés**

| Méthode | Resource | Intégration | Description |
|---------|----------|-------------|-------------|
| `POST` | `/contact` | Lambda | Soumission nouveau contact |
| `GET` | `/submissions` | Lambda | Liste des soumissions |
| `POST` | `/upload-url` | Lambda | Génération URL upload |
| `OPTIONS` | Toutes | CORS | Préflight requests |

### **Configuration CORS**
- **Origines autorisées** : `http://ufhb-frontend-jjk.s3-website.eu-north-1.amazonaws.com`
- **Méthodes autorisées** : GET, POST, PUT, DELETE, OPTIONS
- **Headers autorisés** : Content-Type, Authorization, etc.

---

## 🎨 **Frontend**

### **Structure des Fichiers**
```
ufhb-frontend-jjk/
├── index.html          # Interface utilisateur
├── style.css          # Styles CSS
└── script.js          # Logique client
```

### **Fonctionnalités Implémentées**

#### **1. Gestion des Soumissions**
- Formulaire de contact avec validation
- Upload d'images via signed URLs
- Affichage des confirmations

#### **2. Interface d'Administration**
- Liste des soumissions avec pagination
- Fonction de suppression
- Affichage des métadonnées

#### **3. Gestion des États**
- Loading states pendant les requêtes
- Messages d'erreur/succès
- Validation en temps réel

### **Configuration API**
```javascript
const API_BASE = 'https://wcbe3ec637.execute-api.eu-north-1.amazonaws.com/prod';
```

---

## 🔐 **Gestion IAM**

### **Stratégies Attachées au Rôle Lambda**

1. **AmazonDynamoDBFullAccess**
   - PutItem, Scan, DeleteItem sur table ContactForms

2. **AmazonS3FullAccess**
   - PutObject sur bucket ufhb-uploads-jjk
   - GetObject sur bucket ufhb-frontend-jjk

### **Permissions Détaillées**
```json
{
    "Effect": "Allow",
    "Action": [
        "dynamodb:PutItem",
        "dynamodb:Scan", 
        "dynamodb:DeleteItem",
        "s3:PutObject",
        "s3:GetObject"
    ],
    "Resource": [
        "arn:aws:dynamodb:eu-north-1:*:table/ContactForms",
        "arn:aws:s3:::ufhb-uploads-jjk/*",
        "arn:aws:s3:::ufhb-frontend-jjk/*"
    ]
}
```

---

## 🐛 **Dépannage**

### **Problèmes Courants et Solutions**

#### **1. Erreur CORS**
**Symptôme** : "Blocked by CORS policy"
**Solution** : 
- Vérifier les headers CORS dans Lambda
- Configurer CORS dans API Gateway
- Vérifier l'URL frontend dans corsHeaders

#### **2. Erreur de Permissions**
**Symptôme** : "AccessDeniedException"
**Solution** :
- Vérifier les stratégies IAM du rôle Lambda
- S'assurer que DynamoDBFullAccess et S3FullAccess sont attachés

#### **3. Requêtes qui n'atteignent pas Lambda**
**Symptôme** : Aucun log dans CloudWatch
**Solution** :
- Vérifier l'intégration Lambda dans API Gateway
- Tester avec la fonction "Test" d'API Gateway

### **Monitoring et Logs**

#### **CloudWatch Logs**
```bash
# Patterns à surveiller
✅ Contact sauvegardé:
📊 X soumissions récupérées
📎 URL générée pour:
❌ Erreur:
```

#### **Test des Endpoints**
```bash
# Test santé API
curl -X GET https://wcbe3ec637.execute-api.eu-north-1.amazonaws.com/prod/submissions

# Test soumission
curl -X POST https://wcbe3ec637.execute-api.eu-north-1.amazonaws.com/prod/contact \
  -H "Content-Type: application/json" \
  -d '{"nom":"Test","email":"test@email.com","ufr":"Sciences"}'
```

---

## 📊 **Modèle de Données**

### **Table DynamoDB - ContactForms**

| Champ | Type | Description | Requis |
|-------|------|-------------|---------|
| `id` | String | UUID unique | ✅ Auto-généré |
| `nom` | String | Nom complet | ✅ |
| `email` | String | Adresse email | ✅ |
| `ufr` | String | UFR de rattachement | ❌ |
| `message` | String | Message optionnel | ❌ |
| `imageUrl` | String | URL de l'image S3 | ❌ |
| `timestamp` | Number | Timestamp Unix | ✅ Auto-généré |
| `createdAt` | String | Date ISO de création | ✅ Auto-généré |

---

## 🚀 **Guide de Déploiement**

### **Étapes de Mise en Production**

1. **Préparation des Buckets S3**
   ```bash
   aws s3 sync ./frontend s3://ufhb-frontend-jjk
   ```

2. **Déploiement Lambda**
   - Upload du code via console AWS
   - Configuration des variables d'environnement

3. **Configuration API Gateway**
   - Création des ressources et méthodes
   - Activation CORS
   - Déploiement sur le stage `prod`

4. **Tests de Validation**
   - Test de soumission de formulaire
   - Test d'upload d'image
   - Test de suppression

### **Vérifications Finales**

- [ ] Frontend accessible via URL S3
- [ ] API répond aux requêtes
- [ ] Soumissions sauvegardées en base
- [ ] Upload d'images fonctionnel
- [ ] Suppression opérationnelle

---

## 📈 **Améliorations Futures**

### **Fonctionnalités**
- [ ] Pagination des résultats
- [ ] Recherche et filtrage
- [ ] Export des données
- [ ] Authentification administrateur
- [ ] Notifications par email

### **Techniques**
- [ ] Mise en cache CloudFront
- [ ] Rate limiting API
- [ ] Monitoring avancé avec CloudWatch
- [ ] Sauvegardes automatiques DynamoDB

---

## 🔗 **URLs du Projet**

- **Frontend** : http://ufhb-frontend-jjk.s3-website.eu-north-1.amazonaws.com
- **API** : https://wcbe3ec637.execute-api.eu-north-1.amazonaws.com/prod
- **Bucket Uploads** : https://ufhb-uploads-jjk.s3.eu-north-1.amazonaws.com

---

## 💡 **Bonnes Pratiques Implémentées**

1. **Sécurité**
   - Validation des inputs
   - Signed URLs pour les uploads
   - CORS restrictifs

2. **Performance**
   - Scan limits DynamoDB
   - Tri côté serveur
   - URLs pré-signées S3

3. **Maintenabilité**
   - Logs structurés
   - Gestion centralisée des erreurs
   - Code modulaire

---

## 📞 **Support et Dépannage**

### **Ordre de Diagnostic**
1. ✅ Vérifier les logs CloudWatch Lambda
2. ✅ Tester les endpoints API Gateway
3. ✅ Vérifier les permissions IAM
4. ✅ Confirmer la configuration CORS
5. ✅ Tester la connectivité réseau

### **Contacts**
- **Documentation AWS** : [docs.aws.amazon.com](https://docs.aws.amazon.com)
- **Console AWS** : [eu-north-1.console.aws.amazon.com](https://eu-north-1.console.aws.amazon.com)

---

**Documentation générée le 07/11/2025**  
*TP Application UFHB - Système de Gestion des Contacts*  
*Architecture Cloud AWS Complète*
