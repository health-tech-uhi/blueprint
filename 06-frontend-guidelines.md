# **ABDM-Compliant Healthcare Platform Frontend Guidelines**

## Table of Contents
1. [Healthcare-Specific Design Principles](#healthcare-specific-design-principles)
2. [ABDM Integration Guidelines](#abdm-integration-guidelines)
3. [Healthcare UI Component Architecture](#healthcare-ui-component-architecture)
4. [Patient & Provider Experience Design](#patient--provider-experience-design)
5. [State Management for Healthcare Data](#state-management-for-healthcare-data)
6. [Healthcare-Focused Styling Standards](#healthcare-focused-styling-standards)
7. [Medical Data Visualization](#medical-data-visualization)
8. [Healthcare Performance Practices](#healthcare-performance-practices)
9. [Medical Accessibility Guidelines](#medical-accessibility-guidelines)
10. [Healthcare Security Integration](#healthcare-security-integration)
11. [Testing Standards for Healthcare Apps](#testing-standards-for-healthcare-apps)
12. [Code Organization for Healthcare Platform](#code-organization-for-healthcare-platform)
13. [Platform-Specific Healthcare Guidelines](#platform-specific-healthcare-guidelines)

---

## Healthcare-Specific Design Principles

### 1. Medical-First User Experience
- **Clinical Workflow Integration**: Design interfaces that fit into existing medical workflows
- **Emergency-Ready Design**: Critical information and actions prominently displayed
- **Medical Terminology**: Use standardized medical terms with patient-friendly explanations
- **Trust & Credibility**: Clear verification badges for healthcare providers and secure transaction indicators
- **Cultural Healthcare Sensitivity**: Support for diverse Indian healthcare practices and languages

### 2. ABDM Ecosystem Design Standards
- **Health ID Prominence**: Display ABHA Health ID in user interfaces with QR code access
- **Consent Visualization**: Clear, understandable consent flows for health data sharing
- **Provider Verification**: Visual indicators for HPR-verified healthcare professionals
- **Facility Recognition**: Clear display of HFR-registered healthcare facilities
- **UHI Compliance**: Standardized service discovery and booking interfaces

### 3. Patient Safety & Privacy Design
- **Data Sensitivity Indicators**: Visual cues for sensitive medical information
- **Access Control Visibility**: Clear indication of who can access patient data
- **Audit Trail Access**: Easy access to data sharing history and consent logs
- **Emergency Override**: Clear emergency access procedures for critical situations
- **Family Access Controls**: Intuitive family member healthcare management interfaces

### 2. Responsive Design Standards
- **Mobile-First**: Design for mobile devices first, then scale up
- **Breakpoints**:
  - Mobile: 320px - 768px
  - Tablet: 768px - 1024px
  - Desktop: 1024px - 1440px
  - Large Desktop: 1440px+
- **Flexible Layouts**: Use CSS Grid and Flexbox for adaptive layouts
- **Touch-Friendly**: Minimum 44px touch targets on mobile devices

### 3. Progressive Enhancement
- **Core Functionality**: Ensure basic features work without JavaScript
- **Enhanced Experience**: Layer on interactive features progressively
- **Graceful Degradation**: Fallbacks for unsupported features
- **Offline Capabilities**: Core features available offline using service workers

---

## ABDM Integration Guidelines

### 1. ABHA Health ID Integration
```typescript
// ABHA Health ID Display Component
interface ABHADisplayProps {
  healthId: string;
  showQR?: boolean;
  variant: 'full' | 'compact';
}

const ABHADisplay: React.FC<ABHADisplayProps> = ({ 
  healthId, 
  showQR = false, 
  variant = 'full' 
}) => {
  const formatHealthId = (id: string) => {
    return id.replace(/(\d{2})(\d{4})(\d{4})(\d{4})/, '$1-$2-$3-$4');
  };

  return (
    <div className="abha-display bg-blue-50 border border-blue-200 rounded-lg p-4">
      <div className="flex items-center gap-3">
        <div className="abha-logo">
          <img src="/icons/abha-logo.svg" alt="ABHA" className="w-8 h-8" />
        </div>
        <div>
          <label className="text-sm font-medium text-blue-800">ABHA Health ID</label>
          <div className="text-lg font-mono text-blue-900">
            {formatHealthId(healthId)}
          </div>
        </div>
        {showQR && (
          <QRCodeGenerator value={healthId} size={64} />
        )}
      </div>
    </div>
  );
};
```

### 2. Consent Management Interface
```typescript
// Healthcare Consent Flow Component
interface ConsentFlowProps {
  dataTypes: string[];
  purpose: string;
  provider: HealthcareProvider;
  onConsent: (consent: ConsentData) => void;
}

const ConsentFlow: React.FC<ConsentFlowProps> = ({
  dataTypes,
  purpose,
  provider,
  onConsent
}) => {
  const [selectedData, setSelectedData] = useState<string[]>([]);
  const [expiryDate, setExpiryDate] = useState<Date>();

  return (
    <div className="consent-flow max-w-2xl mx-auto">
      <div className="consent-header bg-green-50 p-6 rounded-t-lg">
        <h3 className="text-xl font-semibold text-green-800">
          Healthcare Data Sharing Consent
        </h3>
        <p className="text-green-700 mt-2">
          {provider.name} is requesting access to your health data for: {purpose}
        </p>
      </div>
      
      <div className="consent-body p-6 border-x border-green-200">
        <h4 className="font-medium mb-4">Select data to share:</h4>
        {dataTypes.map((dataType) => (
          <label key={dataType} className="flex items-center gap-3 py-2">
            <input
              type="checkbox"
              checked={selectedData.includes(dataType)}
              onChange={(e) => {
                if (e.target.checked) {
                  setSelectedData([...selectedData, dataType]);
                } else {
                  setSelectedData(selectedData.filter(d => d !== dataType));
                }
              }}
            />
            <span>{dataType}</span>
          </label>
        ))}
        
        <div className="mt-6">
          <label className="block font-medium mb-2">Consent Expiry:</label>
          <input
            type="date"
            min={new Date().toISOString().split('T')[0]}
            onChange={(e) => setExpiryDate(new Date(e.target.value))}
            className="border rounded-md px-3 py-2"
          />
        </div>
      </div>
      
      <div className="consent-footer bg-gray-50 p-6 rounded-b-lg border border-green-200">
        <div className="flex gap-4">
          <button 
            onClick={() => onConsent({ dataTypes: selectedData, expiryDate, purpose })}
            className="bg-green-600 text-white px-6 py-2 rounded-md font-medium"
            disabled={selectedData.length === 0}
          >
            Grant Consent
          </button>
          <button className="bg-gray-300 text-gray-700 px-6 py-2 rounded-md">
            Deny
          </button>
        </div>
      </div>
    </div>
  );
};
```

### 3. UHI Service Discovery Interface
```typescript
// UHI-Compliant Service Search
interface ServiceSearchProps {
  searchType: 'provider' | 'service' | 'facility';
  location?: GeolocationCoordinates;
  onResults: (results: UHISearchResult[]) => void;
}

const UHIServiceSearch: React.FC<ServiceSearchProps> = ({
  searchType,
  location,
  onResults
}) => {
  const [query, setQuery] = useState('');
  const [filters, setFilters] = useState<UHIFilters>({});

  return (
    <div className="uhi-search bg-white rounded-lg shadow-lg p-6">
      <div className="search-header flex items-center gap-4 mb-6">
        <div className="flex-1">
          <input
            type="text"
            placeholder={`Search for ${searchType}s...`}
            value={query}
            onChange={(e) => setQuery(e.target.value)}
            className="w-full border border-gray-300 rounded-lg px-4 py-3"
          />
        </div>
        <button className="bg-blue-600 text-white px-6 py-3 rounded-lg">
          Search
        </button>
      </div>
      
      <UHIFilters 
        searchType={searchType}
        filters={filters}
        onFiltersChange={setFilters}
      />
    </div>
  );
};
```

---

## Healthcare UI Component Architecture

### 1. Medical Atomic Design Pattern
```
healthcare-atoms/
├── MedicalBadge/           # Drug allergy, condition severity badges
├── VitalSign/              # Blood pressure, heart rate displays
├── MedicationPill/         # Medication dosage indicators
├── HealthScore/            # Health score visualizations
└── EmergencyButton/        # Critical action buttons

healthcare-molecules/
├── PatientVitals/          # Vital signs collection
├── MedicationSchedule/     # Medication timing display
├── AppointmentSlot/        # Appointment time selections
├── LabResultCard/          # Laboratory result display
├── SymptomChecker/         # Symptom input interface
└── InsuranceCard/          # Insurance information display

healthcare-organisms/
├── PatientProfile/         # Complete patient information
├── DoctorSchedule/         # Provider availability calendar
├── MedicalHistory/         # Patient's medical timeline
├── HealthDashboard/        # Health metrics overview
├── TeleconsultationRoom/   # Video consultation interface
├── PrescriptionManager/    # Medication management
├── LabReportsViewer/       # Medical reports interface
└── EmergencyPanel/         # Critical care information

healthcare-templates/
├── PatientPortalLayout/    # Patient application layout
├── ProviderDashboardLayout/# Healthcare provider interface
├── TelehealthLayout/       # Telemedicine consultation layout
├── AdminConsoleLayout/     # Healthcare admin interface
└── EmergencyAccessLayout/  # Emergency care interface

healthcare-pages/
├── PatientRegistration/    # ABHA-enabled patient onboarding
├── ProviderVerification/   # HPR integration for providers
├── AppointmentBooking/     # UHI-compliant booking flow
├── TeleconsultationRoom/   # Remote consultation interface
├── HealthRecordViewer/     # FHIR-compliant record viewing
├── InsuranceClaims/        # NHCX claims processing
└── EmergencyAccess/        # Critical care data access
```

### 2. Healthcare Component Standards
- **Medical Data Integrity**: All medical components validate FHIR R4 data formats
- **Privacy by Design**: Components handle sensitive health data with appropriate security
- **Emergency Accessibility**: Critical medical information prominently displayed
- **Cultural Sensitivity**: Support for Indian healthcare practices and languages
- **Provider Verification**: Visual indicators for verified healthcare professionals

### 3. Medical Component Structure Template
```typescript
// MedicalComponent.tsx
import React from 'react';
import { validateFHIRResource } from '../../utils/fhir-validator';
import { MedicalComponentProps } from './MedicalComponent.types';
import { EncryptedWrapper } from './MedicalComponent.styles';

export const MedicalComponent: React.FC<MedicalComponentProps> = ({
  fhirData,
  patientConsent,
  emergencyAccess = false,
  children,
  ...restProps
}) => {
  // Validate FHIR data structure
  const isValidFHIR = validateFHIRResource(fhirData);
  
  // Check consent for data access
  const hasConsent = patientConsent?.includes(fhirData.resourceType) || emergencyAccess;
  
  if (!isValidFHIR) {
    return <ErrorBoundary error="Invalid medical data format" />;
  }
  
  if (!hasConsent) {
    return <ConsentRequiredNotice resourceType={fhirData.resourceType} />;
  }

  return (
    <EncryptedWrapper 
      className={`medical-component ${emergencyAccess ? 'emergency-access' : ''}`}
      {...restProps}
    >
      {children}
    </EncryptedWrapper>
  );
};
```

---

## Patient & Provider Experience Design

### 1. Patient-Centric Interface Design
```typescript
// Patient Dashboard with Health Overview
const PatientDashboard: React.FC = () => {
  const { patient, healthMetrics, appointments } = usePatientData();
  
  return (
    <div className="patient-dashboard grid grid-cols-1 lg:grid-cols-3 gap-6 p-6">
      {/* Health Overview */}
      <div className="col-span-2">
        <HealthOverviewCard 
          metrics={healthMetrics}
          trends={patient.healthTrends}
          alerts={patient.healthAlerts}
        />
        
        <UpcomingAppointments appointments={appointments} />
        
        <MedicationReminders 
          medications={patient.currentMedications}
          adherenceData={patient.medicationAdherence}
        />
      </div>
      
      {/* Quick Actions */}
      <div className="space-y-4">
        <QuickBookingWidget />
        <SymptomCheckerWidget />
        <EmergencyContactsCard />
        <HealthDocumentsAccess />
      </div>
    </div>
  );
};

// Provider-Optimized Clinical Interface
const ProviderDashboard: React.FC = () => {
  const { provider, todaySchedule, patientQueue } = useProviderData();
  
  return (
    <div className="provider-dashboard">
      <ClinicalHeader provider={provider} />
      
      <div className="grid grid-cols-1 xl:grid-cols-4 gap-6 p-6">
        {/* Patient Queue */}
        <div className="xl:col-span-1">
          <PatientQueue 
            patients={patientQueue}
            onPatientSelect={handlePatientSelect}
          />
        </div>
        
        {/* Clinical Workspace */}
        <div className="xl:col-span-2">
          <ClinicalWorkspace 
            currentPatient={selectedPatient}
            consultationMode="in-person"
          />
        </div>
        
        {/* Clinical Tools */}
        <div className="xl:col-span-1">
          <ClinicalDecisionSupport />
          <DrugInteractionChecker />
          <ClinicalGuidelines />
        </div>
      </div>
    </div>
  );
};
```

### 2. Responsive Healthcare Design
```typescript
// Mobile-First Healthcare Components
const ResponsiveMedicalCard: React.FC<MedicalCardProps> = ({ data, type }) => {
  return (
    <div className={`
      medical-card 
      ${type === 'emergency' ? 'border-red-500 bg-red-50' : 'border-gray-200'}
      rounded-lg border-2 p-4
      
      /* Mobile: Stack vertically */
      flex flex-col space-y-3
      
      /* Tablet: Side by side */
      md:flex-row md:space-y-0 md:space-x-4 md:items-center
      
      /* Desktop: Enhanced layout */
      lg:p-6
    `}>
      <MedicalIcon type={type} className="w-12 h-12 md:w-16 md:h-16" />
      
      <div className="flex-1">
        <h3 className="font-semibold text-lg">{data.title}</h3>
        <p className="text-gray-600 text-sm md:text-base">{data.description}</p>
        
        {type === 'emergency' && (
          <EmergencyActions actions={data.emergencyActions} />
        )}
      </div>
      
      <MedicalDataVisualization 
        data={data.metrics}
        className="w-full md:w-32 lg:w-48"
      />
    </div>
  );
};
```

---

## State Management for Healthcare Data

### 1. Healthcare-Specific State Architecture
```typescript
// Healthcare Context for Patient Data
interface HealthcareContextType {
  // Patient Management
  currentPatient: Patient | null;
  patientHistory: MedicalHistory[];
  
  // Clinical Data
  vitalSigns: VitalSigns[];
  medications: Medication[];
  allergies: Allergy[];
  conditions: Condition[];
  
  // Consent & Privacy
  activeConsents: ConsentArtefact[];
  dataAccessLog: AccessLogEntry[];
  
  // Emergency Access
  emergencyMode: boolean;
  emergencyOverride: (reason: string) => void;
  
  // FHIR Operations
  saveFHIRResource: (resource: FHIRResource) => Promise<void>;
  linkCareContext: (contextId: string) => Promise<void>;
}

// Clinical Decision Support State
interface ClinicalDSSContextType {
  // Decision Support
  recommendations: ClinicalRecommendation[];
  drugInteractions: DrugInteraction[];
  clinicalAlerts: ClinicalAlert[];
  
  // Guidelines & Protocols
  applicableGuidelines: ClinicalGuideline[];
  treatmentProtocols: TreatmentProtocol[];
  
  // Quality Metrics
  qualityIndicators: QualityMetric[];
  outcomeTracking: OutcomeData[];
}
```

### 2. FHIR-Compliant Data Fetching
```typescript
// Custom hooks for FHIR resource management
export const useFHIRResource = <T extends FHIRResource>(
  resourceType: string,
  resourceId: string,
  patientId?: string
) => {
  return useQuery({
    queryKey: ['fhir', resourceType, resourceId, patientId],
    queryFn: async () => {
      const resource = await fhirService.getResource<T>(resourceType, resourceId);
      
      // Validate FHIR structure
      if (!validateFHIRResource(resource)) {
        throw new Error('Invalid FHIR resource structure');
      }
      
      // Check patient consent for data access
      if (patientId && !await checkPatientConsent(patientId, resourceType)) {
        throw new Error('Patient consent required for data access');
      }
      
      return resource;
    },
    staleTime: 10 * 60 * 1000, // 10 minutes for medical data
    cacheTime: 30 * 60 * 1000, // 30 minutes cache
  });
};

// Mutation for creating FHIR resources
export const useCreateFHIRResource = () => {
  const queryClient = useQueryClient();
  
  return useMutation({
    mutationFn: async ({ resource, patientConsent }: {
      resource: FHIRResource;
      patientConsent: boolean;
    }) => {
      // Validate consent before creating resource
      if (!patientConsent) {
        throw new Error('Patient consent required');
      }
      
      // Validate FHIR structure
      if (!validateFHIRResource(resource)) {
        throw new Error('Invalid FHIR resource structure');
      }
      
      return await fhirService.createResource(resource);
    },
    onSuccess: (data, variables) => {
      // Update cache with new resource
      queryClient.setQueryData(
        ['fhir', data.resourceType, data.id],
        data
      );
      
      // Invalidate related queries
      queryClient.invalidateQueries(['patient', variables.resource.subject?.reference]);
    },
  });
};
```

---

## Healthcare-Focused Styling Standards

### 1. Medical Design System
```typescript
// Healthcare-specific design tokens
export const healthcareDesignTokens = {
  colors: {
    // Medical Specialties
    cardiology: '#e53e3e',      // Heart/Cardio - Red
    neurology: '#805ad5',       // Brain/Neuro - Purple  
    orthopedics: '#38a169',     // Bones/Ortho - Green
    pediatrics: '#3182ce',      // Children - Blue
    oncology: '#d69e2e',        // Cancer - Orange
    emergency: '#e53e3e',       // Emergency - Urgent Red
    
    // Medical Status
    critical: '#e53e3e',        // Critical condition
    stable: '#38a169',          // Stable condition
    improving: '#3182ce',       // Improving condition
    deteriorating: '#ed8936',   // Worsening condition
    
    // Healthcare Actions
    approved: '#38a169',        // Insurance approved
    pending: '#d69e2e',         // Pending approval
    denied: '#e53e3e',          // Claim denied
    
    // Drug Classifications
    prescription: '#805ad5',     // Prescription drugs
    otc: '#3182ce',             // Over-the-counter
    controlled: '#e53e3e',      // Controlled substances
    supplement: '#38a169',      // Supplements
  },
  
  spacing: {
    // Healthcare-optimized spacing
    pill: '0.125rem',           // Medication pill spacing
    vital: '0.25rem',           // Vital signs spacing
    clinical: '0.5rem',         // Clinical data spacing
    emergency: '1rem',          // Emergency info spacing
  },
  
  typography: {
    medical: {
      fontFamily: ['Source Sans Pro', 'system-ui', 'sans-serif'],
      sizes: {
        vital: '1.5rem',        // Vital signs display
        medication: '1rem',     // Medication names
        diagnosis: '1.25rem',   // Diagnosis display
        clinical: '0.875rem',   // Clinical notes
      },
    },
  },
  
  shadows: {
    medical: '0 2px 8px rgba(0, 0, 0, 0.1)',
    emergency: '0 4px 16px rgba(229, 62, 62, 0.3)',
    success: '0 2px 8px rgba(56, 161, 105, 0.2)',
  },
};
```

### 2. Healthcare Component Styling
```typescript
// Medical Status Indicators
const MedicalStatusBadge = styled.div<{ status: MedicalStatus }>`
  display: inline-flex;
  align-items: center;
  padding: ${({ theme }) => `${theme.spacing.pill} ${theme.spacing.clinical}`};
  border-radius: 9999px;
  font-size: ${({ theme }) => theme.typography.medical.sizes.clinical};
  font-weight: 600;
  
  ${({ status, theme }) => {
    switch (status) {
      case 'critical':
        return css`
          background-color: ${theme.colors.critical}20;
          color: ${theme.colors.critical};
          border: 1px solid ${theme.colors.critical}40;
        `;
      case 'stable':
        return css`
          background-color: ${theme.colors.stable}20;
          color: ${theme.colors.stable};
          border: 1px solid ${theme.colors.stable}40;
        `;
      case 'emergency':
        return css`
          background-color: ${theme.colors.emergency};
          color: white;
          box-shadow: ${theme.shadows.emergency};
          animation: pulse 2s infinite;
        `;
      default:
        return css`
          background-color: ${theme.colors.pending}20;
          color: ${theme.colors.pending};
        `;
    }
  }}
`;

// Emergency Alert Styling
const EmergencyAlert = styled.div`
  background: linear-gradient(135deg, #fee2e2 0%, #fecaca 100%);
  border: 2px solid #ef4444;
  border-radius: 8px;
  padding: 1rem;
  margin: 1rem 0;
  box-shadow: 0 4px 16px rgba(239, 68, 68, 0.3);
  
  &::before {
    content: '🚨';
    font-size: 1.5rem;
    margin-right: 0.5rem;
  }
  
  @keyframes pulse {
    0% { box-shadow: 0 0 0 0 rgba(239, 68, 68, 0.7); }
    70% { box-shadow: 0 0 0 10px rgba(239, 68, 68, 0); }
    100% { box-shadow: 0 0 0 0 rgba(239, 68, 68, 0); }
  }
  
  animation: pulse 2s infinite;
`;
```

---

## Medical Data Visualization

### 1. Healthcare Charts & Graphs
```typescript
// Vital Signs Trend Chart
const VitalSignsChart: React.FC<VitalSignsChartProps> = ({ 
  data, 
  vitalType, 
  timeRange 
}) => {
  const chartConfig = {
    responsive: true,
    plugins: {
      title: {
        display: true,
        text: `${vitalType} Trends - ${timeRange}`,
        font: { size: 16, weight: 'bold' }
      },
      legend: {
        position: 'top' as const,
      },
    },
    scales: {
      y: {
        beginAtZero: false,
        grid: {
          color: 'rgba(0, 0, 0, 0.1)',
        },
        ticks: {
          callback: function(value: any) {
            return vitalType === 'bloodPressure' 
              ? `${value} mmHg` 
              : `${value} ${getVitalUnit(vitalType)}`;
          }
        }
      },
      x: {
        grid: {
          color: 'rgba(0, 0, 0, 0.1)',
        },
      }
    },
    elements: {
      point: {
        radius: 4,
        hoverRadius: 6,
      },
      line: {
        borderWidth: 2,
        tension: 0.1,
      }
    }
  };

  const chartData = {
    labels: data.map(d => format(new Date(d.timestamp), 'MMM dd')),
    datasets: [
      {
        label: vitalType,
        data: data.map(d => d.value),
        borderColor: getVitalColor(vitalType),
        backgroundColor: `${getVitalColor(vitalType)}20`,
        fill: true,
      },
      // Add normal range bands
      {
        label: 'Normal Range',
        data: data.map(() => getNormalRange(vitalType).upper),
        borderColor: '#10b981',
        borderDash: [5, 5],
        pointRadius: 0,
        fill: false,
      }
    ]
  };

  return (
    <div className="vital-signs-chart bg-white p-6 rounded-lg shadow">
      <Line data={chartData} options={chartConfig} />
      
      <VitalSignsLegend vitalType={vitalType} />
      <AbnormalValueAlerts data={data} vitalType={vitalType} />
    </div>
  );
};

// Health Metrics Dashboard
const HealthMetricsDashboard: React.FC<{ patientId: string }> = ({ 
  patientId 
}) => {
  const { data: healthMetrics } = useHealthMetrics(patientId);
  
  return (
    <div className="health-metrics-dashboard grid grid-cols-1 md:grid-cols-2 lg:grid-cols-4 gap-6">
      <MetricCard
        title="Blood Pressure"
        value={healthMetrics.bloodPressure.latest}
        trend={healthMetrics.bloodPressure.trend}
        unit="mmHg"
        status={getHealthStatus(healthMetrics.bloodPressure.latest, 'bloodPressure')}
        icon="❤️"
      />
      
      <MetricCard
        title="Heart Rate"
        value={healthMetrics.heartRate.latest}
        trend={healthMetrics.heartRate.trend}
        unit="bpm"
        status={getHealthStatus(healthMetrics.heartRate.latest, 'heartRate')}
        icon="💓"
      />
      
      <MetricCard
        title="Blood Sugar"
        value={healthMetrics.bloodSugar.latest}
        trend={healthMetrics.bloodSugar.trend}
        unit="mg/dL"
        status={getHealthStatus(healthMetrics.bloodSugar.latest, 'bloodSugar')}
        icon="🩸"
      />
      
      <MetricCard
        title="BMI"
        value={healthMetrics.bmi.latest}
        trend={healthMetrics.bmi.trend}
        unit="kg/m²"
        status={getHealthStatus(healthMetrics.bmi.latest, 'bmi')}
        icon="⚖️"
      />
    </div>
  );
};
```

### 2. Medical Timeline Visualization
```typescript
// Patient Medical Timeline
const MedicalTimeline: React.FC<{ patientId: string }> = ({ patientId }) => {
  const { data: timeline } = useMedicalHistory(patientId);
  
  return (
    <div className="medical-timeline">
      <h3 className="text-xl font-semibold mb-6">Medical History Timeline</h3>
      
      <div className="timeline-container relative">
        {/* Timeline line */}
        <div className="absolute left-4 top-0 bottom-0 w-0.5 bg-gray-300"></div>
        
        {timeline.map((event, index) => (
          <TimelineEvent
            key={event.id}
            event={event}
            index={index}
            isLatest={index === 0}
          />
        ))}
      </div>
    </div>
  );
};

const TimelineEvent: React.FC<TimelineEventProps> = ({ 
  event, 
  index, 
  isLatest 
}) => {
  const getEventIcon = (type: string) => {
    switch (type) {
      case 'diagnosis': return '🔍';
      case 'treatment': return '💊';
      case 'surgery': return '🏥';
      case 'consultation': return '👨‍⚕️';
      default: return '📋';
    }
  };

  return (
    <div className={`timeline-event flex items-start space-x-4 pb-8 ${
      isLatest ? 'text-blue-600' : 'text-gray-600'
    }`}>
      {/* Timeline marker */}
      <div className={`timeline-marker relative z-10 flex items-center justify-center w-8 h-8 rounded-full ${
        isLatest ? 'bg-blue-100 border-2 border-blue-500' : 'bg-gray-100 border-2 border-gray-300'
      }`}>
        <span className="text-sm">{getEventIcon(event.type)}</span>
      </div>
      
      {/* Event content */}
      <div className="timeline-content flex-1 min-w-0">
        <div className="flex items-center justify-between">
          <h4 className="font-medium">{event.title}</h4>
          <span className="text-sm text-gray-500">
            {format(new Date(event.date), 'MMM dd, yyyy')}
          </span>
        </div>
        
        <p className="text-sm mt-1">{event.description}</p>
        
        {event.provider && (
          <p className="text-xs text-gray-500 mt-2">
            Provider: {event.provider.name}
          </p>
        )}
        
        {event.attachments && event.attachments.length > 0 && (
          <div className="flex gap-2 mt-3">
            {event.attachments.map((attachment) => (
              <AttachmentPreview key={attachment.id} attachment={attachment} />
            ))}
          </div>
        )}
      </div>
    </div>
  );
};
```

Now I'll continue with the remaining sections and then move to the final .md file. Let me update the last .md file in our sequence.

<read_file>
<path>blueprint/08-optimized-react-code-guidelines.md</path>

### 2. Component Standards
- **Single Responsibility**: Each component has one clear purpose
- **Reusability**: Design components to be used across different contexts
- **Composability**: Build complex components from simpler ones
- **Props Interface**: Clear, typed props with default values

### 3. Component Structure Template
```typescript
// ComponentName.tsx
import React from 'react';
import { ComponentNameProps } from './ComponentName.types';
import { StyledWrapper } from './ComponentName.styles';

export const ComponentName: React.FC<ComponentNameProps> = ({
  prop1,
  prop2,
  children,
  ...restProps
}) => {
  return (
    <StyledWrapper {...restProps}>
      {/* Component content */}
    </StyledWrapper>
  );
};

ComponentName.defaultProps = {
  prop1: 'defaultValue',
};

export default ComponentName;
```

---

## State Management

### 1. State Management Strategy
- **Local State**: Use `useState` for component-specific state
- **Shared State**: Use Context API for theme, auth, and global settings
- **Server State**: Use React Query/TanStack Query for API data
- **Form State**: Use React Hook Form for complex forms
- **URL State**: Use Next.js router for navigation state

### 2. Context Structure
```typescript
// contexts/AuthContext.tsx
interface AuthContextType {
  user: User | null;
  login: (credentials: LoginCredentials) => Promise<void>;
  logout: () => void;
  isLoading: boolean;
  error: string | null;
}

// contexts/ThemeContext.tsx
interface ThemeContextType {
  theme: 'light' | 'dark' | 'system';
  setTheme: (theme: ThemeType) => void;
  colors: ColorPalette;
}

// contexts/NotificationContext.tsx
interface NotificationContextType {
  notifications: Notification[];
  addNotification: (notification: Notification) => void;
  removeNotification: (id: string) => void;
  markAsRead: (id: string) => void;
}
```

### 3. Data Fetching Patterns
```typescript
// hooks/useAppointments.ts
export const useAppointments = (patientId: string) => {
  return useQuery({
    queryKey: ['appointments', patientId],
    queryFn: () => appointmentService.getByPatient(patientId),
    staleTime: 5 * 60 * 1000, // 5 minutes
    cacheTime: 10 * 60 * 1000, // 10 minutes
  });
};

// hooks/useCreateAppointment.ts
export const useCreateAppointment = () => {
  const queryClient = useQueryClient();
  
  return useMutation({
    mutationFn: appointmentService.create,
    onSuccess: (data) => {
      queryClient.invalidateQueries(['appointments']);
      queryClient.setQueryData(['appointment', data.id], data);
    },
  });
};
```

---

## Styling Standards

### 1. CSS Architecture
- **Utility-First**: Use Tailwind CSS for rapid development
- **Component Styles**: Use CSS-in-JS (styled-components) for complex components
- **Design Tokens**: Centralized design system with consistent values
- **CSS Modules**: For component-specific styles when needed

### 2. Design System Structure
```typescript
// theme/tokens.ts
export const designTokens = {
  colors: {
    primary: {
      50: '#f0f9ff',
      100: '#e0f2fe',
      500: '#0ea5e9',
      600: '#0284c7',
      900: '#0c4a6e',
    },
    semantic: {
      success: '#10b981',
      warning: '#f59e0b',
      error: '#ef4444',
      info: '#3b82f6',
    },
    healthcare: {
      hospital: '#2563eb',
      pharmacy: '#059669',
      lab: '#dc2626',
      emergency: '#dc2626',
    },
  },
  spacing: {
    xs: '0.25rem',
    sm: '0.5rem',
    md: '1rem',
    lg: '1.5rem',
    xl: '2rem',
    xxl: '3rem',
  },
  typography: {
    fontFamily: {
      sans: ['Inter', 'system-ui', 'sans-serif'],
      mono: ['JetBrains Mono', 'monospace'],
    },
    fontSize: {
      xs: '0.75rem',
      sm: '0.875rem',
      base: '1rem',
      lg: '1.125rem',
      xl: '1.25rem',
      '2xl': '1.5rem',
      '3xl': '1.875rem',
    },
  },
};
```

### 3. Component Styling Patterns
```typescript
// Styled Components Pattern
const StyledButton = styled.button<{ variant: 'primary' | 'secondary' }>`
  padding: ${({ theme }) => `${theme.spacing.sm} ${theme.spacing.md}`};
  border-radius: ${({ theme }) => theme.borderRadius.md};
  font-weight: 600;
  transition: all 0.2s ease-in-out;
  
  ${({ variant, theme }) => variant === 'primary' && css`
    background-color: ${theme.colors.primary[500]};
    color: white;
    
    &:hover {
      background-color: ${theme.colors.primary[600]};
    }
  `}
`;

// Tailwind + CSS Variables Pattern
:root {
  --color-primary: 14 165 233;
  --color-success: 16 185 129;
  --color-error: 239 68 68;
}

.btn-primary {
  @apply bg-primary-500 text-white px-4 py-2 rounded-md font-medium;
  @apply hover:bg-primary-600 focus:ring-2 focus:ring-primary-500;
  @apply transition-colors duration-200;
}
```

---

## Performance Practices

### 1. Code Splitting & Lazy Loading
```typescript
// Route-based code splitting
const PatientDashboard = lazy(() => import('../pages/PatientDashboard'));
const ProviderDashboard = lazy(() => import('../pages/ProviderDashboard'));
const AppointmentBooking = lazy(() => import('../pages/AppointmentBooking'));

// Component-based lazy loading
const HeavyChart = lazy(() => import('../components/HeavyChart'));

const Dashboard = () => (
  <Suspense fallback={<DashboardSkeleton />}>
    <HeavyChart data={chartData} />
  </Suspense>
);
```

### 2. Image Optimization
```typescript
// Next.js Image component with optimization
import Image from 'next/image';

const ProfileAvatar = ({ src, alt }: { src: string; alt: string }) => (
  <Image
    src={src}
    alt={alt}
    width={48}
    height={48}
    placeholder="blur"
    blurDataURL="data:image/jpeg;base64,..."
    quality={85}
    priority={false}
  />
);

// Progressive image loading
const useProgressiveImage = (src: string) => {
  const [imgSrc, setImgSrc] = useState<string>();
  
  useEffect(() => {
    const img = new Image();
    img.src = src;
    img.onload = () => setImgSrc(src);
  }, [src]);
  
  return imgSrc;
};
```

### 3. Bundle Optimization
```javascript
// webpack.config.js optimizations
module.exports = {
  optimization: {
    splitChunks: {
      chunks: 'all',
      cacheGroups: {
        vendor: {
          test: /[\\/]node_modules[\\/]/,
          name: 'vendors',
          chunks: 'all',
        },
        common: {
          name: 'common',
          minChunks: 2,
          chunks: 'all',
        },
      },
    },
  },
  resolve: {
    alias: {
      // Tree-shaking for lodash
      'lodash': 'lodash-es',
    },
  },
};
```

### 4. Caching Strategies
```typescript
// Service Worker for caching
const CACHE_NAME = 'health-tech-v1';
const urlsToCache = [
  '/',
  '/static/js/bundle.js',
  '/static/css/main.css',
  '/manifest.json',
];

self.addEventListener('install', (event) => {
  event.waitUntil(
    caches.open(CACHE_NAME)
      .then((cache) => cache.addAll(urlsToCache))
  );
});

// React Query cache configuration
const queryClient = new QueryClient({
  defaultOptions: {
    queries: {
      staleTime: 5 * 60 * 1000, // 5 minutes
      cacheTime: 10 * 60 * 1000, // 10 minutes
      retry: 3,
      refetchOnWindowFocus: false,
    },
  },
});
```

---

## Accessibility Guidelines

### 1. WCAG 2.1 AA Compliance
- **Color Contrast**: Minimum 4.5:1 for normal text, 3:1 for large text
- **Keyboard Navigation**: All interactive elements accessible via keyboard
- **Screen Readers**: Proper ARIA labels and semantic HTML
- **Focus Management**: Clear focus indicators and logical tab order

### 2. Accessibility Implementation
```typescript
// Accessible Button Component
const AccessibleButton = ({
  children,
  onClick,
  disabled = false,
  ariaLabel,
  ...props
}) => (
  <button
    onClick={onClick}
    disabled={disabled}
    aria-label={ariaLabel}
    aria-disabled={disabled}
    className="focus:ring-2 focus:ring-primary-500 focus:outline-none"
    {...props}
  >
    {children}
  </button>
);

// Accessible Form with proper labeling
const AppointmentForm = () => (
  <form>
    <label htmlFor="appointment-date" className="sr-only">
      Select appointment date
    </label>
    <input
      id="appointment-date"
      type="date"
      aria-required="true"
      aria-describedby="date-error"
    />
    <div id="date-error" role="alert" aria-live="polite">
      {dateError}
    </div>
  </form>
);
```

### 3. Screen Reader Support
```typescript
// Announce dynamic content changes
const useAnnouncement = () => {
  const [announcement, setAnnouncement] = useState('');
  
  const announce = (message: string) => {
    setAnnouncement('');
    setTimeout(() => setAnnouncement(message), 100);
  };
  
  return { announcement, announce };
};

// Live regions for dynamic updates
const NotificationToast = ({ message }: { message: string }) => (
  <div
    role="alert"
    aria-live="assertive"
    className="fixed top-4 right-4 bg-green-500 text-white p-4 rounded"
  >
    {message}
  </div>
);
```

---

## Testing Standards

### 1. Testing Pyramid
- **Unit Tests (70%)**: Test individual components and functions
- **Integration Tests (20%)**: Test component interactions
- **E2E Tests (10%)**: Test complete user workflows

### 2. Component Testing Patterns
```typescript
// React Testing Library examples
import { render, screen, fireEvent, waitFor } from '@testing-library/react';
import userEvent from '@testing-library/user-event';

describe('AppointmentBooking', () => {
  test('should book appointment successfully', async () => {
    const mockBookAppointment = jest.fn();
    const user = userEvent.setup();
    
    render(
      <AppointmentBooking onBook={mockBookAppointment} />
    );
    
    // Select date
    await user.click(screen.getByLabelText(/select date/i));
    await user.click(screen.getByText('Tomorrow'));
    
    // Select time slot
    await user.click(screen.getByText('10:00 AM'));
    
    // Submit form
    await user.click(screen.getByRole('button', { name: /book appointment/i }));
    
    await waitFor(() => {
      expect(mockBookAppointment).toHaveBeenCalledWith({
        date: expect.any(Date),
        time: '10:00 AM',
      });
    });
  });
});
```

### 3. Accessibility Testing
```typescript
// axe-core integration
import { axe, toHaveNoViolations } from 'jest-axe';

expect.extend(toHaveNoViolations);

test('should not have accessibility violations', async () => {
  const { container } = render(<PatientDashboard />);
  const results = await axe(container);
  expect(results).toHaveNoViolations();
});
```

---

## Code Organization

### 1. File Structure
```
src/
├── components/
│   ├── ui/           # Reusable UI components
│   ├── forms/        # Form components
│   ├── charts/       # Data visualization
│   └── layout/       # Layout components
├── pages/            # Page components
├── hooks/            # Custom React hooks
├── contexts/         # React contexts
├── services/         # API services
├── utils/            # Utility functions
├── types/            # TypeScript type definitions
├── constants/        # Application constants
├── theme/            # Design system
└── __tests__/        # Test files
```

### 2. Import Organization
```typescript
// 1. External libraries
import React, { useState, useEffect } from 'react';
import { useQuery } from '@tanstack/react-query';
import styled from 'styled-components';

// 2. Internal utilities and services
import { apiClient } from '../../services/api';
import { formatDate } from '../../utils/date';

// 3. Component imports
import { Button } from '../ui/Button';
import { Input } from '../ui/Input';

// 4. Type imports
import type { Appointment, Patient } from '../../types/medical';

// 5. Local imports
import { StyledContainer } from './AppointmentCard.styles';
```

### 3. Naming Conventions
- **Components**: PascalCase (e.g., `PatientDashboard`)
- **Files**: PascalCase for components, camelCase for utilities
- **Functions**: camelCase (e.g., `handleSubmit`)
- **Constants**: SCREAMING_SNAKE_CASE (e.g., `API_ENDPOINTS`)
- **CSS Classes**: kebab-case (e.g., `appointment-card`)

---

## Platform-Specific Guidelines

### 1. React Web Application
```typescript
// Progressive Web App configuration
// public/manifest.json
{
  "name": "HealthTech UHI Platform",
  "short_name": "HealthTech",
  "description": "Universal Health Interface Platform",
  "start_url": "/",
  "display": "standalone",
  "theme_color": "#0ea5e9",
  "background_color": "#ffffff",
  "icons": [
    {
      "src": "/icons/icon-192x192.png",
      "sizes": "192x192",
      "type": "image/png"
    }
  ]
}

// Service Worker registration
if ('serviceWorker' in navigator) {
  window.addEventListener('load', () => {
    navigator.serviceWorker.register('/sw.js')
      .then((registration) => {
        console.log('SW registered: ', registration);
      })
      .catch((registrationError) => {
        console.log('SW registration failed: ', registrationError);
      });
  });
}
```

### 2. iOS Swift/SwiftUI Guidelines
```swift
// MVVM Architecture
class AppointmentViewModel: ObservableObject {
    @Published var appointments: [Appointment] = []
    @Published var isLoading = false
    @Published var errorMessage: String?
    
    private let appointmentService: AppointmentService
    
    init(appointmentService: AppointmentService = .shared) {
        self.appointmentService = appointmentService
    }
    
    @MainActor
    func loadAppointments() async {
        isLoading = true
        do {
            appointments = try await appointmentService.fetchAppointments()
        } catch {
            errorMessage = error.localizedDescription
        }
        isLoading = false
    }
}

// SwiftUI View with accessibility
struct AppointmentRow: View {
    let appointment: Appointment
    
    var body: some View {
        HStack {
            VStack(alignment: .leading) {
                Text(appointment.providerName)
                    .font(.headline)
                Text(appointment.dateTime, style: .date)
                    .font(.caption)
                    .foregroundColor(.secondary)
            }
            Spacer()
            Image(systemName: "chevron.right")
        }
        .accessibilityElement(children: .combine)
        .accessibilityLabel("Appointment with \(appointment.providerName) on \(appointment.dateTime, style: .date)")
    }
}
```

### 3. Android Kotlin/Jetpack Compose Guidelines
```kotlin
// MVVM with Repository pattern
class AppointmentViewModel(
    private val repository: AppointmentRepository
) : ViewModel() {
    
    private val _uiState = MutableStateFlow(AppointmentUiState())
    val uiState: StateFlow<AppointmentUiState> = _uiState.asStateFlow()
    
    fun loadAppointments() {
        viewModelScope.launch {
            _uiState.update { it.copy(isLoading = true) }
            try {
                val appointments = repository.getAppointments()
                _uiState.update { 
                    it.copy(
                        appointments = appointments,
                        isLoading = false
                    )
                }
            } catch (exception: Exception) {
                _uiState.update { 
                    it.copy(
                        errorMessage = exception.message,
                        isLoading = false
                    )
                }
            }
        }
    }
}

// Jetpack Compose with accessibility
@Composable
fun AppointmentCard(
    appointment: Appointment,
    onClick: () -> Unit,
    modifier: Modifier = Modifier
) {
    Card(
        onClick = onClick,
        modifier = modifier
            .fillMaxWidth()
            .semantics {
                contentDescription = "Appointment with ${appointment.providerName} on ${appointment.date}"
            }
    ) {
        Column(
            modifier = Modifier.padding(16.dp)
        ) {
            Text(
                text = appointment.providerName,
                style = MaterialTheme.typography.headlineSmall
            )
            Text(
                text = appointment.date,
                style = MaterialTheme.typography.bodyMedium,
                color = MaterialTheme.colorScheme.onSurfaceVariant
            )
        }
    }
}
```

---

## Performance Monitoring

### 1. Core Web Vitals Tracking
```typescript
// Performance monitoring setup
import { getCLS, getFID, getFCP, getLCP, getTTFB } from 'web-vitals';

const sendToAnalytics = (metric) => {
  // Send to your analytics service
  analytics.track('performance_metric', {
    name: metric.name,
    value: metric.value,
    rating: metric.rating,
  });
};

getCLS(sendToAnalytics);
getFID(sendToAnalytics);
getFCP(sendToAnalytics);
getLCP(sendToAnalytics);
getTTFB(sendToAnalytics);
```

### 2. React Performance Profiling
```typescript
// Performance boundary component
const PerformanceBoundary = ({ children, name }) => {
  useEffect(() => {
    const observer = new PerformanceObserver((list) => {
      list.getEntries().forEach((entry) => {
        if (entry.name.includes(name)) {
          console.log(`${name} render time:`, entry.duration);
        }
      });
    });
    
    observer.observe({ entryTypes: ['measure'] });
    
    return () => observer.disconnect();
  }, [name]);
  
  return children;
};
```

---

## Security Integration

### 1. Frontend Security Headers
```typescript
// Next.js security headers
const securityHeaders = [
  {
    key: 'X-DNS-Prefetch-Control',
    value: 'on'
  },
  {
    key: 'Strict-Transport-Security',
    value: 'max-age=31536000; includeSubDomains; preload'
  },
  {
    key: 'X-XSS-Protection',
    value: '1; mode=block'
  },
  {
    key: 'X-Frame-Options',
    value: 'SAMEORIGIN'
  },
  {
    key: 'X-Content-Type-Options',
    value: 'nosniff'
  },
  {
    key: 'Referrer-Policy',
