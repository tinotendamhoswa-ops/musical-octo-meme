import React, { useState, useEffect } from 'react';
import { Plus, Send, DollarSign, Clock, CheckCircle, AlertCircle, Copy, Download, Upload, Trash2, Edit2, TrendingUp, Mail } from 'lucide-react';

export default function InvoiceNudge() {
  const [invoices, setInvoices] = useState([]);
  const [showAddForm, setShowAddForm] = useState(false);
  const [editingId, setEditingId] = useState(null);
  const [formData, setFormData] = useState({
    clientName: '',
    clientEmail: '',
    amount: '',
    invoiceNumber: '',
    dueDate: '',
    description: ''
  });

  useEffect(() => {
    const saved = localStorage.getItem('invoiceNudgeData');
    if (saved) {
      setInvoices(JSON.parse(saved));
    }
  }, []);

  useEffect(() => {
    localStorage.setItem('invoiceNudgeData', JSON.stringify(invoices));
  }, [invoices]);

  const addOrUpdateInvoice = () => {
    if (!formData.clientName || !formData.amount || !formData.dueDate) return;

    const invoice = {
      id: editingId || Date.now(),
      ...formData,
      amount: parseFloat(formData.amount),
      status: editingId ? invoices.find(i => i.id === editingId).status : 'sent',
      createdAt: editingId ? invoices.find(i => i.id === editingId).createdAt : new Date().toISOString()
    };

    if (editingId) {
      setInvoices(invoices.map(i => i.id === editingId ? invoice : i));
      setEditingId(null);
    } else {
      setInvoices([...invoices, invoice]);
    }

    setFormData({ clientName: '', clientEmail: '', amount: '', invoiceNumber: '', dueDate: '', description: '' });
    setShowAddForm(false);
  };

  const deleteInvoice = (id) => {
    if (confirm('Delete this invoice?')) {
      setInvoices(invoices.filter(i => i.id !== id));
    }
  };

  const editInvoice = (invoice) => {
    setFormData({
      clientName: invoice.clientName,
      clientEmail: invoice.clientEmail,
      amount: invoice.amount.toString(),
      invoiceNumber: invoice.invoiceNumber,
      dueDate: invoice.dueDate,
      description: invoice.description || ''
    });
    setEditingId(invoice.id);
    setShowAddForm(true);
  };

  const updateStatus = (id, status) => {
    setInvoices(invoices.map(i => i.id === id ? { ...i, status } : i));
  };

  const getDaysOverdue = (dueDate) => {
    const due = new Date(dueDate);
    const today = new Date();
    const diff = Math.floor((today - due) / (1000 * 60 * 60 * 24));
    return diff;
  };

  const getAutoStatus = (invoice) => {
    if (invoice.status === 'paid') return 'paid';
    const days = getDaysOverdue(invoice.dueDate);
    if (days > 0) return 'overdue';
    return 'sent';
  };

  const generateReminderEmail = (invoice) => {
    const daysOverdue = getDaysOverdue(invoice.dueDate);
    const lateFee = daysOverdue > 7 ? (invoice.amount * 0.05).toFixed(2) : 0;
    
    let template = '';
    
    if (daysOverdue <= 3) {
      template = `Subject: Friendly Reminder: Invoice ${invoice.invoiceNumber} Due

Hi ${invoice.clientName},

I hope this email finds you well! This is a friendly reminder that Invoice ${invoice.invoiceNumber} for $${invoice.amount.toFixed(2)} was due on ${new Date(invoice.dueDate).toLocaleDateString()}.

${invoice.description ? `Service: ${invoice.description}\n\n` : ''}If you've already sent payment, please disregard this message. Otherwise, I'd appreciate if you could process this at your earliest convenience.

Thank you for your business!

Best regards`;
    } else if (daysOverdue <= 14) {
      template = `Subject: Payment Reminder: Invoice ${invoice.invoiceNumber} - ${daysOverdue} Days Overdue

Hi ${invoice.clientName},

I wanted to follow up on Invoice ${invoice.invoiceNumber} for $${invoice.amount.toFixed(2)}, which is now ${daysOverdue} days overdue (due date: ${new Date(invoice.dueDate).toLocaleDateString()}).

${invoice.description ? `Service: ${invoice.description}\n\n` : ''}Could you please provide an update on the payment status? If there are any issues or questions about this invoice, I'm happy to discuss.

${lateFee > 0 ? `Please note: A 5% late fee ($${lateFee}) will apply after 7 days overdue.\n\n` : ''}Looking forward to your response.

Best regards`;
    } else {
      template = `Subject: URGENT: Invoice ${invoice.invoiceNumber} - ${daysOverdue} Days Overdue

Hi ${invoice.clientName},

This is an urgent reminder that Invoice ${invoice.invoiceNumber} for $${invoice.amount.toFixed(2)} is now ${daysOverdue} days overdue (original due date: ${new Date(invoice.dueDate).toLocaleDateString()}).

${invoice.description ? `Service: ${invoice.description}\n\n` : ''}Total Amount Due: $${(invoice.amount + parseFloat(lateFee)).toFixed(2)} (including $${lateFee} late fee)

I need immediate payment or communication regarding this outstanding invoice. If payment is not received within 5 business days, I may need to pursue further collection action.

Please contact me immediately to resolve this matter.

Best regards`;
    }
    
    return template;
  };

  const copyEmail = (invoice) => {
    const email = generateReminderEmail(invoice);
    navigator.clipboard.writeText(email);
    alert('✨ Email copied to clipboard!');
  };

  const exportData = () => {
    const dataStr = JSON.stringify(invoices, null, 2);
    const dataBlob = new Blob([dataStr], { type: 'application/json' });
    const url = URL.createObjectURL(dataBlob);
    const link = document.createElement('a');
    link.href = url;
    link.download = `invoices-${new Date().toISOString().split('T')[0]}.json`;
    link.click();
  };

  const importData = (e) => {
    const file = e.target.files[0];
    if (!file) return;
    
    const reader = new FileReader();
    reader.onload = (e) => {
      try {
        const data = JSON.parse(e.target.result);
        setInvoices(data);
        alert('✨ Data imported successfully!');
      } catch (err) {
        alert('❌ Error importing data. Please check the file format.');
      }
    };
    reader.readAsText(file);
  };

  const stats = {
    total: invoices.reduce((sum, i) => sum + (i.status !== 'paid' ? i.amount : 0), 0),
    overdue: invoices.filter(i => getAutoStatus(i) === 'overdue' && i.status !== 'paid').reduce((sum, i) => sum + i.amount, 0),
    sent: invoices.filter(i => i.status === 'sent' && getAutoStatus(i) !== 'overdue').length,
    overdueCount: invoices.filter(i => getAutoStatus(i) === 'overdue' && i.status !== 'paid').length,
    paid: invoices.filter(i => i.status === 'paid').reduce((sum, i) => sum + i.amount, 0)
  };

  return (
    <div style={{
      minHeight: '100vh',
      background: 'linear-gradient(135deg, #f5f7fa 0%, #e8ecf1 100%)',
      fontFamily: '-apple-system, BlinkMacSystemFont, "Segoe UI", Roboto, "Helvetica Neue", Arial, sans-serif',
      padding: '24px'
    }}>
      {/* Header */}
      <div style={{
        maxWidth: '1400px',
        margin: '0 auto',
        marginBottom: '32px'
      }}>
        <div style={{ display: 'flex', alignItems: 'center', justifyContent: 'space-between', marginBottom: '8px', flexWrap: 'wrap', gap: '16px' }}>
          <div>
            <h1 style={{
              fontSize: '40px',
              fontWeight: '800',
              margin: 0,
              background: 'linear-gradient(135deg, #667eea 0%, #764ba2 100%)',
              WebkitBackgroundClip: 'text',
              WebkitTextFillColor: 'transparent',
              letterSpacing: '-0.5px'
            }}>
              InvoiceNudge ✨
            </h1>
            <p style={{ fontSize: '15px', color: '#64748b', margin: '8px 0 0 0', fontWeight: '500' }}>
              Track invoices, automate reminders, get paid faster
            </p>
          </div>
          <div style={{ display: 'flex', gap: '12px' }}>
            <button
              onClick={exportData}
              style={{
                background: 'white',
                border: '2px solid #e2e8f0',
                borderRadius: '12px',
                padding: '10px 20px',
                color: '#475569',
                cursor: 'pointer',
                display: 'flex',
                alignItems: 'center',
                gap: '8px',
                fontSize: '14px',
                fontWeight: '600',
                transition: 'all 0.2s',
                boxShadow: '0 1px 3px rgba(0,0,0,0.05)'
              }}
              onMouseOver={(e) => e.currentTarget.style.transform = 'translateY(-2px)'}
              onMouseOut={(e) => e.currentTarget.style.transform = 'translateY(0)'}
            >
              <Download size={16} />
              Export
            </button>
            <label style={{
              background: 'white',
              border: '2px solid #e2e8f0',
              borderRadius: '12px',
              padding: '10px 20px',
              color: '#475569',
              cursor: 'pointer',
              display: 'flex',
              alignItems: 'center',
              gap: '8px',
              fontSize: '14px',
              fontWeight: '600',
              transition: 'all 0.2s',
              boxShadow: '0 1px 3px rgba(0,0,0,0.05)'
            }}>
              <Upload size={16} />
              Import
              <input type="file" accept=".json" onChange={importData} style={{ display: 'none' }} />
            </label>
          </div>
        </div>
      </div>

      {/* Stats Cards */}
      <div style={{
        maxWidth: '1400px',
        margin: '0 auto 32px',
        display: 'grid',
        gridTemplateColumns: 'repeat(auto-fit, minmax(240px, 1fr))',
        gap: '20px'
      }}>
        <div style={{
          background: 'linear-gradient(135deg, #667eea 0%, #764ba2 100%)',
          borderRadius: '20px',
          padding: '28px',
          boxShadow: '0 10px 40px rgba(102, 126, 234, 0.3)',
          transition: 'transform 0.2s'
        }}
        onMouseOver={(e) => e.currentTarget.style.transform = 'translateY(-4px)'}
        onMouseOut={(e) => e.currentTarget.style.transform = 'translateY(0)'}>
          <div style={{ display: 'flex', alignItems: 'center', gap: '16px' }}>
            <div style={{
              width: '56px',
              height: '56px',
              borderRadius: '16px',
              background: 'rgba(255, 255, 255, 0.2)',
              display: 'flex',
              alignItems: 'center',
              justifyContent: 'center',
              color: 'white'
            }}>
              <DollarSign size={28} strokeWidth={2.5} />
            </div>
            <div>
              <div style={{ fontSize: '14px', color: 'rgba(255,255,255,0.9)', fontWeight: '600', marginBottom: '4px' }}>Outstanding</div>
              <div style={{ fontSize: '32px', fontWeight: '800', color: 'white', letterSpacing: '-1px' }}>
                ${stats.total.toFixed(0)}
              </div>
            </div>
          </div>
        </div>

        <div style={{
          background: 'linear-gradient(135deg, #f093fb 0%, #f5576c 100%)',
          borderRadius: '20px',
          padding: '28px',
          boxShadow: '0 10px 40px rgba(245, 87, 108, 0.3)',
          transition: 'transform 0.2s'
        }}
        onMouseOver={(e) => e.currentTarget.style.transform = 'translateY(-4px)'}
        onMouseOut={(e) => e.currentTarget.style.transform = 'translateY(0)'}>
          <div style={{ display: 'flex', alignItems: 'center', gap: '16px' }}>
            <div style={{
              width: '56px',
              height: '56px',
              borderRadius: '16px',
              background: 'rgba(255, 255, 255, 0.2)',
              display: 'flex',
              alignItems: 'center',
              justifyContent: 'center',
              color: 'white'
            }}>
              <AlertCircle size={28} strokeWidth={2.5} />
            </div>
            <div>
              <div style={{ fontSize: '14px', color: 'rgba(255,255,255,0.9)', fontWeight: '600', marginBottom: '4px' }}>Overdue</div>
              <div style={{ fontSize: '32px', fontWeight: '800', color: 'white', letterSpacing: '-1px' }}>
                ${stats.overdue.toFixed(0)}
              </div>
            </div>
          </div>
        </div>

        <div style={{
          background: 'linear-gradient(135deg, #4facfe 0%, #00f2fe 100%)',
          borderRadius: '20px',
          padding: '28px',
          boxShadow: '0 10px 40px rgba(79, 172, 254, 0.3)',
          transition: 'transform 0.2s'
        }}
        onMouseOver={(e) => e.currentTarget.style.transform = 'translateY(-4px)'}
        onMouseOut={(e) => e.currentTarget.style.transform = 'translateY(0)'}>
          <div style={{ display: 'flex', alignItems: 'center', gap: '16px' }}>
            <div style={{
              width: '56px',
              height: '56px',
              borderRadius: '16px',
              background: 'rgba(255, 255, 255, 0.2)',
              display: 'flex',
              alignItems: 'center',
              justifyContent: 'center',
              color: 'white'
            }}>
              <Clock size={28} strokeWidth={2.5} />
            </div>
            <div>
              <div style={{ fontSize: '14px', color: 'rgba(255,255,255,0.9)', fontWeight: '600', marginBottom: '4px' }}>Pending</div>
              <div style={{ fontSize: '32px', fontWeight: '800', color: 'white', letterSpacing: '-1px' }}>
                {stats.sent}
              </div>
            </div>
          </div>
        </div>

        <div style={{
          background: 'linear-gradient(135deg, #43e97b 0%, #38f9d7 100%)',
          borderRadius: '20px',
          padding: '28px',
          boxShadow: '0 10px 40px rgba(67, 233, 123, 0.3)',
          transition: 'transform 0.2s'
        }}
        onMouseOver={(e) => e.currentTarget.style.transform = 'translateY(-4px)'}
        onMouseOut={(e) => e.currentTarget.style.transform = 'translateY(0)'}>
          <div style={{ display: 'flex', alignItems: 'center', gap: '16px' }}>
            <div style={{
              width: '56px',
              height: '56px',
              borderRadius: '16px',
              background: 'rgba(255, 255, 255, 0.2)',
              display: 'flex',
              alignItems: 'center',
              justifyContent: 'center',
              color: 'white'
            }}>
              <CheckCircle size={28} strokeWidth={2.5} />
            </div>
            <div>
              <div style={{ fontSize: '14px', color: 'rgba(255,255,255,0.9)', fontWeight: '600', marginBottom: '4px' }}>Paid Total</div>
              <div style={{ fontSize: '32px', fontWeight: '800', color: 'white', letterSpacing: '-1px' }}>
                ${stats.paid.toFixed(0)}
              </div>
            </div>
          </div>
        </div>
      </div>

      {/* Add Button */}
      <div style={{ maxWidth: '1400px', margin: '0 auto 24px' }}>
        <button
          onClick={() => {
            setShowAddForm(!showAddForm);
            setEditingId(null);
            setFormData({ clientName: '', clientEmail: '', amount: '', invoiceNumber: '', dueDate: '', description: '' });
          }}
          style={{
            background: 'linear-gradient(135deg, #667eea 0%, #764ba2 100%)',
            border: 'none',
            borderRadius: '16px',
            padding: '16px 32px',
            color: 'white',
            fontSize: '15px',
            fontWeight: '700',
            cursor: 'pointer',
            display: 'flex',
            alignItems: 'center',
            gap: '10px',
            boxShadow: '0 8px 24px rgba(102, 126, 234, 0.4)',
            transition: 'all 0.2s'
          }}
          onMouseOver={(e) => {
            e.currentTarget.style.transform = 'translateY(-2px)';
            e.currentTarget.style.boxShadow = '0 12px 32px rgba(102, 126, 234, 0.5)';
          }}
          onMouseOut={(e) => {
            e.currentTarget.style.transform = 'translateY(0)';
            e.currentTarget.style.boxShadow = '0 8px 24px rgba(102, 126, 234, 0.4)';
          }}
        >
          <Plus size={20} strokeWidth={3} />
          Create New Invoice
        </button>
      </div>

      {/* Add/Edit Form */}
      {showAddForm && (
        <div style={{
          maxWidth: '1400px',
          margin: '0 auto 32px',
          background: 'white',
          borderRadius: '24px',
          padding: '32px',
          boxShadow: '0 20px 60px rgba(0,0,0,0.1)',
          border: '2px solid #e2e8f0'
        }}>
          <h3 style={{ margin: '0 0 24px 0', fontSize: '24px', fontWeight: '700', color: '#1e293b' }}>
            {editingId ? '✏️ Edit Invoice' : '✨ New Invoice'}
          </h3>
          <div style={{ display: 'grid', gridTemplateColumns: 'repeat(auto-fit, minmax(280px, 1fr))', gap: '20px' }}>
            <div>
              <label style={{ display: 'block', fontSize: '13px', fontWeight: '600', color: '#475569', marginBottom: '8px' }}>
                Client Name *
              </label>
              <input
                type="text"
                placeholder="John Doe"
                value={formData.clientName}
                onChange={(e) => setFormData({ ...formData, clientName: e.target.value })}
                style={{
                  width: '100%',
                  background: '#f8fafc',
                  border: '2px solid #e2e8f0',
                  borderRadius: '12px',
                  padding: '14px 16px',
                  color: '#1e293b',
                  fontSize: '15px',
                  outline: 'none',
                  transition: 'all 0.2s',
                  boxSizing: 'border-box'
                }}
                onFocus={(e) => e.currentTarget.style.borderColor = '#667eea'}
                onBlur={(e) => e.currentTarget.style.borderColor = '#e2e8f0'}
              />
            </div>
            <div>
              <label style={{ display: 'block', fontSize: '13px', fontWeight: '600', color: '#475569', marginBottom: '8px' }}>
                Client Email
              </label>
              <input
                type="email"
                placeholder="john@company.com"
                value={formData.clientEmail}
                onChange={(e) => setFormData({ ...formData, clientEmail: e.target.value })}
                style={{
                  width: '100%',
                  background: '#f8fafc',
                  border: '2px solid #e2e8f0',
                  borderRadius: '12px',
                  padding: '14px 16px',
                  color: '#1e293b',
                  fontSize: '15px',
                  outline: 'none',
                  transition: 'all 0.2s',
                  boxSizing: 'border-box'
                }}
                onFocus={(e) => e.currentTarget.style.borderColor = '#667eea'}
                onBlur={(e) => e.currentTarget.style.borderColor = '#e2e8f0'}
              />
            </div>
            <div>
              <label style={{ display: 'block', fontSize: '13px', fontWeight: '600', color: '#475569', marginBottom: '8px' }}>
                Amount *
              </label>
              <input
                type="number"
                placeholder="500.00"
                value={formData.amount}
                onChange={(e) => setFormData({ ...formData, amount: e.target.value })}
                style={{
                  width: '100%',
                  background: '#f8fafc',
                  border: '2px solid #e2e8f0',
                  borderRadius: '12px',
                  padding: '14px 16px',
                  color: '#1e293b',
                  fontSize: '15px',
                  outline: 'none',
                  transition: 'all 0.2s',
                  boxSizing: 'border-box'
                }}
                onFocus={(e) => e.currentTarget.style.borderColor = '#667eea'}
                onBlur={(e) => e.currentTarget.style.borderColor = '#e2e8f0'}
              />
            </div>
            <div>
              <label style={{ display: 'block', fontSize: '13px', fontWeight: '600', color: '#475569', marginBottom: '8px' }}>
                Invoice Number
              </label>
              <input
           
