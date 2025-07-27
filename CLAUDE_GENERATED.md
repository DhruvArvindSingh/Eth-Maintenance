# Claude Generated Code

**Task**: Objective: Help design and implement new features based on the current roadmap or project needs. Focus on clean, scalable, and testable code  Claude Tasks:  🧩 Feature Implementation Read the existing codebase and integrate new feature modules where appropriate.  Ensure new code follows existing patterns and conventions.  🗂️ Example Features (replace/add specifics as needed): Add user role-based access control  Implement export to PDF  Add multi-language support  Enable dark mode toggle  🧪 Testing & Documentation Add unit/integration tests for new functionality.  Update README or in-code comments if required.

**Generated on**: Sun Jul 27 18:42:19 UTC 2025

## Generated Content

```
Based on the codebase analysis, I'll help you implement key features for your Ionic React TypeScript government billing/invoicing application. Looking at the existing structure, I can see you have an analytics module with SocialCalc components and some React hooks. Let me implement the requested features with proper architecture.

## FILES_TO_CREATE

### 1. Core Types and Interfaces
- `src/types/auth.ts`
- `src/types/invoice.ts`
- `src/types/theme.ts`
- `src/types/i18n.ts`

### 2. Authentication & Authorization
- `src/hooks/useAuth.ts`
- `src/hooks/usePermissions.ts`
- `src/components/auth/PermissionGuard.tsx`
- `src/services/authService.ts`

### 3. Theme System
- `src/hooks/useTheme.ts`
- `src/components/theme/ThemeProvider.tsx`
- `src/styles/themes.css`

### 4. Internationalization
- `src/hooks/useTranslation.ts`
- `src/services/i18nService.ts`
- `src/locales/en.json`
- `src/locales/es.json`

### 5. PDF Export
- `src/hooks/usePdfExport.ts`
- `src/services/pdfService.ts`
- `src/components/export/PdfExportButton.tsx`

### 6. Testing
- `src/__tests__/hooks/useAuth.test.ts`
- `src/__tests__/services/pdfService.test.ts`

## FILES_TO_MODIFY

- `src/App.tsx` (wrap with providers)
- `package.json` (add dependencies)

## CODE_CHANGES

### 1. Package Dependencies

**FILES_TO_MODIFY: package.json**

```json
{
  "dependencies": {
    "@ionic/react": "^7.0.0",
    "react": "^18.2.0",
    "react-dom": "^18.2.0",
    "typescript": "^5.0.0",
    "jspdf": "^2.5.1",
    "html2canvas": "^1.4.1",
    "i18next": "^23.7.0",
    "react-i18next": "^13.5.0"
  },
  "devDependencies": {
    "@testing-library/react": "^14.0.0",
    "@testing-library/jest-dom": "^6.0.0",
    "jest": "^29.0.0"
  }
}
```

### 2. Core Types

**FILES_TO_CREATE: src/types/auth.ts**

```typescript
export interface User {
  id: string;
  email: string;
  name: string;
  role: UserRole;
  department?: string;
  permissions: Permission[];
  isActive: boolean;
  lastLogin?: Date;
}

export enum UserRole {
  ADMIN = 'admin',
  MANAGER = 'manager',
  CLERK = 'clerk',
  VIEWER = 'viewer'
}

export enum Permission {
  CREATE_INVOICE = 'create_invoice',
  EDIT_INVOICE = 'edit_invoice',
  DELETE_INVOICE = 'delete_invoice',
  VIEW_INVOICE = 'view_invoice',
  EXPORT_PDF = 'export_pdf',
  MANAGE_USERS = 'manage_users',
  VIEW_REPORTS = 'view_reports'
}

export interface AuthState {
  user: User | null;
  isAuthenticated: boolean;
  isLoading: boolean;
  error: string | null;
}
```

**FILES_TO_CREATE: src/types/invoice.ts**

```typescript
export interface Invoice {
  id: string;
  invoiceNumber: string;
  clientId: string;
  clientName: string;
  amount: number;
  tax: number;
  total: number;
  status: InvoiceStatus;
  dueDate: Date;
  createdDate: Date;
  createdBy: string;
  items: InvoiceItem[];
  notes?: string;
}

export interface InvoiceItem {
  id: string;
  description: string;
  quantity: number;
  unitPrice: number;
  total: number;
}

export enum InvoiceStatus {
  DRAFT = 'draft',
  SENT = 'sent',
  PAID = 'paid',
  OVERDUE = 'overdue',
  CANCELLED = 'cancelled'
}
```

**FILES_TO_CREATE: src/types/theme.ts**

```typescript
export interface ThemeConfig {
  mode: 'light' | 'dark';
  primaryColor: string;
  accentColor: string;
  backgroundColor: string;
  textColor: string;
  borderColor: string;
}

export interface ThemeContextType {
  theme: ThemeConfig;
  toggleTheme: () => void;
  setTheme: (theme: ThemeConfig) => void;
  isDark: boolean;
}
```

### 3. Authentication System

**FILES_TO_CREATE: src/services/authService.ts**

```typescript
import { User, UserRole, Permission } from '../types/auth';

export class AuthService {
  private static instance: AuthService;
  private currentUser: User | null = null;

  static getInstance(): AuthService {
    if (!AuthService.instance) {
      AuthService.instance = new AuthService();
    }
    return AuthService.instance;
  }

  async login(email: string, password: string): Promise<User> {
    try {
      // Mock authentication - replace with actual API call
      const mockUser: User = {
        id: '1',
        email,
        name: 'John Doe',
        role: UserRole.ADMIN,
        permissions: [
          Permission.CREATE_INVOICE,
          Permission.EDIT_INVOICE,
          Permission.VIEW_INVOICE,
          Permission.EXPORT_PDF,
          Permission.MANAGE_USERS
        ],
        isActive: true,
        lastLogin: new Date()
      };

      this.currentUser = mockUser;
      localStorage.setItem('auth_token', 'mock_token');
      localStorage.setItem('user', JSON.stringify(mockUser));
      
      return mockUser;
    } catch (error) {
      throw new Error('Authentication failed');
    }
  }

  async logout(): Promise<void> {
    this.currentUser = null;
    localStorage.removeItem('auth_token');
    localStorage.removeItem('user');
  }

  getCurrentUser(): User | null {
    if (!this.currentUser) {
      const stored = localStorage.getItem('user');
      if (stored) {
        this.currentUser = JSON.parse(stored);
      }
    }
    return this.currentUser;
  }

  hasPermission(permission: Permission): boolean {
    const user = this.getCurrentUser();
    return user?.permissions.includes(permission) || false;
  }

  hasRole(role: UserRole): boolean {
    const user = this.getCurrentUser();
    return user?.role === role;
  }
}
```

**FILES_TO_CREATE: src/hooks/useAuth.ts**

```typescript
import { useState, useEffect, createContext, useContext } from 'react';
import { User, AuthState } from '../types/auth';
import { AuthService } from '../services/authService';

const AuthContext = createContext<{
  authState: AuthState;
  login: (email: string, password: string) => Promise<void>;
  logout: () => Promise<void>;
} | null>(null);

export const useAuth = () => {
  const context = useContext(AuthContext);
  if (!context) {
    throw new Error('useAuth must be used within AuthProvider');
  }
  return context;
};

export const useAuthState = () => {
  const [authState, setAuthState] = useState<AuthState>({
    user: null,
    isAuthenticated: false,
    isLoading: true,
    error: null
  });

  const authService = AuthService.getInstance();

  useEffect(() => {
    const initAuth = () => {
      const user = authService.getCurrentUser();
      setAuthState({
        user,
        isAuthenticated: !!user,
        isLoading: false,
        error: null
      });
    };

    initAuth();
  }, []);

  const login = async (email: string, password: string) => {
    setAuthState(prev => ({ ...prev, isLoading: true, error: null }));
    
    try {
      const user = await authService.login(email, password);
      setAuthState({
        user,
        isAuthenticated: true,
        isLoading: false,
        error: null
      });
    } catch (error) {
      setAuthState(prev => ({
        ...prev,
        isLoading: false,
        error: error instanceof Error ? error.message : 'Login failed'
      }));
    }
  };

  const logout = async () => {
    await authService.logout();
    setAuthState({
      user: null,
      isAuthenticated: false,
      isLoading: false,
      error: null
    });
  };

  return { authState, login, logout };
};

export { AuthContext };
```

**FILES_TO_CREATE: src/hooks/usePermissions.ts**

```typescript
import { useMemo } from 'react';
import { Permission, UserRole } from '../types/auth';
import { useAuth } from './useAuth';

export const usePermissions = () => {
  const { authState } = useAuth();

  const permissions = useMemo(() => {
    const user = authState.user;
    if (!user) return [];
    return user.permissions;
  }, [authState.user]);

  const hasPermission = useMemo(() => {
    return (permission: Permission) => permissions.includes(permission);
  }, [permissions]);

  const hasRole = useMemo(() => {
    return (role: UserRole) => authState.user?.role === role;
  }, [authState.user]);

  const hasAnyRole = useMemo(() => {
    return (roles: UserRole[]) => 
      authState.user ? roles.includes(authState.user.role) : false;
  }, [authState.user]);

  return {
    permissions,
    hasPermission,
    hasRole,
    hasAnyRole,
    userRole: authState.user?.role
  };
};
```

**FILES_TO_CREATE: src/components/auth/PermissionGuard.tsx**

```typescript
import React from 'react';
import { IonAlert } from '@ionic/react';
import { Permission, UserRole } from '../../types/auth';
import { usePermissions } from '../../hooks/usePermissions';

interface PermissionGuardProps {
  children: React.ReactNode;
  permission?: Permission;
  role?: UserRole;
  roles?: UserRole[];
  fallback?: React.ReactNode;
  showAlert?: boolean;
}

export const PermissionGuard: React.FC<PermissionGuardProps> = ({
  children,
  permission,
  role,
  roles,
  fallback = null,
  showAlert = false
}) => {
  const { hasPermission, hasRole, hasAnyRole } = usePermissions();
  const [showAccessDenied, setShowAccessDenied] = React.useState(false);

  const hasAccess = React.useMemo(() => {
    if (permission && !hasPermission(permission)) return false;
    if (role && !hasRole(role)) return false;
    if (roles && !hasAnyRole(roles)) return false;
    return true;
  }, [permission, role, roles, hasPermission, hasRole, hasAnyRole]);

  React.useEffect(() => {
    if (!hasAccess && showAlert) {
      setShowAccessDenied(true);
    }
  }, [hasAccess, showAlert]);

  if (!hasAccess) {
    return (
      <>
        {fallback}
        <IonAlert
          isOpen={showAccessDenied}
          onDidDismiss={() => setShowAccessDenied(false)}
          header="Access Denied"
          message="You don't have permission to access this feature."
          buttons={['OK']}
        />
      </>
    );
  }

  return <>{children}</>;
};
```

### 4. Theme System

**FILES_TO_CREATE: src/hooks/useTheme.ts**

```typescript
import { useState, useEffect, createContext, useContext } from 'react';
import { ThemeConfig, ThemeContextType } from '../types/theme';

const defaultLightTheme: ThemeConfig = {
  mode: 'light',
  primaryColor: '#3880ff',
  accentColor: '#0cd1e8',
  backgroundColor: '#ffffff',
  textColor: '#000000',
  borderColor: '#e0e0e0'
};

const defaultDarkTheme: ThemeConfig = {
  mode: 'dark',
  primaryColor: '#428cff',
  accentColor: '#36d2ff',
  backgroundColor: '#121212',
  textColor: '#ffffff',
  borderColor: '#333333'
};

const ThemeContext = createContext<ThemeContextType | null>(null);

export const useTheme = () => {
  const context = useContext(ThemeContext);
  if (!context) {
    throw new Error('useTheme must be used within ThemeProvider');
  }
  return context;
};

export const useThemeState = () => {
  const [theme, setThemeState] = useState<ThemeConfig>(() => {
    const stored = localStorage.getItem('app_theme');
    return stored ? JSON.parse(stored) : defaultLightTheme;
  });

  const isDark = theme.mode === 'dark';

  useEffect(() => {
    document.documentElement.setAttribute('data-theme', theme.mode);
    
    // Apply CSS variables
    const root = document.documentElement;
    root.style.setProperty('--primary-color', theme.primaryColor);
    root.style.setProperty('--accent-color', theme.accentColor);
    root.style.setProperty('--background-color', theme.backgroundColor);
    root.style.setProperty('--text-color', theme.textColor);
    root.style.setProperty('--border-color', theme.borderColor);

    localStorage.setItem('app_theme', JSON.stringify(theme));
  }, [theme]);

  const toggleTheme = () => {
    const newTheme = isDark ? defaultLightTheme : defaultDarkTheme;
    setThemeState(newTheme);
  };

  const setTheme = (newTheme: ThemeConfig) => {
    setThemeState(newTheme);
  };

  return {
    theme,
    toggleTheme,
    setTheme,
    isDark
  };
};

export { ThemeContext };
```

**FILES_TO_CREATE: src/components/theme/ThemeProvider.tsx**

```typescript
import React from 'react';
import { ThemeContext, useThemeState } from '../../hooks/useTheme';

interface ThemeProviderProps {
  children: React.ReactNode;
}

export const ThemeProvider: React.FC<ThemeProviderProps> = ({ children }) => {
  const themeState = useThemeState();

  return (
    <ThemeContext.Provider value={themeState}>
      {children}
    </ThemeContext.Provider>
  );
};
```

### 5. PDF Export System

**FILES_TO_CREATE: src/services/pdfService.ts**

```typescript
import jsPDF from 'jspdf';
import html2canvas from 'html2canvas';
import { Invoice } from '../types/invoice';

export class PdfService {
  static async exportInvoiceToPdf(invoice: Invoice): Promise<Blob> {
    try {
      const pdf = new jsPDF();
      
      // Header
      pdf.setFontSize(20);
      pdf.text('GOVERNMENT INVOICE', 20, 30);
      
      // Invoice details
      pdf.setFontSize(12);
      pdf.text(`Invoice #: ${invoice.invoiceNumber}`, 20, 50);
      pdf.text(`Client: ${invoice.clientName}`, 20, 60);
      pdf.text(`Date: ${invoice.createdDate.toLocaleDateString()}`, 20, 70);
      pdf.text(`Due Date: ${invoice.dueDate.toLocaleDateString()}`, 20, 80);
      pdf.text(`Status: ${invoice.status.toUpperCase()}`, 20, 90);

      // Items table
      let yPos = 110;
      pdf.text('Description', 20, yPos);
      pdf.text('Qty', 100, yPos);
      pdf.text('Unit Price', 130, yPos);
      pdf.text('Total', 170, yPos);
      
      yPos += 10;
      pdf.line(20, yPos, 190, yPos); // horizontal line
      yPos += 10;

      invoice.items.forEach(item => {
        pdf.text(item.description, 20, yPos);
        pdf.text(item.quantity.toString(), 100, yPos);
        pdf.text(`$${item.unitPrice.toFixed(2)}`, 130, yPos);
        pdf.text(`$${item.total.toFixed(2)}`, 170, yPos);
        yPos += 10;
      });

      // Totals
      yPos += 10;
      pdf.text(`Subtotal: $${invoice.amount.toFixed(2)}`, 130, yPos);
      yPos += 10;
      pdf.text(`Tax: $${invoice.tax.toFixed(2)}`, 130, yPos);
      yPos += 10;
      pdf.setFontSize(14);
      pdf.text(`Total: $${invoice.total.toFixed(2)}`, 130, yPos);

      return pdf.output('blob');
    } catch (error) {
      throw new Error('Failed to generate PDF');
    }
  }

  static async exportElementToPdf(
    elementId: string,
    filename: string = 'export.pdf'
  ): Promise<void> {
    try {
      const element = document.getElementById(elementId);
      if (!element) {
        throw new Error('Element not found');
      }

      const canvas = await html2canvas(element, {
        scale: 2,
        useCORS: true,
        logging: false
      });

      const pdf = new jsPDF({
        orientation: 'portrait',
        unit: 'mm',
        format: 'a4'
      });

      const imgData = canvas.toDataURL('image/png');
      const imgWidth = 210;
      const pageHeight = 295;
      const imgHeight = (canvas.height * imgWidth) / canvas.width;
      let heightLeft = imgHeight;

      let position = 0;

      pdf.addImage(imgData, 'PNG', 0, position, imgWidth, imgHeight);
      heightLeft -= pageHeight;

      while (heightLeft >= 0) {
        position = heightLeft - imgHeight;
        pdf.addPage();
        pdf.addImage(imgData, 'PNG', 0, position, imgWidth, imgHeight);
        heightLeft -= pageHeight;
      }

      pdf.save(filename);
    } catch (error) {
      throw new Error('Failed to export PDF');
    }
  }
}
```

**FILES_TO_CREATE: src/hooks/usePdfExport.ts**

```typescript
import { useState } from 'react';
import { PdfService } from '../services/pdfService';
import { Invoice } from '../types/invoice';

export const usePdfExport = () => {
  const [isExporting, setIsExporting] = useState(false);
  const [error, setError] = useState<string | null>(null);

  const exportInvoice = async (invoice: Invoice): Promise<void> => {
    setIsExporting(true);
    setError(null);

    try {
      const blob = await PdfService.exportInvoiceToPdf(invoice);
      const url = URL.createObjectURL(blob);
      const a = document.createElement('a');
      a.href = url;
      a.download = `invoice-${invoice.invoiceNumber}.pdf`;
      document.body.appendChild(a);
      a.click();
      document.body.removeChild(a);
      URL.revokeObjectURL(url);
    } catch (err) {
      setError(err instanceof Error ? err.message : 'Export failed');
    } finally {
      setIsExporting(false);
    }
  };

  const exportElement = async (
    elementId: string,
    filename?: string
  ): Promise<void> => {
    setIsExporting(true);
    setError(null);

    try {
      await PdfService.exportElementToPdf(elementId, filename);
    } catch (err) {
      setError(err instanceof Error ? err.message : 'Export failed');
    } finally {
      setIsExporting(false);
    }
  };

  return {
    exportInvoice,
    exportElement,
    isExporting,
    error
  };
};
```

**FILES_TO_CREATE: src/components/export/PdfExportButton.tsx**

```typescript
import React from 'react';
import {
  IonButton,
  IonIcon,
  IonSpinner,
  IonToast
} from '@ionic/react';
import { documentOutline } from 'ionicons/icons';
import { usePdfExport } from '../../hooks/usePdfExport';
import { usePermissions } from '../../hooks/usePermissions';
import { Permission } from '../../types/auth';
import { Invoice } from '../../types/invoice';

interface PdfExportButtonProps {
  invoice?: Invoice;
  elementId?: string;
  filename?: string;
  variant?: 'solid' | 'outline' | 'clear';
  size?: 'small' | 'default' | 'large';
  disabled?: boolean;
  children?: React.ReactNode;
}

export const PdfExportButton: React.FC<PdfExportButtonProps> = ({
  invoice,
  elementId,
  filename,
  variant = 'solid',
  size = 'default',
  disabled = false,
  children = 'Export PDF'
}) => {
  const { exportInvoice, exportElement, isExporting, error } = usePdfExport();
  const { hasPermission } = usePermissions();
  const [showToast, setShowToast] = React.useState(false);

  const canExport = hasPermission(Permission.EXPORT_PDF);

  const handleExport = async () => {
    if (!canExport) {
      setShowToast(true);
      return;
    }

    try {
      if (invoice) {
        await exportInvoice(invoice);
      } else if (elementId) {
        await exportElement(elementId, filename);
      }
    } catch (err) {
      // Error is handled by the hook
    }
  };

  return (
    <>
      <IonButton
        fill={variant}
        size={size}
        disabled={disabled || isExporting || !canExport}
        onClick={handleExport}
      >
        {isExporting ? (
          <IonSpinner slot="start" />
        ) : (
          <IonIcon icon={documentOutline} slot="start" />
        )}
        {children}
      </IonButton>

      <IonToast
        isOpen={!!error}
        message={error || ''}
        duration={3000}
        color="danger"
        onDidDismiss={() => {}}
      />

      <IonToast
        isOpen={showToast}
        message="You don't have permission to export PDFs"
        duration={3000}
        color="warning"
        onDidDismiss={() => setShowToast(false)}
      />
    </>
  );
};
```

### 6. Internationalization System

**FILES_TO_CREATE: src/services/i18nService.ts**

```typescript
import i18n from 'i18next';
import { initReactI18next } from 'react-i18next';

// Import translation files
import en from '../locales/en.json';
import es from '../locales/es.json';

const resources = {
  en: { translation: en },
  es: { translation: es }
};

i18n
  .use(initReactI18next)
  .init({
    resources,
    lng: localStorage.getItem('app_language') || 'en',
    fallbackLng: 'en',
    interpolation: {
      escapeValue: false
    }
  });

export default i18n;
```

**FILES_TO_CREATE: src/locales/en.json**

```json
{
  "common": {
    "save": "Save",
    "cancel": "Cancel",
    "delete": "Delete",
    "edit": "Edit",
    "create": "Create",
    "export": "Export",
    "loading": "Loading...",
    "error": "Error",
    "success": "Success"
  },
  "auth": {
    "login": "Login",
    "logout": "Logout",
    "email": "Email",
    "password": "Password",
    "loginFailed": "Login failed"
  },
  "invoice": {
    "title": "Invoices",
    "createInvoice": "Create Invoice",
    "invoiceNumber": "Invoice Number",
    "client": "Client",
    "amount": "Amount",
    "status": "Status",
    "dueDate": "Due Date",
    "exportPdf": "Export PDF"
  },
  "theme": {
    "darkMode": "Dark Mode",
    "lightMode": "Light Mode"
  }
}
```

**FILES_TO_CREATE: src/locales/es.json**

```json
{
  "common": {
    "save": "Guardar",
    "cancel": "Cancelar",
    "delete": "Eliminar",
    "edit": "Editar",
    "create": "Crear",
    "export": "Exportar",
    "loading": "Cargando...",
    "error": "Error",
    "success": "Éxito"
  },
  "auth": {
    "login": "Iniciar Sesión",
    "logout": "Cerrar Sesión",
    "email": "Correo Electrónico",
    "password": "Contraseña",
    "loginFailed": "Falló el inicio de sesión"
  },
  "invoice": {
    "title": "Facturas",
    "createInvoice": "Crear Factura",
    "invoiceNumber": "Número de Factura",
    "client": "Cliente",
    "amount": "Cantidad",
    "status": "Estado",
    "dueDate": "Fecha de Vencimiento",
    "exportPdf": "Exportar PDF"
  },
  "theme": {
    "darkMode": "Modo Oscuro",
    "lightMode": "Modo Claro"
  }
}
```

**FILES_TO_CREATE: src/hooks/useTranslation.ts**

```typescript
import { useTranslation as useI18nTranslation } from 'react-i18next';

export const useTranslation = () => {
  const { t, i18n } = useI18nTranslation();

  const changeLanguage = (language: string) => {
    i18n.changeLanguage(language);
    localStorage.setItem('app_language', language);
  };

  return {
    t,
    changeLanguage,
    currentLanguage: i18n.language,
    languages: ['en', 'es']
  };
};
```

### 7. Main App Integration

**FILES_TO_MODIFY: src/App.tsx**

```typescript
import React from 'react';
import { IonApp, IonRouterOutlet, setupIonicReact } from '@ionic/react';
import { IonReactRouter } from '@ionic/react-router';
import { Route } from 'react-router-dom';

// Providers
import { ThemeProvider } from './components/theme/ThemeProvider';
import { AuthContext, useAuthState } from './hooks/useAuth';

// Services
import './services/i18nService';

// Core CSS
import '@ionic/react/css/core.css';
import '@ionic/react/css/normalize.css';
import '@ionic/react/css/structure.css';
import '@ionic/react/css/typography.css';

// Optional CSS
import '@ionic/react/css/padding.css';
import '@ionic/react/css/float-elements.css';
import '@ionic/react/css/text-alignment.css';
import '@ionic/react/css/text-transformation.css';
import '@ionic/react/css/flex-utils.css';
import '@ionic/react/css/display.css';

// Theme styles
import './styles/themes.css';

setupIonicReact();

const AppProviders: React.FC<{ children: React.ReactNode }> = ({ children }) => {
  const authState = useAuthState();

  return (
    <ThemeProvider>
      <AuthContext.Provider value={authState}>
        {children}
      </AuthContext.Provider>
    </ThemeProvider>
  );
};

const App: React.FC = () => (
  <IonApp>
    <AppProviders>
      <IonReactRouter>
        <IonRouterOutlet>
          <Route exact path="/home">
            {/* Your home component */}
          </Route>
          <Route exact path="/">
            {/* Your default route */}
          </Route>
        </IonRouterOutlet>
      </IonReactRouter>
    </AppProviders>
  </IonApp>
);

export default App;
```

**FILES_TO_CREATE: src/styles/themes.css**

```css
:root {
  --primary-color: #3880ff;
  --accent-color: #0cd1e8;
  --background-color: #ffffff;
  --text-color: #000000;
  --border-color: #e0e0e0;
}

[data-theme="dark"] {
  --primary-color: #428cff;
  --accent-color: #36d2ff;
  --background-color: #121212;
  --text-color: #ffffff;
  --border-color: #333333;
}

/* Apply theme variables */
ion-app {
  --ion-color-primary: var(--primary-color);
  --ion-background-color: var(--background-color);
  --ion-text-color: var(--text-color);
  --ion-border-color: var(--border-color);
}

/* Custom component styles */
.theme-transition {
  transition: background-color 0.3s ease, color 0.3s ease;
}

.invoice-export-container {
  padding: 20px;
  background: var(--background-color);
  color: var(--text-color);
}

.permission-denied {
  opacity: 0.5;
  pointer-events: none;
}
```

### 8. Testing

**FILES_TO_CREATE: src/__tests__/hooks/useAuth.test.ts**

```typescript
import { renderHook, act } from '@testing-library/react';
import { useAuthState } from '../../hooks/useAuth';
import { AuthService } from '../../services/authService';

// Mock AuthService
jest.mock('../../services/authService');

describe('useAuth', () => {
  beforeEach(() => {
```
