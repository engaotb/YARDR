# YARDR - Firebase Complete Setup

## 1. Firebase GenAI Extension Setup

### 1.1 Install Firebase GenAI Chatbot Extension

```bash
# Install the Firebase GenAI Chatbot extension
firebase ext:install googlecloud/firestore-genai-chatbot

# Configure the extension
firebase ext:configure googlecloud/firestore-genai-chatbot
```

### 1.2 Extension Configuration

```typescript
// Extension configuration parameters
export const genAIConfig = {
  // Gemini API Provider (google-ai or vertex-ai)
  geminiApiProvider: 'google-ai',
  
  // Gemini Model (gemini-1.5-flash or gemini-1.5-pro)
  geminiModel: 'gemini-1.5-flash',
  
  // Firestore collection path for conversations
  firestoreCollectionPath: 'ai-conversations/{userId}',
  
  // Response field name
  responseField: 'response',
  
  // Timestamp field for ordering
  timestampField: 'createTime',
  
  // AI Assistant context
  context: `You are a heavy equipment rental assistant for YARDR platform in Kuwait. 
  Help users find the right equipment for their construction tasks. 
  Consider local regulations, weather conditions, and project requirements.
  Provide specific equipment recommendations with brief explanations and estimated costs.
  Support both English and Arabic languages.`,
  
  // Model parameters
  temperature: 0.7,
  maxTokens: 500,
  
  // Enable Genkit monitoring
  enableGenkitMonitoring: true
};
```

## 2. Project Structure

### 2.1 Monorepo Structure (Recommended)

```
yardr-platform/
├── packages/
│   ├── shared/                    # Shared code for all apps
│   │   ├── types/                 # TypeScript interfaces
│   │   ├── utils/                 # Helper functions
│   │   ├── components/            # Shared UI components
│   │   └── firebase/              # Firebase configuration
│   └── functions/                 # Cloud Functions (shared backend)
│       ├── src/
│       │   ├── index.ts
│       │   ├── auth/
│       │   ├── orders/
│       │   ├── payments/
│       │   ├── notifications/
│       │   ├── ai/
│       │   ├── tracking/
│       │   └── admin/
│       └── package.json
├── apps/
│   ├── renter-app/               # Renter mobile app
│   │   ├── src/
│   │   ├── app.json
│   │   └── package.json
│   ├── owner-app/                # Owner mobile app
│   │   ├── src/
│   │   ├── app.json
│   │   └── package.json
│   ├── driver-app/               # Driver mobile app
│   │   ├── src/
│   │   ├── app.json
│   │   └── package.json
│   └── admin-web/                # Admin web dashboard
│       ├── src/
│       ├── next.config.js
│       └── package.json
├── firebase/
│   ├── firebase.json
│   ├── firestore.rules
│   ├── storage.rules
│   └── .firebaserc
└── package.json                  # Root package.json
```

### 2.2 What Each App Actually Needs

#### **Driver App** (Minimal - Only what drivers need):
```
driver-app/
├── src/
│   ├── screens/
│   │   ├── Dashboard.tsx
│   │   ├── Assignment.tsx
│   │   ├── Tracking.tsx
│   │   └── Profile.tsx
│   ├── services/
│   │   ├── location.ts          # GPS tracking
│   │   ├── notifications.ts     # Push notifications
│   │   └── orders.ts            # View assignments
│   └── App.tsx
```

**Driver App Functions Used:**
- `updateLocation` (tracking/)
- `calculateRoute` (tracking/)
- `sendPushNotification` (notifications/)

**Driver App DOESN'T Need:**
- Payment processing functions
- AI recommendation functions
- Admin functions
- Equipment management functions

#### **Renter App** (More features):
```
renter-app/
├── src/
│   ├── screens/
│   │   ├── Dashboard.tsx
│   │   ├── Search.tsx
│   │   ├── EquipmentDetails.tsx
│   │   ├── Booking.tsx
│   │   ├── Orders.tsx
│   │   └── Profile.tsx
│   ├── services/
│   │   ├── equipment.ts         # Search & browse
│   │   ├── orders.ts            # Create & manage orders
│   │   ├── payments.ts          # Payment processing
│   │   ├── ai.ts               # AI recommendations
│   │   └── notifications.ts     # Push notifications
│   └── App.tsx
```

**Renter App Functions Used:**
- `createOrder`, `updateStatus` (orders/)
- `processPayment` (payments/)
- `analyzeRequest`, `recommendEquipment` (ai/)
- `sendPushNotification` (notifications/)

#### **Owner App** (Fleet management):
```
owner-app/
├── src/
│   ├── screens/
│   │   ├── Dashboard.tsx
│   │   ├── Fleet.tsx
│   │   ├── Orders.tsx
│   │   ├── Analytics.tsx
│   │   └── Profile.tsx
│   ├── services/
│   │   ├── equipment.ts         # Manage fleet
│   │   ├── orders.ts            # View orders
│   │   ├── analytics.ts         # Revenue reports
│   │   └── notifications.ts     # Push notifications
│   └── App.tsx
```

#### **Admin Web** (Full access):
```
admin-web/
├── src/
│   ├── pages/
│   │   ├── dashboard/
│   │   ├── users/
│   │   ├── orders/
│   │   ├── analytics/
│   │   └── disputes/
│   ├── services/
│   │   ├── admin.ts             # All admin functions
│   │   ├── analytics.ts         # Platform analytics
│   │   └── notifications.ts     # System notifications
│   └── App.tsx
```

## Firestore Security Rules

```javascript
rules_version = '2';
service cloud.firestore {
  match /databases/{database}/documents {
    
    // Helper functions
    function isAuthenticated() {
      return request.auth != null;
    }
    
    function isOwner(userId) {
      return request.auth.uid == userId;
    }
    
    function hasRole(role) {
      return request.auth.token.role == role;
    }
    
    function isAdmin() {
      return hasRole('admin');
    }
    
    function isVerified() {
      return request.auth.token.verified == true;
    }
    
    // Users collection
    match /users/{userId} {
      allow read: if isAuthenticated();
      allow create: if isOwner(userId);
      allow update: if isOwner(userId) || isAdmin();
      allow delete: if isAdmin();
    }
    
    // Companies collection
    match /companies/{companyId} {
      allow read: if isAuthenticated();
      allow create: if isVerified();
      allow update: if resource.data.ownerId == request.auth.uid || isAdmin();
      allow delete: if isAdmin();
      
      // Company subcollections
      match /drivers/{driverId} {
        allow read: if isAuthenticated();
        allow write: if resource.data.ownerId == request.auth.uid || isAdmin();
      }
    }
    
    // Equipment collection
    match /equipment/{equipmentId} {
      allow read: if true; // Public read for browsing
      allow create: if isVerified() && hasRole('owner');
      allow update: if resource.data.ownerId == request.auth.uid || isAdmin();
      allow delete: if resource.data.ownerId == request.auth.uid || isAdmin();
    }
    
    // Orders collection
    match /orders/{orderId} {
      allow read: if request.auth.uid == resource.data.renterId || 
                     request.auth.uid == resource.data.ownerId ||
                     request.auth.uid == resource.data.driverId ||
                     isAdmin();
      allow create: if isVerified();
      allow update: if request.auth.uid in resource.data.parties || isAdmin();
      allow delete: if isAdmin();
      
      // Order subcollections
      match /tracking/{trackingId} {
        allow read: if request.auth.uid in resource.data.parties || isAdmin();
        allow write: if request.auth.uid == resource.data.driverId || isAdmin();
      }
    }
    
    // Transactions collection
    match /transactions/{transactionId} {
      allow read: if request.auth.uid == resource.data.userId || isAdmin();
      allow create: if false; // Only through Cloud Functions
      allow update: if false; // Only through Cloud Functions
      allow delete: if isAdmin();
    }
    
    // Notifications collection
    match /notifications/{userId}/messages/{messageId} {
      allow read: if isOwner(userId);
      allow create: if false; // Only through Cloud Functions
      allow update: if isOwner(userId); // Mark as read
      allow delete: if isOwner(userId);
    }
    
    // Analytics collection (admin only)
    match /analytics/{document=**} {
      allow read: if isAdmin();
      allow write: if false; // Only through Cloud Functions
    }
  }
}
```

## Firebase Configuration

```typescript
// firebase.config.ts
import { initializeApp } from 'firebase/app';
import { getAuth } from 'firebase/auth';
import { getFirestore } from 'firebase/firestore';
import { getFunctions } from 'firebase/functions';
import { getStorage } from 'firebase/storage';
import { getMessaging } from 'firebase/messaging';
import { getAnalytics } from 'firebase/analytics';

const firebaseConfig = {
  apiKey: process.env.FIREBASE_API_KEY,
  authDomain: process.env.FIREBASE_AUTH_DOMAIN,
  projectId: process.env.FIREBASE_PROJECT_ID,
  storageBucket: process.env.FIREBASE_STORAGE_BUCKET,
  messagingSenderId: process.env.FIREBASE_MESSAGING_SENDER_ID,
  appId: process.env.FIREBASE_APP_ID,
  measurementId: process.env.FIREBASE_MEASUREMENT_ID
};

const app = initializeApp(firebaseConfig);

export const auth = getAuth(app);
export const db = getFirestore(app);
export const functions = getFunctions(app);
export const storage = getStorage(app);
export const messaging = getMessaging(app);
export const analytics = getAnalytics(app);

// Enable offline persistence
enableIndexedDbPersistence(db).catch((err) => {
  if (err.code == 'failed-precondition') {
    console.log('Persistence failed: Multiple tabs open');
  } else if (err.code == 'unimplemented') {
    console.log('Persistence not available');
  }
});
```

## Firebase Project Setup Steps

### 1. Create Firebase Project

```bash
# Install Firebase CLI
npm install -g firebase-tools

# Login to Firebase
firebase login

# Create new project
firebase projects:create yardr-production

# Initialize Firebase in project directory
firebase init
```

### 2. Enable Authentication

```bash
# Enable Authentication providers
firebase auth:enable email
firebase auth:enable phone
```

### 3. Create Firestore Database

```bash
# Create Firestore database
firebase firestore:create

# Deploy security rules
firebase deploy --only firestore:rules
```

### 4. Set up Cloud Storage

```bash
# Create storage bucket
firebase storage:create

# Deploy storage rules
firebase deploy --only storage
```

### 5. Initialize Cloud Functions

```bash
# Initialize Cloud Functions
firebase init functions

# Install dependencies
cd functions
npm install

# Deploy functions
firebase deploy --only functions
```

### 6. Set up Firebase Hosting

```bash
# Initialize hosting
firebase init hosting

# Deploy admin dashboard
firebase deploy --only hosting
```

## Environment Configuration

### Development Environment

```bash
# .env.development
FIREBASE_API_KEY=your_dev_api_key
FIREBASE_AUTH_DOMAIN=yardr-dev.firebaseapp.com
FIREBASE_PROJECT_ID=yardr-dev
FIREBASE_STORAGE_BUCKET=yardr-dev.appspot.com
FIREBASE_MESSAGING_SENDER_ID=123456789
FIREBASE_APP_ID=1:123456789:web:abcdef
FIREBASE_MEASUREMENT_ID=G-ABCDEF123
```

### Production Environment

```bash
# .env.production
FIREBASE_API_KEY=your_prod_api_key
FIREBASE_AUTH_DOMAIN=yardr-production.firebaseapp.com
FIREBASE_PROJECT_ID=yardr-production
FIREBASE_STORAGE_BUCKET=yardr-production.appspot.com
FIREBASE_MESSAGING_SENDER_ID=987654321
FIREBASE_APP_ID=1:987654321:web:fedcba
FIREBASE_MEASUREMENT_ID=G-FEDCBA987
```

## Cloud Functions Setup

### Package.json Configuration

```json
{
  "name": "yardr-functions",
  "version": "1.0.0",
  "description": "YARDR Cloud Functions",
  "main": "lib/index.js",
  "scripts": {
    "build": "tsc",
    "serve": "npm run build && firebase emulators:start --only functions",
    "shell": "npm run build && firebase functions:shell",
    "start": "npm run shell",
    "deploy": "firebase deploy --only functions",
    "logs": "firebase functions:log"
  },
  "engines": {
    "node": "18"
  },
  "dependencies": {
    "firebase-admin": "^11.0.0",
    "firebase-functions": "^4.0.0",
    "@google-cloud/language": "^4.0.0",
    "axios": "^1.0.0",
    "twilio": "^4.0.0",
    "nodemailer": "^6.0.0"
  },
  "devDependencies": {
    "typescript": "^4.0.0",
    "@types/node": "^18.0.0"
  }
}
```

### TypeScript Configuration

```json
{
  "compilerOptions": {
    "module": "commonjs",
    "noImplicitReturns": true,
    "noUnusedLocals": true,
    "outDir": "lib",
    "sourceMap": true,
    "strict": true,
    "target": "es2017"
  },
  "compileOnSave": true,
  "include": [
    "src"
  ]
}
```

## Storage Rules

```javascript
rules_version = '2';
service firebase.storage {
  match /b/{bucket}/o {
    // Equipment photos - public read, owner write
    match /equipment-photos/{allPaths=**} {
      allow read: if true;
      allow write: if request.auth != null && 
                     request.auth.token.role == 'owner';
    }
    
    // User documents - private to user
    match /user-documents/{userId}/{allPaths=**} {
      allow read, write: if request.auth != null && 
                            request.auth.uid == userId;
    }
    
    // Company documents - private to company
    match /company-documents/{companyId}/{allPaths=**} {
      allow read, write: if request.auth != null && 
                            request.auth.token.companyId == companyId;
    }
    
    // Invoices - private to order parties
    match /invoices/{orderId}/{allPaths=**} {
      allow read: if request.auth != null && 
                     (request.auth.uid in resource.metadata.orderParties || 
                      request.auth.token.role == 'admin');
      allow write: if false; // Only through Cloud Functions
    }
  }
}
```

## Firestore Indexes

```json
{
  "indexes": [
    {
      "collectionGroup": "equipment",
      "queryScope": "COLLECTION",
      "fields": [
        {
          "fieldPath": "basic.category",
          "order": "ASCENDING"
        },
        {
          "fieldPath": "availability.status",
          "order": "ASCENDING"
        },
        {
          "fieldPath": "pricing.daily",
          "order": "ASCENDING"
        }
      ]
    },
    {
      "collectionGroup": "orders",
      "queryScope": "COLLECTION",
      "fields": [
        {
          "fieldPath": "parties.renterId",
          "order": "ASCENDING"
        },
        {
          "fieldPath": "status.current",
          "order": "ASCENDING"
        },
        {
          "fieldPath": "createdAt",
          "order": "DESCENDING"
        }
      ]
    },
    {
      "collectionGroup": "orders",
      "queryScope": "COLLECTION",
      "fields": [
        {
          "fieldPath": "parties.ownerId",
          "order": "ASCENDING"
        },
        {
          "fieldPath": "status.current",
          "order": "ASCENDING"
        },
        {
          "fieldPath": "createdAt",
          "order": "DESCENDING"
        }
      ]
    }
  ],
  "fieldOverrides": []
}
```

## Firebase Hosting Configuration

```json
{
  "hosting": {
    "public": "admin-dashboard/dist",
    "ignore": [
      "firebase.json",
      "**/.*",
      "**/node_modules/**"
    ],
    "rewrites": [
      {
        "source": "**",
        "destination": "/index.html"
      }
    ],
    "headers": [
      {
        "source": "**",
        "headers": [
          {
            "key": "X-Content-Type-Options",
            "value": "nosniff"
          },
          {
            "key": "X-Frame-Options",
            "value": "DENY"
          },
          {
            "key": "X-XSS-Protection",
            "value": "1; mode=block"
          }
        ]
      }
    ]
  }
}
```

## Deployment Commands

### Development Deployment

```bash
# Deploy to development project
firebase use yardr-dev
firebase deploy
```

### Production Deployment

```bash
# Deploy to production project
firebase use yardr-production
firebase deploy
```

### Selective Deployment

```bash
# Deploy only functions
firebase deploy --only functions

# Deploy only firestore rules
firebase deploy --only firestore:rules

# Deploy only hosting
firebase deploy --only hosting

# Deploy only storage rules
firebase deploy --only storage
```

## Monitoring and Analytics

### Firebase Analytics Events

```typescript
// Track custom events
import { logEvent } from 'firebase/analytics';

// User registration
logEvent(analytics, 'user_signup', {
  method: 'phone',
  role: 'renter'
});

// Equipment search
logEvent(analytics, 'equipment_search', {
  category: 'excavator',
  location: 'Kuwait City',
  ai_assisted: true
});

// Order creation
logEvent(analytics, 'order_created', {
  equipment_type: 'excavator',
  duration: 5,
  total_amount: 1750
});
```

### Performance Monitoring

```typescript
// Track performance metrics
import { getPerformance } from 'firebase/performance';

const perf = getPerformance();

// Track screen load time
const trace = trace(perf, 'screen_load_time');
trace.start();
// ... screen loading logic
trace.stop();
```

## Security Best Practices

### API Key Management

```typescript
// Use environment variables for sensitive data
const config = {
  apiKey: process.env.FIREBASE_API_KEY,
  // ... other config
};

// Never commit API keys to version control
// Use Firebase App Check for additional security
```

### Custom Claims

```typescript
// Set custom claims for role-based access
await admin.auth().setCustomUserClaims(uid, {
  role: 'owner',
  verified: true,
  companyId: 'company123'
});
```

### Data Validation

```typescript
// Validate data before writing to Firestore
const validateUserData = (data: any) => {
  if (!data.email || !data.phone) {
    throw new Error('Required fields missing');
  }
  // Additional validation logic
};
```
