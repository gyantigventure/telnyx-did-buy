# 10DLC SMS System - TypeScript + MySQL Development Guide

## Table of Contents
1. [System Overview](#system-overview)
2. [Technology Stack](#technology-stack)
3. [Database Schema](#database-schema)
4. [Project Structure](#project-structure)
5. [API Design](#api-design)
6. [Implementation Guide](#implementation-guide)
7. [Security Considerations](#security-considerations)
8. [Testing Strategy](#testing-strategy)
9. [Deployment](#deployment)
10. [Monitoring & Maintenance](#monitoring--maintenance)

## System Overview

This document outlines the development of a comprehensive 10DLC SMS messaging system using TypeScript and MySQL. The system will handle:

- Brand and Campaign registration with Telnyx
- Phone number management and DID purchasing
- SMS sending and receiving via Telnyx API
- Compliance monitoring and opt-out management
- Delivery reports and analytics
- User management and authentication

### Key Features
- **Brand Management**: Register and manage brands with The Campaign Registry (TCR)
- **Campaign Management**: Create and manage messaging campaigns with specific use cases
- **Number Management**: Purchase, assign, and manage phone numbers
- **Message Processing**: Send/receive SMS with proper compliance checks
- **Opt-out Handling**: Automatic opt-out processing and compliance
- **Reporting**: Comprehensive delivery reports and analytics
- **Webhook Processing**: Handle Telnyx webhooks for message status updates

## Technology Stack

### Backend
- **Runtime**: Node.js 18+
- **Framework**: Express.js with TypeScript
- **Database**: MySQL 8.0+
- **ORM**: TypeORM
- **Authentication**: JWT + bcrypt
- **Validation**: Joi or Zod
- **API Documentation**: Swagger/OpenAPI
- **Testing**: Jest + Supertest
- **Process Management**: PM2

### External Services
- **SMS Provider**: Telnyx API
- **Environment**: Docker + Docker Compose
- **Monitoring**: Winston (logging) + Prometheus (metrics)

### Development Tools
- **TypeScript**: 5.0+
- **ESLint**: Code linting
- **Prettier**: Code formatting
- **Husky**: Git hooks
- **Nodemon**: Development server

## Database Schema

### Core Tables

```sql
-- Brands table for TCR registration
CREATE TABLE brands (
    id VARCHAR(36) PRIMARY KEY,
    legal_business_name VARCHAR(255) NOT NULL,
    business_type ENUM('LLC', 'Corporation', 'Sole_Proprietor', 'Partnership', 'Other') NOT NULL,
    ein_tax_id VARCHAR(50),
    duns_number VARCHAR(20),
    business_address TEXT NOT NULL,
    website VARCHAR(255),
    contact_phone VARCHAR(20) NOT NULL,
    contact_email VARCHAR(255) NOT NULL,
    telnyx_brand_id VARCHAR(100) UNIQUE,
    tcr_brand_id VARCHAR(100) UNIQUE,
    trust_score INT DEFAULT 0,
    status ENUM('pending', 'approved', 'rejected', 'suspended') DEFAULT 'pending',
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    INDEX idx_status (status),
    INDEX idx_telnyx_brand_id (telnyx_brand_id)
);

-- Campaigns table for messaging campaigns
CREATE TABLE campaigns (
    id VARCHAR(36) PRIMARY KEY,
    brand_id VARCHAR(36) NOT NULL,
    name VARCHAR(255) NOT NULL,
    use_case ENUM('customer_care', 'marketing', '2fa', 'account_notifications', 'public_service') NOT NULL,
    description TEXT,
    sample_messages JSON,
    opt_in_process TEXT NOT NULL,
    opt_out_instructions VARCHAR(500) NOT NULL DEFAULT 'Reply STOP to unsubscribe',
    telnyx_campaign_id VARCHAR(100) UNIQUE,
    tcr_campaign_id VARCHAR(100) UNIQUE,
    status ENUM('pending', 'approved', 'rejected', 'suspended') DEFAULT 'pending',
    throughput_limit INT DEFAULT 1,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    FOREIGN KEY (brand_id) REFERENCES brands(id) ON DELETE CASCADE,
    INDEX idx_brand_id (brand_id),
    INDEX idx_status (status),
    INDEX idx_use_case (use_case)
);

-- Phone numbers table
CREATE TABLE phone_numbers (
    id VARCHAR(36) PRIMARY KEY,
    phone_number VARCHAR(20) NOT NULL UNIQUE,
    campaign_id VARCHAR(36),
    telnyx_number_id VARCHAR(100) UNIQUE,
    messaging_profile_id VARCHAR(100),
    status ENUM('available', 'assigned', 'suspended') DEFAULT 'available',
    purchased_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    assigned_at TIMESTAMP NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    FOREIGN KEY (campaign_id) REFERENCES campaigns(id) ON DELETE SET NULL,
    INDEX idx_phone_number (phone_number),
    INDEX idx_campaign_id (campaign_id),
    INDEX idx_status (status)
);

-- Messages table for sent/received messages
CREATE TABLE messages (
    id VARCHAR(36) PRIMARY KEY,
    telnyx_message_id VARCHAR(100) UNIQUE,
    campaign_id VARCHAR(36),
    phone_number_id VARCHAR(36) NOT NULL,
    from_number VARCHAR(20) NOT NULL,
    to_number VARCHAR(20) NOT NULL,
    message_text TEXT NOT NULL,
    direction ENUM('inbound', 'outbound') NOT NULL,
    message_type ENUM('SMS', 'MMS') DEFAULT 'SMS',
    status ENUM('queued', 'sent', 'delivered', 'failed', 'received') NOT NULL,
    error_code VARCHAR(50),
    error_message TEXT,
    cost DECIMAL(10, 6),
    segments INT DEFAULT 1,
    media_urls JSON,
    webhook_data JSON,
    sent_at TIMESTAMP NULL,
    delivered_at TIMESTAMP NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    FOREIGN KEY (campaign_id) REFERENCES campaigns(id) ON DELETE SET NULL,
    FOREIGN KEY (phone_number_id) REFERENCES phone_numbers(id) ON DELETE CASCADE,
    INDEX idx_telnyx_message_id (telnyx_message_id),
    INDEX idx_campaign_id (campaign_id),
    INDEX idx_phone_numbers (from_number, to_number),
    INDEX idx_direction (direction),
    INDEX idx_status (status),
    INDEX idx_created_at (created_at)
);

-- Opt-outs table for compliance
CREATE TABLE opt_outs (
    id VARCHAR(36) PRIMARY KEY,
    phone_number VARCHAR(20) NOT NULL,
    campaign_id VARCHAR(36),
    brand_id VARCHAR(36),
    opted_out_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    opt_out_method ENUM('sms_reply', 'manual', 'api') DEFAULT 'sms_reply',
    message_id VARCHAR(36),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (campaign_id) REFERENCES campaigns(id) ON DELETE SET NULL,
    FOREIGN KEY (brand_id) REFERENCES brands(id) ON DELETE SET NULL,
    FOREIGN KEY (message_id) REFERENCES messages(id) ON DELETE SET NULL,
    UNIQUE KEY unique_phone_campaign (phone_number, campaign_id),
    INDEX idx_phone_number (phone_number),
    INDEX idx_campaign_id (campaign_id),
    INDEX idx_opted_out_at (opted_out_at)
);

-- Users table for system access
CREATE TABLE users (
    id VARCHAR(36) PRIMARY KEY,
    email VARCHAR(255) NOT NULL UNIQUE,
    password_hash VARCHAR(255) NOT NULL,
    first_name VARCHAR(100) NOT NULL,
    last_name VARCHAR(100) NOT NULL,
    role ENUM('admin', 'manager', 'user') DEFAULT 'user',
    is_active BOOLEAN DEFAULT TRUE,
    last_login TIMESTAMP NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    INDEX idx_email (email),
    INDEX idx_role (role),
    INDEX idx_is_active (is_active)
);

-- User brand associations
CREATE TABLE user_brands (
    id VARCHAR(36) PRIMARY KEY,
    user_id VARCHAR(36) NOT NULL,
    brand_id VARCHAR(36) NOT NULL,
    role ENUM('owner', 'admin', 'member') DEFAULT 'member',
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (user_id) REFERENCES users(id) ON DELETE CASCADE,
    FOREIGN KEY (brand_id) REFERENCES brands(id) ON DELETE CASCADE,
    UNIQUE KEY unique_user_brand (user_id, brand_id),
    INDEX idx_user_id (user_id),
    INDEX idx_brand_id (brand_id)
);

-- API keys for external access
CREATE TABLE api_keys (
    id VARCHAR(36) PRIMARY KEY,
    user_id VARCHAR(36) NOT NULL,
    key_name VARCHAR(100) NOT NULL,
    api_key VARCHAR(255) NOT NULL UNIQUE,
    permissions JSON,
    is_active BOOLEAN DEFAULT TRUE,
    last_used TIMESTAMP NULL,
    expires_at TIMESTAMP NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    FOREIGN KEY (user_id) REFERENCES users(id) ON DELETE CASCADE,
    INDEX idx_api_key (api_key),
    INDEX idx_user_id (user_id),
    INDEX idx_is_active (is_active)
);

-- Webhook events log
CREATE TABLE webhook_events (
    id VARCHAR(36) PRIMARY KEY,
    event_type VARCHAR(100) NOT NULL,
    resource_id VARCHAR(100),
    payload JSON NOT NULL,
    processed BOOLEAN DEFAULT FALSE,
    processing_error TEXT,
    retry_count INT DEFAULT 0,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    processed_at TIMESTAMP NULL,
    INDEX idx_event_type (event_type),
    INDEX idx_processed (processed),
    INDEX idx_created_at (created_at)
);
```

## Project Structure

```
src/
├── controllers/           # Route controllers
│   ├── auth.controller.ts
│   ├── brand.controller.ts
│   ├── campaign.controller.ts
│   ├── message.controller.ts
│   ├── number.controller.ts
│   └── webhook.controller.ts
├── entities/             # TypeORM entities
│   ├── Brand.ts
│   ├── Campaign.ts
│   ├── Message.ts
│   ├── OptOut.ts
│   ├── PhoneNumber.ts
│   ├── User.ts
│   └── WebhookEvent.ts
├── services/             # Business logic
│   ├── auth.service.ts
│   ├── brand.service.ts
│   ├── campaign.service.ts
│   ├── message.service.ts
│   ├── telnyx.service.ts
│   └── webhook.service.ts
├── middleware/           # Express middleware
│   ├── auth.middleware.ts
│   ├── validation.middleware.ts
│   ├── error.middleware.ts
│   └── rate-limit.middleware.ts
├── routes/              # API routes
│   ├── auth.routes.ts
│   ├── brand.routes.ts
│   ├── campaign.routes.ts
│   ├── message.routes.ts
│   ├── number.routes.ts
│   └── webhook.routes.ts
├── types/               # TypeScript type definitions
│   ├── api.types.ts
│   ├── telnyx.types.ts
│   └── common.types.ts
├── utils/               # Utility functions
│   ├── crypto.util.ts
│   ├── phone.util.ts
│   ├── validation.util.ts
│   └── logger.util.ts
├── config/              # Configuration
│   ├── database.config.ts
│   ├── telnyx.config.ts
│   └── app.config.ts
├── migrations/          # Database migrations
└── tests/              # Test files
    ├── unit/
    ├── integration/
    └── fixtures/

config/
├── docker-compose.yml
├── Dockerfile
└── nginx.conf

docs/
├── api/                # API documentation
└── deployment/         # Deployment guides
```

## API Design

### Core TypeScript Interfaces

```typescript
// types/api.types.ts

export interface CreateBrandRequest {
  legalBusinessName: string;
  businessType: 'LLC' | 'Corporation' | 'Sole_Proprietor' | 'Partnership' | 'Other';
  einTaxId?: string;
  dunsNumber?: string;
  businessAddress: string;
  website?: string;
  contactPhone: string;
  contactEmail: string;
}

export interface CreateCampaignRequest {
  brandId: string;
  name: string;
  useCase: 'customer_care' | 'marketing' | '2fa' | 'account_notifications' | 'public_service';
  description?: string;
  sampleMessages: string[];
  optInProcess: string;
  optOutInstructions?: string;
}

export interface SendMessageRequest {
  campaignId: string;
  from: string;
  to: string;
  text: string;
  mediaUrls?: string[];
}

export interface MessageResponse {
  id: string;
  telnyxMessageId?: string;
  status: 'queued' | 'sent' | 'delivered' | 'failed';
  errorMessage?: string;
  cost?: number;
  segments: number;
  sentAt?: Date;
}

export interface OptOutCheck {
  phoneNumber: string;
  isOptedOut: boolean;
  optedOutAt?: Date;
  campaignId?: string;
}

// types/telnyx.types.ts

export interface TelnyxBrand {
  id: string;
  displayName: string;
  companyName: string;
  ein?: string;
  entityType: string;
  vertical: string;
  altBusinessId?: string;
  altBusinessIdType?: string;
  website?: string;
  stockSymbol?: string;
  stockExchange?: string;
  ipAddress?: string;
  city?: string;
  state?: string;
  zipcode?: string;
  country?: string;
  email?: string;
  phone?: string;
  businessContactFirstName?: string;
  businessContactLastName?: string;
  businessContactEmail?: string;
  businessContactPhone?: string;
  referenceId?: string;
  mock?: boolean;
}

export interface TelnyxCampaign {
  id: string;
  brandId: string;
  campaignId: string;
  cspId: string;
  resellerId?: string;
  usecase: string;
  subUsecases?: string[];
  description: string;
  embeddedLink: boolean;
  embeddedPhone: boolean;
  affiliateMarketing: boolean;
  numberPool: boolean;
  ageGated: boolean;
  directLending: boolean;
  subscriberOptin: boolean;
  subscriberOptout: boolean;
  subscriberHelp: boolean;
  sample1?: string;
  sample2?: string;
  sample3?: string;
  sample4?: string;
  sample5?: string;
  messageFlow?: string;
  helpMessage?: string;
  optinKeywords?: string;
  optoutKeywords?: string;
  helpKeywords?: string;
  optinMessage?: string;
  optoutMessage?: string;
  referenceId?: string;
  mock?: boolean;
}

export interface TelnyxMessage {
  id: string;
  recordType: 'message';
  direction: 'inbound' | 'outbound';
  from: string;
  to: string;
  text?: string;
  media?: Array<{
    url: string;
    contentType: string;
    size: number;
  }>;
  webhookUrl?: string;
  webhookFailoverUrl?: string;
  encoding: 'GSM-7' | 'UCS-2';
  parts: number;
  tags?: string[];
  cost?: {
    amount: string;
    currency: string;
  };
  receivedAt?: string;
  sentAt?: string;
  completedAt?: string;
  validUntil?: string;
  errors?: Array<{
    code: number;
    title: string;
    detail: string;
  }>;
  type: 'SMS' | 'MMS';
}

export interface TelnyxWebhookEvent {
  data: {
    eventType: string;
    id: string;
    occurredAt: string;
    payload: TelnyxMessage | any;
    recordType: string;
  };
  meta: {
    attempt: number;
    deliveredTo: string;
  };
}
```

### REST API Endpoints

```typescript
// routes/brand.routes.ts

/**
 * @swagger
 * /api/brands:
 *   post:
 *     summary: Create a new brand
 *     tags: [Brands]
 *     requestBody:
 *       required: true
 *       content:
 *         application/json:
 *           schema:
 *             $ref: '#/components/schemas/CreateBrandRequest'
 *     responses:
 *       201:
 *         description: Brand created successfully
 *       400:
 *         description: Invalid request data
 *       401:
 *         description: Unauthorized
 */
router.post('/', auth, validateBrand, createBrand);

/**
 * @swagger
 * /api/brands:
 *   get:
 *     summary: Get all brands for authenticated user
 *     tags: [Brands]
 *     responses:
 *       200:
 *         description: List of brands
 */
router.get('/', auth, getBrands);

/**
 * @swagger
 * /api/brands/{id}:
 *   get:
 *     summary: Get brand by ID
 *     tags: [Brands]
 *     parameters:
 *       - in: path
 *         name: id
 *         required: true
 *         schema:
 *           type: string
 *     responses:
 *       200:
 *         description: Brand details
 *       404:
 *         description: Brand not found
 */
router.get('/:id', auth, getBrandById);

/**
 * @swagger
 * /api/brands/{id}:
 *   put:
 *     summary: Update brand
 *     tags: [Brands]
 *     parameters:
 *       - in: path
 *         name: id
 *         required: true
 *         schema:
 *           type: string
 *     requestBody:
 *       required: true
 *       content:
 *         application/json:
 *           schema:
 *             $ref: '#/components/schemas/UpdateBrandRequest'
 *     responses:
 *       200:
 *         description: Brand updated successfully
 */
router.put('/:id', auth, validateBrand, updateBrand);

/**
 * @swagger
 * /api/brands/{id}:
 *   delete:
 *     summary: Delete brand
 *     tags: [Brands]
 *     parameters:
 *       - in: path
 *         name: id
 *         required: true
 *         schema:
 *           type: string
 *     responses:
 *       204:
 *         description: Brand deleted successfully
 */
router.delete('/:id', auth, deleteBrand);
```

## Implementation Guide

### Step 1: Project Setup

```bash
# Initialize project
mkdir telnyx-10dlc-system
cd telnyx-10dlc-system
npm init -y

# Install dependencies
npm install express typeorm mysql2 jsonwebtoken bcryptjs joi cors helmet morgan winston
npm install telnyx uuid dotenv express-rate-limit swagger-jsdoc swagger-ui-express

# Install dev dependencies
npm install -D typescript @types/node @types/express @types/jsonwebtoken @types/bcryptjs
npm install -D @types/cors @types/uuid @types/swagger-jsdoc @types/swagger-ui-express
npm install -D nodemon ts-node eslint prettier husky jest supertest @types/jest
npm install -D @typescript-eslint/eslint-plugin @typescript-eslint/parser

# Initialize TypeScript
npx tsc --init
```

**package.json scripts:**
```json
{
  "scripts": {
    "dev": "nodemon src/app.ts",
    "build": "tsc",
    "start": "node dist/app.js",
    "test": "jest",
    "test:watch": "jest --watch",
    "lint": "eslint src/**/*.ts",
    "lint:fix": "eslint src/**/*.ts --fix",
    "format": "prettier --write src/**/*.ts"
  }
}
```

**tsconfig.json:**
```json
{
  "compilerOptions": {
    "target": "ES2020",
    "module": "commonjs",
    "lib": ["ES2020"],
    "outDir": "./dist",
    "rootDir": "./src",
    "strict": true,
    "esModuleInterop": true,
    "skipLibCheck": true,
    "forceConsistentCasingInFileNames": true,
    "resolveJsonModule": true,
    "experimentalDecorators": true,
    "emitDecoratorMetadata": true,
    "strictPropertyInitialization": false,
    "allowSyntheticDefaultImports": true
  },
  "include": ["src/**/*"],
  "exclude": ["node_modules", "dist", "tests"]
}
```

### Step 2: Environment Configuration

**.env file:**
```bash
# Application
NODE_ENV=development
PORT=3000
JWT_SECRET=your-super-secret-jwt-key-here
JWT_EXPIRES_IN=7d

# Database
DB_HOST=localhost
DB_PORT=3306
DB_USERNAME=root
DB_PASSWORD=password
DB_DATABASE=telnyx_10dlc

# Telnyx
TELNYX_API_KEY=your-telnyx-api-key
TELNYX_WEBHOOK_SECRET=your-webhook-secret
TELNYX_BASE_URL=https://api.telnyx.com/v2

# Rate Limiting
RATE_LIMIT_WINDOW_MS=900000
RATE_LIMIT_MAX_REQUESTS=100

# Logging
LOG_LEVEL=info
LOG_FILE=logs/app.log
```

**src/config/app.config.ts:**
```typescript
import dotenv from 'dotenv';

dotenv.config();

export const config = {
  app: {
    port: parseInt(process.env.PORT || '3000', 10),
    env: process.env.NODE_ENV || 'development',
    jwtSecret: process.env.JWT_SECRET || 'fallback-secret',
    jwtExpiresIn: process.env.JWT_EXPIRES_IN || '7d',
  },
  database: {
    host: process.env.DB_HOST || 'localhost',
    port: parseInt(process.env.DB_PORT || '3306', 10),
    username: process.env.DB_USERNAME || 'root',
    password: process.env.DB_PASSWORD || '',
    database: process.env.DB_DATABASE || 'telnyx_10dlc',
  },
  telnyx: {
    apiKey: process.env.TELNYX_API_KEY || '',
    webhookSecret: process.env.TELNYX_WEBHOOK_SECRET || '',
    baseUrl: process.env.TELNYX_BASE_URL || 'https://api.telnyx.com/v2',
  },
  rateLimit: {
    windowMs: parseInt(process.env.RATE_LIMIT_WINDOW_MS || '900000', 10),
    maxRequests: parseInt(process.env.RATE_LIMIT_MAX_REQUESTS || '100', 10),
  },
  logging: {
    level: process.env.LOG_LEVEL || 'info',
    file: process.env.LOG_FILE || 'logs/app.log',
  },
};
```

### Step 3: Database Setup with TypeORM

**src/config/database.config.ts:**
```typescript
import { DataSource } from 'typeorm';
import { config } from './app.config';
import { Brand } from '../entities/Brand';
import { Campaign } from '../entities/Campaign';
import { PhoneNumber } from '../entities/PhoneNumber';
import { Message } from '../entities/Message';
import { OptOut } from '../entities/OptOut';
import { User } from '../entities/User';
import { UserBrand } from '../entities/UserBrand';
import { ApiKey } from '../entities/ApiKey';
import { WebhookEvent } from '../entities/WebhookEvent';

export const AppDataSource = new DataSource({
  type: 'mysql',
  host: config.database.host,
  port: config.database.port,
  username: config.database.username,
  password: config.database.password,
  database: config.database.database,
  synchronize: config.app.env === 'development',
  logging: config.app.env === 'development',
  entities: [
    Brand,
    Campaign,
    PhoneNumber,
    Message,
    OptOut,
    User,
    UserBrand,
    ApiKey,
    WebhookEvent,
  ],
  migrations: ['src/migrations/*.ts'],
  subscribers: ['src/subscribers/*.ts'],
});
```

### Step 4: Entity Definitions

**src/entities/Brand.ts:**
```typescript
import {
  Entity,
  PrimaryGeneratedColumn,
  Column,
  CreateDateColumn,
  UpdateDateColumn,
  OneToMany,
  Index,
} from 'typeorm';
import { Campaign } from './Campaign';
import { UserBrand } from './UserBrand';
import { OptOut } from './OptOut';

export enum BusinessType {
  LLC = 'LLC',
  CORPORATION = 'Corporation',
  SOLE_PROPRIETOR = 'Sole_Proprietor',
  PARTNERSHIP = 'Partnership',
  OTHER = 'Other',
}

export enum BrandStatus {
  PENDING = 'pending',
  APPROVED = 'approved',
  REJECTED = 'rejected',
  SUSPENDED = 'suspended',
}

@Entity('brands')
export class Brand {
  @PrimaryGeneratedColumn('uuid')
  id: string;

  @Column({ name: 'legal_business_name', length: 255 })
  legalBusinessName: string;

  @Column({
    name: 'business_type',
    type: 'enum',
    enum: BusinessType,
  })
  businessType: BusinessType;

  @Column({ name: 'ein_tax_id', length: 50, nullable: true })
  einTaxId?: string;

  @Column({ name: 'duns_number', length: 20, nullable: true })
  dunsNumber?: string;

  @Column({ name: 'business_address', type: 'text' })
  businessAddress: string;

  @Column({ length: 255, nullable: true })
  website?: string;

  @Column({ name: 'contact_phone', length: 20 })
  contactPhone: string;

  @Column({ name: 'contact_email', length: 255 })
  contactEmail: string;

  @Column({ name: 'telnyx_brand_id', length: 100, unique: true, nullable: true })
  @Index()
  telnyxBrandId?: string;

  @Column({ name: 'tcr_brand_id', length: 100, unique: true, nullable: true })
  tcrBrandId?: string;

  @Column({ name: 'trust_score', default: 0 })
  trustScore: number;

  @Column({
    type: 'enum',
    enum: BrandStatus,
    default: BrandStatus.PENDING,
  })
  @Index()
  status: BrandStatus;

  @CreateDateColumn({ name: 'created_at' })
  createdAt: Date;

  @UpdateDateColumn({ name: 'updated_at' })
  updatedAt: Date;

  @OneToMany(() => Campaign, (campaign) => campaign.brand)
  campaigns: Campaign[];

  @OneToMany(() => UserBrand, (userBrand) => userBrand.brand)
  userBrands: UserBrand[];

  @OneToMany(() => OptOut, (optOut) => optOut.brand)
  optOuts: OptOut[];
}
```

**src/entities/Campaign.ts:**
```typescript
import {
  Entity,
  PrimaryGeneratedColumn,
  Column,
  CreateDateColumn,
  UpdateDateColumn,
  ManyToOne,
  OneToMany,
  JoinColumn,
  Index,
} from 'typeorm';
import { Brand } from './Brand';
import { PhoneNumber } from './PhoneNumber';
import { Message } from './Message';
import { OptOut } from './OptOut';

export enum CampaignUseCase {
  CUSTOMER_CARE = 'customer_care',
  MARKETING = 'marketing',
  TWO_FA = '2fa',
  ACCOUNT_NOTIFICATIONS = 'account_notifications',
  PUBLIC_SERVICE = 'public_service',
}

export enum CampaignStatus {
  PENDING = 'pending',
  APPROVED = 'approved',
  REJECTED = 'rejected',
  SUSPENDED = 'suspended',
}

@Entity('campaigns')
export class Campaign {
  @PrimaryGeneratedColumn('uuid')
  id: string;

  @Column({ name: 'brand_id' })
  @Index()
  brandId: string;

  @Column({ length: 255 })
  name: string;

  @Column({
    name: 'use_case',
    type: 'enum',
    enum: CampaignUseCase,
  })
  @Index()
  useCase: CampaignUseCase;

  @Column({ type: 'text', nullable: true })
  description?: string;

  @Column({ name: 'sample_messages', type: 'json' })
  sampleMessages: string[];

  @Column({ name: 'opt_in_process', type: 'text' })
  optInProcess: string;

  @Column({
    name: 'opt_out_instructions',
    length: 500,
    default: 'Reply STOP to unsubscribe',
  })
  optOutInstructions: string;

  @Column({ name: 'telnyx_campaign_id', length: 100, unique: true, nullable: true })
  telnyxCampaignId?: string;

  @Column({ name: 'tcr_campaign_id', length: 100, unique: true, nullable: true })
  tcrCampaignId?: string;

  @Column({
    type: 'enum',
    enum: CampaignStatus,
    default: CampaignStatus.PENDING,
  })
  @Index()
  status: CampaignStatus;

  @Column({ name: 'throughput_limit', default: 1 })
  throughputLimit: number;

  @CreateDateColumn({ name: 'created_at' })
  createdAt: Date;

  @UpdateDateColumn({ name: 'updated_at' })
  updatedAt: Date;

  @ManyToOne(() => Brand, (brand) => brand.campaigns, { onDelete: 'CASCADE' })
  @JoinColumn({ name: 'brand_id' })
  brand: Brand;

  @OneToMany(() => PhoneNumber, (phoneNumber) => phoneNumber.campaign)
  phoneNumbers: PhoneNumber[];

  @OneToMany(() => Message, (message) => message.campaign)
  messages: Message[];

  @OneToMany(() => OptOut, (optOut) => optOut.campaign)
  optOuts: OptOut[];
}
```

**src/entities/Message.ts:**
```typescript
import {
  Entity,
  PrimaryGeneratedColumn,
  Column,
  CreateDateColumn,
  UpdateDateColumn,
  ManyToOne,
  JoinColumn,
  Index,
} from 'typeorm';
import { Campaign } from './Campaign';
import { PhoneNumber } from './PhoneNumber';

export enum MessageDirection {
  INBOUND = 'inbound',
  OUTBOUND = 'outbound',
}

export enum MessageType {
  SMS = 'SMS',
  MMS = 'MMS',
}

export enum MessageStatus {
  QUEUED = 'queued',
  SENT = 'sent',
  DELIVERED = 'delivered',
  FAILED = 'failed',
  RECEIVED = 'received',
}

@Entity('messages')
export class Message {
  @PrimaryGeneratedColumn('uuid')
  id: string;

  @Column({ name: 'telnyx_message_id', length: 100, unique: true, nullable: true })
  @Index()
  telnyxMessageId?: string;

  @Column({ name: 'campaign_id', nullable: true })
  @Index()
  campaignId?: string;

  @Column({ name: 'phone_number_id' })
  phoneNumberId: string;

  @Column({ name: 'from_number', length: 20 })
  @Index()
  fromNumber: string;

  @Column({ name: 'to_number', length: 20 })
  @Index()
  toNumber: string;

  @Column({ name: 'message_text', type: 'text' })
  messageText: string;

  @Column({
    type: 'enum',
    enum: MessageDirection,
  })
  @Index()
  direction: MessageDirection;

  @Column({
    name: 'message_type',
    type: 'enum',
    enum: MessageType,
    default: MessageType.SMS,
  })
  messageType: MessageType;

  @Column({
    type: 'enum',
    enum: MessageStatus,
  })
  @Index()
  status: MessageStatus;

  @Column({ name: 'error_code', length: 50, nullable: true })
  errorCode?: string;

  @Column({ name: 'error_message', type: 'text', nullable: true })
  errorMessage?: string;

  @Column({ type: 'decimal', precision: 10, scale: 6, nullable: true })
  cost?: number;

  @Column({ default: 1 })
  segments: number;

  @Column({ name: 'media_urls', type: 'json', nullable: true })
  mediaUrls?: string[];

  @Column({ name: 'webhook_data', type: 'json', nullable: true })
  webhookData?: any;

  @Column({ name: 'sent_at', nullable: true })
  sentAt?: Date;

  @Column({ name: 'delivered_at', nullable: true })
  deliveredAt?: Date;

  @CreateDateColumn({ name: 'created_at' })
  @Index()
  createdAt: Date;

  @UpdateDateColumn({ name: 'updated_at' })
  updatedAt: Date;

  @ManyToOne(() => Campaign, (campaign) => campaign.messages, { onDelete: 'SET NULL' })
  @JoinColumn({ name: 'campaign_id' })
  campaign?: Campaign;

  @ManyToOne(() => PhoneNumber, (phoneNumber) => phoneNumber.messages, { onDelete: 'CASCADE' })
  @JoinColumn({ name: 'phone_number_id' })
  phoneNumber: PhoneNumber;
}
```

**src/entities/PhoneNumber.ts:**
```typescript
import {
  Entity,
  PrimaryGeneratedColumn,
  Column,
  CreateDateColumn,
  UpdateDateColumn,
  ManyToOne,
  OneToMany,
  JoinColumn,
  Index,
} from 'typeorm';
import { Campaign } from './Campaign';
import { Message } from './Message';

export enum PhoneNumberStatus {
  AVAILABLE = 'available',
  ASSIGNED = 'assigned',
  SUSPENDED = 'suspended',
}

@Entity('phone_numbers')
export class PhoneNumber {
  @PrimaryGeneratedColumn('uuid')
  id: string;

  @Column({ name: 'phone_number', length: 20, unique: true })
  @Index()
  phoneNumber: string;

  @Column({ name: 'campaign_id', nullable: true })
  @Index()
  campaignId?: string;

  @Column({ name: 'telnyx_number_id', length: 100, unique: true, nullable: true })
  telnyxNumberId?: string;

  @Column({ name: 'messaging_profile_id', length: 100, nullable: true })
  messagingProfileId?: string;

  @Column({
    type: 'enum',
    enum: PhoneNumberStatus,
    default: PhoneNumberStatus.AVAILABLE,
  })
  @Index()
  status: PhoneNumberStatus;

  @Column({ name: 'purchased_at', default: () => 'CURRENT_TIMESTAMP' })
  purchasedAt: Date;

  @Column({ name: 'assigned_at', nullable: true })
  assignedAt?: Date;

  @CreateDateColumn({ name: 'created_at' })
  createdAt: Date;

  @UpdateDateColumn({ name: 'updated_at' })
  updatedAt: Date;

  @ManyToOne(() => Campaign, (campaign) => campaign.phoneNumbers, { onDelete: 'SET NULL' })
  @JoinColumn({ name: 'campaign_id' })
  campaign?: Campaign;

  @OneToMany(() => Message, (message) => message.phoneNumber)
  messages: Message[];
}
```

**src/entities/OptOut.ts:**
```typescript
import {
  Entity,
  PrimaryGeneratedColumn,
  Column,
  CreateDateColumn,
  ManyToOne,
  JoinColumn,
  Index,
  Unique,
} from 'typeorm';
import { Campaign } from './Campaign';
import { Brand } from './Brand';
import { Message } from './Message';

export enum OptOutMethod {
  SMS_REPLY = 'sms_reply',
  MANUAL = 'manual',
  API = 'api',
}

@Entity('opt_outs')
@Unique(['phoneNumber', 'campaignId'])
export class OptOut {
  @PrimaryGeneratedColumn('uuid')
  id: string;

  @Column({ name: 'phone_number', length: 20 })
  @Index()
  phoneNumber: string;

  @Column({ name: 'campaign_id', nullable: true })
  @Index()
  campaignId?: string;

  @Column({ name: 'brand_id', nullable: true })
  brandId?: string;

  @Column({ name: 'opted_out_at', default: () => 'CURRENT_TIMESTAMP' })
  @Index()
  optedOutAt: Date;

  @Column({
    name: 'opt_out_method',
    type: 'enum',
    enum: OptOutMethod,
    default: OptOutMethod.SMS_REPLY,
  })
  optOutMethod: OptOutMethod;

  @Column({ name: 'message_id', nullable: true })
  messageId?: string;

  @CreateDateColumn({ name: 'created_at' })
  createdAt: Date;

  @ManyToOne(() => Campaign, (campaign) => campaign.optOuts, { onDelete: 'SET NULL' })
  @JoinColumn({ name: 'campaign_id' })
  campaign?: Campaign;

  @ManyToOne(() => Brand, (brand) => brand.optOuts, { onDelete: 'SET NULL' })
  @JoinColumn({ name: 'brand_id' })
  brand?: Brand;

  @ManyToOne(() => Message, { onDelete: 'SET NULL' })
  @JoinColumn({ name: 'message_id' })
  message?: Message;
}
```

**src/entities/User.ts:**
```typescript
import {
  Entity,
  PrimaryGeneratedColumn,
  Column,
  CreateDateColumn,
  UpdateDateColumn,
  OneToMany,
  Index,
} from 'typeorm';
import { UserBrand } from './UserBrand';
import { ApiKey } from './ApiKey';

export enum UserRole {
  ADMIN = 'admin',
  MANAGER = 'manager',
  USER = 'user',
}

@Entity('users')
export class User {
  @PrimaryGeneratedColumn('uuid')
  id: string;

  @Column({ length: 255, unique: true })
  @Index()
  email: string;

  @Column({ name: 'password_hash', length: 255 })
  passwordHash: string;

  @Column({ name: 'first_name', length: 100 })
  firstName: string;

  @Column({ name: 'last_name', length: 100 })
  lastName: string;

  @Column({
    type: 'enum',
    enum: UserRole,
    default: UserRole.USER,
  })
  @Index()
  role: UserRole;

  @Column({ name: 'is_active', default: true })
  @Index()
  isActive: boolean;

  @Column({ name: 'last_login', nullable: true })
  lastLogin?: Date;

  @CreateDateColumn({ name: 'created_at' })
  createdAt: Date;

  @UpdateDateColumn({ name: 'updated_at' })
  updatedAt: Date;

  @OneToMany(() => UserBrand, (userBrand) => userBrand.user)
  userBrands: UserBrand[];

  @OneToMany(() => ApiKey, (apiKey) => apiKey.user)
  apiKeys: ApiKey[];

  // Virtual property for full name
  get fullName(): string {
    return `${this.firstName} ${this.lastName}`;
  }
}
```

**src/entities/UserBrand.ts:**
```typescript
import {
  Entity,
  PrimaryGeneratedColumn,
  Column,
  CreateDateColumn,
  ManyToOne,
  JoinColumn,
  Index,
  Unique,
} from 'typeorm';
import { User } from './User';
import { Brand } from './Brand';

export enum UserBrandRole {
  OWNER = 'owner',
  ADMIN = 'admin',
  MEMBER = 'member',
}

@Entity('user_brands')
@Unique(['userId', 'brandId'])
export class UserBrand {
  @PrimaryGeneratedColumn('uuid')
  id: string;

  @Column({ name: 'user_id' })
  @Index()
  userId: string;

  @Column({ name: 'brand_id' })
  @Index()
  brandId: string;

  @Column({
    type: 'enum',
    enum: UserBrandRole,
    default: UserBrandRole.MEMBER,
  })
  role: UserBrandRole;

  @CreateDateColumn({ name: 'created_at' })
  createdAt: Date;

  @ManyToOne(() => User, (user) => user.userBrands, { onDelete: 'CASCADE' })
  @JoinColumn({ name: 'user_id' })
  user: User;

  @ManyToOne(() => Brand, (brand) => brand.userBrands, { onDelete: 'CASCADE' })
  @JoinColumn({ name: 'brand_id' })
  brand: Brand;
}
```

**src/entities/ApiKey.ts:**
```typescript
import {
  Entity,
  PrimaryGeneratedColumn,
  Column,
  CreateDateColumn,
  UpdateDateColumn,
  ManyToOne,
  JoinColumn,
  Index,
} from 'typeorm';
import { User } from './User';

@Entity('api_keys')
export class ApiKey {
  @PrimaryGeneratedColumn('uuid')
  id: string;

  @Column({ name: 'user_id' })
  @Index()
  userId: string;

  @Column({ name: 'key_name', length: 100 })
  keyName: string;

  @Column({ name: 'api_key', length: 255, unique: true })
  @Index()
  apiKey: string;

  @Column({ type: 'json', nullable: true })
  permissions?: any;

  @Column({ name: 'is_active', default: true })
  @Index()
  isActive: boolean;

  @Column({ name: 'last_used', nullable: true })
  lastUsed?: Date;

  @Column({ name: 'expires_at', nullable: true })
  expiresAt?: Date;

  @CreateDateColumn({ name: 'created_at' })
  createdAt: Date;

  @UpdateDateColumn({ name: 'updated_at' })
  updatedAt: Date;

  @ManyToOne(() => User, (user) => user.apiKeys, { onDelete: 'CASCADE' })
  @JoinColumn({ name: 'user_id' })
  user: User;
}
```

**src/entities/WebhookEvent.ts:**
```typescript
import {
  Entity,
  PrimaryGeneratedColumn,
  Column,
  CreateDateColumn,
  Index,
} from 'typeorm';

@Entity('webhook_events')
export class WebhookEvent {
  @PrimaryGeneratedColumn('uuid')
  id: string;

  @Column({ name: 'event_type', length: 100 })
  @Index()
  eventType: string;

  @Column({ name: 'resource_id', length: 100, nullable: true })
  resourceId?: string;

  @Column({ type: 'json' })
  payload: any;

  @Column({ default: false })
  @Index()
  processed: boolean;

  @Column({ name: 'processing_error', type: 'text', nullable: true })
  processingError?: string;

  @Column({ name: 'retry_count', default: 0 })
  retryCount: number;

  @CreateDateColumn({ name: 'created_at' })
  @Index()
  createdAt: Date;

  @Column({ name: 'processed_at', nullable: true })
  processedAt?: Date;
}
```

### Step 5: Service Layer Implementation

**src/services/telnyx.service.ts:**
```typescript
import axios, { AxiosInstance, AxiosResponse } from 'axios';
import { config } from '../config/app.config';
import {
  TelnyxBrand,
  TelnyxCampaign,
  TelnyxMessage,
  TelnyxWebhookEvent,
} from '../types/telnyx.types';
import { logger } from '../utils/logger.util';

export class TelnyxService {
  private client: AxiosInstance;

  constructor() {
    this.client = axios.create({
      baseURL: config.telnyx.baseUrl,
      headers: {
        'Authorization': `Bearer ${config.telnyx.apiKey}`,
        'Content-Type': 'application/json',
      },
      timeout: 30000,
    });

    // Add request/response interceptors for logging
    this.client.interceptors.request.use((request) => {
      logger.debug('Telnyx API Request', {
        method: request.method,
        url: request.url,
        data: request.data,
      });
      return request;
    });

    this.client.interceptors.response.use(
      (response) => {
        logger.debug('Telnyx API Response', {
          status: response.status,
          data: response.data,
        });
        return response;
      },
      (error) => {
        logger.error('Telnyx API Error', {
          status: error.response?.status,
          data: error.response?.data,
          message: error.message,
        });
        throw error;
      }
    );
  }

  // Brand Management
  async createBrand(brandData: Partial<TelnyxBrand>): Promise<TelnyxBrand> {
    try {
      const response: AxiosResponse<{ data: TelnyxBrand }> = await this.client.post(
        '/10dlc/brands',
        brandData
      );
      return response.data.data;
    } catch (error) {
      logger.error('Failed to create brand in Telnyx', error);
      throw new Error('Failed to create brand with Telnyx');
    }
  }

  async getBrand(brandId: string): Promise<TelnyxBrand> {
    try {
      const response: AxiosResponse<{ data: TelnyxBrand }> = await this.client.get(
        `/10dlc/brands/${brandId}`
      );
      return response.data.data;
    } catch (error) {
      logger.error('Failed to get brand from Telnyx', error);
      throw new Error('Failed to get brand from Telnyx');
    }
  }

  async updateBrand(brandId: string, brandData: Partial<TelnyxBrand>): Promise<TelnyxBrand> {
    try {
      const response: AxiosResponse<{ data: TelnyxBrand }> = await this.client.patch(
        `/10dlc/brands/${brandId}`,
        brandData
      );
      return response.data.data;
    } catch (error) {
      logger.error('Failed to update brand in Telnyx', error);
      throw new Error('Failed to update brand with Telnyx');
    }
  }

  // Campaign Management
  async createCampaign(campaignData: Partial<TelnyxCampaign>): Promise<TelnyxCampaign> {
    try {
      const response: AxiosResponse<{ data: TelnyxCampaign }> = await this.client.post(
        '/10dlc/campaigns',
        campaignData
      );
      return response.data.data;
    } catch (error) {
      logger.error('Failed to create campaign in Telnyx', error);
      throw new Error('Failed to create campaign with Telnyx');
    }
  }

  async getCampaign(campaignId: string): Promise<TelnyxCampaign> {
    try {
      const response: AxiosResponse<{ data: TelnyxCampaign }> = await this.client.get(
        `/10dlc/campaigns/${campaignId}`
      );
      return response.data.data;
    } catch (error) {
      logger.error('Failed to get campaign from Telnyx', error);
      throw new Error('Failed to get campaign from Telnyx');
    }
  }

  // Message Sending
  async sendMessage(messageData: {
    from: string;
    to: string;
    text: string;
    mediaUrls?: string[];
    webhookUrl?: string;
  }): Promise<TelnyxMessage> {
    try {
      const response: AxiosResponse<{ data: TelnyxMessage }> = await this.client.post(
        '/messages',
        {
          from: messageData.from,
          to: messageData.to,
          text: messageData.text,
          media_urls: messageData.mediaUrls,
          webhook_url: messageData.webhookUrl,
          use_profile_webhooks: true,
        }
      );
      return response.data.data;
    } catch (error) {
      logger.error('Failed to send message via Telnyx', error);
      throw new Error('Failed to send message via Telnyx');
    }
  }

  // Number Management
  async purchaseNumber(numberData: {
    phoneNumber?: string;
    areaCode?: string;
    city?: string;
    state?: string;
    messagingProfileId?: string;
  }): Promise<any> {
    try {
      // First, search for available numbers
      const searchResponse = await this.client.get('/available_phone_numbers', {
        params: {
          filter: {
            phone_number: numberData.phoneNumber,
            area_code: numberData.areaCode,
            administrative_area: numberData.state,
            locality: numberData.city,
            features: 'sms',
          },
          limit: 1,
        },
      });

      if (!searchResponse.data.data || searchResponse.data.data.length === 0) {
        throw new Error('No available numbers found');
      }

      const availableNumber = searchResponse.data.data[0];

      // Purchase the number
      const purchaseResponse = await this.client.post('/number_orders', {
        phone_numbers: [{ phone_number: availableNumber.phone_number }],
        messaging_profile_id: numberData.messagingProfileId,
      });

      return purchaseResponse.data.data;
    } catch (error) {
      logger.error('Failed to purchase number via Telnyx', error);
      throw new Error('Failed to purchase number via Telnyx');
    }
  }

  // Webhook validation
  validateWebhookSignature(payload: string, signature: string): boolean {
    try {
      const crypto = require('crypto');
      const expectedSignature = crypto
        .createHmac('sha256', config.telnyx.webhookSecret)
        .update(payload)
        .digest('hex');

      return crypto.timingSafeEqual(
        Buffer.from(signature, 'hex'),
        Buffer.from(expectedSignature, 'hex')
      );
    } catch (error) {
      logger.error('Failed to validate webhook signature', error);
      return false;
    }
  }
}

export const telnyxService = new TelnyxService();
```

**src/services/message.service.ts:**
```typescript
import { Repository } from 'typeorm';
import { AppDataSource } from '../config/database.config';
import { Message, MessageDirection, MessageStatus } from '../entities/Message';
import { Campaign } from '../entities/Campaign';
import { PhoneNumber } from '../entities/PhoneNumber';
import { OptOut } from '../entities/OptOut';
import { telnyxService } from './telnyx.service';
import { logger } from '../utils/logger.util';
import { SendMessageRequest, MessageResponse, OptOutCheck } from '../types/api.types';

export class MessageService {
  private messageRepository: Repository<Message>;
  private campaignRepository: Repository<Campaign>;
  private phoneNumberRepository: Repository<PhoneNumber>;
  private optOutRepository: Repository<OptOut>;

  constructor() {
    this.messageRepository = AppDataSource.getRepository(Message);
    this.campaignRepository = AppDataSource.getRepository(Campaign);
    this.phoneNumberRepository = AppDataSource.getRepository(PhoneNumber);
    this.optOutRepository = AppDataSource.getRepository(OptOut);
  }

  async sendMessage(messageData: SendMessageRequest): Promise<MessageResponse> {
    try {
      // Validate campaign exists and is approved
      const campaign = await this.campaignRepository.findOne({
        where: { id: messageData.campaignId },
        relations: ['brand'],
      });

      if (!campaign) {
        throw new Error('Campaign not found');
      }

      if (campaign.status !== 'approved') {
        throw new Error('Campaign is not approved for messaging');
      }

      // Check if recipient has opted out
      const optOutCheck = await this.checkOptOut(messageData.to, messageData.campaignId);
      if (optOutCheck.isOptedOut) {
        throw new Error('Recipient has opted out of messages from this campaign');
      }

      // Get phone number details
      const phoneNumber = await this.phoneNumberRepository.findOne({
        where: { phoneNumber: messageData.from },
      });

      if (!phoneNumber) {
        throw new Error('Phone number not found or not assigned to campaign');
      }

      // Create message record
      const message = this.messageRepository.create({
        campaignId: messageData.campaignId,
        phoneNumberId: phoneNumber.id,
        fromNumber: messageData.from,
        toNumber: messageData.to,
        messageText: messageData.text,
        direction: MessageDirection.OUTBOUND,
        status: MessageStatus.QUEUED,
        mediaUrls: messageData.mediaUrls,
      });

      const savedMessage = await this.messageRepository.save(message);

      // Send via Telnyx
      try {
        const telnyxResponse = await telnyxService.sendMessage({
          from: messageData.from,
          to: messageData.to,
          text: messageData.text,
          mediaUrls: messageData.mediaUrls,
          webhookUrl: `${process.env.WEBHOOK_BASE_URL}/api/webhooks/telnyx`,
        });

        // Update message with Telnyx details
        savedMessage.telnyxMessageId = telnyxResponse.id;
        savedMessage.status = MessageStatus.SENT;
        savedMessage.cost = parseFloat(telnyxResponse.cost?.amount || '0');
        savedMessage.segments = telnyxResponse.parts;
        savedMessage.sentAt = new Date();

        await this.messageRepository.save(savedMessage);

        return {
          id: savedMessage.id,
          telnyxMessageId: telnyxResponse.id,
          status: 'sent',
          cost: savedMessage.cost,
          segments: savedMessage.segments,
          sentAt: savedMessage.sentAt,
        };
      } catch (telnyxError) {
        // Update message with error
        savedMessage.status = MessageStatus.FAILED;
        savedMessage.errorMessage = (telnyxError as Error).message;
        await this.messageRepository.save(savedMessage);

        throw new Error(`Failed to send message: ${(telnyxError as Error).message}`);
      }
    } catch (error) {
      logger.error('Failed to send message', error);
      throw error;
    }
  }

  async checkOptOut(phoneNumber: string, campaignId?: string): Promise<OptOutCheck> {
    try {
      const optOut = await this.optOutRepository.findOne({
        where: {
          phoneNumber,
          ...(campaignId && { campaignId }),
        },
        order: { optedOutAt: 'DESC' },
      });

      return {
        phoneNumber,
        isOptedOut: !!optOut,
        optedOutAt: optOut?.optedOutAt,
        campaignId: optOut?.campaignId,
      };
    } catch (error) {
      logger.error('Failed to check opt-out status', error);
      throw new Error('Failed to check opt-out status');
    }
  }

  async processOptOut(phoneNumber: string, campaignId: string, messageId?: string): Promise<void> {
    try {
      // Check if already opted out
      const existingOptOut = await this.optOutRepository.findOne({
        where: { phoneNumber, campaignId },
      });

      if (!existingOptOut) {
        const optOut = this.optOutRepository.create({
          phoneNumber,
          campaignId,
          messageId,
          optOutMethod: 'sms_reply',
          optedOutAt: new Date(),
        });

        await this.optOutRepository.save(optOut);
        logger.info(`Processed opt-out for ${phoneNumber} from campaign ${campaignId}`);
      }
    } catch (error) {
      logger.error('Failed to process opt-out', error);
      throw error;
    }
  }

  async handleInboundMessage(messageData: any): Promise<void> {
    try {
      // Find phone number
      const phoneNumber = await this.phoneNumberRepository.findOne({
        where: { phoneNumber: messageData.to },
        relations: ['campaign'],
      });

      if (!phoneNumber) {
        logger.warn(`Received message for unknown number: ${messageData.to}`);
        return;
      }

      // Create inbound message record
      const message = this.messageRepository.create({
        telnyxMessageId: messageData.id,
        campaignId: phoneNumber.campaignId,
        phoneNumberId: phoneNumber.id,
        fromNumber: messageData.from,
        toNumber: messageData.to,
        messageText: messageData.text || '',
        direction: MessageDirection.INBOUND,
        status: MessageStatus.RECEIVED,
        messageType: messageData.type,
        mediaUrls: messageData.media?.map((m: any) => m.url),
        webhookData: messageData,
      });

      const savedMessage = await this.messageRepository.save(message);

      // Check for opt-out keywords
      const optOutKeywords = ['STOP', 'UNSUBSCRIBE', 'CANCEL', 'END', 'QUIT'];
      const messageText = messageData.text?.toUpperCase() || '';

      if (optOutKeywords.some(keyword => messageText.includes(keyword))) {
        await this.processOptOut(messageData.from, phoneNumber.campaignId, savedMessage.id);

        // Send confirmation message (optional)
        await this.sendOptOutConfirmation(messageData.from, messageData.to);
      }
    } catch (error) {
      logger.error('Failed to handle inbound message', error);
    }
  }

  private async sendOptOutConfirmation(to: string, from: string): Promise<void> {
    try {
      await telnyxService.sendMessage({
        from,
        to,
        text: 'You have been successfully unsubscribed. You will no longer receive messages from us.',
      });
    } catch (error) {
      logger.error('Failed to send opt-out confirmation', error);
    }
  }

  async getMessageHistory(
    campaignId: string,
    page: number = 1,
    limit: number = 50
  ): Promise<{
    messages: Message[];
    total: number;
    totalPages: number;
    currentPage: number;
  }> {
    try {
      const [messages, total] = await this.messageRepository.findAndCount({
        where: { campaignId },
        relations: ['phoneNumber', 'campaign'],
        order: { createdAt: 'DESC' },
        skip: (page - 1) * limit,
        take: limit,
      });

      return {
        messages,
        total,
        totalPages: Math.ceil(total / limit),
        currentPage: page,
      };
    } catch (error) {
      logger.error('Failed to get message history', error);
      throw new Error('Failed to get message history');
    }
  }
}

export const messageService = new MessageService();
```

**src/services/brand.service.ts:**
```typescript
import { Repository } from 'typeorm';
import { AppDataSource } from '../config/database.config';
import { Brand, BrandStatus } from '../entities/Brand';
import { User } from '../entities/User';
import { UserBrand, UserBrandRole } from '../entities/UserBrand';
import { telnyxService } from './telnyx.service';
import { logger } from '../utils/logger.util';
import { CreateBrandRequest } from '../types/api.types';

export class BrandService {
  private brandRepository: Repository<Brand>;
  private userBrandRepository: Repository<UserBrand>;

  constructor() {
    this.brandRepository = AppDataSource.getRepository(Brand);
    this.userBrandRepository = AppDataSource.getRepository(UserBrand);
  }

  async createBrand(brandData: CreateBrandRequest, userId: string): Promise<Brand> {
    try {
      // Create brand record
      const brand = this.brandRepository.create({
        legalBusinessName: brandData.legalBusinessName,
        businessType: brandData.businessType,
        einTaxId: brandData.einTaxId,
        dunsNumber: brandData.dunsNumber,
        businessAddress: brandData.businessAddress,
        website: brandData.website,
        contactPhone: brandData.contactPhone,
        contactEmail: brandData.contactEmail,
        status: BrandStatus.PENDING,
      });

      const savedBrand = await this.brandRepository.save(brand);

      // Create user-brand association
      const userBrand = this.userBrandRepository.create({
        userId,
        brandId: savedBrand.id,
        role: UserBrandRole.OWNER,
      });

      await this.userBrandRepository.save(userBrand);

      // Register with Telnyx
      try {
        const telnyxBrand = await telnyxService.createBrand({
          displayName: brandData.legalBusinessName,
          companyName: brandData.legalBusinessName,
          entityType: brandData.businessType,
          ein: brandData.einTaxId,
          website: brandData.website,
          phone: brandData.contactPhone,
          email: brandData.contactEmail,
          businessContactEmail: brandData.contactEmail,
          businessContactPhone: brandData.contactPhone,
        });

        // Update brand with Telnyx ID
        savedBrand.telnyxBrandId = telnyxBrand.id;
        await this.brandRepository.save(savedBrand);

        logger.info(`Brand created successfully: ${savedBrand.id}`);
      } catch (telnyxError) {
        logger.error('Failed to register brand with Telnyx', telnyxError);
        // Don't fail the entire operation, just log the error
      }

      return savedBrand;
    } catch (error) {
      logger.error('Failed to create brand', error);
      throw new Error('Failed to create brand');
    }
  }

  async getBrandsByUser(userId: string): Promise<Brand[]> {
    try {
      const userBrands = await this.userBrandRepository.find({
        where: { userId },
        relations: ['brand'],
      });

      return userBrands.map(ub => ub.brand);
    } catch (error) {
      logger.error('Failed to get brands for user', error);
      throw new Error('Failed to get brands');
    }
  }

  async getBrandById(brandId: string, userId: string): Promise<Brand> {
    try {
      // Check if user has access to this brand
      const userBrand = await this.userBrandRepository.findOne({
        where: { userId, brandId },
        relations: ['brand', 'brand.campaigns'],
      });

      if (!userBrand) {
        throw new Error('Brand not found or access denied');
      }

      return userBrand.brand;
    } catch (error) {
      logger.error('Failed to get brand', error);
      throw error;
    }
  }

  async updateBrand(brandId: string, brandData: Partial<CreateBrandRequest>, userId: string): Promise<Brand> {
    try {
      // Check if user has admin access to this brand
      const userBrand = await this.userBrandRepository.findOne({
        where: { userId, brandId, role: UserBrandRole.ADMIN || UserBrandRole.OWNER },
      });

      if (!userBrand) {
        throw new Error('Insufficient permissions to update brand');
      }

      const brand = await this.brandRepository.findOne({
        where: { id: brandId },
      });

      if (!brand) {
        throw new Error('Brand not found');
      }

      // Update brand data
      Object.assign(brand, brandData);
      const updatedBrand = await this.brandRepository.save(brand);

      // Update in Telnyx if registered
      if (brand.telnyxBrandId) {
        try {
          await telnyxService.updateBrand(brand.telnyxBrandId, {
            displayName: brandData.legalBusinessName,
            companyName: brandData.legalBusinessName,
            website: brandData.website,
            phone: brandData.contactPhone,
            email: brandData.contactEmail,
          });
        } catch (telnyxError) {
          logger.error('Failed to update brand in Telnyx', telnyxError);
        }
      }

      return updatedBrand;
    } catch (error) {
      logger.error('Failed to update brand', error);
      throw error;
    }
  }

  async deleteBrand(brandId: string, userId: string): Promise<void> {
    try {
      // Check if user is owner of this brand
      const userBrand = await this.userBrandRepository.findOne({
        where: { userId, brandId, role: UserBrandRole.OWNER },
      });

      if (!userBrand) {
        throw new Error('Insufficient permissions to delete brand');
      }

      await this.brandRepository.delete(brandId);
      logger.info(`Brand deleted: ${brandId}`);
    } catch (error) {
      logger.error('Failed to delete brand', error);
      throw error;
    }
  }

  async syncBrandStatus(brandId: string): Promise<void> {
    try {
      const brand = await this.brandRepository.findOne({
        where: { id: brandId },
      });

      if (!brand || !brand.telnyxBrandId) {
        return;
      }

      const telnyxBrand = await telnyxService.getBrand(brand.telnyxBrandId);
      
      // Update trust score and status based on Telnyx response
      brand.trustScore = telnyxBrand.trustScore || 0;
      // Map Telnyx status to our status enum
      
      await this.brandRepository.save(brand);
    } catch (error) {
      logger.error('Failed to sync brand status', error);
    }
  }
}

export const brandService = new BrandService();
```

**src/services/campaign.service.ts:**
```typescript
import { Repository } from 'typeorm';
import { AppDataSource } from '../config/database.config';
import { Campaign, CampaignStatus } from '../entities/Campaign';
import { Brand } from '../entities/Brand';
import { UserBrand } from '../entities/UserBrand';
import { telnyxService } from './telnyx.service';
import { logger } from '../utils/logger.util';
import { CreateCampaignRequest } from '../types/api.types';

export class CampaignService {
  private campaignRepository: Repository<Campaign>;
  private brandRepository: Repository<Brand>;
  private userBrandRepository: Repository<UserBrand>;

  constructor() {
    this.campaignRepository = AppDataSource.getRepository(Campaign);
    this.brandRepository = AppDataSource.getRepository(Brand);
    this.userBrandRepository = AppDataSource.getRepository(UserBrand);
  }

  async createCampaign(campaignData: CreateCampaignRequest, userId: string): Promise<Campaign> {
    try {
      // Verify user has access to the brand
      const userBrand = await this.userBrandRepository.findOne({
        where: { userId, brandId: campaignData.brandId },
        relations: ['brand'],
      });

      if (!userBrand) {
        throw new Error('Brand not found or access denied');
      }

      const brand = userBrand.brand;
      if (brand.status !== 'approved') {
        throw new Error('Brand must be approved before creating campaigns');
      }

      // Create campaign record
      const campaign = this.campaignRepository.create({
        brandId: campaignData.brandId,
        name: campaignData.name,
        useCase: campaignData.useCase,
        description: campaignData.description,
        sampleMessages: campaignData.sampleMessages,
        optInProcess: campaignData.optInProcess,
        optOutInstructions: campaignData.optOutInstructions || 'Reply STOP to unsubscribe',
        status: CampaignStatus.PENDING,
      });

      const savedCampaign = await this.campaignRepository.save(campaign);

      // Register with Telnyx
      try {
        const telnyxCampaign = await telnyxService.createCampaign({
          brandId: brand.telnyxBrandId,
          usecase: campaignData.useCase,
          description: campaignData.description || campaignData.name,
          sample1: campaignData.sampleMessages[0],
          sample2: campaignData.sampleMessages[1],
          sample3: campaignData.sampleMessages[2],
          sample4: campaignData.sampleMessages[3],
          sample5: campaignData.sampleMessages[4],
          optinMessage: campaignData.optInProcess,
          optoutMessage: campaignData.optOutInstructions,
          optoutKeywords: 'STOP,UNSUBSCRIBE,CANCEL,END,QUIT',
          helpKeywords: 'HELP,INFO',
          helpMessage: 'For help, reply HELP. To opt out, reply STOP.',
        });

        // Update campaign with Telnyx ID
        savedCampaign.telnyxCampaignId = telnyxCampaign.id;
        await this.campaignRepository.save(savedCampaign);

        logger.info(`Campaign created successfully: ${savedCampaign.id}`);
      } catch (telnyxError) {
        logger.error('Failed to register campaign with Telnyx', telnyxError);
        // Don't fail the entire operation
      }

      return savedCampaign;
    } catch (error) {
      logger.error('Failed to create campaign', error);
      throw error;
    }
  }

  async getCampaignsByUser(userId: string): Promise<Campaign[]> {
    try {
      const userBrands = await this.userBrandRepository.find({
        where: { userId },
        select: ['brandId'],
      });

      const brandIds = userBrands.map(ub => ub.brandId);

      const campaigns = await this.campaignRepository.find({
        where: { brandId: In(brandIds) },
        relations: ['brand'],
        order: { createdAt: 'DESC' },
      });

      return campaigns;
    } catch (error) {
      logger.error('Failed to get campaigns for user', error);
      throw new Error('Failed to get campaigns');
    }
  }

  async getCampaignById(campaignId: string, userId: string): Promise<Campaign> {
    try {
      const campaign = await this.campaignRepository.findOne({
        where: { id: campaignId },
        relations: ['brand', 'phoneNumbers', 'messages'],
      });

      if (!campaign) {
        throw new Error('Campaign not found');
      }

      // Check if user has access to this campaign's brand
      const userBrand = await this.userBrandRepository.findOne({
        where: { userId, brandId: campaign.brandId },
      });

      if (!userBrand) {
        throw new Error('Access denied');
      }

      return campaign;
    } catch (error) {
      logger.error('Failed to get campaign', error);
      throw error;
    }
  }

  async updateCampaign(
    campaignId: string,
    campaignData: Partial<CreateCampaignRequest>,
    userId: string
  ): Promise<Campaign> {
    try {
      const campaign = await this.campaignRepository.findOne({
        where: { id: campaignId },
        relations: ['brand'],
      });

      if (!campaign) {
        throw new Error('Campaign not found');
      }

      // Check if user has access to this campaign's brand
      const userBrand = await this.userBrandRepository.findOne({
        where: { userId, brandId: campaign.brandId },
      });

      if (!userBrand) {
        throw new Error('Access denied');
      }

      // Update campaign data
      Object.assign(campaign, campaignData);
      const updatedCampaign = await this.campaignRepository.save(campaign);

      return updatedCampaign;
    } catch (error) {
      logger.error('Failed to update campaign', error);
      throw error;
    }
  }

  async deleteCampaign(campaignId: string, userId: string): Promise<void> {
    try {
      const campaign = await this.campaignRepository.findOne({
        where: { id: campaignId },
      });

      if (!campaign) {
        throw new Error('Campaign not found');
      }

      // Check if user has access to this campaign's brand
      const userBrand = await this.userBrandRepository.findOne({
        where: { userId, brandId: campaign.brandId },
      });

      if (!userBrand) {
        throw new Error('Access denied');
      }

      await this.campaignRepository.delete(campaignId);
      logger.info(`Campaign deleted: ${campaignId}`);
    } catch (error) {
      logger.error('Failed to delete campaign', error);
      throw error;
    }
  }

  async syncCampaignStatus(campaignId: string): Promise<void> {
    try {
      const campaign = await this.campaignRepository.findOne({
        where: { id: campaignId },
      });

      if (!campaign || !campaign.telnyxCampaignId) {
        return;
      }

      const telnyxCampaign = await telnyxService.getCampaign(campaign.telnyxCampaignId);
      
      // Update status and throughput based on Telnyx response
      // Map Telnyx status to our status enum
      
      await this.campaignRepository.save(campaign);
    } catch (error) {
      logger.error('Failed to sync campaign status', error);
    }
  }
}

export const campaignService = new CampaignService();
```

**src/services/webhook.service.ts:**
```typescript
import { Repository } from 'typeorm';
import { AppDataSource } from '../config/database.config';
import { WebhookEvent } from '../entities/WebhookEvent';
import { Message, MessageStatus } from '../entities/Message';
import { messageService } from './message.service';
import { logger } from '../utils/logger.util';
import { TelnyxWebhookEvent } from '../types/telnyx.types';

export class WebhookService {
  private webhookEventRepository: Repository<WebhookEvent>;
  private messageRepository: Repository<Message>;

  constructor() {
    this.webhookEventRepository = AppDataSource.getRepository(WebhookEvent);
    this.messageRepository = AppDataSource.getRepository(Message);
  }

  async processWebhook(payload: TelnyxWebhookEvent): Promise<void> {
    try {
      // Store webhook event
      const webhookEvent = this.webhookEventRepository.create({
        eventType: payload.data.eventType,
        resourceId: payload.data.payload.id,
        payload: payload.data.payload,
        processed: false,
        retryCount: 0,
      });

      const savedWebhookEvent = await this.webhookEventRepository.save(webhookEvent);

      try {
        await this.handleWebhookEvent(payload);
        
        // Mark as processed
        savedWebhookEvent.processed = true;
        savedWebhookEvent.processedAt = new Date();
        await this.webhookEventRepository.save(savedWebhookEvent);
        
        logger.info(`Webhook event processed: ${payload.data.eventType}`);
      } catch (processingError) {
        // Mark processing error
        savedWebhookEvent.processingError = (processingError as Error).message;
        savedWebhookEvent.retryCount += 1;
        await this.webhookEventRepository.save(savedWebhookEvent);
        
        logger.error('Failed to process webhook event', processingError);
        throw processingError;
      }
    } catch (error) {
      logger.error('Failed to process webhook', error);
      throw error;
    }
  }

  private async handleWebhookEvent(payload: TelnyxWebhookEvent): Promise<void> {
    const { eventType, payload: eventPayload } = payload.data;

    switch (eventType) {
      case 'message.sent':
        await this.handleMessageSent(eventPayload);
        break;
      case 'message.delivered':
        await this.handleMessageDelivered(eventPayload);
        break;
      case 'message.delivery_failed':
        await this.handleMessageFailed(eventPayload);
        break;
      case 'message.received':
        await this.handleMessageReceived(eventPayload);
        break;
      default:
        logger.warn(`Unhandled webhook event type: ${eventType}`);
    }
  }

  private async handleMessageSent(messageData: any): Promise<void> {
    try {
      const message = await this.messageRepository.findOne({
        where: { telnyxMessageId: messageData.id },
      });

      if (message) {
        message.status = MessageStatus.SENT;
        message.sentAt = new Date(messageData.sentAt);
        message.cost = parseFloat(messageData.cost?.amount || '0');
        message.segments = messageData.parts || 1;
        
        await this.messageRepository.save(message);
      }
    } catch (error) {
      logger.error('Failed to handle message sent event', error);
      throw error;
    }
  }

  private async handleMessageDelivered(messageData: any): Promise<void> {
    try {
      const message = await this.messageRepository.findOne({
        where: { telnyxMessageId: messageData.id },
      });

      if (message) {
        message.status = MessageStatus.DELIVERED;
        message.deliveredAt = new Date(messageData.completedAt);
        
        await this.messageRepository.save(message);
      }
    } catch (error) {
      logger.error('Failed to handle message delivered event', error);
      throw error;
    }
  }

  private async handleMessageFailed(messageData: any): Promise<void> {
    try {
      const message = await this.messageRepository.findOne({
        where: { telnyxMessageId: messageData.id },
      });

      if (message) {
        message.status = MessageStatus.FAILED;
        message.errorCode = messageData.errors?.[0]?.code?.toString();
        message.errorMessage = messageData.errors?.[0]?.detail;
        
        await this.messageRepository.save(message);
      }
    } catch (error) {
      logger.error('Failed to handle message failed event', error);
      throw error;
    }
  }

  private async handleMessageReceived(messageData: any): Promise<void> {
    try {
      // Use message service to handle inbound message
      await messageService.handleInboundMessage(messageData);
    } catch (error) {
      logger.error('Failed to handle message received event', error);
      throw error;
    }
  }

  async retryFailedWebhooks(maxRetries: number = 3): Promise<void> {
    try {
      const failedEvents = await this.webhookEventRepository.find({
        where: {
          processed: false,
          retryCount: LessThan(maxRetries),
        },
        order: { createdAt: 'ASC' },
        take: 100,
      });

      for (const event of failedEvents) {
        try {
          const payload = {
            data: {
              eventType: event.eventType,
              payload: event.payload,
            },
          } as TelnyxWebhookEvent;

          await this.handleWebhookEvent(payload);
          
          event.processed = true;
          event.processedAt = new Date();
          await this.webhookEventRepository.save(event);
          
          logger.info(`Retried webhook event successfully: ${event.id}`);
        } catch (retryError) {
          event.retryCount += 1;
          event.processingError = (retryError as Error).message;
          await this.webhookEventRepository.save(event);
          
          logger.error(`Failed to retry webhook event: ${event.id}`, retryError);
        }
      }
    } catch (error) {
      logger.error('Failed to retry webhook events', error);
    }
  }

  async getWebhookStats(): Promise<{
    total: number;
    processed: number;
    failed: number;
    pending: number;
  }> {
    try {
      const total = await this.webhookEventRepository.count();
      const processed = await this.webhookEventRepository.count({
        where: { processed: true },
      });
      const failed = await this.webhookEventRepository.count({
        where: { processed: false, retryCount: MoreThan(0) },
      });
      const pending = await this.webhookEventRepository.count({
        where: { processed: false, retryCount: 0 },
      });

      return { total, processed, failed, pending };
    } catch (error) {
      logger.error('Failed to get webhook stats', error);
      throw error;
    }
  }
}

export const webhookService = new WebhookService();
```

**src/services/auth.service.ts:**
```typescript
import { Repository } from 'typeorm';
import bcrypt from 'bcryptjs';
import jwt from 'jsonwebtoken';
import { AppDataSource } from '../config/database.config';
import { User, UserRole } from '../entities/User';
import { ApiKey } from '../entities/ApiKey';
import { config } from '../config/app.config';
import { logger } from '../utils/logger.util';
import { generateApiKey } from '../utils/crypto.util';

export interface AuthTokens {
  accessToken: string;
  refreshToken: string;
}

export interface LoginRequest {
  email: string;
  password: string;
}

export interface RegisterRequest {
  email: string;
  password: string;
  firstName: string;
  lastName: string;
}

export class AuthService {
  private userRepository: Repository<User>;
  private apiKeyRepository: Repository<ApiKey>;

  constructor() {
    this.userRepository = AppDataSource.getRepository(User);
    this.apiKeyRepository = AppDataSource.getRepository(ApiKey);
  }

  async register(userData: RegisterRequest): Promise<{ user: User; tokens: AuthTokens }> {
    try {
      // Check if user already exists
      const existingUser = await this.userRepository.findOne({
        where: { email: userData.email },
      });

      if (existingUser) {
        throw new Error('User already exists with this email');
      }

      // Hash password
      const passwordHash = await bcrypt.hash(userData.password, 12);

      // Create user
      const user = this.userRepository.create({
        email: userData.email,
        passwordHash,
        firstName: userData.firstName,
        lastName: userData.lastName,
        role: UserRole.USER,
        isActive: true,
      });

      const savedUser = await this.userRepository.save(user);

      // Generate tokens
      const tokens = this.generateTokens(savedUser);

      // Update last login
      savedUser.lastLogin = new Date();
      await this.userRepository.save(savedUser);

      logger.info(`User registered: ${savedUser.email}`);

      return { user: savedUser, tokens };
    } catch (error) {
      logger.error('Failed to register user', error);
      throw error;
    }
  }

  async login(loginData: LoginRequest): Promise<{ user: User; tokens: AuthTokens }> {
    try {
      // Find user
      const user = await this.userRepository.findOne({
        where: { email: loginData.email },
      });

      if (!user || !user.isActive) {
        throw new Error('Invalid credentials');
      }

      // Verify password
      const isPasswordValid = await bcrypt.compare(loginData.password, user.passwordHash);
      if (!isPasswordValid) {
        throw new Error('Invalid credentials');
      }

      // Generate tokens
      const tokens = this.generateTokens(user);

      // Update last login
      user.lastLogin = new Date();
      await this.userRepository.save(user);

      logger.info(`User logged in: ${user.email}`);

      return { user, tokens };
    } catch (error) {
      logger.error('Failed to login user', error);
      throw error;
    }
  }

  async verifyToken(token: string): Promise<User> {
    try {
      const decoded = jwt.verify(token, config.app.jwtSecret) as any;
      
      const user = await this.userRepository.findOne({
        where: { id: decoded.userId, isActive: true },
      });

      if (!user) {
        throw new Error('Invalid token');
      }

      return user;
    } catch (error) {
      throw new Error('Invalid token');
    }
  }

  async createApiKey(userId: string, keyName: string, permissions?: any): Promise<ApiKey> {
    try {
      const user = await this.userRepository.findOne({
        where: { id: userId },
      });

      if (!user) {
        throw new Error('User not found');
      }

      const apiKeyValue = generateApiKey();

      const apiKey = this.apiKeyRepository.create({
        userId,
        keyName,
        apiKey: apiKeyValue,
        permissions,
        isActive: true,
      });

      const savedApiKey = await this.apiKeyRepository.save(apiKey);
      
      logger.info(`API key created for user: ${user.email}`);

      return savedApiKey;
    } catch (error) {
      logger.error('Failed to create API key', error);
      throw error;
    }
  }

  async verifyApiKey(apiKey: string): Promise<User> {
    try {
      const keyRecord = await this.apiKeyRepository.findOne({
        where: { apiKey, isActive: true },
        relations: ['user'],
      });

      if (!keyRecord || !keyRecord.user.isActive) {
        throw new Error('Invalid API key');
      }

      // Check expiration
      if (keyRecord.expiresAt && keyRecord.expiresAt < new Date()) {
        throw new Error('API key expired');
      }

      // Update last used
      keyRecord.lastUsed = new Date();
      await this.apiKeyRepository.save(keyRecord);

      return keyRecord.user;
    } catch (error) {
      throw new Error('Invalid API key');
    }
  }

  private generateTokens(user: User): AuthTokens {
    const payload = {
      userId: user.id,
      email: user.email,
      role: user.role,
    };

    const accessToken = jwt.sign(payload, config.app.jwtSecret, {
      expiresIn: config.app.jwtExpiresIn,
    });

    const refreshToken = jwt.sign(payload, config.app.jwtSecret, {
      expiresIn: '30d',
    });

    return { accessToken, refreshToken };
  }

  async changePassword(userId: string, currentPassword: string, newPassword: string): Promise<void> {
    try {
      const user = await this.userRepository.findOne({
        where: { id: userId },
      });

      if (!user) {
        throw new Error('User not found');
      }

      // Verify current password
      const isPasswordValid = await bcrypt.compare(currentPassword, user.passwordHash);
      if (!isPasswordValid) {
        throw new Error('Current password is incorrect');
      }

      // Hash new password
      const newPasswordHash = await bcrypt.hash(newPassword, 12);
      
      user.passwordHash = newPasswordHash;
      await this.userRepository.save(user);

      logger.info(`Password changed for user: ${user.email}`);
    } catch (error) {
      logger.error('Failed to change password', error);
      throw error;
    }
  }
}

export const authService = new AuthService();
```

### Step 6: Controllers and Routes

**src/controllers/message.controller.ts:**
```typescript
import { Request, Response } from 'express';
import { messageService } from '../services/message.service';
import { logger } from '../utils/logger.util';
import { SendMessageRequest } from '../types/api.types';

export class MessageController {
  async sendMessage(req: Request, res: Response): Promise<void> {
    try {
      const messageData: SendMessageRequest = req.body;
      const result = await messageService.sendMessage(messageData);

      res.status(200).json({
        success: true,
        data: result,
      });
    } catch (error) {
      logger.error('Failed to send message', error);
      res.status(400).json({
        success: false,
        error: (error as Error).message,
      });
    }
  }

  async getMessageHistory(req: Request, res: Response): Promise<void> {
    try {
      const { campaignId } = req.params;
      const page = parseInt(req.query.page as string) || 1;
      const limit = parseInt(req.query.limit as string) || 50;

      const result = await messageService.getMessageHistory(campaignId, page, limit);

      res.status(200).json({
        success: true,
        data: result,
      });
    } catch (error) {
      logger.error('Failed to get message history', error);
      res.status(500).json({
        success: false,
        error: (error as Error).message,
      });
    }
  }

  async checkOptOut(req: Request, res: Response): Promise<void> {
    try {
      const { phoneNumber } = req.params;
      const { campaignId } = req.query;

      const result = await messageService.checkOptOut(phoneNumber, campaignId as string);

      res.status(200).json({
        success: true,
        data: result,
      });
    } catch (error) {
      logger.error('Failed to check opt-out status', error);
      res.status(500).json({
        success: false,
        error: (error as Error).message,
      });
    }
  }
}

export const messageController = new MessageController();
```

**src/routes/message.routes.ts:**
```typescript
import { Router } from 'express';
import { messageController } from '../controllers/message.controller';
import { authMiddleware } from '../middleware/auth.middleware';
import { validateMessage } from '../middleware/validation.middleware';

const router = Router();

/**
 * @swagger
 * /api/messages/send:
 *   post:
 *     summary: Send SMS message
 *     tags: [Messages]
 *     security:
 *       - bearerAuth: []
 *     requestBody:
 *       required: true
 *       content:
 *         application/json:
 *           schema:
 *             $ref: '#/components/schemas/SendMessageRequest'
 *     responses:
 *       200:
 *         description: Message sent successfully
 *       400:
 *         description: Invalid request data or recipient opted out
 *       401:
 *         description: Unauthorized
 */
router.post('/send', authMiddleware, validateMessage, messageController.sendMessage);

/**
 * @swagger
 * /api/messages/history/{campaignId}:
 *   get:
 *     summary: Get message history for campaign
 *     tags: [Messages]
 *     security:
 *       - bearerAuth: []
 *     parameters:
 *       - in: path
 *         name: campaignId
 *         required: true
 *         schema:
 *           type: string
 *       - in: query
 *         name: page
 *         schema:
 *           type: integer
 *           default: 1
 *       - in: query
 *         name: limit
 *         schema:
 *           type: integer
 *           default: 50
 *     responses:
 *       200:
 *         description: Message history retrieved
 *       500:
 *         description: Server error
 */
router.get('/history/:campaignId', authMiddleware, messageController.getMessageHistory);

/**
 * @swagger
 * /api/messages/opt-out-check/{phoneNumber}:
 *   get:
 *     summary: Check opt-out status for phone number
 *     tags: [Messages]
 *     security:
 *       - bearerAuth: []
 *     parameters:
 *       - in: path
 *         name: phoneNumber
 *         required: true
 *         schema:
 *           type: string
 *       - in: query
 *         name: campaignId
 *         schema:
 *           type: string
 *     responses:
 *       200:
 *         description: Opt-out status retrieved
 *       500:
 *         description: Server error
 */
router.get('/opt-out-check/:phoneNumber', authMiddleware, messageController.checkOptOut);

export default router;
```

### Step 7: Main Application Setup

**src/app.ts:**
```typescript
import express from 'express';
import cors from 'cors';
import helmet from 'helmet';
import morgan from 'morgan';
import rateLimit from 'express-rate-limit';
import swaggerJsdoc from 'swagger-jsdoc';
import swaggerUi from 'swagger-ui-express';

import { config } from './config/app.config';
import { AppDataSource } from './config/database.config';
import { logger } from './utils/logger.util';
import { errorHandler } from './middleware/error.middleware';

// Route imports
import authRoutes from './routes/auth.routes';
import brandRoutes from './routes/brand.routes';
import campaignRoutes from './routes/campaign.routes';
import messageRoutes from './routes/message.routes';
import numberRoutes from './routes/number.routes';
import webhookRoutes from './routes/webhook.routes';

class App {
  public app: express.Application;

  constructor() {
    this.app = express();
    this.initializeDatabase();
    this.initializeMiddleware();
    this.initializeRoutes();
    this.initializeSwagger();
    this.initializeErrorHandling();
  }

  private async initializeDatabase(): Promise<void> {
    try {
      await AppDataSource.initialize();
      logger.info('Database connection established');
    } catch (error) {
      logger.error('Database connection failed', error);
      process.exit(1);
    }
  }

  private initializeMiddleware(): void {
    // Security middleware
    this.app.use(helmet());
    this.app.use(cors());

    // Rate limiting
    const limiter = rateLimit({
      windowMs: config.rateLimit.windowMs,
      max: config.rateLimit.maxRequests,
      message: 'Too many requests from this IP',
    });
    this.app.use('/api/', limiter);

    // Logging
    this.app.use(morgan('combined', {
      stream: { write: (message) => logger.info(message.trim()) }
    }));

    // Body parsing
    this.app.use(express.json({ limit: '10mb' }));
    this.app.use(express.urlencoded({ extended: true }));

    // Health check
    this.app.get('/health', (req, res) => {
      res.status(200).json({ status: 'OK', timestamp: new Date().toISOString() });
    });
  }

  private initializeRoutes(): void {
    this.app.use('/api/auth', authRoutes);
    this.app.use('/api/brands', brandRoutes);
    this.app.use('/api/campaigns', campaignRoutes);
    this.app.use('/api/messages', messageRoutes);
    this.app.use('/api/numbers', numberRoutes);
    this.app.use('/api/webhooks', webhookRoutes);
  }

  private initializeSwagger(): void {
    const options = {
      definition: {
        openapi: '3.0.0',
        info: {
          title: '10DLC SMS API',
          version: '1.0.0',
          description: 'API for managing 10DLC SMS campaigns and messaging',
        },
        servers: [
          {
            url: `http://localhost:${config.app.port}`,
            description: 'Development server',
          },
        ],
        components: {
          securitySchemes: {
            bearerAuth: {
              type: 'http',
              scheme: 'bearer',
              bearerFormat: 'JWT',
            },
          },
        },
      },
      apis: ['./src/routes/*.ts'],
    };

    const specs = swaggerJsdoc(options);
    this.app.use('/api-docs', swaggerUi.serve, swaggerUi.setup(specs));
  }

  private initializeErrorHandling(): void {
    this.app.use(errorHandler);
  }

  public listen(): void {
    this.app.listen(config.app.port, () => {
      logger.info(`Server is running on port ${config.app.port}`);
      logger.info(`API Documentation available at http://localhost:${config.app.port}/api-docs`);
    });
  }
}

const app = new App();
app.listen();

export default app;
```

## Security Considerations

### Authentication & Authorization
- JWT-based authentication with secure secret rotation
- Role-based access control (RBAC) for different user types
- API key authentication for external integrations
- Rate limiting to prevent abuse

### Data Protection
- Password hashing with bcrypt (minimum 12 rounds)
- Sensitive data encryption at rest
- TLS/SSL for data in transit
- Input validation and sanitization

### Compliance
- Automatic opt-out processing for TCPA compliance
- Message content filtering for SHAFT compliance
- Audit logs for all messaging activities
- Data retention policies

### API Security
- Request/response validation
- CORS configuration
- Helmet.js for security headers
- Webhook signature validation

## Testing Strategy

### Unit Tests
```typescript
// tests/unit/services/message.service.test.ts
import { MessageService } from '../../../src/services/message.service';
import { telnyxService } from '../../../src/services/telnyx.service';

jest.mock('../../../src/services/telnyx.service');

describe('MessageService', () => {
  let messageService: MessageService;

  beforeEach(() => {
    messageService = new MessageService();
  });

  describe('sendMessage', () => {
    it('should send message successfully', async () => {
      // Test implementation
    });

    it('should reject message to opted-out recipient', async () => {
      // Test implementation
    });

    it('should handle Telnyx API errors', async () => {
      // Test implementation
    });
  });
});
```

### Integration Tests
```typescript
// tests/integration/message.integration.test.ts
import request from 'supertest';
import app from '../../src/app';

describe('Message API Integration', () => {
  it('should send message via API', async () => {
    const response = await request(app)
      .post('/api/messages/send')
      .set('Authorization', `Bearer ${testToken}`)
      .send({
        campaignId: 'test-campaign-id',
        from: '+12125551234',
        to: '+13125551234',
        text: 'Test message',
      });

    expect(response.status).toBe(200);
    expect(response.body.success).toBe(true);
  });
});
```

## Deployment

### Docker Configuration

**Dockerfile:**
```dockerfile
FROM node:18-alpine

WORKDIR /app

# Copy package files
COPY package*.json ./

# Install dependencies
RUN npm ci --only=production

# Copy source code
COPY . .

# Build TypeScript
RUN npm run build

# Create non-root user
RUN addgroup -g 1001 -S nodejs
RUN adduser -S nodejs -u 1001

# Change ownership
RUN chown -R nodejs:nodejs /app
USER nodejs

EXPOSE 3000

CMD ["npm", "start"]
```

**docker-compose.yml:**
```yaml
version: '3.8'

services:
  app:
    build: .
    ports:
      - "3000:3000"
    environment:
      - NODE_ENV=production
      - DB_HOST=mysql
      - DB_DATABASE=telnyx_10dlc
      - DB_USERNAME=root
      - DB_PASSWORD=password
    depends_on:
      - mysql
      - redis
    volumes:
      - ./logs:/app/logs

  mysql:
    image: mysql:8.0
    environment:
      - MYSQL_ROOT_PASSWORD=password
      - MYSQL_DATABASE=telnyx_10dlc
    volumes:
      - mysql_data:/var/lib/mysql
      - ./init.sql:/docker-entrypoint-initdb.d/init.sql
    ports:
      - "3306:3306"

  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"

  nginx:
    image: nginx:alpine
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf
      - ./ssl:/etc/nginx/ssl
    depends_on:
      - app

volumes:
  mysql_data:
```

### Environment-Specific Configurations

**Production Environment Variables:**
```bash
NODE_ENV=production
PORT=3000

# Database (use environment-specific values)
DB_HOST=your-production-db-host
DB_PORT=3306
DB_USERNAME=your-db-user
DB_PASSWORD=your-secure-db-password
DB_DATABASE=telnyx_10dlc_prod

# Telnyx (production API keys)
TELNYX_API_KEY=your-production-telnyx-api-key
TELNYX_WEBHOOK_SECRET=your-webhook-secret

# Security
JWT_SECRET=your-super-secure-jwt-secret-at-least-32-chars
JWT_EXPIRES_IN=7d

# External URLs
WEBHOOK_BASE_URL=https://your-production-domain.com

# Monitoring
LOG_LEVEL=warn
SENTRY_DSN=your-sentry-dsn
```

## Monitoring & Maintenance

### Logging Configuration
```typescript
// src/utils/logger.util.ts
import winston from 'winston';
import { config } from '../config/app.config';

export const logger = winston.createLogger({
  level: config.logging.level,
  format: winston.format.combine(
    winston.format.timestamp(),
    winston.format.errors({ stack: true }),
    winston.format.json()
  ),
  defaultMeta: { service: '10dlc-sms-api' },
  transports: [
    new winston.transports.File({ filename: 'logs/error.log', level: 'error' }),
    new winston.transports.File({ filename: 'logs/combined.log' }),
  ],
});

if (config.app.env !== 'production') {
  logger.add(new winston.transports.Console({
    format: winston.format.simple()
  }));
}
```

### Health Checks and Metrics
```typescript
// src/middleware/health.middleware.ts
import { Request, Response } from 'express';
import { AppDataSource } from '../config/database.config';

export const healthCheck = async (req: Request, res: Response) => {
  const health = {
    status: 'OK',
    timestamp: new Date().toISOString(),
    services: {
      database: 'unknown',
      telnyx: 'unknown',
    },
  };

  try {
    // Check database connection
    await AppDataSource.query('SELECT 1');
    health.services.database = 'healthy';
  } catch (error) {
    health.services.database = 'unhealthy';
    health.status = 'ERROR';
  }

  const statusCode = health.status === 'OK' ? 200 : 503;
  res.status(statusCode).json(health);
};
```

### Database Maintenance
- Regular backups with point-in-time recovery
- Index optimization for message queries
- Archive old messages (>6 months) to separate tables
- Monitor slow queries and optimize

### Monitoring Alerts
- Failed message delivery rates > 5%
- Opt-out rates > 2%
- API response times > 2 seconds
- Database connection failures
- Webhook processing delays

### Step 8: Complete Routes and Additional Files

**src/routes/brand.routes.ts:**
```typescript
import { Router } from 'express';
import { brandController } from '../controllers/brand.controller';
import { authMiddleware } from '../middleware/auth.middleware';
import { validateBrand } from '../middleware/validation.middleware';

const router = Router();

router.post('/', authMiddleware, validateBrand, brandController.createBrand);
router.get('/', authMiddleware, brandController.getBrands);
router.get('/:id', authMiddleware, brandController.getBrandById);
router.put('/:id', authMiddleware, validateBrand, brandController.updateBrand);
router.delete('/:id', authMiddleware, brandController.deleteBrand);

export default router;
```

**src/routes/campaign.routes.ts:**
```typescript
import { Router } from 'express';
import { campaignController } from '../controllers/campaign.controller';
import { authMiddleware } from '../middleware/auth.middleware';
import { validateCampaign } from '../middleware/validation.middleware';

const router = Router();

router.post('/', authMiddleware, validateCampaign, campaignController.createCampaign);
router.get('/', authMiddleware, campaignController.getCampaigns);
router.get('/:id', authMiddleware, campaignController.getCampaignById);
router.put('/:id', authMiddleware, validateCampaign, campaignController.updateCampaign);
router.delete('/:id', authMiddleware, campaignController.deleteCampaign);

export default router;
```

**package.json (complete example):**
```json
{
  "name": "telnyx-10dlc-system",
  "version": "1.0.0",
  "description": "10DLC SMS messaging system with TypeScript and MySQL",
  "main": "dist/app.js",
  "scripts": {
    "dev": "nodemon src/app.ts",
    "build": "tsc",
    "start": "node dist/app.js",
    "test": "jest",
    "test:watch": "jest --watch",
    "test:coverage": "jest --coverage",
    "lint": "eslint src/**/*.ts",
    "lint:fix": "eslint src/**/*.ts --fix",
    "format": "prettier --write src/**/*.ts",
    "migration:generate": "typeorm-ts-node-esm migration:generate",
    "migration:run": "typeorm-ts-node-esm migration:run",
    "migration:revert": "typeorm-ts-node-esm migration:revert",
    "db:seed": "ts-node src/seeds/index.ts"
  },
  "dependencies": {
    "express": "^4.18.2",
    "typeorm": "^0.3.17",
    "mysql2": "^3.6.0",
    "jsonwebtoken": "^9.0.2",
    "bcryptjs": "^2.4.3",
    "joi": "^17.9.2",
    "cors": "^2.8.5",
    "helmet": "^7.0.0",
    "morgan": "^1.10.0",
    "winston": "^3.10.0",
    "axios": "^1.5.0",
    "uuid": "^9.0.0",
    "dotenv": "^16.3.1",
    "express-rate-limit": "^6.8.1",
    "swagger-jsdoc": "^6.2.8",
    "swagger-ui-express": "^5.0.0",
    "reflect-metadata": "^0.1.13"
  },
  "devDependencies": {
    "typescript": "^5.1.6",
    "@types/node": "^20.4.9",
    "@types/express": "^4.17.17",
    "@types/jsonwebtoken": "^9.0.2",
    "@types/bcryptjs": "^2.4.2",
    "@types/cors": "^2.8.13",
    "@types/uuid": "^9.0.2",
    "@types/swagger-jsdoc": "^6.0.1",
    "@types/swagger-ui-express": "^4.1.3",
    "@types/morgan": "^1.9.4",
    "nodemon": "^3.0.1",
    "ts-node": "^10.9.1",
    "eslint": "^8.46.0",
    "prettier": "^3.0.0",
    "husky": "^8.0.3",
    "jest": "^29.6.2",
    "supertest": "^6.3.3",
    "@types/jest": "^29.5.3",
    "@types/supertest": "^2.0.12",
    "@typescript-eslint/eslint-plugin": "^6.2.1",
    "@typescript-eslint/parser": "^6.2.1"
  },
  "keywords": ["10dlc", "sms", "telnyx", "typescript", "mysql", "messaging"],
  "author": "Your Name",
  "license": "MIT"
}
```

### Step 9: Quick Start Guide

**Initial Setup Commands:**
```bash
# 1. Clone and setup
git clone <repository-url>
cd telnyx-10dlc-system
npm install

# 2. Setup environment
cp .env.example .env
# Edit .env with your configuration

# 3. Setup database
mysql -u root -p -e "CREATE DATABASE telnyx_10dlc;"

# 4. Run migrations
npm run migration:run

# 5. Start development server
npm run dev
```

### Step 10: API Usage Examples

**Authentication:**
```bash
# Register user
curl -X POST http://localhost:3000/api/auth/register \
  -H "Content-Type: application/json" \
  -d '{
    "email": "user@example.com",
    "password": "securepassword",
    "firstName": "John",
    "lastName": "Doe"
  }'

# Login
curl -X POST http://localhost:3000/api/auth/login \
  -H "Content-Type: application/json" \
  -d '{
    "email": "user@example.com",
    "password": "securepassword"
  }'
```

**Brand Management:**
```bash
# Create brand
curl -X POST http://localhost:3000/api/brands \
  -H "Authorization: Bearer YOUR_JWT_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "legalBusinessName": "Acme Corporation",
    "businessType": "Corporation",
    "einTaxId": "12-3456789",
    "businessAddress": "123 Main St, Anytown, NY 12345",
    "website": "https://acme.com",
    "contactPhone": "+12125551234",
    "contactEmail": "contact@acme.com"
  }'
```

**Campaign Management:**
```bash
# Create campaign
curl -X POST http://localhost:3000/api/campaigns \
  -H "Authorization: Bearer YOUR_JWT_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "brandId": "brand-uuid-here",
    "name": "Customer Notifications",
    "useCase": "account_notifications",
    "description": "Account alerts and notifications",
    "sampleMessages": [
      "Your account balance is $50.00",
      "Payment of $25.00 received",
      "Login detected from new device"
    ],
    "optInProcess": "Users opt in during account creation"
  }'
```

**Send Message:**
```bash
# Send SMS
curl -X POST http://localhost:3000/api/messages/send \
  -H "Authorization: Bearer YOUR_JWT_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "campaignId": "campaign-uuid-here",
    "from": "+12125551234",
    "to": "+13125551234",
    "text": "Hello! This is a test message from our 10DLC campaign."
  }'
```

### Step 11: Production Deployment Checklist

**Pre-Deployment:**
- [ ] Update all environment variables for production
- [ ] Set up SSL certificates
- [ ] Configure production database with backup strategy
- [ ] Set up monitoring and alerting
- [ ] Configure log rotation
- [ ] Set up CI/CD pipeline
- [ ] Security audit and vulnerability scanning
- [ ] Load testing and performance optimization

**Environment Variables for Production:**
```bash
NODE_ENV=production
PORT=3000
JWT_SECRET=your-super-secure-jwt-secret-min-256-bits
DB_HOST=your-prod-db-host
DB_PASSWORD=your-secure-db-password
TELNYX_API_KEY=your-production-telnyx-key
WEBHOOK_BASE_URL=https://yourdomain.com
LOG_LEVEL=warn
```

**Docker Production Configuration:**
```yaml
# docker-compose.prod.yml
version: '3.8'
services:
  app:
    image: your-registry/telnyx-10dlc:latest
    environment:
      - NODE_ENV=production
    deploy:
      replicas: 3
      restart_policy:
        condition: on-failure
        delay: 5s
        max_attempts: 3
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:3000/health"]
      interval: 30s
      timeout: 10s
      retries: 3
```

**Nginx Configuration:**
```nginx
server {
    listen 80;
    server_name yourdomain.com;
    return 301 https://$server_name$request_uri;
}

server {
    listen 443 ssl;
    server_name yourdomain.com;

    ssl_certificate /path/to/certificate.crt;
    ssl_certificate_key /path/to/private.key;

    location / {
        proxy_pass http://localhost:3000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }

    location /api/webhooks/ {
        proxy_pass http://localhost:3000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_buffering off;
    }
}
```

## Conclusion

This comprehensive development guide provides a complete foundation for building a production-ready 10DLC SMS system with TypeScript and MySQL. The architecture is scalable, secure, and compliant with carrier requirements.

### Key Features Implemented:
✅ **Complete TypeScript + MySQL architecture**
✅ **10DLC compliance with Telnyx integration**
✅ **Brand and campaign management**
✅ **Message sending/receiving with opt-out handling**
✅ **Webhook processing for delivery reports**
✅ **JWT and API key authentication**
✅ **Rate limiting and security middleware**
✅ **Comprehensive validation and error handling**
✅ **Docker containerization**
✅ **API documentation with Swagger**
✅ **Logging and monitoring setup**
✅ **Production deployment configuration**

### Next Steps:
1. Set up your development environment using the provided configuration
2. Register with Telnyx and obtain API credentials
3. Configure your database and environment variables
4. Follow the step-by-step implementation guide
5. Test the system with your 10DLC brand and campaign registration
6. Deploy to production following the deployment checklist

The system handles the complete 10DLC workflow from brand registration through message delivery, ensuring compliance with U.S. carrier regulations while providing a robust, scalable foundation for SMS messaging operations.

<function_calls>
<invoke name="todo_write">
<parameter name="merge">true
