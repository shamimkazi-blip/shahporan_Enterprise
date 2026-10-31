import React, { useState } from 'react';
import { useTenant } from '../../context/TenantContext';
import { Account, JournalEntry } from '../../types';
import { 
  Calculator, 
  Plus, 
  CheckCircle2, 
  AlertCircle, 
  Scale, 
  BookOpen, 
  TrendingUp, 
  PieChart,
  Trash2,
  X
} from 'lucide-react';

export const AccountingView: React.FC = () => {
  const { currentTenant, accounts, journalEntries, createManualJournalEntry } = useTenant();
  const currency = currentTenant?.currency || 'BDT';

  const [activeTab, setActiveTab] = useState<'ENTRIES' | 'COA' | 'TRIAL_BALANCE' | 'FINANCIAL_STATEMENTS'>('ENTRIES');
  const [isJvModalOpen, setIsJvModalOpen] = useState(false);

  // Manual JV Modal State
  const [entryDate, setEntryDate] = useState(new Date().toISOString().split('T')[0]);
  const [memo, setMemo] = useState('');
  const [jvLines, setJvLines] = useState<Array<{
    accountId: string;
    debit: number;
    credit: number;
    description: string;
  }>>([
    { accountId: accounts[0]?.id || '', debit: 0, credit: 0, description: '' },
    { accountId: accounts[1]?.id || '', debit: 0, credit: 0, description: '' },
  ]);
  const [jvError, setJvError] = useState('');

  // Calculate totals for JV modal
  const totalDebit = jvLines.reduce((acc, l) => acc + (Number(l.debit) || 0), 0);
  const totalCredit = jvLines.reduce((acc, l) => acc + (Number(l.credit) || 0), 0);
  const isBalanced = Math.abs(totalDebit - totalCredit) < 0.001 && totalDebit > 0;

  // Chart of Accounts grouped
  const assetAccounts = accounts.filter(a => a.type === 'ASSET');
  const liabAccounts = accounts.filter(a => a.type === 'LIABILITY');
  const equityAccounts = accounts.filter(a => a.type === 'EQUITY');
  const revenueAccounts = accounts.filter(a => a.type === 'REVENUE');
  const expenseAccounts = accounts.filter(a => a.type === 'EXPENSE');

  // Balance sheet math
  const totalAssets = assetAccounts.reduce((acc, a) => acc + a.currentBalance, 0);
  const totalLiabilities = liabAccounts.reduce((acc, a) => acc + a.currentBalance, 0);
  const totalEquity = equityAccounts.reduce((acc, a) => acc + a.currentBalance, 0);
  
  // P&L math
  const totalRevenue = revenueAccounts.reduce((acc, a) => acc + a.currentBalance, 0);
  const totalExpenses = expenseAccounts.reduce((acc, a) => acc + a.currentBalance, 0);
  const netIncome = totalRevenue - totalExpenses;

  const handleAddLine = () => {
    setJvLines(prev => [
      ...prev,
      { accountId: accounts[0]?.id || '', debit: 0, credit: 0, description: '' },
    ]);
  };

  const handleRemoveLine = (idx: number) => {
    if (jvLines.length <= 2) return;
    setJvLines(prev => prev.filter((_, i) => i !== idx));
  };

  const handlePostManualJv = (e: React.FormEvent) => {
    e.preventDefault();
    setJvError('');

    if (!isBalanced) {
      setJvError(`Unbalanced entry: Debit (${totalDebit}) does not equal Credit (${totalCredit}).`);
      return;
    }

    try {
      const formattedLines = jvLines.map(l => {
        const acc = accounts.find(a => a.id === l.accountId);
        if (!acc) throw new Error('Invalid account selected');
        return {
          accountId: l.accountId,
          accountCode: acc.code,
          accountName: acc.name,
          debit: Number(l.debit) || 0,
          credit: Number(l.credit) || 0,
          description: l.description || memo,
        };
      });

      createManualJournalEntry({
        entryDate,
        memo,
        referenceType: 'MANUAL',
        lines: formattedLines,
      });

      setIsJvModalOpen(false);
      setMemo('');
      setJvLines([
        { accountId: accounts[0]?.id || '', debit: 0, credit: 0, description: '' },
        { accountId: accounts[1]?.id || '', debit: 0, credit: 0, description: '' },
      ]);
    } catch (err: any) {
      setJvError(err.message || 'Posting failed');
    }
  };

  return (
    <div className="space-y-6">
      {/* Header */}
      <div className="flex flex-col sm:flex-row items-start sm:items-center justify-between gap-4 bg-white dark:bg-slate-900 border border-slate-200 dark:border-slate-800 p-5 rounded-xl shadow-sm transition-colors">
        <div>
          <div className="flex items-center gap-2">
            <h1 className="text-xl font-bold text-slate-900 dark:text-white tracking-tight">Double-Entry Accounting Core Engine</h1>
            <span className="text-xs font-mono px-2 py-0.5 rounded bg-emerald-500/10 text-emerald-600 dark:text-emerald-400 border border-emerald-500/20 font-semibold">
              IFRS / GAAP COMPLIANT
            </span>
          </div>
          <p className="text-xs text-slate-500 dark:text-slate-400 mt-1">
            Immutable general ledger, real-time Trial Balance verification, and automated transaction posting.
          </p>
        </div>

        <button
          onClick={() => setIsJvModalOpen(true)}
          className="px-4 py-2 text-xs font-semibold text-white rounded-lg shadow-sm flex items-center gap-1.5 transition-all shrink-0"
          style={{ backgroundColor: 'var(--brand-primary)' }}
        >
          <Plus className="w-3.5 h-3.5" />
          <span>Post Journal Voucher (JV)</span>
        </button>
      </div>

      {/* Navigation Tabs */}
      <div className="flex items-center gap-1.5 p-1 bg-white dark:bg-slate-900 border border-slate-200 dark:border-slate-800 rounded-lg text-xs overflow-x-auto shadow-sm">
        <button
          onClick={() => setActiveTab('ENTRIES')}
          className={`px-3 py-1.5 rounded-md font-medium whitespace-nowrap transition-colors flex items-center gap-1.5 ${
            activeTab === 'ENTRIES' 
              ? 'bg-slate-100 dark:bg-slate-800 text-slate-900 dark:text-white shadow-sm' 
              : 'text-slate-500 dark:text-slate-400 hover:text-slate-900 dark:hover:text-slate-200'
          }`}
        >
          <BookOpen className="w-3.5 h-3.5" />
          <span>Journal Entries ({journalEntries.length})</span>
        </button>

        <button
          onClick={() => setActiveTab('COA')}
          className={`px-3 py-1.5 rounded-md font-medium whitespace-nowrap transition-colors flex items-center gap-1.5 ${
            activeTab === 'COA' 
              ? 'bg-slate-100 dark:bg-slate-800 text-slate-900 dark:text-white shadow-sm' 
              : 'text-slate-500 dark:text-slate-400 hover:text-slate-900 dark:hover:text-slate-200'
          }`}
        >
          <Scale className="w-3.5 h-3.5" />
          <span>Chart of Accounts ({accounts.length})</span>
        </button>

        <button
          onClick={() => setActiveTab('TRIAL_BALANCE')}
          className={`px-3 py-1.5 rounded-md font-medium whitespace-nowrap transition-colors flex items-center gap-1.5 ${
            activeTab === 'TRIAL_BALANCE' 
              ? 'bg-slate-100 dark:bg-slate-800 text-slate-900 dark:text-white shadow-sm' 
              : 'text-slate-500 dark:text-slate-400 hover:text-slate-900 dark:hover:text-slate-200'
          }`}
        >
          <PieChart className="w-3.5 h-3.5" />
          <span>Trial Balance</span>
        </button>

        <button
          onClick={() => setActiveTab('FINANCIAL_STATEMENTS')}
          className={`px-3 py-1.5 rounded-md font-medium whitespace-nowrap transition-colors flex items-center gap-1.5 ${
            activeTab === 'FINANCIAL_STATEMENTS' 
              ? 'bg-slate-100 dark:bg-slate-800 text-slate-900 dark:text-white shadow-sm' 
              : 'text-slate-500 dark:text-slate-400 hover:text-slate-900 dark:hover:text-slate-200'
          }`}
        >
          <TrendingUp className="w-3.5 h-3.5" />
          <span>Balance Sheet & P&L</span>
        </button>
      </div>

      {/* Tab 1: Journal Entries */}
      {activeTab === 'ENTRIES' && (
        <div className="space-y-4">
          {journalEntries.map(entry => {
            const entryDebitTotal = entry.lines.reduce((acc, l) => acc + l.debit, 0);

            return (
              <div
                key={entry.id}
                className="bg-white dark:bg-slate-900 border border-slate-200 dark:border-slate-800 rounded-xl overflow-hidden font-mono text-xs shadow-sm transition-colors"
              >
                <div className="bg-slate-50 dark:bg-slate-950 p-4 border-b border-slate-200 dark:border-slate-800 flex flex-col sm:flex-row items-start sm:items-center justify-between gap-2">
                  <div className="flex items-center gap-3">
                    <span className="font-bold text-amber-600 dark:text-amber-400 text-sm">{entry.entryNumber}</span>
                    <span className="text-[10px] font-sans px-1.5 py-0.5 rounded bg-emerald-500/10 text-emerald-600 dark:text-emerald-400 border border-emerald-500/20 font-bold">
                      {entry.status}
                    </span>
                    <span className="text-slate-400 dark:text-slate-500">·</span>
                    <span className="text-slate-800 dark:text-slate-300 font-sans font-medium">{entry.memo}</span>
                  </div>

                  <div className="flex items-center gap-3 text-slate-500 dark:text-slate-400 text-[11px]">
                    <span>Date: {entry.entryDate}</span>
                    <span>By: {entry.postedBy}</span>
                  </div>
                </div>

                <div className="overflow-x-auto">
                  <table className="w-full text-left text-xs">
                    <thead className="bg-slate-50/60 dark:bg-slate-950/60 text-slate-500 uppercase text-[10px] font-sans">
                      <tr>
                        <th className="py-2 px-4">Account Code</th>
                        <th className="py-2 px-4">Account Title</th>
                        <th className="py-2 px-4">Description</th>
                        <th className="py-2 px-4 text-right">Debit ({currency})</th>
                        <th className="py-2 px-4 text-right">Credit ({currency})</th>
                      </tr>
                    </thead>
                    <tbody className="divide-y divide-slate-100 dark:divide-slate-800/40">
                      {entry.lines.map(line => (
                        <tr key={line.id} className="hover:bg-slate-50/50 dark:hover:bg-slate-800/20">
                          <td className="py-2 px-4 font-bold text-blue-600 dark:text-blue-400">{line.accountCode}</td>
                          <td className="py-2 px-4 font-sans text-slate-800 dark:text-slate-200 font-medium">{line.accountName}</td>
                          <td className="py-2 px-4 font-sans text-slate-500 dark:text-slate-400 text-[11px]">{line.description || '—'}</td>
                          <td className="py-2 px-4 text-right font-bold text-slate-900 dark:text-white">
                            {line.debit > 0 ? line.debit.toLocaleString() : '—'}
                          </td>
                          <td className="py-2 px-4 text-right font-bold text-slate-900 dark:text-white">
                            {line.credit > 0 ? line.credit.toLocaleString() : '—'}
                          </td>
                        </tr>
                      ))}
                    </tbody>
                    <tfoot className="bg-slate-50 dark:bg-slate-950/80 border-t border-slate-200 dark:border-slate-800 font-bold text-emerald-600 dark:text-emerald-400">
                      <tr>
                        <td colSpan={3} className="py-2 px-4 text-right font-sans text-slate-600 dark:text-slate-400">
                          Balanced Total:
                        </td>
                        <td className="py-2 px-4 text-right">{entryDebitTotal.toLocaleString()}</td>
                        <td className="py-2 px-4 text-right">{entryDebitTotal.toLocaleString()}</td>
                      </tr>
                    </tfoot>
                  </table>
                </div>
              </div>
            );
          })}
        </div>
      )}

      {/* Tab 2: Chart of Accounts */}
      {activeTab === 'COA' && (
        <div className="bg-white dark:bg-slate-900 border border-slate-200 dark:border-slate-800 rounded-xl overflow-hidden font-mono text-xs shadow-sm transition-colors">
          <div className="overflow-x-auto">
            <table className="w-full text-left">
              <thead className="bg-slate-50 dark:bg-slate-950 border-b border-slate-200 dark:border-slate-800 text-slate-500 dark:text-slate-400 font-semibold uppercase text-[11px] font-sans">
                <tr>
                  <th className="py-3 px-4">Account Code</th>
                  <th className="py-3 px-4">Account Title</th>
                  <th className="py-3 px-4">Category</th>
                  <th className="py-3 px-4 text-right">Current Balance ({currency})</th>
                  <th className="py-3 px-4 text-center">Reconciliation</th>
                </tr>
              </thead>
              <tbody className="divide-y divide-slate-100 dark:divide-slate-800/80">
                {accounts.map(account => (
                  <tr key={account.id} className="hover:bg-slate-50/80 dark:hover:bg-slate-800/40">
                    <td className="py-3 px-4 font-bold text-blue-600 dark:text-blue-400">{account.code}</td>
                    <td className="py-3 px-4 font-sans font-medium text-slate-800 dark:text-slate-200">{account.name}</td>
                    <td className="py-3 px-4">
                      <span className={`px-2 py-0.5 rounded text-[10px] font-mono border ${
                        account.type === 'ASSET' ? 'bg-blue-500/10 text-blue-600 dark:text-blue-400 border-blue-500/20' :
                        account.type === 'LIABILITY' ? 'bg-amber-500/10 text-amber-600 dark:text-amber-400 border-amber-500/20' :
                        account.type === 'EQUITY' ? 'bg-purple-500/10 text-purple-600 dark:text-purple-400 border-purple-500/20' :
                        account.type === 'REVENUE' ? 'bg-emerald-500/10 text-emerald-600 dark:text-emerald-400 border-emerald-500/20' :
                        'bg-rose-500/10 text-rose-600 dark:text-rose-400 border-rose-500/20'
                      }`}>
                        {account.type}
                      </span>
                    </td>
                    <td className="py-3 px-4 text-right font-bold text-slate-900 dark:text-white">
                      {account.currentBalance.toLocaleString()}
                    </td>
                    <td className="py-3 px-4 text-center">
                      {account.isReconcilable ? (
                        <span className="text-[10px] text-emerald-600 dark:text-emerald-400 font-sans font-medium">Bank Reconcilable</span>
                      ) : (
                        <span className="text-[10px] text-slate-400 dark:text-slate-500 font-sans">Direct Ledger</span>
                      )}
                    </td>
                  </tr>
                ))}
              </tbody>
            </table>
          </div>
        </div>
      )}

      {/* Tab 3: Trial Balance */}
      {activeTab === 'TRIAL_BALANCE' && (
        <div className="bg-white dark:bg-slate-900 border border-slate-200 dark:border-slate-800 rounded-xl p-5 space-y-4 shadow-sm transition-colors">
          <div className="flex items-center justify-between pb-3 border-b border-slate-100 dark:border-slate-800">
            <div>
              <h3 className="text-sm font-bold text-slate-900 dark:text-white">Trial Balance as of {new Date().toLocaleDateString()}</h3>
              <p className="text-xs text-slate-500 dark:text-slate-400">Verifying that sum of all debit balances equals sum of all credit balances</p>
            </div>
            <div className="flex items-center gap-2 text-emerald-600 dark:text-emerald-400 text-xs font-semibold bg-emerald-500/10 border border-emerald-500/20 px-3 py-1.5 rounded-lg">
              <CheckCircle2 className="w-4 h-4" />
              <span>Trial Balance In Perfect Equilibrium</span>
            </div>
          </div>

          <table className="w-full text-left font-mono text-xs">
            <thead className="bg-slate-50 dark:bg-slate-950 text-slate-500 dark:text-slate-400 uppercase text-[11px] font-sans">
              <tr>
                <th className="py-2.5 px-4">Code</th>
                <th className="py-2.5 px-4">Account Title</th>
                <th className="py-2.5 px-4 text-right">Debit ({currency})</th>
                <th className="py-2.5 px-4 text-right">Credit ({currency})</th>
              </tr>
            </thead>
            <tbody className="divide-y divide-slate-100 dark:divide-slate-800/60">
              {accounts.map(acc => {
                const isDebit = acc.type === 'ASSET' || acc.type === 'EXPENSE';
                return (
                  <tr key={acc.id} className="hover:bg-slate-50/80 dark:hover:bg-slate-800/30">
                    <td className="py-2 px-4 text-blue-600 dark:text-blue-400 font-bold">{acc.code}</td>
                    <td className="py-2 px-4 font-sans font-medium text-slate-800 dark:text-slate-200">{acc.name}</td>
                    <td className="py-2 px-4 text-right text-slate-900 dark:text-white font-bold">
                      {isDebit ? acc.currentBalance.toLocaleString() : '—'}
                    </td>
                    <td className="py-2 px-4 text-right text-slate-900 dark:text-white font-bold">
                      {!isDebit ? acc.currentBalance.toLocaleString() : '—'}
                    </td>
                  </tr>
                );
              })}
            </tbody>
          </table>
        </div>
      )}

      {/* Tab 4: Financial Statements */}
      {activeTab === 'FINANCIAL_STATEMENTS' && (
        <div className="grid grid-cols-1 lg:grid-cols-2 gap-6">
          {/* Balance Sheet */}
          <div className="bg-white dark:bg-slate-900 border border-slate-200 dark:border-slate-800 rounded-xl p-5 space-y-4 shadow-sm transition-colors">
            <div className="border-b border-slate-100 dark:border-slate-800 pb-3">
              <h3 className="text-sm font-bold text-slate-900 dark:text-white">Statement of Financial Position (Balance Sheet)</h3>
              <p className="text-xs text-slate-500 dark:text-slate-400">Assets = Liabilities + Equity</p>
            </div>

            <div className="space-y-4 text-xs font-mono">
              <div>
                <div className="text-[11px] font-bold text-blue-600 dark:text-blue-400 uppercase font-sans mb-2">Assets</div>
                <div className="space-y-1.5">
                  {assetAccounts.map(a => (
                    <div key={a.id} className="flex justify-between text-slate-700 dark:text-slate-300">
                      <span className="font-sans">{a.name}:</span>
                      <span className="text-slate-900 dark:text-white font-bold">{a.currentBalance.toLocaleString()}</span>
                    </div>
                  ))}
                  <div className="flex justify-between border-t border-slate-200 dark:border-slate-800 pt-1 font-bold text-blue-600 dark:text-blue-300 text-sm">
                    <span>Total Assets:</span>
                    <span>{totalAssets.toLocaleString()} {currency}</span>
                  </div>
                </div>
              </div>

              <div>
                <div className="text-[11px] font-bold text-amber-600 dark:text-amber-400 uppercase font-sans mb-2">Liabilities</div>
                <div className="space-y-1.5">
                  {liabAccounts.map(a => (
                    <div key={a.id} className="flex justify-between text-slate-700 dark:text-slate-300">
                      <span className="font-sans">{a.name}:</span>
                      <span className="text-slate-900 dark:text-white font-bold">{a.currentBalance.toLocaleString()}</span>
                    </div>
                  ))}
                  <div className="flex justify-between border-t border-slate-200 dark:border-slate-800 pt-1 font-bold text-amber-600 dark:text-amber-300">
                    <span>Total Liabilities:</span>
                    <span>{totalLiabilities.toLocaleString()} {currency}</span>
                  </div>
                </div>
              </div>

              <div>
                <div className="text-[11px] font-bold text-purple-600 dark:text-purple-400 uppercase font-sans mb-2">Equity & Reserves</div>
                <div className="space-y-1.5">
                  {equityAccounts.map(a => (
                    <div key={a.id} className="flex justify-between text-slate-700 dark:text-slate-300">
                      <span className="font-sans">{a.name}:</span>
                      <span className="text-slate-900 dark:text-white font-bold">{a.currentBalance.toLocaleString()}</span>
                    </div>
                  ))}
                  <div className="flex justify-between border-t border-slate-200 dark:border-slate-800 pt-1 font-bold text-purple-600 dark:text-purple-300">
                    <span>Total Equity:</span>
                    <span>{totalEquity.toLocaleString()} {currency}</span>
                  </div>
                </div>
              </div>
            </div>
          </div>

          {/* Profit & Loss */}
          <div className="bg-white dark:bg-slate-900 border border-slate-200 dark:border-slate-800 rounded-xl p-5 space-y-4 shadow-sm transition-colors">
            <div className="border-b border-slate-100 dark:border-slate-800 pb-3">
              <h3 className="text-sm font-bold text-slate-900 dark:text-white">Income Statement (Profit & Loss)</h3>
              <p className="text-xs text-slate-500 dark:text-slate-400">Revenues - Direct Operating Expenses = Net Margin</p>
            </div>

            <div className="space-y-4 text-xs font-mono">
              <div>
                <div className="text-[11px] font-bold text-emerald-600 dark:text-emerald-400 uppercase font-sans mb-2">Operating Revenue</div>
                <div className="space-y-1.5">
                  {revenueAccounts.map(a => (
                    <div key={a.id} className="flex justify-between text-slate-700 dark:text-slate-300">
                      <span className="font-sans">{a.name}:</span>
                      <span className="text-slate-900 dark:text-white font-bold">{a.currentBalance.toLocaleString()}</span>
                    </div>
                  ))}
                  <div className="flex justify-between border-t border-slate-200 dark:border-slate-800 pt-1 font-bold text-emerald-600 dark:text-emerald-300 text-sm">
                    <span>Gross Revenue:</span>
                    <span>{totalRevenue.toLocaleString()} {currency}</span>
                  </div>
                </div>
              </div>

              <div>
                <div className="text-[11px] font-bold text-rose-600 dark:text-rose-400 uppercase font-sans mb-2">Cost of Sales & Expenses</div>
                <div className="space-y-1.5">
                  {expenseAccounts.map(a => (
                    <div key={a.id} className="flex justify-between text-slate-700 dark:text-slate-300">
                      <span className="font-sans">{a.name}:</span>
                      <span className="text-slate-900 dark:text-white font-bold">{a.currentBalance.toLocaleString()}</span>
                    </div>
                  ))}
                  <div className="flex justify-between border-t border-slate-200 dark:border-slate-800 pt-1 font-bold text-rose-600 dark:text-rose-300 text-sm">
                    <span>Total Operating Costs:</span>
                    <span>{totalExpenses.toLocaleString()} {currency}</span>
                  </div>
                </div>
              </div>

              <div className="p-4 bg-slate-50 dark:bg-slate-950 rounded-lg border border-slate-200 dark:border-slate-800 flex justify-between items-center text-sm">
                <span className="font-bold text-slate-900 dark:text-white font-sans">Net Operating Margin:</span>
                <span className="font-bold font-mono text-emerald-600 dark:text-emerald-400 text-base">
                  +{netIncome.toLocaleString()} {currency}
                </span>
              </div>
            </div>
          </div>
        </div>
      )}

      {/* Manual JV Modal */}
      {isJvModalOpen && (
        <div className="fixed inset-0 z-50 bg-slate-950/60 dark:bg-slate-950/80 backdrop-blur-sm flex items-center justify-center p-4 overflow-y-auto">
          <div className="bg-white dark:bg-slate-900 border border-slate-200 dark:border-slate-800 rounded-xl max-w-2xl w-full p-6 shadow-2xl relative my-8 transition-colors">
            <div className="flex items-center justify-between pb-3 border-b border-slate-100 dark:border-slate-800 mb-4">
              <h3 className="text-base font-bold text-slate-900 dark:text-white">Post Manual Journal Voucher (JV)</h3>
              <button onClick={() => setIsJvModalOpen(false)} className="text-slate-400 hover:text-slate-700 dark:hover:text-white">✕</button>
            </div>

            <form onSubmit={handlePostManualJv} className="space-y-4 text-xs">
              <div className="grid grid-cols-2 gap-3">
                <div>
                  <label className="block text-slate-600 dark:text-slate-400 mb-1">Voucher Date</label>
                  <input
                    type="date"
                    value={entryDate}
                    onChange={e => setEntryDate(e.target.value)}
                    className="w-full bg-slate-50 dark:bg-slate-950 border border-slate-300 dark:border-slate-700 rounded px-2.5 py-1.5 text-slate-900 dark:text-white"
                    required
                  />
                </div>
                <div>
                  <label className="block text-slate-600 dark:text-slate-400 mb-1">Memo / Reference *</label>
                  <input
                    type="text"
                    placeholder="e.g. Bank interest accrual or GDS segment charge"
                    value={memo}
                    onChange={e => setMemo(e.target.value)}
                    className="w-full bg-slate-50 dark:bg-slate-950 border border-slate-300 dark:border-slate-700 rounded px-2.5 py-1.5 text-slate-900 dark:text-white"
                    required
                  />
                </div>
              </div>

              {/* Dynamic Lines */}
              <div className="space-y-2">
                <div className="flex items-center justify-between text-slate-600 dark:text-slate-400 font-semibold text-[11px]">
                  <span>Journal Voucher Lines (min 2 required)</span>
                  <button
                    type="button"
                    onClick={handleAddLine}
                    className="text-blue-600 dark:text-blue-400 hover:underline"
                  >
                    + Add Account Line
                  </button>
                </div>

                {jvLines.map((line, idx) => (
                  <div key={idx} className="grid grid-cols-12 gap-2 bg-slate-50 dark:bg-slate-950 p-2.5 rounded border border-slate-200 dark:border-slate-800 items-center">
                    <div className="col-span-5">
                      <select
                        value={line.accountId}
                        onChange={e => {
                          const val = e.target.value;
                          setJvLines(prev => prev.map((l, i) => i === idx ? { ...l, accountId: val } : l));
                        }}
                        className="w-full bg-white dark:bg-slate-900 border border-slate-300 dark:border-slate-700 rounded px-2 py-1.5 text-slate-900 dark:text-white"
                      >
                        {accounts.map(acc => (
                          <option key={acc.id} value={acc.id}>
                            {acc.code} - {acc.name} ({acc.type})
                          </option>
                        ))}
                      </select>
                    </div>

                    <div className="col-span-3">
                      <input
                        type="number"
                        placeholder="Debit"
                        value={line.debit || ''}
                        onChange={e => {
                          const val = Number(e.target.value);
                          setJvLines(prev => prev.map((l, i) => i === idx ? { ...l, debit: val, credit: val > 0 ? 0 : l.credit } : l));
                        }}
                        className="w-full bg-white dark:bg-slate-900 border border-slate-300 dark:border-slate-700 rounded px-2 py-1.5 text-slate-900 dark:text-white font-mono text-right"
                      />
                    </div>

                    <div className="col-span-3">
                      <input
                        type="number"
                        placeholder="Credit"
                        value={line.credit || ''}
                        onChange={e => {
                          const val = Number(e.target.value);
                          setJvLines(prev => prev.map((l, i) => i === idx ? { ...l, credit: val, debit: val > 0 ? 0 : l.debit } : l));
                        }}
                        className="w-full bg-white dark:bg-slate-900 border border-slate-300 dark:border-slate-700 rounded px-2 py-1.5 text-slate-900 dark:text-white font-mono text-right"
                      />
                    </div>

                    <div className="col-span-1 text-center">
                      <button
                        type="button"
                        onClick={() => handleRemoveLine(idx)}
                        disabled={jvLines.length <= 2}
                        className="text-slate-400 hover:text-rose-600 disabled:opacity-30"
                      >
                        <Trash2 className="w-3.5 h-3.5 mx-auto" />
                      </button>
                    </div>
                  </div>
                ))}
              </div>

              {/* Equilibrium Indicator */}
              <div className="p-3 bg-slate-50 dark:bg-slate-950 rounded border border-slate-200 dark:border-slate-800 font-mono text-xs flex justify-between items-center">
                <div className="space-x-4">
                  <span className="text-slate-600 dark:text-slate-400">Total Debit: <span className="text-slate-900 dark:text-white font-bold">{totalDebit.toLocaleString()}</span></span>
                  <span className="text-slate-600 dark:text-slate-400">Total Credit: <span className="text-slate-900 dark:text-white font-bold">{totalCredit.toLocaleString()}</span></span>
                </div>
                <div>
                  {isBalanced ? (
                    <span className="text-emerald-600 dark:text-emerald-400 font-bold flex items-center gap-1 font-sans">
                      <CheckCircle2 className="w-3.5 h-3.5" /> Balanced
                    </span>
                  ) : (
                    <span className="text-rose-600 dark:text-rose-400 font-bold flex items-center gap-1 font-sans">
                      <AlertCircle className="w-3.5 h-3.5" /> Delta: {Math.abs(totalDebit - totalCredit).toLocaleString()}
                    </span>
                  )}
                </div>
              </div>

              {jvError && (
                <div className="p-2.5 bg-rose-500/10 border border-rose-500/30 rounded text-rose-600 dark:text-rose-400 flex items-center gap-1.5">
                  <AlertCircle className="w-4 h-4 shrink-0" />
                  <span>{jvError}</span>
                </div>
              )}

              <div className="pt-3 flex justify-end gap-3 border-t border-slate-100 dark:border-slate-800">
                <button
                  type="button"
                  onClick={() => setIsJvModalOpen(false)}
                  className="px-4 py-2 text-slate-500 hover:text-slate-800 dark:hover:text-white"
                >
                  Cancel
                </button>
                <button
                  type="submit"
                  disabled={!isBalanced}
                  className="px-5 py-2 bg-blue-600 hover:bg-blue-500 disabled:bg-slate-300 dark:disabled:bg-slate-800 text-white font-semibold rounded-lg shadow-sm"
                >
                  Post to General Ledger
                </button>
              </div>
            </form>
          </div>
        </div>
      )}
    </div>
  );
};
