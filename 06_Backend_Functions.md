# YARDR - Cloud Functions Specifications

## Authentication Functions

### User Creation Handler

```typescript
// auth/onCreate.ts
export const onUserCreate = functions.auth.user().onCreate(async (user) => {
  // 1. Create user profile in Firestore
  const userProfile = {
    uid: user.uid,
    email: user.email,
    phone: user.phoneNumber,
    phoneVerified: false,
    role: 'renter', // default role
    verified: false,
    verificationLevel: 'basic',
    wallet: {
      balance: 0,
      currency: 'KWD',
      holdAmount: 0,
      lastUpdated: admin.firestore.FieldValue.serverTimestamp()
    },
    profile: {
      language: 'en',
      notificationPreferences: {
        push: true,
        email: true,
        sms: true
      }
    },
    createdAt: admin.firestore.FieldValue.serverTimestamp(),
    updatedAt: admin.firestore.FieldValue.serverTimestamp()
  };
  
  await admin.firestore().collection('users').doc(user.uid).set(userProfile);
  
  // 2. Send welcome notification
  await sendWelcomeEmail(user.email);
  
  // 3. Log analytics event
  await logAnalyticsEvent('user_signup', { userId: user.uid });
  
  return { success: true };
});
```

### Custom Claims Management

```typescript
// auth/customClaims.ts
export const setUserRole = functions.https.onCall(async (data, context) => {
  // Verify admin
  if (!context.auth || !context.auth.token.admin) {
    throw new functions.https.HttpsError('permission-denied', 'Must be admin');
  }
  
  const { userId, role } = data;
  
  // Set custom claims
  await admin.auth().setCustomUserClaims(userId, { role, verified: true });
  
  // Update Firestore
  await admin.firestore().collection('users').doc(userId).update({
    role,
    verified: true,
    updatedAt: admin.firestore.FieldValue.serverTimestamp()
  });
  
  return { success: true };
});
```

## Order Management Functions

### Create Order

```typescript
// orders/createOrder.ts
export const createOrder = functions.https.onCall(async (data, context) => {
  // 1. Validate authentication
  if (!context.auth) {
    throw new functions.https.HttpsError('unauthenticated', 'User must be authenticated');
  }
  
  const { equipmentId, startDate, endDate, location, withOperator, needsDelivery } = data;
  
  // 2. Validate equipment availability
  const equipmentDoc = await admin.firestore()
    .collection('equipment')
    .doc(equipmentId)
    .get();
    
  if (!equipmentDoc.exists) {
    throw new functions.https.HttpsError('not-found', 'Equipment not found');
  }
  
  const equipment = equipmentDoc.data();
  
  // 3. Check availability for dates
  const isAvailable = await checkEquipmentAvailability(equipmentId, startDate, endDate);
  if (!isAvailable) {
    throw new functions.https.HttpsError('unavailable', 'Equipment not available for selected dates');
  }
  
  // 4. Calculate pricing
  const duration = calculateDays(startDate, endDate);
  const equipmentCost = equipment.pricing.daily * duration;
  const operatorCost = withOperator ? (equipment.pricing.operator || 0) * duration : 0;
  const deliveryCost = needsDelivery ? (equipment.pricing.mobilization || 0) : 0;
  const subtotal = equipmentCost + operatorCost + deliveryCost;
  const tax = subtotal * 0.05; // 5% tax
  const total = subtotal + tax;
  
  // 5. Create order
  const orderData = {
    orderNumber: generateOrderNumber(),
    parties: {
      renterId: context.auth.uid,
      ownerId: equipment.ownerId,
      companyId: equipment.companyId
    },
    equipment: {
      id: equipmentId,
      name: equipment.basic.name,
      type: equipment.basic.type,
      dailyRate: equipment.pricing.daily
    },
    rental: {
      startDate,
      endDate,
      duration,
      location,
      withOperator,
      needsDelivery
    },
    status: {
      current: 'pending',
      history: [{
        status: 'pending',
        timestamp: admin.firestore.FieldValue.serverTimestamp(),
        by: context.auth.uid,
        note: 'Order created'
      }]
    },
    pricing: {
      equipmentCost,
      operatorCost,
      deliveryCost,
      subtotal,
      tax,
      total,
      currency: 'KWD',
      paid: false
    },
    payment: {
      status: 'pending'
    },
    aiAssisted: data.aiAssisted || false,
    createdAt: admin.firestore.FieldValue.serverTimestamp(),
    updatedAt: admin.firestore.FieldValue.serverTimestamp()
  };
  
  const orderRef = await admin.firestore().collection('orders').add(orderData);
  
  // 6. Send notifications
  await sendOrderNotification(equipment.ownerId, 'new_order', {
    orderId: orderRef.id,
    equipmentName: equipment.basic.name,
    dates: `${startDate} - ${endDate}`,
    total
  });
  
  // 7. Update equipment calendar
  await updateEquipmentCalendar(equipmentId, startDate, endDate, 'tentative');
  
  return { 
    success: true, 
    orderId: orderRef.id,
    total
  };
});
```

### Equipment Matching

```typescript
// orders/matchEquipment.ts
export const matchEquipmentToOrder = functions.firestore
  .document('orders/{orderId}')
  .onCreate(async (snapshot, context) => {
    const order = snapshot.data();
    
    // Find alternative equipment if primary is rejected
    if (order.status.current === 'rejected') {
      const alternatives = await findAlternativeEquipment(
        order.equipment.type,
        order.rental.location,
        order.rental.startDate,
        order.rental.endDate
      );
      
      if (alternatives.length > 0) {
        // Notify renter of alternatives
        await sendAlternativeOptionsNotification(order.parties.renterId, alternatives);
      }
    }
  });
```

### Update Order Status

```typescript
// orders/updateStatus.ts
export const updateOrderStatus = functions.https.onCall(async (data, context) => {
  const { orderId, newStatus, note } = data;
  
  // Get order
  const orderRef = admin.firestore().collection('orders').doc(orderId);
  const orderDoc = await orderRef.get();
  
  if (!orderDoc.exists) {
    throw new functions.https.HttpsError('not-found', 'Order not found');
  }
  
  const order = orderDoc.data();
  
  // Validate status transition
  const validTransitions = {
    'pending': ['accepted', 'rejected'],
    'accepted': ['paid', 'cancelled'],
    'paid': ['active', 'cancelled'],
    'active': ['in_progress', 'cancelled'],
    'in_progress': ['completed', 'disputed'],
    'completed': ['disputed']
  };
  
  if (!validTransitions[order.status.current]?.includes(newStatus)) {
    throw new functions.https.HttpsError('invalid-argument', 'Invalid status transition');
  }
  
  // Update status
  await orderRef.update({
    'status.current': newStatus,
    'status.history': admin.firestore.FieldValue.arrayUnion({
      status: newStatus,
      timestamp: admin.firestore.FieldValue.serverTimestamp(),
      by: context.auth.uid,
      note
    }),
    updatedAt: admin.firestore.FieldValue.serverTimestamp()
  });
  
  // Handle status-specific actions
  switch (newStatus) {
    case 'accepted':
      await handleOrderAccepted(orderId, order);
      break;
    case 'paid':
      await handleOrderPaid(orderId, order);
      break;
    case 'active':
      await handleOrderActive(orderId, order);
      break;
    case 'completed':
      await handleOrderCompleted(orderId, order);
      break;
    case 'cancelled':
      await handleOrderCancelled(orderId, order);
      break;
    case 'disputed':
      await handleOrderDisputed(orderId, order);
      break;
  }
  
  return { success: true };
});
```

## Payment Functions

### Process Payment

```typescript
// payments/processPayment.ts
export const processPayment = functions.https.onCall(async (data, context) => {
  const { orderId, paymentMethod, paymentDetails } = data;
  
  // Get order
  const orderDoc = await admin.firestore()
    .collection('orders')
    .doc(orderId)
    .get();
    
  const order = orderDoc.data();
  
  // Get user wallet
  const userDoc = await admin.firestore()
    .collection('users')
    .doc(context.auth.uid)
    .get();
    
  const user = userDoc.data();
  
  let transactionId: string;
  
  // Process payment based on method
  switch (paymentMethod) {
    case 'wallet':
      // Check wallet balance
      if (user.wallet.balance < order.pricing.total) {
        throw new functions.https.HttpsError('failed-precondition', 'Insufficient wallet balance');
      }
      
      // Deduct from wallet
      transactionId = await processWalletPayment(context.auth.uid, order.pricing.total, orderId);
      break;
      
    case 'myfatoorah':
      // Process MyFatoorah payment
      transactionId = await processMyFatoorahPayment(paymentDetails, order.pricing.total);
      break;
  }
  
  // Update order payment status
  await admin.firestore().collection('orders').doc(orderId).update({
    'payment.status': 'paid',
    'payment.transactionId': transactionId,
    'payment.paidAt': admin.firestore.FieldValue.serverTimestamp(),
    'status.current': 'paid'
  });
  
  // Generate invoice
  const invoiceUrl = await generateInvoice(orderId);
  
  // Send notifications
  await sendPaymentConfirmation(context.auth.uid, orderId, invoiceUrl);
  
  return { 
    success: true, 
    transactionId,
    invoiceUrl 
  };
});
```

### Wallet Management

```typescript
// payments/updateWallet.ts
export const addFundsToWallet = functions.https.onCall(async (data, context) => {
  const { amount, paymentMethod, paymentDetails } = data;
  
  let transactionId: string;
  
  // Process payment
  switch (paymentMethod) {
    case 'myfatoorah':
      transactionId = await processMyFatoorahPayment(paymentDetails, amount);
      break;
  }
  
  // Update wallet
  await admin.firestore().collection('users').doc(context.auth.uid).update({
    'wallet.balance': admin.firestore.FieldValue.increment(amount),
    'wallet.lastUpdated': admin.firestore.FieldValue.serverTimestamp()
  });
  
  // Create transaction record
  await admin.firestore().collection('transactions').add({
    type: 'credit',
    userId: context.auth.uid,
    amount,
    currency: 'KWD',
    details: {
      description: 'Wallet top-up',
      method: paymentMethod,
      reference: transactionId
    },
    status: 'completed',
    createdAt: admin.firestore.FieldValue.serverTimestamp()
  });
  
  return { success: true, transactionId };
});
```

## AI Assistant Functions

### Analyze Equipment Request

```typescript
// ai/analyzeRequest.ts
export const analyzeEquipmentRequest = functions.https.onCall(async (data, context) => {
  const { problemDescription, language = 'en' } = data;
  
  // 1. Process natural language input
  const nlpResult = await processNaturalLanguage(problemDescription, language);
  
  // 2. Extract requirements
  const requirements = {
    taskType: nlpResult.taskType, // digging, lifting, moving, etc.
    specifications: {
      depth: nlpResult.entities.depth,
      weight: nlpResult.entities.weight,
      distance: nlpResult.entities.distance,
      area: nlpResult.entities.area,
      height: nlpResult.entities.height
    },
    constraints: {
      space: nlpResult.entities.spaceConstraints,
      time: nlpResult.entities.timeConstraints,
      terrain: nlpResult.entities.terrain
    }
  };
  
  // 3. Match requirements to equipment
  const matches = await findMatchingEquipment(requirements);
  
  // 4. Rank by suitability
  const rankedMatches = rankEquipmentMatches(matches, requirements);
  
  // 5. Generate explanations
  const recommendations = rankedMatches.slice(0, 5).map(match => ({
    equipmentId: match.id,
    name: match.basic.name,
    type: match.basic.type,
    suitabilityScore: match.score,
    explanation: generateExplanation(match, requirements),
    specifications: match.specs,
    pricing: match.pricing,
    availability: match.availability.status
  }));
  
  // 6. Store analysis for learning
  await storeAIAnalysis(problemDescription, requirements, recommendations);
  
  return {
    success: true,
    requirements,
    recommendations,
    confidence: calculateConfidence(recommendations)
  };
});
```

### Natural Language Processing

```typescript
// ai/processNLP.ts
async function processNaturalLanguage(text: string, language: string) {
  // Use Google Cloud Natural Language API
  const client = new language.LanguageServiceClient();
  
  const document = {
    content: text,
    type: 'PLAIN_TEXT',
    language
  };
  
  // Analyze entities
  const [entities] = await client.analyzeEntities({ document });
  
  // Analyze syntax
  const [syntax] = await client.analyzeSyntax({ document });
  
  // Extract task type from verbs
  const taskType = extractTaskType(syntax.tokens);
  
  // Extract measurements and specifications
  const specifications = extractSpecifications(entities.entities);
  
  return {
    taskType,
    entities: specifications,
    confidence: entities.entities[0]?.salience || 0
  };
}
```

### Equipment Recommendation

```typescript
// ai/recommendEquipment.ts
function generateExplanation(equipment: any, requirements: any): string {
  const explanations = [];
  
  // Match task type
  if (requirements.taskType === 'digging' && equipment.basic.type === 'excavator') {
    explanations.push(`This excavator is ideal for digging tasks`);
  }
  
  // Match specifications
  if (requirements.specifications.depth && equipment.specs.maxDepth >= requirements.specifications.depth) {
    explanations.push(`Can dig up to ${equipment.specs.maxDepth}m deep, meeting your ${requirements.specifications.depth}m requirement`);
  }
  
  if (requirements.specifications.weight && equipment.specs.capacity >= requirements.specifications.weight) {
    explanations.push(`Has ${equipment.specs.capacity} ton capacity, sufficient for your ${requirements.specifications.weight} ton load`);
  }
  
  // Add more explanation logic...
  
  return explanations.join('. ');
}
```

## Tracking Functions

### Update Driver Location

```typescript
// tracking/updateLocation.ts
export const updateDriverLocation = functions.https.onCall(async (data, context) => {
  const { orderId, location, timestamp } = data;
  
  // Verify driver
  const orderDoc = await admin.firestore()
    .collection('orders')
    .doc(orderId)
    .get();
    
  if (orderDoc.data().parties.driverId !== context.auth.uid) {
    throw new functions.https.HttpsError('permission-denied', 'Not authorized driver');
  }
  
  // Update tracking subcollection
  await admin.firestore()
    .collection('orders')
    .doc(orderId)
    .collection('tracking')
    .add({
      location: new admin.firestore.GeoPoint(location.latitude, location.longitude),
      timestamp,
      speed: data.speed,
      heading: data.heading
    });
  
  // Update current location in order
  await admin.firestore()
    .collection('orders')
    .doc(orderId)
    .update({
      'tracking.currentLocation': new admin.firestore.GeoPoint(location.latitude, location.longitude),
      'tracking.lastUpdate': timestamp
    });
  
  return { success: true };
});
```

### Generate Trip Report

```typescript
// tracking/generateReport.ts
export const generateTripReport = functions.https.onCall(async (data, context) => {
  const { orderId } = data;
  
  // Get tracking data
  const trackingSnapshot = await admin.firestore()
    .collection('orders')
    .doc(orderId)
    .collection('tracking')
    .orderBy('timestamp')
    .get();
  
  const trackingPoints = trackingSnapshot.docs.map(doc => doc.data());
  
  // Calculate metrics
  const totalDistance = calculateTotalDistance(trackingPoints);
  const totalTime = calculateTotalTime(trackingPoints);
  const averageSpeed = totalDistance / totalTime;
  
  // Generate report
  const report = {
    orderId,
    startLocation: trackingPoints[0]?.location,
    endLocation: trackingPoints[trackingPoints.length - 1]?.location,
    totalDistance,
    totalTime,
    averageSpeed,
    route: trackingPoints.map(p => p.location),
    generatedAt: admin.firestore.FieldValue.serverTimestamp()
  };
  
  // Store report
  await admin.firestore()
    .collection('orders')
    .doc(orderId)
    .update({
      'tracking.report': report
    });
  
  return report;
});
```

## Admin Functions

### Approve User Verification

```typescript
// admin/approveUser.ts
export const approveUserVerification = functions.https.onCall(async (data, context) => {
  // Check admin role
  if (!context.auth.token.admin) {
    throw new functions.https.HttpsError('permission-denied', 'Admin only');
  }
  
  const { userId, verificationType, approved, reason } = data;
  
  if (approved) {
    // Update user verification
    await admin.firestore().collection('users').doc(userId).update({
      verified: true,
      verificationLevel: verificationType,
      'verification.approvedAt': admin.firestore.FieldValue.serverTimestamp(),
      'verification.approvedBy': context.auth.uid
    });
    
    // Set custom claims
    await admin.auth().setCustomUserClaims(userId, { 
      verified: true,
      verificationLevel: verificationType
    });
    
    // Send approval notification
    await sendNotification(userId, 'verification_approved', {
      title: 'Verification Approved',
      body: 'Your account has been verified. You can now access all features.'
    });
  } else {
    // Reject with reason
    await admin.firestore().collection('users').doc(userId).update({
      'verification.rejected': true,
      'verification.rejectionReason': reason,
      'verification.rejectedAt': admin.firestore.FieldValue.serverTimestamp()
    });
    
    // Send rejection notification
    await sendNotification(userId, 'verification_rejected', {
      title: 'Verification Failed',
      body: `Your verification was rejected: ${reason}`
    });
  }
  
  return { success: true };
});
```

### Generate Daily Analytics

```typescript
// admin/generateAnalytics.ts
export const generateDailyAnalytics = functions.pubsub
  .schedule('every day 00:00')
  .timeZone('Asia/Kuwait')
  .onRun(async (context) => {
    const yesterday = new Date();
    yesterday.setDate(yesterday.getDate() - 1);
    const dateString = yesterday.toISOString().split('T')[0];
    
    // Aggregate metrics
    const metrics = await aggregateDailyMetrics(dateString);
    
    // Store analytics
    await admin.firestore()
      .collection('analytics')
      .doc(dateString)
      .set({
        date: dateString,
        platform: metrics.platform,
        users: metrics.users,
        equipment: metrics.equipment,
        geographic: metrics.geographic,
        ai: metrics.ai,
        createdAt: admin.firestore.FieldValue.serverTimestamp()
      });
    
    // Send daily report to admins
    await sendDailyReportToAdmins(metrics);
    
    return null;
  });
```

### Resolve Dispute

```typescript
// admin/resolveDispute.ts
export const resolveDispute = functions.https.onCall(async (data, context) => {
  if (!context.auth.token.admin) {
    throw new functions.https.HttpsError('permission-denied', 'Admin only');
  }
  
  const { orderId, resolution, refundAmount } = data;
  
  // Update dispute status
  await admin.firestore().collection('orders').doc(orderId).update({
    'dispute.status': 'resolved',
    'dispute.resolution': resolution,
    'dispute.resolvedBy': context.auth.uid,
    'dispute.resolvedAt': admin.firestore.FieldValue.serverTimestamp()
  });
  
  // Process refund if applicable
  if (refundAmount > 0) {
    await processRefund(orderId, refundAmount);
  }
  
  // Notify parties
  const order = await admin.firestore().collection('orders').doc(orderId).get();
  const orderData = order.data();
  
  await sendNotification(orderData.parties.renterId, 'dispute_resolved', {
    title: 'Dispute Resolved',
    body: `Your dispute has been resolved: ${resolution}`
  });
  
  await sendNotification(orderData.parties.ownerId, 'dispute_resolved', {
    title: 'Dispute Resolved',
    body: `The dispute has been resolved: ${resolution}`
  });
  
  return { success: true };
});
```

## Notification Functions

### Send Push Notification (FCM with Expo)

```typescript
// notifications/sendPushNotification.ts
import * as functions from 'firebase-functions';
import * as admin from 'firebase-admin';

export const sendPushNotification = functions.https.onCall(async (data, context) => {
  const { userId, title, body, data: notificationData, channelId } = data;
  
  // Get user's Expo push tokens
  const userDoc = await admin.firestore().collection('users').doc(userId).get();
  const user = userDoc.data();
  
  if (!user.expoPushTokens || user.expoPushTokens.length === 0) {
    throw new functions.https.HttpsError('failed-precondition', 'No Expo push tokens found');
  }
  
  // Create FCM message for Expo
  const messages = user.expoPushTokens.map((token: string) => ({
    token,
    notification: {
      title,
      body
    },
    data: {
      ...notificationData,
      channelId: channelId || 'default',
      sound: 'default',
      priority: 'high'
    },
    android: {
      notification: {
        channelId: channelId || 'default',
        sound: 'default',
        priority: 'high'
      }
    },
    apns: {
      payload: {
        aps: {
          sound: 'default',
          badge: 1
        }
      }
    }
  }));
  
  // Send notifications using FCM
  const response = await admin.messaging().sendAll(messages);
  
  // Handle failed tokens
  if (response.failureCount > 0) {
    const failedTokens = [];
    response.responses.forEach((resp, idx) => {
      if (!resp.success) {
        failedTokens.push(user.expoPushTokens[idx]);
      }
    });
    
    // Remove failed tokens
    await admin.firestore().collection('users').doc(userId).update({
      expoPushTokens: admin.firestore.FieldValue.arrayRemove(...failedTokens)
    });
  }
  
  return { 
    success: true, 
    successCount: response.successCount,
    failureCount: response.failureCount
  };
});
```

### Send Email Notification

```typescript
// notifications/sendEmail.ts
export const sendEmailNotification = functions.https.onCall(async (data, context) => {
  const { to, subject, template, variables } = data;
  
  // Get email template
  const templateDoc = await admin.firestore()
    .collection('emailTemplates')
    .doc(template)
    .get();
    
  if (!templateDoc.exists) {
    throw new functions.https.HttpsError('not-found', 'Email template not found');
  }
  
  const templateData = templateDoc.data();
  
  // Replace variables in template
  let htmlContent = templateData.html;
  let textContent = templateData.text;
  
  Object.keys(variables).forEach(key => {
    const regex = new RegExp(`{{${key}}}`, 'g');
    htmlContent = htmlContent.replace(regex, variables[key]);
    textContent = textContent.replace(regex, variables[key]);
  });
  
  // Send email using SendGrid
  const msg = {
    to,
    from: 'noreply@yardr.com',
    subject,
    text: textContent,
    html: htmlContent
  };
  
  await sgMail.send(msg);
  
  return { success: true };
});
```

## Utility Functions

### Helper Functions

```typescript
// utils/helpers.ts
export const generateOrderNumber = (): string => {
  const timestamp = Date.now().toString(36);
  const random = Math.random().toString(36).substr(2, 5);
  return `YARDR-${timestamp}-${random}`.toUpperCase();
};

export const calculateDays = (startDate: Timestamp, endDate: Timestamp): number => {
  const start = startDate.toDate();
  const end = endDate.toDate();
  const diffTime = Math.abs(end.getTime() - start.getTime());
  return Math.ceil(diffTime / (1000 * 60 * 60 * 24));
};

export const checkEquipmentAvailability = async (
  equipmentId: string, 
  startDate: Timestamp, 
  endDate: Timestamp
): Promise<boolean> => {
  const equipmentDoc = await admin.firestore()
    .collection('equipment')
    .doc(equipmentId)
    .get();
    
  const equipment = equipmentDoc.data();
  
  // Check if equipment is available
  if (equipment.availability.status !== 'available') {
    return false;
  }
  
  // Check calendar for conflicts
  const start = startDate.toDate();
  const end = endDate.toDate();
  
  for (let d = new Date(start); d <= end; d.setDate(d.getDate() + 1)) {
    const dateKey = d.toISOString().split('T')[0];
    if (equipment.availability.calendar[dateKey] === 'booked') {
      return false;
    }
  }
  
  return true;
};
```

### Error Handling

```typescript
// utils/errorHandler.ts
export const handleFunctionError = (error: any, context: functions.https.CallableContext) => {
  console.error('Function error:', error);
  
  if (error.code === 'permission-denied') {
    return new functions.https.HttpsError('permission-denied', 'Insufficient permissions');
  }
  
  if (error.code === 'not-found') {
    return new functions.https.HttpsError('not-found', 'Resource not found');
  }
  
  if (error.code === 'invalid-argument') {
    return new functions.https.HttpsError('invalid-argument', 'Invalid input provided');
  }
  
  return new functions.https.HttpsError('internal', 'An internal error occurred');
};
```

## Helper Functions

### MyFatoorah Payment Processing

```typescript
// Helper function to process MyFatoorah payment
async function processMyFatoorahPayment(paymentDetails: any, amount: number): Promise<string> {
  const myfatoorahApiKey = functions.config().myfatoorah?.api_key;
  const myfatoorahBaseUrl = functions.config().myfatoorah?.base_url || 'https://apitest.myfatoorah.com';
  
  if (!myfatoorahApiKey) {
    throw new functions.https.HttpsError('failed-precondition', 'MyFatoorah API key not configured');
  }
  
  try {
    // Create payment request
    const paymentRequest = {
      InvoiceAmount: amount,
      CurrencyIso: 'KWD',
      CustomerName: paymentDetails.customerName,
      CustomerEmail: paymentDetails.customerEmail,
      CustomerMobile: paymentDetails.customerPhone,
      UserDefinedField: paymentDetails.orderId,
      CallBackUrl: `${functions.config().app?.base_url}/api/myfatoorah/callback`,
      ErrorUrl: `${functions.config().app?.base_url}/api/myfatoorah/error`
    };
    
    const response = await fetch(`${myfatoorahBaseUrl}/v2/SendPayment`, {
      method: 'POST',
      headers: {
        'Content-Type': 'application/json',
        'Authorization': `Bearer ${myfatoorahApiKey}`
      },
      body: JSON.stringify(paymentRequest)
    });
    
    const data = await response.json();
    
    if (!data.IsSuccess) {
      throw new Error(data.Message || 'Payment initiation failed');
    }
    
    // Create transaction record
    const transactionRef = await admin.firestore().collection('transactions').add({
      orderId: paymentDetails.orderId,
      type: 'payment',
      amount,
      method: 'myfatoorah',
      status: 'pending',
      externalId: data.Data.InvoiceId,
      paymentUrl: data.Data.InvoiceURL,
      timestamp: admin.firestore.FieldValue.serverTimestamp()
    });
    
    return transactionRef.id;
  } catch (error) {
    console.error('MyFatoorah payment error:', error);
    throw new functions.https.HttpsError('internal', 'Payment processing failed');
  }
}
```

## Function Deployment

### Package.json

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

### Deployment Commands

```bash
# Deploy all functions
firebase deploy --only functions

# Deploy specific function
firebase deploy --only functions:createOrder

# Deploy with environment variables
firebase functions:config:set myfatoorah.api_key="your_api_key"
firebase functions:config:set myfatoorah.base_url="https://apitest.myfatoorah.com"
firebase deploy --only functions
```
