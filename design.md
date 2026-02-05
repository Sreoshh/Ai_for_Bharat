# Design Document: AI-Powered Emergency Ambulance Dispatch

## Overview

The AI-powered emergency ambulance dispatch system is designed as a distributed, real-time application that optimizes emergency medical response through intelligent ambulance allocation. The system consists of a mobile application for users, a backend API for coordination, an AI decision engine for ambulance selection, and integration points with hospitals and ambulance providers.

The architecture prioritizes low latency for emergency requests, high availability through redundancy, and intelligent decision-making through machine learning algorithms. The system handles variable network conditions and maintains functionality during partial outages through caching and offline capabilities.

## Architecture

The system follows a microservices architecture with the following key components:

![Current Panel](assets/png_01)

The architecture ensures:
- **Scalability**: Microservices can be scaled independently based on load
- **Reliability**: Service redundancy and circuit breakers prevent cascading failures
- **Performance**: Caching and optimized data structures minimize response times
- **Security**: API gateway handles authentication and rate limiting

## Components and Interfaces

### Emergency Request Service
**Responsibility**: Manages the lifecycle of emergency requests from creation to completion.

**Key Interfaces**:
- `POST /emergency/request` - Creates new emergency request
- `GET /emergency/{id}/status` - Retrieves request status
- `PUT /emergency/{id}/update` - Updates request information

**Internal Functions**:
- `validateEmergencyRequest(request: EmergencyRequest): ValidationResult`
- `assignAmbulance(requestId: string, ambulanceId: string): AssignmentResult`
- `notifyStakeholders(request: EmergencyRequest, event: EventType): void`

### AI Decision Service
**Responsibility**: Implements intelligent ambulance selection using machine learning algorithms.

**Key Interfaces**:
- `POST /ai/rank-ambulances` - Returns ranked list of available ambulances
- `POST /ai/predict-eta` - Predicts arrival time using historical data
- `GET /ai/model-status` - Returns current model performance metrics

**Internal Functions**:
- `rankAmbulances(request: EmergencyRequest, ambulances: Ambulance[]): RankedAmbulance[]`
- `calculateScore(ambulance: Ambulance, request: EmergencyRequest): number`
- `updateModel(historicalData: ResponseData[]): ModelUpdateResult`

**Algorithm Details**:
The ranking algorithm uses a weighted scoring system:
- Distance weight: 40%
- Traffic conditions: 30%
- Ambulance capabilities: 20%
- Historical performance: 10%

When sufficient historical data exists (>100 completed requests), the system employs a lightweight linear regression model to improve ETA predictions and ambulance ranking accuracy.

### Location Service
**Responsibility**: Tracks real-time positions of ambulances and users with high accuracy and low latency.

**Key Interfaces**:
- `POST /location/update` - Updates entity location
- `GET /location/{entityId}` - Retrieves current location
- `GET /location/nearby` - Finds entities within radius

**Internal Functions**:
- `updateLocation(entityId: string, coordinates: GeoCoordinates): void`
- `getNearbyAmbulances(location: GeoCoordinates, radius: number): Ambulance[]`
- `calculateDistance(point1: GeoCoordinates, point2: GeoCoordinates): number`

### ETA Calculator
**Responsibility**: Provides accurate arrival time predictions with real-time updates.

**Key Interfaces**:
- `POST /eta/calculate` - Calculates initial ETA
- `PUT /eta/{requestId}/update` - Updates ETA based on current conditions
- `GET /eta/{requestId}` - Retrieves current ETA

**Internal Functions**:
- `calculateInitialETA(origin: GeoCoordinates, destination: GeoCoordinates): ETAResult`
- `updateETAWithTraffic(routeId: string, trafficData: TrafficConditions): ETAResult`
- `applyHistoricalCorrection(baseETA: number, routeCharacteristics: RouteData): number`

### Notification Service
**Responsibility**: Handles real-time notifications to all stakeholders.

**Key Interfaces**:
- `POST /notifications/send` - Sends notification to specific recipients
- `POST /notifications/broadcast` - Broadcasts to all relevant parties
- `GET /notifications/{userId}/history` - Retrieves notification history

**Internal Functions**:
- `sendPushNotification(userId: string, message: NotificationMessage): void`
- `sendSMSFallback(phoneNumber: string, message: string): void`
- `logNotification(notification: NotificationRecord): void`

## Data Models

### Core Entities

```typescript
interface EmergencyRequest {
  id: string;
  userId: string;
  location: GeoCoordinates;
  timestamp: Date;
  status: RequestStatus;
  priority: Priority;
  assignedAmbulanceId?: string;
  estimatedArrival?: Date;
  completedAt?: Date;
  medicalInfo?: MedicalInformation;
}

interface Ambulance {
  id: string;
  providerId: string;
  currentLocation: GeoCoordinates;
  status: AmbulanceStatus;
  capabilities: EquipmentType[];
  crewSize: number;
  lastUpdated: Date;
  isAvailable: boolean;
}

interface Hospital {
  id: string;
  name: string;
  location: GeoCoordinates;
  emergencyCapacity: number;
  currentOccupancy: number;
  specializations: MedicalSpecialty[];
  isAcceptingPatients: boolean;
  lastStatusUpdate: Date;
}

interface User {
  id: string;
  phoneNumber: string;
  emergencyContacts: ContactInfo[];
  medicalConditions?: string[];
  location?: GeoCoordinates;
  isVerified: boolean;
}
```

### Supporting Types

```typescript
interface GeoCoordinates {
  latitude: number;
  longitude: number;
  accuracy: number;
  timestamp: Date;
}

interface RankedAmbulance {
  ambulance: Ambulance;
  score: number;
  estimatedArrival: Date;
  distance: number;
  reasoning: string[];
}

interface ETAResult {
  estimatedMinutes: number;
  confidence: number;
  route: RoutePoint[];
  trafficConditions: TrafficLevel;
  lastUpdated: Date;
}

enum RequestStatus {
  PENDING = 'pending',
  ASSIGNED = 'assigned',
  EN_ROUTE = 'en_route',
  ARRIVED = 'arrived',
  TRANSPORTING = 'transporting',
  COMPLETED = 'completed',
  CANCELLED = 'cancelled'
}

enum AmbulanceStatus {
  AVAILABLE = 'available',
  ASSIGNED = 'assigned',
  EN_ROUTE = 'en_route',
  AT_SCENE = 'at_scene',
  TRANSPORTING = 'transporting',
  OFF_DUTY = 'off_duty',
  MAINTENANCE = 'maintenance'
}
```

### Data Storage Strategy

**PostgreSQL (Primary Database)**:
- Emergency requests and their lifecycle
- User profiles and authentication data
- Ambulance and hospital registration data
- Historical response analytics

**Redis Cache**:
- Real-time location data (TTL: 30 seconds)
- Ambulance availability status (TTL: 60 seconds)
- Active emergency request cache (TTL: 4 hours)
- Session data and temporary tokens

**Time Series Database (InfluxDB)**:
- Location tracking history
- Performance metrics and system monitoring
- ETA accuracy tracking
- Response time analytics

## Correctness Properties

*A property is a characteristic or behavior that should hold true across all valid executions of a system—essentially, a formal statement about what the system should do. Properties serve as the bridge between human-readable specifications and machine-verifiable correctness guarantees.*

Based on the requirements analysis, the following properties ensure the system behaves correctly across all scenarios:

### Property 1: Emergency Request Processing
*For any* valid emergency request, the system should capture GPS location within 5 seconds, validate the request, create a unique emergency ID, and notify all ambulance providers within 10km radius.
**Validates: Requirements 1.1, 1.2, 1.4**

### Property 2: AI Ambulance Ranking Consistency
*For any* set of available ambulances and emergency request, the AI decision engine should rank ambulances based on distance, traffic conditions, and capabilities, ensuring only available ambulances are considered for assignment.
**Validates: Requirements 2.1, 2.2, 2.3, 2.5**

### Property 3: ML Enhancement Utilization
*For any* scenario where sufficient historical data exists (>100 completed requests), the AI decision engine should be able to use regression models to improve ambulance ranking accuracy and ETA predictions.
**Validates: Requirements 2.6, 4.6**

### Property 4: Real-time Location Tracking
*For any* dispatched ambulance, the location service should track GPS position with updates every 10 seconds, display positions within 50-meter accuracy, and update user displays within 15 seconds of location changes.
**Validates: Requirements 3.1, 3.2, 3.3**

### Property 5: Proximity-based Notifications
*For any* ambulance approaching a user location, the system should send arrival notifications when the ambulance is within 100 meters of the destination.
**Validates: Requirements 3.5**

### Property 6: Dynamic ETA Management
*For any* dispatched ambulance, the ETA calculator should provide initial estimates based on traffic conditions, recalculate within 2 minutes when traffic changes, adjust for route deviations, and provide minute-by-minute updates when within 5 minutes of arrival.
**Validates: Requirements 4.1, 4.2, 4.3, 4.5**

### Property 7: ETA Change Notifications
*For any* ETA that changes by more than 5 minutes, the system should notify the user of the updated arrival time.
**Validates: Requirements 4.4**

### Property 8: Hospital Status Management
*For any* hospital in the system, availability status should be provided in real-time for selection requests, updated within 5 minutes when capacity changes, and notifications sent to ambulance providers when availability changes.
**Validates: Requirements 5.1, 5.2, 5.5**

### Property 9: Hospital Recommendation Logic
*For any* patient transport scenario with multiple available hospitals, the system should recommend the closest hospital with appropriate medical specialization.
**Validates: Requirements 5.4**

### Property 10: Bed Reservation Process
*For any* patient transport, the system should reserve a bed at the destination hospital and notify the receiving facility of the incoming patient.
**Validates: Requirements 5.3**

### Property 11: Ambulance Status Lifecycle
*For any* ambulance in the system, status updates should be applied correctly when going on/off duty, being assigned to emergencies, and completing transports, with availability managed appropriately.
**Validates: Requirements 6.1, 6.2, 6.3**

### Property 12: Equipment Capability Management
*For any* ambulance with equipment issues, providers should be able to mark specific capabilities as unavailable, and the system should filter ambulances based on required equipment.
**Validates: Requirements 6.4**

### Property 13: Fleet Data Validation
*For any* ambulance provider fleet update, the system should validate and store the updated data within 30 seconds.
**Validates: Requirements 6.5**

### Property 14: Data Security and Privacy
*For any* user location data collected, the system should encrypt it using AES-256 encryption, share only necessary information with ambulance providers, and manage user authentication with secure methods and 24-hour session expiration.
**Validates: Requirements 7.1, 7.3, 7.5**

### Property 15: Data Deletion Compliance
*For any* user data deletion request, the system should remove all personal data within 30 days while preserving anonymized analytics.
**Validates: Requirements 7.4**

### Property 16: Offline Synchronization
*For any* period of network connectivity restoration after offline operation, the system should synchronize all pending requests and status updates within 60 seconds.
**Validates: Requirements 8.3**

### Property 17: Dashboard Completeness
*For any* coordinator viewing the dashboard, all active emergency requests should be displayed with current status and assigned ambulances, with alerts highlighted for requests exceeding expected response times.
**Validates: Requirements 9.1, 9.2**

### Property 18: System Performance Monitoring
*For any* system performance degradation, alerts should be displayed about affected services or regions, and coordinators should be able to reassign emergencies when ambulances report issues.
**Validates: Requirements 9.3, 9.4**

### Property 19: Analytics Report Generation
*For any* report generation request, the system should provide analytics on response times, success rates, and system utilization.
**Validates: Requirements 9.5**

### Property 20: Route Calculation Integration
*For any* route calculation request, the system should integrate with mapping services to provide optimal paths considering current traffic conditions.
**Validates: Requirements 10.1**

### Property 21: Traffic Data Integration
*For any* traffic update received, the system should incorporate the new data into route calculations within 2 minutes.
**Validates: Requirements 10.3**

### Property 22: Healthcare Data Format Compliance
*For any* hospital system integration, data exchanges should use standardized healthcare data formats (HL7 FHIR).
**Validates: Requirements 10.4**

## Error Handling

The system implements comprehensive error handling strategies to maintain reliability during various failure scenarios:

### Network Connectivity Issues
- **Offline Mode**: Store emergency requests locally when network is unavailable
- **Retry Logic**: Exponential backoff for failed API calls (30s, 60s, 120s intervals)
- **Graceful Degradation**: Use cached data when real-time updates are unavailable
- **Priority Queuing**: Critical emergency data takes precedence over non-essential updates

### GPS and Location Failures
- **Fallback Mechanisms**: Manual location entry when GPS is unavailable
- **Last Known Position**: Use cached location data with clear indicators of staleness
- **Accuracy Validation**: Reject location data with accuracy worse than 100 meters
- **Signal Loss Handling**: Maintain last known position and notify users of signal issues

### External Service Failures
- **Circuit Breaker Pattern**: Prevent cascading failures from external service outages
- **Cached Route Data**: Fall back to stored route information when mapping services fail
- **Service Health Monitoring**: Track external service availability and response times
- **Alternative Providers**: Switch to backup services when primary providers are unavailable

### Data Validation and Integrity
- **Input Sanitization**: Validate all user inputs and API requests
- **Schema Validation**: Ensure data conforms to expected formats before processing
- **Duplicate Detection**: Prevent duplicate emergency requests within 5-minute windows
- **Consistency Checks**: Verify data integrity across distributed components

### Performance and Scalability Issues
- **Load Balancing**: Distribute requests across multiple service instances
- **Auto-scaling**: Automatically scale services based on demand
- **Database Connection Pooling**: Manage database connections efficiently
- **Caching Strategies**: Implement multi-level caching to reduce database load

## Testing Strategy

The testing approach combines unit testing for specific scenarios with property-based testing for comprehensive coverage:

### Unit Testing Approach
Unit tests focus on:
- **Specific Examples**: Test concrete scenarios like "ambulance A is closer than ambulance B"
- **Edge Cases**: Handle GPS unavailability, network failures, and service outages
- **Integration Points**: Verify correct interaction between components
- **Error Conditions**: Ensure proper error handling and user feedback

### Property-Based Testing Configuration
- **Testing Framework**: Use Hypothesis (Python) for the AI Decision Service and fast-check (TypeScript) for the API services
- **Test Iterations**: Minimum 100 iterations per property test to ensure comprehensive coverage
- **Data Generation**: Create realistic test data including:
  - Random geographic coordinates within service areas
  - Varied ambulance configurations and capabilities
  - Different traffic conditions and time scenarios
  - Hospital capacity and specialization combinations

### Property Test Implementation
Each correctness property will be implemented as a property-based test with the following structure:

```python
# Example property test structure
@given(emergency_requests=emergency_request_strategy(),
       ambulances=ambulance_list_strategy())
def test_ambulance_ranking_consistency(emergency_requests, ambulances):
    """
    Feature: ai-ambulance-dispatch, Property 2: AI Ambulance Ranking Consistency
    For any set of available ambulances and emergency request, the AI decision 
    engine should rank ambulances based on distance, traffic conditions, and 
    capabilities, ensuring only available ambulances are considered.
    """
    # Test implementation
    pass
```

### Test Data Strategies
- **Geographic Data**: Generate coordinates within realistic service boundaries
- **Time-based Data**: Test across different times of day and traffic patterns
- **Equipment Variations**: Test all combinations of ambulance capabilities
- **Network Conditions**: Simulate various connectivity scenarios
- **Load Testing**: Verify performance under high request volumes

### Continuous Testing
- **Automated Test Execution**: Run all tests on every code change
- **Performance Benchmarks**: Monitor response times and system throughput
- **Integration Testing**: Verify end-to-end workflows in staging environment
- **Chaos Engineering**: Test system resilience under controlled failure conditions
