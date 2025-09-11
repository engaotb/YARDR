# YARDR - Heavy Equipment Rental Platform

## Overview
YARDR is a comprehensive heavy equipment rental platform serving the Kuwait market. The system connects equipment renters, owners, drivers, and administrators through four dedicated applications, all powered by Firebase backend services.

## Key Features
- 🤖 **AI Equipment Assistant** - Natural language equipment recommendations
- 💳 **MyFatoorah Payment Integration** - Secure payment processing
- 📱 **Multi-Platform Apps** - iOS, Android, and Web applications
- 🗺️ **Real-time Tracking** - GPS tracking and live updates
- 🔔 **Push Notifications** - FCM integration with Expo
- 🌐 **Firebase Backend** - Scalable cloud infrastructure

## Documentation Structure

| File | Description |
|------|-------------|
| `01_Project_Overview.md` | High-level project overview and features |
| `02_Technical_Architecture.md` | System architecture and technical design |
| `03_Firebase_Configuration.md` | Complete Firebase setup and configuration |
| `04_User_Experience_Flows.md` | Comprehensive user journey flows |
| `05_Database_Schema.md` | Firestore database schema and interfaces |
| `06_Backend_Functions.md` | Cloud Functions specifications |
| `07_Design_System.md` | UI/UX design system and guidelines |
| `08_Third_Party_Integrations.md` | External service integrations |
| `YARDR_Complete_System_Documentation.md` | Consolidated documentation |

## Technology Stack

### Frontend
- **Mobile Apps**: Expo (React Native) for iOS/Android
- **Web Dashboard**: Next.js with ShadCN UI
- **State Management**: React Context + Hooks
- **Navigation**: React Navigation

### Backend
- **Database**: Firebase Firestore
- **Authentication**: Firebase Auth
- **Functions**: Firebase Cloud Functions
- **Storage**: Firebase Cloud Storage
- **Hosting**: Firebase Hosting
- **AI**: Firebase GenAI Chatbot Extension

### Integrations
- **Payments**: MyFatoorah Gateway
- **Maps**: Google Maps API
- **Notifications**: Firebase Cloud Messaging (FCM)
- **Verification**: PACI API (Kuwait Civil ID)

## Getting Started

1. **Clone the repository**
   ```bash
   git clone https://github.com/YOUR_USERNAME/YARDR.git
   cd YARDR
   ```

2. **Review Documentation**
   - Start with `01_Project_Overview.md`
   - Follow the documentation structure for implementation

3. **Firebase Setup**
   - Follow `03_Firebase_Configuration.md` for backend setup
   - Configure all required services and extensions

4. **Development**
   - Use `04_User_Experience_Flows.md` for UI/UX implementation
   - Reference `05_Database_Schema.md` for data structure
   - Follow `06_Backend_Functions.md` for server-side logic

## Project Structure

```
YARDR/
├── 01_Project_Overview.md
├── 02_Technical_Architecture.md
├── 03_Firebase_Configuration.md
├── 04_User_Experience_Flows.md
├── 05_Database_Schema.md
├── 06_Backend_Functions.md
├── 07_Design_System.md
├── 08_Third_Party_Integrations.md
├── YARDR_Complete_System_Documentation.md
├── Files/
│   ├── APP Flow Chart.pdf
│   ├── Renter pic 1-4.png
│   ├── Equipment Owner pic 1-3.png
│   └── Administrator pic 1-3.png
└── README.md
```

## Contributing

This is a comprehensive documentation and implementation guide for the YARDR platform. Each file contains detailed specifications, code examples, and implementation guidance.

## License

This project is proprietary and confidential. All rights reserved.

## Contact

For questions or support regarding this documentation, please contact the development team.
