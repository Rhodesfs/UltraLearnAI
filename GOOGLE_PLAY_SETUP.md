# Google Play Billing Setup Guide

This guide explains how to set up real monetization for UltraLearn AI before publishing to Google Play Store.

## Overview

The app uses Google Play Billing API for in-app subscriptions. The frontend code is already prepared - you need to complete the backend and native Android integration.

## Step 1: Google Play Console Setup

1. **Create a Developer Account**
   - Go to https://play.google.com/console
   - Pay the $25 one-time registration fee
   - Complete your developer profile

2. **Create Your App**
   - Click "Create app"
   - Fill in app details (name, language, type)
   - Complete the app content questionnaire

3. **Set Up In-App Products**
   - Navigate to: Monetize → Products → Subscriptions
   - Click "Create subscription"
   - Product ID: `premium_monthly`
   - Name: "UltraLearn AI Premium"
   - Description: "Unlock unlimited AI study tools, unlimited documents, and premium features"
   - Price: $1.99/month
   - Billing period: 1 month
   - Free trial: Optional (recommend 7 days)
   - Save and activate

## Step 2: Android Native Integration

Your app needs a native Android wrapper to bridge Google Play Billing to the WebView.

### Option A: React Native / Capacitor / Cordova

Install a billing plugin:
- **Capacitor**: `@capacitor-community/purchases`
- **React Native**: `react-native-iap`
- **Cordova**: `cordova-plugin-purchase`

### Option B: Custom Native Bridge

Create a native Android bridge in your WebView activity:

```kotlin
// MainActivity.kt
class MainActivity : AppCompatActivity() {
    private lateinit var billingClient: BillingClient

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)

        // Initialize billing client
        billingClient = BillingClient.newBuilder(this)
            .setListener(purchaseUpdateListener)
            .enablePendingPurchases()
            .build()

        // Inject JavaScript bridge
        webView.addJavascriptInterface(
            AndroidBillingBridge(billingClient),
            "AndroidBilling"
        )
    }
}

class AndroidBillingBridge(private val billingClient: BillingClient) {

    @JavascriptInterface
    fun queryProductDetails(productIds: String): String {
        // Query subscription details from Google Play
        val params = QueryProductDetailsParams.newBuilder()
            .setProductList(parseProductIds(productIds))
            .build()

        // Return product details as JSON
    }

    @JavascriptInterface
    fun launchBillingFlow(productId: String, type: String): String {
        // Launch Google Play purchase flow
        val params = BillingFlowParams.newBuilder()
            .setProductDetailsParamsList(...)
            .build()

        billingClient.launchBillingFlow(activity, params)

        // Return purchase result as JSON
    }
}
```

## Step 3: Backend Verification Endpoint

**CRITICAL**: Always verify purchases server-side to prevent fraud.

### Create `/api/verify-purchase` endpoint:

```javascript
// Example Node.js/Express endpoint
app.post('/api/verify-purchase', async (req, res) => {
  const { purchaseToken, productId } = req.body;
  const { userId } = req.user; // from auth middleware

  try {
    // Verify with Google Play API
    const { google } = require('googleapis');
    const androidPublisher = google.androidpublisher('v3');

    const result = await androidPublisher.purchases.subscriptions.get({
      packageName: 'com.yourcompany.ultralearnai',
      subscriptionId: productId,
      token: purchaseToken,
      auth: yourGoogleServiceAccountAuth
    });

    // Check if subscription is valid
    if (result.data.paymentState === 1) { // Payment received
      // Update user's subscription in database
      await db.users.update(userId, {
        isPremium: true,
        subscriptionId: productId,
        purchaseToken: purchaseToken,
        expiresAt: new Date(parseInt(result.data.expiryTimeMillis))
      });

      res.json({ success: true });
    } else {
      res.status(400).json({ error: 'Invalid purchase' });
    }
  } catch (error) {
    console.error('Verification failed:', error);
    res.status(500).json({ error: 'Verification failed' });
  }
});
```

### Set up Google Service Account:

1. Go to Google Cloud Console
2. Create a service account
3. Download JSON credentials
4. Grant API access in Google Play Console → API access

## Step 4: Database Setup

Store subscription information:

```sql
CREATE TABLE user_subscriptions (
  user_id VARCHAR(255) PRIMARY KEY,
  is_premium BOOLEAN DEFAULT false,
  product_id VARCHAR(100),
  purchase_token TEXT,
  subscribed_at TIMESTAMP,
  expires_at TIMESTAMP,
  auto_renewing BOOLEAN DEFAULT true,
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

## Step 5: Testing

1. **Add Test Accounts**
   - In Google Play Console → Setup → License testing
   - Add your test Google accounts
   - Test accounts can make purchases without being charged

2. **Test Purchase Flow**
   - Install your app on a test device
   - Sign in with test account
   - Try purchasing subscription
   - Verify in Google Play Console → Order management

## Step 6: Production Checklist

Before publishing:

- [ ] In-app subscription created and activated in Play Console
- [ ] Native Android billing bridge implemented
- [ ] Backend verification endpoint deployed
- [ ] Service account credentials configured
- [ ] Database tables created
- [ ] Test purchases completed successfully
- [ ] Privacy policy created (required for paid apps)
- [ ] App content rating completed
- [ ] Store listing completed with screenshots
- [ ] APK/AAB uploaded and reviewed

## Important Notes

### Security
- **NEVER** trust client-side purchase verification
- **ALWAYS** verify purchase tokens on your backend
- Store purchase tokens securely
- Validate subscription expiry dates

### Google Play Policies
- Must provide clear subscription terms
- Must allow users to cancel subscriptions
- Must handle subscription lifecycle (renewal, cancellation, refunds)
- Follow Google Play's content policies

### Subscription Management
- Users manage subscriptions through Google Play (not your app)
- Implement webhook to receive subscription events
- Handle grace periods and account holds
- Support subscription upgrades/downgrades

## Resources

- [Google Play Billing Library](https://developer.android.com/google/play/billing)
- [Google Play Developer API](https://developers.google.com/android-publisher)
- [Subscription Testing Guide](https://developer.android.com/google/play/billing/test)
- [Google Play Console](https://play.google.com/console)

## Support

For issues with billing integration:
1. Check Google Play Billing documentation
2. Review Android Billing sample apps
3. Contact Google Play Developer Support
4. Check Stack Overflow with tag `google-play-billing`

---

**Current Status**: Frontend code is ready. Backend verification and native Android bridge need to be implemented before publishing.
