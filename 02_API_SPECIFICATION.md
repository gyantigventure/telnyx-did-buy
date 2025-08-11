# API Specification Document - 10DLC SMS System

## Table of Contents
1. [Overview](#overview)
2. [API Architecture](#api-architecture)
3. [Authentication & Authorization](#authentication--authorization)
4. [API Versioning](#api-versioning)
5. [Core Endpoints](#core-endpoints)
6. [Data Models](#data-models)
7. [Error Handling](#error-handling)
8. [Rate Limiting](#rate-limiting)
9. [Webhooks](#webhooks)
10. [Future API Features](#future-api-features)
11. [SDK Development](#sdk-development)

## Overview

This document provides a comprehensive specification for the 10DLC SMS System REST API. The API follows RESTful principles and provides endpoints for managing brands, campaigns, phone numbers, and message operations.

### API Design Principles
- **RESTful Architecture**: Standard HTTP methods and status codes
- **Resource-Based URLs**: Clear, hierarchical resource structure
- **JSON Communication**: Request and response bodies in JSON format
- **Stateless Operations**: Each request contains all necessary information
- **Idempotent Operations**: Safe retry behavior for critical operations
- **Versioning Support**: Backwards compatible API evolution

### Base URL Structure
```
Production:    https://api.yourdomain.com/v1
Staging:       https://staging-api.yourdomain.com/v1
Development:   http://localhost:3000/api/v1
```

## API Architecture

### RESTful Resource Hierarchy
```
/api/v1/
├── auth/
│   ├── register
│   ├── login
│   ├── refresh
│   └── logout
├── users/
│   ├── {userId}
│   ├── {userId}/profile
│   └── {userId}/api-keys
├── brands/
│   ├── {brandId}
│   ├── {brandId}/campaigns
│   └── {brandId}/sync
├── campaigns/
│   ├── {campaignId}
│   ├── {campaignId}/messages
│   ├── {campaignId}/numbers
│   └── {campaignId}/analytics
├── messages/
│   ├── send
│   ├── {messageId}
│   ├── bulk
│   └── scheduled
├── numbers/
│   ├── search
│   ├── purchase
│   ├── {numberId}
│   └── {numberId}/assign
├── opt-outs/
│   ├── check
│   ├── add
│   └── {phoneNumber}
└── webhooks/
    ├── telnyx
    ├── stats
    └── retry
```

### HTTP Methods and Usage
| Method | Usage | Idempotent | Safe |
|--------|-------|------------|------|
| GET | Retrieve resources | Yes | Yes |
| POST | Create resources | No | No |
| PUT | Update/Replace resources | Yes | No |
| PATCH | Partial updates | No | No |
| DELETE | Remove resources | Yes | No |

## Authentication & Authorization

### JWT Authentication
```http
POST /api/v1/auth/login
Content-Type: application/json

{
  "email": "user@example.com",
  "password": "securepassword"
}
```

**Response:**
```json
{
  "success": true,
  "data": {
    "user": {
      "id": "user-uuid",
      "email": "user@example.com",
      "firstName": "John",
      "lastName": "Doe",
      "role": "user"
    },
    "tokens": {
      "accessToken": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
      "refreshToken": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
      "expiresIn": 3600
    }
  }
}
```

### API Key Authentication
```http
GET /api/v1/brands
Authorization: Bearer api_key_1234567890abcdef
X-API-Version: 1.0
```

### Authorization Scopes
```typescript
interface Permission {
  resource: string;
  actions: string[];
  conditions?: Record<string, any>;
}

// Example permissions
const permissions: Permission[] = [
  {
    resource: "brands",
    actions: ["read", "write", "delete"],
    conditions: { "brand.userId": "current_user" }
  },
  {
    resource: "campaigns",
    actions: ["read", "write"],
    conditions: { "campaign.brand.userId": "current_user" }
  },
  {
    resource: "messages",
    actions: ["read", "write"],
    conditions: { "message.campaign.brand.userId": "current_user" }
  }
];
```

## API Versioning

### Versioning Strategy
```http
# Header-based versioning (Preferred)
GET /api/v1/brands
Accept: application/vnd.api+json;version=1.0

# URL-based versioning
GET /api/v1/brands

# Query parameter versioning
GET /api/brands?version=1.0
```

### Version Lifecycle
| Version | Status | Support End | Features |
|---------|--------|-------------|----------|
| v1.0 | Current | 2025-12-31 | Core messaging |
| v1.1 | Current | 2026-06-30 | Templates, scheduling |
| v2.0 | Beta | TBD | Multi-provider, AI |

### Backwards Compatibility
```typescript
// Version-specific response transformers
interface ApiVersionConfig {
  version: string;
  transformResponse: (data: any) => any;
  deprecatedFields: string[];
  addedFields: string[];
}

const v1Config: ApiVersionConfig = {
  version: '1.0',
  transformResponse: (data) => {
    // Remove v1.1+ fields for v1.0 clients
    delete data.scheduledAt;
    delete data.templateId;
    return data;
  },
  deprecatedFields: [],
  addedFields: []
};
```

## Core Endpoints

### Brand Management

#### Create Brand
```http
POST /api/v1/brands
Authorization: Bearer {token}
Content-Type: application/json

{
  "legalBusinessName": "Acme Corporation",
  "businessType": "Corporation",
  "einTaxId": "12-3456789",
  "dunsNumber": "123456789",
  "businessAddress": "123 Main St, Anytown, NY 12345",
  "website": "https://acme.com",
  "contactPhone": "+12125551234",
  "contactEmail": "contact@acme.com"
}
```

**Response (201 Created):**
```json
{
  "success": true,
  "data": {
    "id": "brand-uuid",
    "legalBusinessName": "Acme Corporation",
    "businessType": "Corporation",
    "status": "pending",
    "trustScore": 0,
    "telnyxBrandId": null,
    "createdAt": "2024-01-15T10:30:00.000Z",
    "updatedAt": "2024-01-15T10:30:00.000Z"
  }
}
```

#### List Brands
```http
GET /api/v1/brands?page=1&limit=20&status=approved
Authorization: Bearer {token}
```

**Response (200 OK):**
```json
{
  "success": true,
  "data": {
    "brands": [
      {
        "id": "brand-uuid",
        "legalBusinessName": "Acme Corporation",
        "businessType": "Corporation",
        "status": "approved",
        "trustScore": 85,
        "campaignCount": 3,
        "createdAt": "2024-01-15T10:30:00.000Z"
      }
    ],
    "pagination": {
      "page": 1,
      "limit": 20,
      "total": 1,
      "totalPages": 1
    }
  }
}
```

#### Get Brand Details
```http
GET /api/v1/brands/{brandId}
Authorization: Bearer {token}
```

**Response (200 OK):**
```json
{
  "success": true,
  "data": {
    "id": "brand-uuid",
    "legalBusinessName": "Acme Corporation",
    "businessType": "Corporation",
    "einTaxId": "12-3456789",
    "businessAddress": "123 Main St, Anytown, NY 12345",
    "website": "https://acme.com",
    "contactPhone": "+12125551234",
    "contactEmail": "contact@acme.com",
    "status": "approved",
    "statusReason": null,
    "trustScore": 85,
    "telnyxBrandId": "telnyx-brand-123",
    "tcrBrandId": "tcr-brand-456",
    "registrationDate": "2024-01-15T10:30:00.000Z",
    "approvalDate": "2024-01-17T14:22:00.000Z",
    "campaigns": [
      {
        "id": "campaign-uuid",
        "name": "Customer Notifications",
        "useCase": "account_notifications",
        "status": "approved"
      }
    ],
    "createdAt": "2024-01-15T10:30:00.000Z",
    "updatedAt": "2024-01-17T14:22:00.000Z"
  }
}
```

### Campaign Management

#### Create Campaign
```http
POST /api/v1/campaigns
Authorization: Bearer {token}
Content-Type: application/json

{
  "brandId": "brand-uuid",
  "name": "Customer Notifications",
  "useCase": "account_notifications",
  "description": "Account alerts and notifications for customers",
  "sampleMessages": [
    "Your account balance is $50.00",
    "Payment of $25.00 received",
    "Login detected from new device"
  ],
  "optInProcess": "Users opt in during account creation by checking the SMS notifications checkbox",
  "optOutInstructions": "Reply STOP to unsubscribe"
}
```

**Response (201 Created):**
```json
{
  "success": true,
  "data": {
    "id": "campaign-uuid",
    "brandId": "brand-uuid",
    "name": "Customer Notifications",
    "useCase": "account_notifications",
    "description": "Account alerts and notifications for customers",
    "status": "pending",
    "throughputLimit": 1,
    "telnyxCampaignId": null,
    "createdAt": "2024-01-15T11:00:00.000Z",
    "updatedAt": "2024-01-15T11:00:00.000Z"
  }
}
```

#### List Campaigns
```http
GET /api/v1/campaigns?brandId=brand-uuid&status=approved
Authorization: Bearer {token}
```

**Response (200 OK):**
```json
{
  "success": true,
  "data": {
    "campaigns": [
      {
        "id": "campaign-uuid",
        "name": "Customer Notifications",
        "useCase": "account_notifications",
        "status": "approved",
        "brand": {
          "id": "brand-uuid",
          "legalBusinessName": "Acme Corporation"
        },
        "messageCount": 1250,
        "lastMessageAt": "2024-01-20T09:15:00.000Z",
        "createdAt": "2024-01-15T11:00:00.000Z"
      }
    ],
    "pagination": {
      "page": 1,
      "limit": 20,
      "total": 1,
      "totalPages": 1
    }
  }
}
```

### Message Operations

#### Send Single Message
```http
POST /api/v1/messages/send
Authorization: Bearer {token}
Content-Type: application/json

{
  "campaignId": "campaign-uuid",
  "from": "+12125551234",
  "to": "+13125551234",
  "text": "Hello! Your order #12345 has been shipped and will arrive tomorrow.",
  "mediaUrls": [
    "https://example.com/tracking-image.png"
  ]
}
```

**Response (200 OK):**
```json
{
  "success": true,
  "data": {
    "id": "message-uuid",
    "telnyxMessageId": "telnyx-msg-123",
    "status": "sent",
    "cost": 0.0075,
    "segments": 1,
    "encoding": "GSM-7",
    "sentAt": "2024-01-20T10:30:00.000Z",
    "estimatedDelivery": "2024-01-20T10:30:05.000Z"
  }
}
```

#### Send Bulk Messages
```http
POST /api/v1/messages/bulk
Authorization: Bearer {token}
Content-Type: application/json

{
  "campaignId": "campaign-uuid",
  "from": "+12125551234",
  "messages": [
    {
      "to": "+13125551234",
      "text": "Hello John! Your order #12345 has shipped.",
      "personalizations": {
        "firstName": "John",
        "orderNumber": "12345"
      }
    },
    {
      "to": "+14155551234",
      "text": "Hello Jane! Your order #12346 has shipped.",
      "personalizations": {
        "firstName": "Jane",
        "orderNumber": "12346"
      }
    }
  ]
}
```

**Response (202 Accepted):**
```json
{
  "success": true,
  "data": {
    "batchId": "batch-uuid",
    "totalMessages": 2,
    "estimatedCost": 0.015,
    "status": "processing",
    "createdAt": "2024-01-20T10:35:00.000Z"
  }
}
```

#### Get Message Status
```http
GET /api/v1/messages/{messageId}
Authorization: Bearer {token}
```

**Response (200 OK):**
```json
{
  "success": true,
  "data": {
    "id": "message-uuid",
    "telnyxMessageId": "telnyx-msg-123",
    "campaignId": "campaign-uuid",
    "from": "+12125551234",
    "to": "+13125551234",
    "text": "Hello! Your order #12345 has been shipped.",
    "direction": "outbound",
    "messageType": "SMS",
    "status": "delivered",
    "cost": 0.0075,
    "segments": 1,
    "encoding": "GSM-7",
    "sentAt": "2024-01-20T10:30:00.000Z",
    "deliveredAt": "2024-01-20T10:30:03.000Z",
    "createdAt": "2024-01-20T10:30:00.000Z",
    "updatedAt": "2024-01-20T10:30:03.000Z"
  }
}
```

### Phone Number Management

#### Search Available Numbers
```http
GET /api/v1/numbers/search?areaCode=212&features=sms&limit=10
Authorization: Bearer {token}
```

**Response (200 OK):**
```json
{
  "success": true,
  "data": {
    "numbers": [
      {
        "phoneNumber": "+12125551234",
        "locality": "New York",
        "region": "NY",
        "countryCode": "US",
        "features": ["voice", "sms", "mms"],
        "pricing": {
          "monthly": 1.00,
          "setup": 0.00
        }
      }
    ],
    "totalAvailable": 45,
    "searchParams": {
      "areaCode": "212",
      "features": ["sms"]
    }
  }
}
```

#### Purchase Phone Number
```http
POST /api/v1/numbers/purchase
Authorization: Bearer {token}
Content-Type: application/json

{
  "phoneNumber": "+12125551234",
  "messagingProfileId": "messaging-profile-123"
}
```

**Response (201 Created):**
```json
{
  "success": true,
  "data": {
    "id": "number-uuid",
    "phoneNumber": "+12125551234",
    "telnyxNumberId": "telnyx-number-123",
    "status": "available",
    "monthlyRentalCost": 1.00,
    "purchasedAt": "2024-01-20T11:00:00.000Z",
    "createdAt": "2024-01-20T11:00:00.000Z"
  }
}
```

#### Assign Number to Campaign
```http
POST /api/v1/numbers/{numberId}/assign
Authorization: Bearer {token}
Content-Type: application/json

{
  "campaignId": "campaign-uuid"
}
```

**Response (200 OK):**
```json
{
  "success": true,
  "data": {
    "id": "number-uuid",
    "phoneNumber": "+12125551234",
    "campaignId": "campaign-uuid",
    "status": "assigned",
    "assignedAt": "2024-01-20T11:05:00.000Z"
  }
}
```

### Opt-out Management

#### Check Opt-out Status
```http
GET /api/v1/opt-outs/check?phoneNumber=%2B13125551234&campaignId=campaign-uuid
Authorization: Bearer {token}
```

**Response (200 OK):**
```json
{
  "success": true,
  "data": {
    "phoneNumber": "+13125551234",
    "isOptedOut": false,
    "optedOutAt": null,
    "campaignId": null,
    "globalOptOut": false
  }
}
```

#### Add Opt-out
```http
POST /api/v1/opt-outs/add
Authorization: Bearer {token}
Content-Type: application/json

{
  "phoneNumber": "+13125551234",
  "campaignId": "campaign-uuid",
  "method": "manual",
  "reason": "Customer request via phone"
}
```

**Response (201 Created):**
```json
{
  "success": true,
  "data": {
    "id": "opt-out-uuid",
    "phoneNumber": "+13125551234",
    "campaignId": "campaign-uuid",
    "optOutMethod": "manual",
    "optedOutAt": "2024-01-20T11:10:00.000Z"
  }
}
```

## Data Models

### Core Data Types

#### User Model
```typescript
interface User {
  id: string;
  email: string;
  firstName: string;
  lastName: string;
  role: 'admin' | 'manager' | 'user';
  isActive: boolean;
  emailVerified: boolean;
  lastLogin?: Date;
  createdAt: Date;
  updatedAt: Date;
}
```

#### Brand Model
```typescript
interface Brand {
  id: string;
  legalBusinessName: string;
  businessType: 'LLC' | 'Corporation' | 'Sole_Proprietor' | 'Partnership' | 'Other';
  einTaxId?: string;
  dunsNumber?: string;
  businessAddress: string;
  website?: string;
  contactPhone: string;
  contactEmail: string;
  telnyxBrandId?: string;
  tcrBrandId?: string;
  trustScore: number;
  status: 'pending' | 'approved' | 'rejected' | 'suspended';
  statusReason?: string;
  registrationDate?: Date;
  approvalDate?: Date;
  createdAt: Date;
  updatedAt: Date;
}
```

#### Campaign Model
```typescript
interface Campaign {
  id: string;
  brandId: string;
  name: string;
  useCase: 'customer_care' | 'marketing' | '2fa' | 'account_notifications' | 'public_service';
  description?: string;
  sampleMessages: string[];
  optInProcess: string;
  optOutInstructions: string;
  telnyxCampaignId?: string;
  tcrCampaignId?: string;
  status: 'pending' | 'approved' | 'rejected' | 'suspended';
  statusReason?: string;
  throughputLimit: number;
  dailyLimit?: number;
  monthlyLimit?: number;
  createdAt: Date;
  updatedAt: Date;
}
```

#### Message Model
```typescript
interface Message {
  id: string;
  telnyxMessageId?: string;
  campaignId?: string;
  phoneNumberId: string;
  fromNumber: string;
  toNumber: string;
  messageText: string;
  direction: 'inbound' | 'outbound';
  messageType: 'SMS' | 'MMS';
  status: 'queued' | 'sent' | 'delivered' | 'failed' | 'received';
  errorCode?: string;
  errorMessage?: string;
  cost?: number;
  segments: number;
  encoding: 'GSM-7' | 'UCS-2';
  mediaUrls?: string[];
  sentAt?: Date;
  deliveredAt?: Date;
  createdAt: Date;
  updatedAt: Date;
}
```

### Request/Response Models

#### Pagination
```typescript
interface PaginationRequest {
  page?: number;
  limit?: number;
  sortBy?: string;
  sortOrder?: 'asc' | 'desc';
}

interface PaginationResponse {
  page: number;
  limit: number;
  total: number;
  totalPages: number;
  hasNext: boolean;
  hasPrev: boolean;
}
```

#### Standard API Response
```typescript
interface ApiResponse<T> {
  success: boolean;
  data?: T;
  error?: string;
  errors?: ValidationError[];
  meta?: {
    version: string;
    timestamp: string;
    requestId: string;
  };
}

interface ValidationError {
  field: string;
  message: string;
  code: string;
}
```

## Error Handling

### Standard HTTP Status Codes
| Code | Meaning | Usage |
|------|---------|-------|
| 200 | OK | Successful GET, PUT, PATCH |
| 201 | Created | Successful POST |
| 202 | Accepted | Async operation started |
| 204 | No Content | Successful DELETE |
| 400 | Bad Request | Invalid request data |
| 401 | Unauthorized | Authentication required |
| 403 | Forbidden | Insufficient permissions |
| 404 | Not Found | Resource doesn't exist |
| 409 | Conflict | Resource conflict |
| 422 | Unprocessable Entity | Validation errors |
| 429 | Too Many Requests | Rate limit exceeded |
| 500 | Internal Server Error | Server error |
| 502 | Bad Gateway | Upstream service error |
| 503 | Service Unavailable | Service temporarily down |

### Error Response Format
```json
{
  "success": false,
  "error": "Validation failed",
  "errors": [
    {
      "field": "phoneNumber",
      "message": "Phone number must be in E.164 format",
      "code": "INVALID_PHONE_FORMAT"
    },
    {
      "field": "campaignId",
      "message": "Campaign must be approved",
      "code": "CAMPAIGN_NOT_APPROVED"
    }
  ],
  "meta": {
    "version": "1.0",
    "timestamp": "2024-01-20T10:30:00.000Z",
    "requestId": "req_1234567890"
  }
}
```

### Error Codes
```typescript
enum ErrorCodes {
  // Authentication
  INVALID_CREDENTIALS = 'INVALID_CREDENTIALS',
  TOKEN_EXPIRED = 'TOKEN_EXPIRED',
  INSUFFICIENT_PERMISSIONS = 'INSUFFICIENT_PERMISSIONS',
  
  // Validation
  INVALID_PHONE_FORMAT = 'INVALID_PHONE_FORMAT',
  INVALID_EMAIL_FORMAT = 'INVALID_EMAIL_FORMAT',
  REQUIRED_FIELD_MISSING = 'REQUIRED_FIELD_MISSING',
  
  // Business Logic
  BRAND_NOT_APPROVED = 'BRAND_NOT_APPROVED',
  CAMPAIGN_NOT_APPROVED = 'CAMPAIGN_NOT_APPROVED',
  RECIPIENT_OPTED_OUT = 'RECIPIENT_OPTED_OUT',
  INSUFFICIENT_BALANCE = 'INSUFFICIENT_BALANCE',
  
  // External Services
  TELNYX_API_ERROR = 'TELNYX_API_ERROR',
  PROVIDER_RATE_LIMITED = 'PROVIDER_RATE_LIMITED',
  
  // System
  DATABASE_ERROR = 'DATABASE_ERROR',
  INTERNAL_ERROR = 'INTERNAL_ERROR'
}
```

## Rate Limiting

### Rate Limit Headers
```http
HTTP/1.1 200 OK
X-RateLimit-Limit: 1000
X-RateLimit-Remaining: 999
X-RateLimit-Reset: 1642694400
X-RateLimit-Window: 3600
```

### Rate Limit Response (429)
```json
{
  "success": false,
  "error": "Rate limit exceeded",
  "meta": {
    "retryAfter": 3600,
    "limit": 1000,
    "window": 3600
  }
}
```

### Rate Limiting Tiers
| Tier | Requests/Hour | Requests/Minute | Messages/Day |
|------|---------------|-----------------|--------------|
| Free | 100 | 10 | 100 |
| Basic | 1,000 | 50 | 1,000 |
| Pro | 10,000 | 200 | 10,000 |
| Enterprise | 100,000 | 1,000 | 100,000 |

## Webhooks

### Webhook Configuration
```http
POST /api/v1/webhooks/configure
Authorization: Bearer {token}
Content-Type: application/json

{
  "url": "https://your-app.com/webhooks/telnyx",
  "events": [
    "message.sent",
    "message.delivered",
    "message.failed",
    "message.received"
  ],
  "secret": "webhook-secret-key"
}
```

### Webhook Event Format
```json
{
  "eventType": "message.delivered",
  "eventId": "event-uuid",
  "timestamp": "2024-01-20T10:30:00.000Z",
  "data": {
    "messageId": "message-uuid",
    "telnyxMessageId": "telnyx-msg-123",
    "status": "delivered",
    "deliveredAt": "2024-01-20T10:30:03.000Z",
    "cost": 0.0075
  },
  "signature": "sha256=abc123..."
}
```

### Webhook Verification
```javascript
const crypto = require('crypto');

function verifyWebhook(payload, signature, secret) {
  const expectedSignature = crypto
    .createHmac('sha256', secret)
    .update(payload)
    .digest('hex');
  
  return crypto.timingSafeEqual(
    Buffer.from(signature, 'hex'),
    Buffer.from(expectedSignature, 'hex')
  );
}
```

## Future API Features

### Version 1.1 Features (Q2 2024)

#### Message Templates
```http
POST /api/v1/templates
Authorization: Bearer {token}

{
  "campaignId": "campaign-uuid",
  "name": "Order Confirmation",
  "content": "Hi {{firstName}}, your order #{{orderNumber}} totaling ${{amount}} has been confirmed!",
  "variables": [
    {"name": "firstName", "type": "string", "required": true},
    {"name": "orderNumber", "type": "string", "required": true},
    {"name": "amount", "type": "number", "required": true}
  ]
}
```

#### Scheduled Messages
```http
POST /api/v1/messages/schedule
Authorization: Bearer {token}

{
  "templateId": "template-uuid",
  "recipients": [
    {
      "phoneNumber": "+13125551234",
      "variables": {
        "firstName": "John",
        "orderNumber": "12345",
        "amount": 99.99
      }
    }
  ],
  "scheduledAt": "2024-01-21T09:00:00.000Z",
  "timezone": "America/New_York"
}
```

#### Message Analytics
```http
GET /api/v1/campaigns/{campaignId}/analytics?period=30d&metrics=delivery_rate,opt_out_rate
Authorization: Bearer {token}
```

### Version 2.0 Features (Q3 2024)

#### Multi-Provider Support
```http
POST /api/v1/providers
Authorization: Bearer {token}

{
  "name": "Twilio Backup",
  "type": "twilio",
  "configuration": {
    "accountSid": "ACxxxx",
    "authToken": "xxxxx"
  },
  "priority": 2,
  "failoverEnabled": true
}
```

#### Advanced Routing
```http
POST /api/v1/routing-rules
Authorization: Bearer {token}

{
  "campaignId": "campaign-uuid",
  "rules": [
    {
      "condition": {"areaCode": "212"},
      "providerId": "provider-1",
      "priority": 1
    },
    {
      "condition": {"messageType": "MMS"},
      "providerId": "provider-2",
      "priority": 1
    }
  ]
}
```

### Version 2.1 Features (Q4 2024)

#### AI-Powered Features
```http
POST /api/v1/messages/optimize
Authorization: Bearer {token}

{
  "campaignId": "campaign-uuid",
  "message": "Your order has been processed",
  "optimizations": ["sentiment", "deliverability", "engagement"]
}
```

#### Predictive Analytics
```http
GET /api/v1/campaigns/{campaignId}/predictions?horizon=7d
Authorization: Bearer {token}
```

## SDK Development

### JavaScript/TypeScript SDK
```typescript
import { TelnyxSMS } from '@your-org/telnyx-sms-sdk';

const client = new TelnyxSMS({
  apiKey: 'your-api-key',
  baseUrl: 'https://api.yourdomain.com/v1'
});

// Send message
const message = await client.messages.send({
  campaignId: 'campaign-uuid',
  from: '+12125551234',
  to: '+13125551234',
  text: 'Hello from SDK!'
});

// Check opt-out status
const optOutStatus = await client.optOuts.check('+13125551234', 'campaign-uuid');
```

### Python SDK
```python
from telnyx_sms import TelnyxSMS

client = TelnyxSMS(
    api_key='your-api-key',
    base_url='https://api.yourdomain.com/v1'
)

# Send message
message = client.messages.send(
    campaign_id='campaign-uuid',
    from_number='+12125551234',
    to_number='+13125551234',
    text='Hello from Python SDK!'
)

# List campaigns
campaigns = client.campaigns.list(brand_id='brand-uuid')
```

### Go SDK
```go
package main

import (
    "github.com/your-org/telnyx-sms-go"
)

func main() {
    client := telnyxsms.NewClient("your-api-key")
    
    message, err := client.Messages.Send(&telnyxsms.SendMessageRequest{
        CampaignID: "campaign-uuid",
        From:       "+12125551234",
        To:         "+13125551234",
        Text:       "Hello from Go SDK!",
    })
}
```

### SDK Features Roadmap
- **Phase 1**: Core CRUD operations, authentication
- **Phase 2**: Webhook helpers, retry logic, rate limiting
- **Phase 3**: Advanced features (templates, scheduling, analytics)
- **Phase 4**: AI integrations, predictive features

## API Documentation Tools

### OpenAPI Specification
```yaml
openapi: 3.0.3
info:
  title: 10DLC SMS API
  version: 1.0.0
  description: Comprehensive API for 10DLC SMS messaging
  contact:
    name: API Support
    email: api-support@yourdomain.com
  license:
    name: MIT
    url: https://opensource.org/licenses/MIT

servers:
  - url: https://api.yourdomain.com/v1
    description: Production server
  - url: https://staging-api.yourdomain.com/v1
    description: Staging server

security:
  - bearerAuth: []
  - apiKeyAuth: []

components:
  securitySchemes:
    bearerAuth:
      type: http
      scheme: bearer
      bearerFormat: JWT
    apiKeyAuth:
      type: apiKey
      in: header
      name: Authorization
```

### Interactive Documentation
- **Swagger UI**: Auto-generated from OpenAPI spec
- **Postman Collection**: Pre-configured API collection
- **Code Examples**: Multi-language code samples
- **Sandbox Environment**: Live testing environment

## Conclusion

This API specification provides:

- **Comprehensive Coverage**: All system functionality exposed via REST API
- **Developer-Friendly**: Clear documentation, consistent patterns
- **Scalable Design**: Support for future features and integrations
- **Security-First**: Robust authentication and authorization
- **Standards-Compliant**: RESTful design following industry best practices

The API is designed to grow with the platform while maintaining backwards compatibility and providing a smooth developer experience.
