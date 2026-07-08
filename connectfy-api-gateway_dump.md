# connectfy-api-gateway Source Dump

## File: connectfy-api-gateway/src/main.ts
```typescript
import helmet from 'helmet';
import { AppModule } from './app.module';
import { NestFactory } from '@nestjs/core';
import * as session from 'express-session';
import * as cookieParser from 'cookie-parser';
import { LoggingInterceptor } from './interceptors/logging.interceptor';
import { AllExceptionsFilter } from './common/exception-filters/all.filter';
import { ENVIRONMENT_VARIABLES } from './common/constants/environment-variables';
import { REDIS_KEYS } from 'connectfy-shared';
import { RedisStore } from 'connect-redis';
import { DeviceIdInterceptor } from './interceptors/deviceId.interceptor';
import { ClsService } from 'nestjs-cls';
import { MicroserviceOptions, Transport } from '@nestjs/microservices';
import { LoggingKafkaServer } from './app-settings/kafka-connections/loggin-kafka.server';

async function bootstrap() {
  const app = await NestFactory.create(AppModule);

  const cls = app.get(ClsService);
  const PORT = Number(ENVIRONMENT_VARIABLES.PORT);
  const NODE_ENV = String(ENVIRONMENT_VARIABLES.NODE_ENV);
  const CLIENT_URL = String(ENVIRONMENT_VARIABLES.CLIENT_URL);
  const SESSION_SECRET_KEY = String(ENVIRONMENT_VARIABLES.SESSION_SECRET_KEY);

  const redisClient = app.get(REDIS_KEYS.REDIS_CLIENT);

  const RedisSessionStore = new RedisStore({
    client: {
      get: (key: string) => redisClient.get(key),
      set: (
        key: string,
        value: string,
        options?: { expiration: { type: 'EX' | 'PX'; value: number } },
      ) => {
        const expType = options?.expiration?.type;
        const expValue = options?.expiration?.value;
        if (expType === 'PX') {
          return redisClient.set(key, value, 'PX', expValue);
        }
        if (expType === 'EX') {
          return redisClient.set(key, value, 'EX', expValue);
        }
        return redisClient.set(key, value, 'EX', 60 * 15);
      },
      del: (key: string) => redisClient.del(key),
    } as any,
    prefix: 'sess:',
  });

  // Prefix
  app.setGlobalPrefix('/api/v1');

  // Helmet
  app.use(helmet());

  // CORS
  app.enableCors({
    origin: CLIENT_URL ?? 'http://localhost:4800',
    credentials: true,
    methods: ['GET', 'POST', 'PATCH', 'DELETE', 'OPTIONS'],
    allowedHeaders: [
      'Content-Type',
      'Authorization',
      'Accept',
      'x-device-id',
      'X-Requested-With',
    ],
    exposedHeaders: ['Set-Cookie'],
    optionsSuccessStatus: 204,
  });

  // Cookie Parser
  app.use(cookieParser());

  // Session
  app.use(
    session({
      store: RedisSessionStore,
      name: 'n_sid',
      secret: SESSION_SECRET_KEY ?? 'session-secret-key',
      resave: false,
      saveUninitialized: false,
      cookie: {
        httpOnly: true,
        secure: NODE_ENV === 'production',
        sameSite: NODE_ENV === 'production' ? 'none' : 'lax',
        maxAge: 1000 * 60 * 60,
        domain: NODE_ENV === 'production' ? undefined : undefined,
      },
    }),
  );

  // Interceptor
  app.useGlobalInterceptors(new LoggingInterceptor());
  app.useGlobalInterceptors(new DeviceIdInterceptor(cls));

  // Filter
  app.useGlobalFilters(new AllExceptionsFilter());

  app.connectMicroservice<MicroserviceOptions>({
    strategy: new LoggingKafkaServer({
      client: {
        clientId: 'connectfy-api-gateway',
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
        groupId: 'consumer-connectfy-api-gateway',
        allowAutoTopicCreation: false,
      },
      run: {
        autoCommit: false,
      },
    }),
  });

  await app.startAllMicroservices();

  console.log('✅ Kafka Microservice is running');

  await app.listen(PORT);

  console.log(`✅ NODE_ENV => `, NODE_ENV);
  console.log(`✅ Server is working on ${PORT} port`);
}
bootstrap();

```

## File: connectfy-api-gateway/src/i18n.ts
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

## File: connectfy-api-gateway/src/app.module.ts
```typescript
import { Module } from '@nestjs/common';
import { ConfigModule } from '@nestjs/config';
import { EventEmitterModule } from '@nestjs/event-emitter';
import { ClsModule } from 'nestjs-cls';
import { JwtModule } from '@nestjs/jwt';
import { AppSettingsModule } from './app-settings/app-settings.module';
import { ModulesModule } from './modules/modules.module';

@Module({
  imports: [
    ConfigModule.forRoot({
      envFilePath: `.env.${process.env.NODE_ENV || 'development'}`,
      isGlobal: true,
    }),
    ClsModule.forRoot({
      global: true, // <– makes ClsService available everywhere
      middleware: {
        mount: true, // <– wraps every HTTP request in a CLS context
        // optional setup if you want to pre-fill something from req:
        // setup: (cls, req) => {
        //   cls.set('requestId', req.headers['x-request-id']);
        // },
      },
    }),
    JwtModule.register({ global: true }),
    EventEmitterModule.forRoot({ global: true }),

    // /src/app-settings
    AppSettingsModule,
    // /src/modules
    ModulesModule,
  ],
})
export class AppModule {}

```

## File: connectfy-api-gateway/src/interceptors/deviceId.interceptor.ts
```typescript
import {
  Injectable,
  NestInterceptor,
  ExecutionContext,
  CallHandler,
} from '@nestjs/common';
import { Observable } from 'rxjs';
import { Request } from 'express';
import {
  BaseException,
  CLS_KEYS,
  ExceptionMessages,
  HttpStatus,
} from 'connectfy-shared';
import { validate } from 'uuid';
import { ClsService } from 'nestjs-cls';

@Injectable()
export class DeviceIdInterceptor implements NestInterceptor {
  constructor(private readonly cls: ClsService) {}

  intercept(context: ExecutionContext, next: CallHandler): Observable<any> {
    const httpContext = context.switchToHttp();
    const request = httpContext.getRequest<Request>();

    const deviceId = request.headers['x-device-id'];

    if (!deviceId || Array.isArray(deviceId)) {
      throw new BaseException(
        ExceptionMessages.UNAUTHORIZED_MESSAGE,
        HttpStatus.UNAUTHORIZED,
        { navigate: true },
      );
    }

    if (!validate(deviceId)) {
      throw new BaseException(
        'Invalid Device ID format',
        HttpStatus.BAD_REQUEST,
        { navigate: true },
      );
    }

    this.cls.set(CLS_KEYS.DEVICE_ID, deviceId);

    return next.handle();
  }
}

```

## File: connectfy-api-gateway/src/interceptors/logging.interceptor.ts
```typescript
import {
  Injectable,
  NestInterceptor,
  ExecutionContext,
  CallHandler,
} from '@nestjs/common';
import { Observable } from 'rxjs';
import { tap } from 'rxjs/operators';
import { Request, Response } from 'express';

@Injectable()
export class LoggingInterceptor implements NestInterceptor {
  intercept(context: ExecutionContext, next: CallHandler): Observable<any> {
    const httpContext = context.switchToHttp();
    const request = httpContext.getRequest<Request>();
    const response = httpContext.getResponse<Response>();

    const now = Date.now();
    const startTime = new Date();

    // Məlumatları toplayırıq
    const controllerName = context.getClass().name;
    const handlerName = context.getHandler().name;
    const method = request.method;
    const url = request.url;
    const clientIP = request.ip || request.connection.remoteAddress;
    const userAgent = request.get('user-agent') || 'N/A';

    // Request ID (mövcuddursa istifadə et)
    const requestId = request.headers['x-request-id'] || 'N/A';

    // Rəng kodları
    const colors = {
      reset: '\x1b[0m',
      bright: '\x1b[1m',
      dim: '\x1b[2m',
      red: '\x1b[31m',
      green: '\x1b[32m',
      yellow: '\x1b[33m',
      blue: '\x1b[34m',
      magenta: '\x1b[35m',
      cyan: '\x1b[36m',
      white: '\x1b[37m',
      bgGray: '\x1b[100m',
    };

    // Konsol log formatı
    console.log(
      '\n' +
        colors.bgGray +
        '══════════════════════════════════════════════════════════' +
        colors.reset,
    );

    // Başlıq
    console.log(
      colors.bright + colors.cyan + '🌐 HTTP REQUEST LOG' + colors.reset,
    );
    console.log(
      colors.dim +
        '╔════════════════════════════════════════════════════════╗' +
        colors.reset,
    );

    // Əsas məlumatlar
    this.printLogEntry('Start Time', startTime.toISOString(), colors.cyan);
    this.printLogEntry('Controller', controllerName, colors.green);
    this.printLogEntry('Handler', handlerName, colors.green);
    this.printLogEntry('Method', method, this.getMethodColor(method));
    this.printLogEntry('URL', url, colors.dim + colors.white);

    // Əlavə məlumatlar
    this.printLogEntry('Request ID', requestId.toString(), colors.magenta);
    this.printLogEntry('Client IP', clientIP ?? 'UNKONWN IP', colors.blue);
    this.printLogEntry(
      'User Agent',
      userAgent.substring(0, 60) + (userAgent.length > 60 ? '...' : ''),
      colors.dim + colors.white,
    );

    console.log(
      colors.dim +
        '╚════════════════════════════════════════════════════════╝' +
        colors.reset,
    );

    return next.handle().pipe(
      tap({
        next: () => {
          const duration = Date.now() - now;
          const statusCode = response.statusCode;

          console.log(
            colors.dim +
              '╔════════════════════════════════════════════════════════╗' +
              colors.reset,
          );

          this.printLogEntry(
            'Status Code',
            statusCode.toString(),
            this.getStatusCodeColor(statusCode),
          );
          this.printLogEntry(
            'Duration',
            `${duration}ms`,
            this.getDurationColor(duration),
          );
          this.printLogEntry('End Time', new Date().toISOString(), colors.cyan);

          console.log(
            colors.dim +
              '╚════════════════════════════════════════════════════════╝' +
              colors.reset,
          );
          console.log(
            colors.bgGray +
              '══════════════════════════════════════════════════════════' +
              colors.reset +
              '\n',
          );
        },
        error: (error) => {
          const duration = Date.now() - now;
          const statusCode = error.status || 500;

          console.log(
            colors.dim +
              '╔════════════════════════════════════════════════════════╗' +
              colors.reset,
          );

          this.printLogEntry('Status Code', statusCode.toString(), colors.red);
          this.printLogEntry(
            'Error',
            error.message || 'Unknown error',
            colors.red,
          );
          this.printLogEntry('Duration', `${duration}ms`, colors.red);
          this.printLogEntry('End Time', new Date().toISOString(), colors.cyan);

          console.log(
            colors.dim +
              '╚════════════════════════════════════════════════════════╗' +
              colors.reset,
          );
          console.log(
            colors.bgGray +
              '══════════════════════════════════════════════════════════' +
              colors.reset +
              '\n',
          );
        },
      }),
    );
  }

  private printLogEntry(
    label: string,
    value: string,
    color: string = '\x1b[0m',
  ): void {
    const colors = {
      reset: '\x1b[0m',
      dim: '\x1b[2m',
    };

    console.log(
      colors.dim +
        '║ ' +
        colors.reset +
        colors.dim +
        label.padEnd(15, ' ') +
        colors.reset +
        ' ' +
        colors.dim +
        ':' +
        colors.reset +
        ' ' +
        color +
        value +
        colors.reset,
    );
  }

  private getMethodColor(method: string): string {
    const colors = {
      reset: '\x1b[0m',
      green: '\x1b[32m',
      blue: '\x1b[34m',
      yellow: '\x1b[33m',
      red: '\x1b[31m',
      magenta: '\x1b[35m',
    };

    switch (method.toUpperCase()) {
      case 'GET':
        return colors.green;
      case 'POST':
        return colors.yellow;
      case 'PUT':
        return colors.blue;
      case 'DELETE':
        return colors.red;
      case 'PATCH':
        return colors.magenta;
      default:
        return colors.reset;
    }
  }

  private getStatusCodeColor(statusCode: number): string {
    const colors = {
      reset: '\x1b[0m',
      green: '\x1b[32m',
      yellow: '\x1b[33m',
      red: '\x1b[31m',
    };

    if (statusCode >= 200 && statusCode < 300) return colors.green;
    if (statusCode >= 300 && statusCode < 400) return colors.yellow;
    if (statusCode >= 400) return colors.red;
    return colors.reset;
  }

  private getDurationColor(duration: number): string {
    const colors = {
      reset: '\x1b[0m',
      green: '\x1b[32m',
      yellow: '\x1b[33m',
      red: '\x1b[31m',
    };

    if (duration < 200) return colors.green;
    if (duration < 500) return colors.yellow;
    return colors.red;
  }
}

```

## File: connectfy-api-gateway/src/modules/modules.module.ts
```typescript
import { Module } from '@nestjs/common';
import { AccountModule } from './account/account.module';
import { AuthModule } from './auth/auth.module';
import { RelationshipModule } from './relationship/relationship.module';
import { NotificationActionHistoryModule } from './notification-action-history/notification-action-history.module';

import { PresenceModule } from './presence/presence.module';

@Module({
  imports: [
    AccountModule,
    AuthModule,
    RelationshipModule,
    NotificationActionHistoryModule,
    PresenceModule,
  ],
  controllers: [],
  providers: [],
  exports: [],
})
export class ModulesModule {}

```

## File: connectfy-api-gateway/src/modules/presence/presence.controller.ts
```typescript
import { Controller, Post, UseGuards } from '@nestjs/common';
import { AuthGuard } from '@/src/guards/auth.guard';
import { PresenceService } from './presence.service';
import { ClsService } from 'nestjs-cls';
import { CLS_KEYS, ILoggedUser } from 'connectfy-shared';

@UseGuards(AuthGuard)
@Controller('user/heartbeat')
export class PresenceController {
  constructor(
    private readonly service: PresenceService,
    private readonly cls: ClsService,
  ) {}

  @Post()
  async heartbeat() {
    const user = this.cls.get<ILoggedUser>(CLS_KEYS.USER);
    if (!user || !user._id) return { ok: false };
    return this.service.handleHeartbeat(user._id);
  }
}

```

## File: connectfy-api-gateway/src/modules/presence/presence.module.ts
```typescript
import { Global, Module } from '@nestjs/common';
import { PresenceService } from './presence.service';
import { PresenceController } from './presence.controller';

@Global()
@Module({
  controllers: [PresenceController],
  providers: [PresenceService],
  exports: [PresenceService],
})
export class PresenceModule {}

```

## File: connectfy-api-gateway/src/modules/presence/presence.service.ts
```typescript
import { Injectable, Inject } from '@nestjs/common';
import { REDIS_KEYS } from 'connectfy-shared';
import Redis from 'ioredis';

@Injectable()
export class PresenceService {
  constructor(
    @Inject(REDIS_KEYS.REDIS_CLIENT)
    private readonly redisClient: Redis,
  ) {}

  async handleHeartbeat(userId: string) {
    await this.redisClient.setex(`presence:online:${userId}`, 60, 1);
    return { ok: true };
  }

  async areUsersOnline(userIds: string[]): Promise<boolean[]> {
    if (!userIds || userIds.length === 0) return [];
    const keys = userIds.map((id) => `presence:online:${id}`);
    const results = await this.redisClient.mget(keys);
    return results.map((res) => res !== null);
  }

  async removePresence(userId: string) {
    await this.redisClient.del(`presence:online:${userId}`);
  }
}

```

## File: connectfy-api-gateway/src/modules/auth/auth.module.ts
```typescript
import { Module } from '@nestjs/common';
import { UserModule } from './user/user.module';
import { AuthModule as AuthServiceModule } from './auth/auth.module';

@Module({
  imports: [AuthServiceModule, UserModule],
})
export class AuthModule {}

```

## File: connectfy-api-gateway/src/modules/auth/user/user.controller.ts
```typescript
import {
  Body,
  Controller,
  Patch,
  Post,
  Req,
  Res,
  Session,
  UseGuards,
} from '@nestjs/common';
import { AuthGuard } from '@guards/auth.guard';
import { UserService } from './user.service';
import { Request, Response } from 'express';

@Controller('user')
export class UserController {
  constructor(private readonly service: UserService) {}

  @UseGuards(AuthGuard)
  @Post('me')
  async me() {
    return this.service.getMe();
  }

  @UseGuards(AuthGuard)
  @Patch('change-username')
  async changeUsername(@Body() data: any) {
    return this.service.changeUsername(data);
  }

  @UseGuards(AuthGuard)
  @Patch('change-email')
  async changeEmail(@Body() data: any) {
    return this.service.changeEmail(data);
  }

  @UseGuards(AuthGuard)
  @Patch('change-email/verify')
  async verifyEmailChange(@Body() data: any) {
    return this.service.verifyEmailChange(data);
  }

  @UseGuards(AuthGuard)
  @Patch('change-password')
  async changePassword(@Body() data: any) {
    return this.service.changePassword(data);
  }

  @UseGuards(AuthGuard)
  @Patch('change-phone-number')
  async changePhoneNumber(@Body() data: any) {
    return this.service.changePhoneNumber(data);
  }

  @Post('check-unique')
  async checkUnique(@Body() data: any) {
    return this.service.checkUnique(data);
  }

  @UseGuards(AuthGuard)
  @Patch('two-factor')
  async updateTwoFactorAuth(@Body() data: any) {
    return this.service.updateTwoFactorAuth(data);
  }

  @UseGuards(AuthGuard)
  @Post('delete-account')
  async deleteAccount(
    @Body() data,
    @Req() req: Request,
    @Res({ passthrough: true }) res: Response,
    @Session() session: Record<string, any>,
  ) {
    const authHeader = req.headers.authorization;

    const [type, token] = (authHeader ?? '').split(' ');
    const accessToken = type === 'Bearer' ? token : '';

    const result = await this.service.deleteAccount(data, accessToken);

    res.clearCookie('refresh_token');
    session.destroy();

    return result;
  }

  @UseGuards(AuthGuard)
  @Post('deactivate-account')
  async deactivateAccount(
    @Body() data,
    @Req() req: Request,
    @Session() session: Record<string, any>,
  ) {
    const authHeader = req.headers.authorization;

    const [type, token] = (authHeader ?? '').split(' ');
    const accessToken = type === 'Bearer' ? token : '';

    const result = await this.service.deactivateAccount(data, accessToken);

    session.destroy();

    return result;
  }
}

```

## File: connectfy-api-gateway/src/modules/auth/user/user.module.ts
```typescript
import { Module } from '@nestjs/common';
import { UserController } from './user.controller';
import { ConfigModule } from '@nestjs/config';
import { UserService } from './user.service';

@Module({
  imports: [ConfigModule],
  providers: [UserService],
  controllers: [UserController],
})
export class UserModule {}

```

## File: connectfy-api-gateway/src/modules/auth/user/user.service.ts
```typescript
import { Injectable } from '@nestjs/common';
import { CACHE_KEYS, CLS_KEYS } from 'connectfy-shared';
import { CacheService } from '@/src/app-settings/cache/cache.service';
import { TcpConnectionService } from '@/src/app-settings/tcp-connections/tcp-connection.service';
import { ClsService } from 'nestjs-cls';

@Injectable()
export class UserService {
  constructor(
    private readonly tcpConnectionService: TcpConnectionService,
    private readonly cacheService: CacheService,
    private readonly cls: ClsService,
  ) {}

  async getMe() {
    const user = await this.cls.get(CLS_KEYS.USER);
    if (!user || !user._id) return undefined;

    const cacheKey = CACHE_KEYS.AUTH.USER(user._id);
    const cached = await this.cacheService.get<Record<string, any>>(cacheKey);

    const { status, role, ...rest } = cached || {};

    return rest;
  }

  async changeUsername(data: any) {
    const user = this.cls.get(CLS_KEYS.USER);
    const res = await this.tcpConnectionService.auth({
      endpoint: 'user/change-username',
      payload: data,
    });

    const cacheKey = CACHE_KEYS.AUTH.USER(user._id);
    const cacheKeyProfile = CACHE_KEYS.ACCOUNT.PROFILE(user._id);
    await this.cacheService.removeMany([cacheKey, cacheKeyProfile]);

    return res;
  }

  async changeEmail(data: any) {
    return this.tcpConnectionService.auth({
      endpoint: 'user/change-email',
      payload: data,
    });
  }

  async verifyEmailChange(data: any) {
    const user = this.cls.get(CLS_KEYS.USER);
    const res = await this.tcpConnectionService.auth({
      endpoint: 'user/change-email/verify',
      payload: data,
    });

    const cacheKey = CACHE_KEYS.AUTH.USER(user._id);
    await this.cacheService.remove(cacheKey);

    return res;
  }

  async changePassword(data: any) {
    return this.tcpConnectionService.auth({
      endpoint: 'user/change-password',
      payload: data,
    });
  }

  async changePhoneNumber(data: any) {
    const user = this.cls.get(CLS_KEYS.USER);
    const res = await this.tcpConnectionService.auth({
      endpoint: 'user/change-phone-number',
      payload: data,
    });

    const cacheKey = CACHE_KEYS.AUTH.USER(user._id);
    await this.cacheService.remove(cacheKey);

    return res;
  }

  async checkUnique(data: any) {
    return this.tcpConnectionService.auth({
      endpoint: 'user/check-unique',
      payload: data,
    });
  }

  async updateTwoFactorAuth(data: any) {
    const user = this.cls.get(CLS_KEYS.USER);
    const res = await this.tcpConnectionService.auth({
      endpoint: 'user/two-factor',
      payload: data,
    });

    const cacheKey = CACHE_KEYS.AUTH.USER(user._id);
    await this.cacheService.remove(cacheKey);

    return res;
  }

  async deleteAccount(data: any, accessToken: string) {
    const user = this.cls.get(CLS_KEYS.USER);
    const userId = user?._id;

    const res = await this.tcpConnectionService.auth({
      endpoint: 'user/delete-account',
      payload: { ...data, userId },
    });

    this.cacheService.clearUserCache(userId, accessToken);

    return res;
  }

  async deactivateAccount(data: any, accessToken: string) {
    const user = this.cls.get(CLS_KEYS.USER);
    const userId = user?._id;

    const res = await this.tcpConnectionService.auth({
      endpoint: 'user/deactivate-account',
      payload: data,
    });

    this.cacheService.clearUserCache(userId, accessToken);

    return res;
  }
}

```

## File: connectfy-api-gateway/src/modules/auth/auth/auth.controller.ts
```typescript
import {
  Body,
  Controller,
  Post,
  Req,
  Res,
  Session,
  UseGuards,
  Get,
} from '@nestjs/common';
import { Request, Response, CookieOptions, response } from 'express';
import { AuthGuard } from '@guards/auth.guard';
import {
  BaseException,
  CLS_KEYS,
  ExceptionMessages,
  EXPIRE_DATES,
  HttpStatus,
  LANGUAGE,
} from 'connectfy-shared';
import { ENVIRONMENT_VARIABLES } from '@/src/common/constants/environment-variables';
import { extractRequestData } from '@/src/common/functions/request';
import { AuthService } from './auth.service';
import { ClsService } from 'nestjs-cls';

@Controller('auth')
export class AuthController {
  constructor(
    private readonly service: AuthService,
    private readonly cls: ClsService,
  ) {}

  private setRefreshCookie(token: string, res: Response) {
    const isProd = ENVIRONMENT_VARIABLES.NODE_ENV === 'production';

    const cookieOptions: CookieOptions = {
      maxAge: EXPIRE_DATES.TOKEN.ONE_MONTH,
      httpOnly: true,
      secure: isProd,
      sameSite: isProd ? 'none' : 'lax',
    };

    res.cookie('refresh_token', token, cookieOptions);
  }

  private saveSession(session: any): Promise<void> {
    return new Promise((resolve, reject) => {
      session.save((err) => (err ? reject(err) : resolve()));
    });
  }

  @Post('signup')
  async signup(@Body() data, @Session() session: Record<string, any>) {
    const res = await this.service.signup(data);

    session.unverifiedUser = res.unverifiedUser;
    session.verifyCode = res.verifyCode;
    session.cookie.maxAge = 1000 * 60 * 15;

    await this.saveSession(session);

    return { statusCode: 200 };
  }

  @Post('signup/verify')
  async verifySignup(
    @Body() data,
    @Session() session,
    @Req() req: Request,
    @Res({ passthrough: true }) res: Response,
  ) {
    const lang = this.cls.get<LANGUAGE>(CLS_KEYS.LANG);
    const deviceId = this.cls.get<string>(CLS_KEYS.DEVICE_ID);
    const unverifiedUser = session.unverifiedUser;
    const code = session.verifyCode;

    if (!unverifiedUser || !code) {
      throw new BaseException(
        ExceptionMessages.NOT_FOUND_MESSAGE(lang),
        HttpStatus.NOT_FOUND,
        { navigate: true },
      );
    }

    data.deviceId = deviceId;
    data.code = session.verifyCode;
    data.unverifiedUser = session.unverifiedUser;
    data.requestData = extractRequestData(req);

    const result = await this.service.verifySignup(data);

    if (result.refresh_token) {
      this.setRefreshCookie(result.refresh_token, res);

      delete session.unverifiedUser;
      delete session.verifyCode;
    }

    return { access_token: result.access_token };
  }

  @Post('signup/verify/resend')
  async resendSignupVerify(@Session() session: Record<string, any>) {
    const lang = this.cls.get<LANGUAGE>(CLS_KEYS.LANG);
    const payload = session.unverifiedUser;

    if (!payload) {
      throw new BaseException(
        ExceptionMessages.NOT_FOUND_MESSAGE(lang),
        HttpStatus.NOT_FOUND,
        { navigate: true },
      );
    }

    const res = await this.service.resendSignupVerify(payload);

    session.verifyCode = res.verifyCode;
    session.cookie.maxAge = 1000 * 60 * 15;

    await this.saveSession(session);

    return { statusCode: 200 };
  }

  @Post('login')
  async login(
    @Body() data,
    @Req() req: Request,
    @Res({ passthrough: true }) res: Response,
    @Session() session: Record<string, any>,
  ) {
    const deviceId = this.cls.get<string>(CLS_KEYS.DEVICE_ID);
    data.requestData = extractRequestData(req);
    data.deviceId = deviceId;

    const result = await this.service.login(data);

    let responseData;

    if (result.isTwoFactorEnabled) {
      const { code, userId, ...rest } = result;
      session.twoFaCode = code;
      session.userId = userId;
      session.cookie.maxAge = 1000 * 60 * 15;

      await this.saveSession(session);

      responseData = rest;
    }

    if (result.refresh_token) {
      this.setRefreshCookie(result.refresh_token, res);

      const { refresh_token, ...rest } = result;
      responseData = rest;
    }

    return responseData;
  }

  @Post('login/verify')
  async verifyLogin(
    @Body() data,
    @Req() req: Request,
    @Res({ passthrough: true }) res: Response,
    @Session() session: Record<string, any>,
  ) {
    const lang = this.cls.get<LANGUAGE>(CLS_KEYS.LANG);
    const deviceId = this.cls.get<string>(CLS_KEYS.DEVICE_ID);
    const userId = session.userId;
    const twoFaCode = session.twoFaCode;

    if (!userId || !twoFaCode) {
      throw new BaseException(
        ExceptionMessages.NOT_FOUND_MESSAGE(lang),
        HttpStatus.NOT_FOUND,
        { navigate: true },
      );
    }

    data.requestData = extractRequestData(req);
    data.userId = userId;
    data.twoFaCode = twoFaCode;
    data.deviceId = deviceId;

    const result = await this.service.verifyLogin(data);

    if (result.refresh_token) {
      this.setRefreshCookie(result.refresh_token, res);

      delete session.twoFaCode;
      delete session.userId;
    }

    const { refresh_token, ...rest } = result;

    return rest;
  }

  @Post('google/login')
  async googleAuthLogin(
    @Body() data,
    @Req() request: Request,
    @Res({ passthrough: true }) response: Response,
  ) {
    const deviceId = this.cls.get<string>(CLS_KEYS.DEVICE_ID);
    data.requestData = extractRequestData(request);
    data.deviceId = deviceId;

    const res = await this.service.googleLogin(data);

    if (res.refresh_token) {
      this.setRefreshCookie(res.refresh_token, response);
    }

    const { refresh_token, ...rest } = res;

    return rest;
  }

  @Post('google/signup')
  async googleAuthSignup(
    @Body() data,
    @Req() request: Request,
    @Res({ passthrough: true }) response: Response,
  ) {
    const deviceId = this.cls.get<string>(CLS_KEYS.DEVICE_ID);
    data.requestData = extractRequestData(request);
    data.deviceId = deviceId;

    const res = await this.service.googleSignup(data);

    if (res.refresh_token) {
      this.setRefreshCookie(res.refresh_token, response);
    }

    return { access_token: res.access_token };
  }

  @Post('refresh')
  async refresh(
    @Req() req: Request,
    @Res({ passthrough: true }) res: Response,
  ) {
    const deviceId = this.cls.get<string>(CLS_KEYS.DEVICE_ID);

    const payload = {
      deviceId,
      refresh_token: req.cookies?.refresh_token,
      requestData: extractRequestData(req),
    };

    const result = await this.service.refreshToken(payload);

    this.setRefreshCookie(result.refresh_token, res);

    return { access_token: result.access_token };
  }

  @UseGuards(AuthGuard)
  @Post('logout')
  async logout(
    @Req() req: Request,
    @Res({ passthrough: true }) res: Response,
    @Session() session: Record<string, any>,
  ) {
    const deviceId = this.cls.get<string>(CLS_KEYS.DEVICE_ID);
    const authHeader = req.headers.authorization;

    const [type, token] = (authHeader ?? '').split(' ');
    const accessToken = type === 'Bearer' ? token : '';

    const result = await this.service.logout({ deviceId }, accessToken);

    res.clearCookie('refresh_token');
    session.destroy();

    return result;
  }

  @Post('restore-account')
  async restoreAccount(
    @Body() data,
    @Req() req: Request,
    @Res({ passthrough: true }) res: Response,
  ) {
    const deviceId = this.cls.get<string>(CLS_KEYS.DEVICE_ID);
    data.requestData = extractRequestData(req);
    data.deviceId = deviceId;
    const result = await this.service.restoreAccount(data);

    if (result.refresh_token) {
      this.setRefreshCookie(result.refresh_token, res);
    }

    const { refresh_token, ...rest } = result;

    return rest;
  }

  @Post('forgot-password')
  async forgotPassword(@Body() data) {
    return this.service.forgotPassword(data);
  }

  @Post('reset-password')
  async resetPassword(@Body() data) {
    return this.service.resetPassword(data);
  }

  @UseGuards(AuthGuard)
  @Post('authenticate-user')
  async authenticateUser(@Body() data) {
    return this.service.authenticateUser(data);
  }

  @Post('is-valid-token')
  async isValidToken(@Body() data) {
    return this.service.isValidToken(data);
  }

  @Get('internal/verify')
  @UseGuards(AuthGuard)
  async internalVerify() {
    return { ok: true };
  }
}

```

## File: connectfy-api-gateway/src/modules/auth/auth/auth.module.ts
```typescript
import { Module } from '@nestjs/common';
import { AuthController } from './auth.controller';
import { ConfigModule } from '@nestjs/config';
import { AuthService } from './auth.service';

@Module({
  imports: [ConfigModule],
  providers: [AuthService],
  controllers: [AuthController],
})
export class AuthModule {}

```

## File: connectfy-api-gateway/src/modules/auth/auth/auth.service.ts
```typescript
import { Injectable } from '@nestjs/common';
import {
  BaseException,
  CACHE_KEYS,
  CLS_KEYS,
  ExceptionMessages,
  HttpStatus,
} from 'connectfy-shared';
import { CacheService } from '@/src/app-settings/cache/cache.service';
import { TcpConnectionService } from '@/src/app-settings/tcp-connections/tcp-connection.service';
import { ClsService } from 'nestjs-cls';

import { PresenceService } from '../../presence/presence.service';

@Injectable()
export class AuthService {
  constructor(
    private readonly tcpConnectionService: TcpConnectionService,
    private readonly cacheService: CacheService,
    private readonly cls: ClsService,
    private readonly presenceService: PresenceService,
  ) {}

  async signup(data: any) {
    return this.tcpConnectionService.auth({
      endpoint: 'auth/signup',
      payload: data,
    });
  }

  async verifySignup(data: any) {
    return this.tcpConnectionService.auth({
      endpoint: 'auth/verify-signup',
      payload: data,
    });
  }

  async resendSignupVerify(payload: any) {
    if (!payload) {
      throw new BaseException(
        ExceptionMessages.CONFLICT_MESSAGE,
        HttpStatus.CONFLICT,
        { navigate: true },
      );
    }

    return this.tcpConnectionService.auth({
      endpoint: 'auth/verify-signup/resend',
      payload,
    });
  }

  async login(data: any) {
    return this.tcpConnectionService.auth({
      endpoint: 'auth/login',
      payload: data,
    });
  }

  async verifyLogin(data: any) {
    return this.tcpConnectionService.auth({
      endpoint: 'auth/verify-login',
      payload: data,
    });
  }

  async googleLogin(data: any) {
    return this.tcpConnectionService.auth({
      endpoint: 'auth/google/login',
      payload: data,
    });
  }

  async googleSignup(data: any) {
    return this.tcpConnectionService.auth({
      endpoint: 'auth/google/signup',
      payload: data,
    });
  }

  async forgotPassword(data: any) {
    return this.tcpConnectionService.auth({
      endpoint: 'auth/forgot-password',
      payload: data,
    });
  }

  async resetPassword(data: any) {
    return this.tcpConnectionService.auth({
      endpoint: 'auth/reset-password',
      payload: data,
    });
  }

  async refreshToken(data: any) {
    return this.tcpConnectionService.auth({
      endpoint: 'auth/refreshToken',
      payload: data,
    });
  }

  async logout(data: any, accessToken: string) {
    const user = this.cls.get(CLS_KEYS.USER);
    const userId = user?._id;

    const res = await this.tcpConnectionService.auth({
      endpoint: 'auth/logout',
      payload: data,
    });

    this.cacheService.clearUserCache(userId, accessToken);
    if (userId) {
      await this.presenceService.removePresence(userId);
    }

    return res;
  }

  async restoreAccount(data: any) {
    return this.tcpConnectionService.auth({
      endpoint: 'auth/restore-account',
      payload: data,
    });
  }

  async authenticateUser(data: any) {
    return this.tcpConnectionService.auth({
      endpoint: 'auth/authenticate-user',
      payload: data,
    });
  }

  async isValidToken(data: any) {
    return this.tcpConnectionService.auth({
      endpoint: 'auth/is-valid-token',
      payload: data,
    });
  }
}

```

## File: connectfy-api-gateway/src/modules/account/account.module.ts
```typescript
import { Module } from '@nestjs/common';
import { ProfileModule } from './profile/profile.module';
import { SettingsModule } from './settings/settings.module';
import { SocialLinkModule } from './social-link/social-link.module';

@Module({
  imports: [ProfileModule, SettingsModule, SocialLinkModule],
})
export class AccountModule {}

```

## File: connectfy-api-gateway/src/modules/account/profile/profile.controller.ts
```typescript
import { AuthGuard } from '@/src/guards/auth.guard';
import {
  Body,
  Controller,
  Param,
  Patch,
  Post,
  UseGuards,
} from '@nestjs/common';
import { ProfileService } from './profile.service';

@UseGuards(AuthGuard)
@Controller('account/profile')
export class ProfileController {
  constructor(private readonly service: ProfileService) {}

  @Post('get')
  async getProfile() {
    return this.service.getProfile();
  }

  @Post('findOneByUserId/:_id')
  async findOneByUserId(@Param('_id') _id: string) {
    return this.service.findOneByUserId({ userId: _id });
  }

  @Patch('update')
  async update(@Body() data) {
    return this.service.update(data);
  }

  @Patch('update-avatar')
  async updateAvatar(@Body() data) {
    return this.service.updateAvatar(data);
  }

  @Patch('update-default-avatar')
  async updateDefaultAvatar(@Body() data) {
    return this.service.updateDefaultAvatar(data);
  }

  @Post('search')
  async search(@Body() data) {
    return this.service.search(data);
  }
}

```

## File: connectfy-api-gateway/src/modules/account/profile/profile.module.ts
```typescript
import { Module } from '@nestjs/common';
import { ProfileController } from './profile.controller';
import { ProfileService } from './profile.service';

@Module({
  imports: [],
  controllers: [ProfileController],
  providers: [ProfileService],
  exports: [],
})
export class ProfileModule {}

```

## File: connectfy-api-gateway/src/modules/account/profile/profile.service.ts
```typescript
import { CacheService } from '@/src/app-settings/cache/cache.service';
import { TcpConnectionService } from '@/src/app-settings/tcp-connections/tcp-connection.service';
import { Injectable } from '@nestjs/common';
import { CACHE_KEYS, CLS_KEYS } from 'connectfy-shared';
import { ClsService } from 'nestjs-cls';

@Injectable()
export class ProfileService {
  constructor(
    private readonly tcpConnectionService: TcpConnectionService,
    private readonly cacheService: CacheService,
    private readonly cls: ClsService,
  ) {}

  // ======================
  // Get profile
  // ======================
  async getProfile() {
    const user = await this.cls.get(CLS_KEYS.USER);
    const cacheKey = CACHE_KEYS.ACCOUNT.PROFILE(user._id);
    const cached = await this.cacheService.get<Record<string, any> | undefined>(
      cacheKey,
    );

    if (cached) return cached;

    const res = await this.tcpConnectionService.account({
      endpoint: 'profile/findOne',
      payload: {
        query: {
          userId: user._id,
        },
      },
    });

    if (res) {
      await this.cacheService.set(cacheKey, res);
    }

    return res;
  }

  // ======================
  // Find one by user id
  // ======================
  async findOneByUserId(data: any) {
    return this.tcpConnectionService.account({
      endpoint: 'profile/findOneByUserId',
      payload: data,
    });
  }

  // ======================
  // Search profiles
  // ======================
  async search(data: any) {
    return this.tcpConnectionService.account({
      endpoint: 'profile/search',
      payload: data,
    });
  }

  // ======================
  // Update profile
  // ======================
  async update(data: any) {
    const user = this.cls.get(CLS_KEYS.USER);
    const res = await this.tcpConnectionService.account({
      endpoint: 'profile/update',
      payload: data,
    });

    const cacheKey = CACHE_KEYS.ACCOUNT.PROFILE(user._id);
    await this.cacheService.remove(cacheKey);

    return res;
  }

  // ======================
  // Update avatar
  // ======================
  async updateAvatar(data: any) {
    const user = this.cls.get(CLS_KEYS.USER);
    const res = await this.tcpConnectionService.account({
      endpoint: 'profile/updateAvatar',
      payload: data,
    });

    const userCacheKey = CACHE_KEYS.AUTH.USER(user._id);
    const profileCacheKey = CACHE_KEYS.ACCOUNT.PROFILE(user._id);
    await this.cacheService.removeMany([userCacheKey, profileCacheKey]);

    return res;
  }

  // ======================
  // Update default avatar
  // ======================
  async updateDefaultAvatar(data: any) {
    const user = this.cls.get(CLS_KEYS.USER);
    const res = await this.tcpConnectionService.account({
      endpoint: 'profile/updateDefaultAvatar',
      payload: data,
    });

    const userCacheKey = CACHE_KEYS.AUTH.USER(user._id);
    const profileCacheKey = CACHE_KEYS.ACCOUNT.PROFILE(user._id);
    await this.cacheService.removeMany([userCacheKey, profileCacheKey]);

    return res;
  }
}

```

## File: connectfy-api-gateway/src/modules/account/settings/settings.module.ts
```typescript
import { Module } from '@nestjs/common';
import { GeneralSettingsModule } from './general/general-settings.module';
import { NotificationSettingsModule } from './notification/notification-settings.module';
import { PrivacySettingsModule } from './privacy/privacy-settings.module';

@Module({
  imports: [
    GeneralSettingsModule,
    NotificationSettingsModule,
    PrivacySettingsModule,
  ],
  controllers: [],
  providers: [],
  exports: [],
})
export class SettingsModule {}

```

## File: connectfy-api-gateway/src/modules/account/settings/notification/notification-settings.module.ts
```typescript
import { Module } from '@nestjs/common';
import { NotificationSettingsController } from './notification-settings.controller';
import { NotificationSettingsService } from './notification-settings.service';

@Module({
  imports: [],
  providers: [NotificationSettingsService],
  controllers: [NotificationSettingsController],
})
export class NotificationSettingsModule {}

```

## File: connectfy-api-gateway/src/modules/account/settings/notification/notification-settings.service.ts
```typescript
import { Injectable } from '@nestjs/common';
import { CACHE_KEYS, CLS_KEYS } from 'connectfy-shared';
import { CacheService } from '@/src/app-settings/cache/cache.service';
import { TcpConnectionService } from '@/src/app-settings/tcp-connections/tcp-connection.service';
import { ClsService } from 'nestjs-cls';

@Injectable()
export class NotificationSettingsService {
  constructor(
    private readonly tcpConnectionService: TcpConnectionService,
    private readonly cacheService: CacheService,
    private readonly cls: ClsService,
  ) {}

  async get() {
    const user = await this.cls.get(CLS_KEYS.USER);
    const cacheKey = CACHE_KEYS.ACCOUNT.SETTINGS.NOTIFICATION(user._id);
    const cached = await this.cacheService.get(cacheKey);

    if (cached) return cached;

    const res = await this.tcpConnectionService.account({
      endpoint: 'notification-settings/get',
    });

    if (res) {
      await this.cacheService.set(cacheKey, res);
    }

    return res;
  }

  async update(data: any) {
    const user = this.cls.get(CLS_KEYS.USER);
    const res = await this.tcpConnectionService.account({
      endpoint: 'notification-settings/update',
      payload: data,
    });

    const cacheKey = CACHE_KEYS.ACCOUNT.SETTINGS.NOTIFICATION(user._id);
    await this.cacheService.remove(cacheKey);

    return res;
  }
}

```

## File: connectfy-api-gateway/src/modules/account/settings/notification/notification-settings.controller.ts
```typescript
import { Body, Controller, Patch, Post, UseGuards } from '@nestjs/common';
import { AuthGuard } from '@/src/guards/auth.guard';
import { NotificationSettingsService } from './notification-settings.service';

@UseGuards(AuthGuard)
@Controller('account/settings/notification-settings')
export class NotificationSettingsController {
  constructor(
    private readonly notificationSettingsService: NotificationSettingsService,
  ) {}

  @Post('get')
  async get() {
    return this.notificationSettingsService.get();
  }

  @Patch('update')
  async update(@Body() data: any) {
    return this.notificationSettingsService.update(data);
  }
}

```

## File: connectfy-api-gateway/src/modules/account/settings/general/general-settings.controller.ts
```typescript
import { Body, Controller, Patch, Post, UseGuards } from '@nestjs/common';
import { AuthGuard } from '@/src/guards/auth.guard';
import { GeneralSettingsService } from './general-settings.service';

@UseGuards(AuthGuard)
@Controller('account/settings/general-settings')
export class GeneralSettingsController {
  constructor(
    private readonly generalSettingsService: GeneralSettingsService,
  ) {}

  @Post('get')
  async get() {
    return this.generalSettingsService.get();
  }

  @Patch('update')
  async update(@Body() data: any) {
    return this.generalSettingsService.update(data);
  }

  @Patch('reset')
  async reset() {
    return this.generalSettingsService.reset();
  }
}

```

## File: connectfy-api-gateway/src/modules/account/settings/general/general-settings.service.ts
```typescript
import { Injectable } from '@nestjs/common';
import { CACHE_KEYS, CLS_KEYS } from 'connectfy-shared';
import { CacheService } from '@/src/app-settings/cache/cache.service';
import { TcpConnectionService } from '@/src/app-settings/tcp-connections/tcp-connection.service';
import { ClsService } from 'nestjs-cls';

@Injectable()
export class GeneralSettingsService {
  constructor(
    private readonly tcpConnectionService: TcpConnectionService,
    private readonly cacheService: CacheService,
    private readonly cls: ClsService,
  ) {}

  async get() {
    const user = await this.cls.get(CLS_KEYS.USER);
    const cacheKey = CACHE_KEYS.ACCOUNT.SETTINGS.GENERAL(user._id);
    const cached = await this.cacheService.get<Record<string, any> | undefined>(
      cacheKey,
    );

    if (cached) return cached;

    const res = await this.tcpConnectionService.account({
      endpoint: 'general-settings/get',
    });

    if (res) {
      await this.cacheService.set(cacheKey, res);
    }

    return res;
  }

  async update(data: any) {
    const user = this.cls.get(CLS_KEYS.USER);
    const res = await this.tcpConnectionService.account({
      endpoint: 'general-settings/update',
      payload: data,
    });

    const userCacheKey = CACHE_KEYS.AUTH.USER(user._id);
    const settingsCacheKey = CACHE_KEYS.ACCOUNT.SETTINGS.GENERAL(user._id);
    await this.cacheService.removeMany([userCacheKey, settingsCacheKey]);

    return res;
  }

  async reset() {
    const user = this.cls.get(CLS_KEYS.USER);
    const res = await this.tcpConnectionService.account({
      endpoint: 'general-settings/reset',
    });

    const generalCacheKey = CACHE_KEYS.ACCOUNT.SETTINGS.GENERAL(user._id);
    const privacyCacheKey = CACHE_KEYS.ACCOUNT.SETTINGS.PRIVACY(user._id);
    const notificationCacheKey = CACHE_KEYS.ACCOUNT.SETTINGS.NOTIFICATION(
      user._id,
    );

    await this.cacheService.removeMany([
      generalCacheKey,
      privacyCacheKey,
      notificationCacheKey,
    ]);

    return res;
  }
}

```

## File: connectfy-api-gateway/src/modules/account/settings/general/general-settings.module.ts
```typescript
import { Module } from '@nestjs/common';
import { GeneralSettingsController } from './general-settings.controller';
import { GeneralSettingsService } from './general-settings.service';

@Module({
  imports: [],
  providers: [GeneralSettingsService],
  controllers: [GeneralSettingsController],
})
export class GeneralSettingsModule {}

```

## File: connectfy-api-gateway/src/modules/account/settings/privacy/privacy-settings.service.ts
```typescript
import { Injectable } from '@nestjs/common';
import { CACHE_KEYS, CLS_KEYS } from 'connectfy-shared';
import { CacheService } from '@/src/app-settings/cache/cache.service';
import { TcpConnectionService } from '@/src/app-settings/tcp-connections/tcp-connection.service';
import { ClsService } from 'nestjs-cls';

@Injectable()
export class PrivacySettingsService {
  constructor(
    private readonly tcpConnectionService: TcpConnectionService,
    private readonly cacheService: CacheService,
    private readonly cls: ClsService,
  ) {}

  async get() {
    const user = await this.cls.get(CLS_KEYS.USER);
    const cacheKey = CACHE_KEYS.ACCOUNT.SETTINGS.PRIVACY(user._id);
    const cached = await this.cacheService.get<Record<string, any> | undefined>(
      cacheKey,
    );

    if (cached) return cached;

    const res = await this.tcpConnectionService.account({
      endpoint: 'privacy-settings/get',
    });

    if (res) {
      await this.cacheService.set(cacheKey, res);
    }

    return res;
  }

  async update(data: any) {
    const user = this.cls.get(CLS_KEYS.USER);
    const res = await this.tcpConnectionService.account({
      endpoint: 'privacy-settings/update',
      payload: data,
    });

    const cacheKey = CACHE_KEYS.ACCOUNT.SETTINGS.PRIVACY(user._id);
    await this.cacheService.remove(cacheKey);

    return res;
  }
}

```

## File: connectfy-api-gateway/src/modules/account/settings/privacy/privacy-settings.module.ts
```typescript
import { Module } from '@nestjs/common';
import { PrivacySettingsController } from './privacy-settings.controller';
import { PrivacySettingsService } from './privacy-settings.service';

@Module({
  imports: [],
  providers: [PrivacySettingsService],
  controllers: [PrivacySettingsController],
})
export class PrivacySettingsModule {}

```

## File: connectfy-api-gateway/src/modules/account/settings/privacy/privacy-settings.controller.ts
```typescript
import { Body, Controller, Patch, Post, UseGuards } from '@nestjs/common';
import { AuthGuard } from '@/src/guards/auth.guard';
import { PrivacySettingsService } from './privacy-settings.service';

@UseGuards(AuthGuard)
@Controller('account/settings/privacy-settings')
export class PrivacySettingsController {
  constructor(
    private readonly privacySettingsService: PrivacySettingsService,
  ) {}

  @Post('get')
  async get() {
    return this.privacySettingsService.get();
  }

  @Patch('update')
  async update(@Body() data: any) {
    return this.privacySettingsService.update(data);
  }
}

```

## File: connectfy-api-gateway/src/modules/account/social-link/social-link.service.ts
```typescript
import { CacheService } from '@/src/app-settings/cache/cache.service';
import { TcpConnectionService } from '@/src/app-settings/tcp-connections/tcp-connection.service';
import { Injectable } from '@nestjs/common';
import { CACHE_KEYS, CLS_KEYS, IReturnedUser } from 'connectfy-shared';
import { ClsService } from 'nestjs-cls';

@Injectable()
export class SocialLinkService {
  constructor(
    private readonly tcpConnectionService: TcpConnectionService,
    private readonly cls: ClsService,
    private readonly cacheService: CacheService,
  ) {}

  async findMany(data: any) {
    const { _id: currentUserId } = this.cls.get<IReturnedUser>(CLS_KEYS.USER);
    const cacheKey = CACHE_KEYS.ACCOUNT.SOCIAL_LINK(currentUserId);
    const userId = data.userId;

    if (currentUserId === userId) {
      const cached = await this.cacheService.get(cacheKey);

      if (cached) {
        return cached;
      }
    }

    const res = await this.tcpConnectionService.account({
      endpoint: 'socialLinks/findMany',
      payload: data,
    });

    if (currentUserId === userId) {
      await this.cacheService.set(cacheKey, res);
    }

    return res;
  }

  async create(data: any) {
    const { _id } = this.cls.get<IReturnedUser>(CLS_KEYS.USER);

    const res = await this.tcpConnectionService.account({
      endpoint: 'socialLinks/create',
      payload: data,
    });

    const cacheKey = CACHE_KEYS.ACCOUNT.SOCIAL_LINK(_id);
    await this.cacheService.remove(cacheKey);

    return res;
  }

  async update(data: any) {
    const { _id } = this.cls.get<IReturnedUser>(CLS_KEYS.USER);

    const res = await this.tcpConnectionService.account({
      endpoint: 'socialLinks/update',
      payload: data,
    });

    const cacheKey = CACHE_KEYS.ACCOUNT.SOCIAL_LINK(_id);
    await this.cacheService.remove(cacheKey);

    return res;
  }

  async updateRank(data: any) {
    const { _id } = this.cls.get<IReturnedUser>(CLS_KEYS.USER);

    const res = await this.tcpConnectionService.account({
      endpoint: 'socialLinks/updateRank',
      payload: data,
    });

    const cacheKey = CACHE_KEYS.ACCOUNT.SOCIAL_LINK(_id);
    await this.cacheService.remove(cacheKey);

    return res;
  }

  async remove(data: any) {
    const { _id } = this.cls.get<IReturnedUser>(CLS_KEYS.USER);

    const res = await this.tcpConnectionService.account({
      endpoint: 'socialLinks/remove',
      payload: data,
    });

    const cacheKey = CACHE_KEYS.ACCOUNT.SOCIAL_LINK(_id);
    await this.cacheService.remove(cacheKey);

    return res;
  }

  async removeMany(data: any) {
    const { _id } = this.cls.get<IReturnedUser>(CLS_KEYS.USER);

    const res = await this.tcpConnectionService.account({
      endpoint: 'socialLinks/removeMany',
      payload: data,
    });

    const cacheKey = CACHE_KEYS.ACCOUNT.SOCIAL_LINK(_id);
    await this.cacheService.remove(cacheKey);

    return res;
  }
}

```

## File: connectfy-api-gateway/src/modules/account/social-link/social-link.module.ts
```typescript
import { Module } from '@nestjs/common';
import { SocialLinkService } from './social-link.service';
import { SocialLinkController } from './social-link.controller';

@Module({
  imports: [],
  providers: [SocialLinkService],
  controllers: [SocialLinkController],
  exports: [],
})
export class SocialLinkModule {}

```

## File: connectfy-api-gateway/src/modules/account/social-link/social-link.controller.ts
```typescript
import {
  Body,
  Controller,
  Delete,
  Patch,
  Post,
  UseGuards,
} from '@nestjs/common';
import { SocialLinkService } from './social-link.service';
import { AuthGuard } from '@/src/guards/auth.guard';

@UseGuards(AuthGuard)
@Controller('account/social-link')
export class SocialLinkController {
  constructor(private readonly service: SocialLinkService) {}

  @Post('get')
  getSocialLinks(@Body() data: any) {
    return this.service.findMany(data);
  }

  @Post('create')
  create(@Body() data: any) {
    return this.service.create(data);
  }

  @Patch('update')
  update(@Body() data: any) {
    return this.service.update(data);
  }

  @Patch('update-rank')
  updateRank(@Body() data: any) {
    return this.service.updateRank(data);
  }

  @Delete('remove')
  remove(@Body() data: any) {
    return this.service.remove(data);
  }

  @Delete('removeMany')
  removeMany(@Body() data: any) {
    return this.service.removeMany(data);
  }
}

```

## File: connectfy-api-gateway/src/modules/relationship/relationship.module.ts
```typescript
import { Module } from '@nestjs/common';
import { FriendshipModule } from './friendship/friendship.module';
import { BlocklistModule } from './blocklist/blocklist.module';

@Module({
  imports: [FriendshipModule, BlocklistModule],
  controllers: [],
  providers: [],
  exports: [],
})
export class RelationshipModule {}

```

## File: connectfy-api-gateway/src/modules/relationship/blocklist/blocklist.controller.ts
```typescript
import { Body, Controller, Post, UseGuards } from '@nestjs/common';
import { BlocklistService } from './blocklist.service';
import { AuthGuard } from '@/src/guards/auth.guard';

@UseGuards(AuthGuard)
@Controller('relationship/blocklist')
export class BlocklistController {
  constructor(private readonly service: BlocklistService) {}

  @Post('find-blocked')
  findBlockedUsers(@Body() data: any) {
    return this.service.findBlockedUsers(data);
  }

  @Post('unblock')
  unblock(@Body() data: any) {
    return this.service.unblock(data);
  }

  @Post('block')
  block(@Body() data: any) {
    return this.service.block(data);
  }
}

```

## File: connectfy-api-gateway/src/modules/relationship/blocklist/blocklist.module.ts
```typescript
import { Module } from '@nestjs/common';
import { BlocklistController } from './blocklist.controller';
import { BlocklistService } from './blocklist.service';

@Module({
  controllers: [BlocklistController],
  providers: [BlocklistService],
})
export class BlocklistModule {}

```

## File: connectfy-api-gateway/src/modules/relationship/blocklist/blocklist.service.ts
```typescript
import { TcpConnectionService } from '@/src/app-settings/tcp-connections/tcp-connection.service';
import { Injectable } from '@nestjs/common';

@Injectable()
export class BlocklistService {
  constructor(private readonly tcpConnectionService: TcpConnectionService) {}

  async findBlockedUsers(data: any) {
    return this.tcpConnectionService.relationship({
      endpoint: 'blocklist/findMany',
      payload: data,
    });
  }

  async unblock(data: any) {
    return this.tcpConnectionService.relationship({
      endpoint: 'blocklist/remove',
      payload: data,
    });
  }

  async block(data: any) {
    return this.tcpConnectionService.relationship({
      endpoint: 'blocklist/create',
      payload: data,
    });
  }
}

```

## File: connectfy-api-gateway/src/modules/relationship/friendship/friendship.service.ts
```typescript
import { TcpConnectionService } from '@/src/app-settings/tcp-connections/tcp-connection.service';
import { Injectable } from '@nestjs/common';
import { PresenceService } from '../../presence/presence.service';

@Injectable()
export class FriendshipService {
  constructor(
    private readonly tcpConnectionService: TcpConnectionService,
    private readonly presenceService: PresenceService,
  ) {}

  async findRequests(data: any) {
    return this.tcpConnectionService.relationship({
      endpoint: 'friendship/findRequests',
      payload: data,
    });
  }

  async findFriends(data: any) {
    return this.tcpConnectionService.relationship({
      endpoint: 'friendship/findFriends',
      payload: data,
    });
  }

  async sendFriendRequest(data: any) {
    return this.tcpConnectionService.relationship({
      endpoint: 'friendship/create',
      payload: data,
    });
  }

  async acceptFriendRequest(data: any) {
    return this.tcpConnectionService.relationship({
      endpoint: 'friendship/acceptFriendshipRequest',
      payload: data,
    });
  }

  async declineFriendRequest(data: any) {
    return this.tcpConnectionService.relationship({
      endpoint: 'friendship/declineFriendshipRequest',
      payload: data,
    });
  }

  async cancelFriendRequest(data: any) {
    return this.tcpConnectionService.relationship({
      endpoint: 'friendship/cancelFriendshipRequest',
      payload: data,
    });
  }

  async unfriend(data: any) {
    return this.tcpConnectionService.relationship({
      endpoint: 'friendship/unfriend',
      payload: data,
    });
  }

  async updateCloseFriend(data: any) {
    return this.tcpConnectionService.relationship({
      endpoint: 'friendship/updateCloseFriend',
      payload: data,
    });
  }

  async updateNotification(data: any) {
    return this.tcpConnectionService.relationship({
      endpoint: 'friendship/updateNotification',
      payload: data,
    });
  }

  async requestsCount() {
    return this.tcpConnectionService.relationship({
      endpoint: 'friendship/requestsCount',
    });
  }

  async findOnlineFriends(data: any) {
    const { userId, skip, limit } = data;
    
    // Proxy to get all friends
    const response = await this.tcpConnectionService.relationship({
      endpoint: 'friendship/findFriendIds',
      payload: { userId },
    });
    
    const friends = response?.friends || [];
    if (friends.length === 0) {
      return { data: [], onlineCount: 0 };
    }

    const friendIds = friends.map((f: any) => f._id);
    const onlineStatuses = await this.presenceService.areUsersOnline(friendIds);
    
    // Build a Set of online IDs for safe lookup (avoids positional mismatch risk)
    const onlineIdSet = new Set<string>();
    friendIds.forEach((id: string, idx: number) => {
      if (onlineStatuses[idx]) onlineIdSet.add(id);
    });
    
    const onlineFriends = friends.filter((f: any) => onlineIdSet.has(f._id));
    
    const onlineCount = onlineFriends.length;
    const offset = (skip - 1) * limit;
    const paginated = onlineFriends.slice(offset, offset + limit);
    
    return { data: paginated, onlineCount };
  }

  async findSuggestions(data: any) {
    return this.tcpConnectionService.relationship({
      endpoint: 'friendship/findSuggestions',
      payload: data,
    });
  }

  async findMutualFriends(data: any) {
    return this.tcpConnectionService.relationship({
      endpoint: 'friendship/findMutualFriends',
      payload: data,
    });
  }
}

```

## File: connectfy-api-gateway/src/modules/relationship/friendship/friendship.module.ts
```typescript
import { Module } from '@nestjs/common';
import { FriendshipController } from './friendship.controller';
import { FriendshipService } from './friendship.service';

@Module({
  imports: [],
  controllers: [FriendshipController],
  providers: [FriendshipService],
})
export class FriendshipModule {}

```

## File: connectfy-api-gateway/src/modules/relationship/friendship/friendship.controller.ts
```typescript
import { Body, Controller, Post, UseGuards } from '@nestjs/common';
import { FriendshipService } from './friendship.service';
import { AuthGuard } from '@/src/guards/auth.guard';

@UseGuards(AuthGuard)
@Controller('relationship/friendship')
export class FriendshipController {
  constructor(private readonly service: FriendshipService) {}

  @Post('find-requests')
  findRequests(@Body() data: any) {
    return this.service.findRequests(data);
  }

  @Post('find-friends')
  findFriends(@Body() data: any) {
    return this.service.findFriends(data);
  }

  @Post('send-request')
  sendFriendRequest(@Body() data: any) {
    return this.service.sendFriendRequest(data);
  }

  @Post('accept-request')
  acceptFriendRequest(@Body() data: any) {
    return this.service.acceptFriendRequest(data);
  }

  @Post('decline-request')
  declineFriendRequest(@Body() data: any) {
    return this.service.declineFriendRequest(data);
  }

  @Post('cancel-request')
  cancelFriendRequest(@Body() data: any) {
    return this.service.cancelFriendRequest(data);
  }

  @Post('unfriend')
  unfriend(@Body() data: any) {
    return this.service.unfriend(data);
  }

  @Post('update-close-friend')
  updateCloseFriend(@Body() data: any) {
    return this.service.updateCloseFriend(data);
  }

  @Post('update-notification')
  updateNotification(@Body() data: any) {
    return this.service.updateNotification(data);
  }

  @Post('requests-count')
  requestsCount() {
    return this.service.requestsCount();
  }

  @Post('find-online-friends')
  findOnlineFriends(@Body() data: any) {
    return this.service.findOnlineFriends(data);
  }

  @Post('find-suggestions')
  findSuggestions(@Body() data: any) {
    return this.service.findSuggestions(data);
  }

  @Post('find-mutual-friends')
  findMutualFriends(@Body() data: any) {
    return this.service.findMutualFriends(data);
  }
}

```

## File: connectfy-api-gateway/src/modules/notification-action-history/notification-action-history.module.ts
```typescript
import { Module } from '@nestjs/common';
import { NotificationModule } from './notification/notification.module';

@Module({
  imports: [NotificationModule],
})
export class NotificationActionHistoryModule {}

```

## File: connectfy-api-gateway/src/modules/notification-action-history/notification/notification.gateway.ts
```typescript
import { CacheService } from '@/src/app-settings/cache/cache.service';
import {
  OnGatewayConnection,
  OnGatewayInit,
  WebSocketGateway,
  WebSocketServer,
} from '@nestjs/websockets';
import {
  BaseException,
  CACHE_KEYS,
  ExceptionMessages,
  HttpStatus,
} from 'connectfy-shared';
import { Server, Socket } from 'socket.io';

@WebSocketGateway({ namespace: '/notification', cors: { origin: '*' } })
export class NotificationGateway implements OnGatewayInit, OnGatewayConnection {
  @WebSocketServer() server: Server;

  constructor(private readonly cacheService: CacheService) {}

  afterInit(server: Server) {
    server.use(async (socket, next) => {
      try {
        const accessToken = this.extractAccessToken(socket);

        if (!accessToken) {
          return next(new Error('Unauthorized: no token'));
        }

        const accessTokenKey = CACHE_KEYS.AUTH.ACCESS_TOKEN(accessToken);
        const cachedAuth =
          await this.cacheService.get<Record<string, any>>(accessTokenKey);
        const userId = this.resolveUserId(cachedAuth);

        if (!userId) {
          return next(
            new BaseException(
              ExceptionMessages.UNAUTHORIZED_MESSAGE,
              HttpStatus.UNAUTHORIZED,
            ),
          );
        }

        socket.data.userId = userId;
        next();
      } catch (error) {
        next(
          new BaseException(
            ExceptionMessages.UNAUTHORIZED_MESSAGE,
            HttpStatus.UNAUTHORIZED,
          ),
        );
      }
    });
  }

  async handleConnection(client: Socket) {
    const userId = this.resolveUserId(client.data.userId);

    if (!userId) {
      client.disconnect(true);
      return;
    }

    const roomName = this.getRoomName(userId);
    client.join(roomName);
    console.log(`[NotificationGateway] User ${userId} joined room ${roomName}`);
  }

  async handleDisconnect(client: Socket) {
    const userId = this.resolveUserId(client.data.userId);
    if (!userId) return;

    const roomName = this.getRoomName(userId);
    client.leave(roomName);
  }

  sendNotificationToUser(userId: string | number, payload: any) {
    const resolvedUserId = this.resolveUserId(userId);
    if (!resolvedUserId) {
      return;
    }

    const roomName = this.getRoomName(resolvedUserId);
    this.server.to(roomName).emit('notification:new', payload);
  }

  private extractAccessToken(socket: Socket): string | undefined {
    const token =
      socket.handshake.auth?.token ?? socket.handshake.auth?.access_token;
    if (!token) return undefined;

    return token.startsWith('Bearer ') ? token.slice(7) : token;
  }

  private resolveUserId(value: unknown): string | undefined {
    if (typeof value === 'string' || typeof value === 'number') {
      const normalized = String(value).trim();
      return normalized.length ? normalized : undefined;
    }

    if (value && typeof value === 'object') {
      const candidate =
        (value as Record<string, unknown>)._id ??
        (value as Record<string, unknown>).id ??
        (value as Record<string, unknown>).userId;

      if (typeof candidate === 'string' || typeof candidate === 'number') {
        const normalized = String(candidate).trim();
        return normalized.length ? normalized : undefined;
      }
    }

    return undefined;
  }

  private getRoomName(userId: string | number) {
    return `user:${String(userId).trim()}`;
  }
}

```

## File: connectfy-api-gateway/src/modules/notification-action-history/notification/notification-event.controller.ts
```typescript
import { Controller } from '@nestjs/common';
import {
  Ctx,
  EventPattern,
  KafkaContext,
  Payload,
  Transport,
} from '@nestjs/microservices';
import { NotificationGateway } from './notification.gateway';
import { commitKafkaOffset } from 'connectfy-shared';

@Controller()
export class NotificationEventController {
  constructor(private readonly notificationGateway: NotificationGateway) {}

  @EventPattern('notification.created', Transport.KAFKA)
  async onNotificationCreated(
    @Payload() notifications: any,
    @Ctx() context: KafkaContext,
  ) {
    this.notificationGateway.sendNotificationToUser(notifications.recipientId, {
      ...notifications,
    });
    await commitKafkaOffset(context);
  }
}

```

## File: connectfy-api-gateway/src/modules/notification-action-history/notification/notification.service.ts
```typescript
import { Injectable } from '@nestjs/common';
import { TcpConnectionService } from '@/src/app-settings/tcp-connections/tcp-connection.service';

@Injectable()
export class NotificationService {
  constructor(private readonly tcpConnectionService: TcpConnectionService) {}

  async getNotifications(data: any) {
    return await this.tcpConnectionService.notificationActionHistory({
      endpoint: 'notification/all',
      payload: data,
    });
  }

  async countUnreadNotifications(data: any) {
    return await this.tcpConnectionService.notificationActionHistory({
      endpoint: 'notification/countUnread',
      payload: data,
    });
  }

  async markAsRead(data: any) {
    return await this.tcpConnectionService.notificationActionHistory({
      endpoint: 'notification/markRead',
      payload: data,
    });
  }

  async markAsUnread(data: any) {
    return await this.tcpConnectionService.notificationActionHistory({
      endpoint: 'notification/markUnread',
      payload: data,
    });
  }

  async markAllAsRead(data: any) {
    console.log('data._ids: ', data._ids);
    await this.tcpConnectionService.notificationActionHistory({
      endpoint: 'notification/markAllRead',
      payload: data,
    });
  }

  async removeNotification(data: any) {
    return await this.tcpConnectionService.notificationActionHistory({
      endpoint: 'notification/remove',
      payload: data,
    });
  }

  async removeAllNotification(data: any) {
    return await this.tcpConnectionService.notificationActionHistory({
      endpoint: 'notification/removeAll',
      payload: data,
    });
  }
}

```

## File: connectfy-api-gateway/src/modules/notification-action-history/notification/notification.module.ts
```typescript
import { Module } from '@nestjs/common';
import { NotificationGateway } from './notification.gateway';
import { NotificationService } from './notification.service';
import { NotificationController } from './notification.controller';
import { NotificationEventController } from './notification-event.controller';

@Module({
  imports: [],
  providers: [NotificationGateway, NotificationService],
  exports: [NotificationGateway, NotificationService],
  controllers: [NotificationEventController, NotificationController],
})
export class NotificationModule {}

```

## File: connectfy-api-gateway/src/modules/notification-action-history/notification/notification.controller.ts
```typescript
import { Body, Controller, Post, UseGuards } from '@nestjs/common';
import { NotificationService } from './notification.service';
import { AuthGuard } from '@/src/guards/auth.guard';

@UseGuards(AuthGuard)
@Controller('/notification-action-history/notification')
export class NotificationController {
  constructor(private readonly service: NotificationService) {}

  @Post('all')
  async getNotifications(@Body() data) {
    return this.service.getNotifications(data);
  }

  @Post('countUnread')
  async countUnreadNotifications(@Body() data) {
    return this.service.countUnreadNotifications(data);
  }

  @Post('markRead')
  async markAsRead(@Body() data) {
    return this.service.markAsRead(data);
  }

  @Post('markUnread')
  async markAsUnread(@Body() data) {
    return this.service.markAsUnread(data);
  }

  @Post('markAllRead')
  async markAllAsRead(@Body() data) {
    await this.service.markAllAsRead(data);
  }

  @Post('remove')
  async removeNotification(@Body() data) {
    return this.service.removeNotification(data);
  }

  @Post('removeAll')
  async removeAllNotification(@Body() data) {
    return this.service.removeAllNotification(data);
  }
}

```

## File: connectfy-api-gateway/src/gateways/socker.gateway.ts
```typescript
import { Inject } from '@nestjs/common';
import {
  ConnectedSocket,
  OnGatewayConnection,
  OnGatewayDisconnect,
  SubscribeMessage,
  WebSocketGateway,
  WebSocketServer,
} from '@nestjs/websockets';
import { Server, Socket } from 'socket.io';
import { CACHE_MANAGER } from '@nestjs/cache-manager';
import { Cache } from 'cache-manager';
import { CACHE_KEYS } from 'connectfy-shared';

@WebSocketGateway({ cors: { origin: '*' } })
export class MySocketGateway
  implements OnGatewayConnection, OnGatewayDisconnect
{
  constructor(@Inject(CACHE_MANAGER) private cacheService: Cache) {}

  private userSocketsMap: Map<string, Socket[]> = new Map();

  @WebSocketServer()
  server: Server;

  socket: Socket;

  @SubscribeMessage('chat')
  handleMessage(dto: any, @ConnectedSocket() client: Socket): string {
    client.emit('chat', 'salam');
    // this.server.to(this.socket.id).emit("chat","salam bro necesen")
    return 'Hello world!';
  }

  // it will be handled when a client connects to the server
  async handleConnection(socket: Socket) {
    const user = await this.getUserId(socket);
    if (user) {
      const userSockets = this.userSocketsMap.get(user.id) || [];
      this.userSocketsMap.set(user.id, [...userSockets, socket]);
    } else {
      console.log('Authorization header is missing');
    }
  }

  async handleDisconnect(socket: Socket) {
    const user = await this.getUserId(socket);
    if (user) {
      const userSockets = this.userSocketsMap.get(user.id) || [];
      const updatedSockets = userSockets.filter((s) => s.id !== socket.id);
      this.userSocketsMap.set(user.id, updatedSockets);
    }
  }

  getCurrentSocket(userId: string) {
    return this.userSocketsMap.get(userId);
  }

  getUserSockets(userId: string): Socket[] {
    return this.userSocketsMap.get(userId) || [];
  }

  async emitMessage(payload: any, userId: string) {
    const userSockets = this.getUserSockets(userId);

    for (const socket of userSockets) {
      socket.emit('chat', payload);
    }
  }

  private async getUserId(socket: Socket) {
    const authorizationHeader = socket.handshake.auth.token;
    if (authorizationHeader) {
      const accessToken = authorizationHeader.split(' ')[1];
      const cacheKey = CACHE_KEYS.AUTH.ACCESS_TOKEN(accessToken);
      let user = (await this.cacheService.get(cacheKey)) as { id: string };
      return user;
    }
    return null;
  }
}

```

## File: connectfy-api-gateway/src/common/exception-filters/all.filter.ts
```typescript
import {
  ArgumentsHost,
  Catch,
  ExceptionFilter,
  HttpException,
  HttpStatus,
  Logger,
} from '@nestjs/common';
import { RpcException } from '@nestjs/microservices';
import { ExceptionMessages, LANGUAGE } from 'connectfy-shared';

@Catch()
export class AllExceptionsFilter implements ExceptionFilter {
  private readonly logger = new Logger(AllExceptionsFilter.name);

  catch(exception: any, host: ArgumentsHost) {
    const ctx = host.switchToHttp();
    const request = ctx.getRequest();
    const response = ctx.getResponse();
    const language = request.body?._lang ?? LANGUAGE.EN;

    let status = HttpStatus.INTERNAL_SERVER_ERROR;
    let message = ExceptionMessages.INTERNAL_SERVER_ERROR_MESSAGE(language);
    let additional = null;

    if (response.headersSent) {
      return;
    }

    // Log for debugging
    console.log('\n');
    if (exception.stack) this.logger.error('Exception stack', exception.stack);
    this.logger.error('Exception caught', JSON.stringify(exception, null, 2));

    // 1. If it's a proper NestJS HttpException
    if (exception instanceof HttpException) {
      status = exception.getStatus();
      const responseBody = exception.getResponse();

      if (typeof responseBody === 'object' && responseBody !== null) {
        message = (responseBody as any).message || message;
        additional = (responseBody as any).additional || null;
      } else message = responseBody as string;
    }

    // 2. If it's an RpcException (e.g. BaseException thrown from guards)
    else if (exception instanceof RpcException) {
      const rpcError = exception.getError();
      if (typeof rpcError === 'object' && rpcError !== null) {
        status = (rpcError as any).statusCode || status;
        message = (rpcError as any).message || message;
        additional = (rpcError as any).additional || null;
      } else {
        message = rpcError as string;
      }
    }

    // 3. If it's an object from CRM service via RPC (Kafka/TCP)
    else if (typeof exception === 'object' && exception !== null) {
      status = exception?.statusCode || status;
      message = exception?.response?.message || exception?.message || message;
      additional = exception?.additional || null;
    }

    // 3. Fallback for non-object or unexpected errors
    else message = typeof exception === 'string' ? exception : message;

    const responsePayload = {
      status: 'error',
      message,
      timestamp: new Date().toISOString(),
      path: request?.url,
    };

    if (additional) responsePayload['additional'] = additional;

    response.status(status).json(responsePayload);
  }
}

```

## File: connectfy-api-gateway/src/common/constants/environment-variables.ts
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
  JWT_ACCESS_SECRET: process.env.JWT_ACCESS_SECRET,
  JWT_REFRESH_SECRET: process.env.JWT_REFRESH_SECRET,

  // Session and CORS
  CLIENT_URL: process.env.CLIENT_URL,
  SESSION_SECRET_KEY: process.env.SESSION_SECRET_KEY,

  // Kafka
  SERVICE_NAME: process.env.SERVICE_NAME,
  BROKER1: process.env.BROKER1 || '',
  BROKER2: process.env.BROKER2 || '',

  // Redis
  REDIS_HOST: process.env.REDIS_HOST,
  REDIS_PORT: Number(process.env.REDIS_PORT),

  // TCP
  AUTH_SERVICE_HOST: process.env.AUTH_SERVICE_HOST,
  AUTH_SERVICE_PORT: Number(process.env.AUTH_SERVICE_PORT),

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

  // HTTP
  FILE_UPLOADER_BASE_URL: process.env.FILE_UPLOADER_BASE_URL,
};

```

## File: connectfy-api-gateway/src/common/functions/request.ts
```typescript
import { Request } from 'express';

export function extractRequestData(request: Request) {
  return {
    headers: {
      'user-agent': request.headers['user-agent'],
      'x-forwarded-for': request.headers['x-forwarded-for'],
      'x-real-ip': request.headers['x-real-ip'],
      'cf-connecting-ip': request.headers['cf-connecting-ip'],
    },
    ip: request.socket.remoteAddress,
  };
}

```

## File: connectfy-api-gateway/src/app-settings/app-settings.module.ts
```typescript
import { Global, Module } from '@nestjs/common';
import { TcpConnectionModule } from './tcp-connections/tcp-connection.module';
import { RedisModule } from './redis/redis.module';
import { CacheModule } from './cache/cache.module';
import { HttpConnectionModule } from './http-connection/http-connection.module';

@Global()
@Module({
  imports: [
    TcpConnectionModule,
    RedisModule,
    CacheModule,
    HttpConnectionModule,
  ],
  controllers: [],
  providers: [],
  exports: [],
})
export class AppSettingsModule {}

```

## File: connectfy-api-gateway/src/app-settings/cache/cache.service.ts
```typescript
import { Inject, Injectable } from '@nestjs/common';
import Redis from 'ioredis';
import { CACHE_KEYS, EXPIRE_DATES, REDIS_KEYS } from 'connectfy-shared';

type SetOpts = { key: string; data: any; ttl?: number };

@Injectable()
export class CacheService {
  // process-local map to dedupe concurrent loads for same key
  private readonly inflightRequests = new Map<string, Promise<any>>();

  constructor(
    @Inject(REDIS_KEYS.REDIS_CLIENT)
    private readonly redis: Redis,
  ) {}

  async set(keyOrOpts: string | SetOpts, data?: any, ttl?: number) {
    let key: string;
    let value: any;
    let _ttl: number | undefined;

    if (typeof keyOrOpts === 'string') {
      key = keyOrOpts;
      value = data;
      _ttl = ttl;
    } else {
      key = keyOrOpts.key;
      value = keyOrOpts.data;
      _ttl = keyOrOpts.ttl;
    }

    const payload = JSON.stringify(value);

    if (_ttl && _ttl > 0) {
      await this.redis.set(key, payload, 'EX', _ttl);
    } else {
      await this.redis.set(
        key,
        payload,
        'EX',
        EXPIRE_DATES.TTL.ONE_MINUTE * 15,
      );
    }
  }

  async get<T = any>(key: string): Promise<T | undefined> {
    const data = await this.redis.get(key);
    if (!data) return undefined;

    try {
      // IF data is JSON string, parse it
      return JSON.parse(data) as T;
    } catch {
      // IF data is not JSON string, return it as is
      return data as unknown as T;
    }
  }

  async remove(key: string) {
    await this.redis.del(key);
  }

  async removeMany(keys: string[]) {
    if (!keys || keys.length === 0) return 0;

    return await this.redis.del(...keys);
  }

  // getOrSet: if key exists return it, otherwise run loader(), set result and return it.
  // uses inflightRequests to dedupe concurrent loads.
  async getOrSet<T = any>(
    key: string,
    loader: () => Promise<T>,
    ttlSeconds?: number,
  ): Promise<T | undefined> {
    // try fast GET first
    const cached = await this.get<T>(key);
    if (cached !== undefined) return cached;

    // if another request is already loading this key, await it
    if (this.inflightRequests.has(key)) {
      return await this.inflightRequests.get(key);
    }

    // create loader promise and store in map
    const p = (async () => {
      try {
        const result = await loader();
        if (result !== undefined && result !== null) {
          await this.set({ key, data: result, ttl: ttlSeconds });
        }
        return result;
      } finally {
        // ensure map cleanup
        this.inflightRequests.delete(key);
      }
    })();

    this.inflightRequests.set(key, p);
    return p;
  }

  // production-friendly clearByPrefix (uses scanStream)
  async clearByPrefix(prefix: string) {
    const stream = this.redis.scanStream({ match: `${prefix}*`, count: 500 });
    const pipeline = this.redis.pipeline();
    let any = false;

    return new Promise<void>((resolve, reject) => {
      stream.on('data', (keys: string[]) => {
        if (keys.length) {
          any = true;
          keys.forEach((k) => pipeline.del(k));
        }
      });

      stream.on('end', async () => {
        if (any) {
          try {
            await pipeline.exec();
            resolve();
          } catch (err) {
            reject(err);
          }
        } else {
          resolve();
        }
      });

      stream.on('error', (err) => reject(err));
    });
  }

  async updatePreserveTtl(key: string, data: any) {
    const ttl = await this.redis.ttl(key);

    if (ttl > 0) {
      await this.set({ key, data, ttl });
    } else {
      await this.set({ key, data });
    }
  }

  async clearUserCache(userId: string, access_token: string) {
    const userCacheKey = CACHE_KEYS.AUTH.USER(userId);
    const profileCacheKey = CACHE_KEYS.ACCOUNT.PROFILE(userId);
    const accessTokenCacheKey = CACHE_KEYS.AUTH.ACCESS_TOKEN(access_token);
    const generalSettingsCacheKey = CACHE_KEYS.ACCOUNT.SETTINGS.GENERAL(userId);
    const privacySettingsCacheKey = CACHE_KEYS.ACCOUNT.SETTINGS.PRIVACY(userId);
    const notificationSettingsCacheKey =
      CACHE_KEYS.ACCOUNT.SETTINGS.NOTIFICATION(userId);

    await this.removeMany([
      userCacheKey,
      profileCacheKey,
      accessTokenCacheKey,
      generalSettingsCacheKey,
      privacySettingsCacheKey,
      notificationSettingsCacheKey,
    ]);
  }
}

```

## File: connectfy-api-gateway/src/app-settings/cache/cache.module.ts
```typescript
import { Module, Global } from '@nestjs/common';
import { CacheService } from './cache.service';
import { RedisModule } from '../redis/redis.module';

@Global()
@Module({
  imports: [RedisModule],
  providers: [CacheService],
  exports: [CacheService],
})
export class CacheModule {}

```

## File: connectfy-api-gateway/src/app-settings/http-connection/http-connection.service.ts
```typescript
import { HttpService } from '@nestjs/axios';
import { Injectable } from '@nestjs/common';
import { AxiosRequestConfig } from 'axios';
import { ClsService } from 'nestjs-cls';
import { firstValueFrom } from 'rxjs';

@Injectable()
export class HttpConnectionService {
  constructor(
    private readonly httpService: HttpService,
    private readonly cls: ClsService,
  ) {}

  async post<T>({
    baseUrl,
    endpoint,
    payload = {},
    config = {},
  }: {
    baseUrl: string;
    endpoint: string;
    payload?: Record<string, any>;
    config?: AxiosRequestConfig;
  }) {
    const url = this.joinBaseUrlAndEndpoint(baseUrl, endpoint);
    const user = this.cls.get('user');
    const lang = this.cls.get('lang');

    const data = {
      ...payload,
      _loggedUser: user,
      _lang: lang,
    };

    const finalConfig: AxiosRequestConfig = {
      ...config,
      headers: {
        ...(config.headers ?? {}),
      },
    };

    const response = await firstValueFrom(
      this.httpService.post<T>(url, data, finalConfig),
    );

    return response.data;
  }

  private joinBaseUrlAndEndpoint(baseUrl: string, endpoint: string): string {
    const b = baseUrl.replace(/\/+$/, '');
    const e = endpoint.replace(/^\/+/, '');
    return `${b}/${e}`;
  }
}

```

## File: connectfy-api-gateway/src/app-settings/http-connection/http-connection.module.ts
```typescript
import { Global, Module } from '@nestjs/common';
import { HttpConnectionService } from './http-connection.service';
import { HttpModule } from '@nestjs/axios';

@Global()
@Module({
  imports: [HttpModule],
  providers: [HttpConnectionService],
  exports: [HttpConnectionService],
})
export class HttpConnectionModule {}

```

## File: connectfy-api-gateway/src/app-settings/kafka-connections/loggin-kafka.server.ts
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

## File: connectfy-api-gateway/src/app-settings/redis/redis.module.ts
```typescript
import { Global, Module } from '@nestjs/common';
import Redis from 'ioredis';
import { ENVIRONMENT_VARIABLES } from '@/src/common/constants/environment-variables';
import { REDIS_KEYS } from 'connectfy-shared';

@Global()
@Module({
  providers: [
    {
      provide: REDIS_KEYS.REDIS_CLIENT,
      useFactory: () => {
        const redis = new Redis({
          host: ENVIRONMENT_VARIABLES.REDIS_HOST,
          port: Number(ENVIRONMENT_VARIABLES.REDIS_PORT),
          maxRetriesPerRequest: null,
          enableReadyCheck: true,
        });

        redis.on('connect', () => {
          console.log('✅ Redis connected');
        });

        redis.on('error', (err) => {
          console.error('❌ Redis error:', err);
        });

        return redis;
      },
    },
  ],
  exports: [REDIS_KEYS.REDIS_CLIENT],
})
export class RedisModule {}

```

## File: connectfy-api-gateway/src/app-settings/tcp-connections/tcp-connection.service.ts
```typescript
import { Inject, Injectable } from '@nestjs/common';
import { ClientProxy } from '@nestjs/microservices';
import {
  CLS_KEYS,
  emitWithContextTcp,
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

    @Inject(MICROSERVICE_NAMES.TCP.AUTH)
    private readonly authService: ClientProxy,

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

  private async emitTcpWithContext({
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

    return await emitWithContextTcp({
      client,
      endpoint,
      payload: enrichedPayload,
      cls: this.cls,
    });
  }

  // ===============================
  // Auth service
  // ===============================
  async auth(opts: Omit<ISendWithContextClientParams, 'client' | 'cls'>) {
    return await this.sendTcpWithContext({
      client: this.authService,
      ...opts,
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
    conf?: {
      isEmit?: boolean;
    },
  ) {
    if (!conf?.isEmit) {
      return await this.sendTcpWithContext({
        client: this.notificationActionHistoryService,
        ...opts,
      });
    }

    await this.emitTcpWithContext({
      client: this.notificationActionHistoryService,
      ...opts,
    });
  }
}

```

## File: connectfy-api-gateway/src/app-settings/tcp-connections/tcp-connection.module.ts
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
          name: MICROSERVICE_NAMES.TCP.AUTH,
          transport: Transport.TCP,
          options: {
            host: ENVIRONMENT_VARIABLES.AUTH_SERVICE_HOST,
            port: ENVIRONMENT_VARIABLES.AUTH_SERVICE_PORT,
          },
        },
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

## File: connectfy-api-gateway/src/guards/safeQuery.guard.ts
```typescript
import {
  HttpStatus,
  Injectable,
  CanActivate,
  ExecutionContext,
  BadRequestException,
} from '@nestjs/common';
import { Request } from 'express';
import { BaseException } from 'connectfy-shared';

@Injectable()
export class SafeQueryGuard implements CanActivate {
  canActivate(context: ExecutionContext): boolean {
    const request = context.switchToHttp().getRequest<Request>();
    const body = request.body;

    if (!body || !body.query) return true;

    try {
      body.query = this.parseSafeQuery(body.query);
    } catch (e) {
      throw new BadRequestException('Invalid query format: ' + e.message);
    }

    return true;
  }

  private parseSafeQuery(queryItems: any[]): Record<string, any> {
    if (!Array.isArray(queryItems)) {
      throw new BaseException(
        'Query must be an array of conditions ',
        HttpStatus.BAD_REQUEST,
      );
    }

    const groups: Record<string, any[]> = {};
    const noSubOperator: any[] = []; //
    const mongoQuery: Record<string, any> = {};
    for (const item of queryItems) {
      const { field, operator, value, subOperator } = item;

      if (!field || !operator) {
        throw new BaseException(
          'Query must be an array of conditions ',
          HttpStatus.BAD_REQUEST,
        );
      }
      const condition = this.buildCondition(field, operator, value);
      if (subOperator) {
        const key = subOperator.toLowerCase();
        if (!groups[key]) groups[key] = [];
        groups[key].push(condition);
      } else {
        noSubOperator.push(condition);
      }
    }

    if (groups.and?.length) mongoQuery.$and = groups.and;
    if (groups.or?.length) mongoQuery.$or = groups.or;
    if (groups.nor?.length) mongoQuery.$nor = groups.nor;

    for (const cond of noSubOperator) {
      Object.assign(mongoQuery, cond);
    }

    // console.log('mongoQuery',mongoQuery);
    return mongoQuery;
  }

  private escapeRegex(input: string): string {
    return input.replace(/[.*+?^${}()|[\]\\]/g, '\\$&');
  }

  private buildCondition(field: string, operator: string, value: any) {
    switch (operator) {
      case 'eq':
        return { [field]: { $eq: value } };
      case 'ne':
        return { [field]: { $ne: value } };
      case 'gt':
        return { [field]: { $gt: value } };
      case 'gte':
        return { [field]: { $gte: value } };
      case 'lt':
        return { [field]: { $lt: value } };
      case 'lte':
        return { [field]: { $lte: value } };
      case 'in':
        if (!Array.isArray(value))
          throw new BaseException(
            'in value must be an array',
            HttpStatus.BAD_REQUEST,
          );
        return { [field]: { $in: value } };

      case 'notIn':
        if (!Array.isArray(value))
          throw new BaseException(
            'notIn value must be an array',
            HttpStatus.BAD_REQUEST,
          );
        return { [field]: { $nin: value } };

      case 'between':
        if (!Array.isArray(value) || value.length !== 2) {
          throw new BaseException(
            'between must be an array with 2 values',
            HttpStatus.BAD_REQUEST,
          );
        }
        return { [field]: { $gte: value[0], $lte: value[1] } };

      case 'betweenExclusive':
        if (!Array.isArray(value) || value.length !== 2) {
          throw new BaseException(
            'betweenExclusive must be an array with 2 values',
            HttpStatus.BAD_REQUEST,
          );
        }
        return { [field]: { $gt: value[0], $lt: value[1] } };

      case 'betweenLeftInclusive':
        if (!Array.isArray(value) || value.length !== 2) {
          throw new BaseException(
            'betweenLeftInclusive must be an array with 2 values',
            HttpStatus.BAD_REQUEST,
          );
        }
        return { [field]: { $gte: value[0], $lt: value[1] } };

      case 'betweenRightInclusive':
        if (!Array.isArray(value) || value.length !== 2) {
          throw new BaseException(
            'betweenRightInclusive must be an array with 2 values',
            HttpStatus.BAD_REQUEST,
          );
        }
        return { [field]: { $gt: value[0], $lte: value[1] } };

      case 'contains':
        return { [field]: { $regex: this.escapeRegex(value), $options: 'i' } };

      case 'notContains':
        return {
          [field]: { $not: { $regex: this.escapeRegex(value), $options: 'i' } },
        };

      case 'startsWith':
        return {
          [field]: { $regex: `^${this.escapeRegex(value)}`, $options: 'i' },
        };

      case 'endsWith':
        return {
          [field]: { $regex: `${this.escapeRegex(value)}$`, $options: 'i' },
        };

      case 'startsNotWith':
        return {
          [field]: {
            $not: { $regex: `^${this.escapeRegex(value)}`, $options: 'i' },
          },
        };

      case 'endsNotWith':
        return {
          [field]: {
            $not: { $regex: `${this.escapeRegex(value)}$`, $options: 'i' },
          },
        };

      case 'exists':
        return { [field]: { $exists: Boolean(value) } };

      case 'isNull':
        return { [field]: { $eq: null } };

      case 'notNull':
        return { [field]: { $ne: null } };

      case 'regex':
        try {
          return { [field]: { $regex: this.escapeRegex(value) } };
        } catch {
          throw new BaseException(
            'Invalid regex pattern',
            HttpStatus.BAD_REQUEST,
          );
        }
      default:
        throw new BaseException(
          `Unsupported operator: ${operator}`,
          HttpStatus.BAD_REQUEST,
        );
    }
  }
}

/**
 * 
 * 
 * 
 * 
 *     "query": [
            {
                "field": "createdBy",
                "operator": "between",
                "value": [
                    3,
                    6
                ]
            },
            {
                "field": "_id",
                "operator": "eq",
                "value": "2256113e-dba7-4a43-8cc3-752013a4e3a2"
            },
            {
                "field": "workplaceId",
                "operator": "eq",
                "value": "392536c0-ca8d-42c1-99e1-a22585961fca"
            }
        ],

    
        | Operator                       | MongoDB Output                   | Description                      |
        | ------------------------------ | -------------------------------- | -------------------------------- |
        | `eq`                           | `{ $eq: value }`                 | Equals                           |
        | `ne`                           | `{ $ne: value }`                 | Not equals                       |
        | `gt`, `gte`, `lt`, `lte`       | `{ $gt: v }`, etc.               | Greater / Less comparisons       |
        | `in`, `notIn`                  | `{ $in: [] }`, `{ $nin: [] }`    | Value in/not in array            |
        | `between`                      | `{ $gte: v1, $lte: v2 }`         | Inclusive range                  |
        | `betweenExclusive`             | `{ $gt: v1, $lt: v2 }`           | Exclusive range                  |
        | `betweenLeftInclusive`         | `{ $gte: v1, $lt: v2 }`          | Left inclusive, right exclusive  |
        | `betweenRightInclusive`        | `{ $gt: v1, $lte: v2 }`          | Left exclusive, right inclusive  |
        | `contains`                     | `{ $regex: /value/i }`           | Case-insensitive substring match |
        | `notContains`                  | `{ $not: /value/i }`             | Not contains                     |
        | `startsWith`, `endsWith`       | Anchored regex                   | Begins/ends with                 |
        | `startsNotWith`, `endsNotWith` | Negated anchored regex           | Not begins/ends with             |
        | `exists`                       | `{ $exists: true/false }`        | Field existence check            |
        | `isNull`, `notNull`            | `{ $eq: null }`, `{ $ne: null }` | Check for null / not null        |
        | `regex`                        | `{ $regex: new RegExp(value) }`  | Raw regex (use carefully!)       |
 * 
 * 
 * 
 * 
 * 
 * 
 */

```

## File: connectfy-api-gateway/src/guards/auth.guard.ts
```typescript
import {
  CanActivate,
  ExecutionContext,
  HttpStatus,
  Injectable,
} from '@nestjs/common';
import { Request } from 'express';
import { JwtService } from '@nestjs/jwt';
import { ClsService } from 'nestjs-cls';
import {
  CLS_KEYS,
  CACHE_KEYS,
  ExceptionMessages,
  BaseException,
  EXPIRE_DATES,
} from 'connectfy-shared';
import { CacheService } from '@/src/app-settings/cache/cache.service';
import { ENVIRONMENT_VARIABLES } from '../common/constants/environment-variables';
import { TcpConnectionService } from '../app-settings/tcp-connections/tcp-connection.service';

@Injectable()
export class AuthGuard implements CanActivate {
  private readonly defaultUserTtl = EXPIRE_DATES.TTL.ONE_MINUTE * 15;
  private readonly defaultTokenTtl = EXPIRE_DATES.TTL.ONE_MINUTE * 15;

  constructor(
    private readonly cacheService: CacheService,
    private readonly jwtService: JwtService,
    private readonly cls: ClsService,
    private readonly tcpConnectionService: TcpConnectionService,
  ) {}

  async canActivate(context: ExecutionContext): Promise<boolean> {
    const request = context.switchToHttp().getRequest<Request>();
    const accessToken = this.extractTokenFromHeader(request);
    const refreshToken = request.cookies?.refresh_token;

    // 1. No token provided -> Force Login
    if (!accessToken || !refreshToken) {
      throw new BaseException(
        ExceptionMessages.UNAUTHORIZED_MESSAGE,
        HttpStatus.UNAUTHORIZED,
        { navigate: true },
      );
    }

    const accessSecret = ENVIRONMENT_VARIABLES.JWT_ACCESS_SECRET;
    let payload: any;

    const accessCacheKey = CACHE_KEYS.AUTH.ACCESS_TOKEN(accessToken);
    const payloadInCache =
      await this.cacheService.get<Record<string, any>>(accessCacheKey);

    if (payloadInCache) {
      payload = payloadInCache;
    } else {
      try {
        // 2. Verify token (Do NOT ignore expiration)
        payload = this.jwtService.verify(accessToken, { secret: accessSecret });
      } catch (error) {
        // 3. If strictly EXPIRED -> Standard 401 (Client will attempt refresh)
        if (error.name === 'TokenExpiredError') {
          throw new BaseException(
            ExceptionMessages.TOKEN_EXPIRED,
            HttpStatus.UNAUTHORIZED,
          );
        }
        // 4. If Invalid/Malformed -> Force Login
        throw new BaseException(
          ExceptionMessages.UNAUTHORIZED_MESSAGE,
          HttpStatus.UNAUTHORIZED,
          { navigate: true },
        );
      }
    }

    if (!payload || !payload._id) {
      throw new BaseException(
        ExceptionMessages.UNAUTHORIZED_MESSAGE,
        HttpStatus.UNAUTHORIZED,
        { navigate: true },
      );
    }

    await this.cacheService.set(accessCacheKey, payload, this.defaultTokenTtl);

    // 5. Token is valid. Check Cache.
    const cacheKey = CACHE_KEYS.AUTH.USER(payload._id);

    const loader = async () => {
      try {
        const result = await this.tcpConnectionService.auth({
          endpoint: 'auth/refresh-token/verify-token',
          payload: { access_token: accessToken, refresh_token: refreshToken },
        });

        return result?.user ?? undefined;
      } catch (error) {
        return undefined;
      }
    };

    let user;

    try {
      user = await this.cacheService.getOrSet<Record<string, any> | undefined>(
        cacheKey,
        loader,
        this.defaultUserTtl,
      );
    } catch (err) {
      // as fallback, attempt to fetch directly (best-effort)
      try {
        const direct = await loader();
        user = direct;
      } catch (err2) {
        // if direct fetch fails -> unauthorized (can't verify user)
        throw new BaseException(
          ExceptionMessages.UNAUTHORIZED_MESSAGE,
          HttpStatus.UNAUTHORIZED,
          { navigate: true },
        );
      }
    }

    if (!user) {
      // loader didn't return user -> unauthorized
      throw new BaseException(
        ExceptionMessages.UNAUTHORIZED_MESSAGE,
        HttpStatus.UNAUTHORIZED,
        { navigate: true },
      );
    }

    this.attachUser(request, user);
    return true;
  }

  private extractTokenFromHeader(request: Request): string | undefined {
    const authHeader = request.headers.authorization;
    if (!authHeader) return undefined;
    const [type, token] = authHeader.split(' ');
    return type === 'Bearer' ? token : undefined;
  }

  private attachUser(request: any, user: any) {
    request.user = user;
    this.cls.set(CLS_KEYS.USER, user);
    this.cls.set(CLS_KEYS.LANG, user.language);

    request.body = { ...request.body, _loggedUser: user };
  }
}

```

