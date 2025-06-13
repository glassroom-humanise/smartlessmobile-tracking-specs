# SmartlessMobile - Tracking Implementation Guide

## Google Analytics 4 & Google Tag Manager Integration

This guide provides complete implementation details for tracking user interactions and conversions across the SmartlessMobile platform.

---

## Configuration Details

**Google Analytics 4:**
- **Measurement ID:** `G-04WQBRC1X9`
- **API Secret:** Retrieve from GA4 Admin Panel → Data Streams → Measurement Protocol API secrets

---

## Implementation Overview

### 1. Page Tracking

#### A. Landing Page Attribution
```javascript
// Only capture attribution data on external entry points
function captureAttribution() {
  const isExternalReferrer = !document.referrer || 
    !document.referrer.includes(window.location.hostname);
  const hasUtmParams = window.location.search.includes('utm_');
  const hasClickIds = window.location.search.includes('gclid') || 
    window.location.search.includes('fbclid');
  
  if (isExternalReferrer || hasUtmParams || hasClickIds) {
    const attributionData = {
      traffic_source: getTrafficSource(),
      campaign_id: getUrlParameter('utm_campaign'),
      ad_group_id: getUrlParameter('utm_term'),
      creative_id: getUrlParameter('utm_content'),
      gclid: getUrlParameter('gclid'),
      fbclid: getUrlParameter('fbclid'),
      landing_page: window.location.pathname,
      timestamp: new Date().toISOString()
    };
    
    // Store for session
    sessionStorage.setItem('attribution_data', JSON.stringify(attributionData));
    
    // Send to analytics
    dataLayer.push({
      'event': 'attribution_capture',
      ...attributionData
    });
  }
}

// Run on page load
captureAttribution();
```


#### B. User Identification
```javascript
// Send user data after login/registration events. Trigger this on every pageview once user is logged in.
function identifyUser(userId, email, phone) {
  dataLayer.push({
    'event': 'user_identified',
    'user_id': userId,
    'user_email_hash': hashString(email),
    'user_phone_hash': hashString(phone),
    'attribution_data': JSON.parse(sessionStorage.getItem('attribution_data') || '{}')
  });
}
```

### 2. User Authentication Events

#### Login Event (`/login` page)
```javascript
window.dataLayer = window.dataLayer || [];
window.dataLayer.push({
  'event': 'login',
  'user_id': '1234567', // Replace with actual user ID
  'user_email': 'x0f5552030a723243cc2f42b38a53b6fb794c', // SHA256 hashed
  'user_phone_number': '8a8f59b70e74b46b21d8aa405c5' // SHA256 hashed
});
```

#### Account Creation Event (`/create-account` page)
```javascript
window.dataLayer = window.dataLayer || [];
window.dataLayer.push({
  'event': 'account_created',
  'user_id': '1234567', // Replace with actual user ID
  'user_email': 'x0f5552030a723243cc2f42b38a53b6fb794c', // SHA256 hashed
  'user_phone_number': '8a8f59b70e74b46b21d8aa405c5' // SHA256 hashed
});
```

---

## Ecommerce Event Tracking

### View Plans Page (`/plans`)
```javascript
dataLayer.push({
  event: "view_item_list",
  items: [
    {
      item_id: "10gb_plan",
      item_name: "10GB Plan",
      item_category: "subscription",
      price: 21.5,
      currency: "USD",
      quantity: 1
    },
    {
      item_id: "30gb_plan",
      item_name: "30GB Plan",
      item_category: "subscription",
      price: 31.75,
      currency: "USD",
      quantity: 1
    }
  ],
  item_list_id: "pricing_plans",
  item_list_name: "Subscription Plans"
});
```

### Plan Selection (`/plans` → `/selected-plan`)
```javascript
dataLayer.push({
  event: "select_item",
  items: [
    {
      item_id: "10gb_plan",
      item_name: "10GB Plan",
      item_category: "subscription",
      item_variant: "eSIM",
      price: 21.5,
      currency: "USD",
      quantity: 1
    }
  ]
});
```

### Add to Cart (`/selected-plan` → `/my-cart`)
```javascript
dataLayer.push({
  event: "add_to_cart",
  items: [
    {
      item_id: "10gb_plan",
      item_name: "10GB Plan",
      item_category: "subscription",
      item_variant: "eSIM",
      price: 21.5,
      currency: "USD",
      quantity: 1
    }
  ],
  currency: "USD",
  value: 21.5
});
```

### Begin Checkout (`/my-cart` → `/address`)
```javascript
dataLayer.push({
  event: "begin_checkout",
  items: [
    {
      item_id: "10gb_plan",
      item_name: "10GB Plan",
      item_category: "subscription",
      item_variant: "eSIM",
      price: 21.5,
      currency: "USD",
      quantity: 1
    }
  ],
  currency: "USD",
  value: 21.5
});
```

### Add Shipping Info (`/address` → `/payment-method`)
```javascript
dataLayer.push({
  event: "add_shipping_info",
  items: [
    {
      item_id: "10gb_plan",
      item_name: "10GB Plan",
      item_category: "subscription",
      item_variant: "SIM Kit", // Note: Use "SIM Kit" for physical delivery
      price: 21.5,
      currency: "USD",
      quantity: 1
    }
  ],
  currency: "USD",
  value: 21.5
});
```

### Add Payment Info (`/payment-method`)
```javascript
dataLayer.push({
  event: "add_payment_info",
  items: [
    {
      item_id: "10gb_plan",
      item_name: "10GB Plan",
      item_category: "subscription",
      item_variant: "eSIM",
      price: 21.5,
      currency: "USD",
      quantity: 1
    }
  ],
  currency: "USD",
  value: 21.5,
  payment_type: "Credit Card"
});
```

### Purchase Confirmation (`/order-confirmed`) 🎯 **CONVERSION EVENT**
```javascript
dataLayer.push({
  event: "purchase",
  items: [
    {
      item_id: "10gb_plan",
      item_name: "10GB Plan",
      item_category: "subscription",
      item_variant: "monthly - eSIM",
      price: 21.5,
      currency: "USD",
      quantity: 1
    }
  ],
  transaction_id: "ORD123456", // Unique order ID
  affiliation: "Smartless Mobile Online",
  value: 21.5,
  tax: 0.0,
  shipping: 0.0,
  currency: "USD",
  coupon: "" // Include coupon code if applicable
});
```

---

## Server-Side Tracking (Backend Implementation)

### SIM Activation Event
When a SIM card is activated in your backend system, send this event via the Measurement Protocol:

```javascript
const axios = require('axios');

// Configuration
const MEASUREMENT_ID = 'G-04WQBRC1X9';
const API_SECRET = 'your_api_secret'; // Get from GA4 Admin Panel

// Event payload
const payload = {
  client_id: '123456789.987654321', // From user's CRM record
  user_id: 'user_123',
  events: [
    {
      name: 'sim_activated',
      params: {
        sim_type: 'eSIM', // or 'Physical SIM'
        plan_name: '10GB Plan',
        activation_date: '2025-06-12',
        value: 0,
        currency: 'USD'
      }
    }
  ]
};

// Send event
axios.post(
  `https://www.google-analytics.com/mp/collect?measurement_id=${MEASUREMENT_ID}&api_secret=${API_SECRET}`,
  payload
).then(response => {
  console.log('SIM activation event sent successfully');
}).catch(error => {
  console.error('Error sending event:', error.response?.data || error.message);
});
```

### Multi-Platform SIM Activation Tracking
For optimal tracking, we would also need to send this event to Google Ads and Facebook Ads. This is not mandatory and should not be prioritized. 

#### Complete Implementation Function
```javascript
const axios = require('axios');
const crypto = require('crypto');

// Platform Configuration
const PLATFORMS = {
  GA4: {
    measurement_id: 'G-04WQBRC1X9',
    api_secret: 'your_ga4_api_secret' // Get from GA4 Admin Panel
  },
  GOOGLE_ADS: {
    conversion_id: 'AW-XXXXXXXXX', // Your Google Ads Conversion ID
    conversion_label: 'your_conversion_label', // From Google Ads
    developer_token: 'your_developer_token' // Google Ads API
  },
  FACEBOOK: {
    pixel_id: 'your_facebook_pixel_id',
    access_token: 'your_facebook_access_token' // Facebook Conversions API
  }
};

/**
 * Send SIM activation event to all platforms
 * @param {Object} userData - User and activation data from CRM
 */
async function sendSimActivationEvent(userData) {
  const {
    client_id,
    user_id,
    email,
    phone,
    sim_type,
    plan_name,
    plan_value,
    activation_date,
    gclid,
    fbclid,
    user_ip,
    user_agent
  } = userData;

  // Hash user data for privacy
  const hashedEmail = hashSHA256(email);
  const hashedPhone = hashSHA256(phone);

  // Send to all platforms simultaneously
  const results = await Promise.allSettled([
    sendToGA4(client_id, user_id, sim_type, plan_name, plan_value, activation_date),
    sendToGoogleAds(gclid, hashedEmail, hashedPhone, plan_value, user_ip, user_agent),
    sendToFacebookAds(fbclid, hashedEmail, hashedPhone, plan_value, user_ip, user_agent)
  ]);

  // Log results
  results.forEach((result, index) => {
    const platform = ['GA4', 'Google Ads', 'Facebook Ads'][index];
    if (result.status === 'fulfilled') {
      console.log(`✅ ${platform}: SIM activation event sent successfully`);
    } else {
      console.error(`❌ ${platform}: Error sending event:`, result.reason);
    }
  });
}

// 1. Send to Google Analytics 4
async function sendToGA4(client_id, user_id, sim_type, plan_name, plan_value, activation_date) {
  const payload = {
    client_id: client_id,
    user_id: user_id,
    events: [
      {
        name: 'sim_activated',
        params: {
          sim_type: sim_type,
          plan_name: plan_name,
          activation_date: activation_date,
          value: plan_value || 0,
          currency: 'USD',
          event_category: 'engagement',
          event_label: 'post_purchase_activation'
        }
      }
    ]
  };

  return axios.post(
    `https://www.google-analytics.com/mp/collect?measurement_id=${PLATFORMS.GA4.measurement_id}&api_secret=${PLATFORMS.GA4.api_secret}`,
    payload
  );
}

// 2. Send to Google Ads (Enhanced Conversions)
async function sendToGoogleAds(gclid, hashedEmail, hashedPhone, conversionValue, userIP, userAgent) {
  // Only send if we have a Google Ads click ID
  if (!gclid) {
    console.log('⚠️ Google Ads: No gclid found, skipping conversion');
    return Promise.resolve();
  }

  const payload = {
    conversions: [
      {
        gclid: gclid,
        conversion_action: `customers/${PLATFORMS.GOOGLE_ADS.conversion_id}/conversionActions/${PLATFORMS.GOOGLE_ADS.conversion_label}`,
        conversion_date_time: new Date().toISOString(),
        conversion_value: conversionValue || 0,
        currency_code: 'USD',
        user_identifiers: [
          {
            hashed_email: hashedEmail,
            hashed_phone_number: hashedPhone
          }
        ],
        user_agent: userAgent,
        custom_variables: [
          {
            conversion_custom_variable: 'sim_activation',
            value: 'true'
          }
        ]
      }
    ]
  };

  // Note: This requires Google Ads API setup - implement according to your auth method
  return axios.post(
    `https://googleads.googleapis.com/v14/customers/${PLATFORMS.GOOGLE_ADS.conversion_id}:uploadConversions`,
    payload,
    {
      headers: {
        'Authorization': `Bearer ${PLATFORMS.GOOGLE_ADS.developer_token}`,
        'Content-Type': 'application/json'
      }
    }
  );
}

// 3. Send to Facebook Ads (Conversions API)
async function sendToFacebookAds(fbclid, hashedEmail, hashedPhone, conversionValue, userIP, userAgent) {
  const eventTime = Math.floor(Date.now() / 1000);
  
  const payload = {
    data: [
      {
        event_name: 'CompleteRegistration', // Facebook standard event
        event_time: eventTime,
        event_source_url: 'https://smartlessmobile.com',
        action_source: 'system_generated', // Server-side event
        user_data: {
          em: [hashedEmail], // Hashed email
          ph: [hashedPhone], // Hashed phone
          client_ip_address: userIP,
          client_user_agent: userAgent,
          fbc: fbclid ? `fb.1.${eventTime}.${fbclid}` : undefined, // Facebook click ID
          fbp: undefined // Facebook browser ID (if available from frontend)
        },
        custom_data: {
          currency: 'USD',
          value: conversionValue || 0,
          content_category: 'sim_activation',
          content_name: 'SIM Card Activated'
        },
        event_id: `sim_activation_${Date.now()}` // Deduplication ID
      }
    ]
  };

  return axios.post(
    `https://graph.facebook.com/v18.0/${PLATFORMS.FACEBOOK.pixel_id}/events`,
    payload,
    {
      params: {
        access_token: PLATFORMS.FACEBOOK.access_token
      }
    }
  );
}

// Utility function for hashing
function hashSHA256(input) {
  if (!input) return null;
  return crypto.createHash('sha256').update(input.toLowerCase().trim()).digest('hex');
}

// Example usage when SIM is activated
const userData = {
  client_id: '123456789.987654321', // From user's CRM record
  user_id: 'user_123',
  email: 'user@example.com',
  phone: '+1234567890',
  sim_type: 'eSIM',
  plan_name: '10GB Plan',
  plan_value: 21.5,
  activation_date: '2025-06-12',
  gclid: 'gclid_from_crm', // Stored from account creation
  fbclid: 'fbclid_from_crm', // Stored from account creation
  user_ip: '192.168.1.1',
  user_agent: 'Mozilla/5.0...'
};

// Send to all platforms
sendSimActivationEvent(userData);
```

---

## Attribution Data Collection

### Capture Client ID and Attribution Data
Implement this when users create accounts to preserve marketing attribution:

```javascript
// Function to get GA4 Client ID
function getClientId(callback) {
  if (typeof ga === 'function') {
    ga(function(tracker) {
      callback(tracker.get('clientId'));
    });
  } else if (typeof gtag === 'function') {
    gtag('get', 'G-04WQBRC1X9', 'client_id', callback);
  }
}

// Helper function to get URL parameters
function getQueryParam(name) {
  const urlParams = new URLSearchParams(window.location.search);
  return urlParams.get(name) || '';
}

// Capture attribution data on account creation
getClientId(function(clientId) {
  const attributionData = {
    client_id: clientId,
    gclid: getQueryParam('gclid'),
    fbclid: getQueryParam('fbclid'),
    utm_source: getQueryParam('utm_source'),
    utm_medium: getQueryParam('utm_medium'),
    utm_campaign: getQueryParam('utm_campaign'),
    utm_term: getQueryParam('utm_term'),
    utm_content: getQueryParam('utm_content'),
    user_id: generatedUserId,
    timestamp: new Date().toISOString()
  };
  
  // Send to your backend API
  fetch('/api/save-attribution', {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json'
    },
    body: JSON.stringify(attributionData)
  });
});
```

---

## Data Privacy & Security

### Email and Phone Number Hashing
**IMPORTANT:** Always hash PII before sending to analytics:

```javascript
// SHA256 hashing function
async function hashString(str) {
  const encoder = new TextEncoder();
  const data = encoder.encode(str);
  const hash = await crypto.subtle.digest('SHA-256', data);
  return Array.from(new Uint8Array(hash))
    .map(b => b.toString(16).padStart(2, '0'))
    .join('');
}

// Usage example
const hashedEmail = await hashString('user@example.com');
const hashedPhone = await hashString('+1234567890');
```

---

## ✅ Implementation Checklist

### Frontend Tasks
- [ ] Implement enhanced pageview tracking on all pages
- [ ] Add login event tracking
- [ ] Add account creation event tracking
- [ ] Implement all ecommerce events in sequence
- [ ] Add attribution data capture on account creation
- [ ] Implement PII hashing functions
- [ ] Test all dataLayer pushes in browser console

### Backend Tasks
- [ ] Set up Measurement Protocol API credentials
- [ ] Implement SIM activation event sending
- [ ] Create attribution data storage in CRM
- [ ] Set up error handling and logging for server-side events
- [ ] Test server-side event delivery

### Analytics Team Tasks
- [ ] Configure GTM tags and triggers
- [ ] Set up GA4 conversion events
- [ ] Create custom reports and dashboards
- [ ] Verify data collection accuracy
- [ ] Set up attribution modeling

---

## Testing & Validation

### Browser Testing
1. Open browser developer tools
2. Navigate to Console tab
3. Check for `dataLayer` events: `console.log(dataLayer)`
4. Verify events are firing correctly at each step

### GA4 Real-Time Reports
1. Go to GA4 → Reports → Real-time
2. Test each event implementation
3. Verify events appear with correct parameters

### Measurement Protocol Testing
Use the [GA4 Measurement Protocol validation server](https://www.google-analytics.com/debug/mp/collect) for testing server-side events.

---

## 📞 Support

For implementation questions or issues:
- **Analytics Team**: Contact Glassroom
- **Development Team**: Contact LotusFlare
- **Documentation**: This GitHub repository
