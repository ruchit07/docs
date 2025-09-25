# Health Data Integration Guide for React Native

## 1. Apple Health Integration

### Available Data Types
Apple HealthKit provides access to a comprehensive set of health data:

**Vital Signs & Body Measurements:**
- Heart rate, Blood pressure, Body temperature
- Height, Weight, BMI, Body fat percentage
- Blood oxygen saturation, Respiratory rate

**Activity & Fitness:**
- Steps, Distance walked/run, Flights climbed
- Active energy burned, Basal energy burned
- Exercise workouts with duration and intensity
- Stand hours, Move minutes

**Nutrition:**
- Dietary energy consumed, Macronutrients (carbs, protein, fat)
- Micronutrients (vitamins, minerals)
- Water intake, Caffeine intake

**Sleep & Mindfulness:**
- Sleep analysis (duration, stages)
- Mindful minutes, Time in bed

**Clinical Data:**
- Blood glucose levels, HbA1c
- Cholesterol levels, Blood type
- Medications, Allergies
- Medical records, Lab results

### Integration Approach

**Required Package:**
```bash
npm install react-native-health
```

**iOS Setup:**
1. Enable HealthKit capability in Xcode
2. Add usage description in Info.plist:
```xml
<key>NSHealthShareUsageDescription</key>
<string>This app needs access to health data to provide personalized insights</string>
<key>NSHealthUpdateUsageDescription</key>
<string>This app needs to update health data for medication tracking</string>
```

**Implementation Example:**
```javascript
import { AppleHealthKit } from 'react-native-health';

const healthKitOptions = {
  permissions: {
    read: [
      'Steps', 'HeartRate', 'BloodPressure', 'Weight', 
      'Height', 'BloodGlucose', 'SleepAnalysis'
    ],
    write: ['Steps', 'Weight']
  }
};

AppleHealthKit.initHealthKit(healthKitOptions, (error) => {
  if (error) console.log('HealthKit init error');
});
```

## 2. Google Health (Health Connect) Integration

### Available Data Types
Google Health Connect provides unified access to health data:

**Activity & Fitness:**
- Steps, Distance, Active calories burned
- Exercise sessions, Heart rate during exercise
- Speed, Power output (cycling)

**Body Measurements:**
- Weight, Height, Body fat percentage
- Bone mass, Muscle mass

**Cycle Tracking:**
- Menstrual cycle data, Ovulation tests
- Cervical mucus, Sexual activity

**Nutrition:**
- Hydration, Total calories consumed
- Meals and nutrition data

**Sleep:**
- Sleep duration and stages
- Sleep sessions with detailed analysis

**Vitals:**
- Heart rate, Blood pressure
- Body temperature, Respiratory rate
- Blood glucose, Oxygen saturation

### Integration Approach

**Required Package:**
```bash
npm install @react-native-async-storage/async-storage
npm install react-native-health-connect
```

**Android Setup (Minimum API Level 26):**
1. Add permissions in AndroidManifest.xml:
```xml
<uses-permission android:name="android.permission.health.READ_STEPS" />
<uses-permission android:name="android.permission.health.READ_HEART_RATE" />
<!-- Add other health permissions as needed -->
```

**Implementation Example:**
```javascript
import { HealthConnect } from 'react-native-health-connect';

const permissions = [
  'Steps', 'HeartRate', 'BloodPressure', 'Weight', 
  'SleepSession', 'BloodGlucose'
];

HealthConnect.requestPermission(permissions)
  .then((granted) => {
    if (granted) {
      // Fetch health data
      HealthConnect.readRecords('Steps', {
        timeRangeFilter: {
          startTime: new Date(Date.now() - 7 * 24 * 60 * 60 * 1000),
          endTime: new Date()
        }
      });
    }
  });
```

**Backward Compatibility for Older Android:**
For Android versions below API 26, integrate with Google Fit:
```bash
npm install react-native-google-fit
```

## 3. Fitbit Integration

### Available Data Types
Fitbit Web API provides comprehensive fitness and health data:

**Activity & Fitness:**
- Steps, Distance, Floors climbed
- Active minutes, Calories burned
- Heart rate zones, Exercise activities

**Sleep:**
- Sleep duration, Sleep efficiency
- Sleep stages (light, deep, REM)
- Time awake during sleep

**Body & Weight:**
- Weight, BMI, Body fat percentage
- Weight goals and logs

**Heart Rate:**
- Resting heart rate, Heart rate zones
- Intraday heart rate data

**Nutrition:**
- Food logs, Water intake
- Calorie intake vs. burned

### Integration Approach

**OAuth 2.0 Setup:**
1. Register app at Fitbit Developer Console
2. Implement OAuth flow for user authorization

**Required Packages:**
```bash
npm install react-native-app-auth
npm install @react-native-async-storage/async-storage
```

**Implementation Example:**
```javascript
import { authorize, refresh } from 'react-native-app-auth';

const config = {
  issuer: 'https://fitbit.com',
  clientId: 'YOUR_FITBIT_CLIENT_ID',
  redirectUrl: 'com.yourapp://oauth/fitbit',
  scopes: ['activity', 'heartrate', 'sleep', 'weight', 'nutrition'],
  authorizeEndpoint: 'https://www.fitbit.com/oauth2/authorize',
  tokenEndpoint: 'https://api.fitbit.com/oauth2/token'
};

// OAuth Authorization
const authState = await authorize(config);

// Fetch Fitbit data
const fetchFitbitData = async (dataType, date) => {
  const response = await fetch(
    `https://api.fitbit.com/1/user/-/${dataType}/date/${date}.json`,
    {
      headers: {
        'Authorization': `Bearer ${authState.accessToken}`
      }
    }
  );
  return response.json();
};
```

## 4. Generic Health Data Schema

### Core Schema Design
```javascript
const HealthDataSchema = {
  // Unified record structure
  id: 'uuid',
  userId: 'string',
  source: 'apple_health | google_health | fitbit | manual', // Extensible for future sources
  dataType: 'string', // Standardized data type
  category: 'activity | vitals | body | sleep | nutrition | clinical',
  
  // Temporal data
  recordedAt: 'ISO8601 timestamp',
  startTime: 'ISO8601 timestamp', // For duration-based data
  endTime: 'ISO8601 timestamp',
  
  // Value with unit standardization
  value: {
    numeric: 'number | null',
    text: 'string | null',
    boolean: 'boolean | null',
    unit: 'string', // Standardized units (kg, steps, bpm, etc.)
    accuracy: 'number | null' // Confidence level if available
  },
  
  // Metadata
  metadata: {
    deviceId: 'string | null',
    deviceModel: 'string | null',
    appName: 'string | null',
    sourceVersion: 'string | null',
    processingFlags: ['raw', 'averaged', 'estimated'] // Data quality indicators
  },
  
  // HIPAA Compliance
  encryptionStatus: 'encrypted | decrypted',
  accessLog: [
    {
      accessedBy: 'string',
      accessTime: 'ISO8601 timestamp',
      operation: 'create | read | update | delete'
    }
  ],
  
  // Sync and versioning
  syncStatus: 'pending | synced | error',
  version: 'number',
  lastModified: 'ISO8601 timestamp'
};
```

### Data Type Mapping
```javascript
const DataTypeMapping = {
  // Activity
  'steps': { category: 'activity', standardUnit: 'count' },
  'distance': { category: 'activity', standardUnit: 'meters' },
  'calories_burned': { category: 'activity', standardUnit: 'kcal' },
  
  // Vitals
  'heart_rate': { category: 'vitals', standardUnit: 'bpm' },
  'blood_pressure': { category: 'vitals', standardUnit: 'mmHg' },
  'blood_glucose': { category: 'vitals', standardUnit: 'mg/dL' },
  
  // Body measurements
  'weight': { category: 'body', standardUnit: 'kg' },
  'height': { category: 'body', standardUnit: 'cm' },
  'bmi': { category: 'body', standardUnit: 'kg/m²' },
  
  // Sleep
  'sleep_duration': { category: 'sleep', standardUnit: 'minutes' },
  'sleep_efficiency': { category: 'sleep', standardUnit: 'percentage' },
  
  // Nutrition
  'water_intake': { category: 'nutrition', standardUnit: 'ml' },
  'calories_consumed': { category: 'nutrition', standardUnit: 'kcal' }
};
```

## 5. Data Insights Generation

### Activity Insights
- **Trend Analysis:** Weekly/monthly step trends, activity consistency
- **Goal Tracking:** Progress toward daily/weekly activity goals
- **Behavior Patterns:** Most active times of day, workout frequency
- **Comparative Analysis:** Performance vs. age group averages

### Health Monitoring
- **Vital Sign Trends:** Heart rate patterns, blood pressure changes
- **Risk Assessment:** Identify concerning trends in vitals
- **Medication Correlation:** Impact of medications on health metrics
- **Clinical Alerts:** Abnormal readings requiring attention

### Sleep Optimization
- **Sleep Quality Score:** Based on duration, efficiency, and consistency
- **Sleep Debt Tracking:** Cumulative sleep deficit analysis
- **Sleep Pattern Analysis:** Bedtime consistency, wake time patterns
- **Recovery Insights:** Correlation between sleep and next-day performance

### Nutrition & Weight Management
- **Caloric Balance:** Calories in vs. calories out analysis
- **Weight Trend Analysis:** Rate of weight change, plateau identification
- **Hydration Tracking:** Daily water intake vs. recommendations
- **Nutritional Balance:** Macro/micronutrient analysis

### Predictive Analytics
- **Health Risk Scoring:** ML-based risk assessment for chronic conditions
- **Medication Adherence Prediction:** Identify patients at risk of non-compliance
- **Hospitalization Risk:** Early warning system based on health trends
- **Intervention Recommendations:** Personalized health improvement suggestions

## 6. HIPAA Compliance Considerations

### Data Security
- **Encryption:** All health data encrypted at rest and in transit (AES-256)
- **Access Controls:** Role-based access with audit logging
- **Data Minimization:** Only collect necessary health data
- **Secure Storage:** Use encrypted local storage and secure cloud solutions

### Privacy Controls
- **User Consent:** Granular consent for each data type
- **Data Portability:** Allow users to export their data
- **Right to Delete:** Implement data deletion mechanisms
- **Audit Trails:** Comprehensive logging of all data access

### Technical Implementation
```javascript
// Example HIPAA-compliant data handling
const secureHealthData = {
  encrypt: (data) => {
    // Use AES-256 encryption
    return CryptoJS.AES.encrypt(JSON.stringify(data), encryptionKey);
  },
  
  logAccess: (userId, operation, dataType) => {
    // Log all health data access
    auditLogger.log({
      userId,
      operation,
      dataType,
      timestamp: new Date().toISOString(),
      ipAddress: getClientIP()
    });
  },
  
  checkConsent: (userId, dataType) => {
    // Verify user consent before data access
    return consentManager.hasConsent(userId, dataType);
  }
};
```

### Integration Architecture
```
Mobile App (React Native)
├── Health Data Collectors
│   ├── Apple HealthKit Bridge
│   ├── Google Health Connect Bridge
│   └── Fitbit OAuth Integration
├── Data Standardization Layer
│   ├── Schema Mapper
│   ├── Unit Converter
│   └── Validation Engine
├── Local Secure Storage
│   ├── Encrypted SQLite DB
│   └── Secure Key Management
├── Sync Engine
│   ├── Conflict Resolution
│   ├── Retry Logic
│   └── Offline Support
└── Analytics Engine
    ├── Trend Analysis
    ├── Insight Generation
    └── Alert System
```

This architecture provides a robust foundation for integrating multiple health data sources while maintaining HIPAA compliance and enabling powerful health insights generation.
