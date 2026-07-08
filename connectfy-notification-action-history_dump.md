# connectfy-notification-action-history Source Dump

## File: connectfy-notification-action-history/src/main.ts
```typescript
import i18n from './i18n';
import { AppModule } from './app.module';
import { NestFactory } from '@nestjs/core';
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
          clientId: 'connectfy-notification-action-history',
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
          groupId: 'consumer-connectfy-notification-action-history',
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

  // Start TCP Microservice
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

## File: connectfy-notification-action-history/src/i18n.ts
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

## File: connectfy-notification-action-history/src/app.module.ts
```typescript
import { Module } from '@nestjs/common';
import { MongooseModule } from '@nestjs/mongoose';
import { MailerModule } from '@nestjs-modules/mailer';
import { ENVIRONMENT_VARIABLES } from './common/constants/environment-variables';
import { ModulesModule } from './modules/modules.module';
import { AppSettingsModule } from './app-settings/app-settings.module';
import { APP_INTERCEPTOR } from '@nestjs/core';
import { ClsInterceptor, ClsModule } from 'nestjs-cls';
import { LoggedUserInterceptor } from './interceptors/logged-user.interceptor';
import { ProjectionsModule } from './projection/projections.module';

@Module({
  imports: [
    MongooseModule.forRoot(ENVIRONMENT_VARIABLES.MONGO_URI, {
      dbName: ENVIRONMENT_VARIABLES.DB_NAME,
    }),
    ClsModule.forRoot({
      global: true,
      interceptor: { mount: false },
    }),
    MailerModule.forRoot({
      transport: {
        host: ENVIRONMENT_VARIABLES.EMAIL_HOST,
        secure: ENVIRONMENT_VARIABLES.NODE_ENV === 'production',
        auth: {
          user: ENVIRONMENT_VARIABLES.EMAIL_USER,
          pass: ENVIRONMENT_VARIABLES.EMAIL_PASS,
        },
      },
    }),

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

## File: connectfy-notification-action-history/src/projection/projections.module.ts
```typescript
import { Module } from '@nestjs/common';
import { ProjectionUserModule } from './user/user.module';

@Module({
  imports: [ProjectionUserModule],
})
export class ProjectionsModule {}

```

## File: connectfy-notification-action-history/src/projection/user/user.controller.ts
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

## File: connectfy-notification-action-history/src/projection/user/user.module.ts
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

## File: connectfy-notification-action-history/src/projection/user/user.service.ts
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

  async update(data: EditUserDto) {
    await this.repo.update({ _id: data._id }, data);
  }

  async remove(data: RemoveUserDto) {
    await this.repo.remove(data);
  }
}

```

## File: connectfy-notification-action-history/src/projection/user/dto/find.user.dto.ts
```typescript
import { BaseFindDto } from 'connectfy-shared';

export class FindUserDto extends BaseFindDto {}

```

## File: connectfy-notification-action-history/src/projection/user/dto/edit.user.dto.ts
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

## File: connectfy-notification-action-history/src/projection/user/dto/remove.user.dto.ts
```typescript
import { BaseRemoveDto } from 'connectfy-shared';

export class RemoveUserDto extends BaseRemoveDto {}

```

## File: connectfy-notification-action-history/src/projection/user/dto/base.user.dto.ts
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

## File: connectfy-notification-action-history/src/projection/user/dto/add.user.dto.ts
```typescript
import { BaseUserDto } from './base.user.dto';

export class AddUserDto extends BaseUserDto {}

```

## File: connectfy-notification-action-history/src/projection/user/dto/nested/phoneNumber.dto.ts
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

## File: connectfy-notification-action-history/src/projection/user/interface/user.interface.ts
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

## File: connectfy-notification-action-history/src/projection/user/interface/nested/phoneNumber.interface.ts
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

## File: connectfy-notification-action-history/src/projection/user/repo/user.repo.ts
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

## File: connectfy-notification-action-history/src/projection/user/entity/user.entity.ts
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

## File: connectfy-notification-action-history/src/projection/user/entity/nested/phoneNumber.entity.ts
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

## File: connectfy-notification-action-history/src/interceptors/logged-user.interceptor.ts
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

## File: connectfy-notification-action-history/src/modules/modules.module.ts
```typescript
import { Module } from '@nestjs/common';
import { EmailModule } from './email/email.module';
import { NotificationModule } from './notification/notification.module';

@Module({
  imports: [EmailModule, NotificationModule],
  exports: [],
  providers: [],
  controllers: [],
})
export class ModulesModule {}

```

## File: connectfy-notification-action-history/src/modules/notification/notification.service.ts
```typescript
import { Injectable } from '@nestjs/common';
import { NotificationRepository } from './repo/notification.repo';
import { INotification } from './interface/notification.interface';
import { KafkaConnectionService } from '@/src/app-settings/kafka-connections/kafka-connection.service';
import { CreateNotificationDto } from './dto/create-notification.dto';
import {
  CLS_KEYS,
  IFindAllResponse,
  IReturnedUser,
  NotificationStatus,
} from 'connectfy-shared';
import { FindNotificationsDto } from './dto/find.notificaitons';
import { ClsService } from 'nestjs-cls';
import {
  MarkAllNotificationDto,
  MarkReadNotificationDto,
  MarkUnreadNotificationDto,
} from './dto/mark-read.notification.dto';
import { UpdateFriendshipNotificationMetadataDto } from './dto/update-friendship.notification.dto';
import {
  RemoveAllNotificationDto,
  RemoveNotificationDto,
} from './dto/remove.notification.dto';
import { NotificationType } from 'connectfy-shared';

@Injectable()
export class NotificationService {
  constructor(
    private readonly cls: ClsService,
    private readonly repo: NotificationRepository,
    private readonly kafkaConnectionService: KafkaConnectionService,
  ) {}

  async handleNotificationEvent(event: CreateNotificationDto): Promise<void> {
    let notification;
    if (event.type === NotificationType.FRIENDSHIP_REQUEST_SENT && event.resourceId) {
      notification = await this.repo.upsertFriendshipNotification(event);
    } else {
      notification = await this.repo.create(
        {
          recipientId: event.recipientId,
          actorId: event.actorId,
          type: event.type,
          title: event.title,
          body: event.body,
          status: NotificationStatus.Unread,
          channel: event.channel,
          resourceId: event.resourceId,
          resourceType: event.resourceType,
          metadata: event.metadata,
          expiresAt: event.expiresAt,
        },
        {
          populate: [
            {
              path: 'actorId',
              select: 'firstName lastName username avatar',
            },
          ],
        },
      );
    }

    this.kafkaConnectionService.emitWithContext({
      topic: 'notification.created',
      payload: notification,
    });
  }

  async getNotifications(
    data: FindNotificationsDto,
  ): Promise<IFindAllResponse<INotification>> {
    const user = this.cls.get<IReturnedUser>(CLS_KEYS.USER);
    const { skip, limit, status, type } = data;

    const query: Record<string, any> = { recipientId: user._id };

    if (status) query.status = status;
    if (type) query.type = type;

    const [res, count] = await Promise.all([
      this.repo.findMany({
        query,
        skip,
        limit,
        sort: { createdAt: -1 },
        populate: [
          {
            path: 'actorId',
            select: 'firstName lastName username avatar',
          },
        ],
      }),
      this.repo.count(query),
    ]);

    return {
      data: res,
      totalCount: count,
      totalPages: Math.ceil(count / limit),
      currentPage: Math.floor(skip / limit) + 1,
      limit,
    };
  }

  async markAsRead(
    data: MarkReadNotificationDto,
  ): Promise<INotification | null> {
    const user = this.cls.get<IReturnedUser>(CLS_KEYS.USER);
    return await this.repo.markAsRead(data._id, user._id);
  }

  async markAllAsRead(data: MarkAllNotificationDto): Promise<void> {
    const user = this.cls.get<IReturnedUser>(CLS_KEYS.USER);
    await this.repo.markAllAsRead(user._id, data._ids);
  }

  async markAsUnread(
    data: MarkUnreadNotificationDto,
  ): Promise<INotification | null> {
    const user = this.cls.get<IReturnedUser>(CLS_KEYS.USER);
    return await this.repo.markAsUnread(data._id, user._id);
  }

  async countUnread(): Promise<number> {
    const user = this.cls.get<IReturnedUser>(CLS_KEYS.USER);
    const count = await this.repo.countUnread(user._id);
    return count;
  }

  async updateFriendshipNotificationMetadata(
    data: UpdateFriendshipNotificationMetadataDto,
  ): Promise<INotification | null> {
    const query: Record<string, any> = {
      resourceId: data.friendshipId,
      type: NotificationType.FRIENDSHIP_REQUEST_SENT,
    };

    if (data.actorId) {
      query.actorId = data.actorId;
    }

    let newStatus = NotificationStatus.Read;
    if (data.isAccepted) newStatus = NotificationStatus.Accepted;
    else if (data.isDeclined) newStatus = NotificationStatus.Declined;
    else newStatus = NotificationStatus.Cancelled;

    await this.repo.updateMany(query, {
      $set: {
        status: newStatus,
        resolvedAt: new Date(),
      },
    });

    return null;
  }

  async removeNotification(
    data: RemoveNotificationDto,
  ): Promise<{ success: true }> {
    const user = this.cls.get<IReturnedUser>(CLS_KEYS.USER);
    await this.repo.removeByRecipientId(data, user._id);
    return { success: true };
  }

  async removeAllNotification(
    data: RemoveAllNotificationDto,
  ): Promise<{ deletedCount: number }> {
    const user = this.cls.get<IReturnedUser>(CLS_KEYS.USER);
    const [count] = await Promise.all([
      this.repo.count({ recipientId: user._id }),
      this.repo.removeAllNotification(data, user._id),
    ]);

    return { deletedCount: count };
  }
}

```

## File: connectfy-notification-action-history/src/modules/notification/notification.module.ts
```typescript
import { Module } from '@nestjs/common';
import { MongooseModule } from '@nestjs/mongoose';
import { NotificationSchema } from './entity/notification.entity';
import { NotificationController } from './notification.controller';
import { NotificationRepository } from './repo/notification.repo';
import { NotificationService } from './notification.service';
import { COLLECTIONS } from 'connectfy-shared';

@Module({
  imports: [
    MongooseModule.forFeature([
      {
        name: COLLECTIONS.NOTIFICATION.NOTIFICATIONS,
        schema: NotificationSchema,
      },
    ]),
  ],
  controllers: [NotificationController],
  providers: [NotificationRepository, NotificationService],
  exports: [NotificationRepository, NotificationService],
})
export class NotificationModule {}

```

## File: connectfy-notification-action-history/src/modules/notification/notification.controller.ts
```typescript
import { Controller } from '@nestjs/common';
import {
  Ctx,
  EventPattern,
  KafkaContext,
  MessagePattern,
  Payload,
  Transport,
} from '@nestjs/microservices';
import { NotificationService } from './notification.service';
import { commitKafkaOffset } from 'connectfy-shared';
import { CreateNotificationDto } from './dto/create-notification.dto';
import { FindNotificationsDto } from './dto/find.notificaitons';
import {
  MarkAllNotificationDto,
  MarkReadNotificationDto,
  MarkUnreadNotificationDto,
} from './dto/mark-read.notification.dto';
import { UpdateFriendshipNotificationMetadataDto } from './dto/update-friendship.notification.dto';
import {
  RemoveAllNotificationDto,
  RemoveNotificationDto,
} from './dto/remove.notification.dto';

@Controller('')
export class NotificationController {
  constructor(private readonly service: NotificationService) {}

  @MessagePattern('notification/all', Transport.TCP)
  async getNotifications(@Payload() data: FindNotificationsDto) {
    return await this.service.getNotifications(data);
  }

  @MessagePattern('notification/markRead', Transport.TCP)
  async markAsRead(@Payload() data: MarkReadNotificationDto) {
    return await this.service.markAsRead(data);
  }

  @MessagePattern('notification/markAllRead', Transport.TCP)
  async markAllAsRead(@Payload() data: MarkAllNotificationDto) {
    await this.service.markAllAsRead(data);
    return { success: true };
  }

  @EventPattern('notification/markUnread', Transport.TCP)
  async markAsUnread(@Payload() data: MarkUnreadNotificationDto) {
    return await this.service.markAsUnread(data);
  }

  @MessagePattern('notification/countUnread', Transport.TCP)
  async countUnread() {
    return await this.service.countUnread();
  }

  @MessagePattern('notification/friendship/metadata', Transport.TCP)
  async updateFriendshipNotificationMetadata(
    @Payload() data: UpdateFriendshipNotificationMetadataDto,
  ) {
    return await this.service.updateFriendshipNotificationMetadata(data);
  }

  @MessagePattern('notification/remove', Transport.TCP)
  async removeNotification(@Payload() data: RemoveNotificationDto) {
    return await this.service.removeNotification(data);
  }

  @MessagePattern('notification/removeAll', Transport.TCP)
  async removeAllNotification(@Payload() data: RemoveAllNotificationDto) {
    return await this.service.removeAllNotification(data);
  }

  @EventPattern('notification.friendship.created', Transport.KAFKA)
  async handleFriendshipEvent(
    @Payload() data: CreateNotificationDto,
    @Ctx() context: KafkaContext,
  ): Promise<void> {
    await Promise.all([
      this.service.handleNotificationEvent(data),
      commitKafkaOffset(context),
    ]);
  }
}

```

## File: connectfy-notification-action-history/src/modules/notification/dto/remove.notification.dto.ts
```typescript
import { BaseRemoveAllDto, BaseRemoveDto } from 'connectfy-shared';

export class RemoveNotificationDto extends BaseRemoveDto {}

export class RemoveAllNotificationDto extends BaseRemoveAllDto {}

```

## File: connectfy-notification-action-history/src/modules/notification/dto/create-notification.dto.ts
```typescript
import {
  NotificationStatus,
  NotificationType,
  FieldValidator,
  FIELD_TYPE,
  NotificationChannel,
  LANGUAGE,
} from 'connectfy-shared';
import { INotificationMessage } from '../interface/notification.interface';

export class CreateNotificationDto {
  @FieldValidator({ type: FIELD_TYPE.UUID })
  recipientId: string;

  @FieldValidator({ type: FIELD_TYPE.UUID, isOptional: true })
  actorId: string | null;

  @FieldValidator({ type: FIELD_TYPE.ENUM, enumObject: NotificationType })
  type: NotificationType;

  @FieldValidator({ type: FIELD_TYPE.OBJECT, isOptional: true })
  title: INotificationMessage | null;

  @FieldValidator({ type: FIELD_TYPE.OBJECT, isOptional: true })
  body: INotificationMessage | null;

  @FieldValidator({
    type: FIELD_TYPE.ENUM,
    enumObject: NotificationStatus,
    isOptional: true,
  })
  status?: NotificationStatus;

  @FieldValidator({ type: FIELD_TYPE.ENUM, enumObject: NotificationChannel })
  channel: NotificationChannel;

  @FieldValidator({ type: FIELD_TYPE.UUID, isOptional: true })
  resourceId: string | null;

  @FieldValidator({ type: FIELD_TYPE.STRING, isOptional: true })
  resourceType: string | null;

  @FieldValidator({ type: FIELD_TYPE.OBJECT, isOptional: true })
  metadata: Record<string, unknown> | null;

  @FieldValidator({ type: FIELD_TYPE.DATE, isOptional: true })
  expiresAt: Date | null;
}

```

## File: connectfy-notification-action-history/src/modules/notification/dto/find.notificaitons.ts
```typescript
import {
  FIELD_TYPE,
  FieldValidator,
  NotificationStatus,
  NotificationType,
} from 'connectfy-shared';

export class FindNotificationsDto {
  @FieldValidator({
    type: FIELD_TYPE.ENUM,
    enumObject: NotificationStatus,
    isOptional: true,
  })
  status?: NotificationStatus;

  @FieldValidator({
    type: FIELD_TYPE.ENUM,
    enumObject: NotificationType,
    isOptional: true,
  })
  type?: NotificationType;

  @FieldValidator({
    type: FIELD_TYPE.NUMBER,
  })
  skip: number;

  @FieldValidator({
    type: FIELD_TYPE.NUMBER,
  })
  limit: number;
}

```

## File: connectfy-notification-action-history/src/modules/notification/dto/mark-read.notification.dto.ts
```typescript
import { FIELD_TYPE, FieldValidator } from 'connectfy-shared';

export class MarkReadNotificationDto {
  @FieldValidator({ type: FIELD_TYPE.UUID })
  _id: string;
}

export class MarkUnreadNotificationDto {
  @FieldValidator({ type: FIELD_TYPE.UUID })
  _id: string;
}

export class MarkAllNotificationDto {
  @FieldValidator({
    type: FIELD_TYPE.ARRAY,
    arrayItemType: FIELD_TYPE.UUID,
    isOptional: true,
  })
  _ids?: string[];
}

```

## File: connectfy-notification-action-history/src/modules/notification/dto/update-friendship.notification.dto.ts
```typescript
import { FIELD_TYPE, FieldValidator } from 'connectfy-shared';

export class UpdateFriendshipNotificationMetadataDto {
  @FieldValidator({ type: FIELD_TYPE.UUID })
  recipientId: string;

  @FieldValidator({ type: FIELD_TYPE.UUID, isOptional: true })
  actorId?: string;

  @FieldValidator({ type: FIELD_TYPE.UUID })
  friendshipId: string;

  @FieldValidator({ type: FIELD_TYPE.BOOLEAN, isOptional: true })
  isAccepted?: boolean;

  @FieldValidator({ type: FIELD_TYPE.BOOLEAN, isOptional: true })
  isDeclined?: boolean;
}

```

## File: connectfy-notification-action-history/src/modules/notification/interface/notification.interface.ts
```typescript
import {
  LANGUAGE,
  NotificationChannel,
  NotificationStatus,
  NotificationType,
} from 'connectfy-shared';

export interface INotification {
  _id: string;
  recipientId: string;
  actorId: string | null;
  type: NotificationType;
  title: INotificationMessage | null;
  body: INotificationMessage | null;
  status: NotificationStatus;
  channel: NotificationChannel;
  resourceId: string | null;
  resourceType: string | null;
  metadata: Record<string, unknown> | null;
  readAt: Date | null;
  expiresAt: Date | null;
  createdAt: Date;
  updatedAt: Date;
}

export interface IFriendshipNotificationEvent {
  actorId: string;
  actorUsername: string;
  actorAvatar: string | null;
  recipientId: string;
  friendshipId: string;
  type: NotificationType;
  timestamp: string;
  title: string;
  body: string;
}

export interface INotificationMessage {
  [LANGUAGE.EN]: string;
  [LANGUAGE.AZ]: string;
  [LANGUAGE.RU]: string;
  [LANGUAGE.TR]: string;
}

```

## File: connectfy-notification-action-history/src/modules/notification/repo/notification.repo.ts
```typescript
import { Injectable } from '@nestjs/common';
import { InjectModel } from '@nestjs/mongoose';
import { Model } from 'mongoose';
import { INotification } from '../interface/notification.interface';
import { NotificationDocument } from '../entity/notification.entity';
import {
  BaseRepository,
  COLLECTIONS,
  NotificationStatus,
} from 'connectfy-shared';
import { CreateNotificationDto } from '../dto/create-notification.dto';
import {
  RemoveAllNotificationDto,
  RemoveNotificationDto,
} from '../dto/remove.notification.dto';

@Injectable()
export class NotificationRepository extends BaseRepository<
  NotificationDocument,
  INotification,
  CreateNotificationDto
> {
  constructor(
    @InjectModel(COLLECTIONS.NOTIFICATION.NOTIFICATIONS)
    protected model: Model<NotificationDocument>,
  ) {
    super(model);
  }

  async findManyByRecipient(
    recipientId: string,
    status?: NotificationStatus,
    limit = 20,
    skip = 0,
  ): Promise<INotification[]> {
    const query: Record<string, unknown> = { recipientId };

    if (status) {
      query.status = status;
    }

    const notifications = await this.model
      .find(query)
      .sort({ createdAt: -1 })
      .skip(skip)
      .limit(limit)
      .lean({ virtuals: true, versionKey: false })
      .exec();

    return notifications as INotification[];
  }

  async markAsRead(
    notificationId: string,
    recipientId: string,
  ): Promise<INotification | null> {
    const notification = await this.model
      .findOneAndUpdate(
        { _id: notificationId, recipientId },
        {
          $set: {
            status: NotificationStatus.Read,
            readAt: new Date(),
          },
        },
        { new: true },
      )
      .lean({ versionKey: false })
      .exec();

    return notification as INotification | null;
  }

  async markAsUnread(
    notificationId: string,
    recipientId: string,
  ): Promise<INotification | null> {
    const notification = await this.model
      .findOneAndUpdate(
        { _id: notificationId, recipientId },
        {
          $set: {
            status: NotificationStatus.Unread,
            readAt: null,
          },
        },
        { new: true },
      )
      .lean({ versionKey: false })
      .exec();

    return notification as INotification | null;
  }

  async markAllAsRead(recipientId: string, _ids?: string[]): Promise<void> {
    if (_ids?.length) {
      await this.model.updateMany(
        { recipientId, _id: { $in: _ids }, status: { $in: [NotificationStatus.Unread, NotificationStatus.Pending] } },
        {
          $set: {
            status: NotificationStatus.Read,
            readAt: new Date(),
          },
        },
      );
    } else {
      await this.model.updateMany(
        { recipientId, status: { $in: [NotificationStatus.Unread, NotificationStatus.Pending] } },
        {
          $set: {
            status: NotificationStatus.Read,
            readAt: new Date(),
          },
        },
      );
    }
  }

  async countUnread(recipientId: string): Promise<number> {
    return await this.model.countDocuments({
      recipientId,
      status: { $in: [NotificationStatus.Unread, NotificationStatus.Pending] },
    });
  }

  async removeByRecipientId(data: RemoveNotificationDto, recipientId: string) {
    return await this.model.deleteOne({ _id: data._id, recipientId });
  }

  async removeAllNotification(
    data: RemoveAllNotificationDto,
    recipientId: string,
  ) {
    return await this.model.deleteMany({
      _id: { $in: data._ids },
      recipientId,
    });
  }

  async upsertFriendshipNotification(event: CreateNotificationDto): Promise<INotification> {
    const { v4: uuid } = await import('uuid');
    const { NotificationType } = await import('connectfy-shared');

    const notification = await this.model.findOneAndUpdate(
      {
        recipientId: event.recipientId,
        actorId: event.actorId,
        resourceId: event.resourceId,
        type: NotificationType.FRIENDSHIP_REQUEST_SENT,
      },
      {
        $set: {
          title: event.title,
          body: event.body,
          status: NotificationStatus.Pending,
          channel: event.channel,
          resourceType: event.resourceType,
          metadata: event.metadata,
          expiresAt: event.expiresAt,
          updatedAt: new Date(),
        },
        $setOnInsert: {
          _id: uuid(),
          readAt: null,
          resolvedAt: null,
        }
      },
      { upsert: true, new: true, setDefaultsOnInsert: true }
    )
      .populate([{ path: 'actorId', select: 'firstName lastName username avatar' }])
      .lean({ virtuals: true, versionKey: false })
      .exec();

    return notification as INotification;
  }
}

```

## File: connectfy-notification-action-history/src/modules/notification/utils/migrate-notifications.ts
```typescript
import mongoose from 'mongoose';

const MONGO_URI = process.env.MONGO_URI || 'mongodb://localhost:27017/connectfy-notification-action-history';

async function migrate() {
  await mongoose.connect(MONGO_URI);
  console.log('Connected to MongoDB');

  const db = mongoose.connection.db;
  const notifications = db?.collection('notifications');

  console.log('Finding duplicates...');
  const duplicates = await notifications?.aggregate([
    { $match: { type: 'friendship_request_sent' } },
    {
      $group: {
        _id: {
          recipientId: '$recipientId',
          actorId: '$actorId',
          resourceId: '$resourceId',
        },
        count: { $sum: 1 },
        docs: { $push: { _id: '$_id', createdAt: '$createdAt' } }
      }
    },
    { $match: { count: { $gt: 1 } } }
  ]).toArray();

  let deletedCount = 0;

  for (const dup of duplicates || []) {
    // Sort by createdAt descending to keep the newest
    const sortedDocs = dup.docs.sort((a: any, b: any) => b.createdAt - a.createdAt);
    
    // Keep the first (newest), delete the rest
    const toDelete = sortedDocs.slice(1).map((doc: any) => doc._id);
    
    if (toDelete.length > 0) {
      await notifications?.deleteMany({ _id: { $in: toDelete } });
      deletedCount += toDelete.length;
    }
  }

  console.log(`Deleted ${deletedCount} duplicate FRIENDSHIP_REQUEST_SENT notifications`);

  console.log('Creating unique partial index...');
  try {
    await notifications?.createIndex(
      { recipientId: 1, actorId: 1, resourceId: 1, type: 1 },
      { 
        unique: true, 
        partialFilterExpression: { type: 'friendship_request_sent' },
        name: 'unique_friendship_request' 
      }
    );
    console.log('Index created successfully');
  } catch (error) {
    console.error('Failed to create index:', error);
  }

  await mongoose.disconnect();
  console.log('Migration complete');
}

migrate().catch(console.error);

```

## File: connectfy-notification-action-history/src/modules/notification/entity/notification.entity.ts
```typescript
import { Prop, Schema, SchemaFactory } from '@nestjs/mongoose';
import { HydratedDocument } from 'mongoose';
import { v4 as uuid } from 'uuid';
import {
  COLLECTIONS,
  LANGUAGE,
  NotificationChannel,
  NotificationStatus,
  NotificationType,
} from 'connectfy-shared';
import {
  INotification,
  INotificationMessage,
} from '../interface/notification.interface';

@Schema({
  timestamps: true,
  collection: COLLECTIONS.NOTIFICATION.NOTIFICATIONS,
  toJSON: { virtuals: true, versionKey: false },
  toObject: { virtuals: true, versionKey: false },
  autoIndex: process.env.NODE_ENV !== 'production',
  minimize: false,
})
export class NotificationModel implements INotification {
  @Prop({ type: String, default: () => uuid() })
  _id: string;

  @Prop({
    type: String,
    required: true,
    index: true,
    immutable: true,
    ref: COLLECTIONS.AUTH.USER.USERS,
  })
  recipientId: string;

  @Prop({
    type: String,
    required: false,
    default: null,
    ref: COLLECTIONS.AUTH.USER.USERS,
  })
  actorId: string | null;

  @Prop({
    type: String,
    enum: Object.values(NotificationType),
    required: true,
  })
  type: NotificationType;

  @Prop({
    type: Object,
    required: false,
    default: null,
  })
  title: INotificationMessage | null;

  @Prop({
    type: Object,
    required: false,
    default: null,
  })
  body: INotificationMessage | null;

  @Prop({
    type: String,
    enum: Object.values(NotificationStatus),
    default: NotificationStatus.Unread,
    index: true,
  })
  status: NotificationStatus;

  @Prop({
    type: String,
    enum: Object.values(NotificationChannel),
    default: NotificationChannel.InApp,
  })
  channel: NotificationChannel;

  @Prop({ type: String, default: null })
  resourceId: string | null;

  @Prop({ type: String, default: null })
  resourceType: string | null;

  @Prop({ type: Object, default: null })
  metadata: Record<string, unknown> | null;

  @Prop({ type: Date, default: null })
  readAt: Date | null;

  @Prop({ type: Date, default: null, index: { expireAfterSeconds: 0 } })
  expiresAt: Date | null;

  @Prop({ type: Date, default: null })
  resolvedAt: Date | null;

  createdAt: Date;
  updatedAt: Date;
}

export const NotificationSchema =
  SchemaFactory.createForClass(NotificationModel);

NotificationSchema.index({ recipientId: 1, status: 1, createdAt: -1 });
NotificationSchema.index(
  { recipientId: 1, actorId: 1, resourceId: 1, type: 1 },
  {
    unique: true,
    partialFilterExpression: { type: NotificationType.FRIENDSHIP_REQUEST_SENT },
  },
);

export type NotificationDocument = HydratedDocument<NotificationModel>;

```

## File: connectfy-notification-action-history/src/modules/email/email.module.ts
```typescript
import { Module } from '@nestjs/common';
import { EmailService } from './email.service';
import { EmailController } from './email.controller';

@Module({
  controllers: [EmailController],
  providers: [EmailService],
})
export class EmailModule {}

```

## File: connectfy-notification-action-history/src/modules/email/email.service.ts
```typescript
import { MailerService } from '@nestjs-modules/mailer';
import { Injectable } from '@nestjs/common';
import { SendMailDto } from './dto/send-email.dto';

@Injectable()
export class EmailService {
  constructor(private readonly mailService: MailerService) {}

  async sendMail(data: SendMailDto): Promise<void> {
    const { from, sender, to, subject, html } = data;

    await this.mailService.sendMail({ from, sender, to, subject, html });
  }
}

```

## File: connectfy-notification-action-history/src/modules/email/email.controller.ts
```typescript
import { Controller } from '@nestjs/common';
import { EmailService } from './email.service';
import {
  Ctx,
  EventPattern,
  KafkaContext,
  Payload,
  Transport,
} from '@nestjs/microservices';
import { SendMailDto } from './dto/send-email.dto';
import { commitKafkaOffset } from 'connectfy-shared';

@Controller('email')
export class EmailController {
  constructor(private readonly service: EmailService) {}

  @EventPattern('mail.send', Transport.KAFKA)
  async sendMail(@Payload() data: SendMailDto, @Ctx() context: KafkaContext) {
    await Promise.all([
      this.service.sendMail(data),
      commitKafkaOffset(context),
    ]);
  }
}

```

## File: connectfy-notification-action-history/src/modules/email/dto/send-email.dto.ts
```typescript
import { FieldValidator, FIELD_TYPE } from 'connectfy-shared';

export class SendMailDto {
  @FieldValidator({ type: FIELD_TYPE.STRING })
  from: string;

  @FieldValidator({ type: FIELD_TYPE.EMAIL })
  to: string;

  @FieldValidator({ type: FIELD_TYPE.STRING })
  sender: string;

  @FieldValidator({ type: FIELD_TYPE.STRING })
  subject: string;

  @FieldValidator({ type: FIELD_TYPE.STRING })
  html: string;
}

```

## File: connectfy-notification-action-history/src/common/constants/environment-variables.ts
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
  CLIENT_ORIGIN:
    process.env.CLIENT_ORIGIN ||
    process.env.CLIENT_URL ||
    'http://localhost:4800',

  // Database
  DB_NAME: process.env.DB_NAME,
  MONGO_URI: process.env.MONGO_URI || '',

  // Kafka
  SERVICE_NAME: process.env.SERVICE_NAME,
  BROKER1: process.env.BROKER1 || '',
  BROKER2: process.env.BROKER2 || '',
  JWT_ACCESS_SECRET: process.env.JWT_ACCESS_SECRET || '',

  // Redis
  REDIS_HOST: process.env.REDIS_HOST,
  REDIS_PORT: Number(process.env.REDIS_PORT),

  EMAIL_HOST: process.env.EMAIL_HOST,
  EMAIL_USER: process.env.EMAIL_USER,
  EMAIL_PASS: process.env.EMAIL_PASS,

  // TCP
  AUTH_SERVICE_HOST: process.env.AUTH_SERVICE_HOST,
  AUTH_SERVICE_PORT: Number(process.env.AUTH_SERVICE_PORT),

  ACCOUNT_SERVICE_HOST: process.env.ACCOUNT_SERVICE_HOST,
  ACCOUNT_SERVICE_PORT: Number(process.env.ACCOUNT_SERVICE_PORT),

  MESSENGER_SERVICE_HOST: process.env.MESSENGER_SERVICE_HOST,
  MESSENGER_SERVICE_PORT: Number(process.env.MESSENGER_SERVICE_PORT),

  RELATIONSHIP_SERVICE_HOST: process.env.RELATIONSHIP_SERVICE_HOST,
  RELATIONSHIP_SERVICE_PORT: Number(process.env.RELATIONSHIP_SERVICE_PORT),
};

```

## File: connectfy-notification-action-history/src/app-settings/app-settings.module.ts
```typescript
import { Global, Module } from '@nestjs/common';
import { TcpConnectionModule } from './tcp-connections/tcp-connection.module';
import { KafkaConnectionModule } from './kafka-connections/kafka-connection.module';
import { RedisModule } from './redis/redis.module';
import { CacheModule } from './cache/cache.module';

@Global()
@Module({
  imports: [
    TcpConnectionModule,
    KafkaConnectionModule,
    RedisModule,
    CacheModule,
  ],
  controllers: [],
  providers: [],
  exports: [
    TcpConnectionModule,
    KafkaConnectionModule,
    RedisModule,
    CacheModule,
  ],
})
export class AppSettingsModule {}

```

## File: connectfy-notification-action-history/src/app-settings/cache/cache.service.ts
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

## File: connectfy-notification-action-history/src/app-settings/cache/cache.module.ts
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

## File: connectfy-notification-action-history/src/app-settings/kafka-connections/loggin-kafka.server.ts
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

## File: connectfy-notification-action-history/src/app-settings/kafka-connections/kafka-connection.service.ts
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

## File: connectfy-notification-action-history/src/app-settings/kafka-connections/kafka-connection.module.ts
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

## File: connectfy-notification-action-history/src/app-settings/redis/redis.module.ts
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

## File: connectfy-notification-action-history/src/app-settings/tcp-connections/tcp-connection.service.ts
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

    @Inject(MICROSERVICE_NAMES.TCP.RELATIONSHIP)
    private readonly relationshipService: ClientProxy,
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
}

```

## File: connectfy-notification-action-history/src/app-settings/tcp-connections/tcp-connection.module.ts
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
      ],
      isGlobal: true,
    }),
  ],
  providers: [TcpConnectionService],
  exports: [TcpConnectionService],
})
export class TcpConnectionModule {}

```

