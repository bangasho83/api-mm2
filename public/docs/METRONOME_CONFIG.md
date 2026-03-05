# 🔧 Metronome Configuration Management

The Metronome billing system now supports **user-configurable settings** with a simple, single-file approach.

## 📋 **Configuration Methods**

### **Priority Order (Highest to Lowest):**
1. **Environment Variables** - Override everything
2. **Direct File Editing** - Edit the constants file
3. **Hardcoded Defaults** - Fallback values

## 🎯 **Quick Start**

### **Method 1: Edit Constants File (Recommended)**
**File:** `constants/metronome-config.js`

Find and edit these lines:
```javascript
const METRONOME_IDS = {
  RATE_CARD_ID: process.env.METRONOME_RATE_CARD_ID || 'YOUR_RATE_CARD_ID_HERE',
  PRODUCT_ID: process.env.METRONOME_PRODUCT_ID || 'YOUR_PRODUCT_ID_HERE',
  ACCESS_CREDIT_TYPE_ID: process.env.METRONOME_ACCESS_CREDIT_TYPE_ID || 'YOUR_ACCESS_CREDIT_TYPE_ID_HERE',
  INVOICE_CREDIT_TYPE_ID: process.env.METRONOME_INVOICE_CREDIT_TYPE_ID || 'YOUR_INVOICE_CREDIT_TYPE_ID_HERE',
};

const FREE_TRIAL_CONFIG = {
  CREDITS: parseInt(process.env.METRONOME_FREE_TRIAL_CREDITS) || 500,
  DURATION_YEARS: parseFloat(process.env.METRONOME_FREE_TRIAL_DURATION_YEARS) || 1,
};
```

### **Method 2: Environment Variables (Highest Priority)**
Add to your `.env` file or deployment environment:
```bash
METRONOME_RATE_CARD_ID=cf9e5d3f-4941-4889-ad76-1c116981a086
METRONOME_PRODUCT_ID=4983cc7c-30b6-4355-a07a-9d5afd99d1a6
METRONOME_ACCESS_CREDIT_TYPE_ID=9cb3fb43-2f8b-4cb9-b1cf-15e5ba7bb08c
METRONOME_INVOICE_CREDIT_TYPE_ID=2714e483-4ff1-48e4-9e25-ac732e8f24f2
METRONOME_FREE_TRIAL_CREDITS=500
METRONOME_FREE_TRIAL_DURATION_YEARS=1
METRONOME_TOPUP_CREDITS_PER_CENT=2
METRONOME_API_TIMEOUT=30000
```

## 📊 **Available Settings**

### **🆔 Billing IDs**
| Environment Variable | Description | Default Value |
|---------------------|-------------|---------------|
| `METRONOME_RATE_CARD_ID` | Your Metronome rate card ID | `cf9e5d3f-4941-4889-ad76-1c116981a086` |
| `METRONOME_PRODUCT_ID` | Your Metronome product ID | `4983cc7c-30b6-4355-a07a-9d5afd99d1a6` |
| `METRONOME_ACCESS_CREDIT_TYPE_ID` | Credit type for API usage | `9cb3fb43-2f8b-4cb9-b1cf-15e5ba7bb08c` |
| `METRONOME_INVOICE_CREDIT_TYPE_ID` | Credit type for invoicing | `2714e483-4ff1-48e4-9e25-ac732e8f24f2` |

### **🎁 Free Trial Settings**
| Environment Variable | Description | Default |
|---------------------|-------------|---------|
| `METRONOME_FREE_TRIAL_CREDITS` | Credits granted in free trial | `500` |
| `METRONOME_FREE_TRIAL_DURATION_YEARS` | Trial duration in years | `1` |

### **💳 Top-up Settings**
| Environment Variable | Description | Default |
|---------------------|-------------|---------|
| `METRONOME_TOPUP_CREDITS_PER_CENT` | Credits per cent charged | `2` (e.g., $1 = 200 credits) |

### **⏱️ API Settings**
| Environment Variable | Description | Default |
|---------------------|-------------|---------|
| `METRONOME_API_TIMEOUT` | Default API timeout (ms) | `30000` |

## 📋 **Quick Reference: What to Edit Where**

| What You Want to Change | Edit This File | Line to Change |
|-------------------------|----------------|----------------|
| **Rate Card ID** | `constants/metronome-config.js` | `RATE_CARD_ID: process.env.METRONOME_RATE_CARD_ID \|\| 'YOUR_ID_HERE'` |
| **Product ID** | `constants/metronome-config.js` | `PRODUCT_ID: process.env.METRONOME_PRODUCT_ID \|\| 'YOUR_ID_HERE'` |
| **Free Trial Credits** | `constants/metronome-config.js` | `CREDITS: parseInt(process.env.METRONOME_FREE_TRIAL_CREDITS) \|\| 500` |
| **Credits per Dollar** | `constants/metronome-config.js` | `CREDITS_PER_CENT: parseFloat(process.env.METRONOME_TOPUP_CREDITS_PER_CENT) \|\| 2` |
| **Override Everything** | `.env` | Add `METRONOME_RATE_CARD_ID=new-uuid` |
| **Production Settings** | Vercel Dashboard | Environment Variables section |

## 🎯 **Most Common Edit: Fix Free Trial Contract**

**File to edit:** `constants/metronome-config.js`

**Find these lines and update the UUIDs:**
```javascript
const METRONOME_IDS = {
  RATE_CARD_ID: process.env.METRONOME_RATE_CARD_ID || 'PUT_YOUR_CORRECT_RATE_CARD_ID_HERE',
  PRODUCT_ID: process.env.METRONOME_PRODUCT_ID || 'PUT_YOUR_CORRECT_PRODUCT_ID_HERE',
  ACCESS_CREDIT_TYPE_ID: process.env.METRONOME_ACCESS_CREDIT_TYPE_ID || 'PUT_YOUR_CORRECT_ACCESS_CREDIT_TYPE_ID_HERE',
  INVOICE_CREDIT_TYPE_ID: process.env.METRONOME_INVOICE_CREDIT_TYPE_ID || 'PUT_YOUR_CORRECT_INVOICE_CREDIT_TYPE_ID_HERE',
};
```

## 🚨 **Important Notes**

- **UUID Format**: All Metronome IDs must be valid UUIDs (36 characters with hyphens)
- **Environment Variables**: Take highest priority and override file values
- **No Restart Required**: Changes take effect immediately
- **Backup**: Git commit your changes before editing

## 🔍 **Troubleshooting**

### **Free Trial Contract Still Failing**
1. Verify your Rate Card ID and Product ID are correct in Metronome dashboard
2. Check that the UUIDs are properly formatted (36 characters with hyphens)
3. Test with: `node scripts/test-free-trial-contract.js`

### **Changes Not Taking Effect**
1. Check if environment variables are overriding your file changes
2. Verify you edited the correct file: `constants/metronome-config.js`
3. Check server logs for any parsing errors

## 🚀 **Test Your Changes**

After editing the configuration:

```bash
# Test the constants are loading correctly
node scripts/test-metronome-constants.js

# Test free trial contract creation
node scripts/test-free-trial-contract.js
```
