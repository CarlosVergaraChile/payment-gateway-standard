# Payment Gateway Standard

Universal payment gateway abstraction layer for all CarlosVergaraChile projects. Provides a consistent, cost-efficient interface for integrating multiple payment providers with support for both local (CLP) and international currencies.

## Overview

This library standardizes payment processing across all projects, eliminating the need to re-implement gateway integrations for each new application. It provides a unified API that abstracts the complexity of different payment providers.

## Supported Providers

### Primary (Recommended)
- **Flow (flow.cl)** - Local Chilean gateway, lowest fees, best for CLP transactions
- **Global66** - International support, good for global reach with multiple currency support

### Secondary
- **PayPal** - Worldwide coverage, established brand, higher fees
- **Mercado Pago** - Regional coverage in Latin America, competitive fees

### Future Support
- Stripe (not available in Chile currently)
- Reveniu (subscription management)
- Khipu (alternative local option)
- Payku (alternative local option)

## Architecture

### Core Abstraction

```typescript
interface PaymentProvider {
  // Create a one-time payment link
  createPayment(config: PaymentConfig): Promise<PaymentLink>;
  
  // Create or manage subscriptions
  createSubscription(config: SubscriptionConfig): Promise<SubscriptionLink>;
  
  // Handle webhook callbacks from provider
  handleWebhook(payload: any): Promise<WebhookResult>;
  
  // Verify transaction status
  verifyTransaction(transactionId: string): Promise<TransactionStatus>;
}
```

### Directory Structure

```
.
├── src/
│   ├── core/
│   │   ├── PaymentGateway.ts       # Main orchestration interface
│   │   ├── types.ts                # TypeScript interfaces
│   │   └── utils.ts                # Helper functions
│   ├── providers/
│   │   ├── flow.ts                 # Flow.cl implementation
│   │   ├── global66.ts             # Global66 implementation
│   │   ├── paypal.ts               # PayPal implementation
│   │   └── mercadopago.ts          # Mercado Pago implementation
│   ├── webhooks/
│   │   └── handler.ts              # Webhook processing middleware
│   └── index.ts                    # Main export
├── .env.example                    # Environment template
├── README.md
└── package.json
```

## Installation

```bash
npm install payment-gateway-standard
# or
yarn add payment-gateway-standard
```

## Configuration

Create a `.env` file based on `.env.example`:

```env
# Flow (CLP - Recommended for Chilean market)
FLOW_MERCHANT_ID=your_merchant_id
FLOW_API_KEY=your_api_key
FLOW_WEBHOOK_KEY=your_webhook_key

# Global66 (International)
GLOBAL66_API_KEY=your_api_key
GLOBAL66_WEBHOOK_SECRET=your_webhook_secret

# PayPal (Fallback/International)
PAYPAL_CLIENT_ID=your_client_id
PAYPAL_CLIENT_SECRET=your_secret

# Mercado Pago (Regional)
MERCADO_PAGO_ACCESS_TOKEN=your_access_token

# Application
APP_ENVIRONMENT=production
BASE_URL=https://yourdomain.com
WEBHOOK_PATH=/api/webhooks/payments
```

## Usage

### Basic Payment Creation

```javascript
import { PaymentGateway } from 'payment-gateway-standard';

const gateway = new PaymentGateway({
  provider: 'flow',           // or 'global66', 'paypal', 'mercadopago'
  environment: 'production',
  credentials: {
    merchantId: process.env.FLOW_MERCHANT_ID,
    apiKey: process.env.FLOW_API_KEY
  }
});

// Create a payment
const payment = await gateway.createPayment({
  amount: 9990,              // CLP (without decimals)
  currency: 'CLP',
  description: 'SAM v3.0 - 100 créditos',
  email: 'student@example.com',
  returnUrl: 'https://yourdomain.com/payment/success',
  notifyUrl: 'https://yourdomain.com/api/webhooks/payments'
});

console.log(payment.paymentUrl); // Redirect user to this URL
```

### Subscription Management

```javascript
const subscription = await gateway.createSubscription({
  amount: 4990,              // Monthly in CLP
  currency: 'CLP',
  period: 'MONTHLY',
  description: 'SAM v3.0 - Suscripción Premium',
  email: 'teacher@example.com',
  maxCharges: 12             // Optional: limit number of charges
});
```

### Webhook Handling

```javascript
import { webhookHandler } from 'payment-gateway-standard';
import express from 'express';

const app = express();

app.post('/api/webhooks/payments', async (req, res) => {
  try {
    const result = await webhookHandler.process({
      provider: req.body.provider || 'flow',
      payload: req.body,
      secret: process.env.FLOW_WEBHOOK_KEY
    });
    
    if (result.isValid) {
      // Update your database
      await updateUserCredits(result.userId, result.creditsAdded);
      res.json({ status: 'processed' });
    } else {
      res.status(400).json({ error: 'Invalid signature' });
    }
  } catch (error) {
    console.error('Webhook error:', error);
    res.status(500).json({ error: 'Webhook processing failed' });
  }
});
```

## Integration Guide for Projects

### For SAM v3.0 and other applications:

1. **Install the library:**
   ```bash
   npm install payment-gateway-standard
   ```

2. **Add to your payment module:**
   ```javascript
   import { PaymentGateway } from 'payment-gateway-standard';
   
   export const paymentGateway = new PaymentGateway({
     provider: process.env.PAYMENT_PROVIDER || 'flow',
     environment: process.env.NODE_ENV,
     credentials: {
       merchantId: process.env.FLOW_MERCHANT_ID,
       apiKey: process.env.FLOW_API_KEY,
       // ... other credentials based on provider
     }
   });
   ```

3. **Use in payment routes:**
   ```javascript
   app.post('/api/buy-credits', async (req, res) => {
     const { credits, email } = req.body;
     const amount = calculatePrice(credits);
     
     const payment = await paymentGateway.createPayment({
       amount,
       currency: 'CLP',
       description: `${credits} créditos SAM v3.0`,
       email,
       returnUrl: `${process.env.BASE_URL}/dashboard?payment=success`,
       notifyUrl: `${process.env.BASE_URL}/api/webhooks/payments`
     });
     
     res.json({ paymentUrl: payment.paymentUrl });
   });
   ```

## Cost Analysis

### Recommended Setup: Flow + Global66

**Flow (Primary for CLP)**
- Commission: 1.49% (local transfers)
- Fixed: None
- Settlement: 1 business day
- Best for: Chilean market, UX

**Global66 (Secondary for International)**
- Commission: Variable (typically 2-3%)
- Fixed: None
- Settlement: 2-3 business days
- Best for: International expansion, multiple currencies

**Example cost for 10,000 CLP transaction:**
- Flow: ~150 CLP
- Global66: ~200-300 CLP
- PayPal: ~350 CLP
- Mercado Pago: ~250 CLP

## Future Enhancements

- [ ] Webhook retry logic
- [ ] Transaction logging and audit trail
- [ ] Rate limiting
- [ ] Multi-currency conversion
- [ ] Batch payment processing
- [ ] Admin dashboard integration
- [ ] Analytics and reporting

## Support

For questions or issues, create an issue in this repository.

## License

MIT
