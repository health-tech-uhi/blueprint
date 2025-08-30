# Section 8: Optimized React Code Guidelines

## Table of Contents
1. [Performance Best Practices](#performance-best-practices)
2. [Common Performance Pitfalls](#common-performance-pitfalls)
3. [Optimization Techniques](#optimization-techniques)
4. [Component Structure & Patterns](#component-structure--patterns)
5. [State Management Optimization](#state-management-optimization)
6. [Rendering Optimization](#rendering-optimization)
7. [Memory Management](#memory-management)
8. [Bundle Optimization](#bundle-optimization)
9. [Monitoring & Profiling](#monitoring--profiling)

---

## Performance Best Practices

### 1. Component Optimization Fundamentals

```typescript
// ❌ BAD: Inline objects cause unnecessary re-renders
const BadComponent = ({ items }: { items: Item[] }) => {
  return (
    <ItemList 
      items={items}
      style={{ padding: '16px' }} // New object every render
      options={{ sortBy: 'date' }}  // New object every render
    />
  );
};

// ✅ GOOD: Define objects outside component or use useMemo
const listStyle = { padding: '16px' };
const defaultOptions = { sortBy: 'date' };

const GoodComponent = ({ items }: { items: Item[] }) => {
  return (
    <ItemList 
      items={items}
      style={listStyle}
      options={defaultOptions}
    />
  );
};

// ✅ BETTER: Use useMemo for dynamic values
const OptimizedComponent = ({ items, padding }: { items: Item[]; padding: number }) => {
  const style = useMemo(() => ({ padding: `${padding}px` }), [padding]);
  const options = useMemo(() => ({ sortBy: 'date' }), []);
  
  return (
    <ItemList 
      items={items}
      style={style}
      options={options}
    />
  );
};
```

### 2. Function Reference Stability

```typescript
// ❌ BAD: Inline functions cause child re-renders
const BadParent = ({ appointments }: { appointments: Appointment[] }) => {
  return (
    <div>
      {appointments.map(appointment => (
        <AppointmentCard
          key={appointment.id}
          appointment={appointment}
          onEdit={() => console.log('edit', appointment.id)} // New function every render
          onCancel={(id) => console.log('cancel', id)}       // New function every render
        />
      ))}
    </div>
  );
};

// ✅ GOOD: Use useCallback for stable function references
const GoodParent = ({ appointments }: { appointments: Appointment[] }) => {
  const handleEdit = useCallback((id: string) => {
    console.log('edit', id);
  }, []);
  
  const handleCancel = useCallback((id: string) => {
    console.log('cancel', id);
  }, []);
  
  return (
    <div>
      {appointments.map(appointment => (
        <AppointmentCard
          key={appointment.id}
          appointment={appointment}
          onEdit={handleEdit}
          onCancel={handleCancel}
        />
      ))}
    </div>
  );
};

// ✅ BETTER: Use a single handler with event delegation
const OptimizedParent = ({ appointments }: { appointments: Appointment[] }) => {
  const handleAction = useCallback((action: string, id: string) => {
    switch (action) {
      case 'edit':
        console.log('edit', id);
        break;
      case 'cancel':
        console.log('cancel', id);
        break;
    }
  }, []);
  
  return (
    <div>
      {appointments.map(appointment => (
        <AppointmentCard
          key={appointment.id}
          appointment={appointment}
          onAction={handleAction}
        />
      ))}
    </div>
  );
};
```

---

## Common Performance Pitfalls

### 1. Unnecessary Re-renders

```typescript
// ❌ PITFALL: Component re-renders on every parent render
const SlowAppointmentCard = ({ appointment, onEdit, onCancel }) => {
  console.log('AppointmentCard rendered'); // This will log on every parent update
  
  return (
    <div className="appointment-card">
      <h3>{appointment.patientName}</h3>
      <p>{appointment.date}</p>
      <button onClick={() => onEdit(appointment.id)}>Edit</button>
      <button onClick={() => onCancel(appointment.id)}>Cancel</button>
    </div>
  );
};

// ✅ SOLUTION: Use React.memo to prevent unnecessary re-renders
const OptimizedAppointmentCard = React.memo<AppointmentCardProps>(({ 
  appointment, 
  onEdit, 
  onCancel 
}) => {
  console.log('AppointmentCard rendered'); // Only logs when props actually change
  
  return (
    <div className="appointment-card">
      <h3>{appointment.patientName}</h3>
      <p>{appointment.date}</p>
      <button onClick={() => onEdit(appointment.id)}>Edit</button>
      <button onClick={() => onCancel(appointment.id)}>Cancel</button>
    </div>
  );
});

// ✅ ADVANCED: Custom comparison function for complex props
const AdvancedAppointmentCard = React.memo<AppointmentCardProps>(
  ({ appointment, onEdit, onCancel }) => {
    return (
      <div className="appointment-card">
        <h3>{appointment.patientName}</h3>
        <p>{appointment.date}</p>
        <button onClick={() => onEdit(appointment.id)}>Edit</button>
        <button onClick={() => onCancel(appointment.id)}>Cancel</button>
      </div>
    );
  },
  (prevProps, nextProps) => {
    // Custom comparison - only re-render if specific fields change
    return (
      prevProps.appointment.id === nextProps.appointment.id &&
      prevProps.appointment.patientName === nextProps.appointment.patientName &&
      prevProps.appointment.date === nextProps.appointment.date &&
      prevProps.onEdit === nextProps.onEdit &&
      prevProps.onCancel === nextProps.onCancel
    );
  }
);
```

### 2. Expensive Computations in Render

```typescript
// ❌ PITFALL: Expensive computation runs on every render
const BadPatientDashboard = ({ patients }: { patients: Patient[] }) => {
  // This expensive operation runs on every render!
  const patientStats = patients.reduce((stats, patient) => {
    stats.totalAge += calculateAge(patient.dateOfBirth);
    stats.byGender[patient.gender] = (stats.byGender[patient.gender] || 0) + 1;
    stats.byCity[patient.address?.city] = (stats.byCity[patient.address?.city] || 0) + 1;
    return stats;
  }, { totalAge: 0, byGender: {}, byCity: {} });
  
  return (
    <div>
      <h2>Patient Statistics</h2>
      <p>Average Age: {patientStats.totalAge / patients.length}</p>
      <div>Gender Distribution: {JSON.stringify(patientStats.byGender)}</div>
      <div>City Distribution: {JSON.stringify(patientStats.byCity)}</div>
    </div>
  );
};

// ✅ SOLUTION: Use useMemo for expensive computations
const OptimizedPatientDashboard = ({ patients }: { patients: Patient[] }) => {
  const patientStats = useMemo(() => {
    return patients.reduce((stats, patient) => {
      stats.totalAge += calculateAge(patient.dateOfBirth);
      stats.byGender[patient.gender] = (stats.byGender[patient.gender] || 0) + 1;
      stats.byCity[patient.address?.city] = (stats.byCity[patient.address?.city] || 0) + 1;
      return stats;
    }, { totalAge: 0, byGender: {}, byCity: {} });
  }, [patients]); // Only recalculate when patients array changes
  
  const averageAge = useMemo(() => 
    patientStats.totalAge / patients.length, 
    [patientStats.totalAge, patients.length]
  );
  
  return (
    <div>
      <h2>Patient Statistics</h2>
      <p>Average Age: {averageAge}</p>
      <div>Gender Distribution: {JSON.stringify(patientStats.byGender)}</div>
      <div>City Distribution: {JSON.stringify(patientStats.byCity)}</div>
    </div>
  );
};
```

### 3. State Update Batching Issues

```typescript
// ❌ PITFALL: Multiple state updates causing multiple re-renders
const BadAppointmentForm = () => {
  const [patientName, setPatientName] = useState('');
  const [appointmentDate, setAppointmentDate] = useState('');
  const [appointmentTime, setAppointmentTime] = useState('');
  const [notes, setNotes] = useState('');
  
  const handleFormSubmit = (data: FormData) => {
    // Each setState call triggers a re-render
    setPatientName(data.patientName);      // Re-render 1
    setAppointmentDate(data.appointmentDate); // Re-render 2
    setAppointmentTime(data.appointmentTime); // Re-render 3
    setNotes(data.notes);                  // Re-render 4
  };
  
  return (
    <form onSubmit={handleFormSubmit}>
      {/* Form fields */}
    </form>
  );
};

// ✅ SOLUTION: Use single state object or useReducer
const OptimizedAppointmentForm = () => {
  const [formData, setFormData] = useState({
    patientName: '',
    appointmentDate: '',
    appointmentTime: '',
    notes: '',
  });
  
  const handleFormSubmit = (data: FormData) => {
    // Single state update, single re-render
    setFormData({
      patientName: data.patientName,
      appointmentDate: data.appointmentDate,
      appointmentTime: data.appointmentTime,
      notes: data.notes,
    });
  };
  
  const updateField = useCallback((field: string, value: string) => {
    setFormData(prev => ({ ...prev, [field]: value }));
  }, []);
  
  return (
    <form onSubmit={handleFormSubmit}>
      {/* Form fields with updateField */}
    </form>
  );
};

// ✅ BETTER: Use React 18's automatic batching with flushSync for edge cases
import { flushSync } from 'react-dom';

const AdvancedForm = () => {
  const [formData, setFormData] = useState(initialFormData);
  const [isSubmitting, setIsSubmitting] = useState(false);
  
  const handleSubmit = async (data: FormData) => {
    // Urgent updates that need immediate rendering
    flushSync(() => {
      setIsSubmitting(true);
    });
    
    try {
      await submitAppointment(data);
      // These will be batched automatically in React 18
      setFormData(initialFormData);
      setIsSubmitting(false);
      showSuccessNotification();
    } catch (error) {
      setIsSubmitting(false);
      showErrorNotification(error.message);
    }
  };
  
  return (
    <form onSubmit={handleSubmit}>
      {/* Form implementation */}
    </form>
  );
};
```

---

## Optimization Techniques

### 1. React.memo Usage Patterns

```typescript
// Simple memo for basic props
const SimplePatientCard = React.memo<{ patient: Patient }>(({ patient }) => (
  <div className="patient-card">
    <h3>{patient.name}</h3>
    <p>{patient.email}</p>
  </div>
));

// Memo with custom comparison for complex objects
const ComplexPatientCard = React.memo<{
  patient: Patient;
  preferences: UserPreferences;
  onSelect: (id: string) => void;
}>(
  ({ patient, preferences, onSelect }) => (
    <div className="patient-card" style={{ theme: preferences.theme }}>
      <h3>{patient.name}</h3>
      <p>{patient.email}</p>
      <button onClick={() => onSelect(patient.id)}>Select</button>
    </div>
  ),
  (prevProps, nextProps) => {
    // Only compare the fields that actually affect rendering
    return (
      prevProps.patient.id === nextProps.patient.id &&
      prevProps.patient.name === nextProps.patient.name &&
      prevProps.patient.email === nextProps.patient.email &&
      prevProps.preferences.theme === nextProps.preferences.theme &&
      prevProps.onSelect === nextProps.onSelect
    );
  }
);

// Memo for expensive list items
const PatientListItem = React.memo<{
  patient: Patient;
  isSelected: boolean;
  onToggleSelect: (id: string) => void;
}>(({ patient, isSelected, onToggleSelect }) => {
  // Expensive computation that should only run when patient data changes
  const patientScore = useMemo(() => {
    return calculatePatientRiskScore(patient);
  }, [patient]);
  
  return (
    <div className={`patient-item ${isSelected ? 'selected' : ''}`}>
      <h4>{patient.name}</h4>
      <p>Risk Score: {patientScore}</p>
      <button onClick={() => onToggleSelect(patient.id)}>
        {isSelected ? 'Deselect' : 'Select'}
      </button>
    </div>
  );
});
```

### 2. useCallback Optimization

```typescript
// ✅ Proper useCallback usage
const PatientManager = ({ patients }: { patients: Patient[] }) => {
  const [selectedPatients, setSelectedPatients] = useState<string[]>([]);
  const [filters, setFilters] = useState<FilterState>({});
  
  // Stable callback that doesn't change unless dependencies change
  const handlePatientSelect = useCallback((patientId: string) => {
    setSelectedPatients(prev => 
      prev.includes(patientId)
        ? prev.filter(id => id !== patientId)
        : [...prev, patientId]
    );
  }, []); // No dependencies - function logic doesn't depend on external values
  
  // Callback with dependencies
  const handleBulkAction = useCallback((action: BulkAction) => {
    // This function uses selectedPatients, so it must be in dependencies
    switch (action) {
      case 'delete':
        deletePatients(selectedPatients);
        break;
      case 'export':
        exportPatients(selectedPatients);
        break;
    }
    setSelectedPatients([]); // Clear selection after action
  }, [selectedPatients]); // Recreate when selectedPatients changes
  
  // Callback for filtering - stable reference when filters don't change
  const handleFilterChange = useCallback((newFilters: FilterState) => {
    setFilters(newFilters);
    setSelectedPatients([]); // Clear selection when filters change
  }, []); // No dependencies needed as we're just setting state
  
  // Don't use useCallback for simple event handlers that don't cause performance issues
  const handleRefresh = () => {
    refreshPatientData();
  };
  
  return (
    <div>
      <PatientFilters onFilterChange={handleFilterChange} />
      <BulkActions 
        selectedCount={selectedPatients.length}
        onBulkAction={handleBulkAction}
      />
      <button onClick={handleRefresh}>Refresh</button>
      <PatientList 
        patients={patients}
        selectedPatients={selectedPatients}
        onPatientSelect={handlePatientSelect}
      />
    </div>
  );
};
```

### 3. useMemo for Complex Computations

```typescript
const AppointmentAnalytics = ({ appointments }: { appointments: Appointment[] }) => {
  // Heavy computation - memoize it
  const analytics = useMemo(() => {
    console.log('Computing analytics...'); // Should only log when appointments change
    
    const now = new Date();
    const thirtyDaysAgo = new Date(now.getTime() - 30 * 24 * 60 * 60 * 1000);
    
    return {
      total: appointments.length,
      completed: appointments.filter(apt => apt.status === 'completed').length,
      cancelled: appointments.filter(apt => apt.status === 'cancelled').length,
      upcoming: appointments.filter(apt => 
        apt.status === 'scheduled' && new Date(apt.date) > now
      ).length,
      recentTrend: appointments.filter(apt => 
        new Date(apt.date) > thirtyDaysAgo
      ).length,
      averageRating: appointments
        .filter(apt => apt.rating)
        .reduce((sum, apt) => sum + apt.rating!, 0) / 
        appointments.filter(apt => apt.rating).length || 0,
      providerBreakdown: appointments.reduce((breakdown, apt) => {
        breakdown[apt.providerId] = (breakdown[apt.providerId] || 0) + 1;
        return breakdown;
      }, {} as Record<string, number>),
    };
  }, [appointments]);
  
  // Derived computations from memoized analytics
  const completionRate = useMemo(() => 
    analytics.total > 0 ? (analytics.completed / analytics.total) * 100 : 0,
    [analytics.total, analytics.completed]
  );
  
  const cancellationRate = useMemo(() => 
    analytics.total > 0 ? (analytics.cancelled / analytics.total) * 100 : 0,
    [analytics.total, analytics.cancelled]
  );
  
  // Chart data computation
  const chartData = useMemo(() => {
    return Object.entries(analytics.providerBreakdown).map(([providerId, count]) => ({
      providerId,
      count,
      percentage: (count / analytics.total) * 100,
    }));
  }, [analytics.providerBreakdown, analytics.total]);
  
  return (
    <div className="analytics-dashboard">
      <div className="metrics-grid">
        <MetricCard title="Total Appointments" value={analytics.total} />
        <MetricCard title="Completion Rate" value={`${completionRate.toFixed(1)}%`} />
        <MetricCard title="Cancellation Rate" value={`${cancellationRate.toFixed(1)}%`} />
        <MetricCard title="Average Rating" value={analytics.averageRating.toFixed(1)} />
      </div>
      
      <div className="charts-section">
        <ProviderBreakdownChart data={chartData} />
        <TrendChart appointments={appointments} />
      </div>
    </div>
  );
};
```

---

## Component Structure & Patterns

### 1. Compound Components Pattern

```typescript
// ✅ Compound component for flexible composition
const AppointmentCard = ({ children, appointment }: {
  children: React.ReactNode;
  appointment: Appointment;
}) => {
  return (
    <div className="appointment-card" data-appointment-id={appointment.id}>
      {children}
    </div>
  );
};

const AppointmentCardHeader = ({ children }: { children: React.ReactNode }) => (
  <div className="appointment-card-header">{children}</div>
);

const AppointmentCardBody = ({ children }: { children: React.ReactNode }) => (
  <div className="appointment-card-body">{children}</div>
);

const AppointmentCardActions = ({ children }: { children: React.ReactNode }) => (
  <div className="appointment-card-actions">{children}</div>
);

// Attach subcomponents
AppointmentCard.Header = AppointmentCardHeader;
AppointmentCard.Body = AppointmentCardBody;
AppointmentCard.Actions = AppointmentCardActions;

// Usage - very flexible and composable
const AppointmentList = ({ appointments }: { appointments: Appointment[] }) => (
  <div>
    {appointments.map(appointment => (
      <AppointmentCard key={appointment.id} appointment={appointment}>
        <AppointmentCard.Header>
          <h3>{appointment.patientName}</h3>
          <span className="appointment-type">{appointment.type}</span>
        </AppointmentCard.Header>
        
        <AppointmentCard.Body>
          <p>Date: {formatDate(appointment.date)}</p>
          <p>Provider: {appointment.providerName}</p>
          {appointment.notes && <p>Notes: {appointment.notes}</p>}
        </AppointmentCard.Body>
        
        <AppointmentCard.Actions>
          <button>Edit</button>
          <button>Cancel</button>
          <button>Reschedule</button>
        </AppointmentCard.Actions>
      </AppointmentCard>
    ))}
  </div>
);
```

### 2. Render Props Pattern for Reusability

```typescript
// ✅ Data fetching component with render props
interface DataFetcherProps<T> {
  url: string;
  children: (data: {
    data: T | null;
    loading: boolean;
    error: string | null;
    refetch: () => void;
  }) => React.ReactNode;
}

const DataFetcher = <T,>({ url, children }: DataFetcherProps<T>) => {
  const [data, setData] = useState<T | null>(null);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState<string | null>(null);
  
  const fetchData = useCallback(async () => {
    setLoading(true);
    setError(null);
    
    try {
      const response = await fetch(url);
      if (!response.ok) throw new Error(`HTTP ${response.status}`);
      const result = await response.json();
      setData(result);
    } catch (err) {
      setError(err instanceof Error ? err.message : 'An error occurred');
    } finally {
      setLoading(false);
    }
  }, [url]);
  
  useEffect(() => {
    fetchData();
  }, [fetchData]);
  
  return children({ data, loading, error, refetch: fetchData });
};

// Usage - very reusable across different data types
const PatientsList = () => (
  <DataFetcher<Patient[]> url="/api/patients">
    {({ data: patients, loading, error, refetch }) => {
      if (loading) return <PatientsSkeleton />;
      if (error) return <ErrorMessage message={error} onRetry={refetch} />;
      if (!patients) return <EmptyState />;
      
      return (
        <div>
          <button onClick={refetch}>Refresh</button>
          {patients.map(patient => (
            <PatientCard key={patient.id} patient={patient} />
          ))}
        </div>
      );
    }}
  </DataFetcher>
);

const AppointmentsList = () => (
  <DataFetcher<Appointment[]> url="/api/appointments">
    {({ data: appointments, loading, error, refetch }) => {
      if (loading) return <AppointmentsSkeleton />;
      if (error) return <ErrorMessage message={error} onRetry={refetch} />;
      if (!appointments) return <EmptyState message="No appointments found" />;
      
      return (
        <div>
          <AppointmentFilters onFilter={() => refetch()} />
          {appointments.map(appointment => (
            <AppointmentCard key={appointment.id} appointment={appointment} />
          ))}
        </div>
      );
    }}
  </DataFetcher>
);
```

### 3. Higher-Order Components for Cross-Cutting Concerns

```typescript
// ✅ HOC for performance monitoring
function withPerformanceMonitoring<P extends object>(
  WrappedComponent: React.ComponentType<P>,
  componentName: string
) {
  const WithPerformanceMonitoring = React.forwardRef<any, P>((props, ref) => {
    const renderStartTime = useRef<number>();
    const [renderCount, setRenderCount] = useState(0);
    
    // Track render start
    renderStartTime.current = performance.now();
    
    useEffect(() => {
      setRenderCount(prev => prev + 1);
      
      // Track render end
      const renderEndTime = performance.now();
      const renderDuration = renderEndTime - (renderStartTime.current || 0);
      
      // Log slow renders
      if (renderDuration > 16) { // Slower than 60fps
        console.warn(`Slow render in ${componentName}: ${renderDuration.toFixed(2)}ms`);
      }
      
      // Send metrics to monitoring service
      if (renderCount % 10 === 0) { // Every 10th render
        sendMetrics({
          component: componentName,
          renderCount,
          lastRenderDuration: renderDuration,
        });
      }
    });
    
    return <WrappedComponent {...props} ref={ref} />;
  });
  
  WithPerformanceMonitoring.displayName = `withPerformanceMonitoring(${componentName})`;
  return WithPerformanceMonitoring;
}

// Usage
const MonitoredPatientDashboard = withPerformanceMonitoring(
  PatientDashboard, 
  'PatientDashboard'
);

// ✅ HOC for error boundaries
function withErrorBoundary<P extends object>(
  WrappedComponent: React.ComponentType<P>,
  fallbackComponent?: React.ComponentType<{ error: Error; resetError: () => void }>
) {
  return class WithErrorBoundary extends React.Component<
    P,
    { hasError: boolean; error: Error | null }
  > {
    constructor(props: P) {
      super(props);
      this.state = { hasError: false, error: null };
    }
    
    static getDerivedStateFromError(error: Error) {
      return { hasError: true, error };
    }
    
    componentDidCatch(error: Error, errorInfo: React.ErrorInfo) {
      console.error('Component error:', error, errorInfo);
      // Send to error tracking service
      sendErrorToService(error, errorInfo);
    }
    
    resetError = () => {
      this.setState({ hasError: false, error: null });
    };
    
    render() {
      if (this.state.hasError && this.state.error) {
        if (fallbackComponent) {
          const FallbackComponent = fallbackComponent;
          return <FallbackComponent error={this.state.error} resetError={this.resetError} />;
        }
        
        return (
          <div className="error-boundary">
            <h2>Something went wrong</h2>
            <p>{this.state.error.message}</p>
            <button onClick={this.resetError}>Try Again</button>
          </div>
        );
      }
      
      return <WrappedComponent {...this.props} />;
    }
  };
}

// Usage
const SafePatientDashboard = withErrorBoundary(PatientDashboard, CustomErrorFallback);
```

---

## State Management Optimization

### 1. Context Optimization

```typescript
// ❌ BAD: Single large context causes unnecessary re-renders
const BadAppContext = createContext<{
  user: User | null;
  patients: Patient[];
  appointments: Appointment[];
  notifications: Notification[];
  theme: Theme;
  updateUser: (user: User) => void;
  addPatient: (patient: Patient) => void;
  updateAppointment: (id: string, updates: Partial<Appointment>) => void;
  dismissNotification: (id: string) => void;
  setTheme: (theme: Theme) => void;
}>({} as any);

// ✅ GOOD: Split contexts by concern and update frequency
const UserContext = createContext<{
  user: User | null;
  updateUser: (user: User) => void;
}>({} as any);

const PatientsContext = createContext<{
  patients: Patient[];
  addPatient: (patient: Patient) => void;
  updatePatient: (id: string, updates: Partial<Patient>) => void;
  removePatient: (id: string) => void;
}>({} as any);

const AppointmentsContext = createContext<{
  appointments: Appointment[];
  addAppointment: (appointment: Appointment) => void;
  updateAppointment: (id: string, updates: Partial<Appointment>) => void;
  removeAppointment: (id: string) => void;
}>({} as any);

const ThemeContext = createContext<{
  theme: Theme;
  setTheme: (theme: Theme) => void;
}>({} as any);

// ✅ BETTER: Use context selectors to prevent unnecessary re-renders
interface AppState {
  user: User | null;
  patients: Patient[];
  appointments: Appointment[];
  theme: Theme;
}

interface AppContextValue {
  state: AppState;
  dispatch: React.Dispatch<AppAction>;
}

const AppContext = createContext<AppContextValue>({} as any);

// Selector hook to prevent unnecessary re-renders
function useAppSelector<T>(selector: (state: AppState) => T): T {
  const { state } = useContext(AppContext);
  return useMemo(() => selector(state), [state, selector]);
}

// Stable selector functions
const selectUser = (state: AppState) => state.user;
const selectPatients = (state: AppState) => state.patients;
const selectAppointments = (state: AppState) => state.appointments;
const selectTheme = (state: AppState) => state.theme;

// Usage - only re-renders when selected data changes
const UserProfile = () => {
  const user = useAppSelector(selectUser); // Only re-renders when user changes
  const theme = useAppSelector(selectTheme); // Only re-renders when theme changes
  
  return (
    <div className={`user-profile theme-${theme}`}>
      <h2>{user?.name}</h2>
      <p>{user?.email}</p>
    </div>
  );
};
```

### 2. Reducer Optimization

```typescript
// ✅ Optimized reducer with immutable updates
interface AppState {
  patients: Record<string, Patient>;
  appointments: Record<string, Appointment>;
  filters: {
    patientFilters: PatientFilters;
    appointmentFilters: AppointmentFilters;
  };
  ui: {
    selectedPatients: string[];
    selectedAppointments: string[];
    modals: {
      createPatient: boolean;
      editAppointment: boolean;
    };
  };
}

type AppAction =
  | { type: 'ADD_PATIENT'; payload: Patient }
  | { type: 'UPDATE_PATIENT'; payload: { id: string; updates: Partial<Patient> } }
  | { type: 'REMOVE_PATIENT'; payload: string }
  | { type: 'SET_PATIENT_FILTERS'; payload: PatientFilters }
  | { type: 'TOGGLE_PATIENT_SELECTION'; payload: string }
  | { type: 'CLEAR_PATIENT_SELECTION' }
  | { type: 'BULK_UPDATE_PATIENTS'; payload: { ids: string[]; updates: Partial<Patient> } };

const appReducer = (state: AppState, action: AppAction): AppState => {
  switch (action.type) {
    case 'ADD_PATIENT':
      return {
        ...state,
        patients: {
          ...state.patients,
          [action.payload.id]: action.payload,
        },
      };
      
    case 'UPDATE_PATIENT':
      return {
        ...state,
        patients: {
          ...state.patients,
          [action.payload.id]: {
            ...state.patients[action.payload.id],
            ...action.payload.updates,
            updatedAt: new Date().toISOString(),
          },
        },
      };
      
    case 'REMOVE_PATIENT':
      const { [action.payload]: removed, ...remainingPatients } = state.patients;
      return {
        ...state,
        patients: remainingPatients,
        ui: {
          ...state.ui,
          selectedPatients: state.ui.selectedPatients.filter(id => id !== action.payload),
        },
      };
      
    case 'BULK_UPDATE_PATIENTS':
      return {
        ...state,
        patients: {
          ...state.patients,
          ...action.payload.ids.reduce((acc, id) => {
            if (state.patients[id]) {
              acc[id] = {
                ...state.patients[id],
                ...action.payload.updates,
                updatedAt: new Date().toISOString(),
              };
            }
            return acc;
          }, {} as Record<string, Patient>),
        },
      };
      
    case 'TOGGLE_PATIENT_SELECTION':
      return {
        ...state,
        ui: {
          ...state.ui,
          selectedPatients: state.ui.selectedPatients.includes(action.payload)
            ? state.ui.selectedPatients.filter(id => id !== action.payload)
            : [...state.ui.selectedPatients, action.payload],
        },
      };
      
    case 'CLEAR_PATIENT_SELECTION':
      return {
        ...state,
        ui: {
          ...state.ui,
          selectedPatients: [],
        },
      };
      
    default:
      return state;
  }
};

// Optimized action creators with useCallback
const useAppActions = () => {
  const { dispatch } = useContext(AppContext);
  
  return useMemo(() => ({
    addPatient: (patient: Patient) => 
      dispatch({ type: 'ADD_PATIENT', payload: patient }),
      
    updatePatient: (id: string, updates: Partial<Patient>) =>
      dispatch({ type: 'UPDATE_PATIENT', payload: { id, updates } }),
      
    removePatient: (id: string) =>
      dispatch({ type: 'REMOVE_PATIENT', payload: id }),
      
    togglePatientSelection: (id: string) =>
      dispatch({ type: 'TOGGLE_PATIENT_SELECTION', payload: id }),
      
    clearPatientSelection: () =>
      dispatch({ type: 'CLEAR_PATIENT_SELECTION' }),
      
    bulkUpdatePatients: (ids: string[], updates: Partial<Patient>) =>
      dispatch({ type: 'BULK_UPDATE_PATIENTS', payload: { ids, updates } }),
  }), [dispatch]);
};
```

---

## Rendering Optimization

### 1. Virtual Lists for Large Datasets

```typescript
import { FixedSizeList as List } from 'react-window';

// ✅ Virtualized list for thousands of patients
const VirtualizedPatientList = ({ patients }: { patients: Patient[] }) => {
  const Row = useCallback(({ index, style }: { index: number; style: React.CSSProperties }) => {
    const patient = patients[index];
    
    return (
      <div style={style}>
        <PatientCard patient={patient} />
      </div>
    );
  }, [patients]);
  
  return (
    <List
      height={600}
      itemCount={patients.length}
      itemSize={120}
      overscanCount={5} // Render 5 extra items outside viewport
    >
      {Row}
    </List>
  );
};

// ✅ Variable size list for different item heights
import { VariableSizeList as VList } from 'react-window';

const VariableSizeAppointmentList = ({ appointments }: { appointments: Appointment[] }) => {
  const getItemSize = useCallback((index: number) => {
    const appointment = appointments[index];
    // Calculate height based on content
    const baseHeight = 80;
    const notesHeight = appointment.notes ? 40 : 0;
    const prescriptionHeight = appointment.prescription ? 60 : 0;
    return baseHeight + notesHeight + prescriptionHeight;
  }, [appointments]);
  
  const Row = useCallback(({ index, style }: { index: number; style: React.CSSProperties }) => {
    const appointment = appointments[index];
    
    return (
      <div style={style}>
        <AppointmentCard appointment={appointment} />
      </div>
    );
  }, [appointments]);
  
  return (
    <VList
      height={600}
      itemCount={appointments.length}
      itemSize={getItemSize}
      overscanCount={3}
    >
      {Row}
    </VList>
  );
};
```

### 2. Lazy Loading and Code Splitting

```typescript
// ✅ Component-level lazy loading
const LazyPatientDetailsModal = lazy(() => 
  import('./PatientDetailsModal').then(module => ({
    default: module.PatientDetailsModal
  }))
);

const LazyAppointmentCalendar = lazy(() => 
  import('./AppointmentCalendar').then(module => ({
    default: module.AppointmentCalendar
  }))
);

const PatientDashboard = () => {
  const [showPatientDetails, setShowPatientDetails] = useState(false);
  const [showCalendar, setShowCalendar] = useState(false);
  
  return (
    <div>
      <PatientList onPatientSelect={() => setShowPatientDetails(true)} />
      
      <button onClick={() => setShowCalendar(true)}>
        View Calendar
      </button>
      
      {showPatientDetails && (
        <Suspense fallback={<ModalSkeleton />}>
          <LazyPatientDetailsModal
            isOpen={showPatientDetails}
            onClose={() => setShowPatientDetails(false)}
          />
        </Suspense>
      )}
      
      {showCalendar && (
        <Suspense fallback={<CalendarSkeleton />}>
          <LazyAppointmentCalendar />
        </Suspense>
      )}
    </div>
  );
};

// ✅ Route-based code splitting with preloading
const AdminDashboard = lazy(() => 
  import('../pages/AdminDashboard').then(module => {
    // Preload related components
    import('../components/UserManagement');
    import('../components/SystemSettings');
    return { default: module.AdminDashboard };
  })
);

// ✅ Conditional loading based on user role
const ConditionalAdminPanel = ({ userRole }: { userRole: string }) => {
  const AdminPanel = useMemo(() => {
    if (userRole === 'admin') {
      return lazy(() => import('../components/AdminPanel'));
    }
    return null;
  }, [userRole]);
  
  if (userRole !== 'admin' || !AdminPanel) {
    return null;
  }
  
  return (
    <Suspense fallback={<AdminPanelSkeleton />}>
      <AdminPanel />
    </Suspense>
  );
};
```

### 3. Image Optimization and Lazy Loading

```typescript
// ✅ Optimized image component with lazy loading
const OptimizedImage = ({ 
  src, 
  alt, 
  width, 
  height, 
  priority = false,
  className = '',
}: {
  src: string;
  alt: string;
  width: number;
  height: number;
  priority?: boolean;
  className?: string;
}) => {
  const [isLoaded, setIsLoaded] = useState(false);
  const [isInView, setIsInView] = useState(priority);
  const imgRef = useRef<HTMLImageElement>(null);
  
  // Intersection observer for lazy loading
  useEffect(() => {
    if (priority) return;
    
    const observer = new IntersectionObserver(
      ([entry]) => {
        if (entry.isIntersecting) {
          setIsInView(true);
          observer.disconnect();
        }
      },
      { threshold: 0.1 }
    );
    
    if (imgRef.current) {
      observer.observe(imgRef.current);
    }
    
    return () => observer.disconnect();
  }, [priority]);
  
  // Generate responsive URLs
  const generateSrcSet = (baseSrc: string) => {
    const sizes = [320, 640, 960, 1280];
    return sizes
      .map(size => `${baseSrc}?w=${size}&q=75 ${size}w`)
      .join(', ');
  };
  
  return (
    <div 
      className={`image-container ${className}`}
      style={{ width, height, backgroundColor: '#f0f0f0' }}
    >
      {isInView && (
        <img
          ref={imgRef}
          src={src}
          srcSet={generateSrcSet(src)}
          sizes="(max-width: 768px) 100vw, (max-width: 1200px) 50vw, 33vw"
          alt={alt}
          loading={priority ? 'eager' : 'lazy'}
          decoding="async"
          onLoad={() => setIsLoaded(true)}
          style={{
            width: '100%',
            height: '100%',
            objectFit: 'cover',
            opacity: isLoaded ? 1 : 0,
            transition: 'opacity 0.3s ease-in-out',
          }}
        />
      )}
      
      {!isLoaded && isInView && (
        <div className="image-skeleton">
          <div className="skeleton-animation" />
        </div>
      )}
    </div>
  );
};

// ✅ Avatar component with fallback and caching
const UserAvatar = ({ 
  user, 
  size = 40 
}: { 
  user: { name: string; avatarUrl?: string }; 
  size?: number; 
}) => {
  const [imageError, setImageError] = useState(false);
  
  const fallbackContent = useMemo(() => {
    const initials = user.name
      .split(' ')
      .map(part => part[0])
      .join('')
      .toUpperCase()
      .slice(0, 2);
      
    return (
      <div 
        className="avatar-fallback"
        style={{
          width: size,
          height: size,
          borderRadius: '50%',
          backgroundColor: `hsl(${user.name.charCodeAt(0) * 137.5 % 360}, 50%, 50%)`,
          display: 'flex',
          alignItems: 'center',
          justifyContent: 'center',
          color: 'white',
          fontWeight: 'bold',
          fontSize: size * 0.4,
        }}
      >
        {initials}
      </div>
    );
  }, [user.name, size]);
  
  if (!user.avatarUrl || imageError) {
    return fallbackContent;
  }
  
  return (
    <OptimizedImage
      src={user.avatarUrl}
      alt={`${user.name}'s avatar`}
      width={size}
      height={size}
      className="user-avatar"
      onError={() => setImageError(true)}
    />
  );
};
```

---

## Memory Management

### 1. Preventing Memory Leaks

```typescript
// ✅ Proper cleanup of subscriptions and timers
const AppointmentNotifications = ({ userId }: { userId: string }) => {
  const [notifications, setNotifications] = useState<Notification[]>([]);
  
  useEffect(() => {
    // WebSocket connection
    const ws = new WebSocket(`ws://localhost:8080/notifications/${userId}`);
    
    ws.onmessage = (event) => {
      const notification = JSON.parse(event.data);
      setNotifications(prev => [...prev, notification]);
    };
    
    // Cleanup WebSocket connection
    return () => {
      ws.close();
    };
  }, [userId]);
  
  useEffect(() => {
    // Auto-clear old notifications
    const interval = setInterval(() => {
      setNotifications(prev => 
        prev.filter(notification => 
          Date.now() - new Date(notification.createdAt).getTime() < 30000 // 30 seconds
        )
      );
    }, 5000);
    
    // Cleanup interval
    return () => clearInterval(interval);
  }, []);
  
  // Event listener cleanup
  useEffect(() => {
    const handleVisibilityChange = () => {
      if (document.hidden) {
        // Pause notifications when tab is hidden
        console.log('Pausing notifications');
      } else {
        // Resume notifications when tab becomes visible
        console.log('Resuming notifications');
      }
    };
    
    document.addEventListener('visibilitychange', handleVisibilityChange);
    
    return () => {
      document.removeEventListener('visibilitychange', handleVisibilityChange);
    };
  }, []);
  
  return (
    <div className="notifications">
      {notifications.map(notification => (
        <NotificationItem
          key={notification.id}
          notification={notification}
          onDismiss={(id) => 
            setNotifications(prev => prev.filter(n => n.id !== id))
          }
        />
      ))}
    </div>
  );
};

// ✅ Debounced search to prevent excessive API calls
const useDebounced = <T>(value: T, delay: number): T => {
  const [debouncedValue, setDebouncedValue] = useState<T>(value);
  
  useEffect(() => {
    const timer = setTimeout(() => {
      setDebouncedValue(value);
    }, delay);
    
    return () => clearTimeout(timer);
  }, [value, delay]);
  
  return debouncedValue;
};

const PatientSearch = () => {
  const [searchTerm, setSearchTerm] = useState('');
  const [searchResults, setSearchResults] = useState<Patient[]>([]);
  const [isSearching, setIsSearching] = useState(false);
  
  const debouncedSearchTerm = useDebounced(searchTerm, 300);
  
  useEffect(() => {
    if (!debouncedSearchTerm.trim()) {
      setSearchResults([]);
      return;
    }
    
    let cancelled = false;
    setIsSearching(true);
    
    const searchPatients = async () => {
      try {
        const results = await fetch(
          `/api/patients/search?q=${encodeURIComponent(debouncedSearchTerm)}`
        ).then(res => res.json());
        
        if (!cancelled) {
          setSearchResults(results);
          setIsSearching(false);
        }
      } catch (error) {
        if (!cancelled) {
          console.error('Search failed:', error);
          setIsSearching(false);
        }
      }
    };
    
    searchPatients();
    
    return () => {
      cancelled = true;
    };
  }, [debouncedSearchTerm]);
  
  return (
    <div>
      <input
        type="text"
        value={searchTerm}
        onChange={(e) => setSearchTerm(e.target.value)}
        placeholder="Search patients..."
      />
      
      {isSearching && <div>Searching...</div>}
      
      {searchResults.map(patient => (
        <PatientCard key={patient.id} patient={patient} />
      ))}
    </div>
  );
};
```

### 2. Efficient Data Structures

```typescript
// ✅ Use Maps for O(1) lookups instead of array.find()
const PatientManager = ({ patients }: { patients: Patient[] }) => {
  // Convert array to Map for efficient lookups
  const patientMap = useMemo(() => {
    return new Map(patients.map(patient => [patient.id, patient]));
  }, [patients]);
  
  // Indexed by different fields for different lookup needs
  const patientsByEmail = useMemo(() => {
    return new Map(patients.map(patient => [patient.email, patient]));
  }, [patients]);
  
  const getPatientById = useCallback((id: string) => {
    return patientMap.get(id); // O(1) instead of O(n)
  }, [patientMap]);
  
  const getPatientByEmail = useCallback((email: string) => {
    return patientsByEmail.get(email); // O(1) instead of O(n)
  }, [patientsByEmail]);
  
  return (
    <div>
      {/* Component implementation */}
    </div>
  );
};

// ✅ Normalized state structure for complex relationships
interface NormalizedState {
  patients: {
    byId: Record<string, Patient>;
    allIds: string[];
  };
  appointments: {
    byId: Record<string, Appointment>;
    allIds: string[];
    byPatientId: Record<string, string[]>;
    byProviderId: Record<string, string[]>;
  };
  providers: {
    byId: Record<string, Provider>;
    allIds: string[];
  };
}

// Selectors for normalized state
const selectAllPatients = (state: NormalizedState) => 
  state.patients.allIds.map(id => state.patients.byId[id]);

const selectPatientById = (state: NormalizedState, patientId: string) =>
  state.patients.byId[patientId];

const selectAppointmentsByPatientId = (state: NormalizedState, patientId: string) =>
  (state.appointments.byPatientId[patientId] || [])
    .map(id => state.appointments.byId[id]);

// ✅ Memoized selectors with reselect-like functionality
const createMemoizedSelector = <Args extends any[], Return>(
  selector: (...args: Args) => Return
) => {
  const cache = new Map();
  
  return (...args: Args): Return => {
    const key = JSON.stringify(args);
    
    if (cache.has(key)) {
      return cache.get(key);
    }
    
    const result = selector(...args);
    cache.set(key, result);
    
    // Clean up old cache entries to prevent memory leaks
    if (cache.size > 100) {
      const firstKey = cache.keys().next().value;
      cache.delete(firstKey);
    }
    
    return result;
  };
};

const selectFilteredPatients = createMemoizedSelector(
  (patients: Patient[], filters: PatientFilters) => {
    return patients.filter(patient => {
      if (filters.search && !patient.name.toLowerCase().includes(filters.search.toLowerCase())) {
        return false;
      }
      if (filters.city && patient.address?.city !== filters.city) {
        return false;
      }
      if (filters.ageRange) {
        const age = calculateAge(patient.dateOfBirth);
        if (age < filters.ageRange.min || age > filters.ageRange.max) {
          return false;
        }
      }
      return true;
    });
  }
);
```

---

## Bundle Optimization

### 1. Tree Shaking and Dead Code Elimination

```typescript
// ✅ Import only what you need
import { format } from 'date-fns/format';
import { addDays } from 'date-fns/addDays';
import { isToday } from 'date-fns/isToday';

// ❌ Don't import entire libraries
// import * as dateFns from 'date-fns'; // Imports everything

// ✅ Use dynamic imports for conditional features
const loadChartsLibrary = async () => {
  const { Chart } = await import('chart.js/auto');
  return Chart;
};

const AppointmentChart = ({ data }: { data: ChartData }) => {
  const [Chart, setChart] = useState<any>(null);
  
  useEffect(() => {
    loadChartsLibrary().then(setChart);
  }, []);
  
  if (!Chart) {
    return <ChartSkeleton />;
  }
  
  return <Chart data={data} />;
};

// ✅ Conditional polyfills
const loadPolyfills = async () => {
  if (!('IntersectionObserver' in window)) {
    await import('intersection-observer');
  }
  
  if (!('ResizeObserver' in window)) {
    await import('resize-observer-polyfill');
  }
};

// ✅ Feature-based code splitting
const loadAdvancedFeatures = async () => {
  if (userHasAdvancedFeatures) {
    const [
      { AdvancedAnalytics },
      { ReportGenerator },
      { BulkOperations }
    ] = await Promise.all([
      import('../components/AdvancedAnalytics'),
      import('../components/ReportGenerator'),
      import('../components/BulkOperations'),
    ]);
    
    return { AdvancedAnalytics, ReportGenerator, BulkOperations };
  }
  
  return null;
};
```

### 2. Webpack Bundle Analysis

```javascript
// webpack.config.js optimizations
const path = require('path');
const BundleAnalyzerPlugin = require('webpack-bundle-analyzer').BundleAnalyzerPlugin;

module.exports = {
  // ... other config
  
  optimization: {
    splitChunks: {
      chunks: 'all',
      cacheGroups: {
        // Vendor chunk for third-party libraries
        vendor: {
          test: /[\\/]node_modules[\\/]/,
          name: 'vendors',
          chunks: 'all',
          enforce: true,
        },
        
        // Common chunk for shared components
        common: {
          name: 'common',
          minChunks: 2,
          chunks: 'all',
          enforce: true,
        },
        
        // Separate chunk for large libraries
        charts: {
          test: /[\\/]node_modules[\\/](chart\.js|recharts|d3)[\\/]/,
          name: 'charts',
          chunks: 'all',
        },
        
        // Date utilities chunk
        dateUtils: {
          test: /[\\/]node_modules[\\/](date-fns|moment|dayjs)[\\/]/,
          name: 'date-utils',
          chunks: 'all',
        },
      },
    },
    
    // Enable tree shaking
    usedExports: true,
    sideEffects: false,
    
    // Minimize bundle
    minimize: true,
  },
  
  plugins: [
    // Analyze bundle size
    new BundleAnalyzerPlugin({
      analyzerMode: process.env.ANALYZE ? 'server' : 'disabled',
    }),
  ],
  
  resolve: {
    // Tree shaking for lodash
    alias: {
      'lodash': 'lodash-es',
    },
  },
};
```

---

## Monitoring & Profiling

### 1. Performance Monitoring

```typescript
// ✅ Performance monitoring hooks
const usePerformanceMonitor = (componentName: string) => {
  const renderStartTime = useRef<number>();
  const [metrics, setMetrics] = useState<{
    renderCount: number;
    averageRenderTime: number;
    slowRenders: number;
  }>({
    renderCount: 0,
    averageRenderTime: 0,
    slowRenders: 0,
  });
  
  // Mark render start
  renderStartTime.current = performance.now();
  
  useEffect(() => {
    const renderEndTime = performance.now();
    const renderDuration = renderEndTime - (renderStartTime.current || 0);
    
    setMetrics(prev => {
      const newRenderCount = prev.renderCount + 1;
      const newAverageRenderTime = 
        (prev.averageRenderTime * prev.renderCount + renderDuration) / newRenderCount;
      const newSlowRenders = renderDuration > 16 ? prev.slowRenders + 1 : prev.slowRenders;
      
      return {
        renderCount: newRenderCount,
        averageRenderTime: newAverageRenderTime,
        slowRenders: newSlowRenders,
      };
    });
    
    // Log performance warnings
    if (renderDuration > 16) {
      console.warn(
        `Slow render in ${componentName}: ${renderDuration.toFixed(2)}ms`
      );
    }
    
    // Send metrics to monitoring service every 50 renders
    if (metrics.renderCount % 50 === 0) {
      sendPerformanceMetrics({
        component: componentName,
        ...metrics,
        lastRenderDuration: renderDuration,
      });
    }
  });
  
  return metrics;
};

// Usage
const PatientDashboard = () => {
  const metrics = usePerformanceMonitor('PatientDashboard');
  
  return (
    <div>
      {/* Component content */}
      {process.env.NODE_ENV === 'development' && (
        <div className="performance-overlay">
          <small>
            Renders: {metrics.renderCount} | 
            Avg: {metrics.averageRenderTime.toFixed(2)}ms |
            Slow: {metrics.slowRenders}
          </small>
        </div>
      )}
    </div>
  );
};

// ✅ React DevTools Profiler integration
const ProfiledComponent = ({ children, id }: { children: React.ReactNode; id: string }) => {
  if (process.env.NODE_ENV === 'development') {
    return (
      <Profiler
        id={id}
        onRender={(id, phase, actualDuration, baseDuration, startTime, commitTime) => {
          console.log('Profiler:', {
            id,
            phase,
            actualDuration,
            baseDuration,
            startTime,
            commitTime,
          });
          
          // Send to analytics in production
          if (actualDuration > 10) {
            sendProfilerData({
              componentId: id,
              phase,
              duration: actualDuration,
              timestamp: commitTime,
            });
          }
        }}
      >
        {children}
      </Profiler>
    );
  }
  
  return <>{children}</>;
};

// ✅ Custom performance observer
const usePerformanceObserver = () => {
  useEffect(() => {
    if ('PerformanceObserver' in window) {
      const observer = new PerformanceObserver((list) => {
        const entries = list.getEntries();
        
        entries.forEach((entry) => {
          // Monitor long tasks
          if (entry.entryType === 'longtask') {
            console.warn('Long task detected:', entry.duration);
            sendPerformanceAlert({
              type: 'longtask',
              duration: entry.duration,
              startTime: entry.startTime,
            });
          }
          
          // Monitor navigation timing
          if (entry.entryType === 'navigation') {
            const navEntry = entry as PerformanceNavigationTiming;
            console.log('Navigation timing:', {
              domContentLoaded: navEntry.domContentLoadedEventEnd - navEntry.navigationStart,
              loadComplete: navEntry.loadEventEnd - navEntry.navigationStart,
              firstContentfulPaint: navEntry.responseEnd - navEntry.navigationStart,
            });
          }
        });
      });
      
      observer.observe({ entryTypes: ['longtask', 'navigation', 'paint'] });
      
      return () => observer.disconnect();
    }
  }, []);
};
```

### 2. Error Boundaries with Performance Tracking

```typescript
// ✅ Enhanced error boundary with performance monitoring
interface ErrorBoundaryState {
  hasError: boolean;
  error: Error | null;
  errorInfo: React.ErrorInfo | null;
  errorId: string;
}

class PerformantErrorBoundary extends React.Component<
  { children: React.ReactNode; fallback?: React.ComponentType<any> },
  ErrorBoundaryState
> {
  private performanceMarker: string;
  
  constructor(props: { children: React.ReactNode }) {
    super(props);
    this.state = {
      hasError: false,
      error: null,
      errorInfo: null,
      errorId: '',
    };
    this.performanceMarker = `error-boundary-${Date.now()}`;
  }
  
  static getDerivedStateFromError(error: Error): Partial<ErrorBoundaryState> {
    const errorId = `error-${Date.now()}-${Math.random().toString(36).substr(2, 9)}`;
    
    return {
      hasError: true,
      error,
      errorId,
    };
  }
  
  componentDidCatch(error: Error, errorInfo: React.ErrorInfo) {
    // Performance mark for error tracking
    performance.mark(`${this.performanceMarker}-error-start`);
    
    this.setState({ errorInfo });
    
    // Enhanced error reporting
    const errorReport = {
      errorId: this.state.errorId,
      message: error.message,
      stack: error.stack,
      componentStack: errorInfo.componentStack,
      userAgent: navigator.userAgent,
      url: window.location.href,
      timestamp: new Date().toISOString(),
      userId: getCurrentUserId(),
      buildVersion: process.env.REACT_APP_VERSION,
    };
    
    // Send to error tracking service
    sendErrorReport(errorReport);
    
    // Performance monitoring
    performance.mark(`${this.performanceMarker}-error-end`);
    performance.measure(
      `error-boundary-processing`,
      `${this.performanceMarker}-error-start`,
      `${this.performanceMarker}-error-end`
