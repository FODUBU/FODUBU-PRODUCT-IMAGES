
1.
Pi ecosystem Token documentation

https://raw.githubusercontent.com/PiNetwork/PiRC/refs/heads/main/PiRC1/ReadMe.md

3)
import axios from 'axios';
import CryptoJS from 'crypto-js';
import { TransactionBuilder, Keypair, Networks, Operation } from '@stellar/stellar-sdk';

class PiNetworkIntegration {
  constructor() {
    // Your Pi App credentials
    this.config = {
      appId: process.env.PI_APP_ID || '9fe18bd0602752ab','68fae1db65ce18fb14fb8d3b','68f5dd437e8c1d6c71d554cf',
      apiKey: process.env.PI_API_KEY || 'cev68cjqhbtwnje5sdfrawbdiyhnuvpj5bahsczqjkvy4dfb2oqrftbgz6pl2qks',
      secretKey: process.env.PI_SECRET_KEY || 'SD3FEO7YUS4YJCWHB63CAVLQUPDDAM36QKV2NOVLBEGXVM6UNT2RYCVI',
      
      // Pi Network endpoints
      apiBaseUrl: 'https://api.minepi.com/v2',
      horizonUrl: 'https://api.mainnet.minepi.com/v2',
      
      // Stellar network (Pi uses Stellar)
      networkPassphrase: 'Pi Network',
      network: Networks.PUBLIC
    };
    
    // Initialize HTTP client
    this.http = axios.create({
      baseURL: this.config.apiBaseUrl,
      headers: {
        'Authorization': `Key ${this.config.apiKey}`,
        'Content-Type': 'application/json',
        'User-Agent': 'FODUBU-API/1.0.0'
      },
      timeout: 30000
    });
  }
  
  // ===== AUTHENTICATION =====
  async authenticateUser(accessToken, appName) {
    try {
      console.log(`🔐 Authenticating user for ${appName}...`);
      
      // Verify Pi access token
      const userData = await this.verifyAccessToken(accessToken);
      
      if (!userData || !userData.uid) {
        throw new Error('Invalid Pi token: No user ID found');
      }
      
      // Create unified user object
      const user = {
        piUid: userData.uid,
        username: userData.username || `pi_user_${userData.uid.substring(0, 8)}`,
        walletAddress: userData.walletAddress,
        app: appName,
        authenticatedAt: new Date().toISOString(),
        scopes: userData.scopes || ['payments', 'username']
      };
      
      console.log(`✅ User authenticated: ${user.username} (${user.piUid})`);
      return user;
      
    } catch (error) {
      console.error(`❌ Pi authentication failed for ${appName}:`, error.message);
      throw new Error(`Pi authentication failed: ${error.message}`);
    }
  }
  
  async verifyAccessToken(accessToken) {
    try {
      const response = await this.http.post('/auth/verify', {
        access_token: accessToken
      });
      
      return response.data;
    } catch (error) {
      console.error('Token verification failed:', error.response?.data || error.message);
      throw error;
    }
  }
  
  // ===== PAYMENTS =====
  async createPayment(userUid, amount, memo, metadata = {}) {
    try {
      console.log(`💰 Creating payment: ${amount} Pi for user ${userUid}`);
      
      const paymentData = {
        amount: amount.toString(),
        memo: memo || 'FODUBU Payment',
        metadata: {
          ...metadata,
          timestamp: new Date().toISOString(),
          app: metadata.app || 'FODUBU'
        },
        uid: userUid
      };
      
      const response = await this.http.post('/payments', {
        payment: paymentData
      });
      
      const payment = response.data;
      
      console.log(`✅ Payment created: ${payment.identifier}`);
      return {
        success: true,
        paymentId: payment.identifier,
        paymentData: payment,
        status: 'incomplete'
      };
      
    } catch (error) {
      console.error('Payment creation failed:', error.response?.data || error.message);
      throw error;
    }
  }
  
  async approvePayment(paymentId) {
    try {
      console.log(`✅ Approving payment: ${paymentId}`);
      
      const response = await this.http.post(`/payments/${paymentId}/approve`);
      
      return {
        success: true,
        paymentId,
        status: 'approved',
        data: response.data
      };
    } catch (error) {
      console.error('Payment approval failed:', error.response?.data || error.message);
      throw error;
    }
  }
  
  async completePayment(paymentId, txid) {
    try {
      console.log(`🏁 Completing payment: ${paymentId}`);
      
      const response = await this.http.post(`/payments/${paymentId}/complete`, {
        txid: txid
      });
      
      return {
        success: true,
        paymentId,
        txid,
        status: 'completed',
        completedAt: new Date().toISOString(),
        data: response.data
      };
    } catch (error) {
      console.error('Payment completion failed:', error.response?.data || error.message);
      throw error;
    }
  }
  
  async getPayment(paymentId) {
    try {
      const response = await this.http.get(`/payments/${paymentId}`);
      return response.data;
    } catch (error) {
      console.error('Get payment failed:', error.message);
      throw error;
    }
  }
  
  // ===== STELLAR TRANSACTIONS (For advanced operations) =====
  async createStellarTransaction(sourceAccount, destination, amount, memo = '') {
    try {
      // Load source account from Horizon
      const server = new StellarSdk.Server(this.config.horizonUrl);
      const source = await server.loadAccount(sourceAccount);
      
      // Create transaction
      const transaction = new TransactionBuilder(source, {
        fee: StellarSdk.BASE_FEE,
        networkPassphrase: this.config.networkPassphrase
      })
      .addOperation(Operation.payment({
        destination: destination,
        asset: StellarSdk.Asset.native(),
        amount: amount.toString()
      }))
      .addMemo(StellarSdk.Memo.text(memo))
      .setTimeout(30)
      .build();
      
      return transaction;
    } catch (error) {
      console.error('Stellar transaction creation failed:', error);
      throw error;
    }
  }
  
  // ===== USER MANAGEMENT =====
  async getUserInfo(userUid) {
    try {
      const response = await this.http.get(`/users/${userUid}`);
      return response.data;
    } catch (error) {
      console.error('Get user info failed:', error.message);
      return null;
    }
  }
  
  async getUserBalance(userUid) {
    try {
      const response = await this.http.get(`/users/${userUid}/balance`);
      return {
        balance: response.data.balance,
        currency: 'PI',
        updatedAt: new Date().toISOString()
      };
    } catch (error) {
      console.warn('Could not fetch user balance:', error.message);
      return {
        balance: 0,
        currency: 'PI',
        note: 'Balance unavailable'
      };
    }
  }
  
  // ===== UTILITIES =====
  generateAuthUrl(appName, redirectUri, scopes = ['username', 'payments']) {
    const params = new URLSearchParams({
      client_id: this.config.appId,
      redirect_uri: redirectUri,
      response_type: 'code',
      scope: scopes.join(' '),
      state: CryptoJS.SHA256(`${appName}_${Date.now()}`).toString()
    });
    
    return `https://minepi.com/oauth/authorize?${params.toString()}`;
  }
  
  async exchangeCodeForToken(code, redirectUri) {
    try {
      const response = await this.http.post('/oauth/token', {
        grant_type: 'authorization_code',
        code: code,
        redirect_uri: redirectUri,
        client_id: this.config.appId,
        client_secret: this.config.secretKey
      });
      
      return response.data;
    } catch (error) {
      console.error('Token exchange failed:', error.response?.data || error.message);
      throw error;
    }
  }
  
  // Health check
  async healthCheck() {
    try {
      await this.http.get('/health');
      return {
        status: 'healthy',
        piApi: 'connected',
        timestamp: new Date().toISOString()
      };
    } catch (error) {
      return {
        status: 'unhealthy',
        piApi: 'disconnected',
        error: error.message,
        timestamp: new Date().toISOString()
      };
    }
  }
}

// Export singleton instance
export default new PiNetworkIntegration();
// Run test
testPiIntegration();

4.

Subscription API documents;
https://raw.githubusercontent.com/PiNetwork/PiRC/refs/heads/main/PiRC2/ReadMe.md
