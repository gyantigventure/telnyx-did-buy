# Testing Strategy Document - 10DLC SMS System

## Table of Contents
1. [Overview](#overview)
2. [Testing Philosophy](#testing-philosophy)
3. [Test Pyramid Strategy](#test-pyramid-strategy)
4. [Unit Testing](#unit-testing)
5. [Integration Testing](#integration-testing)
6. [End-to-End Testing](#end-to-end-testing)
7. [Performance Testing](#performance-testing)
8. [Security Testing](#security-testing)
9. [Compliance Testing](#compliance-testing)
10. [Test Automation](#test-automation)
11. [Quality Gates](#quality-gates)
12. [Future Testing Enhancements](#future-testing-enhancements)

## Overview

This document outlines the comprehensive testing strategy for the 10DLC SMS system, ensuring high quality, reliability, and compliance through systematic testing approaches across all system components.

### Testing Objectives
- **Quality Assurance**: Maintain 95%+ code coverage
- **Reliability**: Achieve 99.9% system uptime
- **Performance**: Ensure <200ms API response times
- **Security**: Validate all security controls
- **Compliance**: Verify 10DLC and TCPA compliance
- **Regression Prevention**: Catch issues before production

### Testing Principles
- **Shift-Left Testing**: Early detection and prevention
- **Test Automation**: Minimize manual testing overhead
- **Risk-Based Testing**: Focus on high-risk areas
- **Continuous Testing**: Integrated into CI/CD pipeline
- **Test Data Management**: Realistic, privacy-compliant data
- **Feedback Loops**: Rapid feedback to development teams

## Testing Philosophy

### Test-Driven Development (TDD)
```typescript
// Example TDD cycle for message service
describe('MessageService', () => {
  describe('sendMessage', () => {
    it('should send message successfully when campaign is approved', async () => {
      // Red: Write failing test first
      const mockCampaign = createMockCampaign({ status: 'approved' });
      const mockTelnyxResponse = createMockTelnyxResponse();
      
      campaignService.getCampaign.mockResolvedValue(mockCampaign);
      telnyxService.sendMessage.mockResolvedValue(mockTelnyxResponse);
      
      const result = await messageService.sendMessage({
        campaignId: 'test-campaign',
        from: '+12125551234',
        to: '+13125551234',
        text: 'Test message'
      });
      
      expect(result.status).toBe('sent');
      expect(result.telnyxMessageId).toBe(mockTelnyxResponse.id);
    });
    
    // Green: Implement minimal code to pass test
    // Refactor: Improve code while keeping tests green
  });
});
```

### Behavior-Driven Development (BDD)
```gherkin
# features/message-sending.feature
Feature: Message Sending
  As a business user
  I want to send SMS messages through approved campaigns
  So that I can communicate with my customers

  Background:
    Given I have an approved brand "Acme Corp"
    And I have an approved campaign "Customer Notifications"
    And I have a phone number "+12125551234" assigned to the campaign

  Scenario: Send message to opted-in recipient
    Given the recipient "+13125551234" has opted in to the campaign
    When I send a message "Your order has shipped" from "+12125551234" to "+13125551234"
    Then the message should be sent successfully
    And the message status should be "sent"
    And I should receive a delivery confirmation

  Scenario: Prevent message to opted-out recipient
    Given the recipient "+13125551234" has opted out of the campaign
    When I attempt to send a message "Your order has shipped" from "+12125551234" to "+13125551234"
    Then the message should be rejected
    And I should receive an error "Recipient has opted out"
```

## Test Pyramid Strategy

### Test Distribution
```
                    🔺
                   /|\
                  / | \
                 /  |  \
                /   |   \
               /    |    \
              /     |     \
             /  E2E |      \
            /   5%  |       \
           /_______|________\
          /        |         \
         /Integration|         \
        /    15%    |          \
       /____________|___________\
      /             |            \
     /     Unit     |             \
    /     80%       |              \
   /________________|_______________\

```

### Test Types by Layer
| Layer | Purpose | Tools | Coverage |
|-------|---------|--------|----------|
| Unit | Individual functions/classes | Jest, Mocha | 80% |
| Integration | Component interactions | Supertest, TestContainers | 15% |
| E2E | Complete user workflows | Playwright, Cypress | 5% |

## Unit Testing

### Jest Configuration
```javascript
// jest.config.js
module.exports = {
  preset: 'ts-jest',
  testEnvironment: 'node',
  roots: ['<rootDir>/src', '<rootDir>/tests'],
  testMatch: [
    '**/__tests__/**/*.ts',
    '**/?(*.)+(spec|test).ts'
  ],
  transform: {
    '^.+\\.ts$': 'ts-jest'
  },
  collectCoverageFrom: [
    'src/**/*.ts',
    '!src/**/*.d.ts',
    '!src/migrations/**',
    '!src/seeds/**'
  ],
  coverageThreshold: {
    global: {
      branches: 80,
      functions: 80,
      lines: 80,
      statements: 80
    }
  },
  setupFilesAfterEnv: ['<rootDir>/tests/setup.ts'],
  globalTeardown: '<rootDir>/tests/teardown.ts'
};
```

### Unit Test Examples

#### Service Layer Testing
```typescript
// tests/unit/services/message.service.test.ts
import { MessageService } from '../../../src/services/message.service';
import { TelnyxService } from '../../../src/services/telnyx.service';
import { Campaign } from '../../../src/entities/Campaign';

jest.mock('../../../src/services/telnyx.service');
jest.mock('../../../src/config/database.config');

describe('MessageService', () => {
  let messageService: MessageService;
  let mockTelnyxService: jest.Mocked<TelnyxService>;
  let mockRepository: any;

  beforeEach(() => {
    mockTelnyxService = new TelnyxService() as jest.Mocked<TelnyxService>;
    mockRepository = {
      findOne: jest.fn(),
      save: jest.fn(),
      create: jest.fn()
    };
    
    messageService = new MessageService();
    (messageService as any).campaignRepository = mockRepository;
    (messageService as any).telnyxService = mockTelnyxService;
  });

  describe('sendMessage', () => {
    const messageRequest = {
      campaignId: 'campaign-123',
      from: '+12125551234',
      to: '+13125551234',
      text: 'Test message'
    };

    it('should send message successfully for approved campaign', async () => {
      // Arrange
      const mockCampaign = {
        id: 'campaign-123',
        status: 'approved',
        brand: { status: 'approved' }
      } as Campaign;

      const mockTelnyxResponse = {
        id: 'telnyx-msg-123',
        cost: { amount: '0.0075' },
        parts: 1
      };

      mockRepository.findOne.mockResolvedValue(mockCampaign);
      mockTelnyxService.sendMessage.mockResolvedValue(mockTelnyxResponse);
      mockRepository.create.mockReturnValue({ id: 'msg-123' });
      mockRepository.save.mockResolvedValue({ id: 'msg-123' });

      // Act
      const result = await messageService.sendMessage(messageRequest);

      // Assert
      expect(result).toMatchObject({
        id: 'msg-123',
        telnyxMessageId: 'telnyx-msg-123',
        status: 'sent',
        cost: 0.0075,
        segments: 1
      });

      expect(mockTelnyxService.sendMessage).toHaveBeenCalledWith({
        from: messageRequest.from,
        to: messageRequest.to,
        text: messageRequest.text,
        mediaUrls: undefined,
        webhookUrl: expect.stringContaining('/api/webhooks/telnyx')
      });
    });

    it('should reject message for unapproved campaign', async () => {
      // Arrange
      const mockCampaign = {
        id: 'campaign-123',
        status: 'pending'
      } as Campaign;

      mockRepository.findOne.mockResolvedValue(mockCampaign);

      // Act & Assert
      await expect(messageService.sendMessage(messageRequest))
        .rejects
        .toThrow('Campaign is not approved for messaging');

      expect(mockTelnyxService.sendMessage).not.toHaveBeenCalled();
    });

    it('should handle Telnyx API failures gracefully', async () => {
      // Arrange
      const mockCampaign = {
        id: 'campaign-123',
        status: 'approved',
        brand: { status: 'approved' }
      } as Campaign;

      mockRepository.findOne.mockResolvedValue(mockCampaign);
      mockTelnyxService.sendMessage.mockRejectedValue(new Error('Telnyx API error'));
      mockRepository.create.mockReturnValue({ id: 'msg-123' });
      mockRepository.save.mockResolvedValue({ id: 'msg-123', status: 'failed' });

      // Act & Assert
      await expect(messageService.sendMessage(messageRequest))
        .rejects
        .toThrow('Failed to send message: Telnyx API error');

      expect(mockRepository.save).toHaveBeenCalledWith(
        expect.objectContaining({
          status: 'failed',
          errorMessage: 'Telnyx API error'
        })
      );
    });
  });

  describe('checkOptOut', () => {
    it('should return opt-out status correctly', async () => {
      // Arrange
      const phoneNumber = '+13125551234';
      const campaignId = 'campaign-123';
      
      const mockOptOut = {
        phoneNumber,
        campaignId,
        optedOutAt: new Date()
      };

      (messageService as any).optOutRepository = {
        findOne: jest.fn().mockResolvedValue(mockOptOut)
      };

      // Act
      const result = await messageService.checkOptOut(phoneNumber, campaignId);

      // Assert
      expect(result).toMatchObject({
        phoneNumber,
        isOptedOut: true,
        campaignId
      });
    });
  });
});
```

#### Repository Testing
```typescript
// tests/unit/repositories/campaign.repository.test.ts
import { Test, TestingModule } from '@nestjs/testing';
import { getRepositoryToken } from '@nestjs/typeorm';
import { Repository } from 'typeorm';
import { Campaign } from '../../../src/entities/Campaign';
import { CampaignRepository } from '../../../src/repositories/campaign.repository';

describe('CampaignRepository', () => {
  let repository: CampaignRepository;
  let mockTypeOrmRepository: Repository<Campaign>;

  beforeEach(async () => {
    const module: TestingModule = await Test.createTestingModule({
      providers: [
        CampaignRepository,
        {
          provide: getRepositoryToken(Campaign),
          useClass: Repository
        }
      ]
    }).compile();

    repository = module.get<CampaignRepository>(CampaignRepository);
    mockTypeOrmRepository = module.get<Repository<Campaign>>(getRepositoryToken(Campaign));
  });

  describe('findByBrandId', () => {
    it('should return campaigns for given brand', async () => {
      // Arrange
      const brandId = 'brand-123';
      const expectedCampaigns = [
        { id: 'campaign-1', brandId, name: 'Campaign 1' },
        { id: 'campaign-2', brandId, name: 'Campaign 2' }
      ] as Campaign[];

      jest.spyOn(mockTypeOrmRepository, 'find').mockResolvedValue(expectedCampaigns);

      // Act
      const result = await repository.findByBrandId(brandId);

      // Assert
      expect(result).toEqual(expectedCampaigns);
      expect(mockTypeOrmRepository.find).toHaveBeenCalledWith({
        where: { brandId },
        relations: ['brand'],
        order: { createdAt: 'DESC' }
      });
    });
  });
});
```

### Mocking Strategies
```typescript
// tests/mocks/telnyx.service.mock.ts
export const createMockTelnyxService = () => ({
  sendMessage: jest.fn().mockResolvedValue({
    id: 'telnyx-msg-123',
    cost: { amount: '0.0075' },
    parts: 1
  }),
  createBrand: jest.fn().mockResolvedValue({
    id: 'telnyx-brand-123',
    displayName: 'Test Brand'
  }),
  createCampaign: jest.fn().mockResolvedValue({
    id: 'telnyx-campaign-123',
    brandId: 'telnyx-brand-123'
  }),
  validateWebhookSignature: jest.fn().mockReturnValue(true)
});

// tests/fixtures/campaign.fixture.ts
export const createMockCampaign = (overrides: Partial<Campaign> = {}): Campaign => ({
  id: 'campaign-123',
  brandId: 'brand-123',
  name: 'Test Campaign',
  useCase: 'customer_care',
  status: 'approved',
  sampleMessages: ['Hello, this is a test message'],
  optInProcess: 'Users opt in during signup',
  optOutInstructions: 'Reply STOP to unsubscribe',
  createdAt: new Date(),
  updatedAt: new Date(),
  ...overrides
} as Campaign);
```

## Integration Testing

### Test Database Setup
```typescript
// tests/integration/setup.ts
import { DataSource } from 'typeorm';
import { PostgreSqlContainer } from 'testcontainers';

export class TestDatabase {
  private static container: PostgreSqlContainer;
  private static dataSource: DataSource;

  static async setup(): Promise<DataSource> {
    // Start test database container
    this.container = await new PostgreSqlContainer('postgres:13')
      .withDatabase('test_sms_system')
      .withUsername('test_user')
      .withPassword('test_password')
      .start();

    // Create data source
    this.dataSource = new DataSource({
      type: 'postgres',
      host: this.container.getHost(),
      port: this.container.getMappedPort(5432),
      username: 'test_user',
      password: 'test_password',
      database: 'test_sms_system',
      entities: ['src/entities/*.ts'],
      synchronize: true,
      logging: false
    });

    await this.dataSource.initialize();
    return this.dataSource;
  }

  static async teardown(): Promise<void> {
    if (this.dataSource) {
      await this.dataSource.destroy();
    }
    if (this.container) {
      await this.container.stop();
    }
  }

  static async cleanup(): Promise<void> {
    const entities = this.dataSource.entityMetadatas;
    
    for (const entity of entities) {
      const repository = this.dataSource.getRepository(entity.name);
      await repository.query(`TRUNCATE TABLE "${entity.tableName}" CASCADE;`);
    }
  }
}
```

### API Integration Tests
```typescript
// tests/integration/api/message.api.test.ts
import request from 'supertest';
import { App } from '../../../src/app';
import { TestDatabase } from '../setup';
import { createTestUser, createTestCampaign } from '../fixtures';

describe('Message API Integration', () => {
  let app: App;
  let server: any;
  let authToken: string;

  beforeAll(async () => {
    await TestDatabase.setup();
    app = new App();
    server = app.listen(0);
  });

  afterAll(async () => {
    await server.close();
    await TestDatabase.teardown();
  });

  beforeEach(async () => {
    await TestDatabase.cleanup();
    
    // Create test user and get auth token
    const user = await createTestUser();
    const loginResponse = await request(server)
      .post('/api/auth/login')
      .send({
        email: user.email,
        password: 'test123'
      });
    
    authToken = loginResponse.body.data.tokens.accessToken;
  });

  describe('POST /api/messages/send', () => {
    it('should send message successfully', async () => {
      // Arrange
      const campaign = await createTestCampaign({ status: 'approved' });
      const messageData = {
        campaignId: campaign.id,
        from: '+12125551234',
        to: '+13125551234',
        text: 'Integration test message'
      };

      // Act
      const response = await request(server)
        .post('/api/messages/send')
        .set('Authorization', `Bearer ${authToken}`)
        .send(messageData)
        .expect(200);

      // Assert
      expect(response.body.success).toBe(true);
      expect(response.body.data).toMatchObject({
        status: 'sent',
        telnyxMessageId: expect.any(String)
      });
    });

    it('should return 401 without authentication', async () => {
      // Act
      const response = await request(server)
        .post('/api/messages/send')
        .send({
          campaignId: 'campaign-123',
          from: '+12125551234',
          to: '+13125551234',
          text: 'Test message'
        })
        .expect(401);

      // Assert
      expect(response.body.success).toBe(false);
      expect(response.body.error).toContain('authorization');
    });

    it('should validate request body', async () => {
      // Act
      const response = await request(server)
        .post('/api/messages/send')
        .set('Authorization', `Bearer ${authToken}`)
        .send({
          campaignId: 'invalid-id',
          from: 'invalid-phone',
          to: 'invalid-phone',
          text: ''
        })
        .expect(400);

      // Assert
      expect(response.body.success).toBe(false);
      expect(response.body.errors).toEqual(
        expect.arrayContaining([
          expect.stringContaining('phone number'),
          expect.stringContaining('text')
        ])
      );
    });
  });
});
```

### Database Integration Tests
```typescript
// tests/integration/database/message.repository.test.ts
import { DataSource } from 'typeorm';
import { TestDatabase } from '../setup';
import { MessageRepository } from '../../../src/repositories/message.repository';
import { Message } from '../../../src/entities/Message';
import { createTestCampaign, createTestPhoneNumber } from '../fixtures';

describe('MessageRepository Integration', () => {
  let dataSource: DataSource;
  let repository: MessageRepository;

  beforeAll(async () => {
    dataSource = await TestDatabase.setup();
    repository = new MessageRepository(dataSource);
  });

  afterAll(async () => {
    await TestDatabase.teardown();
  });

  beforeEach(async () => {
    await TestDatabase.cleanup();
  });

  describe('getMessagesByCampaign', () => {
    it('should return messages ordered by creation date', async () => {
      // Arrange
      const campaign = await createTestCampaign();
      const phoneNumber = await createTestPhoneNumber();
      
      const messages = await Promise.all([
        repository.save({
          campaignId: campaign.id,
          phoneNumberId: phoneNumber.id,
          fromNumber: '+12125551234',
          toNumber: '+13125551234',
          messageText: 'First message',
          direction: 'outbound',
          status: 'sent'
        }),
        repository.save({
          campaignId: campaign.id,
          phoneNumberId: phoneNumber.id,
          fromNumber: '+12125551234',
          toNumber: '+13125551234',
          messageText: 'Second message',
          direction: 'outbound',
          status: 'sent'
        })
      ]);

      // Act
      const result = await repository.getMessagesByCampaign(campaign.id, 10);

      // Assert
      expect(result).toHaveLength(2);
      expect(result[0].messageText).toBe('Second message'); // Most recent first
      expect(result[1].messageText).toBe('First message');
    });

    it('should respect limit parameter', async () => {
      // Arrange
      const campaign = await createTestCampaign();
      const phoneNumber = await createTestPhoneNumber();
      
      // Create 5 messages
      for (let i = 0; i < 5; i++) {
        await repository.save({
          campaignId: campaign.id,
          phoneNumberId: phoneNumber.id,
          fromNumber: '+12125551234',
          toNumber: '+13125551234',
          messageText: `Message ${i}`,
          direction: 'outbound',
          status: 'sent'
        });
      }

      // Act
      const result = await repository.getMessagesByCampaign(campaign.id, 3);

      // Assert
      expect(result).toHaveLength(3);
    });
  });
});
```

## End-to-End Testing

### Playwright Configuration
```typescript
// tests/e2e/playwright.config.ts
import { defineConfig } from '@playwright/test';

export default defineConfig({
  testDir: './tests/e2e',
  fullyParallel: true,
  forbidOnly: !!process.env.CI,
  retries: process.env.CI ? 2 : 0,
  workers: process.env.CI ? 1 : undefined,
  reporter: [
    ['html'],
    ['junit', { outputFile: 'test-results/junit.xml' }]
  ],
  use: {
    baseURL: process.env.BASE_URL || 'http://localhost:3000',
    trace: 'on-first-retry',
    screenshot: 'only-on-failure',
    video: 'retain-on-failure'
  },
  projects: [
    {
      name: 'chromium',
      use: { ...devices['Desktop Chrome'] }
    },
    {
      name: 'firefox',
      use: { ...devices['Desktop Firefox'] }
    },
    {
      name: 'webkit',
      use: { ...devices['Desktop Safari'] }
    }
  ],
  webServer: {
    command: 'npm run start:test',
    port: 3000,
    reuseExistingServer: !process.env.CI
  }
});
```

### E2E Test Examples
```typescript
// tests/e2e/message-flow.spec.ts
import { test, expect } from '@playwright/test';

test.describe('Message Sending Flow', () => {
  test.beforeEach(async ({ page }) => {
    // Login to application
    await page.goto('/login');
    await page.fill('[data-testid=email]', 'test@example.com');
    await page.fill('[data-testid=password]', 'test123');
    await page.click('[data-testid=login-button]');
    await expect(page).toHaveURL('/dashboard');
  });

  test('should send message through complete flow', async ({ page }) => {
    // Navigate to message sending page
    await page.click('[data-testid=send-message-nav]');
    await expect(page).toHaveURL('/messages/send');

    // Select campaign
    await page.selectOption('[data-testid=campaign-select]', 'campaign-123');

    // Fill message form
    await page.fill('[data-testid=from-number]', '+12125551234');
    await page.fill('[data-testid=to-number]', '+13125551234');
    await page.fill('[data-testid=message-text]', 'Hello from E2E test!');

    // Send message
    await page.click('[data-testid=send-button]');

    // Verify success
    await expect(page.locator('[data-testid=success-message]')).toBeVisible();
    await expect(page.locator('[data-testid=message-id]')).toContainText(/msg-/);

    // Verify message appears in history
    await page.click('[data-testid=message-history-nav]');
    await expect(page.locator('[data-testid=message-list]')).toContainText('Hello from E2E test!');
  });

  test('should prevent sending to opted-out recipient', async ({ page }) => {
    // Navigate to message sending page
    await page.click('[data-testid=send-message-nav]');

    // Select campaign
    await page.selectOption('[data-testid=campaign-select]', 'campaign-123');

    // Try to send to opted-out number
    await page.fill('[data-testid=from-number]', '+12125551234');
    await page.fill('[data-testid=to-number]', '+13125555555'); // Opted-out number
    await page.fill('[data-testid=message-text]', 'This should be blocked');

    await page.click('[data-testid=send-button]');

    // Verify error message
    await expect(page.locator('[data-testid=error-message]')).toContainText('opted out');
  });
});

// tests/e2e/campaign-management.spec.ts
test.describe('Campaign Management', () => {
  test('should create and approve campaign', async ({ page }) => {
    // Navigate to campaigns
    await page.goto('/campaigns');
    await page.click('[data-testid=create-campaign-button]');

    // Fill campaign form
    await page.fill('[data-testid=campaign-name]', 'E2E Test Campaign');
    await page.selectOption('[data-testid=use-case]', 'customer_care');
    await page.fill('[data-testid=description]', 'Campaign for E2E testing');
    await page.fill('[data-testid=opt-in-process]', 'Users opt in during signup');

    // Add sample messages
    await page.fill('[data-testid=sample-message-1]', 'Welcome to our service!');
    await page.fill('[data-testid=sample-message-2]', 'Your order has been confirmed.');

    // Submit campaign
    await page.click('[data-testid=submit-campaign]');

    // Verify campaign creation
    await expect(page.locator('[data-testid=success-notification]')).toBeVisible();
    await expect(page).toHaveURL(/\/campaigns\/[a-f0-9-]+/);

    // Verify campaign appears in list
    await page.goto('/campaigns');
    await expect(page.locator('[data-testid=campaign-list]')).toContainText('E2E Test Campaign');
  });
});
```

### API E2E Tests
```typescript
// tests/e2e/api/message-flow.api.spec.ts
import { test, expect } from '@playwright/test';

test.describe('Message API E2E', () => {
  let authToken: string;

  test.beforeAll(async ({ request }) => {
    // Get authentication token
    const loginResponse = await request.post('/api/auth/login', {
      data: {
        email: 'test@example.com',
        password: 'test123'
      }
    });

    const loginData = await loginResponse.json();
    authToken = loginData.data.tokens.accessToken;
  });

  test('should complete full message lifecycle', async ({ request }) => {
    // 1. Create brand
    const brandResponse = await request.post('/api/brands', {
      headers: { Authorization: `Bearer ${authToken}` },
      data: {
        legalBusinessName: 'E2E Test Corp',
        businessType: 'Corporation',
        businessAddress: '123 Test St, Test City, TS 12345',
        contactPhone: '+12125551234',
        contactEmail: 'test@e2etest.com'
      }
    });

    expect(brandResponse.ok()).toBeTruthy();
    const brand = await brandResponse.json();

    // 2. Create campaign
    const campaignResponse = await request.post('/api/campaigns', {
      headers: { Authorization: `Bearer ${authToken}` },
      data: {
        brandId: brand.data.id,
        name: 'E2E Test Campaign',
        useCase: 'customer_care',
        description: 'Campaign for E2E testing',
        sampleMessages: ['Hello from E2E test'],
        optInProcess: 'Users opt in during signup'
      }
    });

    expect(campaignResponse.ok()).toBeTruthy();
    const campaign = await campaignResponse.json();

    // 3. Purchase phone number
    const numberResponse = await request.post('/api/numbers/purchase', {
      headers: { Authorization: `Bearer ${authToken}` },
      data: {
        phoneNumber: '+12125559999'
      }
    });

    expect(numberResponse.ok()).toBeTruthy();
    const phoneNumber = await numberResponse.json();

    // 4. Assign number to campaign
    await request.post(`/api/numbers/${phoneNumber.data.id}/assign`, {
      headers: { Authorization: `Bearer ${authToken}` },
      data: {
        campaignId: campaign.data.id
      }
    });

    // 5. Send message
    const messageResponse = await request.post('/api/messages/send', {
      headers: { Authorization: `Bearer ${authToken}` },
      data: {
        campaignId: campaign.data.id,
        from: '+12125559999',
        to: '+13125551234',
        text: 'Hello from E2E API test!'
      }
    });

    expect(messageResponse.ok()).toBeTruthy();
    const message = await messageResponse.json();
    expect(message.data.status).toBe('sent');

    // 6. Verify message in history
    const historyResponse = await request.get(`/api/messages/history/${campaign.data.id}`, {
      headers: { Authorization: `Bearer ${authToken}` }
    });

    expect(historyResponse.ok()).toBeTruthy();
    const history = await historyResponse.json();
    expect(history.data.messages).toHaveLength(1);
    expect(history.data.messages[0].messageText).toBe('Hello from E2E API test!');
  });
});
```

## Performance Testing

### Load Testing with K6
```javascript
// tests/performance/load-test.js
import http from 'k6/http';
import { check, sleep } from 'k6';
import { Rate } from 'k6/metrics';

export let errorRate = new Rate('errors');

export let options = {
  stages: [
    { duration: '2m', target: 100 }, // Ramp up to 100 users
    { duration: '5m', target: 100 }, // Stay at 100 users
    { duration: '2m', target: 200 }, // Ramp up to 200 users
    { duration: '5m', target: 200 }, // Stay at 200 users
    { duration: '2m', target: 0 },   // Ramp down to 0 users
  ],
  thresholds: {
    http_req_duration: ['p(95)<500'], // 95% of requests must complete below 500ms
    errors: ['rate<0.01'], // Error rate must be below 1%
  },
};

const API_BASE_URL = __ENV.API_BASE_URL || 'http://localhost:3000';
const AUTH_TOKEN = __ENV.AUTH_TOKEN;

export function setup() {
  // Login and get auth token
  const loginResponse = http.post(`${API_BASE_URL}/api/auth/login`, 
    JSON.stringify({
      email: 'loadtest@example.com',
      password: 'loadtest123'
    }),
    { headers: { 'Content-Type': 'application/json' } }
  );
  
  return { authToken: loginResponse.json('data.tokens.accessToken') };
}

export default function(data) {
  const headers = {
    'Authorization': `Bearer ${data.authToken}`,
    'Content-Type': 'application/json'
  };

  // Test message sending endpoint
  const messagePayload = JSON.stringify({
    campaignId: 'load-test-campaign',
    from: '+12125551234',
    to: `+1312555${Math.floor(Math.random() * 10000).toString().padStart(4, '0')}`,
    text: `Load test message ${Math.random()}`
  });

  const messageResponse = http.post(
    `${API_BASE_URL}/api/messages/send`,
    messagePayload,
    { headers }
  );

  check(messageResponse, {
    'message sent successfully': (r) => r.status === 200,
    'response time < 500ms': (r) => r.timings.duration < 500,
  }) || errorRate.add(1);

  // Test campaign listing endpoint
  const campaignResponse = http.get(`${API_BASE_URL}/api/campaigns`, { headers });
  
  check(campaignResponse, {
    'campaigns retrieved': (r) => r.status === 200,
    'response time < 200ms': (r) => r.timings.duration < 200,
  }) || errorRate.add(1);

  sleep(1);
}

export function teardown(data) {
  // Cleanup if needed
}
```

### Stress Testing
```javascript
// tests/performance/stress-test.js
import http from 'k6/http';
import { check } from 'k6';

export let options = {
  stages: [
    { duration: '5m', target: 500 },  // Ramp up to 500 users
    { duration: '10m', target: 500 }, // Stay at 500 users
    { duration: '5m', target: 1000 }, // Ramp up to 1000 users
    { duration: '10m', target: 1000 }, // Stay at 1000 users
    { duration: '5m', target: 0 },    // Ramp down
  ],
  thresholds: {
    http_req_duration: ['p(95)<2000'], // 95% of requests under 2s
    http_req_failed: ['rate<0.05'],    // Error rate under 5%
  },
};

export default function() {
  // High-load message sending
  const response = http.post(`${API_BASE_URL}/api/messages/bulk`, 
    JSON.stringify({
      campaignId: 'stress-test-campaign',
      from: '+12125551234',
      messages: Array.from({ length: 100 }, (_, i) => ({
        to: `+1312555${i.toString().padStart(4, '0')}`,
        text: `Stress test message ${i}`
      }))
    }),
    { 
      headers: { 
        'Authorization': `Bearer ${AUTH_TOKEN}`,
        'Content-Type': 'application/json' 
      } 
    }
  );

  check(response, {
    'bulk send accepted': (r) => r.status === 202,
    'response time acceptable': (r) => r.timings.duration < 5000,
  });
}
```

### Performance Benchmarks
```typescript
// tests/performance/benchmarks.test.ts
import { performance } from 'perf_hooks';
import { MessageService } from '../../src/services/message.service';

describe('Performance Benchmarks', () => {
  let messageService: MessageService;

  beforeEach(() => {
    messageService = new MessageService();
  });

  test('message sending should complete within performance threshold', async () => {
    const iterations = 1000;
    const maxDuration = 100; // 100ms per message
    
    const startTime = performance.now();
    
    const promises = Array.from({ length: iterations }, () =>
      messageService.sendMessage({
        campaignId: 'benchmark-campaign',
        from: '+12125551234',
        to: '+13125551234',
        text: 'Benchmark test message'
      })
    );
    
    await Promise.all(promises);
    
    const endTime = performance.now();
    const avgDuration = (endTime - startTime) / iterations;
    
    expect(avgDuration).toBeLessThan(maxDuration);
  });

  test('database queries should be optimized', async () => {
    const campaignId = 'benchmark-campaign';
    const iterations = 100;
    const maxDuration = 50; // 50ms per query
    
    const startTime = performance.now();
    
    for (let i = 0; i < iterations; i++) {
      await messageService.getMessageHistory(campaignId, 1, 50);
    }
    
    const endTime = performance.now();
    const avgDuration = (endTime - startTime) / iterations;
    
    expect(avgDuration).toBeLessThan(maxDuration);
  });
});
```

## Security Testing

### Security Test Suite
```typescript
// tests/security/security.test.ts
import request from 'supertest';
import { App } from '../../src/app';

describe('Security Tests', () => {
  let app: App;
  let server: any;

  beforeAll(() => {
    app = new App();
    server = app.listen(0);
  });

  afterAll(async () => {
    await server.close();
  });

  describe('Authentication Security', () => {
    test('should reject requests without authentication', async () => {
      const response = await request(server)
        .get('/api/campaigns')
        .expect(401);

      expect(response.body.error).toContain('authorization');
    });

    test('should reject invalid JWT tokens', async () => {
      const response = await request(server)
        .get('/api/campaigns')
        .set('Authorization', 'Bearer invalid-token')
        .expect(401);

      expect(response.body.error).toContain('Invalid token');
    });

    test('should implement rate limiting', async () => {
      const promises = Array.from({ length: 150 }, () =>
        request(server)
          .post('/api/auth/login')
          .send({ email: 'test@example.com', password: 'wrong' })
      );

      const responses = await Promise.all(promises);
      const rateLimitedResponses = responses.filter(r => r.status === 429);
      
      expect(rateLimitedResponses.length).toBeGreaterThan(0);
    });
  });

  describe('Input Validation Security', () => {
    test('should prevent SQL injection', async () => {
      const maliciousInput = "'; DROP TABLE users; --";
      
      const response = await request(server)
        .post('/api/brands')
        .set('Authorization', 'Bearer valid-token')
        .send({
          legalBusinessName: maliciousInput,
          businessType: 'Corporation',
          businessAddress: '123 Test St',
          contactPhone: '+12125551234',
          contactEmail: 'test@example.com'
        });

      // Should either reject or sanitize, not cause internal error
      expect([400, 422]).toContain(response.status);
    });

    test('should prevent XSS attacks', async () => {
      const xssPayload = '<script>alert("xss")</script>';
      
      const response = await request(server)
        .post('/api/campaigns')
        .set('Authorization', 'Bearer valid-token')
        .send({
          brandId: 'brand-123',
          name: xssPayload,
          useCase: 'customer_care',
          sampleMessages: [xssPayload],
          optInProcess: 'Users opt in'
        });

      if (response.status === 201) {
        // If accepted, ensure XSS payload is sanitized
        expect(response.body.data.name).not.toContain('<script>');
      }
    });

    test('should validate phone number formats', async () => {
      const invalidPhones = [
        'not-a-phone',
        '123',
        '+1234567890123456789', // Too long
        '1234567890' // Missing +
      ];

      for (const phone of invalidPhones) {
        const response = await request(server)
          .post('/api/messages/send')
          .set('Authorization', 'Bearer valid-token')
          .send({
            campaignId: 'campaign-123',
            from: phone,
            to: '+13125551234',
            text: 'Test'
          });

        expect(response.status).toBe(400);
        expect(response.body.error).toContain('phone');
      }
    });
  });

  describe('Authorization Security', () => {
    test('should enforce campaign access control', async () => {
      // User A tries to access User B's campaign
      const userAToken = 'user-a-token';
      const userBCampaignId = 'user-b-campaign';

      const response = await request(server)
        .get(`/api/campaigns/${userBCampaignId}`)
        .set('Authorization', `Bearer ${userAToken}`)
        .expect(403);

      expect(response.body.error).toContain('access denied');
    });

    test('should prevent privilege escalation', async () => {
      const userToken = 'regular-user-token';

      const response = await request(server)
        .post('/api/admin/users')
        .set('Authorization', `Bearer ${userToken}`)
        .send({ email: 'newuser@example.com' })
        .expect(403);

      expect(response.body.error).toContain('insufficient permissions');
    });
  });
});
```

### Penetration Testing
```bash
#!/bin/bash
# tests/security/pentest.sh

echo "Starting security penetration tests..."

API_BASE="http://localhost:3000"
RESULTS_DIR="./security-test-results"

mkdir -p $RESULTS_DIR

# 1. SQL Injection Tests
echo "Testing SQL injection vulnerabilities..."
sqlmap -u "$API_BASE/api/campaigns?id=1" \
  --headers="Authorization: Bearer test-token" \
  --batch \
  --output-dir="$RESULTS_DIR/sqlmap"

# 2. XSS Tests
echo "Testing XSS vulnerabilities..."
curl -X POST "$API_BASE/api/brands" \
  -H "Authorization: Bearer test-token" \
  -H "Content-Type: application/json" \
  -d '{"legalBusinessName":"<script>alert(1)</script>","businessType":"Corporation","businessAddress":"123 Test","contactPhone":"+12125551234","contactEmail":"test@test.com"}' \
  > "$RESULTS_DIR/xss-test.json"

# 3. Authentication Bypass Tests
echo "Testing authentication bypass..."
curl -X GET "$API_BASE/api/campaigns" \
  -H "Authorization: Bearer invalid-token" \
  > "$RESULTS_DIR/auth-bypass-test.json"

# 4. OWASP ZAP Scan
echo "Running OWASP ZAP scan..."
docker run -v $(pwd)/$RESULTS_DIR:/zap/wrk/:rw \
  -t owasp/zap2docker-stable zap-baseline.py \
  -t $API_BASE \
  -J zap-report.json

echo "Security tests completed. Results in $RESULTS_DIR"
```

## Compliance Testing

### TCPA Compliance Tests
```typescript
// tests/compliance/tcpa.test.ts
import { TCPAComplianceService } from '../../src/services/tcpa-compliance.service';

describe('TCPA Compliance Tests', () => {
  let complianceService: TCPAComplianceService;

  beforeEach(() => {
    complianceService = new TCPAComplianceService();
  });

  describe('Opt-in Validation', () => {
    test('should require valid opt-in before sending', async () => {
      const result = await complianceService.validateMessageSend({
        campaignId: 'campaign-123',
        from: '+12125551234',
        to: '+13125551234', // Number without opt-in
        text: 'Test message'
      });

      expect(result.compliant).toBe(false);
      expect(result.checks.find(c => c.type === 'opt_in')?.passed).toBe(false);
    });

    test('should allow messages to opted-in recipients', async () => {
      // Mock opt-in record
      jest.spyOn(complianceService, 'checkOptInStatus').mockResolvedValue({
        hasValidOptIn: true,
        optInDate: new Date(),
        optInMethod: 'web_form'
      });

      const result = await complianceService.validateMessageSend({
        campaignId: 'campaign-123',
        from: '+12125551234',
        to: '+13125551234',
        text: 'Test message'
      });

      expect(result.checks.find(c => c.type === 'opt_in')?.passed).toBe(true);
    });
  });

  describe('Time Restrictions', () => {
    test('should block messages outside allowed hours', async () => {
      // Mock 6 AM (before allowed time)
      const earlyMorning = new Date();
      earlyMorning.setHours(6, 0, 0, 0);

      const result = await complianceService.validateMessageSend({
        campaignId: 'campaign-123',
        from: '+12125551234',
        to: '+13125551234',
        text: 'Test message'
      }, earlyMorning);

      expect(result.checks.find(c => c.type === 'time_restriction')?.passed).toBe(false);
    });

    test('should allow messages during business hours', async () => {
      // Mock 2 PM (allowed time)
      const afternoon = new Date();
      afternoon.setHours(14, 0, 0, 0);

      const result = await complianceService.validateMessageSend({
        campaignId: 'campaign-123',
        from: '+12125551234',
        to: '+13125551234',
        text: 'Test message'
      }, afternoon);

      expect(result.checks.find(c => c.type === 'time_restriction')?.passed).toBe(true);
    });
  });

  describe('Content Validation', () => {
    test('should flag SHAFT content', async () => {
      const shaftMessage = 'Buy alcohol now! 50% off cigarettes!';

      const result = await complianceService.validateContent(shaftMessage);

      expect(result.isCompliant).toBe(false);
      expect(result.flags).toContain('alcohol');
      expect(result.flags).toContain('tobacco');
    });

    test('should approve compliant content', async () => {
      const compliantMessage = 'Your order has been shipped and will arrive tomorrow.';

      const result = await complianceService.validateContent(compliantMessage);

      expect(result.isCompliant).toBe(true);
      expect(result.flags).toHaveLength(0);
    });
  });
});
```

### GDPR Compliance Tests
```typescript
// tests/compliance/gdpr.test.ts
import { DataPrivacyService } from '../../src/services/data-privacy.service';

describe('GDPR Compliance Tests', () => {
  let privacyService: DataPrivacyService;

  beforeEach(() => {
    privacyService = new DataPrivacyService();
  });

  describe('Data Subject Rights', () => {
    test('should handle data access requests', async () => {
      const request = {
        type: 'access' as const,
        subjectId: 'user-123',
        email: 'user@example.com',
        verificationMethod: 'email'
      };

      const result = await privacyService.handleDataSubjectRequest(request);

      expect(result.status).toBe('completed');
      expect(result.data).toBeDefined();
      expect(result.data.personalData).toBeDefined();
    });

    test('should handle data deletion requests', async () => {
      const request = {
        type: 'deletion' as const,
        subjectId: 'user-123',
        email: 'user@example.com',
        verificationMethod: 'email'
      };

      const result = await privacyService.handleDataSubjectRequest(request);

      expect(result.status).toBe('completed');
      expect(result.deletionScheduled).toBe(true);
    });

    test('should handle data portability requests', async () => {
      const request = {
        type: 'portability' as const,
        subjectId: 'user-123',
        email: 'user@example.com',
        verificationMethod: 'email',
        format: 'json'
      };

      const result = await privacyService.handleDataSubjectRequest(request);

      expect(result.status).toBe('completed');
      expect(result.downloadUrl).toBeDefined();
      expect(result.format).toBe('json');
    });
  });

  describe('Data Retention', () => {
    test('should automatically purge expired data', async () => {
      const purgeResult = await privacyService.purgeExpiredData();

      expect(purgeResult.totalDeleted).toBeGreaterThanOrEqual(0);
      expect(purgeResult.results).toBeDefined();
    });

    test('should respect retention policies', async () => {
      const policies = await privacyService.getRetentionPolicies();

      for (const policy of policies) {
        expect(policy.retentionDays).toBeGreaterThan(0);
        expect(policy.table).toBeDefined();
        expect(policy.criteriaField).toBeDefined();
      }
    });
  });
});
```

## Test Automation

### CI/CD Test Integration
```yaml
# .github/workflows/test.yml
name: Test Suite

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

jobs:
  unit-tests:
    runs-on: ubuntu-latest
    
    services:
      postgres:
        image: postgres:13
        env:
          POSTGRES_PASSWORD: test
          POSTGRES_DB: test_sms_system
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5

    steps:
    - uses: actions/checkout@v3
    
    - name: Setup Node.js
      uses: actions/setup-node@v3
      with:
        node-version: '18'
        cache: 'npm'
    
    - name: Install dependencies
      run: npm ci
    
    - name: Run unit tests
      run: npm run test:unit
      env:
        DATABASE_URL: postgres://postgres:test@localhost:5432/test_sms_system
    
    - name: Upload coverage to Codecov
      uses: codecov/codecov-action@v3
      with:
        file: ./coverage/lcov.info

  integration-tests:
    runs-on: ubuntu-latest
    needs: unit-tests
    
    services:
      postgres:
        image: postgres:13
        env:
          POSTGRES_PASSWORD: test
          POSTGRES_DB: test_sms_system
      redis:
        image: redis:6
        
    steps:
    - uses: actions/checkout@v3
    
    - name: Setup Node.js
      uses: actions/setup-node@v3
      with:
        node-version: '18'
        cache: 'npm'
    
    - name: Install dependencies
      run: npm ci
    
    - name: Run integration tests
      run: npm run test:integration
      env:
        DATABASE_URL: postgres://postgres:test@localhost:5432/test_sms_system
        REDIS_URL: redis://localhost:6379

  e2e-tests:
    runs-on: ubuntu-latest
    needs: integration-tests
    
    steps:
    - uses: actions/checkout@v3
    
    - name: Setup Node.js
      uses: actions/setup-node@v3
      with:
        node-version: '18'
        cache: 'npm'
    
    - name: Install dependencies
      run: npm ci
    
    - name: Install Playwright
      run: npx playwright install --with-deps
    
    - name: Start application
      run: |
        npm run build
        npm run start:test &
        sleep 30
    
    - name: Run E2E tests
      run: npm run test:e2e
    
    - name: Upload E2E artifacts
      uses: actions/upload-artifact@v3
      if: failure()
      with:
        name: playwright-report
        path: playwright-report/

  performance-tests:
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main'
    
    steps:
    - uses: actions/checkout@v3
    
    - name: Setup K6
      run: |
        curl https://github.com/grafana/k6/releases/download/v0.46.0/k6-v0.46.0-linux-amd64.tar.gz -L | tar xvz --strip-components 1
    
    - name: Run performance tests
      run: ./k6 run tests/performance/load-test.js
      env:
        API_BASE_URL: https://staging-api.10dlc-sms.com

  security-tests:
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main'
    
    steps:
    - uses: actions/checkout@v3
    
    - name: Run security tests
      run: |
        docker run -v $(pwd):/zap/wrk/:rw \
          -t owasp/zap2docker-stable zap-baseline.py \
          -t https://staging-api.10dlc-sms.com \
          -J security-report.json
    
    - name: Upload security report
      uses: actions/upload-artifact@v3
      with:
        name: security-report
        path: security-report.json
```

### Test Data Management
```typescript
// tests/fixtures/test-data-manager.ts
export class TestDataManager {
  private static createdEntities: Map<string, string[]> = new Map();

  static async createTestUser(overrides: Partial<User> = {}): Promise<User> {
    const user = await userRepository.save({
      email: `test-${Date.now()}@example.com`,
      firstName: 'Test',
      lastName: 'User',
      passwordHash: await bcrypt.hash('test123', 10),
      isActive: true,
      ...overrides
    });

    this.trackEntity('users', user.id);
    return user;
  }

  static async createTestBrand(userId: string, overrides: Partial<Brand> = {}): Promise<Brand> {
    const brand = await brandRepository.save({
      legalBusinessName: `Test Brand ${Date.now()}`,
      businessType: 'Corporation',
      businessAddress: '123 Test St, Test City, TS 12345',
      contactPhone: '+12125551234',
      contactEmail: `brand-${Date.now()}@example.com`,
      status: 'approved',
      ...overrides
    });

    // Create user-brand association
    await userBrandRepository.save({
      userId,
      brandId: brand.id,
      role: 'owner'
    });

    this.trackEntity('brands', brand.id);
    return brand;
  }

  static async createTestCampaign(brandId: string, overrides: Partial<Campaign> = {}): Promise<Campaign> {
    const campaign = await campaignRepository.save({
      brandId,
      name: `Test Campaign ${Date.now()}`,
      useCase: 'customer_care',
      description: 'Test campaign for automated testing',
      sampleMessages: ['Hello, this is a test message'],
      optInProcess: 'Users opt in during account creation',
      status: 'approved',
      ...overrides
    });

    this.trackEntity('campaigns', campaign.id);
    return campaign;
  }

  private static trackEntity(type: string, id: string): void {
    if (!this.createdEntities.has(type)) {
      this.createdEntities.set(type, []);
    }
    this.createdEntities.get(type)!.push(id);
  }

  static async cleanupTestData(): Promise<void> {
    // Cleanup in reverse dependency order
    const cleanupOrder = ['messages', 'campaigns', 'user_brands', 'brands', 'users'];
    
    for (const entityType of cleanupOrder) {
      const ids = this.createdEntities.get(entityType) || [];
      if (ids.length > 0) {
        await this.deleteEntities(entityType, ids);
      }
    }

    this.createdEntities.clear();
  }

  private static async deleteEntities(type: string, ids: string[]): Promise<void> {
    const repository = getRepository(type);
    await repository.delete(ids);
  }
}
```

## Quality Gates

### Coverage Requirements
```javascript
// jest.config.js - Coverage thresholds
module.exports = {
  coverageThreshold: {
    global: {
      branches: 80,
      functions: 80,
      lines: 80,
      statements: 80
    },
    // Stricter requirements for critical components
    './src/services/message.service.ts': {
      branches: 90,
      functions: 90,
      lines: 90,
      statements: 90
    },
    './src/services/compliance.service.ts': {
      branches: 95,
      functions: 95,
      lines: 95,
      statements: 95
    }
  }
};
```

### Quality Gate Script
```bash
#!/bin/bash
# scripts/quality-gate.sh

set -e

echo "Running quality gate checks..."

# 1. Test coverage
echo "Checking test coverage..."
npm run test:coverage
COVERAGE_RESULT=$?

# 2. Code quality
echo "Running linting..."
npm run lint
LINT_RESULT=$?

# 3. Security audit
echo "Running security audit..."
npm audit --audit-level=high
AUDIT_RESULT=$?

# 4. Performance benchmarks
echo "Running performance benchmarks..."
npm run test:performance
PERFORMANCE_RESULT=$?

# 5. Integration tests
echo "Running integration tests..."
npm run test:integration
INTEGRATION_RESULT=$?

# Check all results
if [ $COVERAGE_RESULT -ne 0 ]; then
    echo "❌ Coverage threshold not met"
    exit 1
fi

if [ $LINT_RESULT -ne 0 ]; then
    echo "❌ Linting failed"
    exit 1
fi

if [ $AUDIT_RESULT -ne 0 ]; then
    echo "❌ Security vulnerabilities found"
    exit 1
fi

if [ $PERFORMANCE_RESULT -ne 0 ]; then
    echo "❌ Performance benchmarks failed"
    exit 1
fi

if [ $INTEGRATION_RESULT -ne 0 ]; then
    echo "❌ Integration tests failed"
    exit 1
fi

echo "✅ All quality gates passed!"
```

## Future Testing Enhancements

### AI-Powered Test Generation (Phase 1 - Q2 2024)
```typescript
// Future: AI test generation
interface AITestGenerator {
  generateUnitTests: (sourceCode: string) => Promise<TestSuite>;
  generateE2ETests: (userStories: string[]) => Promise<E2ETestSuite>;
  identifyTestGaps: (codebase: string, tests: TestSuite[]) => Promise<TestGap[]>;
}

// Example usage
const aiTestGen = new AITestGenerator();
const unitTests = await aiTestGen.generateUnitTests(sourceCode);
const e2eTests = await aiTestGen.generateE2ETests(userStories);
```

### Visual Regression Testing (Phase 2 - Q3 2024)
```typescript
// Future: Visual regression testing
test('dashboard should match visual baseline', async ({ page }) => {
  await page.goto('/dashboard');
  await expect(page).toHaveScreenshot('dashboard.png');
});

test('campaign form should be visually consistent', async ({ page }) => {
  await page.goto('/campaigns/new');
  await expect(page.locator('[data-testid=campaign-form]')).toHaveScreenshot('campaign-form.png');
});
```

### Chaos Engineering (Phase 3 - Q4 2024)
```typescript
// Future: Chaos engineering tests
describe('Chaos Engineering', () => {
  test('system should handle database failures', async () => {
    // Inject database failure
    await chaosMonkey.killDatabase();
    
    // Verify graceful degradation
    const response = await request(app)
      .get('/api/campaigns')
      .expect(503);
    
    expect(response.body.error).toContain('service temporarily unavailable');
    
    // Restore and verify recovery
    await chaosMonkey.restoreDatabase();
    await waitForHealthy();
    
    await request(app)
      .get('/api/campaigns')
      .expect(200);
  });
});
```

## Conclusion

This testing strategy provides:

- **Comprehensive Coverage**: Unit, integration, E2E, performance, and security testing
- **Quality Assurance**: Automated quality gates and coverage requirements
- **Compliance Validation**: TCPA and GDPR compliance testing
- **Performance Monitoring**: Load testing and benchmarking
- **Security Testing**: Vulnerability assessment and penetration testing
- **Future-Ready**: Roadmap for AI-powered testing and chaos engineering

The strategy ensures high-quality, reliable, and compliant software delivery through systematic testing approaches integrated into the development lifecycle.
