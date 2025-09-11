# YARDR - Firestore Database Schema

## Complete Collection Schemas

### Users Collection

```typescript
interface User {
  uid: string;
  email: string;
  phone: string;
  phoneVerified: boolean;
  displayName: string;
  photoURL?: string;
  role: 'renter' | 'owner' | 'driver' | 'admin';
  companyId?: string; // For owners and drivers
  
  // Verification
  verified: boolean;
  verificationLevel: 'basic' | 'verified' | 'trusted';
  paciVerified?: boolean;
  paciData?: {
    civilId: string;
    fullName: string;
    nationality: string;
    verifiedAt: Timestamp;
  };
  
  // Wallet
  wallet: {
    balance: number;
    currency: 'KWD';
    holdAmount: number;
    lastUpdated: Timestamp;
  };
  
  // Profile
  profile: {
    address?: string;
    city?: string;
    area?: string;
    location?: GeoPoint;
    language: 'en' | 'ar';
    notificationPreferences: {
      push: boolean;
      email: boolean;
      sms: boolean;
    };
  };
  
  // Metadata
  createdAt: Timestamp;
  updatedAt: Timestamp;
  lastActive: Timestamp;
  expoPushTokens: string[];
  
  // Stats (for renters)
  renterStats?: {
    totalOrders: number;
    totalSpent: number;
    averageRating: number;
  };
}
```

### Companies Collection

```typescript
interface Company {
  id: string;
  ownerId: string;
  type: 'individual' | 'company';
  
  // Company Details
  details: {
    name: string;
    commercialLicense?: string;
    taxId?: string;
    address: string;
    city: string;
    phone: string;
    email: string;
    website?: string;
    logo?: string;
  };
  
  // Verification
  verification: {
    status: 'pending' | 'approved' | 'rejected';
    documents: {
      type: string;
      url: string;
      uploadedAt: Timestamp;
      verifiedAt?: Timestamp;
    }[];
    rejectionReason?: string;
    verifiedBy?: string;
  };
  
  // Banking
  banking: {
    bankName: string;
    accountName: string;
    accountNumber: string;
    iban: string;
    swiftCode?: string;
  };
  
  // Stats
  stats: {
    totalEquipment: number;
    totalDrivers: number;
    totalOrders: number;
    totalRevenue: number;
    averageRating: number;
    completionRate: number;
  };
  
  // Settings
  settings: {
    autoAcceptOrders: boolean;
    minimumOrderValue: number;
    operatingHours: {
      [key: string]: { open: string; close: string; };
    };
    serviceAreas: string[];
  };
  
  createdAt: Timestamp;
  updatedAt: Timestamp;
  active: boolean;
}
```

### Equipment Collection

```typescript
interface Equipment {
  id: string;
  ownerId: string;
  companyId: string;
  
  // Basic Info
  basic: {
    name: string;
    category: 'earthmoving' | 'lifting' | 'concrete' | 'compaction' | 'hauling' | 'specialized';
    type: string; // e.g., 'excavator', 'crane', 'bulldozer'
    brand: string;
    model: string;
    year: number;
    serialNumber?: string;
    registrationNumber: string;
    registrationExpiry: Timestamp;
  };
  
  // Specifications
  specs: {
    capacity: string; // e.g., "20 tons", "50m reach"
    enginePower?: string;
    fuelType?: 'diesel' | 'electric' | 'hybrid';
    dimensions?: {
      length: number;
      width: number;
      height: number;
      weight: number;
    };
    attachments?: string[];
    features?: string[];
  };
  
  // Availability
  availability: {
    status: 'available' | 'rented' | 'maintenance' | 'unavailable';
    calendar: {
      [date: string]: 'available' | 'booked' | 'maintenance';
    };
    nextAvailable?: Timestamp;
    location: GeoPoint;
    serviceRadius: number; // in km
  };
  
  // Pricing
  pricing: {
    hourly: number;
    daily: number;
    weekly: number;
    monthly: number;
    mobilization?: number; // delivery/pickup fee
    operator?: number; // operator fee if applicable
    currency: 'KWD';
    negotiable: boolean;
  };
  
  // Media
  media: {
    photos: string[];
    primaryPhoto: string;
    documents: {
      type: 'insurance' | 'registration' | 'inspection';
      url: string;
      expiryDate?: Timestamp;
    }[];
    videos?: string[];
  };
  
  // Operator/Driver
  operator: {
    required: boolean;
    assigned?: string; // driverId
    selfOperated: boolean;
  };
  
  // Maintenance
  maintenance: {
    lastService: Timestamp;
    nextService: Timestamp;
    hoursUsed: number;
    condition: 'excellent' | 'good' | 'fair';
    history: {
      date: Timestamp;
      type: string;
      description: string;
      cost: number;
    }[];
  };
  
  // Stats
  stats: {
    totalOrders: number;
    totalRevenue: number;
    totalHours: number;
    averageRating: number;
    utilizationRate: number;
  };
  
  // Search
  searchKeywords: string[];
  
  createdAt: Timestamp;
  updatedAt: Timestamp;
  active: boolean;
  featured: boolean;
}
```

### Orders Collection

```typescript
interface Order {
  id: string;
  orderNumber: string; // Human-readable
  
  // Parties
  parties: {
    renterId: string;
    renterName: string;
    renterPhone: string;
    ownerId: string;
    ownerName: string;
    ownerPhone: string;
    companyId: string;
    driverId?: string;
    driverName?: string;
    driverPhone?: string;
  };
  
  // Equipment
  equipment: {
    id: string;
    name: string;
    type: string;
    dailyRate: number;
    hourlyRate: number;
  };
  
  // Rental Details
  rental: {
    startDate: Timestamp;
    endDate: Timestamp;
    duration: number; // in days
    location: {
      address: string;
      coordinates: GeoPoint;
      instructions?: string;
    };
    purpose?: string;
    withOperator: boolean;
    needsDelivery: boolean;
  };
  
  // Status
  status: {
    current: 'draft' | 'pending' | 'accepted' | 'rejected' | 'paid' | 'active' | 
             'in_progress' | 'completed' | 'cancelled' | 'disputed';
    history: {
      status: string;
      timestamp: Timestamp;
      by: string;
      note?: string;
    }[];
  };
  
  // Pricing
  pricing: {
    equipmentCost: number;
    operatorCost?: number;
    deliveryCost?: number;
    subtotal: number;
    tax: number;
    discount?: number;
    total: number;
    currency: 'KWD';
    paid: boolean;
    paymentMethod?: 'wallet' | 'myfatoorah';
  };
  
  // Payment
  payment: {
    status: 'pending' | 'holding' | 'paid' | 'refunded';
    transactionId?: string;
    paidAt?: Timestamp;
    refundedAt?: Timestamp;
    refundAmount?: number;
    invoiceUrl?: string;
  };
  
  // Tracking
  tracking?: {
    started: boolean;
    startTime?: Timestamp;
    endTime?: Timestamp;
    currentLocation?: GeoPoint;
    route?: GeoPoint[];
    totalDistance?: number;
    actualHours?: number;
  };
  
  // Documents
  documents: {
    gatePass?: string[];
    deliveryPhotos?: string[];
    completionPhotos?: string[];
    customerSignature?: string;
    invoice?: string;
  };
  
  // Rating
  rating?: {
    byRenter?: {
      score: number;
      comment?: string;
      timestamp: Timestamp;
    };
    byOwner?: {
      score: number;
      comment?: string;
      timestamp: Timestamp;
    };
  };
  
  // Dispute
  dispute?: {
    raised: boolean;
    raisedBy: string;
    reason: string;
    description: string;
    evidence?: string[];
    status: 'open' | 'investigating' | 'resolved';
    resolution?: string;
    resolvedBy?: string;
    resolvedAt?: Timestamp;
  };
  
  // AI Assisted
  aiAssisted: boolean;
  aiRecommendation?: {
    problemDescription: string;
    recommendedEquipment: string[];
    confidence: number;
  };
  
  createdAt: Timestamp;
  updatedAt: Timestamp;
}
```

### Drivers Collection

```typescript
interface Driver {
  id: string;
  userId: string;
  companyId: string;
  
  // Personal Info
  personal: {
    fullName: string;
    phone: string;
    email?: string;
    civilId: string;
    nationality: string;
    dateOfBirth: Timestamp;
    photo?: string;
  };
  
  // License
  license: {
    number: string;
    type: string[];
    issueDate: Timestamp;
    expiryDate: Timestamp;
    documentUrl: string;
  };
  
  // Documents
  documents: {
    civilIdCopy: string;
    gatePass?: string;
    safetyTraining?: string;
    medicalCertificate?: string;
  };
  
  // Status
  status: {
    current: 'available' | 'busy' | 'off_duty';
    verified: boolean;
    active: boolean;
  };
  
  // Current Assignment
  currentAssignment?: {
    orderId: string;
    equipmentId: string;
    startTime: Timestamp;
    expectedEndTime: Timestamp;
  };
  
  // Stats
  stats: {
    totalTrips: number;
    totalHours: number;
    averageRating: number;
    completionRate: number;
    earnings: number;
  };
  
  // Settings
  settings: {
    notifications: boolean;
    shareLocation: boolean;
    language: 'en' | 'ar';
  };
  
  createdAt: Timestamp;
  updatedAt: Timestamp;
}
```

### Transactions Collection

```typescript
interface Transaction {
  id: string;
  type: 'credit' | 'debit' | 'hold' | 'release' | 'refund';
  
  // Parties
  userId: string;
  orderId?: string;
  
  // Amount
  amount: number;
  currency: 'KWD';
  
  // Details
  details: {
    description: string;
    method?: 'wallet' | 'myfatoorah' | 'bank';
    reference?: string;
    gatewayResponse?: any;
  };
  
  // Balance
  balanceBefore: number;
  balanceAfter: number;
  
  // Status
  status: 'pending' | 'processing' | 'completed' | 'failed';
  
  createdAt: Timestamp;
  processedAt?: Timestamp;
}
```

### Notifications Collection

```typescript
interface Notification {
  id: string;
  userId: string;
  
  // Content
  title: string;
  body: string;
  data?: any;
  
  // Type
  type: 'order' | 'payment' | 'system' | 'reminder' | 'marketing';
  priority: 'high' | 'normal' | 'low';
  
  // Delivery
  channels: ('push' | 'email' | 'sms')[];
  
  // Status
  read: boolean;
  readAt?: Timestamp;
  
  // Action
  action?: {
    type: string;
    payload: any;
  };
  
  createdAt: Timestamp;
  expiresAt?: Timestamp;
}
```

### Analytics Collection

```typescript
interface Analytics {
  id: string;
  date: string; // YYYY-MM-DD
  
  // Platform Metrics
  platform: {
    totalUsers: number;
    newUsers: number;
    activeUsers: number;
    totalOrders: number;
    completedOrders: number;
    cancelledOrders: number;
    totalRevenue: number;
    platformFees: number;
  };
  
  // User Metrics
  users: {
    byRole: {
      renters: number;
      owners: number;
      drivers: number;
    };
    byVerification: {
      basic: number;
      verified: number;
      trusted: number;
    };
  };
  
  // Equipment Metrics
  equipment: {
    total: number;
    available: number;
    rented: number;
    byCategory: { [key: string]: number };
    utilizationRate: number;
  };
  
  // Geographic Metrics
  geographic: {
    ordersByCity: { [key: string]: number };
    revenueByCity: { [key: string]: number };
  };
  
  // AI Metrics
  ai: {
    totalRequests: number;
    successfulMatches: number;
    conversionRate: number;
  };
  
  createdAt: Timestamp;
}
```

## Database Relationships

### Primary Relationships

1. **Users ↔ Companies**
   - One-to-many relationship
   - Users can belong to one company
   - Companies have multiple users (owners, drivers)

2. **Companies ↔ Equipment**
   - One-to-many relationship
   - Companies own multiple equipment
   - Equipment belongs to one company

3. **Users ↔ Orders**
   - Many-to-many relationship
   - Users can have multiple orders
   - Orders involve multiple users (renter, owner, driver)

4. **Equipment ↔ Orders**
   - One-to-many relationship
   - Equipment can have multiple orders
   - Orders involve one equipment

5. **Users ↔ Transactions**
   - One-to-many relationship
   - Users can have multiple transactions
   - Transactions belong to one user

### Subcollections

1. **Orders/{orderId}/tracking**
   - Real-time location tracking data
   - Driver location updates
   - Route history

2. **Companies/{companyId}/drivers**
   - Company-specific driver information
   - Driver assignments
   - Performance metrics

3. **Notifications/{userId}/messages**
   - User-specific notifications
   - Message history
   - Read status tracking

## Indexing Strategy

### Composite Indexes

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
  ]
}
```

### Single Field Indexes

- `equipment.basic.category`
- `equipment.availability.status`
- `orders.status.current`
- `users.role`
- `transactions.type`
- `notifications.type`

## Data Validation Rules

### User Data Validation

```typescript
const validateUser = (userData: Partial<User>) => {
  const errors: string[] = [];
  
  if (!userData.email || !isValidEmail(userData.email)) {
    errors.push('Valid email is required');
  }
  
  if (!userData.phone || !isValidKuwaitPhone(userData.phone)) {
    errors.push('Valid Kuwait phone number is required');
  }
  
  if (!userData.role || !['renter', 'owner', 'driver', 'admin'].includes(userData.role)) {
    errors.push('Valid role is required');
  }
  
  return errors;
};
```

### Equipment Data Validation

```typescript
const validateEquipment = (equipmentData: Partial<Equipment>) => {
  const errors: string[] = [];
  
  if (!equipmentData.basic?.name) {
    errors.push('Equipment name is required');
  }
  
  if (!equipmentData.basic?.category) {
    errors.push('Equipment category is required');
  }
  
  if (!equipmentData.pricing?.daily || equipmentData.pricing.daily <= 0) {
    errors.push('Valid daily pricing is required');
  }
  
  return errors;
};
```

### Order Data Validation

```typescript
const validateOrder = (orderData: Partial<Order>) => {
  const errors: string[] = [];
  
  if (!orderData.parties?.renterId) {
    errors.push('Renter ID is required');
  }
  
  if (!orderData.parties?.ownerId) {
    errors.push('Owner ID is required');
  }
  
  if (!orderData.equipment?.id) {
    errors.push('Equipment ID is required');
  }
  
  if (!orderData.rental?.startDate || !orderData.rental?.endDate) {
    errors.push('Rental dates are required');
  }
  
  if (orderData.rental?.startDate >= orderData.rental?.endDate) {
    errors.push('End date must be after start date');
  }
  
  return errors;
};
```

## Data Migration Strategies

### Version Control

```typescript
interface DataVersion {
  version: string;
  migrationDate: Timestamp;
  changes: string[];
  rollbackAvailable: boolean;
}

const dataVersions: DataVersion[] = [
  {
    version: '1.0.0',
    migrationDate: new Date('2024-01-01'),
    changes: ['Initial schema implementation'],
    rollbackAvailable: false
  },
  {
    version: '1.1.0',
    migrationDate: new Date('2024-02-01'),
    changes: ['Added AI recommendation fields', 'Enhanced tracking data'],
    rollbackAvailable: true
  }
];
```

### Migration Functions

```typescript
const migrateToVersion = async (targetVersion: string) => {
  const currentVersion = await getCurrentDataVersion();
  
  if (currentVersion === targetVersion) {
    return { success: true, message: 'Already at target version' };
  }
  
  try {
    await executeMigration(currentVersion, targetVersion);
    await updateDataVersion(targetVersion);
    return { success: true, message: 'Migration completed' };
  } catch (error) {
    return { success: false, message: 'Migration failed', error };
  }
};
```

## Backup and Recovery

### Automated Backups

```typescript
const scheduleBackups = () => {
  // Daily full backup
  cron.schedule('0 2 * * *', async () => {
    await createFullBackup();
  });
  
  // Hourly incremental backup
  cron.schedule('0 * * * *', async () => {
    await createIncrementalBackup();
  });
};
```

### Data Recovery

```typescript
const recoverData = async (backupDate: string, collections: string[]) => {
  try {
    const backup = await getBackup(backupDate);
    await restoreCollections(backup, collections);
    return { success: true, message: 'Data recovered successfully' };
  } catch (error) {
    return { success: false, message: 'Recovery failed', error };
  }
};
```

## Performance Optimization

### Query Optimization

```typescript
// Efficient queries with proper indexing
const getEquipmentByCategory = async (category: string) => {
  return await db.collection('equipment')
    .where('basic.category', '==', category)
    .where('availability.status', '==', 'available')
    .orderBy('pricing.daily')
    .limit(20)
    .get();
};

// Pagination for large datasets
const getOrdersPaginated = async (userId: string, lastDoc?: DocumentSnapshot) => {
  let query = db.collection('orders')
    .where('parties.renterId', '==', userId)
    .orderBy('createdAt', 'desc')
    .limit(10);
  
  if (lastDoc) {
    query = query.startAfter(lastDoc);
  }
  
  return await query.get();
};
```

### Caching Strategy

```typescript
const cacheEquipment = async (equipmentId: string) => {
  const equipment = await db.collection('equipment').doc(equipmentId).get();
  await redis.setex(`equipment:${equipmentId}`, 3600, JSON.stringify(equipment.data()));
};

const getCachedEquipment = async (equipmentId: string) => {
  const cached = await redis.get(`equipment:${equipmentId}`);
  return cached ? JSON.parse(cached) : null;
};
```
