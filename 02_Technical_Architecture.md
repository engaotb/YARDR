# YARDR - System Architecture

## High-Level Architecture

```mermaid
graph TB
    subgraph "Client Layer"
        iOS_Renter[iOS Renter App]
        Android_Renter[Android Renter App]
        iOS_Owner[iOS Owner App]
        Android_Owner[Android Owner App]
        iOS_Driver[iOS Driver App]
        Android_Driver[Android Driver App]
        Web_Admin[Web Admin Dashboard]
    end
    
    subgraph "CDN Layer"
        FH[Firebase Hosting<br/>- Global Distribution<br/>- SSL/TLS Security<br/>- Static Caching]
    end
    
    subgraph "Firebase Services"
        Auth[Firebase Auth<br/>- Email/Phone Auth<br/>- Custom Claims<br/>- Session Management]
        
        Firestore[(Firestore Database<br/>- Real-time Sync<br/>- Offline Support<br/>- Auto-scaling)]
        
        Functions[Cloud Functions<br/>- Business Logic<br/>- AI Processing<br/>- Integrations]
        
        Storage[Cloud Storage<br/>- Images<br/>- Documents<br/>- Backups]
        
        FCM[Cloud Messaging<br/>- Push Notifications<br/>- In-App Messages]
        
        Hosting[Firebase Hosting<br/>- Admin Dashboard<br/>- Landing Pages]
    end
    
    subgraph "External Services"
        PACI[PACI API<br/>Kuwait Civil ID]
        MyFatoorah[MyFatoorah Gateway<br/>Kuwait & International Payments]
        Maps[Google Maps<br/>Location Services]
        AI_Service[Firebase GenAI<br/>Gemini Models]
    end
    
    iOS_Renter --> CF
    Android_Renter --> CF
    iOS_Owner --> CF
    Android_Owner --> CF
    iOS_Driver --> CF
    Android_Driver --> CF
    Web_Admin --> CF
    
    CF --> Auth
    CF --> Functions
    
    Auth --> Firestore
    Functions --> Firestore
    Functions --> Storage
    Functions --> FCM
    Functions --> PACI
    Functions --> MyFatoorah
    Functions --> Maps
    Functions --> AI_Service
```

## Data Flow Architecture

```mermaid
sequenceDiagram
    participant User
    participant App
    participant Firebase_Hosting
    participant Firebase_Auth
    participant Cloud_Functions
    participant Firestore
    participant External_API
    
    User->>App: Open Application
    App->>Firebase_Hosting: Request (with auth token)
    Firebase_Hosting->>Firebase_Auth: Validate Token
    Firebase_Auth->>App: User Context
    
    User->>App: Create Order
    App->>Cloud_Functions: ProcessOrder()
    Cloud_Functions->>Firestore: Validate Equipment
    Cloud_Functions->>External_API: Process Payment
    External_API->>Cloud_Functions: Payment Result
    Cloud_Functions->>Firestore: Update Order
    Cloud_Functions->>FCM: Send Notifications
    
    Firestore-->>App: Real-time Update
    FCM-->>User: Push Notification
```

## Architecture Components

### Client Layer

#### Mobile Applications
- **iOS Renter App**: Native iOS app for equipment renters
- **Android Renter App**: Native Android app for equipment renters
- **iOS Owner App**: Native iOS app for equipment owners
- **Android Owner App**: Native Android app for equipment owners
- **iOS Driver App**: Native iOS app for drivers
- **Android Driver App**: Native Android app for drivers

#### Web Application
- **Admin Dashboard**: Web-based admin interface using Next.js and ShadCN

### CDN Layer

#### Firebase Hosting
- **Global Distribution**: Content delivery across multiple regions
- **SSL/TLS Security**: Automatic HTTPS and security headers
- **Static Caching**: Optimized delivery of static assets
- **SSL/TLS**: Secure connections with automatic certificate management

### Firebase Services

#### Authentication
- **Multi-provider Support**: Email/password and phone authentication
- **Custom Claims**: Role-based access control
- **Session Management**: Secure token handling and refresh

#### Firestore Database
- **Real-time Sync**: Live data updates across all clients
- **Offline Support**: Local data persistence and sync
- **Auto-scaling**: Automatic scaling based on demand
- **Security Rules**: Fine-grained access control

#### Cloud Functions
- **Business Logic**: Server-side processing
- **AI Processing**: Natural language processing for equipment recommendations
- **Integrations**: Third-party service integrations
- **Event Triggers**: Automated responses to database changes

#### Cloud Storage
- **File Management**: Images, documents, and backups
- **Access Control**: Secure file access with signed URLs
- **CDN Integration**: Optimized file delivery

#### Cloud Messaging
- **Push Notifications**: Real-time notifications to mobile devices
- **In-App Messages**: Contextual messaging within applications
- **Topic Subscriptions**: Targeted messaging by user roles

#### Firebase Hosting
- **Admin Dashboard**: Hosting for web admin interface
- **Landing Pages**: Marketing and informational pages
- **Custom Domains**: Branded URLs and SSL certificates

### External Services

#### PACI API
- **Kuwait Civil ID Verification**: Government identity verification
- **Data Validation**: Official identity data validation
- **Compliance**: Meeting local regulatory requirements

#### MyFatoorah Gateway
- **Kuwait & International Payments**: Local and international payment processing
- **Multiple Payment Methods**: Cards, bank transfers, digital wallets
- **Mobile Payments**: Apple Pay, Google Pay integration
- **Currency Support**: Kuwaiti Dinar (KWD) and international currencies
- **Multi-currency**: Support for various currencies
- **Payment Methods**: Credit/debit cards and digital wallets

#### Google Maps
- **Location Services**: GPS and mapping functionality
- **Geocoding**: Address to coordinates conversion
- **Directions**: Route planning and navigation
- **Places API**: Location search and details

#### Firebase GenAI Chatbot
- **Gemini Models**: Advanced AI using Google's Gemini models
- **Natural Language Processing**: Understanding user requests with context awareness
- **Equipment Matching**: AI-powered equipment recommendations
- **Multi-language**: English and Arabic language support
- **Firestore Integration**: Seamless conversation storage and retrieval

