# YARDR - Integration Specifications

## 1. Payment Gateway Integration

### 1.1 MyFatoorah Integration

#### Configuration

```typescript
// src/config/myfatoorah.ts
import { MFSDK } from 'myfatoorah-reactnative';

export const myfatoorahConfig = {
  apiKey: process.env.EXPO_PUBLIC_MYFATOORAH_API_KEY,
  baseURL: process.env.EXPO_PUBLIC_MYFATOORAH_BASE_URL || 'https://apitest.myfatoorah.com',
  directPaymentKey: process.env.EXPO_PUBLIC_MYFATOORAH_DIRECT_PAYMENT_KEY,
  currency: 'KWD',
  language: 'en'
};

export const MyFatoorahWrapper: React.FC<{ children: React.ReactNode }> = ({ children }) => {
  const configure = async () => {
    await MFSDK.init(
      myfatoorahConfig.apiKey,
      myfatoorahConfig.baseURL,
      myfatoorahConfig.directPaymentKey
    );
  };

  useEffect(() => {
    configure();
  }, []);

  return <>{children}</>;
};
```

#### Payment Processing

```typescript
// src/services/myfatoorah.ts
import { 
  MFSDK, 
  MFInitiateSessionRequest, 
  MFExecutePaymentRequest, 
  MFDirectPaymentRequest,
  MFCardRequest,
  MFLanguage,
  MFCurrencyISO
} from 'myfatoorah-reactnative';

export const MyFatoorahService = {
  async initiateSession(customerName: string = 'testCustomer') {
    try {
      const initiateSessionRequest = new MFInitiateSessionRequest(customerName);
      
      const response = await MFSDK.initiateSession(initiateSessionRequest);
      return response;
    } catch (error) {
      console.error('MyFatoorah session initiation error:', error);
      throw error;
    }
  },

  async executePayment(amount: number, sessionId: string, paymentMethodId?: number) {
    try {
      const executePaymentRequest = new MFExecutePaymentRequest(amount);
      executePaymentRequest.sessionId = sessionId;
      
      if (paymentMethodId) {
        executePaymentRequest.paymentMethodId = paymentMethodId;
      }

      const response = await MFSDK.executePayment(executePaymentRequest, MFLanguage.ENGLISH);
      return response;
    } catch (error) {
      console.error('MyFatoorah payment execution error:', error);
      throw error;
    }
  },

  async executeDirectPayment(amount: number, cardInfo: {
    cardNumber: string;
    cardExpiryMonth: string;
    cardExpiryYear: string;
    cardSecurityCode: string;
    saveToken?: boolean;
  }) {
    try {
      const executePaymentRequest = new MFExecutePaymentRequest(amount);
      executePaymentRequest.paymentMethodId = 20; // Direct payment method ID
      executePaymentRequest.displayCurrencyIso = MFCurrencyISO.KUWAIT_KWD;
      
      const mfCardRequest = new MFCardRequest(
        cardInfo.cardNumber,
        cardInfo.cardExpiryMonth,
        cardInfo.cardExpiryYear,
        cardInfo.cardSecurityCode,
        'YARDR',
        cardInfo.saveToken || false
      );

      const directPaymentRequest = new MFDirectPaymentRequest(executePaymentRequest, null, mfCardRequest);

      const response = await MFSDK.executeDirectPayment(
        directPaymentRequest, 
        MFLanguage.ENGLISH,
        (invoiceId: string) => console.log('Invoice ID:', invoiceId)
      );
      
      return response;
    } catch (error) {
      console.error('MyFatoorah direct payment error:', error);
      throw error;
    }
  },

  async getPaymentStatus(invoiceId: string) {
    try {
      const response = await MFSDK.getPaymentStatus(invoiceId);
      return response;
    } catch (error) {
      console.error('MyFatoorah payment status error:', error);
      throw error;
    }
  },

  async makeRefund(invoiceId: string, amount: number, reason: string) {
    try {
      const response = await MFSDK.makeRefund(invoiceId, amount, reason);
      return response;
    } catch (error) {
      console.error('MyFatoorah refund error:', error);
      throw error;
    }
  }
};
```

#### Backend API Endpoints

```typescript
// api/myfatoorah/create-payment.ts
import { NextApiRequest, NextApiResponse } from 'next';

export default async function handler(req: NextApiRequest, res: NextApiResponse) {
  if (req.method !== 'POST') {
    return res.status(405).json({ error: 'Method not allowed' });
  }

  try {
    const { amount, currency, customerInfo, orderId } = req.body;

    const response = await fetch(`${process.env.MYFATOORAH_BASE_URL}/v2/SendPayment`, {
      method: 'POST',
      headers: {
        'Content-Type': 'application/json',
        'Authorization': `Bearer ${process.env.MYFATOORAH_API_KEY}`
      },
      body: JSON.stringify({
        InvoiceAmount: amount,
        CurrencyIso: currency || 'KWD',
        CustomerName: customerInfo.name,
        CustomerEmail: customerInfo.email,
        CustomerMobile: customerInfo.phone,
        UserDefinedField: orderId,
        CallBackUrl: `${process.env.NEXT_PUBLIC_BASE_URL}/api/myfatoorah/callback`,
        ErrorUrl: `${process.env.NEXT_PUBLIC_BASE_URL}/api/myfatoorah/error`
      })
    });

    const data = await response.json();
    
    if (data.IsSuccess) {
      res.status(200).json({ 
        paymentUrl: data.Data.InvoiceURL,
        invoiceId: data.Data.InvoiceId 
      });
    } else {
      throw new Error(data.Message);
    }
  } catch (error) {
    console.error('MyFatoorah API error:', error);
    res.status(500).json({ error: 'Failed to create payment' });
  }
}
```

## 2. Google Maps Integration

### 2.1 Maps Configuration

```typescript
// src/config/maps.ts
import { GoogleMap, Marker, Polyline } from '@react-google-maps/api';

export const mapConfig = {
  apiKey: process.env.EXPO_PUBLIC_GOOGLE_MAPS_API_KEY,
  libraries: ['places', 'geometry'],
  defaultCenter: {
    lat: 29.3759, // Kuwait City
    lng: 47.9774
  },
  defaultZoom: 12,
  mapOptions: {
    disableDefaultUI: false,
    zoomControl: true,
    streetViewControl: false,
    mapTypeControl: true,
    fullscreenControl: true,
    styles: [
      {
        featureType: 'poi',
        elementType: 'labels',
        stylers: [{ visibility: 'off' }]
      }
    ]
  }
};
```

### 2.2 Location Services

```typescript
// src/services/location.ts
import * as Location from 'expo-location';
import { GoogleMap } from '@react-google-maps/api';

export const LocationService = {
  async getCurrentLocation() {
    try {
      const { status } = await Location.requestForegroundPermissionsAsync();
      if (status !== 'granted') {
        throw new Error('Location permission denied');
      }

      const location = await Location.getCurrentPositionAsync({
        accuracy: Location.Accuracy.High
      });

      return {
        latitude: location.coords.latitude,
        longitude: location.coords.longitude,
        accuracy: location.coords.accuracy
      };
    } catch (error) {
      console.error('Location error:', error);
      throw error;
    }
  },

  async watchLocation(callback: (location: any) => void) {
    try {
      const { status } = await Location.requestForegroundPermissionsAsync();
      if (status !== 'granted') {
        throw new Error('Location permission denied');
      }

      const subscription = await Location.watchPositionAsync(
        {
          accuracy: Location.Accuracy.High,
          timeInterval: 5000,
          distanceInterval: 10
        },
        callback
      );

      return subscription;
    } catch (error) {
      console.error('Location watch error:', error);
      throw error;
    }
  },

  async geocodeAddress(address: string) {
    try {
      const response = await fetch(
        `https://maps.googleapis.com/maps/api/geocode/json?address=${encodeURIComponent(address)}&key=${mapConfig.apiKey}`
      );
      
      const data = await response.json();
      
      if (data.status === 'OK' && data.results.length > 0) {
        const location = data.results[0].geometry.location;
        return {
          latitude: location.lat,
          longitude: location.lng,
          formattedAddress: data.results[0].formatted_address
        };
      }
      
      throw new Error('Address not found');
    } catch (error) {
      console.error('Geocoding error:', error);
      throw error;
    }
  },

  async reverseGeocode(latitude: number, longitude: number) {
    try {
      const response = await fetch(
        `https://maps.googleapis.com/maps/api/geocode/json?latlng=${latitude},${longitude}&key=${mapConfig.apiKey}`
      );
      
      const data = await response.json();
      
      if (data.status === 'OK' && data.results.length > 0) {
        return {
          formattedAddress: data.results[0].formatted_address,
          addressComponents: data.results[0].address_components
        };
      }
      
      throw new Error('Reverse geocoding failed');
    } catch (error) {
      console.error('Reverse geocoding error:', error);
      throw error;
    }
  }
};
```

### 2.3 Maps Components

```typescript
// src/components/maps/EquipmentMap.tsx
import React, { useState, useEffect } from 'react';
import { GoogleMap, Marker, InfoWindow } from '@react-google-maps/api';
import { Equipment } from '../../types/equipment';

interface EquipmentMapProps {
  equipment: Equipment[];
  selectedEquipment?: Equipment;
  onEquipmentSelect: (equipment: Equipment) => void;
  center?: { lat: number; lng: number };
  zoom?: number;
}

export const EquipmentMap: React.FC<EquipmentMapProps> = ({
  equipment,
  selectedEquipment,
  onEquipmentSelect,
  center,
  zoom = 12
}) => {
  const [mapCenter, setMapCenter] = useState(center || mapConfig.defaultCenter);

  const handleMarkerClick = (equipment: Equipment) => {
    onEquipmentSelect(equipment);
  };

  return (
    <GoogleMap
      mapContainerStyle={{ width: '100%', height: '100%' }}
      center={mapCenter}
      zoom={zoom}
      options={mapConfig.mapOptions}
    >
      {equipment.map((item) => (
        <Marker
          key={item.id}
          position={{
            lat: item.location.latitude,
            lng: item.location.longitude
          }}
          onClick={() => handleMarkerClick(item)}
          icon={{
            url: getEquipmentIcon(item.category),
            scaledSize: new google.maps.Size(40, 40)
          }}
        />
      ))}
      
      {selectedEquipment && (
        <InfoWindow
          position={{
            lat: selectedEquipment.location.latitude,
            lng: selectedEquipment.location.longitude
          }}
        >
          <div>
            <h3>{selectedEquipment.name}</h3>
            <p>{selectedEquipment.category}</p>
            <p>From {selectedEquipment.pricing.daily} KWD/day</p>
          </div>
        </InfoWindow>
      )}
    </GoogleMap>
  );
};

const getEquipmentIcon = (category: string) => {
  const iconMap: { [key: string]: string } = {
    'earth-moving': '/icons/excavator.png',
    'lifting': '/icons/crane.png',
    'hauling': '/icons/truck.png',
    'concrete': '/icons/concrete-mixer.png'
  };
  
  return iconMap[category] || '/icons/equipment.png';
};
```

## 3. PACI Civil ID Verification

### 3.1 PACI API Integration

```typescript
// src/services/paci.ts
export const PACIService = {
  async verifyCivilID(civilId: string, fullName: string, birthDate: string) {
    try {
      const response = await fetch('/api/paci/verify-civil-id', {
        method: 'POST',
        headers: {
          'Content-Type': 'application/json',
          'Authorization': `Bearer ${await getAuthToken()}`
        },
        body: JSON.stringify({
          civilId,
          fullName,
          birthDate,
          apiKey: process.env.EXPO_PUBLIC_PACI_API_KEY
        })
      });

      if (!response.ok) {
        throw new Error('PACI verification failed');
      }

      const result = await response.json();
      return result;
    } catch (error) {
      console.error('PACI verification error:', error);
      throw error;
    }
  },

  async validateCivilID(civilId: string) {
    // Basic validation for Kuwait Civil ID format
    const civilIdRegex = /^[12]\d{11}$/;
    
    if (!civilIdRegex.test(civilId)) {
      throw new Error('Invalid Civil ID format');
    }

    // Check checksum
    const isValidChecksum = this.calculateChecksum(civilId);
    if (!isValidChecksum) {
      throw new Error('Invalid Civil ID checksum');
    }

    return true;
  },

  calculateChecksum(civilId: string): boolean {
    const digits = civilId.slice(0, 11).split('').map(Number);
    const checkDigit = parseInt(civilId[11]);
    
    let sum = 0;
    for (let i = 0; i < 11; i++) {
      sum += digits[i] * (12 - i);
    }
    
    const remainder = sum % 11;
    const calculatedCheckDigit = remainder < 2 ? remainder : 11 - remainder;
    
    return calculatedCheckDigit === checkDigit;
  }
};
```

### 3.2 Backend PACI Integration

```typescript
// api/paci/verify-civil-id.ts
import { NextApiRequest, NextApiResponse } from 'next';

export default async function handler(req: NextApiRequest, res: NextApiResponse) {
  if (req.method !== 'POST') {
    return res.status(405).json({ error: 'Method not allowed' });
  }

  try {
    const { civilId, fullName, birthDate, apiKey } = req.body;

    // Validate API key
    if (apiKey !== process.env.PACI_API_KEY) {
      return res.status(401).json({ error: 'Invalid API key' });
    }

    // Call PACI API
    const paciResponse = await fetch(process.env.PACI_API_URL, {
      method: 'POST',
      headers: {
        'Content-Type': 'application/json',
        'Authorization': `Bearer ${process.env.PACI_ACCESS_TOKEN}`
      },
      body: JSON.stringify({
        civilId,
        fullName,
        birthDate
      })
    });

    if (!paciResponse.ok) {
      throw new Error('PACI API error');
    }

    const paciData = await paciResponse.json();
    
    // Store verification result
    await storeVerificationResult(civilId, paciData);

    res.status(200).json({
      verified: paciData.verified,
      details: paciData.details
    });
  } catch (error) {
    console.error('PACI verification error:', error);
    res.status(500).json({ error: 'Verification failed' });
  }
}
```

## 4. Firebase GenAI Chatbot Integration

### 4.1 Firebase GenAI Extension Setup

```typescript
// Firebase GenAI Chatbot Extension Configuration
export const genAIConfig = {
  // Extension configuration parameters
  geminiApiProvider: 'google-ai', // or 'vertex-ai'
  geminiModel: 'gemini-1.5-flash', // or 'gemini-1.5-pro'
  firestoreCollectionPath: 'ai-conversations/{userId}',
  responseField: 'response',
  timestampField: 'createTime',
  context: `You are a heavy equipment rental assistant for YARDR platform in Kuwait. 
  Help users find the right equipment for their construction tasks. 
  Consider local regulations, weather conditions, and project requirements.
  Provide specific equipment recommendations with brief explanations and estimated costs.
  Support both English and Arabic languages.`,
  temperature: 0.7,
  maxTokens: 500,
  enableGenkitMonitoring: true
};
```

### 4.2 AI Service Implementation

```typescript
// src/services/ai.ts
import { collection, addDoc, query, orderBy, limit, getDocs } from 'firebase/firestore';
import { db } from '../config/firebase';

export const AIService = {
  async getEquipmentRecommendations(userId: string, userQuery: string, userContext?: any) {
    try {
      // Create conversation document in Firestore
      const conversationRef = collection(db, `ai-conversations/${userId}`);
      
      const messageDoc = await addDoc(conversationRef, {
        prompt: userQuery,
        userContext: userContext || {},
        timestamp: new Date(),
        status: 'PENDING'
      });

      // The Firebase GenAI extension will automatically process this document
      // and add the response field when complete
      
      return {
        messageId: messageDoc.id,
        status: 'processing'
      };
    } catch (error) {
      console.error('AI service error:', error);
      throw new Error('Failed to get AI recommendations');
    }
  },

  async getConversationHistory(userId: string, limitCount: number = 10) {
    try {
      const conversationRef = collection(db, `ai-conversations/${userId}`);
      const q = query(conversationRef, orderBy('createTime', 'desc'), limit(limitCount));
      
      const querySnapshot = await getDocs(q);
      const conversations = querySnapshot.docs.map(doc => ({
        id: doc.id,
        ...doc.data()
      }));

      return conversations.reverse(); // Return in chronological order
    } catch (error) {
      console.error('Get conversation history error:', error);
      throw error;
    }
  },

  async regenerateResponse(userId: string, messageId: string) {
    try {
      // Update the status to trigger regeneration
      const messageRef = doc(db, `ai-conversations/${userId}`, messageId);
      await updateDoc(messageRef, {
        status: 'PENDING'
      });

      return { status: 'regenerating' };
    } catch (error) {
      console.error('Regenerate response error:', error);
      throw error;
    }
  },

  async generateEquipmentDescription(equipmentData: any) {
    try {
      const prompt = `Generate a professional equipment description for a rental platform. 
      Equipment: ${equipmentData.name}
      Category: ${equipmentData.category}
      Specifications: ${JSON.stringify(equipmentData.specifications)}
      
      Include key specifications, use cases, and benefits. Keep it concise and engaging.`;

      // Create a temporary conversation for equipment description
      const tempConversationRef = collection(db, 'ai-conversations/temp');
      const messageDoc = await addDoc(tempConversationRef, {
        prompt: prompt,
        timestamp: new Date(),
        status: 'PENDING'
      });

      return {
        messageId: messageDoc.id,
        status: 'processing'
      };
    } catch (error) {
      console.error('Equipment description error:', error);
      throw error;
    }
  },

  async analyzeProjectRequirements(projectDescription: string) {
    try {
      const prompt = `Analyze this construction project and recommend equipment needed:
      
      Project Description: ${projectDescription}
      
      Consider project size, duration, location, and specific tasks. 
      Provide equipment recommendations with explanations.`;

      const tempConversationRef = collection(db, 'ai-conversations/temp');
      const messageDoc = await addDoc(tempConversationRef, {
        prompt: prompt,
        timestamp: new Date(),
        status: 'PENDING'
      });

      return {
        messageId: messageDoc.id,
        status: 'processing'
      };
    } catch (error) {
      console.error('Project analysis error:', error);
      throw error;
    }
  }
};
```

### 4.3 Real-time AI Response Monitoring

```typescript
// src/hooks/useAIResponse.ts
import { useState, useEffect } from 'react';
import { doc, onSnapshot } from 'firebase/firestore';
import { db } from '../config/firebase';

export const useAIResponse = (userId: string, messageId: string) => {
  const [response, setResponse] = useState<string | null>(null);
  const [status, setStatus] = useState<'PENDING' | 'COMPLETED' | 'ERROR'>('PENDING');
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    if (!messageId) return;

    const messageRef = doc(db, `ai-conversations/${userId}`, messageId);
    
    const unsubscribe = onSnapshot(messageRef, (doc) => {
      if (doc.exists()) {
        const data = doc.data();
        setStatus(data.status);
        
        if (data.status === 'COMPLETED' && data.response) {
          setResponse(data.response);
          setLoading(false);
        } else if (data.status === 'ERROR') {
          setLoading(false);
        }
      }
    });

    return () => unsubscribe();
  }, [userId, messageId]);

  return { response, status, loading };
};
```

## 5. Firebase Hosting Integration

### 5.1 Hosting Configuration

```typescript
// src/config/firebaseHosting.ts
export const hostingConfig = {
  projectId: process.env.EXPO_PUBLIC_FIREBASE_PROJECT_ID,
  siteId: process.env.FIREBASE_HOSTING_SITE_ID || 'yardr-app',
  domain: 'yardr.com',
  cdnUrl: 'https://yardr.com'
};

export const HostingService = {
  async deploySite(buildPath: string) {
    try {
      const response = await fetch('/api/hosting/deploy', {
        method: 'POST',
        headers: {
          'Content-Type': 'application/json',
          'Authorization': `Bearer ${await getAuthToken()}`
        },
        body: JSON.stringify({
          buildPath,
          siteId: hostingConfig.siteId
        })
      });

      return await response.json();
    } catch (error) {
      console.error('Firebase Hosting deploy error:', error);
      throw error;
    }
  },

  async uploadFile(file: File, path: string) {
    try {
      const formData = new FormData();
      formData.append('file', file);
      formData.append('path', path);

      const response = await fetch('/api/hosting/upload', {
        method: 'POST',
        body: formData
      });

      return await response.json();
    } catch (error) {
      console.error('Firebase Hosting upload error:', error);
      throw error;
    }
  }
};
```

### 5.2 Image Optimization

```typescript
// src/utils/imageOptimization.ts
export const ImageOptimization = {
  getOptimizedImageUrl(originalUrl: string, options: {
    width?: number;
    height?: number;
    quality?: number;
    format?: 'webp' | 'jpeg' | 'png';
  } = {}) {
    const { width, height, quality = 80, format = 'webp' } = options;
    
    const params = new URLSearchParams();
    if (width) params.append('width', width.toString());
    if (height) params.append('height', height.toString());
    params.append('quality', quality.toString());
    params.append('format', format);
    
    return `${hostingConfig.cdnUrl}/images/${originalUrl}?${params.toString()}`;
  },

  generateResponsiveImages(originalUrl: string, sizes: number[]) {
    return sizes.map(size => ({
      url: this.getOptimizedImageUrl(originalUrl, { width: size }),
      width: size
    }));
  }
};
```

## 6. Push Notifications

### 6.1 Firebase Cloud Messaging

```typescript
// src/services/notifications.ts
import messaging from '@react-native-firebase/messaging';
import { Platform } from 'react-native';

export const NotificationService = {
  async requestPermission() {
    try {
      const authStatus = await messaging().requestPermission();
      const enabled =
        authStatus === messaging.AuthorizationStatus.AUTHORIZED ||
        authStatus === messaging.AuthorizationStatus.PROVISIONAL;

      if (enabled) {
        const token = await messaging().getToken();
        await this.saveTokenToServer(token);
        return token;
      }
      
      throw new Error('Permission denied');
    } catch (error) {
      console.error('Notification permission error:', error);
      throw error;
    }
  },

  async saveTokenToServer(token: string) {
    try {
      await fetch('/api/notifications/save-token', {
        method: 'POST',
        headers: {
          'Content-Type': 'application/json',
          'Authorization': `Bearer ${await getAuthToken()}`
        },
        body: JSON.stringify({
          token,
          platform: Platform.OS,
          userId: await getCurrentUserId()
        })
      });
    } catch (error) {
      console.error('Save token error:', error);
    }
  },

  async sendNotification(recipientId: string, notification: {
    title: string;
    body: string;
    data?: any;
  }) {
    try {
      await fetch('/api/notifications/send', {
        method: 'POST',
        headers: {
          'Content-Type': 'application/json',
          'Authorization': `Bearer ${await getAuthToken()}`
        },
        body: JSON.stringify({
          recipientId,
          notification,
          platform: Platform.OS
        })
      });
    } catch (error) {
      console.error('Send notification error:', error);
      throw error;
    }
  },

  setupNotificationHandlers() {
    // Handle background messages
    messaging().setBackgroundMessageHandler(async remoteMessage => {
      console.log('Background message:', remoteMessage);
    });

    // Handle foreground messages
    const unsubscribe = messaging().onMessage(async remoteMessage => {
      console.log('Foreground message:', remoteMessage);
      // Show local notification or update UI
    });

    return unsubscribe;
  }
};
```

### 6.2 Backend Notification Service

```typescript
// api/notifications/send.ts
import admin from 'firebase-admin';
import { NextApiRequest, NextApiResponse } from 'next';

export default async function handler(req: NextApiRequest, res: NextApiResponse) {
  if (req.method !== 'POST') {
    return res.status(405).json({ error: 'Method not allowed' });
  }

  try {
    const { recipientId, notification, platform } = req.body;

    // Get user's FCM tokens
    const userDoc = await admin.firestore().collection('users').doc(recipientId).get();
    const userData = userDoc.data();
    
    if (!userData?.fcmTokens) {
      return res.status(404).json({ error: 'User not found or no tokens' });
    }

    const tokens = userData.fcmTokens.filter((token: any) => 
      platform ? token.platform === platform : true
    );

    if (tokens.length === 0) {
      return res.status(404).json({ error: 'No valid tokens found' });
    }

    // Send notification
    const message = {
      notification: {
        title: notification.title,
        body: notification.body,
      },
      data: notification.data || {},
      tokens: tokens.map((token: any) => token.value)
    };

    const response = await admin.messaging().sendMulticast(message);
    
    res.status(200).json({
      successCount: response.successCount,
      failureCount: response.failureCount
    });
  } catch (error) {
    console.error('Notification send error:', error);
    res.status(500).json({ error: 'Failed to send notification' });
  }
}
```

## 7. Analytics Integration

### 7.1 Firebase Analytics

```typescript
// src/services/analytics.ts
import analytics from '@react-native-firebase/analytics';

export const AnalyticsService = {
  async trackEvent(eventName: string, parameters?: any) {
    try {
      await analytics().logEvent(eventName, {
        ...parameters,
        timestamp: new Date().toISOString(),
        platform: Platform.OS
      });
    } catch (error) {
      console.error('Analytics tracking error:', error);
    }
  },

  async trackScreenView(screenName: string, screenClass?: string) {
    try {
      await analytics().logScreenView({
        screen_name: screenName,
        screen_class: screenClass || screenName,
        timestamp: new Date().toISOString()
      });
    } catch (error) {
      console.error('Screen tracking error:', error);
    }
  },

  async setUserProperties(properties: any) {
    try {
      await analytics().setUserProperties(properties);
    } catch (error) {
      console.error('User properties error:', error);
    }
  },

  async trackPurchase(transactionId: string, value: number, currency: string = 'KWD') {
    try {
      await analytics().logPurchase({
        transaction_id: transactionId,
        value: value,
        currency: currency,
        timestamp: new Date().toISOString()
      });
    } catch (error) {
      console.error('Purchase tracking error:', error);
    }
  }
};
```

### 7.2 Custom Analytics Events

```typescript
// src/utils/analyticsEvents.ts
export const AnalyticsEvents = {
  // User events
  USER_SIGNED_UP: 'user_signed_up',
  USER_LOGGED_IN: 'user_logged_in',
  USER_VERIFIED: 'user_verified',
  
  // Equipment events
  EQUIPMENT_VIEWED: 'equipment_viewed',
  EQUIPMENT_SEARCHED: 'equipment_searched',
  EQUIPMENT_SAVED: 'equipment_saved',
  EQUIPMENT_SHARED: 'equipment_shared',
  
  // Order events
  ORDER_CREATED: 'order_created',
  ORDER_CANCELLED: 'order_cancelled',
  ORDER_COMPLETED: 'order_completed',
  
  // Payment events
  PAYMENT_INITIATED: 'payment_initiated',
  PAYMENT_COMPLETED: 'payment_completed',
  PAYMENT_FAILED: 'payment_failed',
  
  // AI events
  AI_ASSISTANT_USED: 'ai_assistant_used',
  AI_RECOMMENDATION_ACCEPTED: 'ai_recommendation_accepted',
  
  // Error events
  ERROR_OCCURRED: 'error_occurred',
  API_ERROR: 'api_error'
};

export const trackUserAction = (action: string, properties?: any) => {
  AnalyticsService.trackEvent(action, {
    ...properties,
    userId: getCurrentUserId(),
    sessionId: getSessionId()
  });
};
```

## 8. Error Tracking

### 8.1 Sentry Integration

```typescript
// src/config/sentry.ts
import * as Sentry from '@sentry/react-native';

Sentry.init({
  dsn: process.env.EXPO_PUBLIC_SENTRY_DSN,
  environment: process.env.NODE_ENV,
  tracesSampleRate: 0.1,
  beforeSend(event) {
    // Filter out sensitive data
    if (event.user) {
      delete event.user.email;
      delete event.user.phone;
    }
    return event;
  }
});

export const SentryService = {
  captureException(error: Error, context?: any) {
    Sentry.captureException(error, {
      tags: {
        platform: Platform.OS,
        version: getAppVersion()
      },
      extra: context
    });
  },

  captureMessage(message: string, level: 'info' | 'warning' | 'error' = 'info') {
    Sentry.captureMessage(message, level);
  },

  setUser(user: { id: string; email?: string }) {
    Sentry.setUser(user);
  }
};
```

## 9. API Rate Limiting

### 9.1 Rate Limiting Middleware

```typescript
// src/middleware/rateLimiting.ts
import { NextApiRequest, NextApiResponse } from 'next';
import { RateLimiter } from 'limiter';

const limiter = new RateLimiter({
  tokensPerInterval: 100,
  interval: 'minute'
});

export const rateLimit = async (req: NextApiRequest, res: NextApiResponse) => {
  try {
    const remainingRequests = await limiter.removeTokens(1);
    
    if (remainingRequests < 0) {
      return res.status(429).json({
        error: 'Too many requests',
        retryAfter: Math.ceil(-remainingRequests / limiter.tokensPerInterval * 60)
      });
    }
    
    res.setHeader('X-RateLimit-Remaining', Math.floor(remainingRequests));
    res.setHeader('X-RateLimit-Reset', new Date(Date.now() + 60000).toISOString());
    
    return true;
  } catch (error) {
    console.error('Rate limiting error:', error);
    return true; // Allow request if rate limiting fails
  }
};
```

## 10. Security Headers

### 10.1 Security Configuration

```typescript
// src/middleware/security.ts
import { NextApiRequest, NextApiResponse } from 'next';

export const securityHeaders = (req: NextApiRequest, res: NextApiResponse) => {
  // CORS
  res.setHeader('Access-Control-Allow-Origin', process.env.ALLOWED_ORIGINS || '*');
  res.setHeader('Access-Control-Allow-Methods', 'GET, POST, PUT, DELETE, OPTIONS');
  res.setHeader('Access-Control-Allow-Headers', 'Content-Type, Authorization');
  
  // Security headers
  res.setHeader('X-Content-Type-Options', 'nosniff');
  res.setHeader('X-Frame-Options', 'DENY');
  res.setHeader('X-XSS-Protection', '1; mode=block');
  res.setHeader('Strict-Transport-Security', 'max-age=31536000; includeSubDomains');
  res.setHeader('Referrer-Policy', 'strict-origin-when-cross-origin');
  
  // CSP
  res.setHeader('Content-Security-Policy', 
    "default-src 'self'; " +
    "script-src 'self' 'unsafe-inline' 'unsafe-eval' https://maps.googleapis.com; " +
    "style-src 'self' 'unsafe-inline' https://fonts.googleapis.com; " +
    "font-src 'self' https://fonts.gstatic.com; " +
    "img-src 'self' data: https:; " +
    "connect-src 'self' https://api.myfatoorah.com https://maps.googleapis.com;"
  );
};
```

## 5. FCM Push Notifications Integration

### 5.1 Expo Notifications Setup

#### Installation

```bash
# Install required packages
npx expo install expo-notifications expo-device expo-constants
npm install @react-native-async-storage/async-storage
```

#### Configuration

```typescript
// app.config.js
export default {
  expo: {
    name: "YARDR",
    slug: "yardr",
    version: "1.0.0",
    orientation: "portrait",
    icon: "./assets/icon.png",
    userInterfaceStyle: "light",
    splash: {
      image: "./assets/splash.png",
      resizeMode: "contain",
      backgroundColor: "#ffffff"
    },
    assetBundlePatterns: [
      "**/*"
    ],
    ios: {
      supportsTablet: true,
      bundleIdentifier: "com.yardr.app"
    },
    android: {
      adaptiveIcon: {
        foregroundImage: "./assets/adaptive-icon.png",
        backgroundColor: "#FFFFFF"
      },
      package: "com.yardr.app",
      googleServicesFile: "./google-services.json"
    },
    web: {
      favicon: "./assets/favicon.png"
    },
    plugins: [
      "expo-location",
      "expo-camera",
      [
        "expo-notifications",
        {
          icon: "./assets/notification-icon.png",
          color: "#ffffff",
          sounds: ["./assets/notification.wav"]
        }
      ]
    ],
    notification: {
      icon: "./assets/notification-icon.png",
      color: "#ffffff"
    }
  }
};
```

### 5.2 FCM Service Implementation

```typescript
// src/services/notifications.ts
import * as Notifications from 'expo-notifications';
import * as Device from 'expo-device';
import Constants from 'expo-constants';
import { Platform } from 'react-native';
import { collection, doc, updateDoc, arrayUnion, arrayRemove } from 'firebase/firestore';
import { db } from '../config/firebase';

// Configure notification behavior
Notifications.setNotificationHandler({
  handleNotification: async () => ({
    shouldShowAlert: true,
    shouldPlaySound: true,
    shouldSetBadge: true,
  }),
});

export class NotificationService {
  private static instance: NotificationService;
  private expoPushToken: string | null = null;

  static getInstance(): NotificationService {
    if (!NotificationService.instance) {
      NotificationService.instance = new NotificationService();
    }
    return NotificationService.instance;
  }

  async registerForPushNotifications(): Promise<string | null> {
    if (!Device.isDevice) {
      console.log('Must use physical device for Push Notifications');
      return null;
    }

    const { status: existingStatus } = await Notifications.getPermissionsAsync();
    let finalStatus = existingStatus;

    if (existingStatus !== 'granted') {
      const { status } = await Notifications.requestPermissionsAsync();
      finalStatus = status;
    }

    if (finalStatus !== 'granted') {
      console.log('Failed to get push token for push notification!');
      return null;
    }

    try {
      const token = (await Notifications.getExpoPushTokenAsync({
        projectId: Constants.expoConfig?.extra?.eas?.projectId,
      })).data;
      
      this.expoPushToken = token;
      await this.saveTokenToFirestore(token);
      
      return token;
    } catch (error) {
      console.error('Error getting push token:', error);
      return null;
    }
  }

  private async saveTokenToFirestore(token: string) {
    try {
      // Get current user ID (you'll need to implement this)
      const userId = await this.getCurrentUserId();
      
      if (userId) {
        const userRef = doc(db, 'users', userId);
        await updateDoc(userRef, {
          expoPushTokens: arrayUnion(token)
        });
      }
    } catch (error) {
      console.error('Error saving token to Firestore:', error);
    }
  }

  async removeTokenFromFirestore(token: string) {
    try {
      const userId = await this.getCurrentUserId();
      
      if (userId) {
        const userRef = doc(db, 'users', userId);
        await updateDoc(userRef, {
          expoPushTokens: arrayRemove(token)
        });
      }
    } catch (error) {
      console.error('Error removing token from Firestore:', error);
    }
  }

  async schedulePushNotification(title: string, body: string, data?: any) {
    await Notifications.scheduleNotificationAsync({
      content: {
        title,
        body,
        data,
        sound: 'default',
      },
      trigger: null, // Send immediately
    });
  }

  async getNotificationHistory() {
    return await Notifications.getPresentedNotificationsAsync();
  }

  async clearAllNotifications() {
    await Notifications.dismissAllNotificationsAsync();
  }

  private async getCurrentUserId(): Promise<string | null> {
    // Implement based on your auth system
    // This should return the current user's ID
    return null;
  }
}

// Hook for using notifications
export const useNotifications = () => {
  const notificationService = NotificationService.getInstance();

  const registerForPushNotifications = async () => {
    return await notificationService.registerForPushNotifications();
  };

  const scheduleNotification = async (title: string, body: string, data?: any) => {
    return await notificationService.schedulePushNotification(title, body, data);
  };

  return {
    registerForPushNotifications,
    scheduleNotification,
  };
};
```

### 5.3 Notification Channels (Android)

```typescript
// src/utils/notificationChannels.ts
import * as Notifications from 'expo-notifications';

export const setupNotificationChannels = async () => {
  if (Platform.OS === 'android') {
    await Notifications.setNotificationChannelAsync('orders', {
      name: 'Order Notifications',
      description: 'Notifications about your equipment orders',
      importance: Notifications.AndroidImportance.HIGH,
      vibrationPattern: [0, 250, 250, 250],
      lightColor: '#FF231F7C',
    });

    await Notifications.setNotificationChannelAsync('payments', {
      name: 'Payment Notifications',
      description: 'Notifications about payments and transactions',
      importance: Notifications.AndroidImportance.HIGH,
      vibrationPattern: [0, 250, 250, 250],
      lightColor: '#FF231F7C',
    });

    await Notifications.setNotificationChannelAsync('general', {
      name: 'General Notifications',
      description: 'General app notifications',
      importance: Notifications.AndroidImportance.DEFAULT,
    });
  }
};
```

### 5.4 Background Notification Handling

```typescript
// App.tsx
import { useEffect, useRef } from 'react';
import * as Notifications from 'expo-notifications';
import { useNotifications } from './src/services/notifications';

export default function App() {
  const notificationListener = useRef<Notifications.Subscription>();
  const responseListener = useRef<Notifications.Subscription>();
  const { registerForPushNotifications } = useNotifications();

  useEffect(() => {
    // Register for push notifications
    registerForPushNotifications();

    // Listen for notifications received while app is running
    notificationListener.current = Notifications.addNotificationReceivedListener(notification => {
      console.log('Notification received:', notification);
      // Handle in-app notification display
    });

    // Listen for user interactions with notifications
    responseListener.current = Notifications.addNotificationResponseReceivedListener(response => {
      console.log('Notification response:', response);
      // Handle deep linking based on notification data
      const data = response.notification.request.content.data;
      if (data?.screen) {
        // Navigate to specific screen
        // navigation.navigate(data.screen, data.params);
      }
    });

    return () => {
      if (notificationListener.current) {
        Notifications.removeNotificationSubscription(notificationListener.current);
      }
      if (responseListener.current) {
        Notifications.removeNotificationSubscription(responseListener.current);
      }
    };
  }, []);

  // ... rest of your app
}
```

This comprehensive integration specification covers all the major third-party services and APIs that the YARDR platform will integrate with, providing detailed implementation examples and configuration for each service.
