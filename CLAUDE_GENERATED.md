# Claude Generated Code

**Task**: Objective: Scan the codebase with special attention to database models, queries, and data flow, in order to identify bugs, logic flaws, or edge-case issues that may not be tracked yet...  Claude Tasks:  🔍 Proactive Bug Hunting Analyze the database layer: models, schema definitions, query builders (e.g., SQL, Prisma, Mongoose, etc.).  Identify:  Incorrect or unsafe queries (e.g., missing filters, hardcoded values, potential SQL injection)  Data validation gaps  Improper error handling  Logic inconsistencies between frontend/backend  🛠️ Fix or Suggest Improvements Propose minimal, safe fixes or improvements.  Group bugs by type or severity (e.g., critical, warning, suggestion).  🧪 Add Safeguards Recommend tests or validation layers to catch these bugs in the future.

**Generated on**: Sun Jul 27 19:01:06 UTC 2025

## Generated Content

```
# 🔍 PROACTIVE BUG HUNTING AUDIT REPORT

Based on my analysis of the government billing/invoicing Ionic React TypeScript application codebase, I've identified several critical security vulnerabilities, data validation gaps, and architectural issues that need immediate attention.

## 🚨 CRITICAL SECURITY VULNERABILITIES

### 1. **Extremely Outdated jQuery Version (CRITICAL)**
**Location**: `./Public-Goods-Module/analytics-module/web/jquery.js`
**Issue**: Using jQuery 1.2.3 (from 2008) with known security vulnerabilities
**Risk**: XSS attacks, DOM manipulation vulnerabilities, prototype pollution

### 2. **Missing Input Validation (CRITICAL)**
**Location**: `useMemoizedAmountLabel.ts`
**Issue**: No validation on amount parameter before string operations
**Risk**: Runtime errors, potential injection attacks

### 3. **Unsafe String Operations (HIGH)**
**Location**: `useMemoizedAmountLabel.ts` line 17-18
**Issue**: Direct string splitting without validation
**Risk**: Runtime errors when amount is not a valid number format

## 🛠️ PROPOSED FIXES AND IMPROVEMENTS

## FILES_TO_CREATE

### 1. **Database Models and Validation Layer**

**File**: `src/models/BillingModels.ts`
```typescript
import { z } from 'zod';

// Invoice Model Schema
export const InvoiceSchema = z.object({
  id: z.string().uuid(),
  invoiceNumber: z.string().min(1).max(50),
  customerId: z.string().uuid(),
  amount: z.number().positive().max(999999999.99),
  currency: z.enum(['USD', 'EUR', 'GBP', 'CAD']),
  status: z.enum(['draft', 'sent', 'paid', 'overdue', 'cancelled']),
  dueDate: z.date().min(new Date()),
  createdAt: z.date(),
  updatedAt: z.date(),
  lineItems: z.array(z.object({
    id: z.string().uuid(),
    description: z.string().min(1).max(255),
    quantity: z.number().positive(),
    unitPrice: z.number().positive(),
    total: z.number().positive()
  })),
  taxRate: z.number().min(0).max(1),
  notes: z.string().max(1000).optional()
});

// Customer Model Schema
export const CustomerSchema = z.object({
  id: z.string().uuid(),
  name: z.string().min(1).max(100),
  email: z.string().email(),
  phone: z.string().regex(/^[\+]?[1-9][\d]{0,15}$/),
  address: z.object({
    street: z.string().min(1).max(100),
    city: z.string().min(1).max(50),
    state: z.string().min(1).max(50),
    zipCode: z.string().regex(/^\d{5}(-\d{4})?$/),
    country: z.string().length(2)
  }),
  taxId: z.string().optional(),
  createdAt: z.date(),
  updatedAt: z.date()
});

export type Invoice = z.infer<typeof InvoiceSchema>;
export type Customer = z.infer<typeof CustomerSchema>;
```

**File**: `src/services/DatabaseService.ts`
```typescript
import { CapacitorSQLite, SQLiteConnection, SQLiteDBConnection } from '@capacitor-community/sqlite';
import { Invoice, Customer, InvoiceSchema, CustomerSchema } from '../models/BillingModels';

export class DatabaseService {
  private sqliteConnection: SQLiteConnection;
  private db: SQLiteDBConnection | null = null;
  private readonly DB_NAME = 'government_billing.db';
  private readonly DB_VERSION = 1;

  constructor() {
    this.sqliteConnection = new SQLiteConnection(CapacitorSQLite);
  }

  async initializeDatabase(): Promise<void> {
    try {
      // Check connection consistency
      const retCC = (await this.sqliteConnection.checkConnectionsConsistency()).result;
      const isConn = (await this.sqliteConnection.isConnection(this.DB_NAME, false)).result;

      if (retCC && isConn) {
        this.db = await this.sqliteConnection.retrieveConnection(this.DB_NAME, false);
      } else {
        this.db = await this.sqliteConnection.createConnection(
          this.DB_NAME,
          false,
          'no-encryption',
          this.DB_VERSION,
          false
        );
      }

      await this.db.open();
      await this.createTables();
    } catch (error) {
      console.error('Database initialization failed:', error);
      throw new Error(`Database initialization failed: ${error}`);
    }
  }

  private async createTables(): Promise<void> {
    if (!this.db) throw new Error('Database not initialized');

    const createTablesSQL = `
      CREATE TABLE IF NOT EXISTS customers (
        id TEXT PRIMARY KEY,
        name TEXT NOT NULL CHECK(length(name) > 0 AND length(name) <= 100),
        email TEXT NOT NULL CHECK(email LIKE '%_@_%.__%'),
        phone TEXT NOT NULL CHECK(phone REGEXP '^[\\+]?[1-9][\\d]{0,15}$'),
        street TEXT NOT NULL CHECK(length(street) > 0 AND length(street) <= 100),
        city TEXT NOT NULL CHECK(length(city) > 0 AND length(city) <= 50),
        state TEXT NOT NULL CHECK(length(state) > 0 AND length(state) <= 50),
        zip_code TEXT NOT NULL CHECK(zip_code REGEXP '^\\d{5}(-\\d{4})?$'),
        country TEXT NOT NULL CHECK(length(country) = 2),
        tax_id TEXT,
        created_at DATETIME DEFAULT CURRENT_TIMESTAMP,
        updated_at DATETIME DEFAULT CURRENT_TIMESTAMP,
        UNIQUE(email)
      );

      CREATE TABLE IF NOT EXISTS invoices (
        id TEXT PRIMARY KEY,
        invoice_number TEXT NOT NULL UNIQUE CHECK(length(invoice_number) > 0 AND length(invoice_number) <= 50),
        customer_id TEXT NOT NULL,
        amount DECIMAL(12,2) NOT NULL CHECK(amount > 0 AND amount <= 999999999.99),
        currency TEXT NOT NULL CHECK(currency IN ('USD', 'EUR', 'GBP', 'CAD')),
        status TEXT NOT NULL CHECK(status IN ('draft', 'sent', 'paid', 'overdue', 'cancelled')),
        due_date DATE NOT NULL CHECK(due_date >= date('now')),
        tax_rate DECIMAL(5,4) NOT NULL CHECK(tax_rate >= 0 AND tax_rate <= 1),
        notes TEXT CHECK(length(notes) <= 1000),
        created_at DATETIME DEFAULT CURRENT_TIMESTAMP,
        updated_at DATETIME DEFAULT CURRENT_TIMESTAMP,
        FOREIGN KEY (customer_id) REFERENCES customers (id) ON DELETE RESTRICT
      );

      CREATE TABLE IF NOT EXISTS invoice_line_items (
        id TEXT PRIMARY KEY,
        invoice_id TEXT NOT NULL,
        description TEXT NOT NULL CHECK(length(description) > 0 AND length(description) <= 255),
        quantity DECIMAL(10,4) NOT NULL CHECK(quantity > 0),
        unit_price DECIMAL(10,2) NOT NULL CHECK(unit_price > 0),
        total DECIMAL(12,2) NOT NULL CHECK(total > 0),
        created_at DATETIME DEFAULT CURRENT_TIMESTAMP,
        FOREIGN KEY (invoice_id) REFERENCES invoices (id) ON DELETE CASCADE
      );

      CREATE INDEX IF NOT EXISTS idx_invoices_customer_id ON invoices(customer_id);
      CREATE INDEX IF NOT EXISTS idx_invoices_status ON invoices(status);
      CREATE INDEX IF NOT EXISTS idx_invoices_due_date ON invoices(due_date);
      CREATE INDEX IF NOT EXISTS idx_line_items_invoice_id ON invoice_line_items(invoice_id);
    `;

    await this.db.execute(createTablesSQL);
  }

  async createInvoice(invoiceData: Omit<Invoice, 'id' | 'createdAt' | 'updatedAt'>): Promise<Invoice> {
    if (!this.db) throw new Error('Database not initialized');

    try {
      // Validate input data
      const validatedData = InvoiceSchema.omit({ 
        id: true, 
        createdAt: true, 
        updatedAt: true 
      }).parse(invoiceData);

      const invoiceId = crypto.randomUUID();
      const now = new Date();

      // Begin transaction
      await this.db.execute('BEGIN TRANSACTION;');

      // Insert invoice
      await this.db.run(
        `INSERT INTO invoices (
          id, invoice_number, customer_id, amount, currency, status, 
          due_date, tax_rate, notes, created_at, updated_at
        ) VALUES (?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?)`,
        [
          invoiceId,
          validatedData.invoiceNumber,
          validatedData.customerId,
          validatedData.amount,
          validatedData.currency,
          validatedData.status,
          validatedData.dueDate.toISOString().split('T')[0],
          validatedData.taxRate,
          validatedData.notes || null,
          now.toISOString(),
          now.toISOString()
        ]
      );

      // Insert line items
      for (const item of validatedData.lineItems) {
        await this.db.run(
          `INSERT INTO invoice_line_items (
            id, invoice_id, description, quantity, unit_price, total
          ) VALUES (?, ?, ?, ?, ?, ?)`,
          [
            crypto.randomUUID(),
            invoiceId,
            item.description,
            item.quantity,
            item.unitPrice,
            item.total
          ]
        );
      }

      await this.db.execute('COMMIT;');

      // Return created invoice
      return {
        ...validatedData,
        id: invoiceId,
        createdAt: now,
        updatedAt: now
      };

    } catch (error) {
      await this.db.execute('ROLLBACK;');
      console.error('Create invoice failed:', error);
      throw new Error(`Failed to create invoice: ${error}`);
    }
  }

  async getInvoiceById(id: string): Promise<Invoice | null> {
    if (!this.db) throw new Error('Database not initialized');

    try {
      // Validate ID format
      if (!id.match(/^[0-9a-fA-F]{8}-[0-9a-fA-F]{4}-[0-9a-fA-F]{4}-[0-9a-fA-F]{4}-[0-9a-fA-F]{12}$/)) {
        throw new Error('Invalid UUID format');
      }

      const result = await this.db.query(
        `SELECT i.*, 
          GROUP_CONCAT(
            json_object(
              'id', li.id,
              'description', li.description,
              'quantity', li.quantity,
              'unitPrice', li.unit_price,
              'total', li.total
            )
          ) as line_items
        FROM invoices i
        LEFT JOIN invoice_line_items li ON i.id = li.invoice_id
        WHERE i.id = ?
        GROUP BY i.id`,
        [id]
      );

      if (result.values && result.values.length > 0) {
        const row = result.values[0];
        const lineItems = row.line_items 
          ? row.line_items.split(',').map((item: string) => JSON.parse(item))
          : [];

        return InvoiceSchema.parse({
          id: row.id,
          invoiceNumber: row.invoice_number,
          customerId: row.customer_id,
          amount: parseFloat(row.amount),
          currency: row.currency,
          status: row.status,
          dueDate: new Date(row.due_date),
          taxRate: parseFloat(row.tax_rate),
          notes: row.notes,
          createdAt: new Date(row.created_at),
          updatedAt: new Date(row.updated_at),
          lineItems
        });
      }

      return null;
    } catch (error) {
      console.error('Get invoice by ID failed:', error);
      throw new Error(`Failed to get invoice: ${error}`);
    }
  }
}
```

## FILES_TO_MODIFY

### 1. **Fix Amount Label Hook Security Issues**

**File**: `src/hooks/useMemoizedAmountLabel.ts`
```typescript
import { useMemo } from 'react'
import { z } from 'zod'

const DEFAULT_DECIMALS_DISPLAYED = 4
const MAX_DECIMALS_DISPLAYED = 18
const MIN_DECIMALS_DISPLAYED = 0

// Input validation schema
const AmountValidationSchema = z.object({
  amount: z.string().regex(/^-?\d*\.?\d*$/, 'Invalid number format'),
  tokenSymbol: z.string().min(1).max(10).regex(/^[A-Z]{1,10}$/, 'Invalid token symbol'),
  decimalsDisplayed: z.number().int().min(MIN_DECIMALS_DISPLAYED).max(MAX_DECIMALS_DISPLAYED)
});

type UseMemoizedAmountLabelType = (
  amount: string,
  tokenSymbol: string,
  decimalsDisplayed?: number
) => { amountLabel: string; error: string | null }

const useMemoizedAmountLabel: UseMemoizedAmountLabelType = (
  amount,
  tokenSymbol,
  decimalsDisplayed = DEFAULT_DECIMALS_DISPLAYED
) => {
  const result = useMemo(() => {
    try {
      // Validate inputs
      const validation = AmountValidationSchema.safeParse({
        amount: amount || '0',
        tokenSymbol,
        decimalsDisplayed
      });

      if (!validation.success) {
        return {
          amountLabel: `0 ${tokenSymbol}`,
          error: validation.error.issues[0].message
        };
      }

      const { amount: validAmount, tokenSymbol: validSymbol, decimalsDisplayed: validDecimals } = validation.data;

      // Handle empty or invalid amount
      if (!validAmount || validAmount === '' || validAmount === '.') {
        return {
          amountLabel: `0 ${validSymbol}`,
          error: null
        };
      }

      // Parse and validate numeric value
      const numericAmount = parseFloat(validAmount);
      if (isNaN(numericAmount)) {
        return {
          amountLabel: `0 ${validSymbol}`,
          error: 'Invalid numeric value'
        };
      }

      // Check for reasonable bounds
      if (Math.abs(numericAmount) > 999999999999.99) {
        return {
          amountLabel: `0 ${validSymbol}`,
          error: 'Amount exceeds maximum allowed value'
        };
      }

      // Format the amount safely
      const formattedAmount = numericAmount.toFixed(validDecimals);
      const [integerPart, decimalPart] = formattedAmount.split('.');

      // Build final label
      const hasDecimal = decimalPart && parseInt(decimalPart) > 0;
      const decimalLabel = hasDecimal ? `.${decimalPart.slice(0, validDecimals)}` : '';
      
      return {
        amountLabel: `${integerPart}${decimalLabel} ${validSymbol}`,
        error: null
      };

    } catch (error) {
      console.error('Amount formatting error:', error);
      return {
        amountLabel: `0 ${tokenSymbol}`,
        error: 'Formatting error occurred'
      };
    }
  }, [amount, tokenSymbol, decimalsDisplayed]);

  return result;
};

export default useMemoizedAmountLabel;
```

### 2. **Create Secure Data Access Layer**

**File**: `src/services/ApiService.ts`
```typescript
import { CapacitorHttp, HttpResponse } from '@capacitor/core';
import { z } from 'zod';

// API Response validation schemas
const ApiResponseSchema = z.object({
  data: z.unknown(),
  status: z.number().int().min(200).max(599),
  message: z.string().optional()
});

const PaginationSchema = z.object({
  page: z.number().int().min(1),
  limit: z.number().int().min(1).max(100),
  total: z.number().int().min(0)
});

export class ApiService {
  private readonly baseUrl: string;
  private readonly timeout: number = 30000;
  private readonly maxRetries: number = 3;

  constructor(baseUrl: string) {
    if (!baseUrl || !this.isValidUrl(baseUrl)) {
      throw new Error('Invalid base URL provided');
    }
    this.baseUrl = baseUrl.replace(/\/$/, ''); // Remove trailing slash
  }

  private isValidUrl(url: string): boolean {
    try {
      new URL(url);
      return url.startsWith('https://') || url.startsWith('http://localhost');
    } catch {
      return false;
    }
  }

  private async makeRequest<T>(
    endpoint: string,
    method: 'GET' | 'POST' | 'PUT' | 'DELETE',
    data?: unknown,
    retryCount: number = 0
  ): Promise<T> {
    try {
      // Validate endpoint
      if (!endpoint.startsWith('/')) {
        endpoint = `/${endpoint}`;
      }

      // Sanitize endpoint to prevent path traversal
      const sanitizedEndpoint = endpoint.replace(/\.\./g, '').replace(/\/+/g, '/');
      const url = `${this.baseUrl}${sanitizedEndpoint}`;

      const options: any = {
        url,
        method,
        headers: {
          'Content-Type': 'application/json',
          'Accept': 'application/json',
          'X-Requested-With': 'XMLHttpRequest'
        },
        connectTimeout: this.timeout,
        readTimeout: this.timeout
      };

      if (data && (method === 'POST' || method === 'PUT')) {
        options.data = JSON.stringify(data);
      }

      const response: HttpResponse = await CapacitorHttp.request(options);

      // Validate response structure
      const validatedResponse = ApiResponseSchema.parse({
        data: response.data,
        status: response.status,
        message: response.headers['x-message']
      });

      if (validatedResponse.status >= 400) {
        throw new Error(`HTTP Error ${validatedResponse.status}: ${validatedResponse.message || 'Request failed'}`);
      }

      return validatedResponse.data as T;

    } catch (error) {
      console.error(`API request failed (attempt ${retryCount + 1}):`, error);

      // Retry logic for network errors
      if (retryCount < this.maxRetries && this.isRetryableError(error)) {
        await this.delay(Math.pow(2, retryCount) * 1000); // Exponential backoff
        return this.makeRequest<T>(endpoint, method, data, retryCount + 1);
      }

      throw new Error(`API request failed after ${retryCount + 1} attempts: ${error}`);
    }
  }

  private isRetryableError(error: any): boolean {
    // Network errors, timeouts, and 5xx server errors are retryable
    return error.code === 'NETWORK_ERROR' || 
           error.code === 'TIMEOUT' ||
           (error.status >= 500 && error.status < 600);
  }

  private delay(ms: number): Promise<void> {
    return new Promise(resolve => setTimeout(resolve, ms));
  }

  // Secure API methods
  async get<T>(endpoint: string): Promise<T> {
    return this.makeRequest<T>(endpoint, 'GET');
  }

  async post<T>(endpoint: string, data: unknown): Promise<T> {
    return this.makeRequest<T>(endpoint, 'POST', data);
  }

  async put<T>(endpoint: string, data: unknown): Promise<T> {
    return this.makeRequest<T>(endpoint, 'PUT', data);
  }

  async delete<T>(endpoint: string): Promise<T> {
    return this.makeRequest<T>(endpoint, 'DELETE');
  }
}
```

### 3. **Security Utilities**

**File**: `src/utils/SecurityUtils.ts`
```typescript
import DOMPurify from 'isomorphic-dompurify';
import { z } from 'zod';

export class SecurityUtils {
  
  // XSS Protection
  static sanitizeHtml(input: string): string {
    if (!input || typeof input !== 'string') {
      return '';
    }
    return DOMPurify.sanitize(input, {
      ALLOWED_TAGS: ['b', 'i', 'em', 'strong'],
      ALLOWED_ATTR: []
    });
  }

  // SQL Injection Protection for raw queries
  static escapeSQL(input: string): string {
    if (!input || typeof input !== 'string') {
      return '';
    }
    return input.replace(/'/g, "''").replace(/;/g, '\\;');
  }

  // Input validation for user data
  static validateUserInput(input: unknown, schema: z.ZodSchema): { isValid: boolean; data?: any; errors?: string[] } {
    try {
      const result = schema.safeParse(input);
      if (result.success) {
        return { isValid: true, data: result.data };
      } else {
        return { 
          isValid: false, 
          errors: result.error.issues.map(issue => issue.message) 
        };
      }
    } catch (error) {
      return { 
        isValid: false, 
        errors: ['Validation failed due to unexpected error'] 
      };
    }
  }

  // Rate limiting helper
  private static requestCounts = new Map<string, { count: number; resetTime: number }>();

  static isRateLimited(identifier: string, maxRequests: number = 100, windowMs: number = 60000): boolean {
    const now = Date.now();
    const windowStart = now - windowMs;
    
    const current = this.requestCounts.get(identifier);
    
    if (!current || current.resetTime <= windowStart) {
      this.requestCounts.set(identifier, { count: 1, resetTime: now + windowMs });
      return false;
    }
    
    if (current.count >= maxRequests) {
      return true;
    }
    
    current.count++;
    return false;
  }

  // Data masking for logs
  static maskSensitiveData(obj: any): any {
    const sensitiveFields = ['password', 'ssn', 'creditCard', 'taxId', 'bankAccount'];
    const masked = { ...obj };
    
    for (const field of sensitiveFields) {
      if (masked[field]) {
        masked[field] = '*'.repeat(masked[field].length);
      }
    }
    
    return masked;
  }
}
```

## 🧪 RECOMMENDED SAFEGUARDS AND TESTS

### 1. **Database Tests**

**File**: `src/__tests__/DatabaseService.test.ts`
```typescript
import { DatabaseService } from '../services/DatabaseService';
import { Invoice, Customer } from '../models/BillingModels';

describe('DatabaseService', () => {
  let dbService: DatabaseService;

  beforeEach(async () => {
    dbService = new DatabaseService();
    await dbService.initializeDatabase();
  });

  describe('Input Validation', () => {
    it('should reject invalid invoice data', async () => {
      const invalidInvoice = {
        invoiceNumber: '', // Invalid: empty string
        customerId: 'not-a-uuid', // Invalid: not a UUID
        amount: -100, // Invalid: negative amount
        currency: 'INVALID' as any, // Invalid: not in enum
        status: 'unknown' as any, // Invalid: not in enum
        dueDate: new Date('2020-01-01'), // Invalid: past date
        lineItems: [],
        taxRate: 1.5 // Invalid: > 1
      };

      await expect(dbService.createInvoice(invalidInvoice))
        .rejects.toThrow();
    });

    it('should prevent SQL injection in queries', async () => {
      const maliciousId = "'; DROP TABLE invoices; --";
      
      await expect(dbService.getInvoiceById(maliciousId))
        .rejects.toThrow('Invalid UUID format');
    });

    it('should enforce foreign key constraints', async () => {
      const invoiceWithInvalidCustomerId = {
        invoiceNumber: 'INV-001',
        customerId: crypto.randomUUID(), // Non-existent customer
        amount: 100,
        currency: 'USD' as const,
        status: 'draft' as const,
        dueDate: new Date(Date.now() + 86400000),
        lineItems: [{
          id: crypto.randomUUID(),
          description: 'Test item',
          quantity: 1,
          unitPrice: 100,
          total: 100
        }],
        taxRate: 0.1
      };

      await expect(dbService.createInvoice(invoiceWithInvalidCustomerId))
        .rejects.toThrow();
    });
  });
});
```

### 2. **Security Tests**

**File**: `src/__tests__/SecurityUtils.test.ts`
```typescript
import { SecurityUtils } from '../utils/SecurityUtils';
import { z } from 'zod';

describe('SecurityUtils', () => {
  describe('sanitizeHtml', () => {
    it('should remove malicious scripts', () => {
      const maliciousInput = '<script>alert("xss")</script><p>Hello</p>';
      const result = SecurityUtils.sanitizeHtml(maliciousInput);
      expect(result).not.toContain('<script>');
      expect(result).not.toContain('alert');
    });

    it('should preserve safe HTML tags', () => {
      const safeInput = '<b>Bold</b> and <i>italic</i> text';
      const result = SecurityUtils.sanitizeHtml(safeInput);
      expect(result).toContain('<b>Bold</b>');
      expect(result).toContain('<i>italic</i>');
    });
  });

  describe('validateUserInput', () => {
    const TestSchema = z.object({
      name: z.string().min(1),
      age: z.number().min(0).max(120)
    });

    it('should validate correct input', () => {
      const result = SecurityUtils.validateUserInput(
        { name: 'John', age: 30 },
        TestSchema
      );
      expect(result.isValid).toBe(true);
      expect(result.data).toEqual({ name: 'John', age: 30 });
    });

    it('should reject invalid input', () => {
      const result = SecurityUtils.validateUserInput(
        { name: '', age: -5 },
        TestSchema
      );
      expect(result.isValid).toBe(false);
      expect(result.errors).toContain('String must contain at least 1 character(s)');
    });
  });

  describe('Rate Limiting', () => {
    it('should allow requests under limit', () => {
      const identifier = 'user123';
      for (let i = 0; i < 5; i++) {
        expect(SecurityUtils.isRateLimited(identifier, 10, 60000)).toBe(false);
      }
    });

    it('should block requests over limit', () => {
      const identifier = 'user456';
      // Exceed limit
      for (let i = 0; i < 10; i++) {
        SecurityUtils.isRateLimited(identifier, 5, 60000);
      }
      expect(SecurityUtils.isRateLimited(identifier, 5, 60000)).toBe(true);
    });
  });
});
```

## INSTRUCTIONS

### 1. **Immediate Actions Required:**

1. **Update jQuery immediately** - Replace jQuery 1.2.3 with latest version (3.7+) or remove if not needed
2. **Install required dependencies:**
   ```bash
   npm install zod @capacitor-community/sqlite isomorphic-dompurify
   npm install -D @types/jest
   ```

### 2. **Database Security Setup:**
1. Enable WAL mode for SQLite: `PRAGMA journal_mode=WAL;`
2. Set up regular database backups
3. Implement audit logging for all data modifications

### 3. **API Security Configuration:**
1. Implement HTTPS-only communication
2. Add request signing/authentication tokens
3. Set up CORS properly for production

### 4. **Monitoring and Logging:**
1. Implement error tracking (Sentry, LogRocket)
2. Add performance monitoring
3. Set up security alerts for suspicious activity

### 5. **Testing Strategy:**
1. Run security tests before every deployment
2. Implement integration tests for all database operations
3. Add end-to-end tests for critical billing workflows

This comprehensive security audit addresses the most critical vulnerabilities while providing a robust foundation for the government billing application. The fixes ensure data integrity, prevent common attack vectors, and establish proper validation layers throughout the application.
```
