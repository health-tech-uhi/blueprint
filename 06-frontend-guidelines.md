# Section 6: Frontend Guidelines

## Table of Contents
1. [Design Principles](#design-principles)
2. [Component Architecture](#component-architecture)
3. [State Management](#state-management)
4. [Styling Standards](#styling-standards)
5. [Performance Practices](#performance-practices)
6. [Accessibility Guidelines](#accessibility-guidelines)
7. [Testing Standards](#testing-standards)
8. [Code Organization](#code-organization)
9. [Platform-Specific Guidelines](#platform-specific-guidelines)

---

## Design Principles

### 1. User-Centered Design
- **Healthcare-First Approach**: Design with medical professionals and patients in mind
- **Accessibility**: WCAG 2.1 AA compliance for all interfaces
- **Cultural Sensitivity**: Support for multiple Indian languages and cultural contexts
- **Trust & Safety**: Clear visual indicators for verified providers and secure transactions

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

## Component Architecture

### 1. Atomic Design Pattern
```
atoms/
├── Button/
├── Input/
├── Icon/
├── Badge/
└── Avatar/

molecules/
├── SearchBox/
├── AppointmentCard/
├── MedicationItem/
└── NotificationBanner/

organisms/
├── Header/
├── PatientDashboard/
├── ProviderSchedule/
└── PaymentForm/

templates/
├── PageLayout/
├── DashboardLayout/
└── AuthLayout/

pages/
├── HomePage/
├── AppointmentBooking/
└── ProfileSettings/
```

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
