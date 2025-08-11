# Telnyx Integration Guide - 10DLC SMS System

## Table of Contents
1. [Overview](#overview)
2. [Telnyx Account Setup](#telnyx-account-setup)
3. [API Integration](#api-integration)
4. [Brand Registration](#brand-registration)
5. [Campaign Management](#campaign-management)
6. [Message Operations](#message-operations)
7. [Webhook Integration](#webhook-integration)
8. [Phone Number Management](#phone-number-management)
9. [Error Handling](#error-handling)
10. [Testing & Validation](#testing--validation)
11. [Future Telnyx Features](#future-telnyx-features)

## Overview

This document provides comprehensive guidance for integrating with Telnyx's 10DLC SMS platform, covering all aspects from initial setup to advanced webhook handling and error management.

### Integration Benefits
- **10DLC Compliance**: Native support for U.S. carrier requirements
- **High Deliverability**: Carrier-approved messaging routes
- **Real-time Status**: Instant delivery reports and status updates
- **Global Reach**: International SMS capabilities
- **Scalable Infrastructure**: Handle high-volume messaging
- **Developer-Friendly**: Comprehensive APIs and SDKs

### Prerequisites
- Telnyx Account with API access
- Valid business information for brand registration
- SSL certificate for webhook endpoints
- Understanding of 10DLC regulations

## Telnyx Account Setup

### Account Registration Process
```typescript
// Account setup verification
interface TelnyxAccountSetup {
  accountId: string;
  apiKey: string;
  webhookUrl: string;
  billingProfile: string;
  messagingProfile: string;
}

class TelnyxAccountManager {
  async verifyAccountSetup(): Promise<AccountVerification> {
    try {
      // Verify API credentials
      const profile = await this.telnyx.messaging.getMessagingProfile();
      
      // Check billing setup
      const billing = await this.telnyx.billing.getProfile();
      
      // Verify webhook endpoint
      const webhookTest = await this.testWebhookEndpoint();
      
      return {
        isValid: true,
        apiAccess: true,
        billingSetup: billing.status === 'active',
        webhookConfigured: webhookTest.success,
        messagingProfile: profile.id
      };
    } catch (error) {
      return {
        isValid: false,
        error: error.message
      };
    }
  }

  async setupMessagingProfile(): Promise<MessagingProfile> {
    const profileConfig = {
      name: '10DLC SMS Profile',
      webhookUrl: process.env.WEBHOOK_URL,
      webhookFailoverUrl: process.env.WEBHOOK_FAILOVER_URL,
      enabled: true,
      settings: {
        deliveryReports: true,
        inboundMessages: true,
        optOutTracking: true
      }
    };

    return await this.telnyx.messaging.createProfile(profileConfig);
  }
}
```

### Environment Configuration
```bash
# .env configuration for Telnyx
TELNYX_API_KEY=your_telnyx_api_key_v2_here
TELNYX_API_URL=https://api.telnyx.com/v2
TELNYX_WEBHOOK_SECRET=your_webhook_secret_here
TELNYX_MESSAGING_PROFILE_ID=your_messaging_profile_id
TELNYX_BILLING_PROFILE_ID=your_billing_profile_id

# Webhook configuration
WEBHOOK_BASE_URL=https://yourdomain.com
WEBHOOK_PATH=/api/webhooks/telnyx
WEBHOOK_TIMEOUT=30000

# 10DLC specific
TCR_BRAND_API_URL=https://csp-api.campaignregistry.com/v2
TCR_CAMPAIGN_API_URL=https://csp-api.campaignregistry.com/v2
```

## API Integration

### Telnyx Service Implementation
```typescript
// src/services/telnyx.service.ts
import axios, { AxiosInstance, AxiosRequestConfig } from 'axios';
import crypto from 'crypto';

export class TelnyxService {
  private client: AxiosInstance;
  private apiKey: string;
  private baseURL: string;

  constructor() {
    this.apiKey = process.env.TELNYX_API_KEY!;
    this.baseURL = process.env.TELNYX_API_URL || 'https://api.telnyx.com/v2';
    
    this.client = axios.create({
      baseURL: this.baseURL,
      headers: {
        'Authorization': `Bearer ${this.apiKey}`,
        'Content-Type': 'application/json',
        'User-Agent': '10DLC-SMS-System/1.0.0'
      },
      timeout: 30000
    });

    this.setupInterceptors();
  }

  private setupInterceptors(): void {
    // Request interceptor for logging and metrics
    this.client.interceptors.request.use(
      (config) => {
        const requestId = crypto.randomUUID();
        config.metadata = { requestId, startTime: Date.now() };
        
        logger.info('Telnyx API request', {
          requestId,
          method: config.method?.toUpperCase(),
          url: config.url,
          endpoint: this.extractEndpoint(config.url)
        });

        return config;
      },
      (error) => {
        logger.error('Telnyx API request error', error);
        return Promise.reject(error);
      }
    );

    // Response interceptor for logging and metrics
    this.client.interceptors.response.use(
      (response) => {
        const { requestId, startTime } = response.config.metadata;
        const duration = Date.now() - startTime;
        
        logger.info('Telnyx API response', {
          requestId,
          status: response.status,
          duration,
          endpoint: this.extractEndpoint(response.config.url)
        });

        // Record metrics
        metricsService.recordExternalApiCall(
          'telnyx',
          this.extractEndpoint(response.config.url),
          response.status,
          duration
        );

        return response;
      },
      (error) => {
        const { requestId, startTime } = error.config?.metadata || {};
        const duration = startTime ? Date.now() - startTime : 0;
        
        logger.error('Telnyx API error', {
          requestId,
          status: error.response?.status,
          duration,
          endpoint: this.extractEndpoint(error.config?.url),
          error: error.response?.data
        });

        // Record error metrics
        metricsService.recordExternalApiCall(
          'telnyx',
          this.extractEndpoint(error.config?.url),
          error.response?.status || 0,
          duration
        );

        return Promise.reject(this.handleError(error));
      }
    );
  }

  private extractEndpoint(url?: string): string {
    if (!url) return 'unknown';
    return url.split('/').slice(-2).join('/');
  }

  private handleError(error: any): TelnyxApiError {
    if (error.response) {
      return new TelnyxApiError(
        error.response.data.message || 'Telnyx API error',
        error.response.status,
        error.response.data.code,
        error.response.data.details
      );
    }
    
    if (error.request) {
      return new TelnyxApiError('Network error - no response received', 0);
    }
    
    return new TelnyxApiError('Request setup error', 0);
  }

  // Message Operations
  async sendMessage(request: SendMessageRequest): Promise<TelnyxMessageResponse> {
    const payload = {
      from: request.from,
      to: request.to,
      text: request.text,
      media_urls: request.mediaUrls,
      webhook_url: request.webhookUrl || `${process.env.WEBHOOK_BASE_URL}${process.env.WEBHOOK_PATH}`,
      use_profile_webhooks: false,
      messaging_profile_id: request.messagingProfileId || process.env.TELNYX_MESSAGING_PROFILE_ID
    };

    const response = await this.client.post('/messages', payload);
    return response.data.data;
  }

  async getMessage(messageId: string): Promise<TelnyxMessageResponse> {
    const response = await this.client.get(`/messages/${messageId}`);
    return response.data.data;
  }

  // Brand Management
  async createBrand(brandData: CreateBrandRequest): Promise<TelnyxBrandResponse> {
    const payload = {
      entity_type: brandData.entityType,
      display_name: brandData.displayName,
      company_name: brandData.companyName,
      ein: brandData.ein,
      phone: brandData.phone,
      street: brandData.street,
      city: brandData.city,
      state: brandData.state,
      postal_code: brandData.postalCode,
      country: brandData.country,
      email: brandData.email,
      stock_symbol: brandData.stockSymbol,
      stock_exchange: brandData.stockExchange,
      vertical: brandData.vertical,
      alt_business_id: brandData.altBusinessId,
      alt_business_id_type: brandData.altBusinessIdType
    };

    const response = await this.client.post('/10dlc/brands', payload);
    return response.data.data;
  }

  async getBrand(brandId: string): Promise<TelnyxBrandResponse> {
    const response = await this.client.get(`/10dlc/brands/${brandId}`);
    return response.data.data;
  }

  async updateBrand(brandId: string, updates: UpdateBrandRequest): Promise<TelnyxBrandResponse> {
    const response = await this.client.patch(`/10dlc/brands/${brandId}`, updates);
    return response.data.data;
  }

  // Campaign Management
  async createCampaign(campaignData: CreateCampaignRequest): Promise<TelnyxCampaignResponse> {
    const payload = {
      brand_id: campaignData.brandId,
      campaign_id: campaignData.campaignId,
      description: campaignData.description,
      message_samples: campaignData.messageSamples,
      use_case: campaignData.useCase,
      vertical: campaignData.vertical,
      help_message: campaignData.helpMessage,
      help_keywords: campaignData.helpKeywords,
      opt_in_message: campaignData.optInMessage,
      opt_in_keywords: campaignData.optInKeywords,
      opt_out_message: campaignData.optOutMessage,
      opt_out_keywords: campaignData.optOutKeywords,
      auto_renewal: campaignData.autoRenewal
    };

    const response = await this.client.post('/10dlc/campaigns', payload);
    return response.data.data;
  }

  async getCampaign(campaignId: string): Promise<TelnyxCampaignResponse> {
    const response = await this.client.get(`/10dlc/campaigns/${campaignId}`);
    return response.data.data;
  }

  // Phone Number Management
  async searchPhoneNumbers(criteria: PhoneNumberSearchCriteria): Promise<AvailableNumber[]> {
    const params = {
      filter: {
        features: criteria.features || ['sms'],
        country_code: criteria.countryCode || 'US',
        number_type: criteria.numberType,
        locality: criteria.locality,
        administrative_area: criteria.administrativeArea
      },
      limit: criteria.limit || 20
    };

    const response = await this.client.get('/available_phone_numbers', { params });
    return response.data.data;
  }

  async purchasePhoneNumber(phoneNumber: string, options?: PurchaseOptions): Promise<PhoneNumberOrder> {
    const payload = {
      phone_numbers: [
        {
          phone_number: phoneNumber,
          messaging_profile_id: options?.messagingProfileId || process.env.TELNYX_MESSAGING_PROFILE_ID,
          connection_id: options?.connectionId
        }
      ]
    };

    const response = await this.client.post('/phone_number_orders', payload);
    return response.data.data;
  }

  async getPhoneNumber(phoneNumberId: string): Promise<PhoneNumberDetails> {
    const response = await this.client.get(`/phone_numbers/${phoneNumberId}`);
    return response.data.data;
  }

  async updatePhoneNumber(phoneNumberId: string, updates: PhoneNumberUpdate): Promise<PhoneNumberDetails> {
    const response = await this.client.patch(`/phone_numbers/${phoneNumberId}`, updates);
    return response.data.data;
  }

  // Webhook signature validation
  validateWebhookSignature(payload: string, signature: string, timestamp: string): boolean {
    const secret = process.env.TELNYX_WEBHOOK_SECRET!;
    const timestampThreshold = 300000; // 5 minutes
    
    // Check timestamp to prevent replay attacks
    const now = Date.now();
    const webhookTime = parseInt(timestamp) * 1000;
    
    if (Math.abs(now - webhookTime) > timestampThreshold) {
      return false;
    }

    // Verify signature
    const expectedSignature = crypto
      .createHmac('sha256', secret)
      .update(payload + timestamp)
      .digest('base64');

    return crypto.timingSafeEqual(
      Buffer.from(signature, 'base64'),
      Buffer.from(expectedSignature, 'base64')
    );
  }

  // Rate limiting and retry logic
  async withRetry<T>(
    operation: () => Promise<T>,
    maxRetries: number = 3,
    backoffMs: number = 1000
  ): Promise<T> {
    let lastError: Error;

    for (let attempt = 1; attempt <= maxRetries; attempt++) {
      try {
        return await operation();
      } catch (error) {
        lastError = error;
        
        // Don't retry on client errors (4xx)
        if (error.status >= 400 && error.status < 500) {
          throw error;
        }

        if (attempt < maxRetries) {
          const delay = backoffMs * Math.pow(2, attempt - 1);
          await new Promise(resolve => setTimeout(resolve, delay));
        }
      }
    }

    throw lastError!;
  }
}

// Custom error class for Telnyx API errors
export class TelnyxApiError extends Error {
  constructor(
    message: string,
    public status: number,
    public code?: string,
    public details?: any
  ) {
    super(message);
    this.name = 'TelnyxApiError';
  }
}
```

## Brand Registration

### Brand Registration Flow
```typescript
// src/services/brand-registration.service.ts
export class BrandRegistrationService {
  constructor(
    private telnyxService: TelnyxService,
    private brandRepository: Repository<Brand>
  ) {}

  async registerBrand(brandData: BrandRegistrationData): Promise<BrandRegistrationResult> {
    try {
      // Step 1: Validate brand data
      await this.validateBrandData(brandData);

      // Step 2: Create brand in local database
      const localBrand = await this.createLocalBrand(brandData);

      // Step 3: Register with Telnyx/TCR
      const telnyxBrand = await this.registerWithTelnyx(brandData);

      // Step 4: Update local brand with Telnyx IDs
      await this.updateLocalBrandWithTelnyxData(localBrand.id, telnyxBrand);

      // Step 5: Monitor registration status
      this.startStatusMonitoring(localBrand.id, telnyxBrand.id);

      return {
        success: true,
        brandId: localBrand.id,
        telnyxBrandId: telnyxBrand.id,
        status: 'pending',
        estimatedApprovalTime: '1-3 business days'
      };

    } catch (error) {
      logger.error('Brand registration failed', {
        error: error.message,
        brandData: this.sanitizeBrandData(brandData)
      });

      throw new BrandRegistrationError(
        'Failed to register brand',
        error.code || 'REGISTRATION_FAILED',
        error.details
      );
    }
  }

  private async validateBrandData(brandData: BrandRegistrationData): Promise<void> {
    const errors: string[] = [];

    // Required fields validation
    if (!brandData.legalBusinessName) errors.push('Legal business name is required');
    if (!brandData.businessType) errors.push('Business type is required');
    if (!brandData.businessAddress) errors.push('Business address is required');
    if (!brandData.contactPhone) errors.push('Contact phone is required');
    if (!brandData.contactEmail) errors.push('Contact email is required');

    // Business type specific validation
    if (brandData.businessType === 'Corporation' && !brandData.einTaxId) {
      errors.push('EIN Tax ID is required for corporations');
    }

    // Phone number format validation
    if (brandData.contactPhone && !this.isValidPhoneNumber(brandData.contactPhone)) {
      errors.push('Contact phone must be in E.164 format');
    }

    // Email format validation
    if (brandData.contactEmail && !this.isValidEmail(brandData.contactEmail)) {
      errors.push('Invalid email format');
    }

    if (errors.length > 0) {
      throw new ValidationError('Brand data validation failed', errors);
    }
  }

  private async registerWithTelnyx(brandData: BrandRegistrationData): Promise<TelnyxBrandResponse> {
    const telnyxRequest: CreateBrandRequest = {
      entityType: this.mapBusinessTypeToTelnyx(brandData.businessType),
      displayName: brandData.legalBusinessName,
      companyName: brandData.legalBusinessName,
      ein: brandData.einTaxId,
      phone: brandData.contactPhone,
      ...this.parseAddress(brandData.businessAddress),
      email: brandData.contactEmail,
      vertical: this.determineVertical(brandData),
      altBusinessId: brandData.dunsNumber,
      altBusinessIdType: brandData.dunsNumber ? 'DUNS' : undefined
    };

    return await this.telnyxService.withRetry(
      () => this.telnyxService.createBrand(telnyxRequest),
      3,
      2000
    );
  }

  private async startStatusMonitoring(localBrandId: string, telnyxBrandId: string): Promise<void> {
    // Monitor brand status every 30 minutes for 7 days
    const monitoringJob = {
      id: `brand-status-${localBrandId}`,
      schedule: '*/30 * * * *', // Every 30 minutes
      maxRetries: 336, // 7 days worth of 30-minute intervals
      handler: async () => {
        await this.checkBrandStatus(localBrandId, telnyxBrandId);
      }
    };

    await this.scheduleService.scheduleJob(monitoringJob);
  }

  async checkBrandStatus(localBrandId: string, telnyxBrandId: string): Promise<void> {
    try {
      const telnyxBrand = await this.telnyxService.getBrand(telnyxBrandId);
      const localBrand = await this.brandRepository.findOne({ where: { id: localBrandId } });

      if (!localBrand) return;

      // Update local brand status if it changed
      if (localBrand.status !== telnyxBrand.brand_status) {
        await this.updateBrandStatus(localBrandId, telnyxBrand);
        
        // Notify user of status change
        await this.notifyStatusChange(localBrand, telnyxBrand);
        
        // If approved or rejected, stop monitoring
        if (['approved', 'rejected'].includes(telnyxBrand.brand_status)) {
          await this.scheduleService.cancelJob(`brand-status-${localBrandId}`);
        }
      }

    } catch (error) {
      logger.error('Failed to check brand status', {
        localBrandId,
        telnyxBrandId,
        error: error.message
      });
    }
  }

  private async updateBrandStatus(localBrandId: string, telnyxBrand: TelnyxBrandResponse): Promise<void> {
    const updates: Partial<Brand> = {
      status: telnyxBrand.brand_status as BrandStatus,
      tcrBrandId: telnyxBrand.tcr_brand_id,
      trustScore: telnyxBrand.trust_score,
      updatedAt: new Date()
    };

    if (telnyxBrand.brand_status === 'approved') {
      updates.approvalDate = new Date();
    }

    await this.brandRepository.update(localBrandId, updates);
  }

  private async notifyStatusChange(localBrand: Brand, telnyxBrand: TelnyxBrandResponse): Promise<void> {
    const notification = {
      userId: localBrand.createdBy,
      type: 'brand_status_change',
      title: `Brand Registration ${telnyxBrand.brand_status.toUpperCase()}`,
      message: this.getStatusMessage(telnyxBrand.brand_status, localBrand.legalBusinessName),
      data: {
        brandId: localBrand.id,
        status: telnyxBrand.brand_status,
        trustScore: telnyxBrand.trust_score
      }
    };

    await this.notificationService.send(notification);
  }

  private getStatusMessage(status: string, brandName: string): string {
    switch (status) {
      case 'approved':
        return `Your brand "${brandName}" has been approved for 10DLC messaging. You can now create campaigns.`;
      case 'rejected':
        return `Your brand "${brandName}" registration has been rejected. Please review the requirements and resubmit.`;
      case 'suspended':
        return `Your brand "${brandName}" has been suspended. Please contact support for assistance.`;
      default:
        return `Your brand "${brandName}" status has been updated to ${status}.`;
    }
  }

  // Utility methods
  private mapBusinessTypeToTelnyx(businessType: string): string {
    const mapping: Record<string, string> = {
      'LLC': 'PRIVATE_FOR_PROFIT',
      'Corporation': 'PUBLIC_FOR_PROFIT',
      'Sole_Proprietor': 'SOLE_PROPRIETOR',
      'Partnership': 'PARTNERSHIP',
      'Other': 'OTHER'
    };
    return mapping[businessType] || 'OTHER';
  }

  private determineVertical(brandData: BrandRegistrationData): string {
    // Logic to determine business vertical based on brand data
    // This could be enhanced with AI/ML classification
    const keywords = brandData.legalBusinessName.toLowerCase();
    
    if (keywords.includes('health') || keywords.includes('medical')) return 'HEALTH';
    if (keywords.includes('bank') || keywords.includes('finance')) return 'FINANCIAL';
    if (keywords.includes('retail') || keywords.includes('store')) return 'RETAIL';
    if (keywords.includes('tech') || keywords.includes('software')) return 'TECHNOLOGY';
    
    return 'OTHER';
  }

  private parseAddress(address: string): AddressParts {
    // Enhanced address parsing logic
    // In production, consider using a proper address parsing service
    const parts = address.split(',').map(part => part.trim());
    
    return {
      street: parts[0] || '',
      city: parts[1] || '',
      state: parts[2]?.split(' ')[0] || '',
      postal_code: parts[2]?.split(' ')[1] || '',
      country: 'US'
    };
  }
}
```

## Campaign Management

### Campaign Registration Implementation
```typescript
// src/services/campaign-registration.service.ts
export class CampaignRegistrationService {
  constructor(
    private telnyxService: TelnyxService,
    private campaignRepository: Repository<Campaign>,
    private brandService: BrandService
  ) {}

  async registerCampaign(campaignData: CampaignRegistrationData): Promise<CampaignRegistrationResult> {
    try {
      // Step 1: Validate campaign data and brand status
      await this.validateCampaignData(campaignData);

      // Step 2: Create local campaign
      const localCampaign = await this.createLocalCampaign(campaignData);

      // Step 3: Register with Telnyx/TCR
      const telnyxCampaign = await this.registerWithTelnyx(campaignData);

      // Step 4: Update local campaign with Telnyx data
      await this.updateLocalCampaignWithTelnyxData(localCampaign.id, telnyxCampaign);

      // Step 5: Start status monitoring
      this.startCampaignStatusMonitoring(localCampaign.id, telnyxCampaign.id);

      return {
        success: true,
        campaignId: localCampaign.id,
        telnyxCampaignId: telnyxCampaign.id,
        status: 'pending',
        estimatedApprovalTime: this.getEstimatedApprovalTime(campaignData.useCase)
      };

    } catch (error) {
      logger.error('Campaign registration failed', {
        error: error.message,
        campaignData: this.sanitizeCampaignData(campaignData)
      });

      throw new CampaignRegistrationError(
        'Failed to register campaign',
        error.code || 'REGISTRATION_FAILED',
        error.details
      );
    }
  }

  private async validateCampaignData(campaignData: CampaignRegistrationData): Promise<void> {
    const errors: string[] = [];

    // Validate brand is approved
    const brand = await this.brandService.getBrandById(campaignData.brandId);
    if (!brand) {
      errors.push('Brand not found');
    } else if (brand.status !== 'approved') {
      errors.push('Brand must be approved before creating campaigns');
    }

    // Validate required fields
    if (!campaignData.name) errors.push('Campaign name is required');
    if (!campaignData.useCase) errors.push('Use case is required');
    if (!campaignData.description) errors.push('Description is required');
    if (!campaignData.sampleMessages || campaignData.sampleMessages.length === 0) {
      errors.push('At least one sample message is required');
    }
    if (!campaignData.optInProcess) errors.push('Opt-in process description is required');

    // Validate use case specific requirements
    await this.validateUseCaseRequirements(campaignData, errors);

    // Validate sample messages
    this.validateSampleMessages(campaignData.sampleMessages, errors);

    if (errors.length > 0) {
      throw new ValidationError('Campaign data validation failed', errors);
    }
  }

  private async validateUseCaseRequirements(
    campaignData: CampaignRegistrationData,
    errors: string[]
  ): Promise<void> {
    switch (campaignData.useCase) {
      case 'marketing':
        if (!campaignData.optOutInstructions) {
          errors.push('Opt-out instructions are required for marketing campaigns');
        }
        break;
      
      case '2fa':
        if (campaignData.sampleMessages.some(msg => msg.length > 160)) {
          errors.push('2FA messages should be under 160 characters');
        }
        break;
      
      case 'customer_care':
        // No additional requirements
        break;
      
      case 'account_notifications':
        // Validate that messages are transactional in nature
        if (this.containsMarketingContent(campaignData.sampleMessages)) {
          errors.push('Account notification messages should not contain marketing content');
        }
        break;
      
      default:
        errors.push('Invalid use case specified');
    }
  }

  private validateSampleMessages(messages: string[], errors: string[]): void {
    if (messages.length < 1 || messages.length > 5) {
      errors.push('Must provide 1-5 sample messages');
    }

    messages.forEach((message, index) => {
      if (message.length > 1600) {
        errors.push(`Sample message ${index + 1} exceeds 1600 character limit`);
      }

      // Check for SHAFT content
      const shaftViolations = this.checkForSHAFTContent(message);
      if (shaftViolations.length > 0) {
        errors.push(`Sample message ${index + 1} contains prohibited content: ${shaftViolations.join(', ')}`);
      }

      // Check for required opt-out language for marketing
      if (this.isMarketingMessage(message) && !this.hasOptOutLanguage(message)) {
        errors.push(`Marketing message ${index + 1} must include opt-out instructions`);
      }
    });
  }

  private async registerWithTelnyx(campaignData: CampaignRegistrationData): Promise<TelnyxCampaignResponse> {
    const brand = await this.brandService.getBrandById(campaignData.brandId);
    
    const telnyxRequest: CreateCampaignRequest = {
      brandId: brand.telnyxBrandId!,
      campaignId: `campaign_${Date.now()}`, // Telnyx generates this
      description: campaignData.description,
      messageSamples: campaignData.sampleMessages,
      useCase: this.mapUseCaseToTelnyx(campaignData.useCase),
      vertical: brand.vertical || 'OTHER',
      helpMessage: campaignData.helpMessage || 'Reply HELP for assistance',
      helpKeywords: ['HELP', 'INFO'],
      optInMessage: this.generateOptInMessage(campaignData.optInProcess),
      optInKeywords: campaignData.optInKeywords || ['START', 'YES', 'SUBSCRIBE'],
      optOutMessage: campaignData.optOutInstructions || 'You have been unsubscribed. Reply START to resubscribe.',
      optOutKeywords: ['STOP', 'UNSUBSCRIBE', 'CANCEL', 'END', 'QUIT'],
      autoRenewal: true
    };

    return await this.telnyxService.withRetry(
      () => this.telnyxService.createCampaign(telnyxRequest),
      3,
      2000
    );
  }

  private mapUseCaseToTelnyx(useCase: string): string {
    const mapping: Record<string, string> = {
      'customer_care': 'CUSTOMER_CARE',
      'marketing': 'MARKETING',
      '2fa': '2FA',
      'account_notifications': 'ACCOUNT_NOTIFICATION',
      'public_service': 'PUBLIC_SERVICE_ANNOUNCEMENT'
    };
    return mapping[useCase] || 'MIXED';
  }

  private getEstimatedApprovalTime(useCase: string): string {
    switch (useCase) {
      case '2fa':
      case 'account_notifications':
        return '1-2 business days';
      case 'customer_care':
        return '2-3 business days';
      case 'marketing':
        return '3-5 business days';
      default:
        return '1-5 business days';
    }
  }

  // SHAFT content detection
  private checkForSHAFTContent(message: string): string[] {
    const violations: string[] = [];
    const lowerMessage = message.toLowerCase();

    // Sex-related content
    const sexKeywords = ['adult', 'sex', 'escort', 'dating', 'hookup'];
    if (sexKeywords.some(keyword => lowerMessage.includes(keyword))) {
      violations.push('Adult content');
    }

    // Hate speech
    const hateKeywords = ['hate', 'racist', 'discrimination'];
    if (hateKeywords.some(keyword => lowerMessage.includes(keyword))) {
      violations.push('Hate content');
    }

    // Alcohol
    const alcoholKeywords = ['beer', 'wine', 'alcohol', 'liquor', 'vodka', 'whiskey'];
    if (alcoholKeywords.some(keyword => lowerMessage.includes(keyword))) {
      violations.push('Alcohol content');
    }

    // Firearms
    const firearmsKeywords = ['gun', 'firearm', 'weapon', 'ammunition', 'rifle'];
    if (firearmsKeywords.some(keyword => lowerMessage.includes(keyword))) {
      violations.push('Firearms content');
    }

    // Tobacco
    const tobaccoKeywords = ['cigarette', 'tobacco', 'smoking', 'vape', 'cigar'];
    if (tobaccoKeywords.some(keyword => lowerMessage.includes(keyword))) {
      violations.push('Tobacco content');
    }

    return violations;
  }

  private containsMarketingContent(messages: string[]): boolean {
    const marketingKeywords = [
      'sale', 'discount', 'offer', 'promotion', 'deal', 'buy now',
      'limited time', 'special offer', 'save money', 'free'
    ];

    return messages.some(message => {
      const lowerMessage = message.toLowerCase();
      return marketingKeywords.some(keyword => lowerMessage.includes(keyword));
    });
  }

  private isMarketingMessage(message: string): boolean {
    return this.containsMarketingContent([message]);
  }

  private hasOptOutLanguage(message: string): boolean {
    const optOutPatterns = [
      /reply\s+stop/i,
      /text\s+stop/i,
      /unsubscribe/i,
      /opt\s?out/i
    ];

    return optOutPatterns.some(pattern => pattern.test(message));
  }

  private generateOptInMessage(optInProcess: string): string {
    return `You've successfully opted in to receive messages from us. ${optInProcess} Reply STOP to unsubscribe.`;
  }
}
```

## Message Operations

### Enhanced Message Service
```typescript
// src/services/enhanced-message.service.ts
export class EnhancedMessageService {
  constructor(
    private telnyxService: TelnyxService,
    private messageRepository: Repository<Message>,
    private campaignService: CampaignService,
    private optOutService: OptOutService
  ) {}

  async sendMessage(request: SendMessageRequest): Promise<MessageSendResult> {
    try {
      // Step 1: Pre-send validation
      await this.validateMessageSend(request);

      // Step 2: Check compliance
      const complianceResult = await this.checkCompliance(request);
      if (!complianceResult.compliant) {
        throw new ComplianceError('Message violates compliance rules', complianceResult.violations);
      }

      // Step 3: Create message record
      const message = await this.createMessageRecord(request);

      // Step 4: Send via Telnyx
      const telnyxResponse = await this.sendViaTelnyx(request, message.id);

      // Step 5: Update message with Telnyx response
      await this.updateMessageWithTelnyxResponse(message.id, telnyxResponse);

      // Step 6: Record metrics
      this.recordMessageMetrics(request, telnyxResponse);

      return {
        success: true,
        messageId: message.id,
        telnyxMessageId: telnyxResponse.id,
        status: 'sent',
        cost: parseFloat(telnyxResponse.cost?.amount || '0'),
        segments: telnyxResponse.parts || 1
      };

    } catch (error) {
      logger.error('Message send failed', {
        error: error.message,
        request: this.sanitizeMessageRequest(request)
      });

      // Update message status to failed if record exists
      if (error.messageId) {
        await this.updateMessageStatus(error.messageId, 'failed', error.message);
      }

      throw error;
    }
  }

  private async validateMessageSend(request: SendMessageRequest): Promise<void> {
    const errors: string[] = [];

    // Phone number validation
    if (!this.isValidPhoneNumber(request.from)) {
      errors.push('Invalid from phone number format');
    }
    if (!this.isValidPhoneNumber(request.to)) {
      errors.push('Invalid to phone number format');
    }

    // Message content validation
    if (!request.text || request.text.trim().length === 0) {
      errors.push('Message text is required');
    }
    if (request.text && request.text.length > 1600) {
      errors.push('Message text exceeds 1600 character limit');
    }

    // Campaign validation
    if (request.campaignId) {
      const campaign = await this.campaignService.getCampaignById(request.campaignId);
      if (!campaign) {
        errors.push('Campaign not found');
      } else if (campaign.status !== 'approved') {
        errors.push('Campaign is not approved for messaging');
      }
    }

    // Opt-out check
    const isOptedOut = await this.optOutService.checkOptOut(request.to, request.campaignId);
    if (isOptedOut) {
      errors.push('Recipient has opted out of messages');
    }

    // Rate limiting check
    const rateLimitResult = await this.checkRateLimit(request);
    if (!rateLimitResult.allowed) {
      errors.push(`Rate limit exceeded. Try again in ${rateLimitResult.retryAfter} seconds`);
    }

    if (errors.length > 0) {
      throw new ValidationError('Message validation failed', errors);
    }
  }

  private async checkCompliance(request: SendMessageRequest): Promise<ComplianceResult> {
    const violations: string[] = [];

    // TCPA time restrictions (8 AM - 9 PM local time)
    const recipientTimezone = await this.getRecipientTimezone(request.to);
    const localTime = this.convertToTimezone(new Date(), recipientTimezone);
    const hour = localTime.getHours();

    if (hour < 8 || hour >= 21) {
      violations.push('Message sent outside allowed hours (8 AM - 9 PM local time)');
    }

    // SHAFT content check
    const shaftViolations = this.checkForSHAFTContent(request.text);
    violations.push(...shaftViolations);

    // Marketing message compliance
    if (this.isMarketingMessage(request.text) && !this.hasOptOutLanguage(request.text)) {
      violations.push('Marketing messages must include opt-out instructions');
    }

    return {
      compliant: violations.length === 0,
      violations
    };
  }

  private async sendViaTelnyx(request: SendMessageRequest, messageId: string): Promise<TelnyxMessageResponse> {
    const telnyxRequest: TelnyxSendMessageRequest = {
      from: request.from,
      to: request.to,
      text: request.text,
      mediaUrls: request.mediaUrls,
      webhookUrl: `${process.env.WEBHOOK_BASE_URL}/api/webhooks/telnyx`,
      messagingProfileId: request.messagingProfileId
    };

    // Add custom headers for tracking
    telnyxRequest.tags = {
      messageId: messageId,
      campaignId: request.campaignId || 'none',
      source: 'api'
    };

    return await this.telnyxService.withRetry(
      () => this.telnyxService.sendMessage(telnyxRequest),
      3,
      1000
    );
  }

  // Bulk message sending
  async sendBulkMessages(requests: BulkSendRequest): Promise<BulkSendResult> {
    const results: MessageSendResult[] = [];
    const errors: BulkSendError[] = [];
    
    // Process in batches to avoid overwhelming the API
    const batchSize = 100;
    const batches = this.chunkArray(requests.messages, batchSize);

    for (const batch of batches) {
      const batchPromises = batch.map(async (messageRequest, index) => {
        try {
          const result = await this.sendMessage({
            ...messageRequest,
            campaignId: requests.campaignId,
            from: requests.from
          });
          results.push(result);
        } catch (error) {
          errors.push({
            index: index,
            phoneNumber: messageRequest.to,
            error: error.message
          });
        }
      });

      await Promise.allSettled(batchPromises);
      
      // Rate limiting between batches
      if (batches.indexOf(batch) < batches.length - 1) {
        await new Promise(resolve => setTimeout(resolve, 1000));
      }
    }

    return {
      totalRequested: requests.messages.length,
      successCount: results.length,
      errorCount: errors.length,
      results,
      errors,
      estimatedCost: results.reduce((sum, result) => sum + result.cost, 0)
    };
  }

  // Message scheduling
  async scheduleMessage(request: ScheduleMessageRequest): Promise<ScheduleResult> {
    // Validate schedule time
    if (request.scheduledAt <= new Date()) {
      throw new ValidationError('Scheduled time must be in the future');
    }

    // Create scheduled message record
    const scheduledMessage = await this.createScheduledMessage(request);

    // Schedule the job
    const jobId = await this.scheduleService.scheduleJob({
      id: `message-${scheduledMessage.id}`,
      runAt: request.scheduledAt,
      handler: async () => {
        await this.sendMessage({
          campaignId: request.campaignId,
          from: request.from,
          to: request.to,
          text: request.text,
          mediaUrls: request.mediaUrls
        });
      }
    });

    await this.updateScheduledMessage(scheduledMessage.id, { jobId });

    return {
      success: true,
      scheduleId: scheduledMessage.id,
      scheduledAt: request.scheduledAt,
      jobId
    };
  }

  // Message templates
  async sendTemplateMessage(request: TemplateMessageRequest): Promise<MessageSendResult> {
    const template = await this.getMessageTemplate(request.templateId);
    if (!template) {
      throw new NotFoundError('Message template not found');
    }

    // Render template with variables
    const renderedText = this.renderTemplate(template.content, request.variables);

    return await this.sendMessage({
      campaignId: request.campaignId,
      from: request.from,
      to: request.to,
      text: renderedText,
      mediaUrls: request.mediaUrls
    });
  }

  private renderTemplate(template: string, variables: Record<string, any>): string {
    return template.replace(/\{\{(\w+)\}\}/g, (match, key) => {
      return variables[key] || match;
    });
  }

  // Helper methods
  private chunkArray<T>(array: T[], chunkSize: number): T[][] {
    const chunks: T[][] = [];
    for (let i = 0; i < array.length; i += chunkSize) {
      chunks.push(array.slice(i, i + chunkSize));
    }
    return chunks;
  }

  private async getRecipientTimezone(phoneNumber: string): Promise<string> {
    // Implement phone number to timezone mapping
    // This could use a service like Google's libphonenumber
    const areaCode = phoneNumber.substring(2, 5);
    return this.areaCodeToTimezone(areaCode);
  }

  private areaCodeToTimezone(areaCode: string): string {
    // Simplified mapping - in production, use a comprehensive database
    const mapping: Record<string, string> = {
      '212': 'America/New_York',
      '213': 'America/Los_Angeles',
      '312': 'America/Chicago',
      // ... more mappings
    };
    return mapping[areaCode] || 'America/New_York';
  }

  private convertToTimezone(date: Date, timezone: string): Date {
    return new Date(date.toLocaleString("en-US", { timeZone: timezone }));
  }

  private async checkRateLimit(request: SendMessageRequest): Promise<RateLimitResult> {
    const key = `rate_limit:${request.campaignId || 'global'}:${request.from}`;
    const current = await this.redisService.get(key) || 0;
    const limit = await this.getRateLimit(request.campaignId);

    if (current >= limit.perMinute) {
      return {
        allowed: false,
        retryAfter: 60,
        current: current,
        limit: limit.perMinute
      };
    }

    await this.redisService.incr(key, 60); // Increment with 60 second expiry

    return {
      allowed: true,
      current: current + 1,
      limit: limit.perMinute
    };
  }

  private recordMessageMetrics(request: SendMessageRequest, response: TelnyxMessageResponse): void {
    metricsService.recordMessageProcessed(
      'outbound',
      'sent',
      request.campaignId || 'none',
      'unknown'
    );

    metricsService.recordMessageCost(
      'telnyx',
      'SMS',
      request.campaignId || 'none',
      parseFloat(response.cost?.amount || '0')
    );
  }
}
```

## Webhook Integration

### Comprehensive Webhook Handler
```typescript
// src/controllers/webhook.controller.ts
export class WebhookController {
  constructor(
    private webhookService: WebhookService,
    private messageService: MessageService,
    private telnyxService: TelnyxService
  ) {}

  @Post('/telnyx')
  async handleTelnyxWebhook(
    @Body() payload: any,
    @Headers('telnyx-signature-ed25519') signature: string,
    @Headers('telnyx-timestamp') timestamp: string,
    @Res() response: Response
  ): Promise<void> {
    try {
      // Step 1: Validate webhook signature
      const isValid = this.telnyxService.validateWebhookSignature(
        JSON.stringify(payload),
        signature,
        timestamp
      );

      if (!isValid) {
        logger.warn('Invalid webhook signature', {
          signature,
          timestamp,
          payloadType: payload?.data?.event_type
        });
        response.status(401).json({ error: 'Invalid signature' });
        return;
      }

      // Step 2: Process webhook asynchronously
      await this.webhookService.processWebhook(payload);

      // Step 3: Respond immediately to Telnyx
      response.status(200).json({ status: 'received' });

    } catch (error) {
      logger.error('Webhook processing error', {
        error: error.message,
        payloadType: payload?.data?.event_type,
        payloadId: payload?.data?.id
      });

      // Return success to prevent retries for unprocessable events
      response.status(200).json({ status: 'error', message: error.message });
    }
  }
}

// src/services/webhook.service.ts
export class WebhookService {
  constructor(
    private messageService: MessageService,
    private campaignService: CampaignService,
    private webhookEventRepository: Repository<WebhookEvent>,
    private queueService: QueueService
  ) {}

  async processWebhook(payload: TelnyxWebhookPayload): Promise<void> {
    const eventType = payload.data.event_type;
    const eventId = payload.data.id;

    // Store raw webhook event for audit trail
    const webhookEvent = await this.storeWebhookEvent(payload);

    try {
      // Process webhook based on event type
      switch (eventType) {
        case 'message.sent':
          await this.handleMessageSent(payload.data);
          break;
        case 'message.delivered':
          await this.handleMessageDelivered(payload.data);
          break;
        case 'message.delivery_failed':
          await this.handleMessageFailed(payload.data);
          break;
        case 'message.received':
          await this.handleMessageReceived(payload.data);
          break;
        case 'message.finalized':
          await this.handleMessageFinalized(payload.data);
          break;
        default:
          logger.warn('Unhandled webhook event type', { eventType, eventId });
      }

      // Mark webhook as processed
      await this.markWebhookProcessed(webhookEvent.id);

    } catch (error) {
      logger.error('Webhook processing failed', {
        eventType,
        eventId,
        error: error.message
      });

      // Mark webhook as failed and schedule retry
      await this.markWebhookFailed(webhookEvent.id, error.message);
      await this.scheduleWebhookRetry(webhookEvent.id, payload);
    }
  }

  private async handleMessageSent(eventData: TelnyxMessageEvent): Promise<void> {
    const message = await this.messageService.findByTelnyxId(eventData.id);
    if (!message) {
      logger.warn('Message not found for sent event', { telnyxMessageId: eventData.id });
      return;
    }

    await this.messageService.updateMessage(message.id, {
      status: 'sent',
      sentAt: new Date(eventData.occurred_at),
      segments: eventData.parts,
      cost: parseFloat(eventData.cost?.amount || '0'),
      telnyxData: eventData
    });

    // Record metrics
    metricsService.recordMessageProcessed(
      'outbound',
      'sent',
      message.campaignId || 'none',
      'unknown'
    );

    logger.info('Message sent event processed', {
      messageId: message.id,
      telnyxMessageId: eventData.id,
      to: eventData.to
    });
  }

  private async handleMessageDelivered(eventData: TelnyxMessageEvent): Promise<void> {
    const message = await this.messageService.findByTelnyxId(eventData.id);
    if (!message) {
      logger.warn('Message not found for delivered event', { telnyxMessageId: eventData.id });
      return;
    }

    await this.messageService.updateMessage(message.id, {
      status: 'delivered',
      deliveredAt: new Date(eventData.occurred_at),
      telnyxData: eventData
    });

    // Record metrics
    metricsService.recordMessageProcessed(
      'outbound',
      'delivered',
      message.campaignId || 'none',
      'unknown'
    );

    // Update campaign performance metrics
    await this.updateCampaignMetrics(message.campaignId, 'delivered');

    logger.info('Message delivered event processed', {
      messageId: message.id,
      telnyxMessageId: eventData.id,
      to: eventData.to
    });
  }

  private async handleMessageFailed(eventData: TelnyxMessageEvent): Promise<void> {
    const message = await this.messageService.findByTelnyxId(eventData.id);
    if (!message) {
      logger.warn('Message not found for failed event', { telnyxMessageId: eventData.id });
      return;
    }

    await this.messageService.updateMessage(message.id, {
      status: 'failed',
      errorCode: eventData.errors?.[0]?.code,
      errorMessage: eventData.errors?.[0]?.title,
      telnyxData: eventData
    });

    // Record metrics
    metricsService.recordMessageProcessed(
      'outbound',
      'failed',
      message.campaignId || 'none',
      'unknown'
    );

    // Alert on high failure rates
    await this.checkFailureRates(message.campaignId);

    logger.error('Message failed event processed', {
      messageId: message.id,
      telnyxMessageId: eventData.id,
      to: eventData.to,
      errorCode: eventData.errors?.[0]?.code,
      errorMessage: eventData.errors?.[0]?.title
    });
  }

  private async handleMessageReceived(eventData: TelnyxMessageEvent): Promise<void> {
    // Handle inbound messages
    const inboundMessage = await this.messageService.createInboundMessage({
      telnyxMessageId: eventData.id,
      from: eventData.from,
      to: eventData.to,
      text: eventData.text,
      mediaUrls: eventData.media_urls,
      receivedAt: new Date(eventData.occurred_at),
      telnyxData: eventData
    });

    // Check for opt-out keywords
    if (this.isOptOutMessage(eventData.text)) {
      await this.processOptOut(eventData.from, eventData.to, inboundMessage.id);
    }

    // Check for help keywords
    if (this.isHelpMessage(eventData.text)) {
      await this.sendHelpResponse(eventData.from, eventData.to);
    }

    logger.info('Inbound message received', {
      messageId: inboundMessage.id,
      from: eventData.from,
      to: eventData.to,
      text: eventData.text.substring(0, 50) // Log first 50 chars
    });
  }

  private async handleMessageFinalized(eventData: TelnyxMessageEvent): Promise<void> {
    // Final state update with complete cost information
    const message = await this.messageService.findByTelnyxId(eventData.id);
    if (!message) return;

    await this.messageService.updateMessage(message.id, {
      cost: parseFloat(eventData.cost?.amount || '0'),
      finalizedAt: new Date(eventData.occurred_at),
      telnyxData: eventData
    });

    // Update billing information
    await this.updateBillingMetrics(message.campaignId, eventData.cost);
  }

  private isOptOutMessage(text: string): boolean {
    const optOutKeywords = ['STOP', 'UNSUBSCRIBE', 'CANCEL', 'END', 'QUIT'];
    const upperText = text.toUpperCase().trim();
    return optOutKeywords.includes(upperText);
  }

  private isHelpMessage(text: string): boolean {
    const helpKeywords = ['HELP', 'INFO', 'SUPPORT'];
    const upperText = text.toUpperCase().trim();
    return helpKeywords.includes(upperText);
  }

  private async processOptOut(from: string, to: string, messageId: string): Promise<void> {
    // Find the campaign based on the phone number
    const phoneNumber = await this.phoneNumberService.findByNumber(to);
    if (!phoneNumber) return;

    // Create opt-out record
    await this.optOutService.createOptOut({
      phoneNumber: from,
      campaignId: phoneNumber.campaignId,
      optOutMethod: 'sms_reply',
      messageId
    });

    // Send confirmation
    await this.sendOptOutConfirmation(from, to);

    logger.info('Opt-out processed', {
      phoneNumber: from,
      campaignId: phoneNumber.campaignId,
      messageId
    });
  }

  private async sendOptOutConfirmation(from: string, to: string): Promise<void> {
    const confirmationMessage = 'You have been unsubscribed. Reply START to resubscribe.';
    
    try {
      await this.messageService.sendMessage({
        from: to,
        to: from,
        text: confirmationMessage,
        // Don't associate with campaign to avoid opt-out loops
        campaignId: undefined
      });
    } catch (error) {
      logger.error('Failed to send opt-out confirmation', {
        from,
        to,
        error: error.message
      });
    }
  }

  private async sendHelpResponse(from: string, to: string): Promise<void> {
    const helpMessage = 'For assistance, contact our support team at support@yourcompany.com or call 1-800-SUPPORT. Reply STOP to unsubscribe.';
    
    try {
      await this.messageService.sendMessage({
        from: to,
        to: from,
        text: helpMessage,
        campaignId: undefined
      });
    } catch (error) {
      logger.error('Failed to send help response', {
        from,
        to,
        error: error.message
      });
    }
  }

  // Webhook retry mechanism
  private async scheduleWebhookRetry(webhookEventId: string, payload: TelnyxWebhookPayload): Promise<void> {
    const retryJob = {
      id: `webhook-retry-${webhookEventId}`,
      delay: this.calculateRetryDelay(1), // Start with attempt 1
      maxRetries: 5,
      handler: async (attempt: number) => {
        if (attempt <= 5) {
          await this.retryWebhookProcessing(webhookEventId, payload, attempt);
        }
      }
    };

    await this.queueService.addJob('webhook-retry', retryJob);
  }

  private async retryWebhookProcessing(
    webhookEventId: string,
    payload: TelnyxWebhookPayload,
    attempt: number
  ): Promise<void> {
    try {
      await this.processWebhook(payload);
      await this.markWebhookProcessed(webhookEventId);
      
      logger.info('Webhook retry successful', {
        webhookEventId,
        attempt,
        eventType: payload.data.event_type
      });
    } catch (error) {
      logger.error('Webhook retry failed', {
        webhookEventId,
        attempt,
        error: error.message
      });

      if (attempt < 5) {
        // Schedule next retry
        const nextRetryJob = {
          id: `webhook-retry-${webhookEventId}-${attempt + 1}`,
          delay: this.calculateRetryDelay(attempt + 1),
          handler: async () => {
            await this.retryWebhookProcessing(webhookEventId, payload, attempt + 1);
          }
        };

        await this.queueService.addJob('webhook-retry', nextRetryJob);
      } else {
        // Max retries exceeded
        await this.markWebhookMaxRetriesExceeded(webhookEventId);
      }
    }
  }

  private calculateRetryDelay(attempt: number): number {
    // Exponential backoff: 1s, 2s, 4s, 8s, 16s
    return Math.pow(2, attempt) * 1000;
  }

  private async storeWebhookEvent(payload: TelnyxWebhookPayload): Promise<WebhookEvent> {
    return await this.webhookEventRepository.save({
      id: crypto.randomUUID(),
      eventType: payload.data.event_type,
      resourceId: payload.data.id,
      payload: payload,
      processed: false,
      retryCount: 0,
      createdAt: new Date()
    });
  }

  private async markWebhookProcessed(webhookEventId: string): Promise<void> {
    await this.webhookEventRepository.update(webhookEventId, {
      processed: true,
      processedAt: new Date()
    });
  }

  private async markWebhookFailed(webhookEventId: string, error: string): Promise<void> {
    await this.webhookEventRepository.update(webhookEventId, {
      processingError: error,
      retryCount: () => 'retry_count + 1'
    });
  }

  private async markWebhookMaxRetriesExceeded(webhookEventId: string): Promise<void> {
    await this.webhookEventRepository.update(webhookEventId, {
      processingError: 'Max retries exceeded',
      processed: false
    });

    // Alert administrators
    await this.alertService.sendAlert({
      type: 'webhook_processing_failed',
      severity: 'high',
      message: `Webhook processing failed after maximum retries: ${webhookEventId}`
    });
  }
}
```

## Error Handling

### Comprehensive Error Management
```typescript
// src/utils/telnyx-error-handler.ts
export class TelnyxErrorHandler {
  
  static handleApiError(error: TelnyxApiError): ProcessedError {
    switch (error.status) {
      case 400:
        return this.handleBadRequest(error);
      case 401:
        return this.handleUnauthorized(error);
      case 403:
        return this.handleForbidden(error);
      case 404:
        return this.handleNotFound(error);
      case 422:
        return this.handleValidationError(error);
      case 429:
        return this.handleRateLimit(error);
      case 500:
      case 502:
      case 503:
      case 504:
        return this.handleServerError(error);
      default:
        return this.handleUnknownError(error);
    }
  }

  private static handleBadRequest(error: TelnyxApiError): ProcessedError {
    return {
      type: 'client_error',
      message: 'Invalid request parameters',
      userMessage: 'Please check your request and try again.',
      retryable: false,
      details: error.details
    };
  }

  private static handleUnauthorized(error: TelnyxApiError): ProcessedError {
    return {
      type: 'auth_error',
      message: 'Authentication failed',
      userMessage: 'API authentication failed. Please check your credentials.',
      retryable: false,
      action: 'check_api_key'
    };
  }

  private static handleRateLimit(error: TelnyxApiError): ProcessedError {
    const retryAfter = this.extractRetryAfter(error);
    
    return {
      type: 'rate_limit',
      message: 'Rate limit exceeded',
      userMessage: `Too many requests. Please try again in ${retryAfter} seconds.`,
      retryable: true,
      retryAfter: retryAfter,
      action: 'wait_and_retry'
    };
  }

  private static handleServerError(error: TelnyxApiError): ProcessedError {
    return {
      type: 'server_error',
      message: 'Telnyx server error',
      userMessage: 'Service temporarily unavailable. Please try again later.',
      retryable: true,
      retryAfter: 30,
      action: 'retry_with_backoff'
    };
  }

  private static extractRetryAfter(error: TelnyxApiError): number {
    // Extract from headers or use default
    return error.details?.retryAfter || 60;
  }

  // Circuit breaker pattern for Telnyx API
  static createCircuitBreaker(): CircuitBreaker {
    return new CircuitBreaker({
      failureThreshold: 5,
      recoveryTimeout: 30000,
      monitoringPeriod: 60000,
      onOpen: () => {
        logger.error('Telnyx API circuit breaker opened');
        metricsService.incrementCounter('telnyx_circuit_breaker_open');
      },
      onClose: () => {
        logger.info('Telnyx API circuit breaker closed');
        metricsService.incrementCounter('telnyx_circuit_breaker_close');
      }
    });
  }
}

// Circuit breaker implementation
class CircuitBreaker {
  private failures = 0;
  private lastFailureTime = 0;
  private state: 'closed' | 'open' | 'half-open' = 'closed';

  constructor(private options: CircuitBreakerOptions) {}

  async execute<T>(operation: () => Promise<T>): Promise<T> {
    if (this.state === 'open') {
      if (Date.now() - this.lastFailureTime < this.options.recoveryTimeout) {
        throw new CircuitBreakerOpenError('Circuit breaker is open');
      } else {
        this.state = 'half-open';
      }
    }

    try {
      const result = await operation();
      this.onSuccess();
      return result;
    } catch (error) {
      this.onFailure();
      throw error;
    }
  }

  private onSuccess(): void {
    this.failures = 0;
    this.state = 'closed';
  }

  private onFailure(): void {
    this.failures++;
    this.lastFailureTime = Date.now();

    if (this.failures >= this.options.failureThreshold) {
      this.state = 'open';
      this.options.onOpen?.();
    }
  }
}
```

## Testing & Validation

### Telnyx Integration Tests
```typescript
// tests/integration/telnyx.integration.test.ts
describe('Telnyx Integration Tests', () => {
  let telnyxService: TelnyxService;
  let testConfig: TestConfig;

  beforeAll(async () => {
    testConfig = await loadTestConfig();
    telnyxService = new TelnyxService();
  });

  describe('Brand Registration', () => {
    test('should register brand successfully', async () => {
      const brandData = createTestBrandData();
      
      const result = await telnyxService.createBrand(brandData);
      
      expect(result).toBeDefined();
      expect(result.id).toBeDefined();
      expect(result.brand_status).toBe('pending');
      
      // Cleanup
      testConfig.createdBrands.push(result.id);
    });

    test('should handle duplicate brand registration', async () => {
      const brandData = createTestBrandData();
      
      // Create brand first time
      await telnyxService.createBrand(brandData);
      
      // Try to create again - should fail
      await expect(telnyxService.createBrand(brandData))
        .rejects
        .toThrow('Brand already exists');
    });
  });

  describe('Message Sending', () => {
    test('should send message successfully', async () => {
      const messageRequest = {
        from: testConfig.testPhoneNumber,
        to: testConfig.recipientPhoneNumber,
        text: 'Integration test message'
      };
      
      const result = await telnyxService.sendMessage(messageRequest);
      
      expect(result).toBeDefined();
      expect(result.id).toBeDefined();
      expect(result.direction).toBe('outbound');
      expect(result.to).toBe(messageRequest.to);
    });

    test('should handle invalid phone numbers', async () => {
      const messageRequest = {
        from: '+1invalid',
        to: '+1invalid',
        text: 'Test message'
      };
      
      await expect(telnyxService.sendMessage(messageRequest))
        .rejects
        .toThrow('Invalid phone number');
    });
  });

  describe('Webhook Validation', () => {
    test('should validate webhook signature correctly', async () => {
      const payload = JSON.stringify({ test: 'data' });
      const timestamp = Math.floor(Date.now() / 1000).toString();
      const signature = generateTestSignature(payload, timestamp);
      
      const isValid = telnyxService.validateWebhookSignature(payload, signature, timestamp);
      
      expect(isValid).toBe(true);
    });

    test('should reject invalid webhook signature', async () => {
      const payload = JSON.stringify({ test: 'data' });
      const timestamp = Math.floor(Date.now() / 1000).toString();
      const invalidSignature = 'invalid-signature';
      
      const isValid = telnyxService.validateWebhookSignature(payload, invalidSignature, timestamp);
      
      expect(isValid).toBe(false);
    });
  });

  afterAll(async () => {
    // Cleanup test resources
    await cleanupTestResources(testConfig);
  });
});

// Mock Telnyx responses for unit testing
export const mockTelnyxResponses = {
  createBrand: {
    id: 'brand-123',
    brand_status: 'pending',
    tcr_brand_id: 'tcr-brand-123'
  },
  
  sendMessage: {
    id: 'msg-123',
    direction: 'outbound',
    from: '+12125551234',
    to: '+13125551234',
    text: 'Test message',
    cost: { amount: '0.0075', currency: 'USD' },
    parts: 1
  }
};
```

## Future Telnyx Features

### Planned Enhancements (2024 Roadmap)

#### Phase 1: Enhanced Analytics (Q2 2024)
```typescript
interface EnhancedAnalytics {
  deliveryAnalytics: {
    carrierDeliveryRates: Map<string, number>;
    timeBasedDeliveryPatterns: DeliveryPattern[];
    geographicDeliveryMetrics: GeographicMetrics;
  };
  
  costOptimization: {
    routeOptimization: RouteOptimizer;
    carrierlevelCosts: CarrierCostAnalysis;
    predictiveCostModeling: CostPredictor;
  };
  
  campaignInsights: {
    performanceBenchmarking: BenchmarkAnalyzer;
    audienceSegmentation: SegmentationEngine;
    messageOptimization: MessageOptimizer;
  };
}
```

#### Phase 2: Multi-Channel Integration (Q3 2024)
```typescript
interface MultiChannelIntegration {
  voiceIntegration: {
    callFallback: CallFallbackService;
    voiceVerification: VoiceVerificationService;
    hybridCampaigns: HybridCampaignManager;
  };
  
  emailIntegration: {
    crossChannelCampaigns: CrossChannelCampaignManager;
    unifiedOptOuts: UnifiedOptOutService;
    channelOrchestration: ChannelOrchestrator;
  };
  
  chatIntegration: {
    webchatFallback: WebchatFallbackService;
    unifiedInbox: UnifiedInboxService;
    botIntegration: BotIntegrationService;
  };
}
```

#### Phase 3: AI-Powered Features (Q4 2024)
```typescript
interface AIFeatures {
  smartSending: {
    optimalTimingPrediction: TimingPredictor;
    deliverabilityOptimization: DeliverabilityOptimizer;
    adaptiveRateLimiting: AdaptiveRateLimiter;
  };
  
  contentIntelligence: {
    autoContentOptimization: ContentOptimizer;
    sentimentAnalysis: SentimentAnalyzer;
    complianceChecking: AIComplianceChecker;
  };
  
  predictiveAnalytics: {
    churnPrediction: ChurnPredictor;
    engagementForecasting: EngagementForecaster;
    revenueOptimization: RevenueOptimizer;
  };
}
```

## Conclusion

This Telnyx integration guide provides:

- **Complete Integration**: End-to-end Telnyx API integration
- **Production-Ready**: Error handling, retry logic, and monitoring
- **Compliance-First**: Built-in 10DLC and TCPA compliance
- **Webhook Management**: Comprehensive webhook processing
- **Testing Strategy**: Integration and unit testing approaches
- **Future-Ready**: Roadmap for advanced features and capabilities

The integration ensures reliable, compliant, and scalable SMS messaging through Telnyx's 10DLC platform while providing comprehensive monitoring and error handling capabilities.
