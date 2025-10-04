# Expense_Management_System_Using_Digital_Ledger_System3
All the codes are here:


Database Schema
   -- Core Tables
CREATE TABLE companies (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name VARCHAR(255) NOT NULL,
    country_code VARCHAR(3) NOT NULL,
    default_currency VARCHAR(3) NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE users (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    company_id UUID REFERENCES companies(id),
    email VARCHAR(255) UNIQUE NOT NULL,
    password_hash VARCHAR(255) NOT NULL,
    role VARCHAR(20) CHECK (role IN ('admin', 'manager', 'employee')) NOT NULL,
    manager_id UUID REFERENCES users(id),
    is_manager_approver BOOLEAN DEFAULT TRUE,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE expense_categories (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    company_id UUID REFERENCES companies(id),
    name VARCHAR(100) NOT NULL,
    description TEXT,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Ledger Core Tables
CREATE TABLE expense_ledger (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    company_id UUID REFERENCES companies(id),
    employee_id UUID REFERENCES users(id),
    amount DECIMAL(15,2) NOT NULL,
    currency VARCHAR(3) NOT NULL,
    converted_amount DECIMAL(15,2),
    category_id UUID REFERENCES expense_categories(id),
    description TEXT,
    expense_date DATE NOT NULL,
    status VARCHAR(20) CHECK (status IN ('pending', 'approved', 'rejected', 'cancelled')) DEFAULT 'pending',
    receipt_url TEXT,
    ocr_data JSONB,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE approval_workflows (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    company_id UUID REFERENCES companies(id),
    name VARCHAR(255) NOT NULL,
    approval_type VARCHAR(20) CHECK (approval_type IN ('sequential', 'conditional', 'hybrid')) NOT NULL,
    approval_percentage INTEGER DEFAULT 100,
    specific_approver_rule JSONB,
    is_active BOOLEAN DEFAULT TRUE,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE approval_sequences (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    workflow_id UUID REFERENCES approval_workflows(id),
    approver_id UUID REFERENCES users(id),
    sequence_order INTEGER NOT NULL,
    is_required BOOLEAN DEFAULT TRUE,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE expense_approvals (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    expense_id UUID REFERENCES expense_ledger(id),
    approver_id UUID REFERENCES users(id),
    sequence_order INTEGER NOT NULL,
    status VARCHAR(20) CHECK (status IN ('pending', 'approved', 'rejected')) DEFAULT 'pending',
    comments TEXT,
    approved_at TIMESTAMP,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE currency_rates (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    base_currency VARCHAR(3) NOT NULL,
    target_currency VARCHAR(3) NOT NULL,
    exchange_rate DECIMAL(10,6) NOT NULL,
    effective_date DATE NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Indexes for performance
CREATE INDEX idx_expense_ledger_employee ON expense_ledger(employee_id);
CREATE INDEX idx_expense_ledger_status ON expense_ledger(status);



3. Backend Implementation(Node.js/TypeScript)
4.    // types.ts
export interface Company {
  id: string;
  name: string;
  country_code: string;
  default_currency: string;
  created_at: Date;
}

export interface User {
  id: string;
  company_id: string;
  email: string;
  role: 'admin' | 'manager' | 'employee';
  manager_id?: string;
  is_manager_approver: boolean;
  created_at: Date;
}

export interface Expense {
  id: string;
  company_id: string;
  employee_id: string;
  amount: number;
  currency: string;
  converted_amount?: number;
  category_id: string;
  description: string;
  expense_date: Date;
  status: 'pending' | 'approved' | 'rejected' | 'cancelled';
  receipt_url?: string;
  ocr_data?: any;
  created_at: Date;
  updated_at: Date;
}

export interface ApprovalWorkflow {
  id: string;
  company_id: string;
  name: string;
  approval_type: 'sequential' | 'conditional' | 'hybrid';
  approval_percentage: number;
  specific_approver_rule?: any;
  is_active: boolean;
}

export interface ApprovalSequence {
  id: string;
  workflow_id: string;
  approver_id: string;
  sequence_order: number;
  is_required: boolean;
}

// expenseService.ts
import { Pool } from 'pg';
import { v4 as uuidv4 } from 'uuid';

export class ExpenseService {
  constructor(private db: Pool) {}

  async createExpense(expenseData: Omit<Expense, 'id' | 'created_at' | 'updated_at'>): Promise<Expense> {
    const client = await this.db.connect();
    
    try {
      await client.query('BEGIN');

      // Convert currency if needed
      const convertedAmount = await this.convertCurrency(
        expenseData.amount,
        expenseData.currency,
        expenseData.company_id
      );

      // Create expense in ledger
      const expenseQuery = `
        INSERT INTO expense_ledger (
          id, company_id, employee_id, amount, currency, converted_amount,
          category_id, description, expense_date, receipt_url, ocr_data
        ) VALUES ($1, $2, $3, $4, $5, $6, $7, $8, $9, $10, $11)
        RETURNING *
      `;

      const expenseResult = await client.query(expenseQuery, [
        uuidv4(), expenseData.company_id, expenseData.employee_id, expenseData.amount,
        expenseData.currency, convertedAmount, expenseData.category_id,
        expenseData.description, expenseData.expense_date, expenseData.receipt_url,
        expenseData.ocr_data
      ]);

      const expense = expenseResult.rows[0];

      // Initialize approval workflow
      await this.initializeApprovalWorkflow(expense.id, expenseData.company_id, client);

      await client.query('COMMIT');
      return expense;
    } catch (error) {
      await client.query('ROLLBACK');
      throw error;
    } finally {
      client.release();
    }
  }

  private async initializeApprovalWorkflow(expenseId: string, companyId: string, client: any) {
    // Get employee details
    const employeeQuery = `
      SELECT u.*, m.is_manager_approver as manager_is_approver
      FROM users u
      LEFT JOIN users m ON u.manager_id = m.id
      WHERE u.id = (SELECT employee_id FROM expense_ledger WHERE id = $1)
    `;
    const employeeResult = await client.query(employeeQuery, [expenseId]);
    const employee = employeeResult.rows[0];

    // Get active workflow for company
    const workflowQuery = `
      SELECT * FROM approval_workflows 
      WHERE company_id = $1 AND is_active = true 
      ORDER BY created_at DESC LIMIT 1
    `;
    const workflowResult = await client.query(workflowQuery, [companyId]);
    const workflow = workflowResult.rows[0];

    if (!workflow) {
      // Use default sequential workflow
      await this.createDefaultApprovalSequence(expenseId, employee, client);
    } else {
      await this.createWorkflowApprovals(expenseId, workflow, employee, client);
    }
  }

  private async createDefaultApprovalSequence(expenseId: string, employee: any, client: any) {
    const sequences: any[] = [];

    // Add manager if applicable
    if (employee.manager_id && employee.manager_is_approver) {
      sequences.push({
        approver_id: employee.manager_id,
        sequence_order: 1
      });
    }

    // Add sequences based on workflow configuration
    for (const seq of sequences) {
      const approvalQuery = `
        INSERT INTO expense_approvals (id, expense_id, approver_id, sequence_order)
        VALUES ($1, $2, $3, $4)
      `;
      await client.query(approvalQuery, [uuidv4(), expenseId, seq.approver_id, seq.sequence_order]);
    }
  }

  private async createWorkflowApprovals(expenseId: string, workflow: ApprovalWorkflow, employee: any, client: any) {
    if (workflow.approval_type === 'sequential') {
      await this.createSequentialApprovals(expenseId, workflow.id, client);
    } else if (workflow.approval_type === 'conditional') {
      await this.createConditionalApprovals(expenseId, workflow, client);
    } else if (workflow.approval_type === 'hybrid') {
      await this.createHybridApprovals(expenseId, workflow, client);
    }
  }

  private async createSequentialApprovals(expenseId: string, workflowId: string, client: any) {
    const sequenceQuery = `
      SELECT * FROM approval_sequences 
      WHERE workflow_id = $1 
      ORDER BY sequence_order
    `;
    const sequenceResult = await client.query(sequenceQuery, [workflowId]);
    const sequences = sequenceResult.rows;

    for (const seq of sequences) {
      const approvalQuery = `
        INSERT INTO expense_approvals (id, expense_id, approver_id, sequence_order)
        VALUES ($1, $2, $3, $4)
      `;
      await client.query(approvalQuery, [uuidv4(), expenseId, seq.approver_id, seq.sequence_order]);
    }
  }

  async approveExpense(expenseId: string, approverId: string, comments?: string): Promise<void> {
    const client = await this.db.connect();
    
    try {
      await client.query('BEGIN');

      // Update approval status
      const updateApprovalQuery = `
        UPDATE expense_approvals 
        SET status = 'approved', comments = $1, approved_at = NOW(), updated_at = NOW()
        WHERE expense_id = $2 AND approver_id = $3 AND status = 'pending'
        RETURNING *
      `;
      await client.query(updateApprovalQuery, [comments, expenseId, approverId]);

      // Check if expense should be fully approved
      await this.checkExpenseApprovalStatus(expenseId, client);

      await client.query('COMMIT');
    } catch (error) {
      await client.query('ROLLBACK');
      throw error;
    } finally {
      client.release();
    }
  }

  private async checkExpenseApprovalStatus(expenseId: string, client: any) {
    const expenseQuery = `
      SELECT el.*, w.approval_type, w.approval_percentage, w.specific_approver_rule
      FROM expense_ledger el
      LEFT JOIN approval_workflows w ON w.company_id = el.company_id AND w.is_active = true
      WHERE el.id = $1
    `;
    const expenseResult = await client.query(expenseQuery, [expenseId]);
    const expense = expenseResult.rows[0];

    const approvalsQuery = `
      SELECT * FROM expense_approvals 
      WHERE expense_id = $1 
      ORDER BY sequence_order
    `;
    const approvalsResult = await client.query(approvalsQuery, [expenseId]);
    const approvals = approvalsResult.rows;

    let isApproved = false;

    if (expense.approval_type === 'sequential') {
      isApproved = this.checkSequentialApproval(approvals);
    } else if (expense.approval_type === 'conditional') {
      isApproved = this.checkConditionalApproval(approvals, expense);
    } else if (expense.approval_type === 'hybrid') {
      isApproved = this.checkHybridApproval(approvals, expense);
    }

    if (isApproved) {
      await client.query(
        'UPDATE expense_ledger SET status = $1, updated_at = NOW() WHERE id = $2',
        ['approved', expenseId]
      );
    }
  }

  private checkSequentialApproval(approvals: any[]): boolean {
    // All required approvals in sequence must be approved
    for (const approval of approvals) {
      if (approval.status !== 'approved') {
        return false;
      }
    }
    return approvals.length > 0;
  }

  private checkConditionalApproval(approvals: any[], expense: any): boolean {
    const totalApprovals = approvals.length;
    const approvedCount = approvals.filter(a => a.status === 'approved').length;
    const approvalPercentage = (approvedCount / totalApprovals) * 100;

    // Check percentage rule
    if (approvalPercentage >= expense.approval_percentage) {
      return true;
    }

    // Check specific approver rule
    if (expense.specific_approver_rule) {
      const specificApproverIds = expense.specific_approver_rule.approver_ids || [];
      const hasSpecificApproval = approvals.some(a => 
        a.status === 'approved' && specificApproverIds.includes(a.approver_id)
      );
      if (hasSpecificApproval) {
        return true;
      }
    }

    return false;
  }

  private checkHybridApproval(approvals: any[], expense: any): boolean {
    // Check both percentage and specific approver rules
    const percentageApproved = this.checkConditionalApproval(approvals, expense);
    
    if (expense.specific_approver_rule) {
      const specificApproverIds = expense.specific_approver_rule.approver_ids || [];
      const hasSpecificApproval = approvals.some(a => 
        a.status === 'approved' && specificApproverIds.includes(a.approver_id)
      );
      return percentageApproved || hasSpecificApproval;
    }

    return percentageApproved;
  }

  private async convertCurrency(amount: number, fromCurrency: string, companyId: string): Promise<number> {
    if (fromCurrency === await this.getCompanyCurrency(companyId)) {
      return amount;
    }

    // Get exchange rate from API or database cache
    const exchangeRate = await this.getExchangeRate(fromCurrency, await this.getCompanyCurrency(companyId));
    return amount * exchangeRate;
  }

  private async getCompanyCurrency(companyId: string): Promise<string> {
    const result = await this.db.query(
      'SELECT default_currency FROM companies WHERE id = $1',
      [companyId]
    );
    return result.rows[0].default_currency;
  }

  private async getExchangeRate(fromCurrency: string, toCurrency: string): Promise<number> {
    // Check cache first
    const cachedRate = await this.getCachedExchangeRate(fromCurrency, toCurrency);
    if (cachedRate) {
      return cachedRate;
    }

    // Fetch from API
    const response = await fetch(https://api.exchangerate-api.com/v4/latest/${fromCurrency});
    const data = await response.json();
    const rate = data.rates[toCurrency];

    // Cache the rate
    await this.cacheExchangeRate(fromCurrency, toCurrency, rate);

    return rate;
  }

  private async getCachedExchangeRate(fromCurrency: string, toCurrency: string): Promise<number | null> {
    const result = await this.db.query(
      `SELECT exchange_rate FROM currency_rates 
       WHERE base_currency = $1 AND target_currency = $2 
       AND effective_date = CURRENT_DATE`,
      [fromCurrency, toCurrency]
    );
    return result.rows[0]?.exchange_rate || null;
  }

  private async cacheExchangeRate(fromCurrency: string, toCurrency: string, rate: number): Promise<void> {
    await this.db.query(
      `INSERT INTO currency_rates (id, base_currency, target_currency, exchange_rate, effective_date)
       VALUES ($1, $2, $3, $4, CURRENT_DATE)`,
      [uuidv4(), fromCurrency, toCurrency, rate]
    );
  }
}

// OCR Service
export class OCRService {
  async processReceipt(imageBuffer: Buffer): Promise<any> {
    // Integration with OCR API (Google Vision, AWS Textract, etc.)
    // This is a mock implementation
    return {
      amount: 150.00,
      currency: 'USD',
      date: new Date(),
      description: 'Business Lunch - Client Meeting',
      merchant: 'Restaurant Name',
      items: [
        { description: 'Main Course', amount: 75.00 },
        { description: 'Beverages', amount: 25.00 },
        { description: 'Tax', amount: 10.00 },
        { description: 'Tip', amount: 40.00 }
      ]
    };
  }
}

// Auth Service
export class AuthService {
  constructor(private db: Pool) {}

  async signUp(email: string, password: string, countryCode: string): Promise<{user: User, company: Company}> {
    const client = await this.db.connect();
    
    try {
      await client.query('BEGIN');

      // Get country currency
      const currency = await this.getCountryCurrency(countryCode);

      // Create company
      const companyQuery = `
        INSERT INTO companies (id, name, country_code, default_currency)
        VALUES ($1, $2, $3, $4)
        RETURNING *
      `;
      const companyResult = await client.query(companyQuery, [
        uuidv4(), ${email.split('@')[1]} Company, countryCode, currency
      ]);
      const company = companyResult.rows[0];

      // Create admin user
      const userQuery = `
        INSERT INTO users (id, company_id, email, password_hash, role)
        VALUES ($1, $2, $3, $4, $5)
        RETURNING *
      `;
      const userResult = await client.query(userQuery, [
        uuidv4(), company.id, email, await this.hashPassword(password), 'admin'
      ]);
      const user = userResult.rows[0];

      await client.query('COMMIT');
      return { user, company };
    } catch (error) {
      await client.query('ROLLBACK');
      throw error;
    } finally {
      client.release();
    }
  }

  private async getCountryCurrency(countryCode: string): Promise<string> {
    const response = await fetch(https://restcountries.com/v3.1/alpha/${countryCode});
    const data = await response.json();
    return Object.keys(data[0].currencies)[0];
  }

  private async hashPassword(password: string): Promise<string> {
    // Implement password hashing (bcrypt)
    return password; // Placeholder
  }
}



4. Frontend Components(React/TypeScript)

    // ExpenseSubmission.tsx
import React, { useState } from 'react';
import { ExpenseService, OCRService } from './services';

interface ExpenseSubmissionProps {
  employeeId: string;
  companyId: string;
}

export const ExpenseSubmission: React.FC<ExpenseSubmissionProps> = ({ employeeId, companyId }) => {
  const [formData, setFormData] = useState({
    amount: '',
    currency: 'USD',
    category_id: '',
    description: '',
    expense_date: new Date().toISOString().split('T')[0]
  });
  const [receiptImage, setReceiptImage] = useState<File | null>(null);
  const [isProcessing, setIsProcessing] = useState(false);

  const handleReceiptUpload = async (event: React.ChangeEvent<HTMLInputElement>) => {
    const file = event.target.files?.[0];
    if (!file) return;

    setReceiptImage(file);
    setIsProcessing(true);

    try {
      const ocrService = new OCRService();
      const ocrResult = await ocrService.processReceipt(await file.arrayBuffer());
      
      setFormData(prev => ({
        ...prev,
        amount: ocrResult.amount.toString(),
        currency: ocrResult.currency,
        description: ocrResult.description
      }));
    } catch (error) {
      console.error('OCR processing failed:', error);
    } finally {
      setIsProcessing(false);
    }
  };

  const handleSubmit = async (event: React.FormEvent) => {
    event.preventDefault();
    
    const expenseData = {
      company_id: companyId,
      employee_id: employeeId,
      amount: parseFloat(formData.amount),
      currency: formData.currency,
      category_id: formData.category_id,
      description: formData.description,
      expense_date: new Date(formData.expense_date),
      receipt_url: receiptImage ? URL.createObjectURL(receiptImage) : undefined
    };

    const expenseService = new ExpenseService();
    await expenseService.createExpense(expenseData);
    
    // Reset form
    setFormData({
      amount: '',
      currency: 'USD',
      category_id: '',
      description: '',
      expense_date: new Date().toISOString().split('T')[0]
    });
    setReceiptImage(null);
  };

  return (
    <div className="expense-submission">
      <h2>Submit Expense</h2>
      <form onSubmit={handleSubmit}>
        <div className="form-group">
          <label>Receipt Upload</label>
          <input 
            type="file" 
            accept="image/*" 
            onChange={handleReceiptUpload}
            disabled={isProcessing}
          />
          {isProcessing && <div>Processing receipt...</div>}
        </div>

        <div className="form-group">
          <label>Amount</label>
          <input
            type="number"
            step="0.01"
            value={formData.amount}
            onChange={e => setFormData(prev => ({ ...prev, amount: e.target.value }))}
            required
          />
        </div>

        <div className="form-group">
          <label>Currency</label>
          <select
            value={formData.currency}
            onChange={e => setFormData(prev => ({ ...prev, currency: e.target.value }))}
          >
            <option value="USD">USD</option>
            <option value="EUR">EUR</option>
            <option value="GBP">GBP</option>
          </select>
        </div>

        <div className="form-group">
          <label>Description</label>
          <textarea
            value={formData.description}
            onChange={e => setFormData(prev => ({ ...prev, description: e.target.value }))}
            required
          />
        </div>

        <button type="submit" disabled={isProcessing}>
          Submit Expense
        </button>
      </form>
    </div>
  );
};

// ApprovalDashboard.tsx
import React, { useState, useEffect } from 'react';

interface ApprovalDashboardProps {
  approverId: string;
}

export const ApprovalDashboard: React.FC<ApprovalDashboardProps> = ({ approverId }) => {
  const [pendingApprovals, setPendingApprovals] = useState<any[]>([]);

  useEffect(() => {
    loadPendingApprovals();
  }, [approverId]);

  const loadPendingApprovals = async () => {
    // API call to fetch pending approvals
    const response = await fetch(/api/approvals/pending?approver_id=${approverId});
    const data = await response.json();
    setPendingApprovals(data);
  };

  const handleApproval = async (expenseId: string, approved: boolean, comments?: string) => {
    await fetch('/api/approvals/decision', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ expense_id: expenseId, approved, comments })
    });
    
    loadPendingApprovals(); // Refresh list
  };

  return (
    <div className="approval-dashboard">
      <h2>Pending Approvals</h2>
      <div className="approval-list">
        {pendingApprovals.map(approval => (
          <div key={approval.id} className="approval-item">
            <div className="expense-details">
              <h3>${approval.converted_amount} {approval.currency}</h3>
              <p>{approval.description}</p>
              <small>Submitted by: {approval.employee_name}</small>
              <small>Date: {new Date(approval.expense_date).toLocaleDateString()}</small>
            </div>
            
            <div className="approval-actions">
              <button 
                onClick={() => handleApproval(approval.expense_id, true)}
                className="approve-btn"
              >
                Approve
              </button>
              <button 
                onClick={() => handleApproval(approval.expense_id, false)}
                className="reject-btn"
              >
                Reject
              </button>
            </div>
          </div>
        ))}
      </div>
    </div>
  );
};









5. API Routes(Express.js)

// routes/expenses.ts
import express from 'express';
import { ExpenseService, AuthService } from '../services';

const router = express.Router();
const expenseService = new ExpenseService();
const authService = new AuthService();

// Submit expense
router.post('/', async (req, res) => {
  try {
    const expense = await expenseService.createExpense(req.body);
    res.json(expense);
  } catch (error) {
    res.status(500).json({ error: 'Failed to create expense' });
  }
});

// Get user expenses
router.get('/user/:userId', async (req, res) => {
  try {
    const expenses = await expenseService.getUserExpenses(req.params.userId);
    res.json(expenses);
  } catch (error) {
    res.status(500).json({ error: 'Failed to fetch expenses' });
  }
});

// routes/approvals.ts
router.get('/pending', async (req, res) => {
  try {
    const approvals = await expenseService.getPendingApprovals(req.query.approver_id as string);
    res.json(approvals);
  } catch (error) {
    res.status(500).json({ error: 'Failed to fetch pending approvals' });
  }
});

router.post('/decision', async (req, res) => {
  try {
    const { expense_id, approved, comments } = req.body;
    
    if (approved) {
      await expenseService.approveExpense(expense_id, req.body.approver_id, comments);
    } else {
      await expenseService.rejectExpense(expense_id, req.body.approver_id, comments);
    }
    
    res.json({ success: true });
  } catch (error) {
    res.status(500).json({ error: 'Failed to process approval' });
  }
});

// routes/auth.ts
router.post('/signup', async (req, res) => {
  try {
    const { email, password, country_code } = req.body;
    const result = await authService.signUp(email, password, country_code);
    res.json(result);
  } catch (error) {
    res.status(500).json({ error: 'Failed to create account' });
  }
});



6.  Configuration and Setup

     // app.ts
import express from 'express';
import { Pool } from 'pg';
import expenseRoutes from './routes/expenses';
import approvalRoutes from './routes/approvals';
import authRoutes from './routes/auth';

const app = express();
const port = process.env.PORT || 3000;

// Database connection
const db = new Pool({
  user: process.env.DB_USER,
  host: process.env.DB_HOST,
  database: process.env.DB_NAME,
  password: process.env.DB_PASSWORD,
  port: parseInt(process.env.DB_PORT || '5432'),
});

app.use(express.json());

// Routes
app.use('/api/expenses', expenseRoutes);
app.use('/api/approvals', approvalRoutes);
app.use('/api/auth', authRoutes);

app.listen(port, () => {
  console.log(Expense management system running on port ${port});
});
CREATE INDEX idx_expense_approvals_expense ON expense_approvals(expense_id);
CREATE INDEX idx_expense_approvals_approver ON expense_approvals(approver_id);
CREATE INDEX idx_currency_rates_date ON currency_rates(effective_date);
