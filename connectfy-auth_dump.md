# connectfy-auth Source Dump

## File: connectfy-auth/src/main.ts
```typescript
import i18n from './i18n';
import { NestFactory } from '@nestjs/core';
import { AppModule } from './app.module';
import { MicroserviceOptions, Transport } from '@nestjs/microservices';
import { AllExceptionsFilter } from 'connectfy-shared';
import { ValidationPipe } from '@nestjs/common';
import { ENVIRONMENT_VARIABLES } from './common/constants/environment-variables';

import { LoggingKafkaServer } from './app-settings/kafka-connections/loggin-kafka.server';

async function bootstrap() {
  // Start Kafka Microservice
  const kafkaApp = await NestFactory.createMicroservice<MicroserviceOptions>(
    AppModule,
    {
      strategy: new LoggingKafkaServer({
        client: {
          clientId: 'connectfy-auth',
          brokers: [
            ENVIRONMENT_VARIABLES.BROKER1,
            ENVIRONMENT_VARIABLES.BROKER2,
          ].filter(Boolean),
          logCreator:
            () =>
            ({ namespace, label, log }) => {
              if (log?.message?.includes('not host this topic') || log?.topic) {
                console.error(
                  `[kafkajs:${label}] ${namespace} topic=${log.topic ?? '?'} partition=${log.partition ?? '?'} → ${log.message}`,
                );
              }
            },
        },
        consumer: {
          groupId: 'consumer-connectfy-auth',
          allowAutoTopicCreation: false,
        },
        run: {
          autoCommit: false,
        },
      }),
    },
  );

  kafkaApp.useGlobalFilters(new AllExceptionsFilter());

  await kafkaApp.listen();
  console.log('✅ Kafka Microservice is running');

  // Starting TCP Microservice
  const PORT = Number(ENVIRONMENT_VARIABLES.PORT);
  const NODE_ENV = String(ENVIRONMENT_VARIABLES.NODE_ENV);
  const HOST = String(ENVIRONMENT_VARIABLES.HOST);

  const tcpApp = await NestFactory.createMicroservice<MicroserviceOptions>(
    AppModule,
    {
      transport: Transport.TCP,
      options: {
        host: HOST,
        port: PORT,
      },
    },
  );

  // Filter
  tcpApp.useGlobalFilters(new AllExceptionsFilter());
  tcpApp.useGlobalPipes(
    new ValidationPipe({ whitelist: true, transform: true }),
  );

  await tcpApp.listen();

  console.log(
    `i18n is working... should be used like this ==> `,
    i18n.t('email_messages.signup_verify.greeting', { lng: 'en' }),
  );
  console.log(`✅ NODE_ENV => `, NODE_ENV);
  console.log(`✅ Server is working on ${PORT} port`);
}
bootstrap();

```

## File: connectfy-auth/src/i18n.ts
```typescript
const i18n = require("i18next");
import { resources } from 'connectfy-i18n';

i18n.init({
  resources,
  fallbackLng: 'en',
  lng: 'en',
  interpolation: { escapeValue: false },
});

export default i18n;
```

## File: connectfy-auth/src/app.module.ts
```typescript
import { Module } from '@nestjs/common';
import { MongooseModule } from '@nestjs/mongoose';
import { ClsInterceptor, ClsModule } from 'nestjs-cls';
import { APP_INTERCEPTOR } from '@nestjs/core';
import { LoggedUserInterceptor } from './interceptors/logged-user.interceptor';
import { AppSettingsModule } from './app-settings/app-settings.module';
import { ExternalModulesModule } from './external-modules/external-modules.module';
import { InternalModulesModule } from './internal-modules/internal-modules.module';
import { ModulesModule } from './modules/modules.module';
import { ENVIRONMENT_VARIABLES } from './common/constants/environment-variables';

@Module({
  imports: [
    MongooseModule.forRoot(ENVIRONMENT_VARIABLES.MONGO_URI, {
      dbName: ENVIRONMENT_VARIABLES.DB_NAME,
    }),
    ClsModule.forRoot({
      global: true,
      interceptor: { mount: false },
    }),

    // /src/app-settings
    AppSettingsModule,
    // /src/external-modules
    ExternalModulesModule,
    // /src/internal-modules
    InternalModulesModule,
    // /src/modules
    ModulesModule,
  ],
  providers: [
    { provide: APP_INTERCEPTOR, useClass: ClsInterceptor },
    { provide: APP_INTERCEPTOR, useClass: LoggedUserInterceptor },
  ],
})
export class AppModule {}

```

## File: connectfy-auth/src/interceptors/logged-user.interceptor.ts
```typescript
import {
  CallHandler,
  ExecutionContext,
  Injectable,
  NestInterceptor,
} from '@nestjs/common';
import { Observable } from 'rxjs';
import { ClsService } from 'nestjs-cls';
import { CLS_KEYS, ILoggedUser, LANGUAGE } from 'connectfy-shared';

@Injectable()
export class LoggedUserInterceptor implements NestInterceptor {
  constructor(private readonly cls: ClsService) {}

  intercept(context: ExecutionContext, next: CallHandler): Observable<any> {
    const rpcCtx = context.switchToRpc();
    const data = rpcCtx.getData();

    const loggedUser: ILoggedUser | undefined = data?._loggedUser;

    if (loggedUser) {
      const language: LANGUAGE | undefined =
        data?._loggedUser?._lang ?? LANGUAGE.EN;
      return new Observable((subscriber) => {
        this.cls.run(() => {
          this.cls.set(CLS_KEYS.USER, loggedUser);
          this.cls.set(CLS_KEYS.LANG, language);
          next.handle().subscribe(subscriber);
        });
      });
    }

    return new Observable((subscriber) => {
      this.cls.run(() => {
        const language = data?._lang ?? LANGUAGE.EN;
        this.cls.set(CLS_KEYS.LANG, language);
        next.handle().subscribe(subscriber);
      });
    });
  }
}

```

## File: connectfy-auth/src/internal-modules/internal-modules.module.ts
```typescript
import { Global, Module } from '@nestjs/common';
import { InternalBcryptModule } from './bcrypt/bcrypt.module';
import { InternalRequestHelperModule } from './request-helper/request-helper.module';

@Global()
@Module({
  imports: [InternalBcryptModule, InternalRequestHelperModule],
  controllers: [],
  providers: [],
  exports: [InternalBcryptModule, InternalRequestHelperModule],
})
export class InternalModulesModule {}

```

## File: connectfy-auth/src/internal-modules/bcrypt/bcrypt.service.ts
```typescript
import { Injectable } from '@nestjs/common';
import { genSalt, hash, compare } from 'bcrypt';

@Injectable()
export class BcryptService {
  // =================================
  // HASH PASSWORD
  // =================================
  async hash(password: string): Promise<string> {
    return await hash(password, await genSalt());
  }

  // =================================
  // COMPARE PASSWORDS
  // =================================
  async compare(firstVal: string, secondVal: string): Promise<boolean> {
    return await compare(firstVal, secondVal);
  }
}

```

## File: connectfy-auth/src/internal-modules/bcrypt/bcrypt.module.ts
```typescript
import { Module } from '@nestjs/common';
import { BcryptService } from './bcrypt.service';

@Module({
  providers: [BcryptService],
  exports: [BcryptService],
})
export class InternalBcryptModule {}

```

## File: connectfy-auth/src/internal-modules/request-helper/request-helper.module.ts
```typescript
import { Module } from '@nestjs/common';
import { RequestHelperService } from './request-helper.service';

@Module({
  imports: [],
  controllers: [],
  providers: [RequestHelperService],
  exports: [RequestHelperService],
})
export class InternalRequestHelperModule {}

```

## File: connectfy-auth/src/internal-modules/request-helper/request-helper.service.ts
```typescript
import { Injectable } from '@nestjs/common';
import { UAParser } from 'ua-parser-js';
import * as geoip from 'geoip-lite';
import { BROWSER_TYPE, DEVICE_TYPE, OS_TYPE } from 'connectfy-shared';
import {
  IDeviceInfo,
  IDeviceInfoWithLocation,
  IGeoLocation,
  IParsedUserAgent,
  IRequestData,
} from './interfaces/request.interface';
import { Request } from 'express';
import countries from 'i18n-iso-countries';

@Injectable()
export class RequestHelperService {
  // ==========================================
  // Parse user agent string and return structured data with enums
  // ==========================================
  parseUserAgent(userAgent: string | undefined): IParsedUserAgent {
    if (!userAgent) {
      return {
        browser: BROWSER_TYPE.UNKNOWN,
        browserVersion: null,
        os: OS_TYPE.UNKNOWN,
        osVersion: null,
        platform: DEVICE_TYPE.UNKNOWN,
      };
    }

    try {
      const parser = new UAParser(userAgent);
      const result = parser.getResult();

      return {
        browser: this.mapBrowser(result.browser.name),
        browserVersion: result.browser.version || null,
        os: this.mapOS(result.os.name),
        osVersion: result.os.version || null,
        platform: this.mapPlatform(result.device.type),
      };
    } catch (error) {
      return {
        browser: BROWSER_TYPE.UNKNOWN,
        browserVersion: null,
        os: OS_TYPE.UNKNOWN,
        osVersion: null,
        platform: DEVICE_TYPE.UNKNOWN,
      };
    }
  }

  // ==========================================
  // Map browser name to BROWSER_TYPE enum
  // ==========================================
  private mapBrowser(browserName: string | undefined): BROWSER_TYPE {
    if (!browserName) return BROWSER_TYPE.UNKNOWN;

    const browser = browserName.toLowerCase();

    if (browser.includes('chrome') || browser.includes('chromium')) {
      return BROWSER_TYPE.CHROME;
    }
    if (browser.includes('firefox')) {
      return BROWSER_TYPE.FIREFOX;
    }
    if (browser.includes('safari') && !browser.includes('chrome')) {
      return BROWSER_TYPE.SAFARI;
    }
    if (browser.includes('edge') || browser.includes('edg')) {
      return BROWSER_TYPE.EDGE;
    }
    if (browser.includes('opera') || browser.includes('opr')) {
      return BROWSER_TYPE.OPERA;
    }
    if (browser.includes('brave')) {
      return BROWSER_TYPE.BRAVE;
    }
    if (browser.includes('samsung')) {
      return BROWSER_TYPE.SAMSUNG;
    }

    return BROWSER_TYPE.UNKNOWN;
  }

  // ==========================================
  // Map OS name to OS_TYPE enum
  // ==========================================
  private mapOS(osName: string | undefined): OS_TYPE {
    if (!osName) return OS_TYPE.UNKNOWN;

    const os = osName.toLowerCase();

    if (os.includes('windows')) {
      return OS_TYPE.WINDOWS;
    }
    if (os.includes('mac') || os.includes('darwin')) {
      return OS_TYPE.MACOS;
    }
    if (os.includes('linux') && !os.includes('android')) {
      return OS_TYPE.LINUX;
    }
    if (os.includes('android')) {
      return OS_TYPE.ANDROID;
    }
    if (os.includes('ios') || os.includes('iphone') || os.includes('ipad')) {
      return OS_TYPE.IOS;
    }

    return OS_TYPE.UNKNOWN;
  }

  // ==========================================
  // Map device type to DEVICE_TYPE enum
  // ==========================================
  private mapPlatform(deviceType: string | undefined): DEVICE_TYPE {
    if (!deviceType) return DEVICE_TYPE.WEB;

    const device = deviceType.toLowerCase();

    if (device === 'mobile') {
      return DEVICE_TYPE.MOBILE;
    }
    if (device === 'tablet') {
      return DEVICE_TYPE.TABLET;
    }
    if (device === 'desktop' || device === 'pc') {
      return DEVICE_TYPE.DESKTOP;
    }

    return DEVICE_TYPE.WEB;
  }

  // ==========================================
  // Get client IP address (handles proxies, load balancers, CDNs)
  // ==========================================
  getClientIp(request: Request): string {
    const ipHeaders = [
      'cf-connecting-ip', // Cloudflare
      'x-real-ip', // Nginx proxy
      'x-forwarded-for', // Standard proxy header
      'x-client-ip', // Apache
      'x-cluster-client-ip', // Rackspace LB
      'forwarded-for',
      'forwarded',
    ];

    for (const header of ipHeaders) {
      const value = request.headers[header];

      if (value) {
        const ip = Array.isArray(value) ? value[0] : value;
        const cleanIp = ip.split(',')[0].trim();

        if (cleanIp && cleanIp !== 'unknown') {
          return this.cleanIpAddress(cleanIp);
        }
      }
    }

    const socketIp = request.socket.remoteAddress;
    return this.cleanIpAddress(socketIp || 'unknown');
  }

  // ==========================================
  // Clean and normalize IP address
  // ==========================================
  cleanIpAddress(ip: string): string {
    if (ip.startsWith('::ffff:')) {
      return ip.substring(7);
    }

    if (ip === '::1' || ip === '127.0.0.1' || ip === 'localhost') {
      return '127.0.0.1';
    }

    const portIndex = ip.lastIndexOf(':');
    if (portIndex > 0 && !ip.includes('::')) {
      return ip.substring(0, portIndex);
    }

    return ip;
  }

  // ==========================================
  // Get full device info from request
  // ==========================================
  getDeviceInfo(request: Request): IDeviceInfo {
    const userAgent = request.headers['user-agent'] || 'Unknown';
    const parsedUA = this.parseUserAgent(userAgent);
    const ipAddress = this.getClientIp(request);

    return {
      userAgent,
      browser: parsedUA.browser,
      browserVersion: parsedUA.browserVersion,
      os: parsedUA.os,
      osVersion: parsedUA.osVersion,
      platform: parsedUA.platform,
      ipAddress,
      deviceName: `${parsedUA.os}-${parsedUA.browser}-${parsedUA.platform}`,
    };
  }

  // ==========================================
  // Get geolocation from IP address
  // ==========================================
  getGeoLocationFromIP(ipAddress: string): IGeoLocation {
    // Skip private/localhost IPs
    if (this.isPrivateIP(ipAddress)) {
      return {
        country: null,
        countryCode: null,
        city: null,
        region: null,
        timezone: null,
        latitude: null,
        longitude: null,
      };
    }

    try {
      const geo = geoip.lookup(ipAddress);

      if (!geo) {
        return {
          country: null,
          countryCode: null,
          city: null,
          region: null,
          timezone: null,
          latitude: null,
          longitude: null,
        };
      }

      return {
        country: this.getCountryName(geo.country),
        countryCode: geo.country || null,
        city: null, // geoip-lite doesn't provide city accurately
        region: geo.region || null,
        timezone: geo.timezone || null,
        latitude: geo.ll?.[0] || null, // ll = [latitude, longitude]
        longitude: geo.ll?.[1] || null,
      };
    } catch (error) {
      return {
        country: null,
        countryCode: null,
        city: null,
        region: null,
        timezone: null,
        latitude: null,
        longitude: null,
      };
    }
  }

  // ==========================================
  // Get device info WITH geolocation
  // ==========================================
  getDeviceInfoWithLocation(request: Request): IDeviceInfoWithLocation {
    const userAgent = request.headers['user-agent'] || 'Unknown';
    const parsedUA = this.parseUserAgent(userAgent);
    const ipAddress = this.getClientIp(request);
    const location = this.getGeoLocationFromIP(ipAddress);

    return {
      // Device info
      userAgent,
      browser: parsedUA.browser,
      browserVersion: parsedUA.browserVersion,
      os: parsedUA.os,
      osVersion: parsedUA.osVersion,
      platform: parsedUA.platform,
      deviceName: `${parsedUA.os}-${parsedUA.browser}-${parsedUA.platform}-${location.countryCode}`,
      ipAddress,

      // Geolocation
      country: location.country,
      countryCode: location.countryCode,
      city: location.city,
      region: location.region,
      timezone: location.timezone,
      latitude: location.latitude,
      longitude: location.longitude,
    };
  }

  // ==========================================
  // Generate human-readable device name
  // ==========================================
  generateDeviceName(
    browser: BROWSER_TYPE,
    os: OS_TYPE,
    platform: DEVICE_TYPE,
  ): string {
    const browserName = this.getBrowserDisplayName(browser);
    const osName = this.getOSDisplayName(os);
    const platformName = this.getPlatformDisplayName(platform);

    if (browser === BROWSER_TYPE.UNKNOWN && os === OS_TYPE.UNKNOWN) {
      return `${platformName} Device`;
    }

    if (browser === BROWSER_TYPE.UNKNOWN) {
      return `${platformName} on ${osName}`;
    }

    return `${browserName} on ${osName}`;
  }

  // ==========================================
  // Get display name for browser enum
  // ==========================================
  private getBrowserDisplayName(browser: BROWSER_TYPE): string {
    const names: Record<BROWSER_TYPE, string> = {
      [BROWSER_TYPE.CHROME]: 'Chrome',
      [BROWSER_TYPE.FIREFOX]: 'Firefox',
      [BROWSER_TYPE.SAFARI]: 'Safari',
      [BROWSER_TYPE.EDGE]: 'Edge',
      [BROWSER_TYPE.OPERA]: 'Opera',
      [BROWSER_TYPE.BRAVE]: 'Brave',
      [BROWSER_TYPE.SAMSUNG]: 'Samsung Internet',
      [BROWSER_TYPE.UNKNOWN]: 'Browser',
    };

    return names[browser];
  }

  // ==========================================
  // Get display name for OS enum
  // ==========================================
  private getOSDisplayName(os: OS_TYPE): string {
    const names: Record<OS_TYPE, string> = {
      [OS_TYPE.WINDOWS]: 'Windows',
      [OS_TYPE.MACOS]: 'macOS',
      [OS_TYPE.LINUX]: 'Linux',
      [OS_TYPE.ANDROID]: 'Android',
      [OS_TYPE.IOS]: 'iOS',
      [OS_TYPE.UNKNOWN]: 'Unknown OS',
    };

    return names[os];
  }

  // ==========================================
  // Get display name for platform enum
  // ==========================================
  private getPlatformDisplayName(platform: DEVICE_TYPE): string {
    const names: Record<DEVICE_TYPE, string> = {
      [DEVICE_TYPE.WEB]: 'Web',
      [DEVICE_TYPE.MOBILE]: 'Mobile',
      [DEVICE_TYPE.TABLET]: 'Tablet',
      [DEVICE_TYPE.DESKTOP]: 'Desktop',
      [DEVICE_TYPE.UNKNOWN]: 'Unknown',
    };

    return names[platform];
  }

  // ==========================================
  // Get country name from country code
  // ==========================================
  private getCountryName(code?: string | null): string | null {
    if (!code) return null;
    return countries.getName(code, 'en') ?? code;
  }

  // ==========================================
  // Validate if IP is internal/private
  // ==========================================
  isPrivateIP(ip: string): boolean {
    const cleanIp = this.cleanIpAddress(ip);

    if (cleanIp === '127.0.0.1' || cleanIp === '::1') return true;

    const privateRanges = [
      /^10\./,
      /^172\.(1[6-9]|2[0-9]|3[0-1])\./,
      /^192\.168\./,
    ];

    return privateRanges.some((range) => range.test(cleanIp));
  }

  // ==========================================
  // Parse device info WITH geolocation from request data
  // ==========================================
  parseDeviceInfoFromRequestData(
    requestData?: IRequestData,
  ): IDeviceInfoWithLocation {
    if (!requestData) {
      return {
        userAgent: 'Unknown',
        browser: BROWSER_TYPE.UNKNOWN,
        browserVersion: null,
        os: OS_TYPE.UNKNOWN,
        osVersion: null,
        platform: DEVICE_TYPE.UNKNOWN,
        ipAddress: 'unknown',
        deviceName: BROWSER_TYPE.UNKNOWN + OS_TYPE.UNKNOWN,
        country: null,
        countryCode: null,
        city: null,
        region: null,
        timezone: null,
        latitude: null,
        longitude: null,
      };
    }

    // Extract user agent
    const userAgent = Array.isArray(requestData.headers['user-agent'])
      ? requestData.headers['user-agent'][0]
      : requestData.headers['user-agent'] || 'Unknown';

    // Parse user agent
    const parsedUA = this.parseUserAgent(userAgent);

    // Extract IP address
    const ipAddress = this.extractIpFromRequestData(requestData);

    // Get geolocation
    const location = this.getGeoLocationFromIP(ipAddress);

    return {
      userAgent,
      browser: parsedUA.browser,
      browserVersion: parsedUA.browserVersion,
      os: parsedUA.os,
      osVersion: parsedUA.osVersion,
      platform: parsedUA.platform,
      deviceName: `${parsedUA.os}-${parsedUA.browser}-${parsedUA.platform}-${location.countryCode}`,
      ipAddress,
      country: location.country,
      countryCode: location.countryCode,
      city: location.city,
      region: location.region,
      timezone: location.timezone,
      latitude: location.latitude,
      longitude: location.longitude,
    };
  }

  // ==========================================
  // Extract IP address from request data
  // ==========================================
  extractIpFromRequestData(requestData: IRequestData): string {
    const headers = requestData.headers;

    if (headers['cf-connecting-ip']) {
      const ip = Array.isArray(headers['cf-connecting-ip'])
        ? headers['cf-connecting-ip'][0]
        : headers['cf-connecting-ip'];
      return this.cleanIpAddress(ip);
    }

    if (headers['x-real-ip']) {
      const ip = Array.isArray(headers['x-real-ip'])
        ? headers['x-real-ip'][0]
        : headers['x-real-ip'];
      return this.cleanIpAddress(ip);
    }

    if (headers['x-forwarded-for']) {
      const forwarded = Array.isArray(headers['x-forwarded-for'])
        ? headers['x-forwarded-for'][0]
        : headers['x-forwarded-for'];
      const ip = forwarded.split(',')[0].trim();
      return this.cleanIpAddress(ip);
    }

    if (requestData.ip) {
      return this.cleanIpAddress(requestData.ip);
    }

    return 'unknown';
  }
}

```

## File: connectfy-auth/src/internal-modules/request-helper/dto/request-data.dto.ts
```typescript
import { FIELD_TYPE, FieldValidator } from 'connectfy-shared';

export class RequestDataHeadersDto {
  @FieldValidator({ type: FIELD_TYPE.STRING })
  'user-agent': string;

  @FieldValidator({ type: FIELD_TYPE.STRING })
  'x-forwarded-for': string;

  @FieldValidator({ type: FIELD_TYPE.STRING })
  'x-real-ip': string;

  @FieldValidator({ type: FIELD_TYPE.STRING })
  'cf-connecting-ip': string;
}

export class RequestDataDto {
  @FieldValidator({ type: FIELD_TYPE.OBJECT, classType: RequestDataHeadersDto })
  headers: RequestDataHeadersDto;

  @FieldValidator({ type: FIELD_TYPE.STRING })
  ip: string;
}

```

## File: connectfy-auth/src/internal-modules/request-helper/interfaces/request.interface.ts
```typescript
import { BROWSER_TYPE, DEVICE_TYPE, OS_TYPE } from 'connectfy-shared';

export interface IParsedUserAgent {
  browser: BROWSER_TYPE;
  browserVersion: string | null;
  os: OS_TYPE;
  osVersion: string | null;
  platform: DEVICE_TYPE;
}

export interface IDeviceInfo {
  userAgent: string;
  browser: BROWSER_TYPE;
  browserVersion: string | null;
  os: OS_TYPE;
  osVersion: string | null;
  platform: DEVICE_TYPE;
  ipAddress: string;
  deviceName: string;
}

export interface IGeoLocation {
  country: string | null; // "Azerbaijan"
  countryCode: string | null; // "AZ"
  city: string | null; // null (geoip-lite doesn't provide)
  region: string | null; // "04" or region name
  timezone: string | null; // "Asia/Baku"
  latitude: number | null; // 40.4093
  longitude: number | null; // 49.8671
}

export interface IDeviceInfoWithLocation extends IDeviceInfo, IGeoLocation {}

export interface IRequestData {
  headers: {
    'user-agent': string;
    'x-forwarded-for': string;
    'x-real-ip': string;
    'cf-connecting-ip': string;
  };
  ip: string;
}

```

## File: connectfy-auth/src/external-modules/external-modules.module.ts
```typescript
import { Global, Module } from '@nestjs/common';
import { ExternalAccountModule } from './account/account.module';
import { ExternalNotificationsModule } from './notifications/notifications.module';

@Global()
@Module({
  imports: [ExternalAccountModule, ExternalNotificationsModule],
  controllers: [],
  providers: [],
  exports: [ExternalAccountModule, ExternalNotificationsModule],
})
export class ExternalModulesModule {}

```

## File: connectfy-auth/src/external-modules/notifications/notifications.module.ts
```typescript
import { Module } from '@nestjs/common';
import { NotificationsService } from './notifications.service';

@Module({
  imports: [],
  providers: [NotificationsService],
  exports: [NotificationsService],
})
export class ExternalNotificationsModule {}

```

## File: connectfy-auth/src/external-modules/notifications/notifications.service.ts
```typescript
import { Injectable } from '@nestjs/common';
import i18n from '@/src/i18n';
import { ISendEmail } from './interfaces/notifications.interface';
import {
  accountDeletedMessage,
  emailNotFoundMessage,
  forgotPasswordMessage,
  googleSignInMessage,
  signupVerifyMessage,
  changeEmailMessage,
  twoFactorVerifyMessage,
} from 'connectfy-shared';
import { KafkaConnectionService } from '@/src/app-settings/kafka-connections/kafka-connection.service';

@Injectable()
export class NotificationsService {
  constructor(
    private readonly kafkaConnectionService: KafkaConnectionService,
  ) {}

  // =================================
  // EMAIL SENDER
  // =================================
  private sendEmail(to: string, subject: string, html: string): void {
    this.kafkaConnectionService.emitWithContext({
      topic: 'mail.send',
      payload: {
        from: '"Connectfy Team" <connectfy.team@gmail.com>',
        sender: 'Connectfy Team',
        to,
        subject,
        html,
      },
    });
  }

  // =================================
  // SIGNUP VERIFY MESSAGE
  // =================================
  verifySignup(data: ISendEmail): void {
    const { to, language: lang, additional } = data;
    const { firstName, lastName, verifyCode } = additional!;
    this.sendEmail(
      to,
      i18n.t('email_messages.signup_verify.mail_subject', { lang }),
      signupVerifyMessage(firstName, lastName, verifyCode, lang),
    );
  }

  // =================================
  // EMAIL NOT FOUND FOR RESET PASSWORD
  // =================================
  emailNotFound(data: ISendEmail): void {
    const { to, language: lang } = data;
    this.sendEmail(
      to,
      i18n.t('email_messages.email_not_found.mail_subject', { lang }),
      emailNotFoundMessage(to, lang),
    );
  }

  // =================================
  // GOOGLE PROVIDER DETECETED FOR RESET PASSWORD
  // =================================
  googleSignInDetected(data: ISendEmail): void {
    const { to, language: lang } = data;
    this.sendEmail(
      to,
      i18n.t('email_messages.google_sign_in.mail_subject', { lang }),
      googleSignInMessage(to, lang),
    );
  }

  // =================================
  // FORGOT PASSWORD EMAIL TO RESET PASSWORD
  // =================================
  forgotPassword(data: ISendEmail): void {
    const { to, language: lang, additional } = data;
    const { token } = additional!;
    this.sendEmail(
      to,
      i18n.t('email_messages.forgot_password.mail_subject', { lang }),
      forgotPasswordMessage(token, lang),
    );
  }

  // =================================
  // RESTORE ACCOUNT EMAIL AFTER DELETE
  // =================================
  deleteAccountCompleted(data: ISendEmail): void {
    const { to, language: lang, additional } = data;
    const { token } = additional!;
    this.sendEmail(
      to,
      i18n.t('email_messages.delete_account_completed.mail_subject', { lang }),
      accountDeletedMessage(token, lang),
    );
  }

  // =================================
  // MESSAGE FOR CHANGE EMAIL
  // =================================
  changeEmail(data: ISendEmail): void {
    const { to, language: lang, additional } = data;
    const { token } = additional!;
    this.sendEmail(
      to,
      i18n.t('email_messages.change_email.mail_subject', { lang }),
      changeEmailMessage(token, lang),
    );
  }

  // =================================
  // MESSAGE FOR TWO FACTOR AUTHENTICATION
  // =================================
  twoFaEmail(data: ISendEmail): void {
    const { to, language: lang, additional } = data;
    const { firstName, lastName, verifyCode } = additional!;

    this.sendEmail(
      to,
      i18n.t('email_messages.two_factor.subject', { lng: lang }),
      twoFactorVerifyMessage(firstName, lastName, verifyCode, lang),
    );
  }
}

```

## File: connectfy-auth/src/external-modules/notifications/interfaces/notifications.interface.ts
```typescript
import { LANGUAGE } from 'connectfy-shared';

export interface ISendEmail {
  to: string;
  language: LANGUAGE;
  additional?: Record<string, any>;
}

```

## File: connectfy-auth/src/external-modules/account/account.service.ts
```typescript
import { TcpConnectionService } from '@/src/app-settings/tcp-connections/tcp-connection.service';
import { Injectable } from '@nestjs/common';
import { GENDER, LANGUAGE, CLS_KEYS, BaseFindDto } from 'connectfy-shared';
import { ClsService } from 'nestjs-cls';

@Injectable()
export class AccountService {
  constructor(
    private readonly tcpConnectionService: TcpConnectionService,
    private readonly cls: ClsService,
  ) {}

  // =================================
  // CREATE USER PROFILE AND SETTINGS
  // =================================
  async createAccountRelatedServices(opts: {
    userId: string;
    username: string;
    firstName: string;
    lastName: string;
    gender: GENDER;
    avatar?: string | null;
    birthdayDate: Date | null;
    theme: string | null;
    location: string | null;
  }): Promise<void> {
    const language = this.cls.get<LANGUAGE>(CLS_KEYS.LANG);
    const {
      userId,
      firstName = '',
      lastName = '',
      username,
      gender,
      avatar,
      birthdayDate,
      theme,
    } = opts;

    await Promise.all([
      // account create
      this.tcpConnectionService.account({
        endpoint: 'profile/create',
        payload: {
          userId,
          firstName,
          lastName,
          username,
          gender,
          avatar,
          birthdayDate,
        },
      }),

      // privacy settings create
      this.tcpConnectionService.account({
        endpoint: 'privacy-settings/create',
        payload: { userId },
      }),

      // general settings create
      this.tcpConnectionService.account({
        endpoint: 'general-settings/create',
        payload: { userId, theme, language },
      }),

      // notification settings create
      this.tcpConnectionService.account({
        endpoint: 'notification-settings/create',
        payload: { userId },
      }),
    ]);
  }

  // =================================
  // FIND USER PROFILE
  // =================================
  async findProfile(payload: BaseFindDto): Promise<any> {
    return await this.tcpConnectionService.account({
      endpoint: 'profile/findOne',
      payload,
    });
  }

  // =================================
  // FIND GENERAL SETTINGS
  // =================================
  async findGeneralSettings(payload: BaseFindDto): Promise<any> {
    return await this.tcpConnectionService.account({
      endpoint: 'general-settings/findOne',
      payload,
    });
  }

  // =================================
  // FIND NOTIFICATION SETTINGS
  // =================================
  async findNotificationSettings(payload: BaseFindDto): Promise<any> {
    return await this.tcpConnectionService.account({
      endpoint: 'notification-settings/findOne',
      payload,
    });
  }

  // =================================
  // FIND PRIVACY SETTINGS
  // =================================
  async findPrivacySettings(payload: BaseFindDto): Promise<any> {
    return await this.tcpConnectionService.account({
      endpoint: 'privacy-settings/findOne',
      payload,
    });
  }
}

```

## File: connectfy-auth/src/external-modules/account/account.module.ts
```typescript
import { Module } from '@nestjs/common';
import { AccountService } from './account.service';

@Module({
  imports: [],
  providers: [AccountService],
  exports: [AccountService],
})
export class ExternalAccountModule {}

```

## File: connectfy-auth/src/modules/modules.module.ts
```typescript
import { Module } from '@nestjs/common';
import { AuthModule } from './auth/auth.module';
import { TokensModule } from './tokens/tokens.module';
import { UsersModule } from './users/users.module';

@Module({
  imports: [AuthModule, TokensModule, UsersModule],
  controllers: [],
  providers: [],
  exports: [],
})
export class ModulesModule {}

```

## File: connectfy-auth/src/modules/auth/auth.controller.ts
```typescript
import { MessagePattern, Payload, Transport } from '@nestjs/microservices';
import { Controller, HttpStatus } from '@nestjs/common';
import { AuthService } from './auth.service';
import { GoogleAuthSignupDto, SignupDto } from './dto/signup.dto';
import { VerifySignupDto, VerifyLoginDto } from './dto/verify.dto';
import { GoogleAuthLoginDto, LoginDto } from './dto/login.dto';
import { ResetPasswordDto } from './dto/reset-password.dto';
import { ForgotPasswordDto } from './dto/forgot-password.dto';
import { LANGUAGE, BaseException, ExceptionMessages } from 'connectfy-shared';
import { ValidateTokenDto } from './dto/validate-token.dto';
import { AuthenticateUserDto } from './dto/authenticate-user.dto';
import { LogoutDto } from './dto/logout.dto';
import { RestoreAccountDto } from './dto/restore-account.dto';

@Controller('')
export class AuthController {
  constructor(private readonly service: AuthService) {}

  @MessagePattern('auth/signup', Transport.TCP)
  async signup(@Payload() data: SignupDto) {
    return this.service.signup(data);
  }

  @MessagePattern('auth/verify-signup', Transport.TCP)
  async verifySignup(@Payload() data: VerifySignupDto) {
    return this.service.verifySignup(data);
  }

  @MessagePattern('auth/verify-signup/resend', Transport.TCP)
  async resendSignupVerify(@Payload() data: SignupDto) {
    if (!data)
      throw new BaseException(
        ExceptionMessages.NOT_FOUND_MESSAGE(LANGUAGE.EN),
        HttpStatus.NOT_FOUND,
        { navigate: true },
      );

    return this.service.resendSignupVerify(data);
  }

  @MessagePattern('auth/login', Transport.TCP)
  async login(@Payload() data: LoginDto) {
    return this.service.login(data);
  }

  @MessagePattern('auth/verify-login', Transport.TCP)
  async verifyLogin(@Payload() data: VerifyLoginDto) {
    return this.service.verifyLogin(data);
  }

  @MessagePattern('auth/google/login', Transport.TCP)
  async googleAuthLogin(@Payload() data: GoogleAuthLoginDto) {
    return this.service.googleLogin(data);
  }

  @MessagePattern('auth/google/signup', Transport.TCP)
  async googleAuthSignup(@Payload() data: GoogleAuthSignupDto) {
    return this.service.googleSignup(data);
  }

  @MessagePattern('auth/forgot-password', Transport.TCP)
  async forgotPassword(@Payload() data: ForgotPasswordDto) {
    return this.service.forgotPassword(data);
  }

  @MessagePattern('auth/reset-password', Transport.TCP)
  async resetPassword(@Payload() data: ResetPasswordDto) {
    return this.service.resetPassword(data);
  }

  @MessagePattern('auth/is-valid-token', Transport.TCP)
  async isTokenValid(@Payload() data: ValidateTokenDto) {
    return this.service.isTokenValid(data);
  }

  @MessagePattern('auth/logout', Transport.TCP)
  async logout(@Payload() data: LogoutDto) {
    return this.service.logout(data);
  }

  @MessagePattern('auth/refresh-token/verify-token', Transport.TCP)
  async verifyAuthToken(
    @Payload()
    {
      access_token,
      refresh_token,
    }: {
      access_token: string;
      refresh_token: string;
    },
  ) {
    return this.service.verifyAuthToken(access_token, refresh_token);
  }

  @MessagePattern('auth/refreshToken', Transport.TCP)
  async refreshToken(@Payload() data: Record<string, any>) {
    if (!data.refresh_token || !data.deviceId || !data.requestData)
      throw new BaseException(
        ExceptionMessages.NOT_FOUND_MESSAGE(LANGUAGE.EN),
        HttpStatus.NOT_FOUND,
        { navigate: true },
      );

    return this.service.refreshToken(data);
  }

  @MessagePattern('auth/authenticate-user', Transport.TCP)
  async authenticateUser(@Payload() data: AuthenticateUserDto) {
    return this.service.authenticateUser(data);
  }

  @MessagePattern('auth/restore-account', Transport.TCP)
  async restoreAccount(@Payload() data: RestoreAccountDto) {
    return this.service.restoreAccount(data);
  }
}

```

## File: connectfy-auth/src/modules/auth/auth.module.ts
```typescript
import { Module } from '@nestjs/common';
import { JwtModule } from '@nestjs/jwt';
import { AuthService } from './auth.service';
import { AuthController } from './auth.controller';
import { UsersModule } from '../users/users.module';
import { RefreshTokenModule } from '../tokens/refresh-token/refresh-token.module';
import { UserModule } from '../users/user/user.module';
import { DeletedUserModule } from '../users/deleted-user/deleted-user.module';
import { BannedUserModule } from '../users/banned-user/banned-user.module';
import { TokenModule } from '../tokens/token/token.module';
import { DeactivatedUsersModule } from '../users/deactivated-users/deactivated-users.module';

@Module({
  imports: [
    UsersModule,
    RefreshTokenModule,
    JwtModule,
    UserModule,
    DeletedUserModule,
    BannedUserModule,
    TokenModule,
    DeactivatedUsersModule,
  ],
  controllers: [AuthController],
  providers: [AuthService],
})
export class AuthModule {}

```

## File: connectfy-auth/src/modules/auth/auth.service.ts
```typescript
import {
  CLS_KEYS,
  FORGOT_PASSWORD_IDENTIFIER_TYPE,
  IDENTIFIER_TYPE,
  LANGUAGE,
  PROVIDER,
  TOKEN_TYPE,
  USER_STATUS,
  BaseException,
  ExceptionMessages,
  EXPIRE_DATES,
  THEME,
  STARTUP_PAGE,
} from 'connectfy-shared';
import { generateVerifyCode } from '@/src/common/functions/function';
import { HttpStatus, Injectable } from '@nestjs/common';
import { RefreshTokenService } from '../tokens/refresh-token/refresh-token.service';
import { GoogleAuthSignupDto, SignupDto } from './dto/signup.dto';
import { VerifySignupDto, VerifyLoginDto } from './dto/verify.dto';
import { GoogleAuthLoginDto, LoginDto } from './dto/login.dto';
import { IReturnedUser } from '../users/user/interface/user.interface';
import { OAuth2Client } from 'google-auth-library';
import { ForgotPasswordDto } from './dto/forgot-password.dto';
import { ResetPasswordDto } from './dto/reset-password.dto';
import { ValidateTokenDto } from './dto/validate-token.dto';
import { ClsService } from 'nestjs-cls';
import { AuthenticateUserDto } from './dto/authenticate-user.dto';
import { JwtService } from '@nestjs/jwt';
import { RequestHelperService } from '@/src/internal-modules/request-helper/request-helper.service';
import { NotificationsService } from '@/src/external-modules/notifications/notifications.service';
import { BcryptService } from '@/src/internal-modules/bcrypt/bcrypt.service';
import { TokenService } from '../tokens/token/token.service';
import { AccountService } from '@/src/external-modules/account/account.service';
import { ENVIRONMENT_VARIABLES } from '@/src/common/constants/environment-variables';
import { LogoutDto } from './dto/logout.dto';
import { BannedUserService } from '../users/banned-user/banned-user.service';
import { DeactivatedUsersService } from '../users/deactivated-users/deactivated-users.service';
import { DeletedUserService } from '../users/deleted-user/deleted-user.service';
import { RestoreAccountDto } from './dto/restore-account.dto';
import { UserService } from '../users/user/user.service';

@Injectable()
export class AuthService {
  private googleClient: OAuth2Client;

  constructor(
    private cls: ClsService,
    private readonly refreshTokenService: RefreshTokenService,
    private readonly tokenService: TokenService,
    private readonly jwtService: JwtService,
    private readonly emailService: NotificationsService,
    private readonly bcryptService: BcryptService,
    private readonly accountService: AccountService,
    private readonly requestHelperService: RequestHelperService,
    private readonly bannedUserService: BannedUserService,
    private readonly deactivatedUserService: DeactivatedUsersService,
    private readonly deletedUserService: DeletedUserService,
    private readonly userService: UserService,
  ) {
    const clientId = ENVIRONMENT_VARIABLES.GOOGLE_CLIENT_ID;
    this.googleClient = new OAuth2Client(clientId);
  }

  // =================================
  // SIGNUP
  // =================================
  async signup(
    data: SignupDto,
  ): Promise<{ unverifiedUser: Record<string, any>; verifyCode: string }> {
    const language = this.cls.get<LANGUAGE>(CLS_KEYS.LANG);
    const { firstName, lastName, email, username } = data;

    const userWithUsername = await this.userService.existByField({
      username,
    });

    if (userWithUsername) {
      throw new BaseException(
        ExceptionMessages.ALREADY_EXISTS_MESSAGE(username, language),
        HttpStatus.CONFLICT,
      );
    }

    const userWithEmail = await this.userService.existByField({ email });

    if (userWithEmail) {
      throw new BaseException(
        ExceptionMessages.ALREADY_EXISTS_MESSAGE(email, language),
        HttpStatus.CONFLICT,
      );
    }

    const verifyCode = generateVerifyCode();

    this.emailService.verifySignup({
      to: email,
      language,
      additional: { firstName, lastName, verifyCode },
    });

    return {
      unverifiedUser: data,
      verifyCode,
    };
  }

  // =================================
  // VERIFY SIGNUP
  // =================================
  async verifySignup(
    data: VerifySignupDto,
  ): Promise<{ _id: string; access_token?: string; refresh_token: string }> {
    const language = this.cls.get<LANGUAGE>(CLS_KEYS.LANG);
    const { code, verifyCode, unverifiedUser, deviceId, requestData } = data;

    if (verifyCode !== code)
      throw new BaseException(
        ExceptionMessages.INVALID_CREDENTIALS(language),
        HttpStatus.CONFLICT,
      );

    const { firstName, lastName, birthdayDate, gender, theme, ...authDatas } =
      unverifiedUser;

    const userIp =
      this.requestHelperService.extractIpFromRequestData(requestData);
    const userLocation = this.requestHelperService.getGeoLocationFromIP(userIp);

    const { _id, username } = await this.userService.create({
      ...authDatas,
      firstName,
      lastName,
      provider: PROVIDER.PASSWORD,
      timeZone: userLocation.timezone || null,
      location: `${userLocation.country ?? ''}${userLocation.city ? `, ${userLocation.city}` : ''}`,
    });

    await this.accountService.createAccountRelatedServices({
      userId: _id,
      username,
      firstName,
      lastName,
      gender,
      birthdayDate,
      theme,
      location: `${userLocation.country ?? ''}${userLocation.city ? `, ${userLocation.city}` : ''}`,
    });

    const { access_token, refresh_token } =
      await this.refreshTokenService.generateTokens({ _id });

    await this.refreshTokenService.saveTokens({
      refresh_token,
      userId: _id,
      deviceId,
      requestData,
    });

    return { _id, access_token, refresh_token };
  }

  // =================================
  // RESEND VERIFY SIGNUP
  // =================================
  async resendSignupVerify(data: SignupDto) {
    const language = this.cls.get<LANGUAGE>(CLS_KEYS.LANG);
    const verifyCode = generateVerifyCode();

    this.emailService.verifySignup({
      to: data.email,
      language,
      additional: {
        firstName: data.firstName,
        lastName: data.lastName,
        verifyCode,
      },
    });

    return { verifyCode };
  }

  // =================================
  // LOGIN
  // =================================
  async login(data: LoginDto): Promise<
    | {
        access_token?: string;
        refresh_token: string;
        language: LANGUAGE;
        theme: THEME;
        startupPage: STARTUP_PAGE;
      }
    | { success: true; isTwoFactorEnabled: true; code: string; userId: string }
  > {
    const language = this.cls.get<LANGUAGE>(CLS_KEYS.LANG);
    const { identifierType, identifier, password, requestData, deviceId } =
      data;

    let user: IReturnedUser | null;

    switch (identifierType) {
      case IDENTIFIER_TYPE.EMAIL:
        user = (await this.userService.findOne({
          query: { email: identifier },
          fields: 'password email provider status isTwoFactorEnabled',
        })) as IReturnedUser;
        break;

      case IDENTIFIER_TYPE.USERNAME:
        user = (await this.userService.findOne({
          query: { username: identifier },
          fields: 'password email provider status isTwoFactorEnabled',
        })) as IReturnedUser;
        break;

      default:
        user = (await this.userService.findOne({
          query: { 'phoneNumber.fullPhoneNumber': identifier },
          fields: 'password email provider status isTwoFactorEnabled',
        })) as IReturnedUser;
        break;
    }

    if (user.provider !== PROVIDER.PASSWORD) {
      throw new BaseException(
        ExceptionMessages.INVALID_CREDENTIALS(language),
        HttpStatus.CONFLICT,
      );
    }

    const isPasswordMatch = await this.bcryptService.compare(
      password,
      user.password,
    );

    if (!isPasswordMatch) {
      throw new BaseException(
        ExceptionMessages.INVALID_CREDENTIALS(language),
        HttpStatus.CONFLICT,
      );
    }

    await this.bannedUserService.isUserBanned({ userId: user._id });

    if (
      user.status !== USER_STATUS.ACTIVE &&
      user.status !== USER_STATUS.INACTIVE
    ) {
      throw new BaseException(
        ExceptionMessages.INVALID_CREDENTIALS(language),
        HttpStatus.CONFLICT,
      );
    }

    if (user.isTwoFactorEnabled) {
      const twoFaCode = generateVerifyCode();

      const account = await this.accountService.findProfile({
        query: { userId: user._id },
        fields: 'firstName lastName',
      });

      this.emailService.twoFaEmail({
        to: user.email,
        language,
        additional: {
          firstName: account?.firstName || '',
          lastName: account?.lastName || '',
          verifyCode: twoFaCode,
        },
      });

      return {
        success: true,
        isTwoFactorEnabled: true,
        code: twoFaCode,
        userId: user._id,
      };
    }

    if (user.status === USER_STATUS.INACTIVE) {
      await this.deactivatedUserService.activateUserIfExist(user._id);
    }

    const { refresh_token, access_token } =
      await this.refreshTokenService.generateTokens({
        _id: user._id,
      });

    const generalSettings = await this.accountService.findGeneralSettings({
      query: { userId: user._id },
      fields: 'language theme startupPage',
    });

    await this.refreshTokenService.saveTokens({
      refresh_token,
      userId: user._id,
      deviceId,
      requestData,
    });

    return {
      access_token,
      refresh_token,
      language: generalSettings.language,
      theme: generalSettings.theme,
      startupPage: generalSettings.startupPage,
    };
  }

  // =================================
  // VERIFY LOGIN
  // =================================
  async verifyLogin(data: VerifyLoginDto) {
    const language = this.cls.get<LANGUAGE>(CLS_KEYS.LANG);
    const { twoFaCode, code, requestData, deviceId, userId } = data;

    const user = (await this.userService.findOne({
      query: { _id: userId },
      fields: 'password provider status isTwoFactorEnabled',
    })) as IReturnedUser;

    if (!user.isTwoFactorEnabled || user.provider !== PROVIDER.PASSWORD) {
      throw new BaseException(
        ExceptionMessages.INVALID_CREDENTIALS(language),
        HttpStatus.CONFLICT,
      );
    }

    await this.bannedUserService.isUserBanned({ userId: user._id });

    if (twoFaCode !== code) {
      throw new BaseException(
        ExceptionMessages.INVALID_CREDENTIALS(language),
        HttpStatus.CONFLICT,
      );
    }

    if (user.status === USER_STATUS.INACTIVE) {
      await this.deactivatedUserService.activateUserIfExist(user._id);
    }

    const { refresh_token, access_token } =
      await this.refreshTokenService.generateTokens({
        _id: user._id,
      });

    const generalSettings = await this.accountService.findGeneralSettings({
      query: { userId: user._id },
      fields: 'language theme startupPage',
    });

    await this.refreshTokenService.saveTokens({
      refresh_token,
      userId: user._id,
      deviceId,
      requestData,
    });

    return {
      access_token,
      refresh_token,
      language: generalSettings.language,
      theme: generalSettings.theme,
      startupPage: generalSettings.startupPage,
    };
  }

  // =================================
  // GOOGLE LOGIN
  // =================================
  async googleLogin(data: GoogleAuthLoginDto): Promise<{
    access_token?: string;
    refresh_token: string;
    language: LANGUAGE;
    theme: THEME;
    startupPage: STARTUP_PAGE;
  }> {
    const language = this.cls.get<LANGUAGE>(CLS_KEYS.LANG);
    const { idToken, deviceId, requestData } = data;

    const ticket = await this.googleClient.verifyIdToken({
      idToken,
      audience: ENVIRONMENT_VARIABLES.GOOGLE_CLIENT_ID,
    });

    const payload = ticket.getPayload();

    const email = payload?.email;

    if (!email) {
      throw new BaseException(
        ExceptionMessages.INVALID_CREDENTIALS(language),
        HttpStatus.CONFLICT,
      );
    }

    const user = (await this.userService.findOne({
      query: { email },
    })) as IReturnedUser;

    if (user.provider !== PROVIDER.GOOGLE) {
      throw new BaseException(
        ExceptionMessages.INVALID_CREDENTIALS(language),
        HttpStatus.CONFLICT,
      );
    }

    await this.bannedUserService.isUserBanned({ userId: user._id });

    if (
      user.status !== USER_STATUS.ACTIVE &&
      user.status !== USER_STATUS.INACTIVE
    ) {
      throw new BaseException(
        ExceptionMessages.INVALID_CREDENTIALS(language),
        HttpStatus.CONFLICT,
      );
    }

    if (user.status === USER_STATUS.INACTIVE) {
      await this.deactivatedUserService.activateUserIfExist(user._id);
    }

    const { access_token, refresh_token } =
      await this.refreshTokenService.generateTokens({
        _id: user._id,
      });

    const generalSettings = await this.accountService.findGeneralSettings({
      query: { userId: user._id },
      fields: 'language theme startupPage',
    });

    await this.refreshTokenService.saveTokens({
      refresh_token,
      userId: user._id,
      deviceId,
      requestData,
    });

    return {
      access_token,
      refresh_token,
      language: generalSettings.language,
      theme: generalSettings.theme,
      startupPage: generalSettings.startupPage,
    };
  }

  // =================================
  // GOOGLE SIGNUP
  // =================================
  async googleSignup(
    data: GoogleAuthSignupDto,
  ): Promise<{ _id: string; access_token?: string; refresh_token: string }> {
    const language = this.cls.get<LANGUAGE>(CLS_KEYS.LANG);
    const {
      idToken,
      username,
      gender,
      theme,
      birthdayDate,
      deviceId,
      requestData,
    } = data;

    const ticket = await this.googleClient.verifyIdToken({
      idToken,
      audience: ENVIRONMENT_VARIABLES.GOOGLE_CLIENT_ID,
    });

    const payload = ticket.getPayload();

    const email = payload?.email;
    const firstName = payload?.given_name || '';
    const lastName = payload?.family_name || '';

    if (!email)
      throw new BaseException(
        ExceptionMessages.INVALID_CREDENTIALS(language),
        HttpStatus.CONFLICT,
      );

    const userWithEmail = await this.userService.existByField({ email });

    if (userWithEmail) {
      throw new BaseException(
        ExceptionMessages.ALREADY_EXISTS_MESSAGE(email, language),
        HttpStatus.CONFLICT,
      );
    }

    const userWithUsername = await this.userService.existByField({ username });

    if (userWithUsername) {
      throw new BaseException(
        ExceptionMessages.ALREADY_EXISTS_MESSAGE(username, language),
        HttpStatus.CONFLICT,
      );
    }

    const userIp =
      this.requestHelperService.extractIpFromRequestData(requestData);
    const userLocation = this.requestHelperService.getGeoLocationFromIP(userIp);

    const { _id } = await this.userService.create({
      firstName,
      lastName,
      email,
      username,
      provider: PROVIDER.GOOGLE,
      password: await this.bcryptService.hash('signed_up_with_google'),
      timeZone: userLocation.timezone || null,
      location: `${userLocation.country ?? ''}${userLocation.city ? `, ${userLocation.city}` : ''}`,
    });

    await this.accountService.createAccountRelatedServices({
      userId: _id,
      username,
      firstName,
      lastName,
      gender,
      birthdayDate,
      theme,
      location: `${userLocation.country ?? ''}${userLocation.city ? `, ${userLocation.city}` : ''}`,
    });

    const { access_token, refresh_token } =
      await this.refreshTokenService.generateTokens({ _id });

    await this.refreshTokenService.saveTokens({
      refresh_token,
      userId: _id,
      deviceId,
      requestData,
    });

    return { _id, access_token, refresh_token };
  }

  // =================================
  // FORGOT PASSWORD
  // =================================
  async forgotPassword(
    data: ForgotPasswordDto,
  ): Promise<{ statusCode: 200; email?: string }> {
    const language = this.cls.get<LANGUAGE>(CLS_KEYS.LANG);
    const { identifierType, identifier } = data;

    const isEmail = identifierType === FORGOT_PASSWORD_IDENTIFIER_TYPE.EMAIL;
    const user = await this.userService.findOne(
      isEmail
        ? { query: { email: identifier } }
        : { query: { 'phoneNumber.fullPhoneNumber': identifier } },
      false,
    );

    if (!user) {
      if (!isEmail) return { statusCode: 200 };

      this.emailService.emailNotFound({ to: identifier, language });

      return { statusCode: 200 };
    }

    if (user.provider !== PROVIDER.PASSWORD) {
      this.emailService.googleSignInDetected({ to: identifier, language });

      return { statusCode: 200 };
    }

    const token = await this.tokenService.generateAndSaveJwtToken({
      userId: user._id,
      type: TOKEN_TYPE.PASSWORD_RESET,
      secret: 'FORGOT_PASSWORD_SECRET',
      jwtExp: EXPIRE_DATES.JWT.ONE_HOUR,
      tokenExp: EXPIRE_DATES.TOKEN.ONE_HOUR,
    });

    this.emailService.forgotPassword({
      to: identifier,
      language,
      additional: { token },
    });

    const emailParts = user.email.split('@');

    const local = emailParts[0] || '';
    const localLength = local.length;
    const domain = emailParts[0] || '';

    const email =
      local[0] + '*'.repeat(localLength - 2) + local[localLength - 1] + domain;

    return { statusCode: 200, email };
  }

  // =================================
  // IS TOKEN VALID
  // =================================
  async isTokenValid(data: ValidateTokenDto): Promise<boolean> {
    try {
      const { token, type } = data;
      const hashedToken = this.tokenService.hashToken(token);

      const res = await this.tokenService.findToken({
        query: { $and: [{ token: hashedToken }, { type }] },
      });

      const expiresAt = res.expiresAt ? res.expiresAt : new Date(res.expiresAt);
      const now = new Date();

      if (now >= expiresAt) return false;
      return !res.isUsed;
    } catch {
      return false;
    }
  }

  // =================================
  // RESET PASSWORD
  // =================================
  async resetPassword(data: ResetPasswordDto): Promise<{ statusCode: 200 }> {
    const language = this.cls.get<LANGUAGE>(CLS_KEYS.LANG);
    const { resetToken, password, confirmPassword } = data;

    const hashedToken = this.tokenService.hashToken(resetToken);

    const token = await this.tokenService.findToken({
      query: {
        $and: [{ token: hashedToken }, { type: TOKEN_TYPE.PASSWORD_RESET }],
      },
      populate: [{ path: 'userId', select: '_id, password' }],
    });

    const decoded = this.jwtService.verify(resetToken, {
      secret: ENVIRONMENT_VARIABLES.FORGOT_PASSWORD_SECRET,
    });

    if (decoded.type !== TOKEN_TYPE.PASSWORD_RESET)
      throw new BaseException(
        ExceptionMessages.CONFLICT_MESSAGE(language),
        HttpStatus.CONFLICT,
      );

    const user = token.userId as IReturnedUser;

    if (decoded.userId !== user._id)
      throw new BaseException(
        ExceptionMessages.FORBIDDEN_MESSAGE(language),
        HttpStatus.FORBIDDEN,
      );

    const now = new Date();

    if (now >= token.expiresAt || token.isUsed) {
      if (token) await this.tokenService.removeToken({ _id: token._id });

      throw new BaseException(
        ExceptionMessages.TOKEN_EXPIRED(language),
        HttpStatus.BAD_REQUEST,
      );
    }

    if (password !== confirmPassword)
      throw new BaseException(
        ExceptionMessages.BAD_REQUEST_MESSAGE(language),
        HttpStatus.BAD_REQUEST,
      );

    const isPasswordSame = await this.bcryptService.compare(
      password,
      user.password,
    );

    if (isPasswordSame)
      throw new BaseException(
        ExceptionMessages.SAME_DATA('password', language),
        HttpStatus.BAD_REQUEST,
      );

    const hashedPassword = await this.bcryptService.hash(password);

    await Promise.all([
      this.userService.edit({ _id: user._id, password: hashedPassword }),
      this.tokenService.removeToken({ _id: token._id }),
    ]);
    return { statusCode: 200 };
  }

  // =================================
  // LOGOUT
  // =================================
  async logout(data: LogoutDto): Promise<{ statusCode: 200 }> {
    const { deviceId } = data;
    const { _id } = this.cls.get<IReturnedUser>(CLS_KEYS.USER);

    const findToken = await this.refreshTokenService.findOne({
      query: {
        $and: [{ userId: _id }, { deviceId }],
      },
      fields: 'userId deviceId',
    });

    await this.refreshTokenService.removeTokenByUserId(findToken.userId);

    return { statusCode: 200 };
  }

  // =================================
  // VERIFY AUTH TOKEN
  // =================================
  async verifyAuthToken(access_token: string, refresh_token: string) {
    const payload = await this.refreshTokenService.verifyToken(access_token);
    const language = this.cls.get<LANGUAGE>(CLS_KEYS.LANG);

    if (!payload._id) {
      throw new BaseException(
        ExceptionMessages.UNAUTHORIZED_MESSAGE(language),
        HttpStatus.UNAUTHORIZED,
        { navigate: true },
      );
    }

    const storedToken = await this.refreshTokenService.findToken(refresh_token);

    if (!storedToken || storedToken.userId !== payload._id) {
      throw new BaseException(
        ExceptionMessages.UNAUTHORIZED_MESSAGE(language),
        HttpStatus.UNAUTHORIZED,
        { navigate: true },
      );
    }

    const user = (await this.userService.findOne({
      query: { _id: payload._id },
      fields: '-password',
    })) as IReturnedUser;

    await this.bannedUserService.isUserBanned({ userId: user._id });

    const [account, generalSettings] = await Promise.all([
      this.accountService.findProfile({
        query: { userId: user._id },
        fields: 'avatar defaultAvatar',
      }),
      this.accountService.findGeneralSettings({
        query: { userId: user._id },
        fields: 'language',
      }),
    ]);

    if (!account || !generalSettings) {
      throw new BaseException(
        ExceptionMessages.UNAUTHORIZED_MESSAGE(language),
        HttpStatus.UNAUTHORIZED,
        { navigate: true },
      );
    }

    const result = {
      ...user,
      language: generalSettings.language,
      avatar: account.avatar,
      defaultAvatar: account.defaultAvatar,
    };

    return { status: 200, user: result };
  }

  // =================================
  // REFRESH TOKEN
  // =================================
  async refreshToken(data: Record<string, any>) {
    const language = this.cls.get<LANGUAGE>(CLS_KEYS.LANG);
    const payload = await this.refreshTokenService.verifyToken(
      data.refresh_token,
      false,
    );

    const isExpired = Date.now() >= payload.exp * 1000;

    if (isExpired) {
      throw new BaseException(
        ExceptionMessages.TOKEN_EXPIRED(language),
        HttpStatus.UNAUTHORIZED,
      );
    }

    await this.refreshTokenService.findToken(data.refresh_token);

    const user = (await this.userService.findOne({
      query: { _id: payload._id },
    })) as IReturnedUser;

    const { access_token, refresh_token } =
      await this.refreshTokenService.generateTokens({
        _id: user._id,
      });

    await this.refreshTokenService.saveTokens({
      refresh_token,
      userId: user._id,
      deviceId: data.deviceId,
      requestData: data.requestData,
    });

    return {
      user_id: user._id,
      access_token,
      refresh_token,
    };
  }

  // =================================
  // AUTHENTICATE USER
  // =================================
  async authenticateUser(
    data: AuthenticateUserDto,
  ): Promise<{ statusCode: number; token: string }> {
    const { password, type, idToken } = data;
    const { _id, email: userEmail } = this.cls.get<IReturnedUser>(
      CLS_KEYS.USER,
    );
    const lang = this.cls.get<LANGUAGE>(CLS_KEYS.LANG);

    const user = (await this.userService.findOne({
      query: { _id },
      fields: 'provider +password',
    })) as IReturnedUser;

    if (user.provider === PROVIDER.PASSWORD) {
      if (!password) {
        throw new BaseException(
          ExceptionMessages.INVALID_CREDENTIALS(lang),
          HttpStatus.BAD_REQUEST,
        );
      }

      const isPasswordMatch = await this.bcryptService.compare(
        password,
        user.password,
      );

      if (!isPasswordMatch) {
        throw new BaseException(
          ExceptionMessages.INVALID_CREDENTIALS(lang),
          HttpStatus.BAD_REQUEST,
        );
      }
    }

    if (user.provider === PROVIDER.GOOGLE) {
      if (!idToken) {
        throw new BaseException(
          ExceptionMessages.INVALID_CREDENTIALS(lang),
          HttpStatus.BAD_REQUEST,
        );
      }

      const ticket = await this.googleClient.verifyIdToken({
        idToken,
        audience: ENVIRONMENT_VARIABLES.GOOGLE_CLIENT_ID,
      });

      const payload = ticket.getPayload();

      const email = payload?.email;

      if (!email || email !== userEmail) {
        throw new BaseException(
          ExceptionMessages.INVALID_CREDENTIALS(lang),
          HttpStatus.CONFLICT,
        );
      }
    }

    await this.tokenService.removeMany({
      $and: [{ userId: _id }, { type }],
    });

    const { rawToken: token } = await this.tokenService.generateAndSaveToken({
      userId: _id,
      type,
      expiresInMs: 10 * 60 * 60,
    });

    return { statusCode: 200, token };
  }

  // =================================
  // RESTORE ACCOUNT
  // =================================
  async restoreAccount(data: RestoreAccountDto) {
    const language = this.cls.get<LANGUAGE>(CLS_KEYS.LANG);
    const { token, deviceId, requestData } = data;

    const hashedToken = this.tokenService.hashToken(token);

    const restoreToken = await this.tokenService.findToken({
      query: { token: hashedToken, type: TOKEN_TYPE.RESTORE_ACCOUNT },
    });

    if (restoreToken.expiresAt && restoreToken.expiresAt < new Date()) {
      throw new BaseException(
        ExceptionMessages.TOKEN_EXPIRED(language),
        HttpStatus.GONE,
      );
    }

    const decoded = this.jwtService.verify(token, {
      secret: ENVIRONMENT_VARIABLES.RESTORE_ACCOUNT_SECRET,
    });

    if (decoded.type !== TOKEN_TYPE.RESTORE_ACCOUNT)
      throw new BaseException(
        ExceptionMessages.CONFLICT_MESSAGE(language),
        HttpStatus.CONFLICT,
      );

    if (decoded.userId !== restoreToken.userId)
      throw new BaseException(
        ExceptionMessages.FORBIDDEN_MESSAGE(language),
        HttpStatus.FORBIDDEN,
      );

    await Promise.all([
      this.tokenService.removeTokensByUserId(decoded.userId),
      this.deletedUserService.restoreAccount(decoded.userId),
    ]);

    const { access_token, refresh_token } =
      await this.refreshTokenService.generateTokens({ _id: decoded.userId });

    const generalSettings = await this.accountService.findGeneralSettings({
      query: { userId: decoded.userId },
      fields: 'language theme startupPage',
    });

    await this.refreshTokenService.saveTokens({
      refresh_token,
      userId: decoded.userId,
      deviceId,
      requestData,
    });

    return {
      access_token,
      refresh_token,
      language: generalSettings.language,
      theme: generalSettings.theme,
      startupPage: generalSettings.startupPage,
    };
  }
}

```

## File: connectfy-auth/src/modules/auth/dto/verify.dto.ts
```typescript
import { FIELD_TYPE, VALIDATION_TYPE, FieldValidator } from 'connectfy-shared';
import { SignupDto } from './signup.dto';
import { IRequestData } from '@/src/internal-modules/request-helper/interfaces/request.interface';
import { RequestDataDto } from '@/src/internal-modules/request-helper/dto/request-data.dto';

export class VerifySignupDto {
  @FieldValidator({
    type: FIELD_TYPE.STRING,
    minLength: 6,
    maxLength: 6,
    matches: {
      regexp: /^\d+$/,
      message: {
        type: VALIDATION_TYPE.NUMBER,
      },
    },
  })
  code: string;

  @FieldValidator({
    type: FIELD_TYPE.STRING,
    minLength: 6,
    maxLength: 6,
    matches: {
      regexp: /^\d+$/,
      message: {
        type: VALIDATION_TYPE.NUMBER,
      },
    },
  })
  verifyCode: string;

  @FieldValidator({
    type: FIELD_TYPE.OBJECT,
    classType: SignupDto,
    validateNested: {},
  })
  unverifiedUser: SignupDto;

  @FieldValidator({
    type: FIELD_TYPE.UUID,
    uuidVersion: '4',
  })
  deviceId: string;

  @FieldValidator({
    type: FIELD_TYPE.OBJECT,
    classType: RequestDataDto,
  })
  requestData: IRequestData;
}

export class VerifyLoginDto {
  @FieldValidator({
    type: FIELD_TYPE.STRING,
    minLength: 6,
    maxLength: 6,
    matches: {
      regexp: /^\d+$/,
      message: {
        type: VALIDATION_TYPE.NUMBER,
      },
    },
  })
  twoFaCode: string;

  @FieldValidator({
    type: FIELD_TYPE.STRING,
    minLength: 6,
    maxLength: 6,
    matches: {
      regexp: /^\d+$/,
      message: {
        type: VALIDATION_TYPE.NUMBER,
      },
    },
  })
  code: string;

  @FieldValidator({
    type: FIELD_TYPE.UUID,
    uuidVersion: '4',
  })
  userId: string;

  @FieldValidator({
    type: FIELD_TYPE.UUID,
    uuidVersion: '4',
  })
  deviceId: string;

  @FieldValidator({
    type: FIELD_TYPE.OBJECT,
    classType: RequestDataDto,
  })
  requestData: IRequestData;
}

```

## File: connectfy-auth/src/modules/auth/dto/logout.dto.ts
```typescript
import { FIELD_TYPE, FieldValidator } from 'connectfy-shared';

export class LogoutDto {
  @FieldValidator({
    type: FIELD_TYPE.UUID,
    uuidVersion: '4',
  })
  deviceId: string;
}

```

## File: connectfy-auth/src/modules/auth/dto/signup.dto.ts
```typescript
import { IRequestData } from '@/src/internal-modules/request-helper/interfaces/request.interface';
import { RequestDataDto } from '@/src/internal-modules/request-helper/dto/request-data.dto';
import {
  FIELD_TYPE,
  VALIDATION_TYPE,
  FieldValidator,
  GENDER,
  THEME,
} from 'connectfy-shared';

export class SignupDto {
  @FieldValidator({
    type: FIELD_TYPE.STRING,
    maxLength: 50,
  })
  firstName: string;

  @FieldValidator({
    type: FIELD_TYPE.STRING,
    maxLength: 50,
  })
  lastName: string;

  @FieldValidator({
    type: FIELD_TYPE.STRING,
    minLength: 3,
    maxLength: 30,
    matches: {
      regexp: /^[^\s.,?()$:;"'{}\[\]=+&!\\|/<>`~@#№%^]+$/,
      message: {
        type: VALIDATION_TYPE.MISMATCH,
        params: { characters: '(.,?()$:;"\'{}[]-=+&!\\|/<>`~@#№%^)' },
      },
    },
  })
  username: string;

  @FieldValidator({
    type: FIELD_TYPE.EMAIL,
    maxLength: 254,
  })
  email: string;

  @FieldValidator({
    type: FIELD_TYPE.STRING,
    minLength: 8,
    maxLength: 30,
    matches: {
      regexp: /^(?=.*[a-z])(?=.*[A-Z])(?=.*\d)(?=.*[^A-Za-z0-9])\S{8,30}$/,
      message: {
        type: VALIDATION_TYPE.PASSWORD,
      },
    },
  })
  password: string;

  @FieldValidator({
    type: FIELD_TYPE.ENUM,
    enumObject: GENDER,
  })
  gender: GENDER;

  @FieldValidator({
    type: FIELD_TYPE.DATE,
  })
  birthdayDate: Date;

  @FieldValidator({
    type: FIELD_TYPE.ENUM,
    enumObject: THEME,
  })
  theme: THEME;
}

export class GoogleAuthSignupDto {
  @FieldValidator({
    type: FIELD_TYPE.STRING,
    maxLength: 2048,
  })
  idToken: string;

  @FieldValidator({
    type: FIELD_TYPE.STRING,
    minLength: 3,
    maxLength: 30,
    matches: {
      regexp: /^[^\s.,?()$:;"'{}\[\]=+&!\\|/<>`~@#№%^]+$/,
      message: {
        type: VALIDATION_TYPE.MISMATCH,
        params: { characters: '(.,?()$:;"\'{}[]-=+&!\\|/<>`~@#№%^)' },
      },
    },
  })
  username: string;

  @FieldValidator({
    type: FIELD_TYPE.ENUM,
    enumObject: GENDER,
  })
  gender: GENDER;

  @FieldValidator({
    type: FIELD_TYPE.DATE,
  })
  birthdayDate: Date;

  @FieldValidator({
    type: FIELD_TYPE.ENUM,
    enumObject: THEME,
  })
  theme: THEME;

  @FieldValidator({
    type: FIELD_TYPE.UUID,
    uuidVersion: '4',
  })
  deviceId: string;

  @FieldValidator({
    type: FIELD_TYPE.OBJECT,
    classType: RequestDataDto,
  })
  requestData: IRequestData;
}

```

## File: connectfy-auth/src/modules/auth/dto/validate-token.dto.ts
```typescript
import { FIELD_TYPE, TOKEN_TYPE, FieldValidator } from 'connectfy-shared';

export class ValidateTokenDto {
  @FieldValidator({
    type: FIELD_TYPE.STRING,
    maxLength: 1000,
  })
  token: string;

  @FieldValidator({
    type: FIELD_TYPE.ENUM,
    enumObject: TOKEN_TYPE,
  })
  type: TOKEN_TYPE;
}

```

## File: connectfy-auth/src/modules/auth/dto/authenticate-user.dto.ts
```typescript
import { FIELD_TYPE, TOKEN_TYPE, FieldValidator } from 'connectfy-shared';

export class AuthenticateUserDto {
  @FieldValidator({
    type: FIELD_TYPE.STRING,
    isOptional: true,
    validateIf: (obj) => obj.password || obj.type !== TOKEN_TYPE.CHANGE_EMAIL,
  })
  password: string | null;

  @FieldValidator({
    type: FIELD_TYPE.ENUM,
    enumObject: {
      DELETE_ACCOUNT: TOKEN_TYPE.DELETE_ACCOUNT,
      RESTORE_ACCOUNT: TOKEN_TYPE.RESTORE_ACCOUNT,
      CHANGE_USERNAME: TOKEN_TYPE.CHANGE_USERNAME,
      CHANGE_EMAIL: TOKEN_TYPE.CHANGE_EMAIL,
      CHANGE_PASSWORD: TOKEN_TYPE.CHANGE_PASSWORD,
      CHANGE_PHONE_NUMBER: TOKEN_TYPE.CHANGE_PHONE_NUMBER,
      DEACTIVATE_ACCOUNT: TOKEN_TYPE.DEACTIVATE_ACCOUNT,
      TWO_FACTOR: TOKEN_TYPE.TWO_FACTOR,
    },
  })
  type: TOKEN_TYPE;

  @FieldValidator({
    type: FIELD_TYPE.STRING,
    isOptional: true,
    maxLength: 1500,
    validateIf: (obj) => obj.idToken,
  })
  idToken: string | null;
}

```

## File: connectfy-auth/src/modules/auth/dto/forgot-password.dto.ts
```typescript
import {
  FIELD_TYPE,
  FieldValidator,
  FORGOT_PASSWORD_IDENTIFIER_TYPE,
} from 'connectfy-shared';

export class ForgotPasswordDto {
  @FieldValidator({
    type: FIELD_TYPE.ENUM,
    enumObject: FORGOT_PASSWORD_IDENTIFIER_TYPE,
  })
  identifierType: FORGOT_PASSWORD_IDENTIFIER_TYPE;

  @FieldValidator({
    type: FIELD_TYPE.STRING,
  })
  identifier: string;
}

```

## File: connectfy-auth/src/modules/auth/dto/reset-password.dto.ts
```typescript
import { FIELD_TYPE, VALIDATION_TYPE, FieldValidator } from 'connectfy-shared';

export class ResetPasswordDto {
  @FieldValidator({
    type: FIELD_TYPE.STRING,
    maxLength: 1000,
  })
  resetToken: string;

  @FieldValidator({
    type: FIELD_TYPE.STRING,
    minLength: 8,
    maxLength: 30,
    matches: {
      regexp: /^(?=.*[a-z])(?=.*[A-Z])(?=.*\d)(?=.*[^A-Za-z0-9])\S{8,30}$/,
      message: {
        type: VALIDATION_TYPE.PASSWORD,
      },
    },
  })
  password: string;

  @FieldValidator({
    type: FIELD_TYPE.STRING,
    minLength: 8,
    maxLength: 30,
  })
  confirmPassword: string;
}

```

## File: connectfy-auth/src/modules/auth/dto/restore-account.dto.ts
```typescript
import { RequestDataDto } from '@/src/internal-modules/request-helper/dto/request-data.dto';
import { IRequestData } from '@/src/internal-modules/request-helper/interfaces/request.interface';
import { FIELD_TYPE, FieldValidator } from 'connectfy-shared';

export class RestoreAccountDto {
  @FieldValidator({
    type: FIELD_TYPE.STRING,
    maxLength: 1000,
  })
  token: string;

  @FieldValidator({
    type: FIELD_TYPE.UUID,
    uuidVersion: '4',
  })
  deviceId: string;

  @FieldValidator({
    type: FIELD_TYPE.OBJECT,
    classType: RequestDataDto,
  })
  requestData: IRequestData;
}

```

## File: connectfy-auth/src/modules/auth/dto/login.dto.ts
```typescript
import { IRequestData } from '@/src/internal-modules/request-helper/interfaces/request.interface';
import { RequestDataDto } from '@/src/internal-modules/request-helper/dto/request-data.dto';
import { FIELD_TYPE, FieldValidator, IDENTIFIER_TYPE } from 'connectfy-shared';

export class LoginDto {
  @FieldValidator({
    type: FIELD_TYPE.ENUM,
    enumObject: IDENTIFIER_TYPE,
  })
  identifierType: IDENTIFIER_TYPE;

  @FieldValidator({
    type: FIELD_TYPE.STRING,
  })
  identifier: string;

  @FieldValidator({
    type: FIELD_TYPE.STRING,
  })
  password: string;

  @FieldValidator({
    type: FIELD_TYPE.UUID,
    uuidVersion: '4',
  })
  deviceId: string;

  @FieldValidator({
    type: FIELD_TYPE.OBJECT,
    classType: RequestDataDto,
  })
  requestData: IRequestData;
}

export class GoogleAuthLoginDto {
  @FieldValidator({
    type: FIELD_TYPE.STRING,
  })
  idToken: string;

  @FieldValidator({
    type: FIELD_TYPE.UUID,
    uuidVersion: '4',
  })
  deviceId: string;

  @FieldValidator({
    type: FIELD_TYPE.OBJECT,
    classType: RequestDataDto,
  })
  requestData: IRequestData;
}

```

## File: connectfy-auth/src/modules/auth/dto/nested/requestData.ts
```typescript

```

## File: connectfy-auth/src/modules/auth/interface/auth.interface.ts
```typescript
export interface IUserToken {
  user_id: string;
  access_token: string;
  refresh_token: string;
}
```

## File: connectfy-auth/src/modules/tokens/tokens.module.ts
```typescript
import { Module } from '@nestjs/common';
import { TokenModule } from './token/token.module';
import { RefreshTokenModule } from './refresh-token/refresh-token.module';

@Module({
  imports: [TokenModule, RefreshTokenModule],
})
export class TokensModule {}

```

## File: connectfy-auth/src/modules/tokens/refresh-token/refresh-token.service.ts
```typescript
import { HttpStatus, Injectable } from '@nestjs/common';
import {
  IGenerateRefreshToken,
  IRefreshTokenPayload,
  IReturnedRefreshToken,
  IUpdateRefreshToken,
} from './interface/refresh-token.interface';
import { JwtService } from '@nestjs/jwt';
import { RefreshTokenRepository } from './repo/refresh-token.repo';
import { RequestHelperService } from '@/src/internal-modules/request-helper/request-helper.service';
import {
  ExceptionMessages,
  LANGUAGE,
  EXPIRE_DATES,
  BaseException,
  CLS_KEYS,
  BaseFindDto,
} from 'connectfy-shared';
import { ClsService } from 'nestjs-cls';
import { IRequestData } from '@/src/internal-modules/request-helper/interfaces/request.interface';
import { ENVIRONMENT_VARIABLES } from '@/src/common/constants/environment-variables';

@Injectable()
export class RefreshTokenService {
  constructor(
    private readonly repo: RefreshTokenRepository,
    private readonly jwtService: JwtService,
    private readonly cls: ClsService,
    private readonly requestHelperService: RequestHelperService,
  ) {}

  async generateTokens(
    payload: IRefreshTokenPayload,
  ): Promise<IGenerateRefreshToken> {
    const accessSecretKey = ENVIRONMENT_VARIABLES.JWT_ACCESS_SECRET;
    const accessExpiry = ENVIRONMENT_VARIABLES.JWT_ACCESS_EXPIRES_IN;

    const refreshSecretKey = ENVIRONMENT_VARIABLES.JWT_REFRESH_SECRET;
    const refreshExpiry = ENVIRONMENT_VARIABLES.JWT_REFRESH_EXPIRES_IN;

    const access_token = await this.jwtService.signAsync(payload, {
      secret: accessSecretKey,
      expiresIn: accessExpiry,
    });

    const refresh_token = await this.jwtService.signAsync(payload, {
      secret: refreshSecretKey,
      expiresIn: refreshExpiry,
    });

    return { access_token, refresh_token };
  }

  async saveTokens(data: {
    userId: string;
    deviceId: string;
    refresh_token: string;
    requestData: IRequestData;
  }): Promise<IReturnedRefreshToken> {
    const { userId, deviceId, refresh_token, requestData } = data;

    const findToken = await this.repo.findOne({
      query: {
        $and: [{ userId }, { deviceId }],
      },
    });

    const deviceInfo =
      this.requestHelperService.parseDeviceInfoFromRequestData(requestData);

    const finalData: IUpdateRefreshToken = {
      userId,
      refresh_token,
      deviceId,
      deviceName: deviceInfo.deviceName,
      userAgent: deviceInfo.userAgent,
      browser: deviceInfo.browser,
      os: deviceInfo.os,
      platform: deviceInfo.platform,
      ipAddress: deviceInfo.ipAddress,
      country: deviceInfo.country,
      countryCode: deviceInfo.countryCode,
      city: deviceInfo.city,
      region: deviceInfo.region,
      longitude: deviceInfo.longitude,
      latitude: deviceInfo.latitude,
      timezone: deviceInfo.timezone,
      expiresAt: new Date(Date.now() + EXPIRE_DATES.TOKEN.ONE_MONTH),
    };

    if (findToken) {
      return await this.repo.update({ _id: findToken._id }, finalData);
    }

    return await this.repo.create(finalData);
  }

  async verifyToken(
    token: string,
    isAccessToken: boolean = true,
    ignoreExpiration: boolean = true,
  ): Promise<any> {
    return await this.jwtService.verifyAsync(token, {
      secret: isAccessToken
        ? ENVIRONMENT_VARIABLES.JWT_ACCESS_SECRET
        : ENVIRONMENT_VARIABLES.JWT_REFRESH_SECRET,
      ignoreExpiration,
    });
  }

  async removeTokenByUserId(
    userId: string,
  ): Promise<IReturnedRefreshToken | null> {
    return await this.repo.removeOne({ userId });
  }

  async findToken(refresh_token: string): Promise<IReturnedRefreshToken> {
    const language = this.cls.get<LANGUAGE>(CLS_KEYS.LANG);
    const res = await this.repo.findOne({ query: { refresh_token } });

    if (!res)
      throw new BaseException(
        ExceptionMessages.UNAUTHORIZED_MESSAGE(language),
        HttpStatus.UNAUTHORIZED,
        { navigate: true },
      );

    return res;
  }

  async findOne(options: BaseFindDto): Promise<IReturnedRefreshToken> {
    const res = await this.repo.findOne(options);

    if (!res)
      throw new BaseException(
        ExceptionMessages.NOT_FOUND_MESSAGE(LANGUAGE.EN),
        HttpStatus.NOT_FOUND,
      );

    return res;
  }
}

```

## File: connectfy-auth/src/modules/tokens/refresh-token/refresh-token.module.ts
```typescript
import { Module } from '@nestjs/common';
import { JwtModule } from '@nestjs/jwt';
import { MongooseModule } from '@nestjs/mongoose';
import { RefreshTokenSchema } from './entity/refresh-token.entity';
import { RefreshTokenService } from './refresh-token.service';
import { RefreshTokenRepository } from './repo/refresh-token.repo';
import { COLLECTIONS } from 'connectfy-shared';

@Module({
  imports: [
    MongooseModule.forFeature([
      {
        name: COLLECTIONS.AUTH.TOKEN.REFRESH_TOKENS,
        schema: RefreshTokenSchema,
      },
    ]),
    JwtModule,
  ],
  providers: [RefreshTokenService, RefreshTokenRepository],
  exports: [RefreshTokenService, RefreshTokenRepository],
})
export class RefreshTokenModule {}

```

## File: connectfy-auth/src/modules/tokens/refresh-token/interface/refresh-token.interface.ts
```typescript
import { DEVICE_TYPE } from 'connectfy-shared';

export interface IRefreshToken {
  _id: string;
  userId: string;
  refresh_token: string;

  deviceId: string | null;
  userAgent: string | null;
  deviceName: string | null;
  platform: DEVICE_TYPE;
  browser: string | null;
  os: string | null;

  ipAddress: string | null;
  country: string | null;
  countryCode: string | null;
  city: string | null;
  longitude: number | null;
  latitude: number | null;
  region: string | null;
  timezone: string | null;

  // Security
  lastUsedAt: Date;
  expiresAt: Date;
  isActive: boolean;

  // Metadata
  metadata: Record<string, any> | null;

  createdAt: Date;
  updatedAt: Date;
}

export interface IReturnedRefreshToken {
  _id: string;
  userId: string;
  refresh_token: string;

  deviceId: string | null;
  userAgent: string | null;
  deviceName: string | null;
  platform: DEVICE_TYPE;
  browser: string | null;
  os: string | null;

  ipAddress: string | null;
  country: string | null;
  countryCode: string | null;
  city: string | null;
  longitude: number | null;
  latitude: number | null;
  region: string | null;
  timezone: string | null;

  // Security
  lastUsedAt: Date;
  expiresAt: Date;
  isActive: boolean;

  // Metadata
  metadata: Record<string, any> | null;

  createdAt: Date;
  updatedAt: Date;
}

export interface IGenerateRefreshToken {
  access_token: string;
  refresh_token: string;
}

export interface IRefreshTokenPayload {
  [key: string]: string | number;
}

export interface ISaveRefreshToken {
  userId: string;
  refresh_token: string;
  deviceId?: string | null;
  userAgent?: string | null;
  deviceName?: string | null;
  platform?: string | null;
  browser?: string | null;
  os?: string | null;
  ipAddress?: string | null;
  country?: string | null;
  countryCode?: string | null;
  city?: string | null;
  longitude?: number | null;
  latitude?: number | null;
  region?: string | null;
  timezone?: string | null;
  expiresAt?: Date;
}

export interface IUpdateRefreshToken extends ISaveRefreshToken {}

```

## File: connectfy-auth/src/modules/tokens/refresh-token/repo/refresh-token.repo.ts
```typescript
import { Model } from 'mongoose';
import { Injectable } from '@nestjs/common';
import { InjectModel } from '@nestjs/mongoose';
import { RefreshTokenDocument } from '../entity/refresh-token.entity';
import {
  IReturnedRefreshToken,
  ISaveRefreshToken,
  IUpdateRefreshToken,
} from '../interface/refresh-token.interface';
import { COLLECTIONS, BaseRepository } from 'connectfy-shared';

@Injectable()
export class RefreshTokenRepository extends BaseRepository<
  RefreshTokenDocument,
  IReturnedRefreshToken,
  ISaveRefreshToken,
  IUpdateRefreshToken
> {
  constructor(
    @InjectModel(COLLECTIONS.AUTH.TOKEN.REFRESH_TOKENS)
    protected readonly model: Model<RefreshTokenDocument>,
  ) {
    super(model);
  }
}

```

## File: connectfy-auth/src/modules/tokens/refresh-token/entity/refresh-token.entity.ts
```typescript
import { v4 as uuid, validate } from 'uuid';
import { HydratedDocument } from 'mongoose';
import { Prop, Schema, SchemaFactory } from '@nestjs/mongoose';
import { IRefreshToken } from '../interface/refresh-token.interface';
import { t } from 'i18next';
import { LANGUAGE, DEVICE_TYPE, COLLECTIONS } from 'connectfy-shared';

@Schema({
  timestamps: true,
  collection: COLLECTIONS.AUTH.TOKEN.REFRESH_TOKENS,
  toJSON: {
    virtuals: true,
    versionKey: false,
    getters: true,
  },
  toObject: {
    virtuals: true,
    versionKey: false,
  },
  autoIndex: process.env.NODE_ENV !== 'production',
  minimize: false,
  strict: true,
  strictQuery: true,
})
export class RefreshTokenModel implements IRefreshToken {
  @Prop({
    type: String,
    default: () => uuid(),
    immutable: true,
  })
  _id: string;

  @Prop({
    type: String,
    required: [
      true,
      t('validation_messages.required', {
        lng: LANGUAGE.EN,
        field: 'userId',
      }),
    ],
    index: true,
    ref: COLLECTIONS.AUTH.USER.USERS,
    validate: {
      validator: function (value: string | null): boolean {
        return !!(value && validate(value));
      },
      message: t('validation_messages.uuid', {
        lng: LANGUAGE.EN,
        field: 'userId',
      }),
    },
  })
  userId: string;

  @Prop({
    type: String,
    required: [
      true,
      t('validation_messages.required', {
        lng: LANGUAGE.EN,
        field: 'refresh_token',
      }),
    ],
    unique: true,
    index: true,
    select: false, // Security: default query-lərdə gəlməsin
  })
  refresh_token: string;

  // ============================================
  // DEVICE & SESSION IDENTIFICATION
  // ============================================

  @Prop({
    type: String,
    required: false,
    default: null,
    index: true,
    trim: true,
  })
  deviceId: string | null;

  @Prop({
    type: String,
    required: false,
    default: null,
    maxlength: [
      500,
      t('validation_messages.max_length', {
        lng: LANGUAGE.EN,
        field: 'userAgent',
        length: 500,
      }),
    ],
    trim: true,
  })
  userAgent: string | null;

  @Prop({
    type: String,
    required: false,
    default: null,
    maxlength: [
      100,
      t('validation_messages.max_length', {
        lng: LANGUAGE.EN,
        field: 'deviceName',
        length: 100,
      }),
    ],
    trim: true,
  })
  deviceName: string | null;

  @Prop({
    type: String,
    enum: {
      values: Object.values(DEVICE_TYPE),
      message: t('validation_messages.enum', {
        lng: LANGUAGE.EN,
        field: 'platform',
        values: Object.values(DEVICE_TYPE),
      }),
    },
    default: DEVICE_TYPE.UNKNOWN,
    index: true,
  })
  platform: DEVICE_TYPE;

  @Prop({
    type: String,
    required: false,
    default: null,
    maxlength: [50, 'Browser name too long'],
    trim: true,
  })
  browser: string | null;

  @Prop({
    type: String,
    required: false,
    default: null,
    maxlength: [50, 'OS name too long'],
    trim: true,
  })
  os: string | null;

  // ============================================
  // NETWORK & LOCATION
  // ============================================

  @Prop({
    type: String,
    required: false,
    default: null,
    index: true,
    trim: true,
    validate: {
      validator: function (value: string | null): boolean {
        if (!value) return true;
        // IPv4 və IPv6 validation
        const ipv4Regex = /^(\d{1,3}\.){3}\d{1,3}$/;
        const ipv6Regex = /^([0-9a-fA-F]{0,4}:){7}[0-9a-fA-F]{0,4}$/;
        return ipv4Regex.test(value) || ipv6Regex.test(value);
      },
      message: 'Invalid IP address format',
    },
  })
  ipAddress: string | null;

  @Prop({
    type: String,
    required: false,
    default: null,
    maxlength: [100, 'Country name too long'],
    trim: true,
    index: true,
  })
  country: string | null;

  @Prop({
    type: String,
    required: false,
    default: null,
    maxlength: [3, 'Country code too long'],
    uppercase: true,
    trim: true,
  })
  countryCode: string | null;

  @Prop({
    type: String,
    required: false,
    default: null,
    maxlength: [100, 'City name too long'],
    trim: true,
  })
  city: string | null;

  @Prop({
    type: Number,
    required: false,
    default: null,
    min: [-180, 'Longitude must be >= -180'],
    max: [180, 'Longitude must be <= 180'],
  })
  longitude: number | null;

  @Prop({
    type: Number,
    required: false,
    default: null,
    min: [-90, 'Latitude must be >= -90'],
    max: [90, 'Latitude must be <= 90'],
  })
  latitude: number | null;

  @Prop({
    type: String,
    required: false,
    default: null,
    maxlength: [100, 'Region name too long'],
    trim: true,
  })
  region: string | null;

  @Prop({
    type: String,
    required: false,
    default: null,
    maxlength: [50, 'Timezone too long'],
    trim: true,
  })
  timezone: string | null;

  // ============================================
  // SECURITY & TRACKING
  // ============================================

  @Prop({
    type: Date,
    required: [
      true,
      t('validation_messages.required', {
        lng: LANGUAGE.EN,
        field: 'lastUsedAt',
      }),
    ],
    default: Date.now,
    index: true,
  })
  lastUsedAt: Date;

  @Prop({
    type: Date,
    required: [
      true,
      t('validation_messages.required', {
        lng: LANGUAGE.EN,
        field: 'expiresAt',
      }),
    ],
    index: true,
  })
  expiresAt: Date;

  @Prop({
    type: Boolean,
    required: [
      true,
      t('validation_messages.required', {
        lng: LANGUAGE.EN,
        field: 'isActive',
      }),
    ],
    default: true,
    index: true,
  })
  isActive: boolean;

  @Prop({
    type: String,
    required: false,
    default: null,
    select: false,
  })
  fingerprint: string | null;

  // ============================================
  // METADATA
  // ============================================

  @Prop({
    type: Object,
    required: false,
    default: null,
  })
  metadata: Record<string, any> | null;

  createdAt: Date;
  updatedAt: Date;
}

export const RefreshTokenSchema =
  SchemaFactory.createForClass(RefreshTokenModel);

// ================================================
// INDEXES - Performance optimization
// ================================================

// Compound indexes (tez-tez query olunan kombinasiyalar)
RefreshTokenSchema.index({ userId: 1, isActive: 1 });
RefreshTokenSchema.index({ userId: 1, deviceId: 1 });
RefreshTokenSchema.index({ userId: 1, platform: 1 });
// ================================================
// MIDDLEWARE / HOOKS
// ================================================

// Pre-update hook
RefreshTokenSchema.pre('findOneAndUpdate', function (next) {
  const update = this.getUpdate() as any;

  // expiresAt validation
  if (update.$set?.expiresAt && update.$set.expiresAt < new Date()) {
    return next(new Error('expiresAt cannot be in the past'));
  }

  // Token istifadə ediləndə lastUsedAt yenilə
  if (update.$set && !update.$set.lastUsedAt) {
    update.$set.lastUsedAt = new Date();
  }

  this.set({ updatedAt: new Date() });
  next();
});

export type RefreshTokenDocument = HydratedDocument<RefreshTokenModel>;

```

## File: connectfy-auth/src/modules/tokens/token/token.controller.ts
```typescript
import { Controller } from '@nestjs/common';
import { TokenService } from './token.service';

@Controller('token')
export class TokenController {
  constructor(private readonly service: TokenService) {}
}

```

## File: connectfy-auth/src/modules/tokens/token/token.service.ts
```typescript
import {
  ExceptionMessages,
  BaseException,
  IRemoveAllResponse,
  CLS_KEYS,
  LANGUAGE,
  TOKEN_TYPE,
} from 'connectfy-shared';
import { HttpStatus, Injectable } from '@nestjs/common';
import { TokenRepository } from './repo/token.repo';
import { RemoveTokenDto, RemoveAllTokensDto } from './dto/remove.token.dto';
import { FindTokenDto } from './dto/find.token.dto';
import * as crypto from 'crypto';
import { JwtService } from '@nestjs/jwt';
import { ClsService } from 'nestjs-cls';
import { IReturnedToken } from '@modules/tokens/token/interface/token.interface';

@Injectable()
export class TokenService {
  constructor(
    private readonly repo: TokenRepository,
    private readonly jwtService: JwtService,
    private readonly cls: ClsService,
  ) {}

  hashToken(token: string): string {
    return crypto.createHash('sha256').update(token).digest('hex');
  }

  async generateAndSaveToken(options: {
    userId: string;
    type: TOKEN_TYPE;
    expiresInMs: number;
  }): Promise<{ rawToken: string; hashedToken: string }> {
    const { userId, type, expiresInMs } = options;

    const rawToken = crypto.randomBytes(128).toString('hex');
    const hashedToken = this.hashToken(rawToken);
    const tokenExpiry = new Date(Date.now() + expiresInMs);

    await this.repo.create({
      userId,
      token: hashedToken,
      type,
      expiresAt: tokenExpiry,
    });

    return { rawToken, hashedToken };
  }

  async generateAndSaveJwtToken(options: {
    userId: string;
    type: TOKEN_TYPE;
    secret: string;
    jwtExp: string;
    tokenExp: number;
    payload?: Record<string, any>;
  }): Promise<string> {
    const { userId, type, secret, jwtExp, tokenExp, payload } = options;

    const token = this.jwtService.sign(
      { userId, type, ...payload },
      {
        secret: secret,
        expiresIn: jwtExp,
      },
    );

    const hashedToken = this.hashToken(token);
    const expiresAt = new Date(Date.now() + tokenExp);

    await this.repo.create({
      userId,
      type,
      token: hashedToken,
      expiresAt,
    });

    return token;
  }

  async findToken(option: FindTokenDto): Promise<IReturnedToken> {
    const lang = this.cls.get<LANGUAGE>(CLS_KEYS.LANG);

    const token = await this.repo.findOne(option);

    if (!token)
      throw new BaseException(
        ExceptionMessages.NOT_FOUND_MESSAGE(lang),
        HttpStatus.NOT_FOUND,
      );

    return token;
  }

  async findAllTokens(options: FindTokenDto): Promise<IReturnedToken[]> {
    return await this.repo.findAll(options);
  }

  async removeToken(data: RemoveTokenDto): Promise<IReturnedToken> {
    const { _id, _lang } = data;
    const lang = this.cls.get<LANGUAGE>(CLS_KEYS.LANG) || _lang;

    const res = await this.repo.remove({ _id });

    if (!res)
      throw new BaseException(
        ExceptionMessages.NOT_FOUND_MESSAGE(lang),
        HttpStatus.NOT_FOUND,
      );

    return res;
  }

  async removeTokens({
    _ids,
  }: RemoveAllTokensDto): Promise<IRemoveAllResponse> {
    const allTokens = await this.repo.findAll({
      query: { _id: { $in: _ids } },
    });

    if (!allTokens.length)
      return { deletedCount: 0, notDeleted: [], deletedIds: [] };

    const removableIds = allTokens.map((token) => token._id);

    await this.repo.removeMany({ _ids: removableIds });

    return {
      deletedCount: removableIds.length,
      notDeleted: [],
      deletedIds: removableIds,
    };
  }

  async removeTokensByUserId(userId: string): Promise<IRemoveAllResponse> {
    const allTokens = await this.repo.findMany({
      query: {
        $and: [
          { userId: { $in: userId } },
          { type: { $ne: TOKEN_TYPE.RESTORE_ACCOUNT } },
        ],
      },
    });

    if (!allTokens.length)
      return { deletedCount: 0, notDeleted: [], deletedIds: [] };

    const removableIds = allTokens.map((token) => token._id);

    await this.repo.removeMany({ _ids: removableIds });

    return {
      deletedCount: removableIds.length,
      notDeleted: [],
      deletedIds: removableIds,
    };
  }

  async removeMany(query: Record<string, any>): Promise<IRemoveAllResponse> {
    const allTokens = await this.repo.findMany({
      query,
    });

    if (!allTokens.length)
      return { deletedCount: 0, notDeleted: [], deletedIds: [] };

    const removableIds = allTokens.map((token) => token._id);

    await this.repo.removeMany({ _ids: removableIds });

    return {
      deletedCount: removableIds.length,
      notDeleted: [],
      deletedIds: removableIds,
    };
  }

  async remove(query: Record<string, any>) {
    const lang = this.cls.get<LANGUAGE>(CLS_KEYS.LANG);

    const res = await this.repo.removeOne(query);

    if (!res)
      throw new BaseException(
        ExceptionMessages.NOT_FOUND_MESSAGE(lang ?? LANGUAGE.EN),
        HttpStatus.NOT_FOUND,
      );

    return res;
  }
}

```

## File: connectfy-auth/src/modules/tokens/token/token.module.ts
```typescript
import { Module } from '@nestjs/common';
import { TokenService } from './token.service';
import { TokenController } from './token.controller';
import { MongooseModule } from '@nestjs/mongoose';
import { TokenSchema } from './entity/token.entity';
import { TokenRepository } from './repo/token.repo';
import { JwtModule } from '@nestjs/jwt';
import { COLLECTIONS } from 'connectfy-shared';

@Module({
  imports: [
    MongooseModule.forFeature([
      { name: COLLECTIONS.AUTH.TOKEN.TOKENS, schema: TokenSchema },
    ]),
    JwtModule,
  ],
  controllers: [TokenController],
  providers: [TokenService, TokenRepository],
  exports: [TokenService, TokenRepository],
})
export class TokenModule {}

```

## File: connectfy-auth/src/modules/tokens/token/dto/find.token.dto.ts
```typescript
import {
  BaseFindDto,
  FieldValidator,
  FIELD_TYPE,
  LANGUAGE,
} from 'connectfy-shared';

export class FindTokenDto extends BaseFindDto {
  @FieldValidator({
    type: FIELD_TYPE.ENUM,
    isOptional: true,
    enumObject: LANGUAGE,
  })
  _lang?: LANGUAGE;
}

```

## File: connectfy-auth/src/modules/tokens/token/dto/edit.token.dto.ts
```typescript
import { PartialType } from '@nestjs/mapped-types';
import { BaseTokenDto } from './base.token.dto';
import { FieldValidator, FIELD_TYPE } from 'connectfy-shared';

export class UpdateTokenDto extends PartialType(BaseTokenDto) {
  @FieldValidator({
    type: FIELD_TYPE.UUID,
  })
  _id: string;

  @FieldValidator({
    type: FIELD_TYPE.BOOLEAN,
    isOptional: true,
  })
  isUsed?: boolean;
}

```

## File: connectfy-auth/src/modules/tokens/token/dto/base.token.dto.ts
```typescript
import { FIELD_TYPE, TOKEN_TYPE, FieldValidator } from 'connectfy-shared';

export class BaseTokenDto {
  @FieldValidator({
    type: FIELD_TYPE.UUID,
  })
  userId: string;

  @FieldValidator({
    type: FIELD_TYPE.STRING,
  })
  token: string;

  @FieldValidator({
    type: FIELD_TYPE.ENUM,
    enumObject: TOKEN_TYPE,
  })
  type: TOKEN_TYPE;

  @FieldValidator({
    type: FIELD_TYPE.DATE,
  })
  expiresAt: Date;
}

```

## File: connectfy-auth/src/modules/tokens/token/dto/add.token.dto.ts
```typescript
import { BaseTokenDto } from './base.token.dto';

export class AddTokenDto extends BaseTokenDto {}

```

## File: connectfy-auth/src/modules/tokens/token/dto/remove.token.dto.ts
```typescript
import {
  BaseRemoveAllDto,
  BaseRemoveDto,
  FieldValidator,
  FIELD_TYPE,
  LANGUAGE,
} from 'connectfy-shared';

export class RemoveTokenDto extends BaseRemoveDto {
  @FieldValidator({
    type: FIELD_TYPE.ENUM,
    isOptional: true,
    enumObject: LANGUAGE,
  })
  _lang?: LANGUAGE;
}

export class RemoveAllTokensDto extends BaseRemoveAllDto {}

```

## File: connectfy-auth/src/modules/tokens/token/interface/token.interface.ts
```typescript
import { TOKEN_TYPE } from 'connectfy-shared';

export interface IToken {
  _id: string;
  userId: string;
  token: string;
  type: TOKEN_TYPE;
  expiresAt: Date;
  isUsed: boolean;
  createdAt: Date;
  updatedAt: Date;
}

export interface IReturnedToken {
  _id: string;
  userId: string | Record<string, any>;
  token: string;
  type: TOKEN_TYPE;
  expiresAt: Date;
  isUsed: boolean;
  createdAt: Date;
  updatedAt: Date;
}

```

## File: connectfy-auth/src/modules/tokens/token/repo/token.repo.ts
```typescript
import { Model } from 'mongoose';
import { InjectModel } from '@nestjs/mongoose';
import { AddTokenDto } from '../dto/add.token.dto';
import { UpdateTokenDto } from '../dto/edit.token.dto';
import { TokenDocument } from '../entity/token.entity';
import { COLLECTIONS, BaseRepository } from 'connectfy-shared';
import { IReturnedToken } from '@modules/tokens/token/interface/token.interface';

export class TokenRepository extends BaseRepository<
  TokenDocument,
  IReturnedToken,
  AddTokenDto,
  UpdateTokenDto
> {
  constructor(
    @InjectModel(COLLECTIONS.AUTH.TOKEN.TOKENS)
    protected readonly model: Model<TokenDocument>,
  ) {
    super(model);
  }
}

```

## File: connectfy-auth/src/modules/tokens/token/entity/token.entity.ts
```typescript
import { v4 as uuid, validate } from 'uuid';
import { HydratedDocument } from 'mongoose';
import { IToken } from '../interface/token.interface';
import { Prop, Schema, SchemaFactory } from '@nestjs/mongoose';
import { TOKEN_TYPE, LANGUAGE, COLLECTIONS } from 'connectfy-shared';
import { t } from 'i18next';

@Schema({
  timestamps: true,
  collection: COLLECTIONS.AUTH.TOKEN.TOKENS,
  toJSON: { virtuals: true, versionKey: false, getters: true },
  toObject: { virtuals: true, versionKey: false },
  autoIndex: process.env.NODE_ENV !== 'production',
  minimize: false,
  strict: true,
  strictQuery: true,
})
export class TokenModel implements IToken {
  @Prop({
    type: String,
    default: () => uuid(),
    immutable: true,
  })
  _id: string;

  @Prop({
    type: String,
    required: [
      true,
      t('validation_messages.required', {
        lng: LANGUAGE.EN,
        field: 'userId',
      }),
    ],
    index: true,
    ref: COLLECTIONS.AUTH.USER.USERS,
    validate: {
      validator: function (value: string | null): boolean {
        return !!(value && validate(value));
      },
      message: t('validation_messages.uuid', {
        lng: LANGUAGE.EN,
        field: 'userId',
      }),
    },
  })
  userId: string;

  @Prop({
    type: String,
    required: [
      true,
      t('validation_messages.required', {
        lng: LANGUAGE.EN,
        field: 'token',
      }),
    ],
    unique: true,
    index: true,
    // select: false, // Security: default query-lərdə token gəlməsin
  })
  token: string;

  @Prop({
    type: String,
    required: [
      true,
      t('validation_messages.required', {
        lng: LANGUAGE.EN,
        field: 'type',
      }),
    ],
    enum: {
      values: Object.values(TOKEN_TYPE),
      message: t('validation_messages.enum', {
        lng: LANGUAGE.EN,
        field: 'type',
        values: Object.values(TOKEN_TYPE),
      }),
    },
    default: TOKEN_TYPE.PASSWORD_RESET,
    index: true,
  })
  type: TOKEN_TYPE;

  @Prop({
    type: Date,
    required: [
      true,
      t('validation_messages.required', {
        lng: LANGUAGE.EN,
        field: 'expiresAt',
      }),
    ],
    index: true,
  })
  expiresAt: Date;

  @Prop({
    type: Boolean,
    required: [
      true,
      t('validation_messages.required', {
        lng: LANGUAGE.EN,
        field: 'isUsed',
      }),
    ],
    default: false,
    index: true,
  })
  isUsed: boolean;

  createdAt: Date;
  updatedAt: Date;
}

export const TokenSchema = SchemaFactory.createForClass(TokenModel);

// ================================================
// INDEXES - Performance optimization
// ================================================

// Compound indexes (tez-tez query olunan kombinasiyalar)
TokenSchema.index({ userId: 1, type: 1 });
TokenSchema.index({ userId: 1, isUsed: 1 });
TokenSchema.index({ token: 1, type: 1 });
TokenSchema.index({ type: 1, isUsed: 1 });
TokenSchema.index({ expiresAt: 1, isUsed: 1 });

TokenSchema.index({ createdAt: -1 });
TokenSchema.index({ updatedAt: -1 });

// Compound index for cleanup queries
TokenSchema.index({ isUsed: 1, expiresAt: 1 });

export type TokenDocument = HydratedDocument<TokenModel>;

```

## File: connectfy-auth/src/modules/users/users.module.ts
```typescript
import { Module } from '@nestjs/common';
import { UserModule } from './user/user.module';
import { BannedUserModule } from './banned-user/banned-user.module';
import { DeletedUserModule } from './deleted-user/deleted-user.module';
import { DeactivatedUsersModule } from './deactivated-users/deactivated-users.module';

@Module({
  imports: [
    UserModule,
    BannedUserModule,
    DeletedUserModule,
    DeactivatedUsersModule,
  ],
})
export class UsersModule {}

```

## File: connectfy-auth/src/modules/users/deleted-user/deleted-user.service.ts
```typescript
import { forwardRef, HttpStatus, Inject, Injectable } from '@nestjs/common';
import { DeletedUserRepository } from './repo/deleted-user.repo';
import { AddDeletedUserDto } from './dto/add.deleted-user.dto';
import { IReturnedDeletedUser } from './interface/deleted-user.interface';
import { RemoveDeletedUserDto } from './dto/remove.deleted-user.dto';
import {
  BaseException,
  CLS_KEYS,
  ExceptionMessages,
  LANGUAGE,
  USER_STATUS,
} from 'connectfy-shared';
import { ClsService } from 'nestjs-cls';
import { UserService } from '../user/user.service';

@Injectable()
export class DeletedUserService {
  constructor(
    @Inject(forwardRef(() => UserService))
    private readonly userService: UserService,

    private readonly repo: DeletedUserRepository,
    private readonly cls: ClsService,
  ) {}

  async create(data: AddDeletedUserDto): Promise<void> {
    const userData = {
      _id: data.userId,
      status: USER_STATUS.DELETED,
    };

    await Promise.all([
      this.repo.create(data),
      this.userService.edit(userData),
    ]);
  }

  async restoreAccount(userId: string): Promise<void> {
    const deletedAccount = await this.repo.findOne({ query: { userId } });
    const language = this.cls.get<LANGUAGE>(CLS_KEYS.LANG);

    if (!deletedAccount) {
      throw new BaseException(
        ExceptionMessages.NOT_FOUND_MESSAGE(language),
        HttpStatus.NOT_FOUND,
      );
    }

    const isUserExist = await this.userService.existByField({ _id: userId });

    if (!isUserExist) {
      throw new BaseException(
        ExceptionMessages.NOT_FOUND_MESSAGE(language),
        HttpStatus.NOT_FOUND,
      );
    }

    const userData = {
      _id: userId,
      status: USER_STATUS.ACTIVE,
    };

    await Promise.all([
      await this.repo.remove({ _id: deletedAccount._id }),
      this.userService.edit(userData),
    ]);
  }

  async findOne(
    query: Record<string, any>,
  ): Promise<IReturnedDeletedUser | null> {
    const res = await this.repo.findOne(query);
    return res;
  }
}

```

## File: connectfy-auth/src/modules/users/deleted-user/deleted-user.controller.ts
```typescript
import { Controller } from '@nestjs/common';
import { DeletedUserService } from './deleted-user.service';

@Controller('')
export class DeletedUserController {
  constructor(private readonly service: DeletedUserService) {}
}

```

## File: connectfy-auth/src/modules/users/deleted-user/deleted-user.module.ts
```typescript
import { forwardRef, Module } from '@nestjs/common';
import { DeletedUserService } from './deleted-user.service';
import { DeletedUserController } from './deleted-user.controller';
import { MongooseModule } from '@nestjs/mongoose';
import { DeletedUserSchema } from './entity/deleted-user.entity';
import { DeletedUserRepository } from './repo/deleted-user.repo';
import { COLLECTIONS } from 'connectfy-shared';
import { UserModule } from '../user/user.module';

@Module({
  imports: [
    MongooseModule.forFeature([
      { name: COLLECTIONS.AUTH.USER.DELETED, schema: DeletedUserSchema },
    ]),
    forwardRef(() => UserModule),
  ],
  controllers: [DeletedUserController],
  providers: [DeletedUserService, DeletedUserRepository],
  exports: [DeletedUserService, DeletedUserRepository],
})
export class DeletedUserModule {}

```

## File: connectfy-auth/src/modules/users/deleted-user/dto/add.deleted-user.dto.ts
```typescript
import { BaseUserDto } from './base.deleted-user.dto';

export class AddDeletedUserDto extends BaseUserDto {}

```

## File: connectfy-auth/src/modules/users/deleted-user/dto/base.deleted-user.dto.ts
```typescript
import {
  DELETE_REASON,
  DELETE_REASON_CODE,
  FIELD_TYPE,
  FieldValidator,
} from 'connectfy-shared';

export class BaseUserDto {
  @FieldValidator({
    type: FIELD_TYPE.UUID,
  })
  userId: string;

  @FieldValidator({
    type: FIELD_TYPE.ENUM,
    enumObject: DELETE_REASON,
  })
  reason: DELETE_REASON;

  @FieldValidator({
    type: FIELD_TYPE.ENUM,
    isOptional: true,
    enumObject: DELETE_REASON_CODE,
  })
  reasonCode: DELETE_REASON_CODE | null;

  @FieldValidator({
    type: FIELD_TYPE.STRING,
    isOptional: true,
    maxLength: 200,
  })
  reasonDescription: string | null;
}

```

## File: connectfy-auth/src/modules/users/deleted-user/dto/find.deleted-user.dto.ts
```typescript
import { BaseFindDto } from 'connectfy-shared';

export class FindDeletedUserDto extends BaseFindDto {}

```

## File: connectfy-auth/src/modules/users/deleted-user/dto/remove.deleted-user.dto.ts
```typescript
import { BaseRemoveDto } from 'connectfy-shared';

export class RemoveDeletedUserDto extends BaseRemoveDto {}

```

## File: connectfy-auth/src/modules/users/deleted-user/interface/deleted-user.interface.ts
```typescript
import { DELETE_REASON, DELETE_REASON_CODE } from 'connectfy-shared';

export interface IDeletedUser {
  _id: string;
  userId: string;
  deletedAt: Date;
  reason: DELETE_REASON;
  reasonCode: DELETE_REASON_CODE | null;
  reasonDescription: string | null;
  createdAt: Date;
  updatedAt: Date;
}

export interface IReturnedDeletedUser {
  _id: string;
  userId: string;
  deletedAt: Date;
  reason: DELETE_REASON;
  reasonCode: DELETE_REASON_CODE | null;
  reasonDescription: string | null;
  createdAt: Date;
  updatedAt: Date;
}

```

## File: connectfy-auth/src/modules/users/deleted-user/repo/deleted-user.repo.ts
```typescript
import { Injectable } from '@nestjs/common';
import { InjectModel } from '@nestjs/mongoose';
import { Model } from 'mongoose';
import { DeletedUserDocument } from '../entity/deleted-user.entity';
import { AddDeletedUserDto } from '../dto/add.deleted-user.dto';
import { COLLECTIONS, BaseRepository } from 'connectfy-shared';
import { IReturnedDeletedUser } from '@modules/users/deleted-user/interface/deleted-user.interface';

@Injectable()
export class DeletedUserRepository extends BaseRepository<
  DeletedUserDocument,
  IReturnedDeletedUser,
  AddDeletedUserDto
> {
  constructor(
    @InjectModel(COLLECTIONS.AUTH.USER.DELETED)
    protected readonly model: Model<DeletedUserDocument>,
  ) {
    super(model);
  }
}

```

## File: connectfy-auth/src/modules/users/deleted-user/entity/deleted-user.entity.ts
```typescript
import { v4 as uuid, validate } from 'uuid';
import { HydratedDocument } from 'mongoose';
import { Prop, Schema, SchemaFactory } from '@nestjs/mongoose';
import { IDeletedUser } from '../interface/deleted-user.interface';
import {
  DELETE_REASON,
  LANGUAGE,
  COLLECTIONS,
  DELETE_REASON_CODE,
} from 'connectfy-shared';
import { t } from 'i18next';

@Schema({
  timestamps: true,
  collection: COLLECTIONS.AUTH.USER.DELETED,
  toJSON: { virtuals: true, versionKey: false, getters: true },
  toObject: { virtuals: true, versionKey: false },
  autoIndex: process.env.NODE_ENV !== 'production',
  minimize: false,
  strict: true,
  strictQuery: true,
})
export class DeletedUserModel implements IDeletedUser {
  @Prop({
    type: String,
    default: () => uuid(),
    immutable: true,
  })
  _id: string;

  @Prop({
    type: String,
    required: [
      true,
      t('validation_messages.required', {
        lng: LANGUAGE.EN,
        field: 'userId',
      }),
    ],
    unique: true,
    index: true,
    immutable: true,
    ref: COLLECTIONS.AUTH.USER.USERS,
    validate: {
      validator: function (value: string | null): boolean {
        return !!(value && validate(value));
      },
      message: t('validation_messages.uuid', {
        lng: LANGUAGE.EN,
        field: 'userId',
      }),
    },
  })
  userId: string;

  @Prop({
    type: Date,
    required: [
      true,
      t('validation_messages.required', {
        lng: LANGUAGE.EN,
        field: 'deletedAt',
      }),
    ],
    default: Date.now,
    index: true,
    immutable: true,
  })
  deletedAt: Date;

  @Prop({
    type: String,
    required: [
      true,
      t('validation_messages.required', {
        lng: LANGUAGE.EN,
        field: 'reason',
      }),
    ],
    enum: {
      values: Object.values(DELETE_REASON),
      message: t('validation_messages.enum', {
        lng: LANGUAGE.EN,
        field: 'reason',
        values: Object.values(DELETE_REASON),
      }),
    },
    index: true,
    immutable: true,
  })
  reason: DELETE_REASON;

  @Prop({
    type: String,
    enum: DELETE_REASON_CODE,
    required: false,
    default: null,
    validate: {
      validator: function (
        this: DeletedUserModel,
        value: DELETE_REASON_CODE | null,
      ): boolean {
        if (this.reason === DELETE_REASON.USER_REQUEST) {
          return !!(value && Object.values(DELETE_REASON_CODE).includes(value));
        }
        return true;
      },
      message: t('validation_messages.required', {
        lng: LANGUAGE.EN,
        field: 'reasonCode',
      }),
    },
    trim: true,
    immutable: true,
  })
  reasonCode: DELETE_REASON_CODE | null;

  @Prop({
    type: String,
    required: false,
    default: null,
    maxlength: [
      200,
      t('validation_messages.max_length', {
        lng: LANGUAGE.EN,
        field: 'reasonDescription',
        length: 200,
      }),
    ],
    trim: true,
    immutable: true,
  })
  reasonDescription: string | null;

  createdAt: Date;
  updatedAt: Date;
}

export const DeletedUserSchema = SchemaFactory.createForClass(DeletedUserModel);

// ================================================
// INDEXES - Performance optimization
// ================================================
DeletedUserSchema.index({ createdAt: -1 });

export type DeletedUserDocument = HydratedDocument<DeletedUserModel>;

```

## File: connectfy-auth/src/modules/users/user/user.controller.ts
```typescript
import { Controller } from '@nestjs/common';
import { UserService } from './user.service';
import { MessagePattern, Payload, Transport } from '@nestjs/microservices';
import { ChangeUsernameDto } from './dto/change-username.dto';
import { ChangeEmailDto, VerifyEmailChangeDto } from './dto/change-email.dto';
import { ChangePasswordDto } from './dto/change-password.dto';
import { ChangePhoneNumberDto } from './dto/change-phone-number.dto';
import { CheckUniqueDto } from './dto/check-unique.dto';
import { TwoFactorDto } from './dto/two-factor.dto';
import { DeleteAccountDto } from './dto/delete-account.dto';
import { RestoreAccountDto } from '../../auth/dto/restore-account.dto';
import { DeactivateAccountDto } from './dto/deactivate-account.dto';

@Controller('')
export class UserController {
  constructor(private readonly service: UserService) {}

  @MessagePattern('user/change-username', Transport.TCP)
  async changeUsername(@Payload() data: ChangeUsernameDto) {
    return this.service.changeUsername(data);
  }

  @MessagePattern('user/change-email', Transport.TCP)
  async changeEmail(@Payload() data: ChangeEmailDto) {
    return this.service.changeEmail(data);
  }

  @MessagePattern('user/change-email/verify', Transport.TCP)
  async verifyEmailChange(@Payload() data: VerifyEmailChangeDto) {
    return this.service.verifyEmailChange(data);
  }

  @MessagePattern('user/change-password', Transport.TCP)
  async changePassword(@Payload() data: ChangePasswordDto) {
    return this.service.changePassword(data);
  }

  @MessagePattern('user/change-phone-number', Transport.TCP)
  async changePhoneNumber(@Payload() data: ChangePhoneNumberDto) {
    return this.service.changePhoneNumber(data);
  }

  @MessagePattern('user/check-unique', Transport.TCP)
  async checkUnique(@Payload() data: CheckUniqueDto) {
    return this.service.checkUnique(data);
  }

  @MessagePattern('user/two-factor', Transport.TCP)
  async updateTwoFactorAuth(@Payload() data: TwoFactorDto) {
    return this.service.updateTwoFactorAuth(data);
  }

  @MessagePattern('user/delete-account', Transport.TCP)
  async deleteAccount(@Payload() data: DeleteAccountDto) {
    return this.service.deleteAccount(data);
  }

  @MessagePattern('user/deactivate-account', Transport.TCP)
  async deactivateAccount(@Payload() data: DeactivateAccountDto) {
    return this.service.deactivateAccount(data);
  }
}

```

## File: connectfy-auth/src/modules/users/user/user.module.ts
```typescript
import { forwardRef, Module } from '@nestjs/common';
import { UserService } from './user.service';
import { UserController } from './user.controller';
import { MongooseModule } from '@nestjs/mongoose';
import { UserSchema } from './entity/user.entity';
import { UserRepository } from './repo/user.repo';
import { DeletedUserModule } from '../deleted-user/deleted-user.module';
import { TokenModule } from '../../tokens/token/token.module';
import { JwtModule } from '@nestjs/jwt';
import { RefreshTokenModule } from '../../tokens/refresh-token/refresh-token.module';
import { COLLECTIONS } from 'connectfy-shared';
import { DeactivatedUsersModule } from '../deactivated-users/deactivated-users.module';

@Module({
  imports: [
    MongooseModule.forFeature([
      { name: COLLECTIONS.AUTH.USER.USERS, schema: UserSchema },
    ]),
    TokenModule,
    JwtModule,
    RefreshTokenModule,
    forwardRef(() => DeletedUserModule),
    forwardRef(() => DeactivatedUsersModule),
  ],
  controllers: [UserController],
  providers: [UserService, UserRepository],
  exports: [UserService, UserRepository],
})
export class UserModule {}

```

## File: connectfy-auth/src/modules/users/user/user.service.ts
```typescript
import { forwardRef, HttpStatus, Inject, Injectable } from '@nestjs/common';
import { UserRepository } from './repo/user.repo';
import { AddUserDto } from './dto/add.user.dto';
import { IReturnedUser } from './interface/user.interface';
import { EditUserDto } from './dto/edit.user.dto';
import { RemoveUserDto } from './dto/remove.user.dto';
import { ClsService } from 'nestjs-cls';
import { ChangeUsernameDto } from './dto/change-username.dto';
import { ChangeEmailDto, VerifyEmailChangeDto } from './dto/change-email.dto';
import { ChangePasswordDto } from './dto/change-password.dto';
import { JwtService } from '@nestjs/jwt';
import {
  CLS_KEYS,
  LANGUAGE,
  PHONE_NUMBER_ACTION,
  TOKEN_TYPE,
  ExceptionMessages,
  BaseException,
  COUNTRIES,
  EXPIRE_DATES,
  TWO_FACTOR_ACTION,
  PROVIDER,
  USER_STATUS,
  DELETE_REASON_CODE,
  DELETE_REASON,
  IResponse,
} from 'connectfy-shared';
import { ChangePhoneNumberDto } from './dto/change-phone-number.dto';
import { NotificationsService } from '@/src/external-modules/notifications/notifications.service';
import { BcryptService } from '@/src/internal-modules/bcrypt/bcrypt.service';
import { TokenService } from '../../tokens/token/token.service';
import i18n from '@/src/i18n';
import { CheckUniqueDto } from './dto/check-unique.dto';
import { ENVIRONMENT_VARIABLES } from '@/src/common/constants/environment-variables';
import { TwoFactorDto } from './dto/two-factor.dto';
import { DeletedUserService } from '../deleted-user/deleted-user.service';
import { DeactivatedUsersService } from '../deactivated-users/deactivated-users.service';
import { DeactivateAccountDto } from './dto/deactivate-account.dto';
import { DeleteAccountDto } from './dto/delete-account.dto';
import { RefreshTokenService } from '../../tokens/refresh-token/refresh-token.service';
import { FindUserDto } from './dto/find.user.dto';
import { KafkaConnectionService } from '@/src/app-settings/kafka-connections/kafka-connection.service';

@Injectable()
export class UserService {
  constructor(
    @Inject(forwardRef(() => DeactivatedUsersService))
    private readonly deactivatedUserService: DeactivatedUsersService,
    @Inject(forwardRef(() => DeletedUserService))
    private readonly deletedUserService: DeletedUserService,

    private readonly repo: UserRepository,
    private readonly cls: ClsService,
    private readonly jwtService: JwtService,
    private readonly emailService: NotificationsService,
    private readonly bcryptService: BcryptService,
    private readonly tokenService: TokenService,
    private readonly refreshTokenService: RefreshTokenService,
    private readonly kafkaConnectionService: KafkaConnectionService,
  ) {}

  // =======================
  // CREATE USER
  // =======================
  async create(data: AddUserDto): Promise<IReturnedUser> {
    const res = await this.repo.create(data);

    const projectionPayload = {
      _id: res._id,
      firstName: data.firstName,
      lastName: data.lastName,
      fullName: `${data.firstName} ${data.lastName}`,
      username: res.username,
      email: res.email,
      phoneNumber: res.phoneNumber,
      status: res.status,
      createdAt: res.createdAt,
    };

    this.kafkaConnectionService.emitWithContext({
      topic: 'projection.user.created',
      payload: projectionPayload,
    });

    return res;
  }

  // =======================
  // EDIT USER INFORMATION'S
  // =======================
  async edit(data: EditUserDto): Promise<IResponse> {
    await this.repo.update({ _id: data._id }, data);
    return { success: true };
  }

  // =======================
  // REMOVE USER
  // =======================
  async remove(data: RemoveUserDto): Promise<IResponse> {
    const language = await this.cls.get(CLS_KEYS.LANG);
    const { _id } = data;

    const foundData = await this.repo.existsByField({ _id });

    if (!foundData) {
      throw new BaseException(
        ExceptionMessages.NOT_FOUND_MESSAGE(language),
        HttpStatus.NOT_FOUND,
      );
    }

    await this.repo.remove({ _id });
    return { success: true };
  }

  // =======================
  // FIND ONE USER
  // =======================
  async findOne(
    data: FindUserDto,
    throwException: boolean = true,
  ): Promise<IReturnedUser | null> {
    const res = await this.repo.findOne(data);
    const language = this.cls.get<LANGUAGE>(CLS_KEYS.LANG);

    if (!res && throwException) {
      throw new BaseException(
        ExceptionMessages.NOT_FOUND_MESSAGE(language),
        HttpStatus.NOT_FOUND,
      );
    }

    return res;
  }

  // =======================
  // CHECK USER IS EXIST BY FIELD
  // =======================
  async existByField(query: Record<string, any>): Promise<boolean> {
    return await this.repo.existsByField(query);
  }

  // =======================
  // CHANGE USERNAME
  // =======================
  async changeUsername(data: ChangeUsernameDto): Promise<IResponse> {
    const { username, token } = data;
    const { _id, username: oldUsername } = this.cls.get<IReturnedUser>(
      CLS_KEYS.USER,
    );
    const lang = this.cls.get<LANGUAGE>(CLS_KEYS.LANG);

    const hashedToken = this.tokenService.hashToken(token);

    await this.tokenService.findToken({
      query: {
        $and: [
          { token: hashedToken },
          { userId: _id },
          { type: TOKEN_TYPE.CHANGE_USERNAME },
        ],
      },
    });

    if (username === oldUsername) {
      throw new BaseException(
        ExceptionMessages.SAME_DATA(
          i18n.t('common.username', { lng: lang }),
          lang,
        ),
        HttpStatus.BAD_REQUEST,
      );
    }

    const isExist = await this.existByField({ username });

    if (isExist) {
      throw new BaseException(
        ExceptionMessages.ALREADY_EXISTS_MESSAGE(username, lang),
        HttpStatus.BAD_REQUEST,
      );
    }

    await this.repo.update(
      { _id },
      {
        _id,
        username,
      },
    );

    await this.tokenService.removeMany({
      $and: [{ userId: _id }, { type: TOKEN_TYPE.CHANGE_USERNAME }],
    });

    this.kafkaConnectionService.emitWithContext({
      topic: 'projection.user.updated',
      payload: {
        _id,
        username,
      },
    });

    return { success: true };
  }

  // =======================
  // CHANGE EMAIL
  // =======================
  async changeEmail(data: ChangeEmailDto): Promise<{ statusCode: number }> {
    const { email, token } = data;
    const {
      _id,
      email: oldEmail,
      provider,
    } = this.cls.get<IReturnedUser>(CLS_KEYS.USER);
    const language = this.cls.get<LANGUAGE>(CLS_KEYS.LANG);

    if (provider !== PROVIDER.PASSWORD) {
      throw new BaseException(
        ExceptionMessages.BAD_REQUEST_MESSAGE(language),
        HttpStatus.BAD_REQUEST,
      );
    }

    const hashedToken = this.tokenService.hashToken(token);

    await this.tokenService.findToken({
      query: {
        $and: [
          { token: hashedToken },
          { userId: _id },
          { type: TOKEN_TYPE.CHANGE_EMAIL },
        ],
      },
    });

    if (email === oldEmail) {
      throw new BaseException(
        ExceptionMessages.SAME_DATA(
          i18n.t('common.username', { lng: language }),
          language,
        ),
        HttpStatus.BAD_REQUEST,
      );
    }

    const userWithEmail = await this.existByField({ email });

    if (userWithEmail) {
      throw new BaseException(
        ExceptionMessages.ALREADY_EXISTS_MESSAGE(email, language),
        HttpStatus.BAD_REQUEST,
      );
    }

    await this.tokenService.removeMany({
      $and: [{ userId: _id }, { type: TOKEN_TYPE.CHANGE_EMAIL }],
    });

    const emailChangeToken = await this.tokenService.generateAndSaveJwtToken({
      userId: _id,
      type: TOKEN_TYPE.CHANGE_EMAIL,
      secret: ENVIRONMENT_VARIABLES.CHANGE_EMAIL_SECRET || '',
      jwtExp: EXPIRE_DATES.JWT.ONE_HOUR,
      tokenExp: EXPIRE_DATES.TOKEN.ONE_HOUR,
      payload: { email },
    });

    this.emailService.changeEmail({
      to: email,
      language,
      additional: { token: emailChangeToken },
    });

    return { statusCode: 200 };
  }

  // =======================
  // VERIFY CHANGE EMAIL
  // =======================
  async verifyEmailChange(data: VerifyEmailChangeDto): Promise<IResponse> {
    const { token } = data;
    const { _id } = this.cls.get<IReturnedUser>(CLS_KEYS.USER);
    const lang = this.cls.get<LANGUAGE>(CLS_KEYS.LANG);

    const decoded = this.jwtService.verify(token, {
      secret: ENVIRONMENT_VARIABLES.CHANGE_EMAIL_SECRET,
    });

    if (
      decoded.type !== TOKEN_TYPE.CHANGE_EMAIL ||
      decoded.userId !== _id.toString()
    ) {
      throw new BaseException(
        ExceptionMessages.TOKEN_EXPIRED(lang),
        HttpStatus.BAD_REQUEST,
      );
    }

    const hashedToken = this.tokenService.hashToken(token);

    await this.tokenService.findToken({
      query: {
        $and: [
          { token: hashedToken },
          { userId: _id },
          { type: TOKEN_TYPE.CHANGE_EMAIL },
        ],
      },
    });

    await Promise.all([
      await this.repo.update(
        {
          _id,
        },
        {
          _id,
          email: decoded.email,
        },
      ),
      await this.tokenService.removeMany({
        $and: [{ type: TOKEN_TYPE.CHANGE_EMAIL }, { userId: _id }],
      }),
    ]);

    this.kafkaConnectionService.emitWithContext({
      topic: 'projection.user.updated',
      payload: {
        _id,
        email: decoded.email,
      },
    });

    return { success: true, email: decoded.email };
  }

  // =======================
  // CHANGE PASSWORD
  // =======================
  async changePassword(data: ChangePasswordDto): Promise<IResponse> {
    const { password, confirmPassword, token } = data;
    const { _id, provider } = this.cls.get<IReturnedUser>(CLS_KEYS.USER);
    const lang = this.cls.get<LANGUAGE>(CLS_KEYS.LANG);

    if (password !== confirmPassword) {
      throw new BaseException(
        ExceptionMessages.BAD_REQUEST_MESSAGE(lang),
        HttpStatus.BAD_REQUEST,
      );
    }

    const user = await this.repo.findOne({
      query: { _id },
      fields: '+password',
    });

    if (!user) {
      throw new BaseException(
        ExceptionMessages.NOT_FOUND_MESSAGE(lang),
        HttpStatus.NOT_FOUND,
      );
    }

    if (provider !== PROVIDER.PASSWORD) {
      throw new BaseException(
        ExceptionMessages.CONFLICT_MESSAGE(lang),
        HttpStatus.CONFLICT,
      );
    }

    const hashedToken = this.tokenService.hashToken(token);

    await this.tokenService.findToken({
      query: {
        $and: [
          { token: hashedToken },
          { userId: _id },
          { type: TOKEN_TYPE.CHANGE_PASSWORD },
        ],
      },
    });

    const isPasswordSame = await this.bcryptService.compare(
      password,
      user.password,
    );

    if (isPasswordSame) {
      throw new BaseException(
        ExceptionMessages.SAME_DATA(
          i18n.t('common.password', { lng: lang }),
          lang,
        ),
        HttpStatus.BAD_REQUEST,
      );
    }

    const hashedPassword = await this.bcryptService.hash(password);

    await this.repo.update(
      {
        _id,
      },
      {
        _id,
        password: hashedPassword,
      },
    );

    await this.tokenService.removeMany({
      $and: [{ userId: _id }, { type: TOKEN_TYPE.CHANGE_PASSWORD }],
    });

    return { success: true };
  }

  // =======================
  // CHANGE PHONE NUMBER
  // =======================
  async changePhoneNumber(data: ChangePhoneNumberDto): Promise<IResponse> {
    const { phoneNumber, token, action } = data;
    const { _id, phoneNumber: oldPhoneNumber } = this.cls.get<IReturnedUser>(
      CLS_KEYS.USER,
    );
    const lang = this.cls.get<LANGUAGE>(CLS_KEYS.LANG);

    const hashedToken = this.tokenService.hashToken(token);

    await this.tokenService.findToken({
      query: {
        $and: [
          { token: hashedToken },
          { userId: _id },
          { type: TOKEN_TYPE.CHANGE_PHONE_NUMBER },
        ],
      },
    });

    if (
      phoneNumber &&
      phoneNumber?.fullPhoneNumber === oldPhoneNumber?.fullPhoneNumber
    ) {
      throw new BaseException(
        ExceptionMessages.SAME_DATA(phoneNumber.fullPhoneNumber, lang),
        HttpStatus.BAD_REQUEST,
      );
    }

    if (action === PHONE_NUMBER_ACTION.UPDATE) {
      const country = COUNTRIES.find(
        (c) => c.code === phoneNumber?.countryCode,
      );

      if (!country) {
        throw new BaseException(
          ExceptionMessages.BAD_REQUEST_MESSAGE(lang),
          HttpStatus.BAD_REQUEST,
        );
      }

      const { numberLength } = country;

      if (phoneNumber?.number.length !== numberLength) {
        throw new BaseException(
          ExceptionMessages.INVALID_LENGTH_MESSAGE(lang),
          HttpStatus.BAD_REQUEST,
        );
      }

      const userWithPhoneNumber = await this.existByField({
        $and: [
          { 'phoneNumber.fullPhoneNumber': phoneNumber?.fullPhoneNumber },
          { _id: { $ne: _id } },
        ],
      });

      if (userWithPhoneNumber) {
        throw new BaseException(
          ExceptionMessages.ALREADY_EXISTS_MESSAGE(
            phoneNumber?.fullPhoneNumber,
            lang,
          ),
          HttpStatus.BAD_REQUEST,
        );
      }
    }

    let updatedPhoneNumber;

    if (action === PHONE_NUMBER_ACTION.REMOVE) {
      updatedPhoneNumber = null;
    } else {
      updatedPhoneNumber = phoneNumber;
    }

    await this.repo.update(
      { _id },
      {
        _id,
        phoneNumber: phoneNumber,
      },
    );

    await this.tokenService.removeMany({
      $and: [{ userId: _id }, { type: TOKEN_TYPE.CHANGE_PHONE_NUMBER }],
    });

    this.kafkaConnectionService.emitWithContext({
      topic: 'projection.user.updated',
      payload: {
        _id,
        phoneNumber: updatedPhoneNumber,
      },
    });

    return { success: true, updatedPhoneNumber };
  }

  // =======================
  // CHECK UNIQUE
  // =======================
  async checkUnique(data: CheckUniqueDto): Promise<boolean> {
    const { field, value } = data;
    const lang = this.cls.get<LANGUAGE>(CLS_KEYS.LANG);

    const isUserExist = await this.existByField({ [field]: value });

    if (isUserExist)
      throw new BaseException(
        ExceptionMessages.ALREADY_EXISTS_MESSAGE(value, lang),
        HttpStatus.BAD_REQUEST,
      );

    return true;
  }

  // =======================
  // ENABLE/DISABLE 2FA
  // =======================
  async updateTwoFactorAuth(data: TwoFactorDto): Promise<IResponse> {
    const { token, action } = data;

    const { _id, isTwoFactorEnabled, provider } = this.cls.get<IReturnedUser>(
      CLS_KEYS.USER,
    );
    const lang = this.cls.get<LANGUAGE>(CLS_KEYS.LANG);

    if (provider !== PROVIDER.PASSWORD) {
      throw new BaseException(
        ExceptionMessages.FORBIDDEN_MESSAGE(lang),
        HttpStatus.FORBIDDEN,
      );
    }

    const hashedToken = this.tokenService.hashToken(token);

    await this.tokenService.findToken({
      query: {
        $and: [
          { token: hashedToken },
          { userId: _id },
          { type: TOKEN_TYPE.TWO_FACTOR },
        ],
      },
    });

    if (
      (isTwoFactorEnabled && action === TWO_FACTOR_ACTION.ENABLE) ||
      (!isTwoFactorEnabled && action === TWO_FACTOR_ACTION.DISABLE)
    ) {
      throw new BaseException(
        ExceptionMessages.CONFLICT_MESSAGE,
        HttpStatus.CONFLICT,
      );
    }

    const finalData = {
      _id,
      isTwoFactorEnabled: action === TWO_FACTOR_ACTION.ENABLE,
    };

    await Promise.all([
      this.repo.update({ _id }, finalData),
      this.tokenService.removeMany({
        $and: [{ userId: _id }, { type: TOKEN_TYPE.TWO_FACTOR }],
      }),
    ]);

    return { success: true };
  }

  // =================================
  // DEACTIVATE ACCOUNT
  // =================================
  async deactivateAccount(data: DeactivateAccountDto) {
    const { token } = data;
    const { _id } = this.cls.get<IReturnedUser>(CLS_KEYS.USER);
    const lang = this.cls.get<LANGUAGE>(CLS_KEYS.LANG);

    const hashedToken = this.tokenService.hashToken(token);

    const deactivateToken = await this.tokenService.findToken({
      query: {
        $and: [
          { token: hashedToken },
          { userId: _id },
          { type: TOKEN_TYPE.DEACTIVATE_ACCOUNT },
        ],
      },
    });

    if (deactivateToken.expiresAt && deactivateToken.expiresAt < new Date()) {
      throw new BaseException(
        ExceptionMessages.TOKEN_EXPIRED(lang),
        HttpStatus.GONE,
      );
    }

    await Promise.all([
      this.deactivatedUserService.create({ userId: _id }),
      this.tokenService.removeTokensByUserId(_id),
      this.refreshTokenService.removeTokenByUserId(_id),
      this.repo.update({ _id }, { _id, status: USER_STATUS.INACTIVE }),
    ]);

    this.kafkaConnectionService.emitWithContext({
      topic: 'projection.user.updated',
      payload: {
        _id,
        status: USER_STATUS.INACTIVE,
      },
    });

    return { statusCode: 200 };
  }

  // =================================
  // DELETE ACCOUNT
  // =================================
  async deleteAccount(data: DeleteAccountDto): Promise<{ statusCode: 200 }> {
    const { token } = data;
    const { _id, email } = this.cls.get<IReturnedUser>(CLS_KEYS.USER);
    const lang = this.cls.get<LANGUAGE>(CLS_KEYS.LANG);

    const hashedToken = this.tokenService.hashToken(token);
    const deleteToken = await this.tokenService.findToken({
      query: {
        $and: [
          { token: hashedToken },
          { userId: _id },
          { type: TOKEN_TYPE.DELETE_ACCOUNT },
        ],
      },
    });

    const now = new Date();

    if (!deleteToken || now >= deleteToken.expiresAt) {
      if (deleteToken) {
        await this.tokenService.removeToken({ _id: deleteToken._id });
      }

      throw new BaseException(
        ExceptionMessages.TOKEN_EXPIRED(lang),
        HttpStatus.BAD_REQUEST,
      );
    }

    const user = await this.existByField({ _id });

    if (!user) {
      throw new BaseException(
        ExceptionMessages.NOT_FOUND_MESSAGE(lang),
        HttpStatus.NOT_FOUND,
      );
    }

    const reasonDescription =
      data.reasonCode === DELETE_REASON_CODE.FOUND_ALTERNATIVE
        ? null
        : data.reasonDescription;

    const [newToken] = await Promise.all([
      this.tokenService.generateAndSaveJwtToken({
        userId: _id,
        type: TOKEN_TYPE.RESTORE_ACCOUNT,
        secret: ENVIRONMENT_VARIABLES.RESTORE_ACCOUNT_SECRET || '',
        jwtExp: EXPIRE_DATES.JWT.ONE_MONTH,
        tokenExp: EXPIRE_DATES.TOKEN.ONE_MONTH,
      }),
      this.tokenService.removeMany({
        $and: [{ userId: _id }, { type: { $ne: TOKEN_TYPE.RESTORE_ACCOUNT } }],
      }),
      this.refreshTokenService.removeTokenByUserId(_id),
      this.deletedUserService.create({
        userId: _id,
        reason: DELETE_REASON.USER_REQUEST,
        reasonCode: data.reasonCode,
        reasonDescription,
      }),
    ]);

    this.emailService.deleteAccountCompleted({
      to: email,
      language: lang,
      additional: { token: newToken },
    });

    this.kafkaConnectionService.emitWithContext({
      topic: 'projection.user.updated',
      payload: {
        _id,
        status: USER_STATUS.INACTIVE,
      },
    });

    return { statusCode: 200 };
  }
}

```

## File: connectfy-auth/src/modules/users/user/dto/change-email.dto.ts
```typescript
import { FieldValidator, FIELD_TYPE } from 'connectfy-shared';

export class ChangeEmailDto {
  @FieldValidator({
    type: FIELD_TYPE.EMAIL,
    maxLength: 254,
  })
  email: string;

  @FieldValidator({
    type: FIELD_TYPE.STRING,
    maxLength: 1000,
  })
  token: string;
}

export class VerifyEmailChangeDto {
  @FieldValidator({
    type: FIELD_TYPE.JWT,
    maxLength: 1000,
  })
  token: string;
}

```

## File: connectfy-auth/src/modules/users/user/dto/find.user.dto.ts
```typescript
import { BaseFindDto, FIELD_TYPE, FieldValidator } from 'connectfy-shared';

export class FindUserDto extends BaseFindDto {}

export class FindUserByIdDto {
  @FieldValidator({ type: FIELD_TYPE.UUID })
  _id: string;
}

```

## File: connectfy-auth/src/modules/users/user/dto/change-username.dto.ts
```typescript
import { FieldValidator, FIELD_TYPE, VALIDATION_TYPE } from 'connectfy-shared';

export class ChangeUsernameDto {
  @FieldValidator({
    type: FIELD_TYPE.STRING,
    minLength: 3,
    maxLength: 30,
    matches: {
      regexp: /^[A-Za-z0-9._-]+$/,
      message: {
        type: VALIDATION_TYPE.MISMATCH,
        params: { characters: '(.,?()$:;"\'{}[]-=+&!\\|/<>`~@#№%^)' },
      },
    },
  })
  username: string;

  @FieldValidator({
    type: FIELD_TYPE.STRING,
    maxLength: 1000,
  })
  token: string;
}

```

## File: connectfy-auth/src/modules/users/user/dto/deactivate-account.dto.ts
```typescript
import { FIELD_TYPE, FieldValidator } from 'connectfy-shared';

export class DeactivateAccountDto {
  @FieldValidator({
    type: FIELD_TYPE.STRING,
    maxLength: 1000,
  })
  token: string;
}

```

## File: connectfy-auth/src/modules/users/user/dto/edit.user.dto.ts
```typescript
import { BaseUserDto } from './base.user.dto';
import { PartialType } from '@nestjs/mapped-types';
import { PhoneNumberDto } from './nested/phoneNumber.dto';
import { FIELD_TYPE, USER_STATUS, FieldValidator } from 'connectfy-shared';

export class EditUserDto extends PartialType(BaseUserDto) {
  @FieldValidator({
    type: FIELD_TYPE.UUID,
  })
  _id: string;

  @FieldValidator({
    type: FIELD_TYPE.OBJECT,
    classType: PhoneNumberDto,
    isOptional: true,
  })
  phoneNumber?: PhoneNumberDto | null;

  @FieldValidator({
    type: FIELD_TYPE.ENUM,
    enumObject: USER_STATUS,
    isOptional: true,
  })
  status?: USER_STATUS;

  @FieldValidator({
    type: FIELD_TYPE.BOOLEAN,
    isOptional: true,
  })
  isTwoFactorEnabled?: boolean;
}

```

## File: connectfy-auth/src/modules/users/user/dto/remove.user.dto.ts
```typescript
import { BaseRemoveDto } from 'connectfy-shared';

export class RemoveUserDto extends BaseRemoveDto {}

```

## File: connectfy-auth/src/modules/users/user/dto/delete-account.dto.ts
```typescript
import {
  FIELD_TYPE,
  FieldValidator,
  DELETE_REASON_CODE,
} from 'connectfy-shared';

export class DeleteAccountDto {
  @FieldValidator({
    type: FIELD_TYPE.STRING,
    maxLength: 1000,
  })
  token: string;

  @FieldValidator({
    type: FIELD_TYPE.ENUM,
    enumObject: DELETE_REASON_CODE,
  })
  reasonCode: DELETE_REASON_CODE;

  @FieldValidator({
    type: FIELD_TYPE.STRING,
    isOptional: true,
    maxLength: 200,
  })
  reasonDescription: string | null;
}

```

## File: connectfy-auth/src/modules/users/user/dto/two-factor.dto.ts
```typescript
import {
  FIELD_TYPE,
  FieldValidator,
  TWO_FACTOR_ACTION,
} from 'connectfy-shared';

export class TwoFactorDto {
  @FieldValidator({
    type: FIELD_TYPE.STRING,
    maxLength: 1000,
  })
  token: string;

  @FieldValidator({
    type: FIELD_TYPE.ENUM,
    enumObject: TWO_FACTOR_ACTION,
  })
  action: TWO_FACTOR_ACTION;
}

```

## File: connectfy-auth/src/modules/users/user/dto/change-password.dto.ts
```typescript
import { FieldValidator, FIELD_TYPE, VALIDATION_TYPE } from 'connectfy-shared';

export class ChangePasswordDto {
  @FieldValidator({
    type: FIELD_TYPE.STRING,
    minLength: 8,
    maxLength: 30,
    matches: {
      regexp: /^(?=.*[a-z])(?=.*[A-Z])(?=.*\d)(?=.*[^A-Za-z0-9])\S{8,30}$/,
      message: {
        type: VALIDATION_TYPE.PASSWORD,
      },
    },
  })
  password: string;

  @FieldValidator({
    type: FIELD_TYPE.STRING,
    minLength: 8,
    maxLength: 30,
  })
  confirmPassword: string;

  @FieldValidator({
    type: FIELD_TYPE.STRING,
    maxLength: 1000,
  })
  token: string;
}

```

## File: connectfy-auth/src/modules/users/user/dto/base.user.dto.ts
```typescript
import { FieldValidator, FIELD_TYPE } from 'connectfy-shared';

export class BaseUserDto {
  @FieldValidator({
    type: FIELD_TYPE.STRING,
  })
  username: string;

  @FieldValidator({
    type: FIELD_TYPE.EMAIL,
  })
  email: string;

  @FieldValidator({
    type: FIELD_TYPE.STRING,
  })
  password: string;
}

```

## File: connectfy-auth/src/modules/users/user/dto/change-phone-number.dto.ts
```typescript
import {
  FieldValidator,
  FIELD_TYPE,
  PHONE_NUMBER_ACTION,
} from 'connectfy-shared';
import { PhoneNumberDto } from './nested/phoneNumber.dto';

export class ChangePhoneNumberDto {
  @FieldValidator({
    type: FIELD_TYPE.STRING,
  })
  token: string;

  @FieldValidator({
    type: FIELD_TYPE.ENUM,
    enumObject: PHONE_NUMBER_ACTION,
  })
  action: PHONE_NUMBER_ACTION;

  @FieldValidator({
    type: FIELD_TYPE.OBJECT,
    isOptional: true,
    validateIf: (obj) => obj.action === PHONE_NUMBER_ACTION.UPDATE,
    validateNested: {},
    classType: PhoneNumberDto,
  })
  phoneNumber: PhoneNumberDto | null;
}

```

## File: connectfy-auth/src/modules/users/user/dto/add.user.dto.ts
```typescript
import { BaseUserDto } from './base.user.dto';
import { FIELD_TYPE, PROVIDER, FieldValidator } from 'connectfy-shared';

export class AddUserDto extends BaseUserDto {
  @FieldValidator({
    type: FIELD_TYPE.ENUM,
    enumObject: PROVIDER,
  })
  provider: PROVIDER;

  @FieldValidator({
    type: FIELD_TYPE.STRING,
    isOptional: true,
  })
  timeZone: string | null;

  @FieldValidator({
    type: FIELD_TYPE.STRING,
    isOptional: true,
  })
  location: string | null;

  @FieldValidator({
    type: FIELD_TYPE.STRING,
  })
  firstName: string;

  @FieldValidator({
    type: FIELD_TYPE.STRING,
  })
  lastName: string;
}

```

## File: connectfy-auth/src/modules/users/user/dto/check-unique.dto.ts
```typescript
import {
  FieldValidator,
  FIELD_TYPE,
  CHECK_UNIQUE_FIELD,
} from 'connectfy-shared';

export class CheckUniqueDto {
  @FieldValidator({ type: FIELD_TYPE.ENUM, enumObject: CHECK_UNIQUE_FIELD })
  field: CHECK_UNIQUE_FIELD;

  @FieldValidator({ type: FIELD_TYPE.STRING })
  value: string;
}

```

## File: connectfy-auth/src/modules/users/user/dto/nested/phoneNumber.dto.ts
```typescript
import { FieldValidator, FIELD_TYPE, VALIDATION_TYPE } from 'connectfy-shared';

export class PhoneNumberDto {
  @FieldValidator({
    type: FIELD_TYPE.STRING,
    matches: {
      regexp: /^\+\d+$/,
      message: {
        type: VALIDATION_TYPE.PHONE_CODE,
      },
    },
  })
  countryCode: string;

  @FieldValidator({
    type: FIELD_TYPE.STRING,
    matches: {
      regexp: /^\d+$/,
      message: {
        type: VALIDATION_TYPE.NUMBER,
      },
    },
  })
  number: string;

  @FieldValidator({
    type: FIELD_TYPE.STRING,
    matches: {
      regexp: /^\+\d+$/,
      message: {
        type: VALIDATION_TYPE.FULL_PHONE,
      },
    },
  })
  fullPhoneNumber: string;
}

```

## File: connectfy-auth/src/modules/users/user/interface/user.interface.ts
```typescript
import { PROVIDER, ROLE, USER_STATUS } from 'connectfy-shared';
import { IPhoneNumber } from './nested/phoneNumber.interface';

export interface IUser {
  _id: string;
  username: string;
  email: string;
  role: ROLE;
  provider: PROVIDER;
  password: string;
  phoneNumber: IPhoneNumber | null;
  status: USER_STATUS;
  isTwoFactorEnabled: boolean;
  timeZone: string | null;
  location: string | null;
  createdAt: Date;
  updatedAt: Date;
}

export interface IReturnedUser {
  _id: string;
  username: string;
  email: string;
  role: ROLE;
  provider: PROVIDER;
  password: string;
  phoneNumber: IPhoneNumber | null;
  status: USER_STATUS;
  isTwoFactorEnabled: boolean;
  timeZone: string | null;
  location: string | null;
  createdAt: Date;
  updatedAt: Date;
}

```

## File: connectfy-auth/src/modules/users/user/interface/nested/phoneNumber.interface.ts
```typescript
export interface IPhoneNumber {
  countryCode: string | null;
  number: string | null;
  fullPhoneNumber: string | null;
}

export interface IReturnedPhoneNumber {
  countryCode: string | null;
  number: string | null;
  fullPhoneNumber: string | null;
}

```

## File: connectfy-auth/src/modules/users/user/repo/user.repo.ts
```typescript
import { Injectable } from '@nestjs/common';
import { UserDocument } from '../entity/user.entity';
import { AddUserDto } from '../dto/add.user.dto';
import { EditUserDto } from '../dto/edit.user.dto';
import { InjectModel } from '@nestjs/mongoose';
import { Model } from 'mongoose';
import { IReturnedUser } from '@modules/users/user/interface/user.interface';
import { COLLECTIONS, BaseRepository } from 'connectfy-shared';

@Injectable()
export class UserRepository extends BaseRepository<
  UserDocument,
  IReturnedUser,
  AddUserDto,
  EditUserDto
> {
  constructor(
    @InjectModel(COLLECTIONS.AUTH.USER.USERS)
    protected readonly model: Model<UserDocument>,
  ) {
    super(model);
  }
}

```

## File: connectfy-auth/src/modules/users/user/entity/user.entity.ts
```typescript
import { v4 as uuid } from 'uuid';
import { HydratedDocument } from 'mongoose';
import { IUser } from '../interface/user.interface';
import { Prop, Schema, SchemaFactory } from '@nestjs/mongoose';
import {
  PhoneNumberModel,
  PhoneNumberSchema,
} from './nested/phoneNumber.entity';
import { t } from 'i18next';
import {
  LANGUAGE,
  COLLECTIONS,
  PROVIDER,
  ROLE,
  USER_STATUS,
} from 'connectfy-shared';
import * as bcrypt from 'bcrypt';
import * as mongooseLeanVirtuals from 'mongoose-lean-virtuals';

@Schema({
  timestamps: true,
  collection: COLLECTIONS.AUTH.USER.USERS,
  toJSON: {
    virtuals: true,
    versionKey: false,
    getters: true,
  },
  toObject: {
    virtuals: true,
    versionKey: false,
  },
  autoIndex: process.env.NODE_ENV !== 'production',
  minimize: false,
  strict: true,
  strictQuery: true,
})
export class UserModel implements IUser {
  @Prop({
    type: String,
    default: () => uuid(),
    immutable: true,
  })
  _id: string;

  @Prop({
    type: String,
    required: [
      true,
      t('validation_messages.required', {
        lng: LANGUAGE.EN,
        field: 'username',
      }),
    ],
    unique: true,
    minlength: [
      3,
      t('validation_messages.min_length', {
        lng: LANGUAGE.EN,
        field: 'username',
        length: 3,
      }),
    ],
    maxlength: [
      30,
      t('validation_messages.max_length', {
        lng: LANGUAGE.EN,
        field: 'username',
        length: 30,
      }),
    ],
    trim: true,
    lowercase: true,
    index: true,
    validate: {
      validator: function (value: string): boolean {
        return /^[a-zA-Z0-9_-]+$/.test(value);
      },
      message: t('validation_messages.invalid_username', {
        lng: LANGUAGE.EN,
      }),
    },
  })
  username: string;

  @Prop({
    type: String,
    required: [
      true,
      t('validation_messages.required', {
        lng: LANGUAGE.EN,
        field: 'email',
      }),
    ],
    unique: true,
    maxlength: [
      254,
      t('validation_messages.max_length', {
        lng: LANGUAGE.EN,
        field: 'email',
        length: 254,
      }),
    ],
    trim: true,
    lowercase: true,
    index: true,
    validate: {
      validator: function (value: string): boolean {
        return /^[^\s@]+@[^\s@]+\.[^\s@]+$/.test(value);
      },
      message: t('validation_messages.invalid_email', {
        lng: LANGUAGE.EN,
      }),
    },
  })
  email: string;

  @Prop({
    type: String,
    enum: {
      values: Object.values(ROLE),
      message: t('validation_messages.enum', {
        lng: LANGUAGE.EN,
        field: 'role',
        values: Object.values(ROLE),
      }),
    },
    required: [
      true,
      t('validation_messages.required', {
        lng: LANGUAGE.EN,
        field: 'role',
      }),
    ],
    default: ROLE.USER,
    index: true,
  })
  role: ROLE;

  @Prop({
    type: String,
    enum: {
      values: Object.values(PROVIDER),
      message: t('validation_messages.enum', {
        lng: LANGUAGE.EN,
        field: 'provider',
        values: Object.values(PROVIDER),
      }),
    },
    required: [
      true,
      t('validation_messages.required', {
        lng: LANGUAGE.EN,
        field: 'provider',
      }),
    ],
    default: PROVIDER.PASSWORD,
    index: true,
  })
  provider: PROVIDER;

  @Prop({
    type: String,
    required: [
      true,
      t('validation_messages.required', {
        lng: LANGUAGE.EN,
        field: 'password',
      }),
    ],
    maxlength: [
      100,
      t('validation_messages.max_length', {
        lng: LANGUAGE.EN,
        field: 'password',
        length: 100,
      }),
    ],
    select: false,
  })
  password: string;

  @Prop({
    type: Boolean,
    required: false,
    default: false,
    index: true,
  })
  isTwoFactorEnabled: boolean;

  @Prop({
    type: PhoneNumberSchema,
    required: false,
    default: null,
  })
  phoneNumber: PhoneNumberModel | null;

  @Prop({
    type: String,
    required: [
      true,
      t('validation_messages.required', {
        lng: LANGUAGE.EN,
        field: 'status',
      }),
    ],
    enum: {
      values: Object.values(USER_STATUS),
      message: t('validation_messages.enum', {
        lng: LANGUAGE.EN,
        field: 'status',
        values: Object.values(USER_STATUS),
      }),
    },
    default: USER_STATUS.ACTIVE,
    index: true,
  })
  status: USER_STATUS;

  @Prop({
    type: String,
    required: false,
    default: null,
    trim: true,
    maxlength: [
      100,
      t('validation_messages.max_length', {
        lng: LANGUAGE.EN,
        field: 'timeZone',
        length: 100,
      }),
    ],
  })
  timeZone: string | null;

  @Prop({
    type: String,
    required: false,
    default: null,
    trim: true,
    maxlength: [
      100,
      t('validation_messages.max_length', {
        lng: LANGUAGE.EN,
        field: 'location',
        length: 100,
      }),
    ],
  })
  location: string | null;

  createdAt: Date;
  updatedAt: Date;
}

export const UserSchema = SchemaFactory.createForClass(UserModel);

// ================================================
// INDEXES - Performance optimization
// ================================================

// Compound indexes
UserSchema.index({ email: 1, status: 1 });
UserSchema.index({ username: 1, status: 1 });
UserSchema.index({ role: 1, status: 1 });
UserSchema.index({ provider: 1, status: 1 });
UserSchema.index({ createdAt: -1 });
UserSchema.index({ updatedAt: -1 });
UserSchema.index({ isTwoFactorEnabled: 1 });

// Text search index
UserSchema.index({ username: 'text', email: 'text' });

// Sparse indexes (null dəyərləri skip edir)
UserSchema.index({ 'phoneNumber.number': 1 }, { sparse: true });

// ================================================
// MIDDLEWARE / HOOKS
// ================================================

// Pre-save hook - Password hashing
UserSchema.pre('save', async function (next) {
  if (this.isModified('password')) {
    try {
      const salt = await bcrypt.genSalt(10);
      this.password = await bcrypt.hash(this.password, salt);
    } catch (error) {
      return next(error as Error);
    }
  }

  // Username və email lowercase
  if (this.username) {
    this.username = this.username.toLowerCase().trim();
  }
  if (this.email) {
    this.email = this.email.toLowerCase().trim();
  }

  next();
});

UserSchema.plugin(mongooseLeanVirtuals.mongooseLeanVirtuals);
export type UserDocument = HydratedDocument<UserModel>;

```

## File: connectfy-auth/src/modules/users/user/entity/nested/phoneNumber.entity.ts
```typescript
import { Prop, Schema, SchemaFactory } from '@nestjs/mongoose';
import { IPhoneNumber } from '../../interface/nested/phoneNumber.interface';

@Schema({ _id: false, timestamps: false })
export class PhoneNumberModel implements IPhoneNumber {
  @Prop({ type: String, required: false, default: null, unique: false })
  countryCode: string;

  @Prop({ type: String, required: false, default: null, unique: false })
  number: string;

  @Prop({ type: String, required: false, default: null, unique: false })
  fullPhoneNumber: string;
}

export const PhoneNumberSchema = SchemaFactory.createForClass(PhoneNumberModel);
export type PhoneNumberDocument = PhoneNumberModel & Document;

```

## File: connectfy-auth/src/modules/users/banned-user/banned-user.controller.ts
```typescript
import { Controller } from '@nestjs/common';
import { BannedUserService } from './banned-user.service';

@Controller('banned-user')
export class BannedUserController {
  constructor(private readonly service: BannedUserService) {}
}

```

## File: connectfy-auth/src/modules/users/banned-user/banned-user.service.ts
```typescript
import { HttpStatus, Injectable } from '@nestjs/common';
import { BannedUserRepository } from './repo/banned-user.repo';
import { AddBannedUserDto } from './dto/add.banned-user.dto';
import { RemoveBannedUserDto } from './dto/remove.banned-user.dto';
import { IReturnedBannedUser } from './interface/banned-user.interface';
import {
  ExceptionMessages,
  BaseException,
  CLS_KEYS,
  LANGUAGE,
} from 'connectfy-shared';
import { ClsService } from 'nestjs-cls';

@Injectable()
export class BannedUserService {
  constructor(
    private readonly repo: BannedUserRepository,
    private readonly cls: ClsService,
  ) {}

  async create(data: AddBannedUserDto): Promise<IReturnedBannedUser> {
    const res = await this.repo.create(data);

    return res;
  }

  async remove(data: RemoveBannedUserDto): Promise<IReturnedBannedUser> {
    const { _id, _lang } = data;

    const foundData = await this.repo.findOne({ query: { _id } });

    if (!foundData) {
      throw new BaseException(
        ExceptionMessages.NOT_FOUND_MESSAGE(_lang),
        HttpStatus.NOT_FOUND,
      );
    }

    const res = await this.repo.remove({ _id });

    return res;
  }

  async findOne(
    query: Record<string, any>,
  ): Promise<IReturnedBannedUser | null> {
    const res = await this.repo.findOne(query);
    return res;
  }

  async isUserBanned(query: Record<string, any>): Promise<void> {
    const lang = this.cls.get<LANGUAGE>(CLS_KEYS.LANG);
    const res = await this.repo.existsByField(query);

    if (res) {
      throw new BaseException(
        ExceptionMessages.FORBIDDEN_MESSAGE(lang),
        HttpStatus.FORBIDDEN,
        { navigate: true },
      );
    }
  }
}

```

## File: connectfy-auth/src/modules/users/banned-user/banned-user.module.ts
```typescript
import { Module } from '@nestjs/common';
import { MongooseModule } from '@nestjs/mongoose';
import { BannedUserService } from './banned-user.service';
import { BannedUserSchema } from './entity/banned-user.entity';
import { BannedUserController } from './banned-user.controller';
import { BannedUserRepository } from './repo/banned-user.repo';
import { COLLECTIONS } from 'connectfy-shared';

@Module({
  imports: [
    MongooseModule.forFeature([
      { name: COLLECTIONS.AUTH.USER.BANNED, schema: BannedUserSchema },
    ]),
  ],
  controllers: [BannedUserController],
  providers: [BannedUserService, BannedUserRepository],
  exports: [BannedUserService, BannedUserRepository],
})
export class BannedUserModule {}

```

## File: connectfy-auth/src/modules/users/banned-user/dto/base.banned-user.dto.ts
```typescript
import { FieldValidator, FIELD_TYPE } from 'connectfy-shared';

export class BaseBannedUserDto {
  @FieldValidator({
    type: FIELD_TYPE.UUID,
  })
  userId: string;

  @FieldValidator({
    type: FIELD_TYPE.DATE,
  })
  bannedToDate: Date;
}

```

## File: connectfy-auth/src/modules/users/banned-user/dto/remove.banned-user.dto.ts
```typescript
import {
  FIELD_TYPE,
  LANGUAGE,
  BaseRemoveAllDto,
  BaseRemoveDto,
  FieldValidator,
} from 'connectfy-shared';

export class RemoveBannedUserDto extends BaseRemoveDto {
  @FieldValidator({
    type: FIELD_TYPE.ENUM,
    isOptional: true,
    enumObject: LANGUAGE,
  })
  _lang?: LANGUAGE;
}

export class RemoveAllBannedUsersDto extends BaseRemoveAllDto {
  @FieldValidator({
    type: FIELD_TYPE.ENUM,
    isOptional: true,
    enumObject: LANGUAGE,
  })
  _lang?: LANGUAGE;
}

```

## File: connectfy-auth/src/modules/users/banned-user/dto/add.banned-user.dto.ts
```typescript
import { BaseBannedUserDto } from './base.banned-user.dto';

export class AddBannedUserDto extends BaseBannedUserDto {}

```

## File: connectfy-auth/src/modules/users/banned-user/dto/find.banned-user.dto.ts
```typescript
import { BaseFindDto } from 'connectfy-shared';

export class FindBannedUserDto extends BaseFindDto {}

```

## File: connectfy-auth/src/modules/users/banned-user/interface/banned-user.interface.ts
```typescript
export interface IBannedUser {
  _id: string;
  userId: string;
  bannedToDate: Date | null;
  createdAt: Date;
  updatedAt: Date;
}

export interface IReturnedBannedUser {
  _id: string;
  userId: string;
  bannedToDate: Date | null;
  createdAt: Date;
  updatedAt: Date;
}

```

## File: connectfy-auth/src/modules/users/banned-user/repo/banned-user.repo.ts
```typescript
import { Model } from 'mongoose';
import { Injectable } from '@nestjs/common';
import { InjectModel } from '@nestjs/mongoose';
import { AddBannedUserDto } from '../dto/add.banned-user.dto';
import { BannedUserDocument } from '../entity/banned-user.entity';
import { COLLECTIONS, BaseRepository } from 'connectfy-shared';
import { IReturnedBannedUser } from '@modules/users/banned-user/interface/banned-user.interface';

@Injectable()
export class BannedUserRepository extends BaseRepository<
  BannedUserDocument,
  IReturnedBannedUser,
  AddBannedUserDto
> {
  constructor(
    @InjectModel(COLLECTIONS.AUTH.USER.BANNED)
    protected readonly model: Model<BannedUserDocument>,
  ) {
    super(model);
  }
}

```

## File: connectfy-auth/src/modules/users/banned-user/entity/banned-user.entity.ts
```typescript
import { v4 as uuid, validate } from 'uuid';
import { HydratedDocument } from 'mongoose';
import { Prop, Schema, SchemaFactory } from '@nestjs/mongoose';
import { IBannedUser } from '../interface/banned-user.interface';
import { t } from 'i18next';
import { LANGUAGE, COLLECTIONS } from 'connectfy-shared';

@Schema({
  timestamps: true,
  collection: COLLECTIONS.AUTH.USER.BANNED,
  toJSON: { virtuals: true, versionKey: false, getters: true },
  toObject: { virtuals: true, versionKey: false },
  autoIndex: process.env.NODE_ENV !== 'production',
  minimize: false,
  strict: true,
  strictQuery: true,
})
export class BannedUserModel implements IBannedUser {
  @Prop({
    type: String,
    default: () => uuid(),
    immutable: true,
  })
  _id: string;

  @Prop({
    type: String,
    required: [
      true,
      t('validation_messages.required', {
        lng: LANGUAGE.EN,
        field: 'userId',
      }),
    ],
    unique: true,
    index: true,
    immutable: true,
    ref: COLLECTIONS.AUTH.USER.USERS,
    validate: {
      validator: function (value: string | null): boolean {
        return !!(value && validate(value));
      },
      message: t('validation_messages.uuid', {
        lng: LANGUAGE.EN,
        field: 'userId',
      }),
    },
  })
  userId: string;

  @Prop({
    type: Date,
    required: false,
    default: null,
    index: true,
    sparse: true,
  })
  bannedToDate: Date | null;

  createdAt: Date;
  updatedAt: Date;
}

export const BannedUserSchema = SchemaFactory.createForClass(BannedUserModel);

// ================================================
// INDEXES - Performance optimization
// ================================================

// Timestamp indexes
BannedUserSchema.index({ createdAt: -1 });
BannedUserSchema.index({ updatedAt: -1 });

// Compound indexes (queries üçün)
BannedUserSchema.index({ userId: 1, bannedToDate: 1 });
BannedUserSchema.index({ bannedToDate: 1, createdAt: -1 });

export type BannedUserDocument = HydratedDocument<BannedUserModel>;

```

## File: connectfy-auth/src/modules/users/deactivated-users/deactivated-users.service.ts
```typescript
import { forwardRef, Inject, Injectable } from '@nestjs/common';
import { DeactivatedUserRepository } from './repo/deactivated-user.repo';
import { UserService } from '../user/user.service';
import {
  BaseException,
  CLS_KEYS,
  ExceptionMessages,
  HttpStatus,
  LANGUAGE,
  USER_STATUS,
} from 'connectfy-shared';
import { ClsService } from 'nestjs-cls';
import { AddDeactivatedUserDto } from './dto/add.deactivated-user.dto';

@Injectable()
export class DeactivatedUsersService {
  constructor(
    @Inject(forwardRef(() => UserService))
    private readonly userService: UserService,

    private readonly cls: ClsService,
    private readonly repo: DeactivatedUserRepository,
  ) {}

  async create(data: AddDeactivatedUserDto): Promise<void> {
    const userData = {
      _id: data.userId,
      status: USER_STATUS.INACTIVE,
    };

    await Promise.all([
      this.repo.create(data),
      this.userService.edit(userData),
    ]);
  }

  async activateUserIfExist(userId: string): Promise<void> {
    const deactivatedUser = await this.repo.findOne({ query: { userId } });
    const language = this.cls.get<LANGUAGE>(CLS_KEYS.LANG);

    if (!deactivatedUser) {
      throw new BaseException(
        ExceptionMessages.INVALID_CREDENTIALS(language),
        HttpStatus.BAD_REQUEST,
      );
    }

    const data = {
      _id: userId,
      status: USER_STATUS.ACTIVE,
    };

    Promise.all([
      this.userService.edit(data),
      this.repo.remove({ _id: deactivatedUser._id }),
    ]);
  }
}

```

## File: connectfy-auth/src/modules/users/deactivated-users/deactivated-users.module.ts
```typescript
import { forwardRef, Module } from '@nestjs/common';
import { DeactivatedUsersService } from './deactivated-users.service';
import { DeactivatedUsersController } from './deactivated-users.controller';
import { DeactivatedUserRepository } from './repo/deactivated-user.repo';
import { MongooseModule } from '@nestjs/mongoose';
import { DeactivatedUserSchema } from './entity/deactivated-user.entity';
import { COLLECTIONS } from 'connectfy-shared';
import { UserModule } from '../user/user.module';

@Module({
  imports: [
    MongooseModule.forFeature([
      {
        name: COLLECTIONS.AUTH.USER.DEACTIVATED,
        schema: DeactivatedUserSchema,
      },
    ]),
    forwardRef(() => UserModule),
  ],
  controllers: [DeactivatedUsersController],
  providers: [DeactivatedUsersService, DeactivatedUserRepository],
  exports: [DeactivatedUsersService, DeactivatedUserRepository],
})
export class DeactivatedUsersModule {}

```

## File: connectfy-auth/src/modules/users/deactivated-users/deactivated-users.controller.ts
```typescript
import { Controller } from '@nestjs/common';
import { DeactivatedUsersService } from './deactivated-users.service';

@Controller()
export class DeactivatedUsersController {
  constructor(private readonly service: DeactivatedUsersService) {}
}

```

## File: connectfy-auth/src/modules/users/deactivated-users/dto/base.deactivated-user.dto.ts
```typescript
import { FieldValidator, FIELD_TYPE } from 'connectfy-shared';

export class BaseDeactivatedUserDto {
  @FieldValidator({
    type: FIELD_TYPE.UUID,
  })
  userId: string;
}

```

## File: connectfy-auth/src/modules/users/deactivated-users/dto/remove.deactivated-user.dto.ts
```typescript
import { BaseRemoveDto } from 'connectfy-shared';

export class RemoveDeactivatedUserRepo extends BaseRemoveDto {}

```

## File: connectfy-auth/src/modules/users/deactivated-users/dto/add.deactivated-user.dto.ts
```typescript
import { BaseDeactivatedUserDto } from './base.deactivated-user.dto';

export class AddDeactivatedUserDto extends BaseDeactivatedUserDto {}

```

## File: connectfy-auth/src/modules/users/deactivated-users/dto/find.deactivated-user.dto.ts
```typescript
import { BaseFindDto } from 'connectfy-shared';

export class FindDeactivatedUserDto extends BaseFindDto {}

```

## File: connectfy-auth/src/modules/users/deactivated-users/interface/deactivated-user.intreface.ts
```typescript
export interface IDeactivatedUser {
  _id: string;
  userId: string;
  createdAt: Date;
  updatedAt: Date;
}

export interface IReturnedDeactivatedUser {
  _id: string;
  userId: string;
  createdAt: Date;
  updatedAt: Date;
}

```

## File: connectfy-auth/src/modules/users/deactivated-users/repo/deactivated-user.repo.ts
```typescript
import { Injectable } from '@nestjs/common';
import { InjectModel } from '@nestjs/mongoose';
import { Model } from 'mongoose';
import { DeactivatedUserDocument } from '../entity/deactivated-user.entity';
import { AddDeactivatedUserDto } from '../dto/add.deactivated-user.dto';
import { IReturnedDeactivatedUser } from '../interface/deactivated-user.intreface';
import { COLLECTIONS, BaseRepository } from 'connectfy-shared';

@Injectable()
export class DeactivatedUserRepository extends BaseRepository<
  DeactivatedUserDocument,
  IReturnedDeactivatedUser,
  AddDeactivatedUserDto
> {
  constructor(
    @InjectModel(COLLECTIONS.AUTH.USER.DEACTIVATED)
    protected readonly model: Model<DeactivatedUserDocument>,
  ) {
    super(model);
  }
}

```

## File: connectfy-auth/src/modules/users/deactivated-users/entity/deactivated-user.entity.ts
```typescript
import { Prop, Schema, SchemaFactory } from '@nestjs/mongoose';
import { IDeactivatedUser } from '../interface/deactivated-user.intreface';
import { v4 as uuid, validate } from 'uuid';
import { HydratedDocument } from 'mongoose';
import { t } from 'i18next';
import { LANGUAGE, COLLECTIONS } from 'connectfy-shared';

@Schema({
  timestamps: true,
  collection: COLLECTIONS.AUTH.USER.DEACTIVATED,
  toJSON: { virtuals: true, versionKey: false, getters: true },
  toObject: { virtuals: true, versionKey: false },
  autoIndex: process.env.NODE_ENV !== 'production',
  minimize: false,
  strict: true,
  strictQuery: true,
})
export class DeactivatedUserModel implements IDeactivatedUser {
  @Prop({
    type: String,
    default: () => uuid(),
    immutable: true,
  })
  _id: string;

  @Prop({
    type: String,
    required: [
      true,
      t('validation_messages.required', {
        lng: LANGUAGE.EN,
        field: 'userId',
      }),
    ],
    unique: true,
    index: true,
    immutable: true,
    ref: COLLECTIONS.AUTH.USER.USERS,
    validate: {
      validator: function (value: string | null): boolean {
        return !!(value && validate(value));
      },
      message: t('validation_messages.uuid', {
        lng: LANGUAGE.EN,
        field: 'userId',
      }),
    },
  })
  userId: string;

  createdAt: Date;
  updatedAt: Date;
}

export const DeactivatedUserSchema =
  SchemaFactory.createForClass(DeactivatedUserModel);

// ================================================
// INDEXES - Performance optimization
// ================================================

// Timestamp indexes (queries və analytics üçün)
DeactivatedUserSchema.index({ createdAt: -1 });
DeactivatedUserSchema.index({ updatedAt: -1 });

export type DeactivatedUserDocument = HydratedDocument<DeactivatedUserModel>;

```

## File: connectfy-auth/src/common/constants/environment-variables.ts
```typescript
import * as dotenv from 'dotenv';
import * as path from 'path';

const appEnv = process.env.NODE_ENV || 'remote';

const envPath = path.resolve(process.cwd(), `.env.${appEnv}`);

dotenv.config({ path: envPath });

export const ENVIRONMENT_VARIABLES = {
  // Core
  PORT: process.env.PORT,
  NODE_ENV: process.env.NODE_ENV,
  HOST: process.env.HOST,

  // Database
  DB_NAME: process.env.DB_NAME,
  MONGO_URI: process.env.MONGO_URI || '',

  // Kafka
  SERVICE_NAME: process.env.SERVICE_NAME,
  BROKER1: process.env.BROKER1 || '',
  BROKER2: process.env.BROKER2 || '',

  // Secret Keys
  JWT_ACCESS_SECRET: process.env.JWT_ACCESS_SECRET,
  JWT_REFRESH_SECRET: process.env.JWT_REFRESH_SECRET,
  JWT_ACCESS_EXPIRES_IN: process.env.JWT_ACCESS_EXPIRES_IN,
  JWT_REFRESH_EXPIRES_IN: process.env.JWT_REFRESH_EXPIRES_IN,

  FACE_DESCRIPTOR_KEY: process.env.FACE_DESCRIPTOR_KEY,
  USER_PRIVATE_KEY: process.env.USER_PRIVATE_KEY,

  FORGOT_PASSWORD_SECRET: process.env.FORGOT_PASSWORD_SECRET,
  CHANGE_EMAIL_SECRET: process.env.CHANGE_EMAIL_SECRET,
  RESTORE_ACCOUNT_SECRET: process.env.RESTORE_ACCOUNT_SECRET,

  GOOGLE_CLIENT_ID: process.env.GOOGLE_CLIENT_ID,
  GOOGLE_CLIENT_SECRET: process.env.GOOGLE_CLIENT_SECRET,
  GOOGLE_CALLBACK_URL: process.env.GOOGLE_CALLBACK_URL,
  GOOGLE_CLIENT_REDIRECT_URL: process.env.GOOGLE_CLIENT_REDIRECT_URL,

  // TCP
  ACCOUNT_SERVICE_HOST: process.env.ACCOUNT_SERVICE_HOST,
  ACCOUNT_SERVICE_PORT: Number(process.env.ACCOUNT_SERVICE_PORT),

  MESSENGER_SERVICE_HOST: process.env.MESSENGER_SERVICE_HOST,
  MESSENGER_SERVICE_PORT: Number(process.env.MESSENGER_SERVICE_PORT),

  RELATIONSHIP_SERVICE_HOST: process.env.RELATIONSHIP_SERVICE_HOST,
  RELATIONSHIP_SERVICE_PORT: Number(process.env.RELATIONSHIP_SERVICE_PORT),

  NOTIFICATION_ACTION_HISTORY_SERVICE_HOST:
    process.env.NOTIFICATION_ACTION_HISTORY_SERVICE_HOST,
  NOTIFICATION_ACTION_HISTORY_SERVICE_PORT: Number(
    process.env.NOTIFICATION_ACTION_HISTORY_SERVICE_PORT,
  ),
};

```

## File: connectfy-auth/src/common/functions/function.ts
```typescript
export function generateVerifyCode(): string {
  return Math.floor(100000 + Math.random() * 900000).toString();
}

```

## File: connectfy-auth/src/app-settings/app-settings.module.ts
```typescript
import { Global, Module } from '@nestjs/common';
import { TcpConnectionModule } from './tcp-connections/tcp-connection.module';
import { KafkaConnectionModule } from './kafka-connections/kafka-connection.module';

@Global()
@Module({
  imports: [TcpConnectionModule, KafkaConnectionModule],
  controllers: [],
  providers: [],
  exports: [TcpConnectionModule, KafkaConnectionModule],
})
export class AppSettingsModule {}

```

## File: connectfy-auth/src/app-settings/kafka-connections/loggin-kafka.server.ts
```typescript
import { ServerKafka } from '@nestjs/microservices';
import { Consumer } from '@nestjs/microservices/external/kafka.interface';

export class LoggingKafkaServer extends ServerKafka {
  async bindEvents(consumer: Consumer) {
    const patterns = [...this.messageHandlers.keys()];

    const admin = this.client?.admin();
    try {
      await admin?.connect();
      const existing = await admin?.listTopics();
      const existingSet = new Set(existing);
      const missing = patterns.filter((p) => !existingSet.has(p));
      const present = patterns.filter((p) => existingSet.has(p));

      this.logger.log(
        `📡 ${patterns.length} Kafka patterns registered — ${present.length} present in broker, ${missing.length} missing,`,
      );

      if (missing.length > 0) {
        this.logger.error(
          `❌ Missing topics in broker (subscribe will fail with "Server doesn't host this topic"):,`,
        );
        for (const t of missing) this.logger.error(`     • ${t}`);
        this.logger.error(
          `   → add them to nova-config/topics.txt and run ./init-topics.sh`,
        );
      }
    } catch (err) {
      this.logger.warn(
        `Could not pre-check topic existence via admin: ${(err as Error).message},`,
      );
    } finally {
      await admin?.disconnect().catch(() => undefined);
    }

    return super.bindEvents(consumer);
  }
}

```

## File: connectfy-auth/src/app-settings/kafka-connections/kafka-connection.service.ts
```typescript
import {
  Inject,
  Injectable,
  Logger,
  OnModuleDestroy,
  OnModuleInit,
} from '@nestjs/common';
import { ClientKafka } from '@nestjs/microservices';
import { lastValueFrom, timeout } from 'rxjs';
import { ClsService } from 'nestjs-cls';
import { CLS_KEYS, LANGUAGE, MICROSERVICE_NAMES } from 'connectfy-shared';

export type KafkaCallArgs<TPayload> = {
  topic: string; // pattern / topic name
  payload: TPayload;
  timeoutMs?: number; // only for send (RPC)
};

@Injectable()
export class KafkaConnectionService implements OnModuleInit, OnModuleDestroy {
  private readonly logger = new Logger(KafkaConnectionService.name);
  private connected = false;

  constructor(
    @Inject(MICROSERVICE_NAMES.KAFKA)
    private readonly client: ClientKafka,
    private readonly cls: ClsService,
  ) {}

  async onModuleInit() {
    if (this.connected) return;

    await this.client.connect();
    this.connected = true;

    this.logger.log('✅ Kafka client connected');
  }

  async onModuleDestroy() {
    // Nest ClientKafka-da close() mövcuddur (versiondan asılı ola bilər)
    // yoxdursa problem deyil, sadəcə connect qalacaq.
    try {
      await this.client.close?.();
      this.logger.log('🛑 Kafka client closed');
    } catch (e) {
      this.logger.warn('Kafka client close skipped/failed');
    }
  }

  /**
   * RPC (request-reply) üçün cavab topic-ini subscribe etmək lazımdır.
   * send() istifadə edəcəyin topic-ləri app start-da qeydiyyata al.
   */
  registerRequestPattern(topic: string) {
    this.client.subscribeToResponseOf(topic);
  }

  registerRequestPatterns(topic: string[]) {
    topic.forEach((e) => this.client.subscribeToResponseOf(e));
  }

  /**
   * RPC: cavab gözləyərək mesaj göndərir (MessagePattern tərəfdə).
   */
  async send<TPayload, TResponse = any>({
    topic,
    payload,
    timeoutMs = 15000,
  }: KafkaCallArgs<TPayload>): Promise<TResponse> {
    const obs$ = this.client
      .send<TResponse, TPayload>(topic, payload)
      .pipe(timeout(timeoutMs));

    return lastValueFrom(obs$);
  }

  /**
   * RPC safe: error olsa app-i yıxmır, null qaytarır.
   */
  async sendSafe<TPayload, TResponse = any>(
    args: KafkaCallArgs<TPayload>,
  ): Promise<TResponse | null> {
    try {
      return await this.send<TPayload, TResponse>(args);
    } catch (e) {
      this.logger.error(`❌ Kafka send failed: ${args.topic}`, e as any);
      return null;
    }
  }

  /**
   * Event: cavab gözləmədən mesaj göndərir (EventPattern tərəfdə).
   */
  async emit<TPayload>({
    topic,
    payload,
  }: KafkaCallArgs<TPayload>): Promise<void> {
    await lastValueFrom(this.client.emit(topic, payload));
  }

  /**
   * Event safe: error olsa false qaytarır.
   */
  async emitSafe<TPayload>(args: KafkaCallArgs<TPayload>): Promise<boolean> {
    try {
      await this.emit(args);
      return true;
    } catch (e) {
      this.logger.error(`❌ Kafka emit failed: ${args.topic}`, e as any);
      return false;
    }
  }

  /**
   * Bir topic-ə çox event göndərmək (parallel).
   */
  async emitMany<TPayload>(topic: string, payloads: TPayload[]): Promise<void> {
    await Promise.all(payloads.map((payload) => this.emit({ topic, payload })));
  }

  /**
   * Sadə context ötürmə (istəsən istifadə et).
   * Consumer tərəfdə __ctx-dən oxuya bilərsən.
   */
  async sendWithContext<TPayload extends object, TResponse = any>(args: {
    topic: string;
    payload: TPayload;
    ctx?: Record<string, any>;
    timeoutMs?: number;
  }): Promise<TResponse> {
    const { topic, payload, ctx, timeoutMs } = args;
    return this.send<TPayload & { __ctx?: any }, TResponse>({
      topic,
      payload: ctx ? ({ ...payload, __ctx: ctx } as any) : (payload as any),
      timeoutMs,
    });
  }

  async emitWithContext<TPayload extends object>(args: {
    topic: string;
    payload: TPayload;
    ctx?: Record<string, any>;
  }): Promise<void> {
    const user = this.cls?.get(CLS_KEYS.USER) || null;
    const lang = this.cls?.get(CLS_KEYS.LANG) || LANGUAGE.EN;
    const { topic, payload } = args;

    const enrichedPayload: Record<string, any> = {
      ...payload,
    };

    if (!enrichedPayload?._loggedUser) {
      enrichedPayload._loggedUser = user;
    }

    if (!enrichedPayload?._lang) {
      enrichedPayload._lang = lang;
    }

    return this.emit({ topic, payload: enrichedPayload });
  }
}

```

## File: connectfy-auth/src/app-settings/kafka-connections/kafka-connection.module.ts
```typescript
import { ENVIRONMENT_VARIABLES } from '@/src/common/constants/environment-variables';
import { Global, Module } from '@nestjs/common';
import { ClientsModule, Transport } from '@nestjs/microservices';
import { MICROSERVICE_NAMES } from 'connectfy-shared';
import { KafkaConnectionService } from './kafka-connection.service';

@Global()
@Module({
  imports: [
    ClientsModule.register([
      {
        name: MICROSERVICE_NAMES.KAFKA,
        transport: Transport.KAFKA,
        options: {
          client: {
            clientId: `${ENVIRONMENT_VARIABLES.SERVICE_NAME}-client`,
            brokers: [
              ENVIRONMENT_VARIABLES.BROKER1,
              ENVIRONMENT_VARIABLES.BROKER2,
            ].filter(Boolean),
          },
          consumer: {
            groupId: `${ENVIRONMENT_VARIABLES.SERVICE_NAME}-client-group`,
          },
        },
      },
    ]),
  ],
  providers: [KafkaConnectionService],
  exports: [KafkaConnectionService],
})
export class KafkaConnectionModule {}

```

## File: connectfy-auth/src/app-settings/tcp-connections/tcp-connection.service.ts
```typescript
import { Inject, Injectable } from '@nestjs/common';
import { ClientProxy } from '@nestjs/microservices';
import {
  CLS_KEYS,
  ISendWithContextClientParams,
  LANGUAGE,
  MICROSERVICE_NAMES,
  sendWithContext,
} from 'connectfy-shared';
import { ClsService } from 'nestjs-cls';

@Injectable()
export class TcpConnectionService {
  constructor(
    private readonly cls: ClsService,

    @Inject(MICROSERVICE_NAMES.TCP.ACCOUNT)
    private readonly accountService: ClientProxy,

    @Inject(MICROSERVICE_NAMES.TCP.MESSENGER)
    private readonly messengerService: ClientProxy,

    @Inject(MICROSERVICE_NAMES.TCP.RELATIONSHIP)
    private readonly relationshipService: ClientProxy,

    @Inject(MICROSERVICE_NAMES.TCP.NOTIFICATION_ACTION_HISTORY)
    private readonly notificationActionHistoryService: ClientProxy,
  ) {}

  // ===============================
  // Main send method
  // ===============================
  private async sendTcpWithContext({
    client,
    endpoint,
    payload,
  }: ISendWithContextClientParams) {
    const user = this.cls?.get(CLS_KEYS.USER) || null;
    const lang = this.cls?.get(CLS_KEYS.LANG) || LANGUAGE.EN;

    const enrichedPayload: Record<string, any> = {
      ...payload,
    };

    if (!enrichedPayload?._loggedUser) {
      enrichedPayload._loggedUser = user;
    }

    if (!enrichedPayload?._lang) {
      enrichedPayload._lang = lang;
    }

    return await sendWithContext({
      client,
      endpoint,
      payload: enrichedPayload,
      cls: this.cls,
    });
  }

  // ===============================
  // Account service
  // ===============================
  async account(opts: Omit<ISendWithContextClientParams, 'client' | 'cls'>) {
    return await this.sendTcpWithContext({
      client: this.accountService,
      ...opts,
    });
  }

  // ===============================
  // Messenger service
  // ===============================
  async messenger(opts: Omit<ISendWithContextClientParams, 'client' | 'cls'>) {
    return await this.sendTcpWithContext({
      client: this.messengerService,
      ...opts,
    });
  }

  // ===============================
  // Relationship service
  // ===============================
  async relationship(
    opts: Omit<ISendWithContextClientParams, 'client' | 'cls'>,
  ) {
    return await this.sendTcpWithContext({
      client: this.relationshipService,
      ...opts,
    });
  }

  // ===============================
  // Notification action history service
  // ===============================
  async notificationActionHistory(
    opts: Omit<ISendWithContextClientParams, 'client' | 'cls'>,
  ) {
    return await this.sendTcpWithContext({
      client: this.notificationActionHistoryService,
      ...opts,
    });
  }
}

```

## File: connectfy-auth/src/app-settings/tcp-connections/tcp-connection.module.ts
```typescript
import { Global, Module } from '@nestjs/common';
import { ClientsModule, Transport } from '@nestjs/microservices';
import { MICROSERVICE_NAMES } from 'connectfy-shared';
import { TcpConnectionService } from './tcp-connection.service';
import { ENVIRONMENT_VARIABLES } from '@/src/common/constants/environment-variables';

@Global()
@Module({
  imports: [
    ClientsModule.register({
      clients: [
        {
          name: MICROSERVICE_NAMES.TCP.ACCOUNT,
          transport: Transport.TCP,
          options: {
            host: ENVIRONMENT_VARIABLES.ACCOUNT_SERVICE_HOST,
            port: ENVIRONMENT_VARIABLES.ACCOUNT_SERVICE_PORT,
          },
        },
        {
          name: MICROSERVICE_NAMES.TCP.MESSENGER,
          transport: Transport.TCP,
          options: {
            host: ENVIRONMENT_VARIABLES.MESSENGER_SERVICE_HOST,
            port: ENVIRONMENT_VARIABLES.MESSENGER_SERVICE_PORT,
          },
        },
        {
          name: MICROSERVICE_NAMES.TCP.RELATIONSHIP,
          transport: Transport.TCP,
          options: {
            host: ENVIRONMENT_VARIABLES.RELATIONSHIP_SERVICE_HOST,
            port: ENVIRONMENT_VARIABLES.RELATIONSHIP_SERVICE_PORT,
          },
        },
        {
          name: MICROSERVICE_NAMES.TCP.NOTIFICATION_ACTION_HISTORY,
          transport: Transport.TCP,
          options: {
            host: ENVIRONMENT_VARIABLES.NOTIFICATION_ACTION_HISTORY_SERVICE_HOST,
            port: ENVIRONMENT_VARIABLES.NOTIFICATION_ACTION_HISTORY_SERVICE_PORT,
          },
        },
      ],
      isGlobal: true,
    }),
  ],
  providers: [TcpConnectionService],
  exports: [TcpConnectionService],
})
export class TcpConnectionModule {}

```

