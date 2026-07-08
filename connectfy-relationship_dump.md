# connectfy-relationship Source Dump

## File: connectfy-relationship/src/main.ts
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
          clientId: 'connectfy-relationship',
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
          groupId: 'consumer-connectfy-relationship',
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
  const HOST = String(ENVIRONMENT_VARIABLES.HOST);
  const NODE_ENV = String(ENVIRONMENT_VARIABLES.NODE_ENV);

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
    new ValidationPipe({
      transform: true,
      whitelist: true,
    }),
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

## File: connectfy-relationship/src/i18n.ts
```typescript
import i18n from 'i18next';
import { resources } from 'connectfy-i18n';

const i18nInstance = i18n.createInstance();

i18nInstance.init({
  resources,
  fallbackLng: 'en',
  lng: 'en',
  initImmediate: false,
  interpolation: { escapeValue: false },
});

export default i18nInstance;

```

## File: connectfy-relationship/src/app.module.ts
```typescript
import { Module } from '@nestjs/common';
import { MongooseModule } from '@nestjs/mongoose';
import { ENVIRONMENT_VARIABLES } from './common/constants/environment-variables';
import { ClsInterceptor, ClsModule } from 'nestjs-cls';
import { AppSettingsModule } from './app-settings/app-settings.module';
import { APP_INTERCEPTOR } from '@nestjs/core';
import { LoggedUserInterceptor } from './interceptors/logged-user.interceptor';
import { ModulesModule } from './modules/modules.module';
import { ProjectionsModule } from './projection/projections.module';
import { BullModule } from '@nestjs/bullmq';
import { EventEmitterModule } from '@nestjs/event-emitter';

@Module({
  imports: [
    MongooseModule.forRoot(ENVIRONMENT_VARIABLES.MONGO_URI, {
      dbName: ENVIRONMENT_VARIABLES.DB_NAME,
    }),
    BullModule.forRoot({
      connection: {
        host: ENVIRONMENT_VARIABLES.REDIS_HOST,
        port: ENVIRONMENT_VARIABLES.REDIS_PORT,
      },
    }),
    ClsModule.forRoot({
      global: true,
      interceptor: { mount: false },
    }),
    EventEmitterModule.forRoot({ global: true }),

    // src/modules
    ModulesModule,

    // src/app-settings
    AppSettingsModule,

    // src/projection
    ProjectionsModule,
  ],
  providers: [
    { provide: APP_INTERCEPTOR, useClass: ClsInterceptor },
    { provide: APP_INTERCEPTOR, useClass: LoggedUserInterceptor },
  ],
})
export class AppModule {}

```

## File: connectfy-relationship/src/projection/projections.module.ts
```typescript
import { Module } from '@nestjs/common';
import { ProjectionUserModule } from './user/user.module';

@Module({
  imports: [ProjectionUserModule],
})
export class ProjectionsModule {}

```

## File: connectfy-relationship/src/projection/user/user.controller.ts
```typescript
import { Controller } from '@nestjs/common';
import { UserService } from './user.service';
import {
  Ctx,
  EventPattern,
  KafkaContext,
  Payload,
  Transport,
} from '@nestjs/microservices';
import { AddUserDto } from './dto/add.user.dto';
import { EditUserDto } from './dto/edit.user.dto';
import { RemoveUserDto } from './dto/remove.user.dto';
import { commitKafkaOffset } from 'connectfy-shared';

@Controller('user')
export class UserController {
  constructor(private readonly service: UserService) {}

  @EventPattern('projection.user.created', Transport.KAFKA)
  async create(@Payload() data: AddUserDto, @Ctx() context: KafkaContext) {
    await Promise.all([this.service.create(data), commitKafkaOffset(context)]);
  }

  @EventPattern('projection.user.updated', Transport.KAFKA)
  async update(@Payload() data: EditUserDto, @Ctx() context: KafkaContext) {
    await Promise.all([this.service.update(data), commitKafkaOffset(context)]);
  }

  @EventPattern('projection.user.removed', Transport.KAFKA)
  async remove(@Payload() query: RemoveUserDto, @Ctx() context: KafkaContext) {
    await Promise.all([this.service.remove(query), commitKafkaOffset(context)]);
  }
}

```

## File: connectfy-relationship/src/projection/user/user.module.ts
```typescript
import { Module } from '@nestjs/common';
import { MongooseModule } from '@nestjs/mongoose';
import { COLLECTIONS } from 'connectfy-shared';
import { UserSchema } from './entity/user.entity';
import { UserService } from './user.service';
import { UserRepository } from './repo/user.repo';
import { UserController } from './user.controller';

@Module({
  imports: [
    MongooseModule.forFeature([
      { name: COLLECTIONS.AUTH.USER.USERS, schema: UserSchema },
    ]),
  ],
  controllers: [UserController],
  providers: [UserService, UserRepository],
  exports: [UserService, UserRepository],
})
export class ProjectionUserModule {}

```

## File: connectfy-relationship/src/projection/user/user.service.ts
```typescript
import { Injectable } from '@nestjs/common';
import { UserRepository } from './repo/user.repo';
import { AddUserDto } from './dto/add.user.dto';
import { FindUserDto } from './dto/find.user.dto';
import { EditUserDto } from './dto/edit.user.dto';
import { RemoveUserDto } from './dto/remove.user.dto';
import { IReturnedUser } from 'connectfy-shared';

@Injectable()
export class UserService {
  constructor(private readonly repo: UserRepository) {}

  async create(data: AddUserDto) {
    await this.repo.create(data);
  }

  async findOne(query: FindUserDto) {
    return this.repo.findOne(query);
  }

  async findMany(query: FindUserDto): Promise<IReturnedUser[]> {
    return this.repo.findMany(query) as unknown as IReturnedUser[];
  }

  async count(query: any): Promise<number> {
    return this.repo.count(query);
  }

  async update(data: EditUserDto) {
    await this.repo.update({ _id: data._id }, data);
  }

  async remove(data: RemoveUserDto) {
    await this.repo.remove(data);
  }
}

```

## File: connectfy-relationship/src/projection/user/dto/find.user.dto.ts
```typescript
import { BaseFindDto } from 'connectfy-shared';

export class FindUserDto extends BaseFindDto {}

```

## File: connectfy-relationship/src/projection/user/dto/edit.user.dto.ts
```typescript
import { BaseUserDto } from './base.user.dto';
import { PartialType } from '@nestjs/mapped-types';
import { FIELD_TYPE, FieldValidator } from 'connectfy-shared';

export class EditUserDto extends PartialType(BaseUserDto) {
  @FieldValidator({
    type: FIELD_TYPE.UUID,
  })
  _id: string;
}

```

## File: connectfy-relationship/src/projection/user/dto/remove.user.dto.ts
```typescript
import { BaseRemoveDto } from 'connectfy-shared';

export class RemoveUserDto extends BaseRemoveDto {}

```

## File: connectfy-relationship/src/projection/user/dto/base.user.dto.ts
```typescript
import { FieldValidator, FIELD_TYPE, USER_STATUS } from 'connectfy-shared';
import { PhoneNumberDto } from './nested/phoneNumber.dto';

export class BaseUserDto {
  @FieldValidator({
    type: FIELD_TYPE.STRING,
  })
  firstName: string;

  @FieldValidator({
    type: FIELD_TYPE.STRING,
  })
  lastName: string;

  @FieldValidator({
    type: FIELD_TYPE.STRING,
  })
  fullName: string;

  @FieldValidator({
    type: FIELD_TYPE.STRING,
  })
  username: string;

  @FieldValidator({
    type: FIELD_TYPE.EMAIL,
  })
  email: string;

  @FieldValidator({
    type: FIELD_TYPE.OBJECT,
    isOptional: true,
    classType: PhoneNumberDto,
  })
  phoneNumber: PhoneNumberDto | null;

  @FieldValidator({
    type: FIELD_TYPE.ENUM,
    enumObject: USER_STATUS,
  })
  status: USER_STATUS;

  @FieldValidator({
    type: FIELD_TYPE.STRING,
    isOptional: true,
  })
  avatar: string | null;
}

```

## File: connectfy-relationship/src/projection/user/dto/add.user.dto.ts
```typescript
import { BaseUserDto } from './base.user.dto';

export class AddUserDto extends BaseUserDto {}

```

## File: connectfy-relationship/src/projection/user/dto/nested/phoneNumber.dto.ts
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

## File: connectfy-relationship/src/projection/user/interface/user.interface.ts
```typescript
import { USER_STATUS } from 'connectfy-shared';
import { IPhoneNumber } from './nested/phoneNumber.interface';

export interface IUser {
  _id: string;
  firstName: string;
  lastName: string;
  username: string;
  email: string;
  phoneNumber: IPhoneNumber | null;
  status: USER_STATUS;
  avatar: string | null;
  createdAt: Date;
  updatedAt: Date;
}

export interface IReturnedUser {
  _id: string;
  firstName: string;
  lastName: string;
  username: string;
  email: string;
  phoneNumber: IPhoneNumber | null;
  status: USER_STATUS;
  avatar: string | null;
  createdAt: Date;
  updatedAt: Date;
}

```

## File: connectfy-relationship/src/projection/user/interface/nested/phoneNumber.interface.ts
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

## File: connectfy-relationship/src/projection/user/repo/user.repo.ts
```typescript
import { Injectable } from '@nestjs/common';
import { UserDocument } from '../entity/user.entity';
import { AddUserDto } from '../dto/add.user.dto';
import { EditUserDto } from '../dto/edit.user.dto';
import { InjectModel } from '@nestjs/mongoose';
import { Model } from 'mongoose';
import { IReturnedUser } from '../interface/user.interface';
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

## File: connectfy-relationship/src/projection/user/entity/user.entity.ts
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
import { LANGUAGE, COLLECTIONS, USER_STATUS } from 'connectfy-shared';

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
        field: 'firstName',
      }),
    ],
    maxlength: [
      50,
      t('validation_messages.max_length', {
        lng: LANGUAGE.EN,
        field: 'firstName',
        length: 50,
      }),
    ],
    trim: true,
    set: (value: string) =>
      value?.charAt(0).toUpperCase() + value?.slice(1).toLowerCase(),
  })
  firstName: string;

  @Prop({
    type: String,
    required: [
      true,
      t('validation_messages.required', {
        lng: LANGUAGE.EN,
        field: 'lastName',
      }),
    ],
    maxlength: [
      50,
      t('validation_messages.max_length', {
        lng: LANGUAGE.EN,
        field: 'lastName',
        length: 50,
      }),
    ],
    trim: true,
    set: (value: string) =>
      value?.charAt(0).toUpperCase() + value?.slice(1).toLowerCase(),
  })
  lastName: string;

  @Prop({
    type: String,
    required: [
      true,
      t('validation_messages.required', {
        lng: LANGUAGE.EN,
        field: 'fullName',
      }),
    ],
    maxlength: [
      50,
      t('validation_messages.max_length', {
        lng: LANGUAGE.EN,
        field: 'fullName',
        length: 101,
      }),
    ],
    trim: true,
  })
  fullName: string;

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
  })
  avatar: string | null;

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

export type UserDocument = HydratedDocument<UserModel>;

```

## File: connectfy-relationship/src/projection/user/entity/nested/phoneNumber.entity.ts
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

## File: connectfy-relationship/src/interceptors/logged-user.interceptor.ts
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

## File: connectfy-relationship/src/modules/modules.module.ts
```typescript
import { Module } from '@nestjs/common';
import { BlocklistsModule } from './blocklists/blocklists.module';
import { FriendshipsModule } from './friendships/friendships.module';

@Module({
  imports: [BlocklistsModule, FriendshipsModule],
})
export class ModulesModule {}

```

## File: connectfy-relationship/src/modules/blocklists/blocklists.module.ts
```typescript
import { Module } from '@nestjs/common';
import { BlocklistModule } from './blocklist/blocklist.module';

@Module({
  imports: [BlocklistModule],
})
export class BlocklistsModule {}

```

## File: connectfy-relationship/src/modules/blocklists/blocklist/blocklist.controller.ts
```typescript
import { Controller } from '@nestjs/common';
import { MessagePattern, Payload, Transport } from '@nestjs/microservices';
import { BlocklistService } from './blocklist.service';
import { FindBlockedUsersDto } from './dto/find.blocklist.dto';
import { BlockDto, UnblockDto } from './dto/actions.blocklist.dto';

@Controller()
export class BlocklistController {
  constructor(private readonly service: BlocklistService) {}

  @MessagePattern('blocklist/findMany', Transport.TCP)
  findBlockedUsers(@Payload() data: FindBlockedUsersDto) {
    return this.service.findBlockedUsers(data);
  }

  @MessagePattern('blocklist/isExist', Transport.TCP)
  isExist(@Payload() data: Record<string, any>) {
    return this.service.isExist(data);
  }

  @MessagePattern('blocklist/findBlockedUserIds', Transport.TCP)
  findBlockedUserIds(@Payload() data: { userId: string }) {
    return this.service.findBlockedUserIds(data.userId);
  }

  @MessagePattern('blocklist/remove', Transport.TCP)
  unblock(@Payload() data: UnblockDto) {
    return this.service.unblock(data);
  }

  @MessagePattern('blocklist/create', Transport.TCP)
  block(@Payload() data: BlockDto) {
    return this.service.block(data);
  }
}

```

## File: connectfy-relationship/src/modules/blocklists/blocklist/blocklist.module.ts
```typescript
import { Module } from '@nestjs/common';
import { BlocklistController } from './blocklist.controller';
import { BlocklistService } from './blocklist.service';
import { BlocklistRepository } from './repo/blocklist.repo';
import { MongooseModule } from '@nestjs/mongoose';
import { BlocklistSchema } from './entity/blocklist.entity';
import { COLLECTIONS } from 'connectfy-shared';
import { ProjectionUserModule } from '@/src/projection/user/user.module';

@Module({
  imports: [
    MongooseModule.forFeature([
      { name: COLLECTIONS.RELATIONSHIP.BLOCKS_LIST, schema: BlocklistSchema },
    ]),
    ProjectionUserModule,
  ],
  controllers: [BlocklistController],
  providers: [BlocklistService, BlocklistRepository],
  exports: [BlocklistService, BlocklistRepository],
})
export class BlocklistModule {}

```

## File: connectfy-relationship/src/modules/blocklists/blocklist/blocklist.service.ts
```typescript
import { Injectable, HttpStatus } from '@nestjs/common';
import { BlocklistRepository } from './repo/blocklist.repo';
import { UserService } from '../../../projection/user/user.service';
import { FindBlockedUsersDto } from './dto/find.blocklist.dto';
import { UnblockDto } from './dto/actions.blocklist.dto';
import {
  IFindAllResponse,
  BaseException,
  ExceptionMessages,
  LANGUAGE,
  CLS_KEYS,
  IReturnedUser,
} from 'connectfy-shared';
import { ClsService } from 'nestjs-cls';

@Injectable()
export class BlocklistService {
  constructor(
    private readonly repo: BlocklistRepository,
    private readonly userService: UserService,
    private readonly cls: ClsService,
  ) {}

  async isExist(query: Record<string, any>) {
    if (query.currentUserId && query.targetUserId) {
      const { currentUserId, targetUserId } = query;
      const blocks = await this.repo.findMany({
        query: {
          $or: [
            { blocker: currentUserId, blocked: targetUserId },
            { blocker: targetUserId, blocked: currentUserId },
          ],
        },
      });

      let hasBlocked = false;
      let isBlockedBy = false;

      for (const block of blocks as any[]) {
        if (
          block.blocker.toString() === currentUserId.toString() &&
          block.blocked.toString() === targetUserId.toString()
        ) {
          hasBlocked = true;
        }
        if (
          block.blocker.toString() === targetUserId.toString() &&
          block.blocked.toString() === currentUserId.toString()
        ) {
          isBlockedBy = true;
        }
      }

      return {
        isBlocked: hasBlocked || isBlockedBy,
        hasBlocked,
      };
    }

    const res = await this.repo.findOne(query);
    
    if (res && (query.blocker || query.blocked)) {
      return {
        isBlocked: query.blocker === res.blocker && query.blocked === res.blocked,
        hasBlocked: query.blocker === res.blocker && query.blocked !== res.blocked,
      };
    }

    return res;
  }

  async findBlockedUsers(
    params: FindBlockedUsersDto,
  ): Promise<IFindAllResponse<any>> {
    const { blockerId, skip, limit, search } = params;
    const offset = (skip - 1) * limit;

    let searchCondition: Record<string, any> = {};

    if (search?.trim()) {
      const regex = new RegExp(search.trim(), 'i');

      const matchedUsers = await this.userService.findMany({
        query: {
          $or: [{ username: regex }, { firstName: regex }, { lastName: regex }],
        },
        fields: '_id',
      });

      const matchedIds = matchedUsers.map((u) => u._id);

      if (!matchedIds.length) {
        return {
          data: [],
          limit,
          totalCount: 0,
          currentPage: skip,
          totalPages: 0,
        };
      }

      searchCondition = { blocked: { $in: matchedIds } };
    }

    const baseQuery = {
      $and: [
        { blocker: blockerId },
        ...(Object.keys(searchCondition).length ? [searchCondition] : []),
      ],
    };

    const [data, count] = await Promise.all([
      this.repo.findMany({
        query: baseQuery,
        skip: offset,
        limit,
        populate: [
          {
            path: 'blocked',
            select: '_id firstName lastName username avatar',
          },
        ],
      }),
      this.repo.count(baseQuery),
    ]);

    return {
      data: data as any,
      limit,
      totalCount: count,
      currentPage: skip,
      totalPages: Math.ceil(count / limit),
    };
  }

  async unblock(data: UnblockDto): Promise<boolean> {
    const lang = this.cls.get<LANGUAGE>(CLS_KEYS.LANG);
    const { _id: currentUserId } = this.cls.get<IReturnedUser>(CLS_KEYS.USER);
    const { blockedId } = data;

    const isExist = await this.repo.findOne({
      query: { blocker: currentUserId, blocked: blockedId },
      fields: '_id',
    });

    if (!isExist) {
      throw new BaseException(
        ExceptionMessages.NOT_FOUND_MESSAGE(lang),
        HttpStatus.NOT_FOUND,
      );
    }

    await this.repo.removeManyByQuery({
      blocker: currentUserId,
      blocked: blockedId,
    });

    return true;
  }

  async block(data: UnblockDto): Promise<boolean> {
    const lang = this.cls.get<LANGUAGE>(CLS_KEYS.LANG);
    const { _id: currentUserId } = this.cls.get<IReturnedUser>(CLS_KEYS.USER);
    const { blockedId } = data;

    const isExist = await this.repo.findOne({
      query: { blocker: currentUserId, blocked: blockedId },
      fields: '_id',
    });

    if (isExist) {
      throw new BaseException(
        ExceptionMessages.ALREADY_EXISTS_MESSAGE(lang),
        HttpStatus.BAD_REQUEST,
      );
    }

    await this.repo.create({
      blocker: currentUserId as any,
      blocked: blockedId as any,
    });

    return true;
  }

  async findBlockedUserIds(userId: string): Promise<string[]> {
    const blocks = await this.repo.findMany({
      query: { $or: [{ blocker: userId }, { blocked: userId }] },
      fields: 'blocker blocked',
    });

    const ids = new Set<string>();
    for (const b of blocks) {
      if (b.blocker !== userId) ids.add(b.blocker);
      if (b.blocked !== userId) ids.add(b.blocked);
    }
    return Array.from(ids);
  }
}

```

## File: connectfy-relationship/src/modules/blocklists/blocklist/dto/actions.blocklist.dto.ts
```typescript
import { FIELD_TYPE, FieldValidator } from 'connectfy-shared';

export class UnblockDto {
  @FieldValidator({ type: FIELD_TYPE.UUID })
  blockedId: string;
}

export class BlockDto {
  @FieldValidator({ type: FIELD_TYPE.UUID })
  blockedId: string;
}

```

## File: connectfy-relationship/src/modules/blocklists/blocklist/dto/base.blocklist.dto.ts
```typescript
import { FIELD_TYPE, FieldValidator } from 'connectfy-shared';

export class BaseBlocklistDto {
  @FieldValidator({ type: FIELD_TYPE.UUID })
  blocker: string;

  @FieldValidator({ type: FIELD_TYPE.UUID })
  blocked: string;
}

```

## File: connectfy-relationship/src/modules/blocklists/blocklist/dto/remove.blocklist.dto.ts
```typescript
import { BaseRemoveAllDto, BaseRemoveDto } from 'connectfy-shared';

export class RemoveBlocklistDto extends BaseRemoveDto {}

export class RemoveAllBlocklistsDto extends BaseRemoveAllDto {}

```

## File: connectfy-relationship/src/modules/blocklists/blocklist/dto/add.blocklist.dto.ts
```typescript
import { BaseBlocklistDto } from './base.blocklist.dto';

export class AddBlocklistDto extends BaseBlocklistDto {}

```

## File: connectfy-relationship/src/modules/blocklists/blocklist/dto/find.blocklist.dto.ts
```typescript
import { BaseFindDto, FIELD_TYPE, FieldValidator } from 'connectfy-shared';

export class FindBlocklistDto extends BaseFindDto {}

export class FindBlockedUsersDto {
  @FieldValidator({ type: FIELD_TYPE.NUMBER })
  skip: number;

  @FieldValidator({ type: FIELD_TYPE.NUMBER })
  limit: number;

  @FieldValidator({ type: FIELD_TYPE.UUID })
  blockerId: string;

  @FieldValidator({ type: FIELD_TYPE.STRING, isOptional: true })
  search?: string;
}

```

## File: connectfy-relationship/src/modules/blocklists/blocklist/interface/blocklist.interface.ts
```typescript
export interface IBlocklist {
  _id: string;
  blocker: string;
  blocked: string;
  createdAt: Date;
  updatedAt: Date;
}

export interface IReturnedBlocklist {
  _id: string;
  blocker: string;
  blocked: {
    _id: string;
    firstName: string;
    lastName: string;
    username: string;
    avatar: string;
  };
  createdAt: Date;
  updatedAt: Date;
}

```

## File: connectfy-relationship/src/modules/blocklists/blocklist/repo/blocklist.repo.ts
```typescript
import { Injectable } from '@nestjs/common';
import { BaseRepository, COLLECTIONS } from 'connectfy-shared';
import { BlocklistDocument } from '../entity/blocklist.entity';
import { AddBlocklistDto } from '../dto/add.blocklist.dto';
import { IBlocklist } from '../interface/blocklist.interface';
import { InjectModel } from '@nestjs/mongoose';
import { Model } from 'mongoose';

@Injectable()
export class BlocklistRepository extends BaseRepository<
  BlocklistDocument,
  IBlocklist,
  AddBlocklistDto
> {
  constructor(
    @InjectModel(COLLECTIONS.RELATIONSHIP.BLOCKS_LIST)
    protected model: Model<BlocklistDocument>,
  ) {
    super(model);
  }
}

```

## File: connectfy-relationship/src/modules/blocklists/blocklist/entity/blocklist.entity.ts
```typescript
import { Prop, Schema, SchemaFactory } from '@nestjs/mongoose';
import { COLLECTIONS, LANGUAGE } from 'connectfy-shared';
import { IBlocklist } from '../interface/blocklist.interface';
import { t } from 'i18next';
import { v4 as uuid, validate } from 'uuid';
import { HydratedDocument } from 'mongoose';

@Schema({
  timestamps: true,
  collection: COLLECTIONS.RELATIONSHIP.BLOCKS_LIST,
  toJSON: { virtuals: true, versionKey: false, getters: true },
  toObject: { virtuals: true, versionKey: false },
  autoIndex: process.env.NODE_ENV !== 'production',
  minimize: false,
  strict: true,
  strictQuery: true,
})
export class BlocklistModel implements IBlocklist {
  @Prop({ type: String, default: () => uuid(), immutable: true })
  _id: string;

  @Prop({
    type: String,
    required: [
      true,
      t('validation_messages.required', {
        lng: LANGUAGE.EN,
        field: 'blocker',
      }),
    ],
    index: true,
    immutable: true,
    trim: true,
    validate: {
      validator: function (value: string | null): boolean {
        return !!(value && validate(value));
      },
      message: t('validation_messages.uuid', {
        lng: LANGUAGE.EN,
        field: 'blocker',
      }),
    },
    ref: COLLECTIONS.AUTH.USER.USERS,
  })
  blocker: string;

  @Prop({
    type: String,
    required: [
      true,
      t('validation_messages.required', {
        lng: LANGUAGE.EN,
        field: 'blocked',
      }),
    ],
    index: true,
    immutable: true,
    trim: true,
    validate: {
      validator: function (value: string | null): boolean {
        return !!(value && validate(value));
      },
      message: t('validation_messages.uuid', {
        lng: LANGUAGE.EN,
        field: 'blocked',
      }),
    },
    ref: COLLECTIONS.AUTH.USER.USERS,
  })
  blocked: string;

  createdAt: Date;
  updatedAt: Date;
}

export const BlocklistSchema = SchemaFactory.createForClass(BlocklistModel);

BlocklistSchema.index({ blocker: 1, blocked: 1 }, { unique: true });

export type BlocklistDocument = HydratedDocument<BlocklistModel>;

```

## File: connectfy-relationship/src/modules/friendships/friendships.module.ts
```typescript
import { Module } from '@nestjs/common';
import { FriendshipModule } from './friendship/friendship.module';

@Module({
  imports: [FriendshipModule],
})
export class FriendshipsModule {}

```

## File: connectfy-relationship/src/modules/friendships/friendship/friendship.service.ts
```typescript
import { InjectQueue } from '@nestjs/bullmq';
import { forwardRef, Inject, Injectable } from '@nestjs/common';
import { FriendshipRepository } from './repo/friendship.repo';
import { AddFriendshipDto } from './dto/add.friendship.dto';
import {
  AcceptFriendshipRequestDto,
  CancelFriendshipRequestDto,
  DeclineFriendshipRequestDto,
  UnfriendDto,
  UpdateCloseFriendDto,
  UpdateNotificationDto,
} from './dto/actions.friendship.dto';
import {
  BaseException,
  CACHE_KEYS,
  CLS_KEYS,
  EXPIRE_DATES,
  ExceptionMessages,
  FriendshipRequestType,
  FriendshipStatus,
  HttpStatus,
  IFindAllResponse,
  ILoggedUser,
  IReturnedUser,
  LANGUAGE,
  NotificationType,
} from 'connectfy-shared';
import { ClsService } from 'nestjs-cls';
import { TcpConnectionService } from '@/src/app-settings/tcp-connections/tcp-connection.service';
import {
  IFriendship,
  IReturnedFriendship,
} from './interface/friendship.interface';
import {
  FindFriendshipsDto,
  FindFriendshipRequestsDto,
  FindInternalFriendshipDto,
  FindManyInternalFriendshipDto,
  FindFriendIdsDto,
  FindSuggestionsDto,
} from './dto/find.friendship.dto';
import { BlocklistService } from '../../blocklists/blocklist/blocklist.service';
import { COLLECTIONS } from 'connectfy-shared';
import { UserService } from '@/src/projection/user/user.service';
import { CacheService } from '@/src/app-settings/cache/cache.service';
import { Queue } from 'bullmq';
import { FRIENDSHIP_BULL_MQ_QUEUES } from './constants/friendship.bullmq';
import { BullMqConnectionService } from '@/src/app-settings/bullMq-connection/bullMq-connection.service';

@Injectable()
export class FriendshipService {
  constructor(
    @Inject(forwardRef(() => BlocklistService))
    private readonly blocklistService: BlocklistService,
    @InjectQueue(FRIENDSHIP_BULL_MQ_QUEUES.NOTIFICATION.QUEUE_NAME)
    private readonly notificationQueue: Queue,

    private readonly cls: ClsService,
    private readonly userService: UserService,
    private readonly repo: FriendshipRepository,
    private readonly cacheService: CacheService,
    private readonly tcpConnectionService: TcpConnectionService,
    private readonly bullMqConnectionService: BullMqConnectionService,
  ) {}

  private async findPrivacySettings(userId: string) {
    const cacheKey = CACHE_KEYS.ACCOUNT.SETTINGS.PRIVACY(userId);

    return await this.cacheService.getOrSet(
      cacheKey,
      async () =>
        await this.tcpConnectionService.account({
          endpoint: 'privacy-settings/findOne',
          payload: {
            query: { userId },
          },
        }),
      EXPIRE_DATES.TTL.ONE_MINUTE * 15,
    );
  }

  private async syncFriendshipNotificationMetadata(params: {
    recipientId: string;
    actorId?: string;
    friendshipId: string;
    isAccepted?: boolean;
    isDeclined?: boolean;
  }): Promise<void> {
    await this.tcpConnectionService
      .notificationActionHistory({
        endpoint: 'notification/friendship/metadata',
        payload: {
          ...params,
          type: NotificationType.FRIENDSHIP_REQUEST_SENT,
        },
      })
      .catch(() => {});
  }

  // ========================================
  // Send Friendship Request
  // ========================================
  async create(data: AddFriendshipDto): Promise<IFriendship> {
    const loggedUser = this.cls.get<ILoggedUser>(CLS_KEYS.USER);
    const { _id: currentUserId } = loggedUser;
    const lang = this.cls.get<LANGUAGE>(CLS_KEYS.LANG);

    const isExist = await this.repo.findOne({
      query: {
        $or: [
          { userId: currentUserId, friendId: data.friendId },
          { userId: data.friendId, friendId: currentUserId },
        ],
      },
      fields: '_id userId friendId status',
    });

    if (isExist) {
      throw new BaseException(
        ExceptionMessages.BAD_REQUEST_MESSAGE(lang),
        HttpStatus.BAD_REQUEST,
        { relationship: isExist },
      );
    }

    const privacySettings = await this.findPrivacySettings(data.friendId);

    if (!privacySettings || !privacySettings.friendshipRequest) {
      throw new BaseException(
        ExceptionMessages.FORBIDDEN_MESSAGE(lang),
        HttpStatus.FORBIDDEN,
      );
    }

    const isBlocked = await this.blocklistService.isExist({
      $or: [
        { userId: currentUserId, friendId: data.friendId },
        { userId: data.friendId, friendId: currentUserId },
      ],
    });

    if (isBlocked) {
      throw new BaseException(
        ExceptionMessages.FORBIDDEN_MESSAGE(lang),
        HttpStatus.FORBIDDEN,
      );
    }

    const res = await this.repo.create({
      ...data,
      userId: currentUserId,
      status: FriendshipStatus.Pending,
    });

    this.bullMqConnectionService.add({
      queue: this.notificationQueue,
      queueName: FRIENDSHIP_BULL_MQ_QUEUES.NOTIFICATION.REQUEST_SENT,
      payload: {
        actorId: loggedUser._id,
        actorUsername: loggedUser.username,
        actorAvatar: loggedUser.avatar?.url ?? null,
        recipientId: data.friendId,
        friendshipId: res._id,
      },
    });

    return res;
  }

  // ========================================
  // Accept Friendship Request
  // ========================================
  async acceptFriendshipRequest(
    data: AcceptFriendshipRequestDto,
  ): Promise<IFriendship> {
    const lang = this.cls.get<LANGUAGE>(CLS_KEYS.LANG);
    const loggedUser = this.cls.get<ILoggedUser>(CLS_KEYS.USER);
    const { _id: currentUserId } = loggedUser;
    const { friendshipId } = data;

    const isExist = await this.repo.findOne({
      query: {
        $and: [{ _id: friendshipId }, { friendId: currentUserId }],
      },
      fields: '_id userId friendId status',
    });

    if (!isExist) {
      await this.syncFriendshipNotificationMetadata({
        recipientId: currentUserId,
        friendshipId,
        isDeclined: true,
      });
      throw new BaseException(
        ExceptionMessages.NOT_FOUND_MESSAGE(lang),
        HttpStatus.NOT_FOUND,
        { _id: friendshipId, alreadyCancelled: true },
      );
    }

    if (isExist.status === FriendshipStatus.Accepted) {
      const relationship = await this.repo.findOne({
        query: {
          $and: [{ userId: currentUserId }, { friendId: isExist.userId }],
        },
      });

      if (relationship) {
        await this.syncFriendshipNotificationMetadata({
          recipientId: currentUserId,
          actorId: isExist.userId,
          friendshipId,
          isAccepted: relationship.status === FriendshipStatus.Accepted,
          isDeclined: relationship.status !== FriendshipStatus.Accepted,
        });
        throw new BaseException(
          ExceptionMessages.BAD_REQUEST_MESSAGE(lang),
          HttpStatus.BAD_REQUEST,
          { relationship },
        );
      }

      throw new BaseException(
        ExceptionMessages.BAD_REQUEST_MESSAGE(lang),
        HttpStatus.BAD_REQUEST,
      );
    }

    const isBlocked = await this.blocklistService.isExist({
      $or: [
        { userId: currentUserId, friendId: isExist.userId },
        { userId: isExist.userId, friendId: currentUserId },
      ],
    });

    if (isBlocked) {
      throw new BaseException(
        ExceptionMessages.FORBIDDEN_MESSAGE(lang),
        HttpStatus.FORBIDDEN,
      );
    }

    const [res] = await Promise.all([
      this.repo.create({
        userId: currentUserId,
        friendId: isExist.userId,
        status: FriendshipStatus.Accepted,
      }),
      this.repo.update(
        { _id: friendshipId },
        {
          _id: friendshipId,
          status: FriendshipStatus.Accepted,
        },
      ),
    ]);

    await this.syncFriendshipNotificationMetadata({
      recipientId: currentUserId,
      actorId: isExist.userId,
      friendshipId,
      isAccepted: true,
      isDeclined: false,
    });

    this.bullMqConnectionService.add({
      queue: this.notificationQueue,
      queueName: FRIENDSHIP_BULL_MQ_QUEUES.NOTIFICATION.REQUEST_ACCEPTED,
      payload: {
        actorId: loggedUser._id,
        actorUsername: loggedUser.username,
        actorAvatar: loggedUser.avatar?.url ?? null,
        recipientId: isExist.userId,
        friendshipId,
      },
    });

    return res;
  }

  // ========================================
  // Decline Friendship Request
  // ========================================
  async declineFriendshipRequest(
    data: DeclineFriendshipRequestDto,
  ): Promise<boolean> {
    const lang = this.cls.get<LANGUAGE>(CLS_KEYS.LANG);
    const loggedUser = this.cls.get<ILoggedUser>(CLS_KEYS.USER);
    const { _id: currentUserId } = loggedUser;
    const { friendshipId } = data;

    const isExist = await this.repo.findOne({
      query: {
        $and: [
          { _id: friendshipId },
          { friendId: currentUserId },
          { status: FriendshipStatus.Pending },
        ],
      },
      fields: '_id userId friendId status',
    });

    if (!isExist) {
      await this.syncFriendshipNotificationMetadata({
        recipientId: currentUserId,
        friendshipId,
        isDeclined: true,
      });
      throw new BaseException(
        ExceptionMessages.NOT_FOUND_MESSAGE(lang),
        HttpStatus.NOT_FOUND,
        { _id: friendshipId, alreadyCancelled: true },
      );
    }

    await this.repo.remove({ _id: friendshipId });

    return true;
  }

  // ========================================
  // Cancel Friendship Request
  // ========================================
  async cancelFriendshipRequest(
    data: CancelFriendshipRequestDto,
  ): Promise<boolean> {
    const lang = this.cls.get<LANGUAGE>(CLS_KEYS.LANG);
    const loggedUser = this.cls.get<ILoggedUser>(CLS_KEYS.USER);
    const { _id: currentUserId } = loggedUser;
    const { friendshipId } = data;

    const isExist = await this.repo.findOne({
      query: {
        $and: [{ _id: friendshipId }, { userId: currentUserId }],
      },
    });

    if (!isExist) {
      throw new BaseException(
        ExceptionMessages.NOT_FOUND_MESSAGE(lang),
        HttpStatus.NOT_FOUND,
        { _id: friendshipId, alreadyCancelled: true },
      );
    }

    if (isExist.status !== FriendshipStatus.Pending) {
      await this.syncFriendshipNotificationMetadata({
        recipientId: currentUserId,
        actorId: isExist.userId,
        friendshipId,
        isAccepted: isExist.status === FriendshipStatus.Accepted,
        isDeclined: isExist.status !== FriendshipStatus.Accepted,
      });
      throw new BaseException(
        ExceptionMessages.BAD_REQUEST_MESSAGE(lang),
        HttpStatus.BAD_REQUEST,
        { relationship: isExist },
      );
    }

    await this.repo.remove({ _id: friendshipId });

    await this.syncFriendshipNotificationMetadata({
      recipientId: currentUserId,
      actorId: isExist.userId,
      friendshipId,
      isDeclined: true,
    });

    return true;
  }

  // ========================================
  // Unfriend
  // ========================================
  async unfriend(data: UnfriendDto): Promise<boolean> {
    const lang = this.cls.get<LANGUAGE>(CLS_KEYS.LANG);
    const { _id: currentUserId } = this.cls.get<IReturnedUser>(CLS_KEYS.USER);
    const { friendshipId } = data;

    const isExist = await this.repo.findOne({
      query: {
        $and: [{ _id: friendshipId }, { userId: currentUserId }],
      },
      fields: 'friendId',
    });

    if (!isExist) {
      throw new BaseException(
        ExceptionMessages.NOT_FOUND_MESSAGE(lang),
        HttpStatus.NOT_FOUND,
        { _id: friendshipId, alreadyCancelled: true },
      );
    }

    await this.repo.removeManyByQuery({
      $or: [
        { $and: [{ userId: currentUserId }, { friendId: isExist.friendId }] },
        { $and: [{ userId: isExist.friendId }, { friendId: currentUserId }] },
      ],
    });

    return true;
  }

  // ========================================
  // Update Close Friend
  // ========================================
  async updateCloseFriend(data: UpdateCloseFriendDto): Promise<boolean> {
    const lang = this.cls.get<LANGUAGE>(CLS_KEYS.LANG);
    const { _id: currentUserId } = this.cls.get<IReturnedUser>(CLS_KEYS.USER);
    const { friendshipId, isCloseFriend } = data;

    const isExist = await this.isFriendshipExist({
      $and: [
        { _id: friendshipId },
        { userId: currentUserId },
        { status: FriendshipStatus.Accepted },
        { isFavorite: { $ne: isCloseFriend } },
      ],
    });

    if (!isExist) {
      throw new BaseException(
        ExceptionMessages.NOT_FOUND_MESSAGE(lang),
        HttpStatus.NOT_FOUND,
      );
    }

    await this.repo.update(
      { _id: friendshipId },
      { _id: friendshipId, isFavorite: isCloseFriend },
    );

    return true;
  }

  // ========================================
  // Update Notification
  // ========================================
  async updateNotification(data: UpdateNotificationDto): Promise<boolean> {
    const lang = this.cls.get<LANGUAGE>(CLS_KEYS.LANG);
    const { _id: currentUserId } = this.cls.get<IReturnedUser>(CLS_KEYS.USER);
    const { friendshipId, notification } = data;

    const isExist = await this.isFriendshipExist({
      $and: [
        { _id: friendshipId },
        { userId: currentUserId },
        { status: FriendshipStatus.Accepted },
        { isMuted: { $ne: !notification } },
      ],
    });

    if (!isExist) {
      throw new BaseException(
        ExceptionMessages.NOT_FOUND_MESSAGE(lang),
        HttpStatus.NOT_FOUND,
      );
    }

    await this.repo.update(
      { _id: friendshipId },
      { _id: friendshipId, isMuted: !notification },
    );

    return true;
  }

  // ========================================
  // Is Friendship Exist
  // ========================================
  async isFriendshipExist(query: Record<string, any>): Promise<boolean> {
    return await this.repo.existsByField(query);
  }

  // ========================================
  // Find Many Internal
  // ========================================
  async findManyInternal(params: FindManyInternalFriendshipDto) {
    const { currentUserId, targetUserIds, fields } = params;

    const project = fields
      ? fields.split(' ').reduce((acc, field) => {
          if (field.startsWith('-')) {
            acc[field.slice(1)] = 0;
          } else {
            acc[field] = 1;
          }
          return acc;
        }, {})
      : null;

    const pipeline: any[] = [
      {
        $match: {
          $or: [
            { userId: currentUserId, friendId: { $in: targetUserIds } },
            { userId: { $in: targetUserIds }, friendId: currentUserId },
          ],
        },
      },
      {
        $addFields: {
          _targetUserId: {
            $cond: [
              { $eq: ['$userId', currentUserId] },
              '$friendId',
              '$userId',
            ],
          },
          _priority: {
            $cond: [{ $eq: ['$userId', currentUserId] }, 0, 1],
          },
        },
      },
      { $sort: { _priority: 1 } },
      {
        $group: {
          _id: '$_targetUserId',
          doc: { $first: '$$ROOT' },
        },
      },
      { $replaceRoot: { newRoot: '$doc' } },
      { $unset: ['_priority', '_targetUserId'] },

      ...(project ? [{ $project: project }] : []),
    ];

    return this.repo.aggregate(pipeline);
  }

  // ========================================
  // Find One Internal With Count
  // ========================================
  async findOneInternalWithCount(params: FindInternalFriendshipDto) {
    const { currentUserId, targetUserId, countQuery, fields } = params;

    const project = fields
      ? fields.split(' ').reduce((acc, field) => {
          if (field.startsWith('-')) {
            acc[field.slice(1)] = 0;
          } else {
            acc[field] = 1;
          }
          return acc;
        }, {})
      : { _id: 1, userId: 1, friendId: 1, status: 1 };

    const friendship = await this.repo.aggregate([
      {
        $match: {
          $or: [
            { userId: currentUserId, friendId: targetUserId },
            { userId: targetUserId, friendId: currentUserId },
          ],
        },
      },
      {
        $addFields: {
          _priority: {
            $cond: [{ $eq: ['$userId', currentUserId] }, 0, 1],
          },
        },
      },
      { $sort: { _priority: 1 } },
      { $limit: 1 },
      { $unset: '_priority' }, // ✅ Ayrı stage
      { $project: project },
    ]);

    let count = 0;

    if (countQuery) {
      count = await this.repo.count(countQuery);
    }

    return {
      data: friendship[0] ?? null,
      count,
    };
  }

  // ========================================
  // Find Friends
  // ========================================
  async findFriends(
    params: FindFriendshipsDto,
  ): Promise<IFindAllResponse<IReturnedFriendship>> {
    const { _id: currentUserId } = this.cls.get<IReturnedUser>(CLS_KEYS.USER);
    const { userId, skip, limit, search, isFavorite, isMuted } = params;
    const offset = (skip - 1) * limit;

    const isOwner = userId === currentUserId;

    // Owner deyilsə favourite/muted filterləri keçərsizdir
    if (!isOwner && (isFavorite || isMuted)) {
      return {
        data: [],
        limit,
        totalCount: 0,
        currentPage: skip,
        totalPages: 0,
      };
    }

    // Step 1: Search condition
    let searchCondition: Record<string, any> = {};

    if (search?.trim()) {
      const regex = new RegExp(search.trim(), 'i');

      const matchedUsers = await this.userService.findMany({
        query: {
          $or: [{ username: regex }, { firstName: regex }, { lastName: regex }],
        },
        fields: '_id',
      });

      const matchedIds = matchedUsers.map((u) => u._id);

      if (!matchedIds.length) {
        return {
          data: [],
          limit,
          totalCount: 0,
          currentPage: skip,
          totalPages: 0,
        };
      }

      searchCondition = { friendId: { $in: matchedIds } };
    }

    // Step 2: Filter condition (owner-only)
    const filterConditions: Record<string, any>[] = [];

    if (isOwner) {
      if (isFavorite) filterConditions.push({ isFavorite: true });
      if (isMuted) filterConditions.push({ isMuted: true });
    }

    // Step 3: Base query-ni yığ
    const baseQuery = {
      $and: [
        { userId },
        { status: FriendshipStatus.Accepted },
        ...(Object.keys(searchCondition).length ? [searchCondition] : []),
        ...filterConditions,
      ],
    };

    const [data, count] = await Promise.all([
      this.repo.findMany({
        query: baseQuery,
        skip: offset,
        limit,
        populate: [
          {
            path: 'friendId',
            select: '_id firstName lastName fullName username avatar',
          },
        ],
      }),
      this.repo.count(baseQuery),
    ]);

    return {
      data: data as any,
      limit,
      totalCount: count,
      currentPage: skip,
      totalPages: Math.ceil(count / limit),
    };
  }

  // ========================================
  // Find Friend Requests
  // ========================================
  async findFriendRequests(
    params: FindFriendshipRequestsDto,
  ): Promise<IFindAllResponse<IReturnedFriendship>> {
    const { _id: currentUserId } = this.cls.get<IReturnedUser>(CLS_KEYS.USER);
    const { requestType, skip, limit } = params;
    const offset = (skip - 1) * limit;

    const query: Record<string, any>[] = [{ status: FriendshipStatus.Pending }];
    let fieldName: 'userId' | 'friendId' = 'userId';

    if (requestType === FriendshipRequestType.Received) {
      query.push({ friendId: currentUserId });
    } else if (requestType === FriendshipRequestType.Sent) {
      query.push({ userId: currentUserId });
      fieldName = 'friendId';
    }

    const [data, count] = await Promise.all([
      this.repo.findMany({
        query: {
          $and: query,
        },
        skip: offset,
        limit,
        populate: [
          {
            path: fieldName,
            select: '_id firstName lastName fullName username avatar',
          },
        ],
        fields: `userId friendId status createdAt`,
        sort: { createdAt: -1 },
      }),
      this.repo.count({
        $and: query,
      }),
    ]);

    return {
      data: data as any,
      limit,
      totalCount: count,
      currentPage: skip,
      totalPages: Math.ceil(count / limit),
    };
  }

  // ========================================
  // Find Friendship Requests
  // ========================================
  async requestsCount(): Promise<number> {
    const { _id: currentUserId } = this.cls.get<IReturnedUser>(CLS_KEYS.USER);
    return await this.repo.count({
      friendId: currentUserId,
      status: FriendshipStatus.Pending,
    });
  }

  // ========================================
  // Find Friend IDs With Projection
  // ========================================
  async findFriendIds(data: FindFriendIdsDto) {
    const { userId } = data;
    const friendships = await this.repo.findFriendIdsWithProjection(userId);
    const friends = friendships.map((f: any) => f.friendId);
    return { friends };
  }

  // ========================================
  // Find Suggestions
  // ========================================
  async findSuggestions(data: FindSuggestionsDto) {
    const { userId, skip, limit } = data;
    const offset = (skip - 1) * limit;

    const existingFriendships = await this.repo.findMany({
      query: {
        $or: [{ userId }, { friendId: userId }],
      },
      fields: 'userId friendId status',
    });

    const existingIds = Array.from(new Set(existingFriendships.map((f: any) =>
      String(f.userId) === String(userId) ? String(f.friendId) : String(f.userId)
    )));

    const acceptedFriendships = existingFriendships.filter((f: any) => f.status === FriendshipStatus.Accepted);
    const acceptedFriendIds = Array.from(new Set(acceptedFriendships.map((f: any) =>
      String(f.userId) === String(userId) ? String(f.friendId) : String(f.userId)
    )));

    if (acceptedFriendIds.length === 0) {
      return this.findRandomSuggestions({ userId, existingIds, offset, limit });
    }

    const excludedIds = [...existingIds, userId];

    const pipeline: any[] = [
      {
        $match: {
          userId: { $in: acceptedFriendIds },
          status: FriendshipStatus.Accepted,
        }
      },
      {
        $addFields: {
          suggestedId: '$friendId'
        }
      },
      {
        $group: {
          _id: '$suggestedId',
          mutualCount: { $sum: 1 },
        }
      },
      {
        $match: {
          _id: { $nin: excludedIds }
        }
      },
      {
        $lookup: {
          from: COLLECTIONS.RELATIONSHIP.BLOCKS_LIST,
          let: { suggestedId: '$_id' },
          pipeline: [
            {
              $match: {
                $expr: {
                  $or: [
                    { $and: [{ $eq: ['$blocker', userId] }, { $eq: ['$blocked', '$$suggestedId'] }] },
                    { $and: [{ $eq: ['$blocker', '$$suggestedId'] }, { $eq: ['$blocked', userId] }] }
                  ]
                }
              }
            }
          ],
          as: 'blockData'
        }
      },
      {
        $match: {
          blockData: { $size: 0 }
        }
      },
      {
        $sort: { mutualCount: -1 }
      },
      {
        $facet: {
          paginatedResults: [
            { $skip: offset },
            { $limit: limit },
            {
              $lookup: {
                from: COLLECTIONS.AUTH.USER.USERS,
                localField: '_id',
                foreignField: '_id',
                as: 'user'
              }
            },
            {
              $unwind: '$user'
            },
            {
              $project: {
                userId: '$_id',
                firstName: '$user.firstName',
                lastName: '$user.lastName',
                fullName: '$user.fullName',
                username: '$user.username',
                avatar: '$user.avatar',
                mutualCount: 1,
                _id: 0
              }
            }
          ],
          totalCount: [
            { $count: 'count' }
          ]
        }
      }
    ];

    const result = await this.repo.aggregate(pipeline);
    const dataList = result[0]?.paginatedResults || [];
    const count = result[0]?.totalCount?.[0]?.count || 0;

    // Fallback: if mutual-friend pipeline yielded 0 results, suggest random users
    if (count === 0) {
      return this.findRandomSuggestions({ userId, existingIds, offset, limit });
    }

    return { data: dataList, count };
  }

  // ========================================
  // Helper: Find Random Suggestions (fallback)
  // ========================================
  private async findRandomSuggestions(params: {
    userId: string;
    existingIds: string[];
    offset: number;
    limit: number;
  }) {
    const { userId, existingIds, offset, limit } = params;
    const blockedIds = await this.blocklistService.findBlockedUserIds(userId);
    const excludedIds = [...new Set([...existingIds, ...blockedIds, userId])];

    const [randomUsers, count] = await Promise.all([
      this.userService.findMany({
        query: { _id: { $nin: excludedIds } },
        limit,
        skip: offset,
      }),
      this.userService.count({ _id: { $nin: excludedIds } }),
    ]);

    const data = randomUsers.map((u: any) => ({
      userId: u._id,
      firstName: u.firstName,
      lastName: u.lastName,
      fullName: u.fullName,
      username: u.username,
      avatar: u.avatar,
      mutualCount: 0,
    }));

    return { data, count };
  }

  // ========================================
  // Helper: Get Accepted Friend IDs
  // ========================================
  private async getAcceptedFriendIds(userId: string): Promise<string[]> {
    const existingFriendships = await this.repo.findMany({
      query: { userId, status: FriendshipStatus.Accepted },
      fields: 'friendId',
    });
    return Array.from(new Set(existingFriendships.map((f: any) => String(f.friendId))));
  }

  // ========================================
  // Find Mutual Friends
  // ========================================
  async findMutualFriends(data: { targetUserId: string; skip: number; limit: number }) {
    const { _id: currentUserId } = this.cls.get<IReturnedUser>(CLS_KEYS.USER);
    const { targetUserId, skip, limit } = data;

    const currentUserFriends = await this.getAcceptedFriendIds(currentUserId);
    const targetUserFriends = await this.getAcceptedFriendIds(targetUserId);

    const mutualIds = currentUserFriends.filter((id) => targetUserFriends.includes(id));
    const count = mutualIds.length;

    const offset = (skip - 1) * limit;
    const paginatedIds = mutualIds.slice(offset, offset + limit);

    if (paginatedIds.length === 0) return { data: [], count };

    const users = await this.userService.findMany({
      query: { _id: { $in: paginatedIds } },
    });

    const dataList = users.map((u: any) => ({
      userId: u._id,
      firstName: u.firstName,
      lastName: u.lastName,
      fullName: u.fullName,
      username: u.username,
      avatar: u.avatar,
    }));

    return { data: dataList, count };
  }
}

```

## File: connectfy-relationship/src/modules/friendships/friendship/friendship.module.ts
```typescript
import { forwardRef, Module } from '@nestjs/common';
import { FriendshipService } from './friendship.service';
import { FriendshipController } from './friendship.controller';
import { MongooseModule } from '@nestjs/mongoose';
import { FriendshipSchema } from './entity/friendship.entity';
import { FriendshipRepository } from './repo/friendship.repo';
import { COLLECTIONS } from 'connectfy-shared';
import { BlocklistModule } from '../../blocklists/blocklist/blocklist.module';
import { ProjectionUserModule } from '@/src/projection/user/user.module';
import { BullModule } from '@nestjs/bullmq';
import { FRIENDSHIP_BULL_MQ_QUEUES } from './constants/friendship.bullmq';
import { FriendshipNotificationProcessor } from './publishers/notification/friendship.notification.processor';
import { FriendshipNotificationService } from './publishers/notification/friendship.notification.service';

@Module({
  imports: [
    MongooseModule.forFeature([
      { name: COLLECTIONS.RELATIONSHIP.FRIENDSHIPS, schema: FriendshipSchema },
    ]),
    BullModule.registerQueue({
      name: FRIENDSHIP_BULL_MQ_QUEUES.NOTIFICATION.QUEUE_NAME,
    }),
    forwardRef(() => BlocklistModule),
    ProjectionUserModule,
  ],
  controllers: [FriendshipController],
  providers: [
    FriendshipService,
    FriendshipRepository,
    FriendshipNotificationService,
    FriendshipNotificationProcessor,
  ],
  exports: [
    FriendshipService,
    FriendshipRepository,
    FriendshipNotificationService,
    FriendshipNotificationProcessor,
  ],
})
export class FriendshipModule {}

```

## File: connectfy-relationship/src/modules/friendships/friendship/friendship.controller.ts
```typescript
import { Controller } from '@nestjs/common';
import { FriendshipService } from './friendship.service';
import { MessagePattern, Transport } from '@nestjs/microservices';
import { AddFriendshipDto } from './dto/add.friendship.dto';
import {
  AcceptFriendshipRequestDto,
  CancelFriendshipRequestDto,
  DeclineFriendshipRequestDto,
  UnfriendDto,
  UpdateCloseFriendDto,
  UpdateNotificationDto,
} from './dto/actions.friendship.dto';
import {
  FindFriendshipRequestsDto,
  FindFriendshipsDto,
  FindInternalFriendshipDto,
  FindManyInternalFriendshipDto,
  FindFriendIdsDto,
  FindSuggestionsDto,
} from './dto/find.friendship.dto';

@Controller('')
export class FriendshipController {
  constructor(private readonly service: FriendshipService) {}

  @MessagePattern('friendship/create', Transport.TCP)
  async create(data: AddFriendshipDto) {
    return this.service.create(data);
  }

  @MessagePattern('friendship/acceptFriendshipRequest', Transport.TCP)
  async acceptFriendshipRequest(data: AcceptFriendshipRequestDto) {
    return this.service.acceptFriendshipRequest(data);
  }

  @MessagePattern('friendship/declineFriendshipRequest', Transport.TCP)
  async declineFriendshipRequest(data: DeclineFriendshipRequestDto) {
    return this.service.declineFriendshipRequest(data);
  }

  @MessagePattern('friendship/cancelFriendshipRequest', Transport.TCP)
  async cancelFriendshipRequest(data: CancelFriendshipRequestDto) {
    return this.service.cancelFriendshipRequest(data);
  }

  @MessagePattern('friendship/unfriend', Transport.TCP)
  async unfriend(data: UnfriendDto) {
    return this.service.unfriend(data);
  }

  @MessagePattern('friendship/updateCloseFriend', Transport.TCP)
  async updateCloseFriend(data: UpdateCloseFriendDto) {
    return this.service.updateCloseFriend(data);
  }

  @MessagePattern('friendship/updateNotification', Transport.TCP)
  async updateNotification(data: UpdateNotificationDto) {
    return this.service.updateNotification(data);
  }

  @MessagePattern('friendship/findManyInternal', Transport.TCP)
  async findManyInternal(data: FindManyInternalFriendshipDto) {
    return this.service.findManyInternal(data);
  }

  @MessagePattern('friendship/findOneInternalWithCount', Transport.TCP)
  async findOneInternalWithCount(data: FindInternalFriendshipDto) {
    return this.service.findOneInternalWithCount(data);
  }

  @MessagePattern('friendship/findFriends', Transport.TCP)
  async findFriends(data: FindFriendshipsDto) {
    return this.service.findFriends(data);
  }

  @MessagePattern('friendship/findRequests', Transport.TCP)
  async findFriendRequests(data: FindFriendshipRequestsDto) {
    return this.service.findFriendRequests(data);
  }

  @MessagePattern('friendship/requestsCount', Transport.TCP)
  async requestsCount() {
    return this.service.requestsCount();
  }

  @MessagePattern('friendship/findFriendIds', Transport.TCP)
  async findFriendIds(data: FindFriendIdsDto) {
    return this.service.findFriendIds(data);
  }

  @MessagePattern('friendship/findSuggestions', Transport.TCP)
  async findSuggestions(data: FindSuggestionsDto) {
    return this.service.findSuggestions(data);
  }

  @MessagePattern('friendship/findMutualFriends', Transport.TCP)
  async findMutualFriends(data: { targetUserId: string; skip: number; limit: number }) {
    return this.service.findMutualFriends(data);
  }
}

```

## File: connectfy-relationship/src/modules/friendships/friendship/dto/add.friendship.dto.ts
```typescript
import { FIELD_TYPE, FieldValidator, FriendshipStatus } from 'connectfy-shared';

export class AddFriendshipDto {
  @FieldValidator({ type: FIELD_TYPE.UUID })
  userId: string;

  @FieldValidator({ type: FIELD_TYPE.UUID })
  friendId: string;

  @FieldValidator({
    type: FIELD_TYPE.ENUM,
    enumObject: FriendshipStatus,
    isOptional: true,
  })
  status?: FriendshipStatus;
}

```

## File: connectfy-relationship/src/modules/friendships/friendship/dto/find.friendship.dto.ts
```typescript
import {
  FIELD_TYPE,
  FieldValidator,
  FriendshipRequestType,
} from 'connectfy-shared';

export class FindInternalFriendshipDto {
  @FieldValidator({ type: FIELD_TYPE.OBJECT, isOptional: true })
  countQuery?: Record<string, any>;

  @FieldValidator({ type: FIELD_TYPE.UUID })
  currentUserId: string;

  @FieldValidator({ type: FIELD_TYPE.UUID })
  targetUserId: string;

  @FieldValidator({ type: FIELD_TYPE.STRING, isOptional: true })
  fields?: string;
}

export class FindManyInternalFriendshipDto {
  @FieldValidator({ type: FIELD_TYPE.UUID })
  currentUserId: string;

  @FieldValidator({ type: FIELD_TYPE.ARRAY, isOptional: true })
  targetUserIds?: string[];

  @FieldValidator({ type: FIELD_TYPE.STRING, isOptional: true })
  fields?: string;
}

export class BaseFindFriendshipDto {
  @FieldValidator({ type: FIELD_TYPE.NUMBER })
  skip: number;

  @FieldValidator({ type: FIELD_TYPE.NUMBER })
  limit: number;
}

export class FindFriendshipsDto extends BaseFindFriendshipDto {
  @FieldValidator({ type: FIELD_TYPE.UUID })
  userId: string;

  @FieldValidator({ type: FIELD_TYPE.STRING, isOptional: true })
  search?: string;

  @FieldValidator({ type: FIELD_TYPE.BOOLEAN, isOptional: true })
  isFavorite?: boolean;

  @FieldValidator({ type: FIELD_TYPE.BOOLEAN, isOptional: true })
  isMuted?: boolean;
}

export class FindFriendshipRequestsDto extends BaseFindFriendshipDto {
  @FieldValidator({ type: FIELD_TYPE.ENUM, enumObject: FriendshipRequestType })
  requestType: FriendshipRequestType;
}

export class FindFriendIdsDto {
  @FieldValidator({ type: FIELD_TYPE.UUID })
  userId: string;
}

export class FindSuggestionsDto extends BaseFindFriendshipDto {
  @FieldValidator({ type: FIELD_TYPE.UUID })
  userId: string;
}

```

## File: connectfy-relationship/src/modules/friendships/friendship/dto/edit.friendship.dto.ts
```typescript
import { PartialType } from '@nestjs/mapped-types';
import { BaseFriendshipDto } from './base.friendhip.dto';
import { FIELD_TYPE, FieldValidator } from 'connectfy-shared';

export class EditFriendshipDto extends PartialType(BaseFriendshipDto) {
  @FieldValidator({ type: FIELD_TYPE.UUID })
  _id: string;
}

```

## File: connectfy-relationship/src/modules/friendships/friendship/dto/remove.friendship.dto.ts
```typescript
import { BaseRemoveDto } from 'connectfy-shared';

export class RemoveFriendshipDto extends BaseRemoveDto {}

export class RemoveAllFriendshipsDto extends BaseRemoveDto {}

```

## File: connectfy-relationship/src/modules/friendships/friendship/dto/actions.friendship.dto.ts
```typescript
import { FIELD_TYPE, FieldValidator } from 'connectfy-shared';

export class DeclineFriendshipRequestDto {
  @FieldValidator({ type: FIELD_TYPE.UUID })
  friendshipId: string;
}

export class CancelFriendshipRequestDto {
  @FieldValidator({ type: FIELD_TYPE.UUID })
  friendshipId: string;
}

export class UnfriendDto {
  @FieldValidator({ type: FIELD_TYPE.UUID })
  friendshipId: string;
}

export class AcceptFriendshipRequestDto {
  @FieldValidator({ type: FIELD_TYPE.UUID })
  friendshipId: string;
}

export class UpdateCloseFriendDto {
  @FieldValidator({ type: FIELD_TYPE.UUID })
  friendshipId: string;

  @FieldValidator({ type: FIELD_TYPE.BOOLEAN })
  isCloseFriend: boolean;
}

export class UpdateNotificationDto {
  @FieldValidator({ type: FIELD_TYPE.UUID })
  friendshipId: string;

  @FieldValidator({ type: FIELD_TYPE.BOOLEAN })
  notification: boolean;
}

```

## File: connectfy-relationship/src/modules/friendships/friendship/dto/base.friendhip.dto.ts
```typescript
import { FIELD_TYPE, FieldValidator, FriendshipStatus } from 'connectfy-shared';

export class BaseFriendshipDto {
  @FieldValidator({
    type: FIELD_TYPE.ENUM,
    enumObject: FriendshipStatus,
  })
  status: FriendshipStatus;

  @FieldValidator({ type: FIELD_TYPE.BOOLEAN })
  isFavorite: boolean;

  @FieldValidator({ type: FIELD_TYPE.BOOLEAN })
  isMuted: boolean;
}

```

## File: connectfy-relationship/src/modules/friendships/friendship/interface/friendship.interface.ts
```typescript
import {
  FriendshipStatus,
  IReturnedUser,
  LANGUAGE,
  NotificationType,
} from 'connectfy-shared';

export interface IFriendship {
  _id: string;
  userId: string;
  friendId: string;
  status: FriendshipStatus;
  isFavorite: boolean;
  isMuted: boolean;
  createdAt: Date;
  updatedAt: Date;
}

export interface IReturnedFriendship {
  _id: string;
  userId: string;
  friendId: {
    _id: string;
    firstName: string;
    lastName: string;
    username: string;
    avatar: string;
  };
  status: FriendshipStatus;
  isFavorite: boolean;
  isMuted: boolean;
  createdAt: Date;
  updatedAt: Date;
}

export interface IFriendshipJobData {
  actorId: string;
  actorUsername: string;
  actorAvatar: string | null;
  recipientId: string;
  friendshipId: string;
  _loggedUser: IReturnedUser;
  _lang: LANGUAGE;
}

export interface IFriendshipNotificationEvent extends IFriendshipJobData {
  type: NotificationType;
  timestamp: string;
}

```

## File: connectfy-relationship/src/modules/friendships/friendship/repo/friendship.repo.ts
```typescript
import { Injectable } from '@nestjs/common';
import { FriendshipDocument } from '../entity/friendship.entity';
import { InjectModel } from '@nestjs/mongoose';
import { Model } from 'mongoose';
import { AddFriendshipDto } from '../dto/add.friendship.dto';
import { BaseRepository, COLLECTIONS, FriendshipStatus } from 'connectfy-shared';
import { IFriendship } from '../interface/friendship.interface';
import { EditFriendshipDto } from '../dto/edit.friendship.dto';

@Injectable()
export class FriendshipRepository extends BaseRepository<
  FriendshipDocument,
  IFriendship,
  AddFriendshipDto,
  EditFriendshipDto
> {
  constructor(
    @InjectModel(COLLECTIONS.RELATIONSHIP.FRIENDSHIPS)
    protected model: Model<FriendshipDocument>,
  ) {
    super(model);
  }

  async findFriendIdsWithProjection(userId: string) {
    return this.model
      .find({
        userId,
        status: FriendshipStatus.Accepted
      })
      .populate({
        path: 'friendId',
        select: '_id firstName lastName fullName username avatar',
      })

      .limit(500) // MVP-scale cap, revisit if friend counts grow much larger
      .lean();
  }
}

```

## File: connectfy-relationship/src/modules/friendships/friendship/constants/friendship.bullmq.ts
```typescript
export const FRIENDSHIP_BULL_MQ_QUEUES = {
  NOTIFICATION: {
    QUEUE_NAME: 'friendship_notification_queue',
    REQUEST_SENT: 'friendship.notification.request.sent',
    REQUEST_ACCEPTED: 'friendship.notification.request.accepted',
  },
};

```

## File: connectfy-relationship/src/modules/friendships/friendship/entity/friendship.entity.ts
```typescript
import { v4 as uuid, validate } from 'uuid';
import { Prop, Schema, SchemaFactory } from '@nestjs/mongoose';
import { IFriendship } from '../interface/friendship.interface';
import { COLLECTIONS, FriendshipStatus, LANGUAGE } from 'connectfy-shared';
import { HydratedDocument } from 'mongoose';
import { t } from 'i18next';

@Schema({
  timestamps: true,
  collection: COLLECTIONS.RELATIONSHIP.FRIENDSHIPS,
  toJSON: { virtuals: true, versionKey: false, getters: true },
  toObject: { virtuals: true, versionKey: false },
  autoIndex: process.env.NODE_ENV !== 'production',
  minimize: false,
  strict: true,
  strictQuery: true,
})
export class FriendshipModel implements IFriendship {
  @Prop({ type: String, default: () => uuid(), immutable: true })
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
    immutable: true,
    trim: true,
    validate: {
      validator: function (value: string | null): boolean {
        return !!(value && validate(value));
      },
      message: t('validation_messages.uuid', {
        lng: LANGUAGE.EN,
        field: 'userId',
      }),
    },
    ref: COLLECTIONS.AUTH.USER.USERS,
  })
  userId: string;

  @Prop({
    type: String,
    required: [
      true,
      t('validation_messages.required', {
        lng: LANGUAGE.EN,
        field: 'friendId',
      }),
    ],
    index: true,
    immutable: true,
    trim: true,
    validate: {
      validator: function (value: string | null): boolean {
        return !!(value && validate(value));
      },
      message: t('validation_messages.uuid', {
        lng: LANGUAGE.EN,
        field: 'friendId',
      }),
    },
    ref: COLLECTIONS.AUTH.USER.USERS,
  })
  friendId: string;

  @Prop({
    type: String,
    enum: {
      values: Object.values(FriendshipStatus),
      message: t('validation_messages.enum', {
        lng: LANGUAGE.EN,
        field: 'status',
        values: Object.values(FriendshipStatus),
      }),
    },
    default: FriendshipStatus.Pending,
    index: true,
  })
  status: FriendshipStatus;

  @Prop({
    type: Boolean,
    default: false,
    index: true,
  })
  isFavorite: boolean;

  @Prop({
    type: Boolean,
    default: false,
    index: true,
  })
  isMuted: boolean;

  createdAt: Date;
  updatedAt: Date;
}

export const FriendshipSchema = SchemaFactory.createForClass(FriendshipModel);

FriendshipSchema.index({ userId: 1, friendId: 1 }, { unique: true });
FriendshipSchema.index({ userId: 1, status: 1 });
FriendshipSchema.index({ userId: 1, isFavorite: 1 });
FriendshipSchema.index({ userId: 1, isMuted: 1 });

export type FriendshipDocument = HydratedDocument<FriendshipModel>;

```

## File: connectfy-relationship/src/modules/friendships/friendship/publishers/notification/friendship.notification.processor.ts
```typescript
import { Processor } from '@nestjs/bullmq';
import { Logger } from '@nestjs/common';
import { BaseProcessor } from '@/src/common/base/bullmq.processor.base';
import { FRIENDSHIP_BULL_MQ_QUEUES } from '../../constants/friendship.bullmq';
import { FriendshipNotificationService } from './friendship.notification.service';
import { IFriendship } from '../../interface/friendship.interface';
import { EventEmitter2 } from '@nestjs/event-emitter';

@Processor(FRIENDSHIP_BULL_MQ_QUEUES.NOTIFICATION.QUEUE_NAME, {
  concurrency: 5,
})
export class FriendshipNotificationProcessor extends BaseProcessor<IFriendship> {
  protected readonly logger = new Logger(FriendshipNotificationProcessor.name);

  constructor(
    protected readonly notificationService: FriendshipNotificationService,
    protected readonly eventEmitter: EventEmitter2,
  ) {
    super(eventEmitter, notificationService, [
      {
        eventName: FRIENDSHIP_BULL_MQ_QUEUES.NOTIFICATION.REQUEST_SENT,
        eventHandler: (data) => this.handleRequestSentEvent(data),
      },
      {
        eventName: FRIENDSHIP_BULL_MQ_QUEUES.NOTIFICATION.REQUEST_ACCEPTED,
        eventHandler: (data) => this.handleRequestAcceptedEvent(data),
      },
    ]);
  }
}

```

## File: connectfy-relationship/src/modules/friendships/friendship/publishers/notification/friendship.notification.service.ts
```typescript
import { Injectable } from '@nestjs/common';
import { KafkaConnectionService } from '@/src/app-settings/kafka-connections/kafka-connection.service';
import {
  IFriendshipJobData,
  IFriendshipNotificationEvent,
} from '../../interface/friendship.interface';
import i18n from '@/src/i18n';
import {
  LANGUAGE,
  NotificationChannel,
  NotificationResource,
  NotificationType,
} from 'connectfy-shared';

@Injectable()
export class FriendshipNotificationService {
  constructor(
    private readonly kafkaConnectionService: KafkaConnectionService,
  ) {}

  async onRequestSent(data: IFriendshipJobData): Promise<void> {
    await this.emit(NotificationType.FRIENDSHIP_REQUEST_SENT, data);
  }

  async onRequestAccepted(data: IFriendshipJobData): Promise<void> {
    await this.emit(NotificationType.FRIENDSHIP_REQUEST_ACCEPTED, data);
  }

  private async emit(
    type: IFriendshipNotificationEvent['type'],
    data: IFriendshipJobData,
  ): Promise<void> {
    const eventPayload = {
      actorId: data.actorId,
      actorUsername: data.actorUsername,
      actorAvatar: data.actorAvatar,
      recipientId: data.recipientId,
      friendshipId: data.friendshipId,
      type,
      timestamp: new Date().toISOString(),
      title: this.generateTitle(type, data),
      body: this.generateBody(type, data),
      channel: NotificationChannel.InApp,
      resourceId: data.friendshipId,
      resourceType: NotificationResource.Friendship,
      metadata: {
        actorUsername: data.actorUsername,
        actorAvatar: data.actorAvatar,
      },
      expiresAt: null,
      _loggedUser: data._loggedUser,
      _lang: data._lang,
    };

    await this.kafkaConnectionService.emitWithContext({
      topic: 'notification.friendship.created',
      payload: eventPayload,
    });
  }

  private generateTitle(
    type: IFriendshipNotificationEvent['type'],
    data: IFriendshipJobData,
  ): {
    [LANGUAGE.EN]: string;
    [LANGUAGE.AZ]: string;
    [LANGUAGE.RU]: string;
    [LANGUAGE.TR]: string;
  } {
    const title = `notification.title.${type}`;
    return {
      [LANGUAGE.EN]: i18n.t(title, {
        lng: LANGUAGE.EN,
        username: data.actorUsername,
      }),
      [LANGUAGE.AZ]: i18n.t(title, {
        lng: LANGUAGE.AZ,
        username: data.actorUsername,
      }),
      [LANGUAGE.RU]: i18n.t(title, {
        lng: LANGUAGE.RU,
        username: data.actorUsername,
      }),
      [LANGUAGE.TR]: i18n.t(title, {
        lng: LANGUAGE.TR,
        username: data.actorUsername,
      }),
    };
  }

  private generateBody(
    type: IFriendshipNotificationEvent['type'],
    data: IFriendshipJobData,
  ): {
    [LANGUAGE.EN]: string;
    [LANGUAGE.AZ]: string;
    [LANGUAGE.RU]: string;
    [LANGUAGE.TR]: string;
  } {
    return {
      [LANGUAGE.EN]: i18n.t(`notification.body.${type}`, {
        lng: LANGUAGE.EN,
        username: data.actorUsername,
      }),
      [LANGUAGE.AZ]: i18n.t(`notification.body.${type}`, {
        lng: LANGUAGE.AZ,
        username: data.actorUsername,
      }),
      [LANGUAGE.RU]: i18n.t(`notification.body.${type}`, {
        lng: LANGUAGE.RU,
        username: data.actorUsername,
      }),
      [LANGUAGE.TR]: i18n.t(`notification.body.${type}`, {
        lng: LANGUAGE.TR,
        username: data.actorUsername,
      }),
    };
  }
}

```

## File: connectfy-relationship/src/common/constants/environment-variables.ts
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

  // Redis
  REDIS_HOST: process.env.REDIS_HOST,
  REDIS_PORT: Number(process.env.REDIS_PORT),

  // BullMQ
  BULL_HOST: process.env.BULL_HOST,
  BULL_PORT: Number(process.env.BULL_PORT),

  // TCP
  AUTH_SERVICE_HOST: process.env.AUTH_SERVICE_HOST,
  AUTH_SERVICE_PORT: Number(process.env.AUTH_SERVICE_PORT),

  MESSENGER_SERVICE_HOST: process.env.MESSENGER_SERVICE_HOST,
  MESSENGER_SERVICE_PORT: Number(process.env.MESSENGER_SERVICE_PORT),

  ACCOUNT_SERVICE_HOST: process.env.ACCOUNT_SERVICE_HOST,
  ACCOUNT_SERVICE_PORT: Number(process.env.ACCOUNT_SERVICE_PORT),

  NOTIFICATION_ACTION_HISTORY_SERVICE_HOST:
    process.env.NOTIFICATION_ACTION_HISTORY_SERVICE_HOST,
  NOTIFICATION_ACTION_HISTORY_SERVICE_PORT: Number(
    process.env.NOTIFICATION_ACTION_HISTORY_SERVICE_PORT,
  ),
};

```

## File: connectfy-relationship/src/common/base/bullmq.processor.base.ts
```typescript
import { WorkerHost } from '@nestjs/bullmq';
import { Job } from 'bullmq';
import { EventEmitter2 } from '@nestjs/event-emitter';
import { IReturnedUser, LANGUAGE } from 'connectfy-shared';
import { INotificationService } from '../interfaces/interfaces';
import { OnModuleInit } from '@nestjs/common';
import { IFriendshipJobData } from '@/src/modules/friendships/friendship/interface/friendship.interface';

interface IBaseProcessorEvents {
  eventName: string;
  eventHandler: (data: any) => void;
}

export abstract class BaseProcessor<T>
  extends WorkerHost
  implements OnModuleInit
{
  constructor(
    protected readonly eventEmitter: EventEmitter2,
    protected readonly service: INotificationService<T>,
    protected readonly procesorEvents: IBaseProcessorEvents[],
  ) {
    super();
  }

  onModuleInit() {
    this.procesorEvents.forEach((event) => {
      this.eventEmitter.on(event.eventName, event.eventHandler);
    });
  }

  async process(
    job: Job<{ payload: any; user: IReturnedUser; lang: LANGUAGE }>,
  ): Promise<any> {
    this.eventEmitter.emit(job.name, job.data);

    return { processed: true };
  }

  async handleCreateEvent(data: {
    payload: T;
    user: IReturnedUser;
    lang: LANGUAGE;
    additionalData?: any;
  }) {
    await this.service.create?.(
      data.payload,
      data.user,
      data.lang,
      data.additionalData,
    );
  }

  async handleRequestAcceptedEvent(data: {
    payload: IFriendshipJobData;
    user: IReturnedUser;
    lang: LANGUAGE;
  }) {
    await this.service.onRequestAccepted?.({
      ...data.payload,
      _loggedUser: data.user,
      _lang: data.lang,
    });
  }

  async handleRequestSentEvent(data: {
    payload: IFriendshipJobData;
    user: IReturnedUser;
    lang: LANGUAGE;
  }) {
    await this.service.onRequestSent?.({
      ...data.payload,
      _loggedUser: data.user,
      _lang: data.lang,
    });
  }
}

```

## File: connectfy-relationship/src/common/interfaces/interfaces.ts
```typescript
import { IFriendshipJobData } from '@/src/modules/friendships/friendship/interface/friendship.interface';
import { IReturnedUser, LANGUAGE } from 'connectfy-shared';

export interface IActionHistoryService<T> {
  create(
    payload: T,
    user: IReturnedUser,
    lang: LANGUAGE,
    additionalData?: Record<string, any>,
  ): Promise<void>;
}

export interface INotificationService<T> {
  create?(
    payload: T,
    user: IReturnedUser,
    lang: LANGUAGE,
    additionalData?: Record<string, any>,
  ): Promise<void>;
  onRequestAccepted?(data: IFriendshipJobData): Promise<void>;
  onRequestSent?(data: IFriendshipJobData): Promise<void>;
}

```

## File: connectfy-relationship/src/app-settings/app-settings.module.ts
```typescript
import { Global, Module } from '@nestjs/common';
import { TcpConnectionModule } from './tcp-connections/tcp-connection.module';
import { KafkaConnectionModule } from './kafka-connections/kafka-connection.module';
import { RedisModule } from './redis/redis.module';
import { CacheModule } from './cache/cache.module';
import { BullMqConnectionModule } from './bullMq-connection/bullMq-connection.module';

@Global()
@Module({
  imports: [
    TcpConnectionModule,
    KafkaConnectionModule,
    RedisModule,
    CacheModule,
    BullMqConnectionModule,
  ],
  controllers: [],
  providers: [],
  exports: [],
})
export class AppSettingsModule {}

```

## File: connectfy-relationship/src/app-settings/cache/cache.service.ts
```typescript
import { Inject, Injectable } from '@nestjs/common';
import Redis from 'ioredis';
import { EXPIRE_DATES, REDIS_KEYS } from 'connectfy-shared';

type SetOpts = { key: string; data: any; ttl?: number };

@Injectable()
export class CacheService {
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
      return JSON.parse(data) as T;
    } catch {
      return data as unknown as T;
    }
  }

  async remove(key: string) {
    await this.redis.del(key);
  }

  async removeMany(keys: string[]) {
    if (!keys?.length) return 0;
    return await this.redis.del(...keys);
  }

  async getOrSet<T = any>(
    key: string,
    loader: () => Promise<T>,
    ttlSeconds?: number,
  ): Promise<T | undefined> {
    const cached = await this.get<T>(key);
    if (cached !== undefined) return cached;

    const inflight = this.inflightRequests.get(key);
    if (inflight) return inflight;

    const promise = (async () => {
      try {
        const result = await loader();
        if (result !== undefined && result !== null) {
          await this.set({ key, data: result, ttl: ttlSeconds });
        }
        return result;
      } finally {
        this.inflightRequests.delete(key);
      }
    })();

    this.inflightRequests.set(key, promise);
    return promise;
  }
}

```

## File: connectfy-relationship/src/app-settings/cache/cache.module.ts
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

## File: connectfy-relationship/src/app-settings/kafka-connections/loggin-kafka.server.ts
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

## File: connectfy-relationship/src/app-settings/kafka-connections/kafka-connection.service.ts
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

## File: connectfy-relationship/src/app-settings/kafka-connections/kafka-connection.module.ts
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

## File: connectfy-relationship/src/app-settings/redis/redis.module.ts
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

## File: connectfy-relationship/src/app-settings/bullMq-connection/bullMq-connection.module.ts
```typescript
import { Global, Module } from '@nestjs/common';
import { BullModule } from '@nestjs/bullmq';
import { BullMqConnectionService } from './bullMq-connection.service';
import { ENVIRONMENT_VARIABLES } from 'src/common/constants/environment-variables';

@Global()
@Module({
  imports: [
    BullModule.forRoot({
      connection: {
        host: ENVIRONMENT_VARIABLES.BULL_HOST,
        port: ENVIRONMENT_VARIABLES.BULL_PORT,
      },
      defaultJobOptions: {
        removeOnComplete: true, // Uğurlu olanları sil (Yaddaşa qənaət)
        removeOnFail: 1000, // Son 1000 xətanı yadda saxla (Debug üçün lazımdır)
        attempts: 3, // Xəta olsa, 3 dəfəyə qədər təkrar yoxla
        // backoff: {              // Təkrar cəhdlər arası gözləmə vaxtı
        //   type: 'exponential',
        //   delay: 1000,          // 1san, 2san, 4san ara ilə yoxlayacaq
        // },
      },
    }),
  ],
  providers: [BullMqConnectionService],
  exports: [BullMqConnectionService],
})
export class BullMqConnectionModule {}

```

## File: connectfy-relationship/src/app-settings/bullMq-connection/bullMq-connection.service.ts
```typescript
import { Injectable } from '@nestjs/common';
import { Queue } from 'bullmq';
import { CLS_KEYS, LANGUAGE } from 'connectfy-shared';
import { ClsService } from 'nestjs-cls';

@Injectable()
export class BullMqConnectionService {
  constructor(private cls: ClsService) {}

  async add({
    queue,
    queueName,
    payload,
    additionalData,
  }: {
    queue: Queue;
    queueName: string;
    payload: Record<string, any>;
    additionalData?: Record<string, any>;
  }): Promise<void> {
    const user = this.cls.get(CLS_KEYS.USER) || undefined;
    const lang = this.cls.get(CLS_KEYS.LANG) || LANGUAGE.EN;
    await queue.add(queueName, { payload, user, lang, additionalData });
  }
}

```

## File: connectfy-relationship/src/app-settings/tcp-connections/tcp-connection.service.ts
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

    @Inject(MICROSERVICE_NAMES.TCP.AUTH)
    private readonly authService: ClientProxy,

    @Inject(MICROSERVICE_NAMES.TCP.ACCOUNT)
    private readonly accountService: ClientProxy,

    @Inject(MICROSERVICE_NAMES.TCP.MESSENGER)
    private readonly messengerService: ClientProxy,

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
  // Auth service
  // ===============================
  async auth(opts: Omit<ISendWithContextClientParams, 'client' | 'cls'>) {
    return await this.sendTcpWithContext({
      client: this.authService,
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
  // Account service
  // ===============================
  async account(opts: Omit<ISendWithContextClientParams, 'client' | 'cls'>) {
    return await this.sendTcpWithContext({
      client: this.accountService,
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

## File: connectfy-relationship/src/app-settings/tcp-connections/tcp-connection.module.ts
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
          name: MICROSERVICE_NAMES.TCP.MESSENGER,
          transport: Transport.TCP,
          options: {
            host: ENVIRONMENT_VARIABLES.MESSENGER_SERVICE_HOST,
            port: ENVIRONMENT_VARIABLES.MESSENGER_SERVICE_PORT,
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

