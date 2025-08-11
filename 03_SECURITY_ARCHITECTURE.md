# Security Architecture Document - 10DLC SMS System

## Table of Contents
1. [Overview](#overview)
2. [Security Framework](#security-framework)
3. [Authentication & Authorization](#authentication--authorization)
4. [Data Protection](#data-protection)
5. [Network Security](#network-security)
6. [Application Security](#application-security)
7. [Compliance & Regulatory](#compliance--regulatory)
8. [Incident Response](#incident-response)
9. [Security Monitoring](#security-monitoring)
10. [Future Security Enhancements](#future-security-enhancements)

## Overview

This document outlines the comprehensive security architecture for the 10DLC SMS messaging system, ensuring data protection, regulatory compliance, and secure operations across all system components.

### Security Objectives
- **Confidentiality**: Protect sensitive customer and business data
- **Integrity**: Ensure data accuracy and prevent unauthorized modifications
- **Availability**: Maintain system uptime and resilience against attacks
- **Compliance**: Meet 10DLC, TCPA, and industry security standards
- **Auditability**: Maintain comprehensive audit trails for all operations
- **Privacy**: Protect customer privacy and enable data subject rights

### Security Principles
- **Defense in Depth**: Multiple layers of security controls
- **Least Privilege**: Minimal access rights for users and systems
- **Zero Trust**: Verify everything, trust nothing
- **Encryption Everywhere**: Protect data at rest and in transit
- **Secure by Design**: Built-in security from the ground up
- **Continuous Monitoring**: Real-time threat detection and response

## Security Framework

### NIST Cybersecurity Framework Alignment
```
Identify → Protect → Detect → Respond → Recover
    ↓         ↓         ↓         ↓         ↓
Asset Mgmt  Access   Monitoring  Response  Recovery
Risk Mgmt   Training  Detection  Comms     Improve
Governance  Security  Analysis   Analysis  Lessons
```

### Security Architecture Layers
```
┌─────────────────────────────────────────────────────────────┐
│                    User Interface Layer                     │
│  • Web Application Security (HTTPS, CSP, HSTS)             │
│  • Client-side Input Validation                            │
│  • Session Management                                       │
└─────────────────────────────────────────────────────────────┘
                              │
┌─────────────────────────────────────────────────────────────┐
│                     API Gateway Layer                       │
│  • Rate Limiting & DDoS Protection                         │
│  • API Authentication & Authorization                       │
│  • Request Validation & Sanitization                       │
│  • WAF (Web Application Firewall)                          │
└─────────────────────────────────────────────────────────────┘
                              │
┌─────────────────────────────────────────────────────────────┐
│                   Application Layer                         │
│  • Business Logic Security                                 │
│  • Secure Coding Practices                                 │
│  • Input/Output Validation                                 │
│  • Error Handling & Logging                                │
└─────────────────────────────────────────────────────────────┘
                              │
┌─────────────────────────────────────────────────────────────┐
│                     Data Layer                              │
│  • Database Security (Encryption, Access Control)          │
│  • Data Classification & Handling                          │
│  • Backup Encryption                                       │
│  • Key Management                                          │
└─────────────────────────────────────────────────────────────┘
                              │
┌─────────────────────────────────────────────────────────────┐
│                Infrastructure Layer                         │
│  • Network Segmentation                                    │
│  • Firewall Rules                                          │
│  • VPC Security Groups                                     │
│  • Container Security                                      │
└─────────────────────────────────────────────────────────────┘
```

## Authentication & Authorization

### Multi-Factor Authentication (MFA)
```typescript
interface MFAConfig {
  enabled: boolean;
  methods: ('totp' | 'sms' | 'email' | 'hardware_key')[];
  backupCodes: boolean;
  gracePeriod: number; // hours
}

// Implementation
class MFAService {
  async enableMFA(userId: string, method: string): Promise<MFASetupResult> {
    const secret = this.generateTOTPSecret();
    const qrCode = this.generateQRCode(secret, userId);
    
    await this.storeMFASecret(userId, secret, method);
    
    return {
      secret,
      qrCode,
      backupCodes: this.generateBackupCodes()
    };
  }

  async verifyMFA(userId: string, token: string): Promise<boolean> {
    const secret = await this.getMFASecret(userId);
    return this.verifyTOTP(secret, token);
  }
}
```

### JWT Security Implementation
```typescript
interface JWTConfig {
  algorithm: 'RS256' | 'ES256';
  accessTokenExpiry: string;
  refreshTokenExpiry: string;
  issuer: string;
  audience: string[];
}

class JWTService {
  private privateKey: string;
  private publicKey: string;

  async generateTokens(user: User): Promise<AuthTokens> {
    const accessPayload = {
      sub: user.id,
      email: user.email,
      role: user.role,
      permissions: await this.getUserPermissions(user.id),
      iat: Math.floor(Date.now() / 1000),
      exp: Math.floor(Date.now() / 1000) + (15 * 60), // 15 minutes
      iss: this.config.issuer,
      aud: this.config.audience
    };

    const refreshPayload = {
      sub: user.id,
      type: 'refresh',
      iat: Math.floor(Date.now() / 1000),
      exp: Math.floor(Date.now() / 1000) + (7 * 24 * 60 * 60), // 7 days
      iss: this.config.issuer,
      aud: this.config.audience
    };

    const accessToken = jwt.sign(accessPayload, this.privateKey, { algorithm: 'RS256' });
    const refreshToken = jwt.sign(refreshPayload, this.privateKey, { algorithm: 'RS256' });

    // Store refresh token hash for validation
    await this.storeRefreshTokenHash(user.id, this.hashToken(refreshToken));

    return { accessToken, refreshToken };
  }

  async verifyToken(token: string): Promise<JWTPayload> {
    try {
      const payload = jwt.verify(token, this.publicKey, {
        algorithms: ['RS256'],
        issuer: this.config.issuer,
        audience: this.config.audience
      }) as JWTPayload;

      // Check if user is still active
      const user = await this.userService.findById(payload.sub);
      if (!user || !user.isActive) {
        throw new Error('User inactive');
      }

      return payload;
    } catch (error) {
      throw new Error('Invalid token');
    }
  }
}
```

### Role-Based Access Control (RBAC)
```typescript
interface Permission {
  resource: string;
  action: string;
  conditions?: Record<string, any>;
}

interface Role {
  id: string;
  name: string;
  permissions: Permission[];
  inherits?: string[]; // Role inheritance
}

class RBACService {
  private roles: Map<string, Role> = new Map();

  async checkPermission(
    userId: string, 
    resource: string, 
    action: string, 
    context?: Record<string, any>
  ): Promise<boolean> {
    const userRoles = await this.getUserRoles(userId);
    
    for (const role of userRoles) {
      const permissions = await this.getRolePermissions(role);
      
      for (const permission of permissions) {
        if (this.matchesPermission(permission, resource, action, context)) {
          return true;
        }
      }
    }
    
    return false;
  }

  private matchesPermission(
    permission: Permission, 
    resource: string, 
    action: string, 
    context?: Record<string, any>
  ): boolean {
    // Resource and action matching
    if (!this.matchesPattern(permission.resource, resource)) return false;
    if (!this.matchesPattern(permission.action, action)) return false;
    
    // Condition evaluation
    if (permission.conditions && context) {
      return this.evaluateConditions(permission.conditions, context);
    }
    
    return true;
  }
}
```

### API Key Security
```typescript
interface APIKeyConfig {
  keyLength: number;
  hashAlgorithm: string;
  rotationPeriod: number; // days
  maxKeys: number;
}

class APIKeyService {
  async createAPIKey(userId: string, permissions: Permission[]): Promise<APIKeyResult> {
    // Generate cryptographically secure key
    const keyValue = crypto.randomBytes(32).toString('base64url');
    const keyHash = await bcrypt.hash(keyValue, 12);
    
    const apiKey = await this.repository.save({
      id: uuidv4(),
      userId,
      keyHash,
      permissions,
      isActive: true,
      createdAt: new Date(),
      expiresAt: new Date(Date.now() + 365 * 24 * 60 * 60 * 1000) // 1 year
    });

    // Log key creation
    await this.auditService.log({
      userId,
      action: 'api_key_created',
      resource: apiKey.id,
      timestamp: new Date()
    });

    return {
      id: apiKey.id,
      key: keyValue, // Only returned once
      permissions,
      expiresAt: apiKey.expiresAt
    };
  }

  async validateAPIKey(keyValue: string): Promise<User | null> {
    // Rate limiting for API key validation
    await this.rateLimiter.check(`api_key_validation:${keyValue.substring(0, 8)}`);

    // Find key by partial hash match (timing-safe)
    const keys = await this.repository.findActiveKeys();
    
    for (const key of keys) {
      if (await bcrypt.compare(keyValue, key.keyHash)) {
        // Update last used timestamp
        await this.repository.updateLastUsed(key.id);
        
        return await this.userService.findById(key.userId);
      }
    }
    
    return null;
  }
}
```

## Data Protection

### Encryption at Rest
```typescript
interface EncryptionConfig {
  algorithm: 'AES-256-GCM';
  keyRotationPeriod: number; // days
  keyDerivation: 'PBKDF2' | 'Argon2';
}

class EncryptionService {
  private masterKey: Buffer;
  private keyCache: Map<string, Buffer> = new Map();

  async encryptSensitiveData(data: string, keyId?: string): Promise<EncryptedData> {
    const dataKey = keyId ? await this.getDataKey(keyId) : await this.generateDataKey();
    const iv = crypto.randomBytes(16);
    
    const cipher = crypto.createCipher('aes-256-gcm', dataKey);
    cipher.setAAD(Buffer.from('10dlc-sms-system'));
    
    let encrypted = cipher.update(data, 'utf8', 'hex');
    encrypted += cipher.final('hex');
    
    const authTag = cipher.getAuthTag();
    
    return {
      data: encrypted,
      iv: iv.toString('hex'),
      authTag: authTag.toString('hex'),
      keyId: dataKey.keyId,
      algorithm: 'AES-256-GCM'
    };
  }

  async decryptSensitiveData(encryptedData: EncryptedData): Promise<string> {
    const dataKey = await this.getDataKey(encryptedData.keyId);
    const iv = Buffer.from(encryptedData.iv, 'hex');
    const authTag = Buffer.from(encryptedData.authTag, 'hex');
    
    const decipher = crypto.createDecipher('aes-256-gcm', dataKey);
    decipher.setAAD(Buffer.from('10dlc-sms-system'));
    decipher.setAuthTag(authTag);
    
    let decrypted = decipher.update(encryptedData.data, 'hex', 'utf8');
    decrypted += decipher.final('utf8');
    
    return decrypted;
  }
}
```

### Database Encryption
```sql
-- Enable MySQL transparent data encryption
ALTER INSTANCE ROTATE INNODB MASTER KEY;

-- Create encrypted tablespace
CREATE TABLESPACE encrypted_space 
ADD DATAFILE 'encrypted_space.ibd' 
ENCRYPTION='Y';

-- Encrypt sensitive tables
ALTER TABLE users ENCRYPTION='Y';
ALTER TABLE messages ENCRYPTION='Y';
ALTER TABLE api_keys ENCRYPTION='Y';

-- Column-level encryption for PII
ALTER TABLE users 
ADD COLUMN phone_encrypted VARBINARY(256),
ADD COLUMN email_encrypted VARBINARY(256);
```

### PII Data Handling
```typescript
interface PIIClassification {
  type: 'phone' | 'email' | 'name' | 'address';
  sensitivity: 'high' | 'medium' | 'low';
  retention: number; // days
  encryption: boolean;
  masking: boolean;
}

class PIIService {
  private classifications: Map<string, PIIClassification> = new Map([
    ['phoneNumber', { type: 'phone', sensitivity: 'high', retention: 365, encryption: true, masking: true }],
    ['email', { type: 'email', sensitivity: 'medium', retention: 730, encryption: true, masking: true }],
    ['firstName', { type: 'name', sensitivity: 'medium', retention: 730, encryption: false, masking: true }]
  ]);

  async maskPII(data: any, context: string): Promise<any> {
    const masked = { ...data };
    
    for (const [field, classification] of this.classifications) {
      if (masked[field] && classification.masking) {
        masked[field] = await this.maskField(masked[field], classification.type);
      }
    }
    
    return masked;
  }

  private async maskField(value: string, type: string): Promise<string> {
    switch (type) {
      case 'phone':
        return value.replace(/(\+\d{1,3})\d{6,10}(\d{4})/, '$1******$2');
      case 'email':
        const [local, domain] = value.split('@');
        return `${local.substring(0, 2)}***@${domain}`;
      case 'name':
        return value.replace(/(.{2}).*/, '$1***');
      default:
        return '***';
    }
  }
}
```

### Data Loss Prevention (DLP)
```typescript
class DLPService {
  private patterns: Map<string, RegExp> = new Map([
    ['ssn', /\b\d{3}-?\d{2}-?\d{4}\b/g],
    ['credit_card', /\b\d{4}[-\s]?\d{4}[-\s]?\d{4}[-\s]?\d{4}\b/g],
    ['phone', /\b\+?1?[-.\s]?\(?[0-9]{3}\)?[-.\s]?[0-9]{3}[-.\s]?[0-9]{4}\b/g]
  ]);

  async scanContent(content: string): Promise<DLPResult> {
    const findings: DLPFinding[] = [];
    
    for (const [type, pattern] of this.patterns) {
      const matches = content.match(pattern);
      if (matches) {
        findings.push({
          type,
          count: matches.length,
          positions: this.getMatchPositions(content, pattern),
          severity: this.getSeverity(type)
        });
      }
    }
    
    return {
      hasSensitiveData: findings.length > 0,
      findings,
      riskScore: this.calculateRiskScore(findings)
    };
  }

  async sanitizeContent(content: string): Promise<string> {
    let sanitized = content;
    
    for (const [type, pattern] of this.patterns) {
      sanitized = sanitized.replace(pattern, '[REDACTED]');
    }
    
    return sanitized;
  }
}
```

## Network Security

### VPC and Network Segmentation
```yaml
# AWS VPC Configuration
Resources:
  VPC:
    Type: AWS::EC2::VPC
    Properties:
      CidrBlock: 10.0.0.0/16
      EnableDnsHostnames: true
      EnableDnsSupport: true
      
  # Public Subnets (Load Balancers)
  PublicSubnet1:
    Type: AWS::EC2::Subnet
    Properties:
      VpcId: !Ref VPC
      CidrBlock: 10.0.1.0/24
      AvailabilityZone: !Select [0, !GetAZs '']
      
  # Private Subnets (Application)
  PrivateSubnet1:
    Type: AWS::EC2::Subnet
    Properties:
      VpcId: !Ref VPC
      CidrBlock: 10.0.10.0/24
      AvailabilityZone: !Select [0, !GetAZs '']
      
  # Database Subnets (Isolated)
  DatabaseSubnet1:
    Type: AWS::EC2::Subnet
    Properties:
      VpcId: !Ref VPC
      CidrBlock: 10.0.20.0/24
      AvailabilityZone: !Select [0, !GetAZs '']
```

### Security Groups and NACLs
```yaml
# Application Security Group
AppSecurityGroup:
  Type: AWS::EC2::SecurityGroup
  Properties:
    GroupDescription: Application tier security group
    VpcId: !Ref VPC
    SecurityGroupIngress:
      - IpProtocol: tcp
        FromPort: 443
        ToPort: 443
        SourceSecurityGroupId: !Ref LoadBalancerSecurityGroup
      - IpProtocol: tcp
        FromPort: 3000
        ToPort: 3000
        SourceSecurityGroupId: !Ref LoadBalancerSecurityGroup
    SecurityGroupEgress:
      - IpProtocol: tcp
        FromPort: 3306
        ToPort: 3306
        DestinationSecurityGroupId: !Ref DatabaseSecurityGroup
      - IpProtocol: tcp
        FromPort: 443
        ToPort: 443
        CidrIp: 0.0.0.0/0  # For external API calls
        
# Database Security Group
DatabaseSecurityGroup:
  Type: AWS::EC2::SecurityGroup
  Properties:
    GroupDescription: Database tier security group
    VpcId: !Ref VPC
    SecurityGroupIngress:
      - IpProtocol: tcp
        FromPort: 3306
        ToPort: 3306
        SourceSecurityGroupId: !Ref AppSecurityGroup
```

### TLS/SSL Configuration
```nginx
# Nginx SSL Configuration
server {
    listen 443 ssl http2;
    server_name api.yourdomain.com;
    
    # SSL Certificates
    ssl_certificate /etc/ssl/certs/api.yourdomain.com.pem;
    ssl_certificate_key /etc/ssl/private/api.yourdomain.com.key;
    
    # SSL Security
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers ECDHE-RSA-AES256-GCM-SHA512:DHE-RSA-AES256-GCM-SHA512:ECDHE-RSA-AES256-GCM-SHA384;
    ssl_prefer_server_ciphers off;
    ssl_session_cache shared:SSL:10m;
    ssl_session_timeout 10m;
    
    # HSTS
    add_header Strict-Transport-Security "max-age=31536000; includeSubDomains; preload" always;
    
    # OCSP Stapling
    ssl_stapling on;
    ssl_stapling_verify on;
    
    # Security Headers
    add_header X-Frame-Options DENY always;
    add_header X-Content-Type-Options nosniff always;
    add_header X-XSS-Protection "1; mode=block" always;
    add_header Referrer-Policy "strict-origin-when-cross-origin" always;
    add_header Content-Security-Policy "default-src 'self'; script-src 'self' 'unsafe-inline'; style-src 'self' 'unsafe-inline'" always;
}
```

### DDoS Protection and Rate Limiting
```typescript
class DDoSProtection {
  private rateLimiter: RateLimiter;
  private suspiciousIPs: Set<string> = new Set();

  async checkRequest(ip: string, endpoint: string): Promise<boolean> {
    // Rate limiting by IP
    const ipKey = `rate:ip:${ip}`;
    const ipCount = await this.rateLimiter.get(ipKey);
    if (ipCount > 1000) { // 1000 requests per hour
      await this.blockIP(ip, 'rate_limit_exceeded');
      return false;
    }

    // Rate limiting by endpoint
    const endpointKey = `rate:endpoint:${endpoint}:${ip}`;
    const endpointCount = await this.rateLimiter.get(endpointKey);
    if (endpointCount > 100) { // 100 requests per endpoint per hour
      return false;
    }

    // Suspicious pattern detection
    if (await this.detectSuspiciousActivity(ip)) {
      this.suspiciousIPs.add(ip);
      await this.alertSecurityTeam(ip);
    }

    return true;
  }

  private async detectSuspiciousActivity(ip: string): Promise<boolean> {
    const patterns = await this.getRequestPatterns(ip);
    
    // Check for bot-like behavior
    if (patterns.requestsPerSecond > 10) return true;
    if (patterns.uniqueEndpoints < 3 && patterns.totalRequests > 100) return true;
    if (patterns.errorRate > 0.5) return true;
    
    return false;
  }
}
```

## Application Security

### Input Validation and Sanitization
```typescript
class InputValidator {
  private schemas: Map<string, Joi.ObjectSchema> = new Map();

  constructor() {
    this.initializeSchemas();
  }

  private initializeSchemas(): void {
    // Phone number validation
    this.schemas.set('phoneNumber', Joi.string()
      .pattern(/^\+[1-9]\d{1,14}$/)
      .required()
      .error(new Error('Phone number must be in E.164 format'))
    );

    // Message content validation
    this.schemas.set('messageContent', Joi.string()
      .max(1600)
      .custom((value, helpers) => {
        // Check for malicious content
        if (this.containsMaliciousContent(value)) {
          return helpers.error('any.malicious');
        }
        return value;
      })
      .required()
    );
  }

  async validateInput(data: any, schemaName: string): Promise<ValidationResult> {
    const schema = this.schemas.get(schemaName);
    if (!schema) {
      throw new Error(`Schema ${schemaName} not found`);
    }

    try {
      const validated = await schema.validateAsync(data, { abortEarly: false });
      return { isValid: true, data: validated };
    } catch (error) {
      return { 
        isValid: false, 
        errors: error.details.map((d: any) => d.message) 
      };
    }
  }

  private containsMaliciousContent(content: string): boolean {
    const maliciousPatterns = [
      /<script/i,
      /javascript:/i,
      /on\w+\s*=/i,
      /eval\s*\(/i,
      /expression\s*\(/i
    ];

    return maliciousPatterns.some(pattern => pattern.test(content));
  }
}
```

### SQL Injection Prevention
```typescript
class DatabaseSecurityService {
  async executeQuery(query: string, params: any[]): Promise<any> {
    // Validate query doesn't contain dynamic SQL
    if (this.containsDynamicSQL(query)) {
      throw new Error('Dynamic SQL not allowed');
    }

    // Use parameterized queries
    return await this.connection.execute(query, params);
  }

  private containsDynamicSQL(query: string): boolean {
    const dangerousPatterns = [
      /UNION\s+SELECT/i,
      /DROP\s+TABLE/i,
      /DELETE\s+FROM/i,
      /INSERT\s+INTO/i,
      /UPDATE\s+\w+\s+SET/i
    ];

    return dangerousPatterns.some(pattern => pattern.test(query));
  }
}

// Example safe query usage
class MessageRepository {
  async getMessagesByCampaign(campaignId: string, limit: number): Promise<Message[]> {
    const query = `
      SELECT m.*, c.name as campaign_name 
      FROM messages m 
      JOIN campaigns c ON m.campaign_id = c.id 
      WHERE m.campaign_id = ? 
      ORDER BY m.created_at DESC 
      LIMIT ?
    `;
    
    return await this.db.executeQuery(query, [campaignId, limit]);
  }
}
```

### XSS Prevention
```typescript
class XSSProtection {
  static sanitizeHTML(html: string): string {
    return DOMPurify.sanitize(html, {
      ALLOWED_TAGS: ['b', 'i', 'em', 'strong', 'a'],
      ALLOWED_ATTR: ['href', 'title'],
      ALLOW_DATA_ATTR: false
    });
  }

  static escapeHTML(text: string): string {
    const map: { [key: string]: string } = {
      '&': '&amp;',
      '<': '&lt;',
      '>': '&gt;',
      '"': '&quot;',
      "'": '&#x27;',
      '/': '&#x2F;'
    };

    return text.replace(/[&<>"'/]/g, (match) => map[match]);
  }

  static setSecurityHeaders(res: Response): void {
    res.setHeader('Content-Security-Policy', 
      "default-src 'self'; " +
      "script-src 'self' 'unsafe-inline'; " +
      "style-src 'self' 'unsafe-inline'; " +
      "img-src 'self' data: https:; " +
      "connect-src 'self'; " +
      "font-src 'self'; " +
      "object-src 'none'; " +
      "media-src 'self'; " +
      "frame-src 'none';"
    );
    
    res.setHeader('X-XSS-Protection', '1; mode=block');
    res.setHeader('X-Content-Type-Options', 'nosniff');
    res.setHeader('X-Frame-Options', 'DENY');
    res.setHeader('Referrer-Policy', 'strict-origin-when-cross-origin');
  }
}
```

### Session Security
```typescript
interface SessionConfig {
  name: string;
  secret: string;
  resave: boolean;
  saveUninitialized: boolean;
  rolling: boolean;
  cookie: {
    secure: boolean;
    httpOnly: boolean;
    maxAge: number;
    sameSite: 'strict' | 'lax' | 'none';
  };
}

class SessionManager {
  private config: SessionConfig = {
    name: 'sessionId',
    secret: process.env.SESSION_SECRET!,
    resave: false,
    saveUninitialized: false,
    rolling: true,
    cookie: {
      secure: process.env.NODE_ENV === 'production',
      httpOnly: true,
      maxAge: 24 * 60 * 60 * 1000, // 24 hours
      sameSite: 'strict'
    }
  };

  async createSession(userId: string, deviceInfo: DeviceInfo): Promise<string> {
    const sessionId = crypto.randomBytes(32).toString('hex');
    const expiresAt = new Date(Date.now() + this.config.cookie.maxAge);

    await this.storeSession({
      id: sessionId,
      userId,
      deviceInfo,
      createdAt: new Date(),
      expiresAt,
      isActive: true
    });

    return sessionId;
  }

  async validateSession(sessionId: string): Promise<SessionData | null> {
    const session = await this.getSession(sessionId);
    
    if (!session || !session.isActive || session.expiresAt < new Date()) {
      await this.invalidateSession(sessionId);
      return null;
    }

    // Update last accessed time
    await this.updateSessionAccess(sessionId);
    
    return session;
  }

  async invalidateAllUserSessions(userId: string): Promise<void> {
    await this.repository.update(
      { userId, isActive: true },
      { isActive: false, invalidatedAt: new Date() }
    );
  }
}
```

## Compliance & Regulatory

### TCPA Compliance
```typescript
class TCPACompliance {
  async validateMessageSend(request: SendMessageRequest): Promise<ComplianceResult> {
    const checks: ComplianceCheck[] = [];

    // Check opt-in status
    const optInStatus = await this.checkOptInStatus(request.to, request.campaignId);
    checks.push({
      type: 'opt_in',
      passed: optInStatus.hasValidOptIn,
      details: optInStatus
    });

    // Check time restrictions
    const timeCheck = await this.checkTimeRestrictions(request.to, new Date());
    checks.push({
      type: 'time_restriction',
      passed: timeCheck.isAllowed,
      details: timeCheck
    });

    // Check content compliance
    const contentCheck = await this.validateContent(request.text);
    checks.push({
      type: 'content',
      passed: contentCheck.isCompliant,
      details: contentCheck
    });

    const allPassed = checks.every(check => check.passed);

    return {
      compliant: allPassed,
      checks,
      requiresHumanReview: this.requiresReview(checks)
    };
  }

  private async checkTimeRestrictions(phoneNumber: string, timestamp: Date): Promise<TimeRestrictionResult> {
    // Get timezone for phone number
    const timezone = await this.getPhoneTimezone(phoneNumber);
    const localTime = this.convertToTimezone(timestamp, timezone);
    
    // TCPA allows calls/texts 8 AM to 9 PM local time
    const hour = localTime.getHours();
    const isAllowed = hour >= 8 && hour < 21;
    
    return {
      isAllowed,
      localTime,
      timezone,
      hour
    };
  }
}
```

### Data Privacy (GDPR/CCPA)
```typescript
class DataPrivacyService {
  async handleDataSubjectRequest(request: DataSubjectRequest): Promise<DataSubjectResponse> {
    switch (request.type) {
      case 'access':
        return await this.handleAccessRequest(request);
      case 'deletion':
        return await this.handleDeletionRequest(request);
      case 'portability':
        return await this.handlePortabilityRequest(request);
      case 'rectification':
        return await this.handleRectificationRequest(request);
      default:
        throw new Error(`Unsupported request type: ${request.type}`);
    }
  }

  private async handleDeletionRequest(request: DataSubjectRequest): Promise<DataSubjectResponse> {
    const subject = await this.verifyDataSubject(request);
    
    // Soft delete user data
    await this.softDeleteUserData(subject.userId);
    
    // Schedule hard deletion after retention period
    await this.scheduleHardDeletion(subject.userId, 30); // 30 days
    
    // Log the deletion request
    await this.auditService.log({
      type: 'data_deletion',
      subject: subject.userId,
      requestId: request.id,
      timestamp: new Date()
    });

    return {
      requestId: request.id,
      status: 'completed',
      completedAt: new Date(),
      details: 'Personal data marked for deletion'
    };
  }

  async purgeExpiredData(): Promise<PurgeResult> {
    const retentionPolicies = await this.getRetentionPolicies();
    const purgeResults: PurgeResult[] = [];

    for (const policy of retentionPolicies) {
      const expiredRecords = await this.findExpiredRecords(policy);
      
      for (const record of expiredRecords) {
        await this.hardDeleteRecord(record);
        purgeResults.push({
          table: policy.table,
          recordId: record.id,
          deletedAt: new Date()
        });
      }
    }

    return {
      totalDeleted: purgeResults.length,
      results: purgeResults
    };
  }
}
```

### Audit Trail Implementation
```typescript
interface AuditEvent {
  id: string;
  userId?: string;
  action: string;
  resource: string;
  resourceId?: string;
  metadata?: Record<string, any>;
  ipAddress?: string;
  userAgent?: string;
  timestamp: Date;
  result: 'success' | 'failure';
  errorMessage?: string;
}

class AuditService {
  async log(event: Omit<AuditEvent, 'id' | 'timestamp'>): Promise<void> {
    const auditEvent: AuditEvent = {
      id: uuidv4(),
      timestamp: new Date(),
      ...event
    };

    // Store in audit log
    await this.repository.save(auditEvent);

    // Send to SIEM if critical event
    if (this.isCriticalEvent(event.action)) {
      await this.sendToSIEM(auditEvent);
    }
  }

  async searchAuditLog(criteria: AuditSearchCriteria): Promise<AuditEvent[]> {
    const queryBuilder = this.repository.createQueryBuilder('audit');

    if (criteria.userId) {
      queryBuilder.andWhere('audit.userId = :userId', { userId: criteria.userId });
    }

    if (criteria.action) {
      queryBuilder.andWhere('audit.action = :action', { action: criteria.action });
    }

    if (criteria.dateRange) {
      queryBuilder.andWhere('audit.timestamp BETWEEN :start AND :end', {
        start: criteria.dateRange.start,
        end: criteria.dateRange.end
      });
    }

    return await queryBuilder
      .orderBy('audit.timestamp', 'DESC')
      .limit(criteria.limit || 1000)
      .getMany();
  }

  private isCriticalEvent(action: string): boolean {
    const criticalActions = [
      'user_created',
      'user_deleted',
      'permission_changed',
      'api_key_created',
      'brand_approved',
      'campaign_approved'
    ];

    return criticalActions.includes(action);
  }
}
```

## Incident Response

### Security Incident Classification
```typescript
enum IncidentSeverity {
  LOW = 'low',
  MEDIUM = 'medium',
  HIGH = 'high',
  CRITICAL = 'critical'
}

enum IncidentType {
  DATA_BREACH = 'data_breach',
  UNAUTHORIZED_ACCESS = 'unauthorized_access',
  MALWARE = 'malware',
  DDOS = 'ddos',
  INSIDER_THREAT = 'insider_threat',
  COMPLIANCE_VIOLATION = 'compliance_violation'
}

interface SecurityIncident {
  id: string;
  type: IncidentType;
  severity: IncidentSeverity;
  title: string;
  description: string;
  affectedSystems: string[];
  affectedUsers?: string[];
  detectedAt: Date;
  reportedBy: string;
  assignedTo?: string;
  status: 'open' | 'investigating' | 'contained' | 'resolved' | 'closed';
  timeline: IncidentTimelineEntry[];
}

class IncidentResponseService {
  async createIncident(incident: Omit<SecurityIncident, 'id' | 'detectedAt' | 'timeline'>): Promise<SecurityIncident> {
    const newIncident: SecurityIncident = {
      id: `INC-${Date.now()}`,
      detectedAt: new Date(),
      timeline: [],
      ...incident
    };

    // Auto-assign based on severity
    if (incident.severity === IncidentSeverity.CRITICAL) {
      newIncident.assignedTo = await this.getOnCallSecurityEngineer();
      await this.triggerEmergencyResponse(newIncident);
    }

    await this.repository.save(newIncident);
    await this.notifyStakeholders(newIncident);

    return newIncident;
  }

  async containThreat(incidentId: string, containmentActions: ContainmentAction[]): Promise<void> {
    const incident = await this.getIncident(incidentId);

    for (const action of containmentActions) {
      switch (action.type) {
        case 'block_ip':
          await this.blockIP(action.target);
          break;
        case 'disable_user':
          await this.disableUser(action.target);
          break;
        case 'isolate_system':
          await this.isolateSystem(action.target);
          break;
        case 'rotate_keys':
          await this.rotateKeys(action.target);
          break;
      }

      await this.addTimelineEntry(incidentId, {
        action: `Containment: ${action.type}`,
        description: action.description,
        performedBy: action.performedBy,
        timestamp: new Date()
      });
    }

    await this.updateIncidentStatus(incidentId, 'contained');
  }
}
```

### Automated Response Actions
```typescript
class AutomatedResponse {
  private responseRules: Map<string, ResponseRule> = new Map();

  constructor() {
    this.initializeRules();
  }

  private initializeRules(): void {
    // Brute force attack response
    this.responseRules.set('brute_force_attack', {
      trigger: {
        type: 'failed_login_attempts',
        threshold: 5,
        timeWindow: 300 // 5 minutes
      },
      actions: [
        { type: 'block_ip', duration: 3600 }, // 1 hour
        { type: 'alert_security_team' },
        { type: 'log_incident' }
      ]
    });

    // Suspicious API usage
    this.responseRules.set('api_abuse', {
      trigger: {
        type: 'api_rate_exceeded',
        threshold: 1000,
        timeWindow: 60 // 1 minute
      },
      actions: [
        { type: 'throttle_api_key' },
        { type: 'alert_user' },
        { type: 'require_mfa' }
      ]
    });
  }

  async evaluateEvent(event: SecurityEvent): Promise<void> {
    for (const [ruleName, rule] of this.responseRules) {
      if (await this.ruleMatches(event, rule)) {
        await this.executeResponseActions(rule.actions, event);
        
        await this.auditService.log({
          action: 'automated_response_triggered',
          resource: ruleName,
          metadata: { event, rule },
          result: 'success'
        });
      }
    }
  }

  private async executeResponseActions(actions: ResponseAction[], event: SecurityEvent): Promise<void> {
    for (const action of actions) {
      try {
        switch (action.type) {
          case 'block_ip':
            await this.firewallService.blockIP(event.sourceIP, action.duration);
            break;
          case 'disable_user':
            await this.userService.disableUser(event.userId, 'security_incident');
            break;
          case 'alert_security_team':
            await this.notificationService.alertSecurityTeam(event);
            break;
          case 'rotate_keys':
            await this.keyRotationService.rotateUserKeys(event.userId);
            break;
        }
      } catch (error) {
        logger.error(`Failed to execute response action ${action.type}`, error);
      }
    }
  }
}
```

## Security Monitoring

### Real-time Threat Detection
```typescript
class ThreatDetectionService {
  private readonly anomalyThresholds = {
    loginAttempts: 5,
    apiCalls: 1000,
    messageVolume: 10000,
    errorRate: 0.1
  };

  async analyzeUserBehavior(userId: string): Promise<ThreatAssessment> {
    const behavior = await this.getUserBehavior(userId);
    const baseline = await this.getUserBaseline(userId);
    
    const anomalies: Anomaly[] = [];

    // Check for unusual login patterns
    if (this.isAnomalousLoginPattern(behavior.logins, baseline.logins)) {
      anomalies.push({
        type: 'unusual_login_pattern',
        severity: 'medium',
        details: 'Login from unusual location or time'
      });
    }

    // Check for excessive API usage
    if (behavior.apiCalls > baseline.apiCalls * 3) {
      anomalies.push({
        type: 'excessive_api_usage',
        severity: 'high',
        details: `API calls: ${behavior.apiCalls} vs baseline: ${baseline.apiCalls}`
      });
    }

    // Check for unusual message patterns
    if (this.isAnomalousMessagePattern(behavior.messages, baseline.messages)) {
      anomalies.push({
        type: 'unusual_message_pattern',
        severity: 'high',
        details: 'Unusual message volume or recipients'
      });
    }

    return {
      userId,
      riskScore: this.calculateRiskScore(anomalies),
      anomalies,
      timestamp: new Date()
    };
  }

  private calculateRiskScore(anomalies: Anomaly[]): number {
    const weights = {
      low: 1,
      medium: 3,
      high: 5,
      critical: 10
    };

    return anomalies.reduce((score, anomaly) => {
      return score + weights[anomaly.severity];
    }, 0);
  }
}
```

### Security Metrics and KPIs
```typescript
interface SecurityMetrics {
  authenticationMetrics: {
    successfulLogins: number;
    failedLogins: number;
    mfaAdoptions: number;
    passwordResets: number;
  };
  accessMetrics: {
    privilegedAccess: number;
    failedAuthorizations: number;
    dormantAccounts: number;
  };
  incidentMetrics: {
    totalIncidents: number;
    criticalIncidents: number;
    meanTimeToDetection: number;
    meanTimeToResolution: number;
  };
  complianceMetrics: {
    dataSubjectRequests: number;
    retentionViolations: number;
    auditFindings: number;
  };
}

class SecurityMetricsService {
  async generateMetrics(period: DateRange): Promise<SecurityMetrics> {
    const metrics: SecurityMetrics = {
      authenticationMetrics: await this.getAuthenticationMetrics(period),
      accessMetrics: await this.getAccessMetrics(period),
      incidentMetrics: await this.getIncidentMetrics(period),
      complianceMetrics: await this.getComplianceMetrics(period)
    };

    // Calculate derived metrics
    metrics.authenticationMetrics['failureRate'] = 
      metrics.authenticationMetrics.failedLogins / 
      (metrics.authenticationMetrics.successfulLogins + metrics.authenticationMetrics.failedLogins);

    return metrics;
  }

  async generateSecurityDashboard(): Promise<SecurityDashboard> {
    const now = new Date();
    const last24h = { start: new Date(now.getTime() - 24 * 60 * 60 * 1000), end: now };
    const last7d = { start: new Date(now.getTime() - 7 * 24 * 60 * 60 * 1000), end: now };

    return {
      realTimeAlerts: await this.getActiveAlerts(),
      metrics24h: await this.generateMetrics(last24h),
      metrics7d: await this.generateMetrics(last7d),
      topThreats: await this.getTopThreats(),
      systemHealth: await this.getSystemHealthStatus()
    };
  }
}
```

## Future Security Enhancements

### Zero Trust Architecture (Phase 1 - Q2 2024)
```typescript
interface ZeroTrustPolicy {
  id: string;
  name: string;
  conditions: ZeroTrustCondition[];
  actions: ZeroTrustAction[];
  priority: number;
}

class ZeroTrustEngine {
  async evaluateAccess(request: AccessRequest): Promise<AccessDecision> {
    const context = await this.buildContext(request);
    const policies = await this.getApplicablePolicies(request);
    
    for (const policy of policies.sort((a, b) => a.priority - b.priority)) {
      if (await this.evaluateConditions(policy.conditions, context)) {
        return await this.executeActions(policy.actions, context);
      }
    }
    
    // Default deny
    return { allowed: false, reason: 'No matching policy' };
  }

  private async buildContext(request: AccessRequest): Promise<ZeroTrustContext> {
    return {
      user: await this.getUserContext(request.userId),
      device: await this.getDeviceContext(request.deviceId),
      network: await this.getNetworkContext(request.sourceIP),
      resource: await this.getResourceContext(request.resource),
      time: new Date(),
      riskScore: await this.calculateRiskScore(request)
    };
  }
}
```

### AI-Powered Security (Phase 2 - Q3 2024)
```typescript
class AISecurityService {
  private mlModel: SecurityMLModel;

  async detectAnomalies(events: SecurityEvent[]): Promise<AnomalyDetection[]> {
    const features = await this.extractFeatures(events);
    const predictions = await this.mlModel.predict(features);
    
    return predictions.map((prediction, index) => ({
      event: events[index],
      anomalyScore: prediction.score,
      confidence: prediction.confidence,
      explanation: prediction.explanation
    }));
  }

  async classifyThreat(event: SecurityEvent): Promise<ThreatClassification> {
    const features = await this.extractThreatFeatures(event);
    const classification = await this.mlModel.classify(features);
    
    return {
      threatType: classification.type,
      severity: classification.severity,
      confidence: classification.confidence,
      recommendedActions: classification.actions
    };
  }

  async predictSecurityRisk(userId: string): Promise<RiskPrediction> {
    const userHistory = await this.getUserSecurityHistory(userId);
    const behaviorProfile = await this.buildBehaviorProfile(userHistory);
    
    const riskScore = await this.mlModel.predictRisk(behaviorProfile);
    
    return {
      userId,
      riskScore,
      riskFactors: riskScore.factors,
      recommendations: riskScore.recommendations,
      predictedAt: new Date()
    };
  }
}
```

### Blockchain-based Audit Trail (Phase 3 - Q4 2024)
```typescript
class BlockchainAuditService {
  private blockchain: AuditBlockchain;

  async recordAuditEvent(event: AuditEvent): Promise<BlockchainRecord> {
    const block = await this.createAuditBlock(event);
    const hash = await this.calculateBlockHash(block);
    
    await this.blockchain.addBlock({
      ...block,
      hash,
      previousHash: await this.getLastBlockHash(),
      timestamp: new Date()
    });

    return {
      blockId: block.id,
      hash,
      immutable: true
    };
  }

  async verifyAuditIntegrity(fromDate: Date, toDate: Date): Promise<IntegrityVerification> {
    const blocks = await this.blockchain.getBlocks(fromDate, toDate);
    
    for (let i = 1; i < blocks.length; i++) {
      const currentBlock = blocks[i];
      const previousBlock = blocks[i - 1];
      
      // Verify hash chain
      if (currentBlock.previousHash !== previousBlock.hash) {
        return {
          isValid: false,
          tamperDetected: true,
          tamperedBlock: currentBlock.id
        };
      }
      
      // Verify block hash
      const calculatedHash = await this.calculateBlockHash(currentBlock);
      if (calculatedHash !== currentBlock.hash) {
        return {
          isValid: false,
          tamperDetected: true,
          tamperedBlock: currentBlock.id
        };
      }
    }
    
    return { isValid: true, tamperDetected: false };
  }
}
```

## Conclusion

This security architecture provides:

- **Comprehensive Protection**: Multi-layered security across all system components
- **Regulatory Compliance**: Built-in support for TCPA, GDPR, and industry standards
- **Threat Detection**: Real-time monitoring and automated response capabilities
- **Incident Response**: Structured approach to security incident management
- **Future-Ready**: Extensible architecture supporting emerging security technologies

The security framework ensures the 10DLC SMS system maintains the highest standards of security while enabling scalable, compliant messaging operations.
