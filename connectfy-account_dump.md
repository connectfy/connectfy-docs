# connectfy-account Source Dump

## File: connectfy-account/src/main.ts
```typescript
import i18n from './i18n';
import { NestFactory } from '@nestjs/core';
import { AppModule } from './app.module';
import { MicroserviceOptions, Transport } from '@nestjs/microservices';
import { ValidationPipe } from '@nestjs/common';
import { AllExceptionsFilter } from 'connectfy-shared';
import { ENVIRONMENT_VARIABLES } from './common/constants/environment-variables';
import { LoggingKafkaServer } from './app-settings/kafka-connections/loggin-kafka.server';

async function bootstrap() {
  // Start Kafka Microservice
  const kafkaApp = await NestFactory.createMicroservice<MicroserviceOptions>(
    AppModule,
    {
      strategy: new LoggingKafkaServer({
        client: {
          clientId: 'connectfy-account',
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
          groupId: 'consumer-connectfy-account',
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

  // await kafkaApp.listen();
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

## File: connectfy-account/src/i18n.ts
```typescript
const i18n = require('i18next');
import { resources } from 'connectfy-i18n';

i18n.init({
  resources,
  fallbackLng: 'en',
  lng: 'en',
  interpolation: { escapeValue: false },
});

export default i18n;

```

## File: connectfy-account/src/app.module.ts
```typescript
import { Module } from '@nestjs/common';
import { MongooseModule } from '@nestjs/mongoose';
import { ClsInterceptor, ClsModule } from 'nestjs-cls';
import { APP_INTERCEPTOR } from '@nestjs/core';
import { LoggedUserInterceptor } from './interceptors/logged-user.interceptor';
import { ENVIRONMENT_VARIABLES } from './common/constants/environment-variables';
import { ModulesModule } from './modules/modules.module';
import { AppSettingsModule } from './app-settings/app-settings.module';
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

## File: connectfy-account/src/projection/projections.module.ts
```typescript
import { Module } from '@nestjs/common';
import { ProjectionUserModule } from './user/user.module';

@Module({
  imports: [ProjectionUserModule],
})
export class ProjectionsModule {}

```

## File: connectfy-account/src/projection/user/user.controller.ts
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

## File: connectfy-account/src/projection/user/user.module.ts
```typescript
import { Module } from '@nestjs/common';
import { MongooseModule } from '@nestjs/mongoose';
import { COLLECTIONS } from 'connectfy-shared';
import { UserSchema } from './entity/user.entity';
import { UserService } from './user.service';
import { UserRepository } from './repo/user.repo';
import { UserController } from './user.controller';
import { ProfileModule } from '@/src/modules/profile/profile.module';

@Module({
  imports: [
    MongooseModule.forFeature([
      { name: COLLECTIONS.AUTH.USER.USERS, schema: UserSchema },
    ]),
    ProfileModule,
  ],
  controllers: [UserController],
  providers: [UserService, UserRepository],
  exports: [UserService, UserRepository],
})
export class ProjectionUserModule {}

```

## File: connectfy-account/src/projection/user/user.service.ts
```typescript
import { Injectable } from '@nestjs/common';
import { UserRepository } from './repo/user.repo';
import { AddUserDto } from './dto/add.user.dto';
import { FindUserDto } from './dto/find.user.dto';
import { EditUserDto } from './dto/edit.user.dto';
import { RemoveUserDto } from './dto/remove.user.dto';
import { ProfileService } from '@/src/modules/profile/profile.service';

@Injectable()
export class UserService {
  constructor(
    private readonly repo: UserRepository,
    private readonly profileService: ProfileService,
  ) {}

  async create(data: AddUserDto) {
    await this.repo.create(data);
  }

  async findOne(query: FindUserDto) {
    await this.repo.findOne(query);
  }

  async findMany(query: FindUserDto) {
    await this.repo.findMany(query);
  }

  async update(data: EditUserDto) {
    const { _id, username } = data;

    if (username) {
      await this.profileService.updateUsername({ userId: _id, username });
    }

    await this.repo.update({ _id }, data);
  }

  async remove(data: RemoveUserDto) {
    await this.repo.remove(data);
  }
}

```

## File: connectfy-account/src/projection/user/dto/find.user.dto.ts
```typescript
import { BaseFindDto } from 'connectfy-shared';

export class FindUserDto extends BaseFindDto {}

```

## File: connectfy-account/src/projection/user/dto/edit.user.dto.ts
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

## File: connectfy-account/src/projection/user/dto/remove.user.dto.ts
```typescript
import { BaseRemoveDto } from 'connectfy-shared';

export class RemoveUserDto extends BaseRemoveDto {}

```

## File: connectfy-account/src/projection/user/dto/base.user.dto.ts
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

## File: connectfy-account/src/projection/user/dto/add.user.dto.ts
```typescript
import { BaseUserDto } from './base.user.dto';

export class AddUserDto extends BaseUserDto {}

```

## File: connectfy-account/src/projection/user/dto/nested/phoneNumber.dto.ts
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

## File: connectfy-account/src/projection/user/interface/user.interface.ts
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

## File: connectfy-account/src/projection/user/interface/nested/phoneNumber.interface.ts
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

## File: connectfy-account/src/projection/user/repo/user.repo.ts
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

## File: connectfy-account/src/projection/user/entity/user.entity.ts
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

## File: connectfy-account/src/projection/user/entity/nested/phoneNumber.entity.ts
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

## File: connectfy-account/src/interceptors/logged-user.interceptor.ts
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

## File: connectfy-account/src/modules/modules.module.ts
```typescript
import { Module } from '@nestjs/common';
import { ProfileModule } from './profile/profile.module';
import { SettingsModule } from './settings/settings.module';
import { SocialLinkModule } from './social-link/social-link.module';

import { MediaModule } from './media/media.module';

@Module({
  imports: [
    ProfileModule,
    SocialLinkModule,
    SettingsModule,
    MediaModule,
  ],
  controllers: [],
  providers: [],
  exports: [],
})
export class ModulesModule {}

```

## File: connectfy-account/src/modules/profile/profile.controller.ts
```typescript
import { Controller } from '@nestjs/common';
import { ProfileService } from './profile.service';
import { MessagePattern, Payload, Transport } from '@nestjs/microservices';
import { AddProfileDto } from './dto/add.profile.dto';
import {
  FindProfileByUserIdDto,
  FindProfileDto,
  SearchProfileDto,
} from './dto/find.profile.dto';
import { EditProfileDto } from './dto/edit.profile.dto';
import {
  UpdateAvatarDto,
  UpdateDefaultAvatarDto,
} from './dto/nested/avatar.dto';

@Controller('')
export class ProfileController {
  constructor(private readonly service: ProfileService) {}

  @MessagePattern('profile/create', Transport.TCP)
  async create(@Payload() data: AddProfileDto) {
    return this.service.create(data);
  }

  @MessagePattern('profile/findOne', Transport.TCP)
  async findOne(@Payload() data: FindProfileDto) {
    return this.service.findOne(data);
  }

  @MessagePattern('profile/findOneByUserId', Transport.TCP)
  async findOneByUserId(@Payload() data: FindProfileByUserIdDto) {
    return this.service.findOneByUserId(data);
  }

  @MessagePattern('profile/search', Transport.TCP)
  async findMany(@Payload() data: SearchProfileDto) {
    return this.service.findMany(data);
  }

  @MessagePattern('profile/update', Transport.TCP)
  async update(@Payload() data: EditProfileDto) {
    return this.service.update(data);
  }

  @MessagePattern('profile/updateAvatar', Transport.TCP)
  async updateAvatar(@Payload() data: UpdateAvatarDto) {
    return this.service.updateAvatar(data);
  }

  @MessagePattern('profile/updateDefaultAvatar', Transport.TCP)
  async updateDefaultAvatar(@Payload() data: UpdateDefaultAvatarDto) {
    return this.service.updateDefaultAvatar(data);
  }
}

```

## File: connectfy-account/src/modules/profile/profile.module.ts
```typescript
import { Module } from '@nestjs/common';
import { ProfileService } from './profile.service';
import { ProfileController } from './profile.controller';
import { ProfileRepository } from './repo/profile.repo';
import { MongooseModule } from '@nestjs/mongoose';
import { ProfileSchema } from './entity/profile.entity';
import { COLLECTIONS } from 'connectfy-shared';
import { PrivacySettingsModule } from '../settings/privacy-setting/privacy-settings.module';
import { MediaModule } from '../media/media.module';

@Module({
  imports: [
    MongooseModule.forFeature([
      { name: COLLECTIONS.ACCOUNT.PROFILES, schema: ProfileSchema },
    ]),
    PrivacySettingsModule,
    MediaModule,
  ],
  controllers: [ProfileController],
  providers: [ProfileService, ProfileRepository],
  exports: [ProfileService, ProfileRepository],
})
export class ProfileModule {}

```

## File: connectfy-account/src/modules/profile/profile.service.ts
```typescript
import { HttpStatus, Injectable } from '@nestjs/common';
import { ProfileRepository } from './repo/profile.repo';
import { AddProfileDto } from './dto/add.profile.dto';
import { IProfile, IReturnedProfile } from './interface/profile.interface';
import {
  FindProfileByUserIdDto,
  FindProfileDto,
  SearchProfileDto,
} from './dto/find.profile.dto';
import { ClsService } from 'nestjs-cls';
import {
  CLS_KEYS,
  LANGUAGE,
  BaseException,
  ExceptionMessages,
  IReturnedUser,
  ProfilePhotoUpdateAction,
  IAvatar,
  AvatarFormats,
  FriendshipStatus,
  PRIVACY_SETTINGS_CHOICE,
  USER_STATUS,
  shouldShowField,
} from 'connectfy-shared';
import { EditProfileDto } from './dto/edit.profile.dto';
import {
  UpdateAvatarDto,
  UpdateDefaultAvatarDto,
} from './dto/nested/avatar.dto';
import { KafkaConnectionService } from '@/src/app-settings/kafka-connections/kafka-connection.service';
import { IDefaultAvatar } from './interface/nested/avatar.interface';
import { PrivacySettingsService } from '../settings/privacy-setting/privacy-settings.service';
import { TcpConnectionService } from '@/src/app-settings/tcp-connections/tcp-connection.service';
import { IPrivacySettings } from '../settings/privacy-setting/interface/privacy-settings.interface';
import { MediaService } from '../media/media.service';
import { ENVIRONMENT_VARIABLES } from '@/src/common/constants/environment-variables';
import { HttpConnectionService } from '@/src/app-settings/http-connections/http-connection.service';

@Injectable()
export class ProfileService {
  constructor(
    private readonly cls: ClsService,
    private readonly repo: ProfileRepository,
    private readonly tcpConnectionService: TcpConnectionService,
    private readonly kafkaConnectionService: KafkaConnectionService,
    private readonly privacySettingsService: PrivacySettingsService,
    private readonly mediaService: MediaService,
    private readonly httpConnectionService: HttpConnectionService,
  ) {}

  // =================================
  // SET DEFAULT USER AVATAR
  // =================================
  private setAvatar(username: string): {
    format: AvatarFormats;
    seed: string;
    url: string;
  } {
    const formats = Object.values(AvatarFormats);
    const randomFormat = formats[Math.floor(Math.random() * formats.length)];

    return {
      format: randomFormat,
      seed: username,
      url: `https://api.dicebear.com/9.x/${randomFormat}/svg?seed=${encodeURIComponent(username)}`,
    };
  }

  // ======================
  // Create
  // ======================
  async create(data: AddProfileDto): Promise<IProfile> {
    const language = this.cls.get<LANGUAGE>(CLS_KEYS.LANG);
    const isProfileExist = await this.repo.existsByField({
      userId: data.userId,
    });

    if (isProfileExist) {
      throw new BaseException(
        ExceptionMessages.ALREADY_EXISTS_MESSAGE('profile', language),
        HttpStatus.CONFLICT,
      );
    }

    const generatedDicebear = this.setAvatar(data.username);

    const res = await this.repo.create({
      ...data,
      fullName: `${data.firstName} ${data.lastName}`,
      defaultAvatar: generatedDicebear,
      avatar: {
        key: null,
        url: generatedDicebear.url,
        isCustom: false,
      },
    });

    this.kafkaConnectionService.emitWithContext({
      topic: 'projection.user.updated',
      payload: {
        _id: data.userId,
        avatar: generatedDicebear.url,
      },
    });

    return res;
  }

  // ======================
  // Find one
  // ======================
  async findOne(data: FindProfileDto): Promise<IReturnedProfile> {
    const language = this.cls.get<LANGUAGE>(CLS_KEYS.LANG);

    const res = await this.repo.findOne(data);

    if (!res) {
      throw new BaseException(
        ExceptionMessages.NOT_FOUND_MESSAGE(language),
        HttpStatus.NOT_FOUND,
      );
    }

    return res;
  }

  // ======================
  // Find one by user id
  // ======================
  async findOneByUserId(data: FindProfileByUserIdDto) {
    const { _id: currentUserId } = this.cls.get<IReturnedUser>(CLS_KEYS.USER);
    const { userId } = data;

    // 1. Edge Case: İstifadəçi öz profilinə baxırsa?
    const isOwnProfile = currentUserId.toString() === userId.toString();

    let isBlocked = false;
    let hasBlocked = false;

    if (!isOwnProfile) {
      const blockInfo = await this.tcpConnectionService.relationship({
        endpoint: 'blocklist/isExist',
        payload: { currentUserId, targetUserId: userId },
      });

      if (blockInfo?.isBlocked) {
        return {
          user: {
            _id: userId,
          },
          actions: {
            isBlocked: blockInfo.isBlocked,
            hasBlocked: blockInfo.hasBlocked,
          },
        };
      }
    }

    // 2. Profil məlumatının gətirilməsi (User populate edilərək)
    const res = await this.findOne({
      query: { userId },
      populate: [
        {
          path: 'userId', // Bura populate olan User modelidir
          match: { status: USER_STATUS.ACTIVE },
          select: 'username email phoneNumber createdAt',
        },
      ],
      fields: '-defaultAvatar -updatedAt',
    });

    // Məlumatları parçalayırıq (Mongoose/TypeORM asılı olaraq userId obyekti verir)
    const { userId: populatedUser, ...rawProfile } = res;
    const user = populatedUser as unknown as IReturnedUser;

    // 3. Paralel məlumat çəkilişi (Əgər öz profili deyilsə)
    let friendship;
    let count = 0;
    let privacySettings: IPrivacySettings | null = null;

    if (!isOwnProfile) {
      const [relRes, privRes] = await Promise.all([
        this.tcpConnectionService.relationship({
          endpoint: 'friendship/findOneInternalWithCount',
          payload: {
            currentUserId,
            targetUserId: userId,
            countQuery: {
              $and: [{ userId }, { status: FriendshipStatus.Accepted }],
            },
            fields: '-createdAt -updatedAt -__v',
          },
        }),
        this.privacySettingsService.findOne({
          query: { userId },
          fields: '-userId -socialLinks -readReceipts -createdAt -updatedAt',
        }),
      ]);

      friendship = relRes?.data;
      count = relRes?.count || 0;
      privacySettings = privRes;
    }

    const isAcceptedFriend = friendship?.status === FriendshipStatus.Accepted;

    // 4. Qısa yol (Helper) - Hər field üçün eyni obyekti təkrar yazmamaq üçün
    const checkField = (setting?: PRIVACY_SETTINGS_CHOICE) =>
      shouldShowField(setting, isAcceptedFriend, isOwnProfile);

    // 5. User məlumatlarının süzgəcdən keçirilməsi (email, phoneNumber)
    const filteredUser = {
      ...user,
      email: checkField(privacySettings?.email) ? user.email : null,
      phoneNumber: checkField(privacySettings?.phoneNumber)
        ? user.phoneNumber
        : null,
    };

    // 6. Profile məlumatlarının süzgəcdən keçirilməsi (bio, gender, location və s.)
    const filteredProfile = {
      ...rawProfile,
      bio: checkField(privacySettings?.bio) ? rawProfile.bio : null,
      gender: checkField(privacySettings?.gender) ? rawProfile.gender : null,
      location: checkField(privacySettings?.location)
        ? rawProfile.location
        : null,
      lastSeen: checkField(privacySettings?.lastSeen)
        ? rawProfile.lastSeen
        : null,
      birthdayDate: checkField(privacySettings?.birthdayDate)
        ? rawProfile.birthdayDate
        : null,
      avatar: checkField(privacySettings?.avatar) ? rawProfile.avatar : null,
    };

    // 7. Client tərəfdə UI düymələrini idarə edəcək Action Boolean-ları
    const canSendFriendRequest = isOwnProfile
      ? false
      : (privacySettings?.friendshipRequest ?? false);

    const canSendMessage = isOwnProfile
      ? false
      : checkField(privacySettings?.messageRequest);

    return {
      user: filteredUser,
      profile: filteredProfile,
      relationship: {
        friendship,
        count,
      },
      actions: {
        canSendFriendRequest,
        canSendMessage,
        isBlocked,
        hasBlocked,
      },
    };
  }

  // ======================
  // Find many
  // ======================
  async findMany(params: SearchProfileDto) {
    const { _id: currentUserId } = this.cls.get<IReturnedUser>(CLS_KEYS.USER);
    const { skip, limit, search } = params;
    const offset = (skip - 1) * limit;

    const searchRegex = new RegExp(search.trim(), 'i');

    const blockedIds = await this.tcpConnectionService.relationship({
      endpoint: 'blocklist/findBlockedUserIds',
      payload: { userId: currentUserId },
    }) || [];

    const query = {
      $and: [
        {
          $or: [{ username: searchRegex }, { fullName: searchRegex }],
        },
        { userId: { $nin: [currentUserId, ...blockedIds] } },
      ],
    };

    const profiles = (await this.repo.findMany({
      query,
      skip: offset,
      limit,
      fields: '_id userId firstName lastName username avatar',
    })) as IReturnedProfile[];

    if (profiles.length === 0) return { data: [], totalCount: 0 };

    const targetUserIds = profiles.map((p) => p.userId.toString());

    const [privacySettings, friendships] = await Promise.all([
      this.privacySettingsService.findMany({
        query: { userId: { $in: targetUserIds } },
        fields: 'userId friendshipRequest avatar',
      }),
      this.tcpConnectionService.relationship({
        endpoint: 'friendship/findManyInternal',
        payload: {
          currentUserId,
          targetUserIds,
          fields: '_id status friendId userId',
        },
      }),
    ]);

    const privacyMap = new Map(
      privacySettings.map((ps) => [ps.userId.toString(), ps]),
    );

    const friendshipMap = new Map<
      string,
      {
        _id: string;
        status: FriendshipStatus;
        userId: string;
        friendId: string;
      }
    >(
      friendships.map((f: any) => {
        const targetId =
          f.userId.toString() === currentUserId.toString()
            ? f.friendId.toString()
            : f.userId.toString();
        return [targetId, f];
      }),
    );

    const result = profiles.map((profile) => {
      const targetId = profile.userId.toString();
      const ps = privacyMap.get(targetId);
      const friendInfo = friendshipMap.get(targetId);
      const isAcceptedFriend = friendInfo?.status === FriendshipStatus.Accepted;
      const friendshipRequest = friendInfo
        ? false
        : (ps?.friendshipRequest ?? true);
      const avatar = shouldShowField(ps?.avatar, isAcceptedFriend, false)
        ? profile.avatar
        : null;

      return {
        ...profile,
        _id: profile.userId,
        relationship: friendInfo || null,
        friendshipRequest,
        avatar,
      };
    });

    const totalCount = result.length;

    return {
      data: result,
      totalCount,
      limit,
      currentPage: skip,
      totalPages: Math.ceil(totalCount / limit),
    };
  }

  // ========================
  // Update
  // ========================
  async update(
    data: Omit<EditProfileDto, 'avatar' | 'defaultAvatar'>,
  ): Promise<{ success: boolean }> {
    const { _id } = data;
    const { _id: userId } = this.cls.get<IReturnedUser>(CLS_KEYS.USER);
    const language = this.cls.get<LANGUAGE>(CLS_KEYS.LANG);

    const isProfileExist = await this.repo.findOne({
      query: {
        $and: [{ _id }, { userId }],
      },
    });

    if (!isProfileExist) {
      throw new BaseException(
        ExceptionMessages.NOT_FOUND_MESSAGE(language),
        HttpStatus.NOT_FOUND,
      );
    }

    if (data.firstName || data.lastName) {
      data.fullName = `${data.firstName || isProfileExist.firstName} ${data.lastName || isProfileExist.lastName}`;
    }

    await this.repo.update({ _id }, data);

    if (data.firstName || data.lastName) {
      const payload: Record<string, unknown> = { _id: userId };

      if (data.firstName) payload.firstName = data.firstName;
      if (data.lastName) payload.lastName = data.lastName;
      payload.fullName = `${payload.firstName || isProfileExist.firstName} ${payload.lastName || isProfileExist.lastName}`

      this.kafkaConnectionService.emitWithContext({
        topic: 'projection.user.updated',
        payload,
      });
    }

    return { success: true };
  }

  // ========================
  // Update Avatar
  // ========================
  async updateAvatar(
    data: UpdateAvatarDto,
  ): Promise<{ success: boolean; avatar: IAvatar | null }> {
    const { _id, action, avatar } = data;
    const { _id: userId } = this.cls.get<IReturnedUser>(CLS_KEYS.USER);
    const language = this.cls.get<LANGUAGE>(CLS_KEYS.LANG);

    const profile = (await this.repo.findOne({
      query: { $and: [{ _id }, { userId }] },
      fields: '_id avatar defaultAvatar',
    })) as IReturnedProfile;

    if (!profile) {
      throw new BaseException(
        ExceptionMessages.NOT_FOUND_MESSAGE(language),
        HttpStatus.NOT_FOUND,
      );
    }

    let newAvatarData: IAvatar | null = null;

    switch (action) {
      case ProfilePhotoUpdateAction.Remove:
        newAvatarData = null;
        break;

      case ProfilePhotoUpdateAction.SetDefault:
        newAvatarData = {
          key: null,
          url: profile.defaultAvatar.url,
          isCustom: false,
        };
        break;

      case ProfilePhotoUpdateAction.Update:
        if (!avatar?.key || !avatar?.url) {
          throw new BaseException(
            ExceptionMessages.BAD_REQUEST_MESSAGE(language),
            HttpStatus.BAD_REQUEST,
          );
        }

        try {
          const stat = await this.httpConnectionService.post<{ size: number, contentType: string }>({
            baseUrl: ENVIRONMENT_VARIABLES.FILE_UPLOADER_BASE_URL,
            endPoint: '/files/stat',
            payload: { key: avatar.key }
          });
          
          if (stat.size > 5 * 1024 * 1024) {
            throw new BaseException('File size exceeds 5MB limit', HttpStatus.BAD_REQUEST);
          }
          if (!stat.contentType.startsWith('image/')) {
            throw new BaseException('Invalid file type', HttpStatus.BAD_REQUEST);
          }

          await this.mediaService.create({
            userId,
            key: avatar.key,
            url: avatar.url,
            mimeType: stat.contentType,
            size: stat.size,
            moduleName: 'avatar',
          });
        } catch (err) {
           const message = err instanceof BaseException ? err.message : 'Invalid avatar file metadata';
           throw new BaseException(message, HttpStatus.BAD_REQUEST);
        }

        newAvatarData = {
          key: avatar.key,
          url: avatar.url,
          isCustom: true,
        };
        break;

      default:
        throw new BaseException(
          ExceptionMessages.BAD_REQUEST_MESSAGE(language),
          HttpStatus.BAD_REQUEST,
        );
    }

    if (JSON.stringify(profile.avatar) === JSON.stringify(newAvatarData)) {
      return { success: true, avatar: newAvatarData };
    }
    try {
      await this.repo.update(
        { _id: profile._id },
        { _id: profile._id, avatar: newAvatarData },
      );

      if (profile.avatar?.isCustom && profile.avatar.key) {
        this.kafkaConnectionService.emitWithContext({
          topic: 'profile.file.delete',
          payload: { fileKey: profile.avatar.key },
        });
      }

      this.kafkaConnectionService.emitWithContext({
        topic: 'projection.user.updated',
        payload: {
          _id: userId,
          avatar: newAvatarData?.url,
        },
      });

      return { success: true, avatar: newAvatarData };
    } catch (error) {
      // ❌ UĞURSUZ (Rollback): DB-yə yaza bilmədik.
      // Əgər yeni şəkil yükləmişdiksə, onu MinIO/S3-dən təcili silirik ki zibillik yaranmasın.
      if (action === ProfilePhotoUpdateAction.Update && newAvatarData?.key) {
        this.kafkaConnectionService.emitWithContext({
          topic: 'profile.file.delete',
          payload: { fileKey: newAvatarData.key },
        });
      }

      throw new BaseException(
        'Avatar could not be updated',
        HttpStatus.INTERNAL_SERVER_ERROR,
      );
    }
  }

  // ========================
  // Update Default Avatar
  // ========================
  async updateDefaultAvatar(data: UpdateDefaultAvatarDto): Promise<{
    success: boolean;
    defaultAvatar: IDefaultAvatar;
    avatar?: IAvatar;
  }> {
    const { _id, format, useDefaultAvatar } = data;
    const { _id: userId } = this.cls.get<IReturnedUser>(CLS_KEYS.USER);
    const language = this.cls.get<LANGUAGE>(CLS_KEYS.LANG);

    const profile = (await this.repo.findOne({
      query: { $and: [{ _id }, { userId }] },
      fields: '_id avatar defaultAvatar',
    })) as IReturnedProfile;

    if (!profile) {
      throw new BaseException(
        ExceptionMessages.NOT_FOUND_MESSAGE(language),
        HttpStatus.NOT_FOUND,
      );
    }

    const newDefaultAvatar = {
      format,
      seed: profile.defaultAvatar.seed,
      url: `https://api.dicebear.com/9.x/${format}/svg?seed=${encodeURIComponent(profile.defaultAvatar.seed)}`,
    };

    if (
      JSON.stringify(profile.defaultAvatar) === JSON.stringify(newDefaultAvatar)
    ) {
      return { success: true, defaultAvatar: newDefaultAvatar };
    }

    const updatePayload: EditProfileDto = {
      _id: profile._id,
      defaultAvatar: newDefaultAvatar,
    };

    if (useDefaultAvatar) {
      updatePayload.avatar = {
        key: null,
        url: newDefaultAvatar.url,
        isCustom: false,
      };

      if (profile.avatar?.isCustom && profile.avatar.key) {
        this.kafkaConnectionService.emitWithContext({
          topic: 'profile.file.delete',
          payload: { fileKey: profile.avatar.key },
        });
      }
    }

    if (profile.avatar && !profile.avatar.isCustom && !profile.avatar.key) {
      updatePayload.avatar = {
        key: null,
        url: newDefaultAvatar.url,
        isCustom: false,
      };
    }

    await this.repo.update({ _id: profile._id }, updatePayload);

    if (updatePayload.avatar) {
      this.kafkaConnectionService.emitWithContext({
        topic: 'projection.user.updated',
        payload: {
          _id: userId,
          avatar: updatePayload.avatar.url,
        },
      });
    }

    return {
      success: true,
      defaultAvatar: newDefaultAvatar,
      avatar: updatePayload.avatar as IAvatar,
    };
  }

  // ========================
  // Update Username
  // ========================
  async updateUsername(data: {
    userId: string;
    username: string;
  }): Promise<void> {
    const { userId, username } = data;
    const language = this.cls.get<LANGUAGE>(CLS_KEYS.LANG);

    const profile = await this.repo.findOne({
      query: { userId },
      fields: '_id avatar defaultAvatar',
    });

    if (!profile) {
      throw new BaseException(
        ExceptionMessages.NOT_FOUND_MESSAGE(language),
        HttpStatus.NOT_FOUND,
      );
    }

    const newDefaultAvatar = {
      format: profile.defaultAvatar.format,
      seed: username,
      url: `https://api.dicebear.com/9.x/${profile.defaultAvatar.format}/svg?seed=${encodeURIComponent(username)}`,
    };
    const updatePayload: EditProfileDto = {
      _id: profile._id,
      username,
      defaultAvatar: newDefaultAvatar,
    };

    if (profile.avatar && !profile.avatar.isCustom && !profile.avatar.key) {
      updatePayload.avatar = {
        key: null,
        url: newDefaultAvatar.url,
        isCustom: false,
      };
    }

    await this.repo.update({ _id: profile._id }, updatePayload);

    if (updatePayload.avatar) {
      this.kafkaConnectionService.emitWithContext({
        topic: 'projection.user.updated',
        payload: {
          _id: userId,
          avatar: updatePayload.avatar.url,
        },
      });
    }
  }
}

```

## File: connectfy-account/src/modules/profile/dto/find.profile.dto.ts
```typescript
import { BaseFindDto, FIELD_TYPE, FieldValidator } from 'connectfy-shared';

export class FindProfileDto extends BaseFindDto {}

export class SearchProfileDto {
  @FieldValidator({
    type: FIELD_TYPE.INT,
    classType: Number,
  })
  limit: number;

  @FieldValidator({
    type: FIELD_TYPE.INT,
    classType: Number,
  })
  skip: number;

  @FieldValidator({
    type: FIELD_TYPE.STRING,
    classType: String,
  })
  search: string;
}

export class FindProfileByUserIdDto {
  @FieldValidator({
    type: FIELD_TYPE.UUID,
    classType: String,
  })
  userId: string;
}

```

## File: connectfy-account/src/modules/profile/dto/add.profile.dto.ts
```typescript
import { BaseProfileDto } from './base.profile.dto';
import { FieldValidator, FIELD_TYPE } from 'connectfy-shared';

export class AddProfileDto extends BaseProfileDto {
  @FieldValidator({ type: FIELD_TYPE.UUID })
  userId: string;
}

```

## File: connectfy-account/src/modules/profile/dto/base.profile.dto.ts
```typescript
import { FieldValidator, FIELD_TYPE, GENDER } from 'connectfy-shared';
import { AvatarDto, DefaultAvatarDto } from './nested/avatar.dto';

export class BaseProfileDto {
  @FieldValidator({ type: FIELD_TYPE.STRING, maxLength: 50 })
  firstName: string;

  @FieldValidator({ type: FIELD_TYPE.STRING, maxLength: 50 })
  lastName: string;

  @FieldValidator({ type: FIELD_TYPE.STRING, maxLength: 101, isOptional: true })
  fullName: string;

  @FieldValidator({ type: FIELD_TYPE.STRING })
  username: string;

  @FieldValidator({ type: FIELD_TYPE.ENUM, enumObject: GENDER })
  gender: GENDER;

  @FieldValidator({ type: FIELD_TYPE.STRING, isOptional: true, maxLength: 500 })
  bio: string | null;

  @FieldValidator({ type: FIELD_TYPE.STRING, isOptional: true, maxLength: 100 })
  location: string | null;

  @FieldValidator({
    type: FIELD_TYPE.OBJECT,
    isOptional: true,
    classType: AvatarDto,
  })
  avatar: AvatarDto | null;

  @FieldValidator({
    type: FIELD_TYPE.OBJECT,
    isOptional: true,
    classType: DefaultAvatarDto,
  })
  defaultAvatar: DefaultAvatarDto;

  @FieldValidator({ type: FIELD_TYPE.DATE })
  birthdayDate: Date;
}

```

## File: connectfy-account/src/modules/profile/dto/edit.profile.dto.ts
```typescript
import { PartialType } from '@nestjs/mapped-types';
import { BaseProfileDto } from './base.profile.dto';
import { FieldValidator, FIELD_TYPE } from 'connectfy-shared';

export class EditProfileDto extends PartialType(BaseProfileDto) {
  @FieldValidator({ type: FIELD_TYPE.UUID })
  _id: string;
}

```

## File: connectfy-account/src/modules/profile/dto/remove.profile.dto.ts
```typescript
import { BaseRemoveAllDto, BaseRemoveDto } from 'connectfy-shared';

export class RemoveProfileDto extends BaseRemoveDto {}

export class RemoveAllProfilesDto extends BaseRemoveAllDto {}

```

## File: connectfy-account/src/modules/profile/dto/nested/avatar.dto.ts
```typescript
import {
  AvatarFormats,
  FIELD_TYPE,
  FieldValidator,
  ProfilePhotoUpdateAction,
} from 'connectfy-shared';

export class AvatarDto {
  @FieldValidator({ type: FIELD_TYPE.STRING, isOptional: true })
  key: string | null;

  @FieldValidator({ type: FIELD_TYPE.STRING })
  url: string;

  @FieldValidator({ type: FIELD_TYPE.BOOLEAN })
  isCustom: boolean;
}

export class UpdateAvatarDto {
  @FieldValidator({ type: FIELD_TYPE.UUID })
  _id: string;

  @FieldValidator({
    type: FIELD_TYPE.ENUM,
    enumObject: ProfilePhotoUpdateAction,
  })
  action: ProfilePhotoUpdateAction;

  @FieldValidator({
    type: FIELD_TYPE.OBJECT,
    isOptional: true,
    classType: AvatarDto,
    validateIf: (obj) => obj.action === ProfilePhotoUpdateAction.Update,
  })
  avatar: AvatarDto | null;
}

// ============================================================================
// DEFAULT AVATAR DTO
// ============================================================================

export class DefaultAvatarDto {
  @FieldValidator({ type: FIELD_TYPE.ENUM, enumObject: AvatarFormats })
  format: AvatarFormats;

  @FieldValidator({ type: FIELD_TYPE.STRING })
  seed: string;

  @FieldValidator({ type: FIELD_TYPE.STRING })
  url: string;
}

export class UpdateDefaultAvatarDto {
  @FieldValidator({ type: FIELD_TYPE.UUID })
  _id: string;

  @FieldValidator({ type: FIELD_TYPE.ENUM, enumObject: AvatarFormats })
  format: AvatarFormats;

  @FieldValidator({ type: FIELD_TYPE.BOOLEAN })
  useDefaultAvatar: boolean;
}

```

## File: connectfy-account/src/modules/profile/interface/profile.interface.ts
```typescript
import { GENDER } from 'connectfy-shared';
import { IAvatar, IDefaultAvatar } from './nested/avatar.interface';

export interface IProfile {
  _id: string;
  userId: string;
  firstName: string;
  lastName: string;
  fullName: string;
  username: string;
  gender: GENDER;
  bio: string | null;
  location: string | null;
  avatar: IAvatar | null;
  defaultAvatar: IDefaultAvatar;
  lastSeen: Date;
  birthdayDate: Date;
  createdAt: Date;
  updatedAt: Date;
}

export interface IReturnedProfile {
  _id: string;
  userId: string;
  firstName: string;
  lastName: string;
  fullName: string;
  username: string;
  gender: GENDER;
  bio: string | null;
  location: string | null;
  avatar: IAvatar | null;
  defaultAvatar: IDefaultAvatar;
  lastSeen: Date;
  birthdayDate: Date;
  createdAt: Date;
  updatedAt: Date;
}

```

## File: connectfy-account/src/modules/profile/interface/nested/avatar.interface.ts
```typescript
import { AvatarFormats } from 'connectfy-shared';

export interface IAvatar {
  key: string | null;
  url: string;
  isCustom: boolean;
}

export interface IDefaultAvatar {
  format: AvatarFormats;
  seed: string;
  url: string;
}

```

## File: connectfy-account/src/modules/profile/repo/profile.repo.ts
```typescript
import { Injectable } from '@nestjs/common';
import { ProfileDocument } from '../entity/profile.entity';
import { AddProfileDto } from '../dto/add.profile.dto';
import { EditProfileDto } from '../dto/edit.profile.dto';
import { Model } from 'mongoose';
import { InjectModel } from '@nestjs/mongoose';
import { COLLECTIONS, BaseRepository } from 'connectfy-shared';
import { IReturnedProfile } from '../interface/profile.interface';

@Injectable()
export class ProfileRepository extends BaseRepository<
  ProfileDocument,
  IReturnedProfile,
  AddProfileDto,
  EditProfileDto
> {
  constructor(
    @InjectModel(COLLECTIONS.ACCOUNT.PROFILES)
    protected model: Model<ProfileDocument>,
  ) {
    super(model);
  }
}

```

## File: connectfy-account/src/modules/profile/entity/profile.entity.ts
```typescript
import { Prop, Schema, SchemaFactory } from '@nestjs/mongoose';
import { IProfile } from '../interface/profile.interface';
import { v4 as uuid } from 'uuid';
import { HydratedDocument } from 'mongoose';
import { t } from 'i18next';
import { LANGUAGE, COLLECTIONS, GENDER } from 'connectfy-shared';
import { validate } from 'uuid';
import {
  AvatarModel,
  AvatarSchema,
  DefaultAvatarModel,
  DefaultAvatarSchema,
} from './nested/avatar.entity';

@Schema({
  timestamps: true,
  collection: COLLECTIONS.ACCOUNT.PROFILES,
  toJSON: { virtuals: true, versionKey: false, getters: true },
  toObject: { virtuals: true, versionKey: false },
  autoIndex: process.env.NODE_ENV !== 'production',
  minimize: false,
  strict: true,
  strictQuery: true,
})
export class ProfileModel implements IProfile {
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
      t('validation_messages.required', { lng: LANGUAGE.EN, field: 'userId' }),
    ],
    unique: true,
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
    enum: {
      values: Object.values(GENDER),
      message: t('validation_messages.enum', {
        lng: LANGUAGE.EN,
        field: 'gender',
        values: Object.values(GENDER),
      }),
    },
    default: GENDER.OTHER,
    index: true,
  })
  gender: GENDER;

  @Prop({
    type: String,
    default: null,
    maxlength: [
      500,
      t('validation_messages.max_lenght', {
        lng: LANGUAGE.EN,
        field: 'bio',
        length: 500,
      }),
    ],
    trim: true,
  })
  bio: string | null;

  @Prop({
    type: String,
    default: null,
    maxlength: [
      100,
      t('validation_messages.max_lenght', {
        lng: LANGUAGE.EN,
        field: 'location',
        length: 100,
      }),
    ],
    trim: true,
    index: true,
  })
  location: string | null;

  @Prop({ type: AvatarSchema, require: false, default: null })
  avatar: AvatarModel | null;

  @Prop({
    type: DefaultAvatarSchema,
    require: true,
  })
  defaultAvatar: DefaultAvatarModel;

  @Prop({
    type: Date,
    default: Date.now,
    index: true,
  })
  lastSeen: Date;

  @Prop({
    type: Date,
    required: [
      true,
      t('validation_messages.required', {
        lng: LANGUAGE.EN,
        field: 'birthday',
      }),
    ],
    validate: {
      validator: (v: Date) => {
        const age = new Date().getFullYear() - new Date(v).getFullYear();
        return age >= 13;
      },
      message: t('validation_messages.min', {
        lng: LANGUAGE.EN,
        field: 'birthdayDate',
        min: 13,
      }),
    },
  })
  birthdayDate: Date;

  createdAt: Date;
  updatedAt: Date;
}

export const ProfileSchema = SchemaFactory.createForClass(ProfileModel);

// ================================================
// INDEXES - Performance optimization
// ================================================

ProfileSchema.index({ createdAt: -1 });
ProfileSchema.index({ lastSeen: -1 });
ProfileSchema.index({ location: 1, gender: 1 });
// ProfileSchema.index(
//   { firstName: 'text', lastName: 'text', username: 'text' },
//   {
//     weights: {
//       username: 10,
//       firstName: 5,
//       lastName: 3,
//     },
//     name: 'profiles_text_search',
//   },
// );

// Sparse index (null dəyərləri skip edir - memory save)
ProfileSchema.index({ avatar: 1 }, { sparse: true });
ProfileSchema.index({ firstName: 1 });
ProfileSchema.index({ lastName: 1 });
// ProfileSchema.index({ username: 1 });

// ================================================
// MIDDLEWARE / HOOKS
// ================================================
// Pre-update hook
ProfileSchema.pre('findOneAndUpdate', function (next) {
  this.set({ lastSeen: new Date() });
  next();
});

export type ProfileDocument = HydratedDocument<ProfileModel>;

```

## File: connectfy-account/src/modules/profile/entity/nested/avatar.entity.ts
```typescript
import { Prop, Schema, SchemaFactory } from '@nestjs/mongoose';
import {
  IAvatar,
  IDefaultAvatar,
} from '../../interface/nested/avatar.interface';
import { AvatarFormats } from 'connectfy-shared';

@Schema({ _id: false, timestamps: false })
export class AvatarModel implements IAvatar {
  @Prop({ type: String, required: false, default: null })
  key: string | null;

  @Prop({ type: String, required: true })
  url: string;

  @Prop({ type: Boolean, required: true, default: false })
  isCustom: boolean;
}

export const AvatarSchema = SchemaFactory.createForClass(AvatarModel);

@Schema({ _id: false, timestamps: false })
export class DefaultAvatarModel implements IDefaultAvatar {
  @Prop({
    type: String,
    required: true,
    enum: AvatarFormats,
    default: AvatarFormats.Adventurer,
  })
  format: AvatarFormats;

  @Prop({ type: String, required: true })
  seed: string;

  @Prop({ type: String, required: true })
  url: string;
}

export const DefaultAvatarSchema =
  SchemaFactory.createForClass(DefaultAvatarModel);

```

## File: connectfy-account/src/modules/settings/settings.module.ts
```typescript
import { Module } from '@nestjs/common';
import { PrivacySettingsModule } from './privacy-setting/privacy-settings.module';
import { GeneralSettingsModule } from './general-settings/general-settings.module';
import { NotificationSettingsModule } from './notification-settings/notification-settings.module';

@Module({
  imports: [
    PrivacySettingsModule,
    GeneralSettingsModule,
    NotificationSettingsModule,
  ],
})
export class SettingsModule {}

```

## File: connectfy-account/src/modules/settings/privacy-setting/privacy-settings.service.ts
```typescript
import { HttpStatus, Injectable } from '@nestjs/common';
import { PrivacySettingRepository } from './repo/privacy-settings.repo';
import { AddPrivacySettingsto } from './dto/add.privacy-settings.dto';
import { IReturnedPrivacySettings } from './interface/privacy-settings.interface';
import {
  ExceptionMessages,
  BaseException,
  IReturnedUser,
  CLS_KEYS,
  LANGUAGE,
  IResponse,
} from 'connectfy-shared';
import { EditPrivacySettingsDto } from './dto/edit.privacy-settings.dto';
import { RemovePrivacySettingsDto } from './dto/remove.privacy-settings.dto';
import { FindPrivacySettingsDto } from './dto/find.privacy-settings.dto';
import { ClsService } from 'nestjs-cls';

@Injectable()
export class PrivacySettingsService {
  constructor(
    private readonly repo: PrivacySettingRepository,
    private readonly cls: ClsService,
  ) {}

  // ========================= GET
  async get(): Promise<IReturnedPrivacySettings> {
    const { _id: userId } = this.cls.get<IReturnedUser>(CLS_KEYS.USER);
    const lang = this.cls.get<LANGUAGE>(CLS_KEYS.LANG);

    const res = await this.repo.findOneByUserId(userId);

    if (!res) {
      throw new BaseException(
        ExceptionMessages.NOT_FOUND_MESSAGE(lang),
        HttpStatus.NOT_FOUND,
      );
    }

    return res;
  }

  // ========================= FIND ONE
  async findOne(
    data: FindPrivacySettingsDto,
  ): Promise<IReturnedPrivacySettings> {
    const language = this.cls.get<LANGUAGE>(CLS_KEYS.LANG);

    const res = await this.repo.findOne(data);

    if (!res) {
      throw new BaseException(
        ExceptionMessages.NOT_FOUND_MESSAGE(language),
        HttpStatus.NOT_FOUND,
      );
    }

    return res;
  }

  // ========================= FIND MANY
  async findMany(
    data: FindPrivacySettingsDto,
  ): Promise<IReturnedPrivacySettings[]> {
    return await this.repo.findAll(data);
  }

  // ========================= CREATE
  async create(data: AddPrivacySettingsto): Promise<IReturnedPrivacySettings> {
    const { userId } = data;
    const language = this.cls.get<LANGUAGE>(CLS_KEYS.LANG);

    const isPrivacyExist = await this.repo.findOneByUserId(userId);

    if (isPrivacyExist) {
      throw new BaseException(
        ExceptionMessages.ALREADY_EXISTS_MESSAGE('privacySettings', language),
        HttpStatus.CONFLICT,
      );
    }

    return await this.repo.create(data);
  }

  // ========================= EDIT
  async edit(data: EditPrivacySettingsDto): Promise<IResponse> {
    const { _id } = data;
    const { _id: userId } = this.cls.get<IReturnedUser>(CLS_KEYS.USER);
    const lang = this.cls.get<LANGUAGE>(CLS_KEYS.LANG);

    const isExist = await this.repo.findOne({
      query: { $and: [{ _id }, { userId }] },
    });

    if (!isExist) {
      throw new BaseException(
        ExceptionMessages.NOT_FOUND_MESSAGE(lang),
        HttpStatus.NOT_FOUND,
      );
    }

    await this.repo.update({ _id: data._id }, data);

    return { success: true };
  }

  // ========================= REMOVE
  async remove(data: RemovePrivacySettingsDto): Promise<IResponse> {
    const { _id } = data;
    const { _id: userId } = this.cls.get<IReturnedUser>(CLS_KEYS.USER);
    const lang = this.cls.get<LANGUAGE>(CLS_KEYS.LANG);

    const isExist = await this.repo.findOne({
      query: { $and: [{ _id }, { userId }] },
    });

    if (!isExist)
      throw new BaseException(
        ExceptionMessages.NOT_FOUND_MESSAGE(lang),
        HttpStatus.NOT_FOUND,
      );

    await this.repo.removeOne({ $and: [{ _id }, { userId }] });
    return { success: true };
  }
}

```

## File: connectfy-account/src/modules/settings/privacy-setting/privacy-settings.module.ts
```typescript
import { Module } from '@nestjs/common';
import { PrivacySettingsService } from './privacy-settings.service';
import { PrivacySettingsController } from './privacy-settings.controller';
import { PrivacySettingRepository } from './repo/privacy-settings.repo';
import { MongooseModule } from '@nestjs/mongoose';
import { PrivacySettingsSchema } from './entity/privacy-settings.entity';
import { COLLECTIONS } from 'connectfy-shared';

@Module({
  imports: [
    MongooseModule.forFeature([
      {
        name: COLLECTIONS.ACCOUNT.SETTINGS.PRIVACY,
        schema: PrivacySettingsSchema,
      },
    ]),
  ],
  controllers: [PrivacySettingsController],
  providers: [PrivacySettingsService, PrivacySettingRepository],
  exports: [PrivacySettingsService, PrivacySettingRepository],
})
export class PrivacySettingsModule {}

```

## File: connectfy-account/src/modules/settings/privacy-setting/privacy-settings.controller.ts
```typescript
import { Controller } from '@nestjs/common';
import { PrivacySettingsService } from './privacy-settings.service';
import {
  Ctx,
  EventPattern,
  KafkaContext,
  MessagePattern,
  Payload,
  Transport,
} from '@nestjs/microservices';
import { AddPrivacySettingsto } from './dto/add.privacy-settings.dto';
import { FindPrivacySettingsDto } from './dto/find.privacy-settings.dto';
import { EditPrivacySettingsDto } from './dto/edit.privacy-settings.dto';
import { RemovePrivacySettingsDto } from './dto/remove.privacy-settings.dto';
import { commitKafkaOffset } from 'connectfy-shared';

@Controller('')
export class PrivacySettingsController {
  constructor(private readonly service: PrivacySettingsService) {}

  @MessagePattern('privacy-settings/get', Transport.TCP)
  async get() {
    return this.service.get();
  }

  @MessagePattern('privacy-settings/findOne', Transport.TCP)
  async findOne(@Payload() data: FindPrivacySettingsDto) {
    return this.service.findOne(data);
  }

  @MessagePattern('privacy-settings/create', Transport.TCP)
  async create(@Payload() data: AddPrivacySettingsto) {
    return this.service.create(data);
  }

  @MessagePattern('privacy-settings/update', Transport.TCP)
  async edit(@Payload() data: EditPrivacySettingsDto) {
    return this.service.edit(data);
  }

  @EventPattern('account.privacy-settings.remove', Transport.KAFKA)
  async remove(
    @Payload() data: RemovePrivacySettingsDto,
    @Ctx() context: KafkaContext,
  ) {
    await Promise.all([this.service.remove(data), commitKafkaOffset(context)]);
  }
}

```

## File: connectfy-account/src/modules/settings/privacy-setting/dto/remove.privacy-settings.dto.ts
```typescript
import { BaseRemoveDto } from 'connectfy-shared';

export class RemovePrivacySettingsDto extends BaseRemoveDto {}

```

## File: connectfy-account/src/modules/settings/privacy-setting/dto/edit.privacy-settings.dto.ts
```typescript
import { PartialType } from '@nestjs/mapped-types';
import { BasePrivacySettingsDto } from './base.privacy-settings.dto';
import { FieldValidator, FIELD_TYPE } from 'connectfy-shared';

export class EditPrivacySettingsDto extends PartialType(
  BasePrivacySettingsDto,
) {
  @FieldValidator({ type: FIELD_TYPE.UUID })
  _id: string;
}

```

## File: connectfy-account/src/modules/settings/privacy-setting/dto/base.privacy-settings.dto.ts
```typescript
import {
  FieldValidator,
  FIELD_TYPE,
  PRIVACY_SETTINGS_CHOICE,
} from 'connectfy-shared';

export class BasePrivacySettingsDto {
  @FieldValidator({
    type: FIELD_TYPE.ENUM,
    enumObject: PRIVACY_SETTINGS_CHOICE,
  })
  email: PRIVACY_SETTINGS_CHOICE;

  @FieldValidator({
    type: FIELD_TYPE.ENUM,
    enumObject: PRIVACY_SETTINGS_CHOICE,
  })
  bio: PRIVACY_SETTINGS_CHOICE;

  @FieldValidator({
    type: FIELD_TYPE.ENUM,
    enumObject: PRIVACY_SETTINGS_CHOICE,
  })
  gender: PRIVACY_SETTINGS_CHOICE;

  @FieldValidator({
    type: FIELD_TYPE.ENUM,
    enumObject: PRIVACY_SETTINGS_CHOICE,
  })
  location: PRIVACY_SETTINGS_CHOICE;

  @FieldValidator({
    type: FIELD_TYPE.ENUM,
    enumObject: PRIVACY_SETTINGS_CHOICE,
  })
  socialLinks: PRIVACY_SETTINGS_CHOICE;

  @FieldValidator({
    type: FIELD_TYPE.ENUM,
    enumObject: PRIVACY_SETTINGS_CHOICE,
  })
  lastSeen: PRIVACY_SETTINGS_CHOICE;

  @FieldValidator({
    type: FIELD_TYPE.ENUM,
    enumObject: PRIVACY_SETTINGS_CHOICE,
  })
  avatar: PRIVACY_SETTINGS_CHOICE;

  @FieldValidator({
    type: FIELD_TYPE.ENUM,
    enumObject: PRIVACY_SETTINGS_CHOICE,
  })
  messageRequest: PRIVACY_SETTINGS_CHOICE;

  @FieldValidator({
    type: FIELD_TYPE.ENUM,
    enumObject: PRIVACY_SETTINGS_CHOICE,
  })
  birthdayDate: PRIVACY_SETTINGS_CHOICE;

  @FieldValidator({
    type: FIELD_TYPE.ENUM,
    enumObject: PRIVACY_SETTINGS_CHOICE,
  })
  phoneNumber: PRIVACY_SETTINGS_CHOICE;

  @FieldValidator({ type: FIELD_TYPE.BOOLEAN })
  friendshipRequest: boolean;

  @FieldValidator({ type: FIELD_TYPE.BOOLEAN })
  readReceipts: boolean;
}

```

## File: connectfy-account/src/modules/settings/privacy-setting/dto/add.privacy-settings.dto.ts
```typescript
import { FIELD_TYPE, FieldValidator } from 'connectfy-shared';

export class AddPrivacySettingsto {
  @FieldValidator({ type: FIELD_TYPE.UUID })
  userId: string;
}

```

## File: connectfy-account/src/modules/settings/privacy-setting/dto/find.privacy-settings.dto.ts
```typescript
import { BaseFindDto } from 'connectfy-shared';

export class FindPrivacySettingsDto extends BaseFindDto {}

```

## File: connectfy-account/src/modules/settings/privacy-setting/interface/privacy-settings.interface.ts
```typescript
import { PRIVACY_SETTINGS_CHOICE } from 'connectfy-shared';

export interface IPrivacySettings {
  _id: string;
  userId: string;
  email: PRIVACY_SETTINGS_CHOICE;
  bio: PRIVACY_SETTINGS_CHOICE;
  gender: PRIVACY_SETTINGS_CHOICE;
  location: PRIVACY_SETTINGS_CHOICE;
  socialLinks: PRIVACY_SETTINGS_CHOICE;
  lastSeen: PRIVACY_SETTINGS_CHOICE;
  avatar: PRIVACY_SETTINGS_CHOICE;
  messageRequest: PRIVACY_SETTINGS_CHOICE;
  birthdayDate: PRIVACY_SETTINGS_CHOICE;
  phoneNumber: PRIVACY_SETTINGS_CHOICE;
  friendshipRequest: boolean;
  readReceipts: boolean;
  createdAt: Date;
  updatedAt: Date;
}

export interface IReturnedPrivacySettings {
  _id: string;
  userId: string;
  email: PRIVACY_SETTINGS_CHOICE;
  bio: PRIVACY_SETTINGS_CHOICE;
  gender: PRIVACY_SETTINGS_CHOICE;
  location: PRIVACY_SETTINGS_CHOICE;
  socialLinks: PRIVACY_SETTINGS_CHOICE;
  lastSeen: PRIVACY_SETTINGS_CHOICE;
  avatar: PRIVACY_SETTINGS_CHOICE;
  messageRequest: PRIVACY_SETTINGS_CHOICE;
  birthdayDate: PRIVACY_SETTINGS_CHOICE;
  phoneNumber: PRIVACY_SETTINGS_CHOICE;
  friendshipRequest: boolean;
  readReceipts: boolean;
  createdAt: Date;
  updatedAt: Date;
}

```

## File: connectfy-account/src/modules/settings/privacy-setting/repo/privacy-settings.repo.ts
```typescript
import { Model } from 'mongoose';
import { Injectable } from '@nestjs/common';
import { InjectModel } from '@nestjs/mongoose';
import { PrivacySettingsDocument } from '../entity/privacy-settings.entity';
import { AddPrivacySettingsto } from '../dto/add.privacy-settings.dto';
import { EditPrivacySettingsDto } from '../dto/edit.privacy-settings.dto';
import { COLLECTIONS, BaseRepository } from 'connectfy-shared';
import { IReturnedPrivacySettings } from '@modules/settings/privacy-setting/interface/privacy-settings.interface';

@Injectable()
export class PrivacySettingRepository extends BaseRepository<
  PrivacySettingsDocument,
  IReturnedPrivacySettings,
  AddPrivacySettingsto,
  EditPrivacySettingsDto
> {
  constructor(
    @InjectModel(COLLECTIONS.ACCOUNT.SETTINGS.PRIVACY)
    protected readonly model: Model<PrivacySettingsDocument>,
  ) {
    super(model);
  }
}

```

## File: connectfy-account/src/modules/settings/privacy-setting/entity/privacy-settings.entity.ts
```typescript
import { Prop, Schema, SchemaFactory } from '@nestjs/mongoose';
import { v4 as uuid, validate } from 'uuid';
import { IPrivacySettings } from '../interface/privacy-settings.interface';
import { HydratedDocument } from 'mongoose';
import { t } from 'i18next';
import {
  LANGUAGE,
  COLLECTIONS,
  PRIVACY_SETTINGS_CHOICE,
} from 'connectfy-shared';

@Schema({
  timestamps: true,
  collection: COLLECTIONS.ACCOUNT.SETTINGS.PRIVACY,
  toJSON: { virtuals: true, versionKey: false, getters: true },
  toObject: { virtuals: true, versionKey: false },
  autoIndex: process.env.NODE_ENV !== 'production',
  minimize: false,
  strict: true,
  strictQuery: true,
})
export class PrivacySettingsModel implements IPrivacySettings {
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
  })
  userId: string;

  @Prop({
    type: String,
    enum: {
      values: Object.values(PRIVACY_SETTINGS_CHOICE),
      message: t('validation_messages.enum', {
        lng: LANGUAGE.EN,
        field: 'email',
        values: Object.values(PRIVACY_SETTINGS_CHOICE),
      }),
    },
    default: PRIVACY_SETTINGS_CHOICE.NOBODY,
    index: true,
  })
  email: PRIVACY_SETTINGS_CHOICE;

  @Prop({
    type: String,
    enum: {
      values: Object.values(PRIVACY_SETTINGS_CHOICE),
      message: t('validation_messages.enum', {
        lng: LANGUAGE.EN,
        field: 'bio',
        values: Object.values(PRIVACY_SETTINGS_CHOICE),
      }),
    },
    default: PRIVACY_SETTINGS_CHOICE.EVERYONE,
    index: true,
  })
  bio: PRIVACY_SETTINGS_CHOICE;

  @Prop({
    type: String,
    enum: {
      values: Object.values(PRIVACY_SETTINGS_CHOICE),
      message: t('validation_messages.enum', {
        lng: LANGUAGE.EN,
        field: 'gender',
        values: Object.values(PRIVACY_SETTINGS_CHOICE),
      }),
    },
    default: PRIVACY_SETTINGS_CHOICE.EVERYONE,
  })
  gender: PRIVACY_SETTINGS_CHOICE;

  @Prop({
    type: String,
    enum: {
      values: Object.values(PRIVACY_SETTINGS_CHOICE),
      message: t('validation_messages.enum', {
        lng: LANGUAGE.EN,
        field: 'location',
        values: Object.values(PRIVACY_SETTINGS_CHOICE),
      }),
    },
    default: PRIVACY_SETTINGS_CHOICE.EVERYONE,
  })
  location: PRIVACY_SETTINGS_CHOICE;

  @Prop({
    type: String,
    enum: {
      values: Object.values(PRIVACY_SETTINGS_CHOICE),
      message: t('validation_messages.enum', {
        lng: LANGUAGE.EN,
        field: 'socialLinks',
        values: Object.values(PRIVACY_SETTINGS_CHOICE),
      }),
    },
    default: PRIVACY_SETTINGS_CHOICE.EVERYONE,
  })
  socialLinks: PRIVACY_SETTINGS_CHOICE;

  @Prop({
    type: String,
    enum: {
      values: Object.values(PRIVACY_SETTINGS_CHOICE),
      message: t('validation_messages.enum', {
        lng: LANGUAGE.EN,
        field: 'lastSeen',
        values: Object.values(PRIVACY_SETTINGS_CHOICE),
      }),
    },
    default: PRIVACY_SETTINGS_CHOICE.EVERYONE,
    index: true,
  })
  lastSeen: PRIVACY_SETTINGS_CHOICE;

  @Prop({
    type: String,
    enum: {
      values: Object.values(PRIVACY_SETTINGS_CHOICE),
      message: t('validation_messages.enum', {
        lng: LANGUAGE.EN,
        field: 'avatar',
        values: Object.values(PRIVACY_SETTINGS_CHOICE),
      }),
    },
    default: PRIVACY_SETTINGS_CHOICE.EVERYONE,
    index: true,
  })
  avatar: PRIVACY_SETTINGS_CHOICE;

  @Prop({
    type: String,
    enum: {
      values: Object.values(PRIVACY_SETTINGS_CHOICE),
      message: t('validation_messages.enum', {
        lng: LANGUAGE.EN,
        field: 'messageRequest',
        values: Object.values(PRIVACY_SETTINGS_CHOICE),
      }),
    },
    default: PRIVACY_SETTINGS_CHOICE.EVERYONE,
    index: true,
  })
  messageRequest: PRIVACY_SETTINGS_CHOICE;

  @Prop({
    type: String,
    enum: {
      values: Object.values(PRIVACY_SETTINGS_CHOICE),
      message: t('validation_messages.enum', {
        lng: LANGUAGE.EN,
        field: 'birthdayDate',
        values: Object.values(PRIVACY_SETTINGS_CHOICE),
      }),
    },
    default: PRIVACY_SETTINGS_CHOICE.EVERYONE,
  })
  birthdayDate: PRIVACY_SETTINGS_CHOICE;

  @Prop({
    type: String,
    enum: {
      values: Object.values(PRIVACY_SETTINGS_CHOICE),
      message: t('validation_messages.enum', {
        lng: LANGUAGE.EN,
        field: 'phoneNumber',
        values: Object.values(PRIVACY_SETTINGS_CHOICE),
      }),
    },
    default: PRIVACY_SETTINGS_CHOICE.NOBODY,
    index: true,
  })
  phoneNumber: PRIVACY_SETTINGS_CHOICE;

  @Prop({
    type: Boolean,
    required: [
      true,
      t('validation_messages.required', {
        lng: LANGUAGE.EN,
        field: 'friendshipRequest',
      }),
    ],
    default: true,
    index: true,
  })
  friendshipRequest: boolean;

  @Prop({
    type: Boolean,
    required: [
      true,
      t('validation_messages.required', {
        lng: LANGUAGE.EN,
        field: 'readReceipts',
      }),
    ],
    default: true,
    index: true,
  })
  readReceipts: boolean;

  createdAt: Date;
  updatedAt: Date;
}

export const PrivacySettingsSchema =
  SchemaFactory.createForClass(PrivacySettingsModel);

// ================================================
// INDEXES - Performance optimization
// ================================================

// Timestamp indexes
PrivacySettingsSchema.index({ createdAt: -1 });
PrivacySettingsSchema.index({ updatedAt: -1 });

export type PrivacySettingsDocument = HydratedDocument<PrivacySettingsModel>;

```

## File: connectfy-account/src/modules/settings/notification-settings/notification-settings.module.ts
```typescript
import { Module } from '@nestjs/common';
import { NotificationSettingsService } from './notification-settings.service';
import { NotificationSettingsController } from './notification-settings.controller';
import { MongooseModule } from '@nestjs/mongoose';
import { NotificationSettingsSchema } from './entity/notification-settings.entity';
import { NotificationSettingsRepository } from './repo/notification-settings.repo';
import { COLLECTIONS } from 'connectfy-shared';

@Module({
  imports: [
    MongooseModule.forFeature([
      {
        name: COLLECTIONS.ACCOUNT.SETTINGS.NOTIFICATION,
        schema: NotificationSettingsSchema,
      },
    ]),
  ],
  controllers: [NotificationSettingsController],
  providers: [NotificationSettingsService, NotificationSettingsRepository],
  exports: [NotificationSettingsService, NotificationSettingsRepository],
})
export class NotificationSettingsModule {}

```

## File: connectfy-account/src/modules/settings/notification-settings/notification-settings.service.ts
```typescript
import { HttpStatus, Injectable } from '@nestjs/common';
import { NotificationSettingsRepository } from './repo/notification-settings.repo';
import { FindNotificationSettingsDto } from './dto/find.notification-settings.dto';
import {
  ExceptionMessages,
  BaseException,
  IReturnedUser,
  CLS_KEYS,
  LANGUAGE,
  IResponse,
} from 'connectfy-shared';
import { AddNotificationSettingsDto } from './dto/add.notification-settings.dto';
import { EditNotificationSettingsDto } from './dto/edit.notification-settings.dto';
import { RemoveNotificationSettingsDto } from './dto/remove.notification-settings.dto';
import { ClsService } from 'nestjs-cls';
import { IReturnedNotificationSettings } from '@modules/settings/notification-settings/interface/notification-settings.interface';

@Injectable()
export class NotificationSettingsService {
  constructor(
    private readonly repo: NotificationSettingsRepository,
    private readonly cls: ClsService,
  ) {}

  // ========================= GET
  async get(): Promise<IReturnedNotificationSettings> {
    const { _id: userId } = this.cls.get<IReturnedUser>(CLS_KEYS.USER);
    const lang = this.cls.get<LANGUAGE>(CLS_KEYS.LANG);

    const res = await this.repo.findOneByUserId(userId);

    if (!res)
      throw new BaseException(
        ExceptionMessages.NOT_FOUND_MESSAGE(lang),
        HttpStatus.NOT_FOUND,
      );

    return res as IReturnedNotificationSettings;
  }

  // ========================= FIND ONE
  async findOne(
    data: FindNotificationSettingsDto,
  ): Promise<IReturnedNotificationSettings> {
    const language = this.cls.get<LANGUAGE>(CLS_KEYS.LANG);

    const res = await this.repo.findOne(data);

    if (!res)
      throw new BaseException(
        ExceptionMessages.NOT_FOUND_MESSAGE(language),
        HttpStatus.NOT_FOUND,
      );

    return res as IReturnedNotificationSettings;
  }

  // ========================= CREATE
  async create(
    data: AddNotificationSettingsDto,
  ): Promise<IReturnedNotificationSettings> {
    const { userId } = data;
    const language = this.cls.get<LANGUAGE>(CLS_KEYS.LANG);

    const isExist = await this.repo.findOneByUserId(userId);

    if (isExist) {
      throw new BaseException(
        ExceptionMessages.ALREADY_EXISTS_MESSAGE('privacySettings', language),
        HttpStatus.CONFLICT,
      );
    }

    return await this.repo.create(data);
  }

  // ========================= EDIT
  async edit(data: EditNotificationSettingsDto): Promise<IResponse> {
    const { _id } = data;
    const { _id: userId } = this.cls.get<IReturnedUser>(CLS_KEYS.USER);
    const lang = this.cls.get<LANGUAGE>(CLS_KEYS.LANG);

    const isExist = await this.repo.existsByField({
      query: {
        $and: [{ _id }, { userId }],
      },
    });

    if (!isExist) {
      throw new BaseException(
        ExceptionMessages.NOT_FOUND_MESSAGE(lang),
        HttpStatus.NOT_FOUND,
      );
    }

    await this.repo.update({ _id: data._id }, data);

    return { success: true };
  }

  // ========================= REMOVE
  async remove(data: RemoveNotificationSettingsDto): Promise<IResponse> {
    const { _id } = data;
    const { _id: userId } = this.cls.get<IReturnedUser>(CLS_KEYS.USER);
    const lang = this.cls.get<LANGUAGE>(CLS_KEYS.LANG);

    const isExist = await this.repo.findOne({
      query: { $and: [{ _id }, { userId }] },
    });

    if (!isExist)
      throw new BaseException(
        ExceptionMessages.NOT_FOUND_MESSAGE(lang),
        HttpStatus.NOT_FOUND,
      );

    await this.repo.removeOne({
      $and: [{ _id }, { userId }],
    });

    return { success: true };
  }
}

```

## File: connectfy-account/src/modules/settings/notification-settings/notification-settings.controller.ts
```typescript
import { Controller } from '@nestjs/common';
import { NotificationSettingsService } from './notification-settings.service';
import {
  Ctx,
  EventPattern,
  KafkaContext,
  MessagePattern,
  Payload,
  Transport,
} from '@nestjs/microservices';
import { FindNotificationSettingsDto } from './dto/find.notification-settings.dto';
import { AddNotificationSettingsDto } from './dto/add.notification-settings.dto';
import { EditNotificationSettingsDto } from './dto/edit.notification-settings.dto';
import { RemoveNotificationSettingsDto } from './dto/remove.notification-settings.dto';
import { commitKafkaOffset } from 'connectfy-shared';

@Controller('')
export class NotificationSettingsController {
  constructor(private readonly service: NotificationSettingsService) {}

  @MessagePattern('notification-settings/get', Transport.TCP)
  async get() {
    return this.service.get();
  }

  @MessagePattern('notification-settings/findOne', Transport.TCP)
  async findOne(@Payload() data: FindNotificationSettingsDto) {
    return this.service.findOne(data);
  }

  @MessagePattern('notification-settings/create', Transport.TCP)
  async create(@Payload() data: AddNotificationSettingsDto) {
    return this.service.create(data);
  }

  @MessagePattern('notification-settings/update', Transport.TCP)
  async update(@Payload() data: EditNotificationSettingsDto) {
    return this.service.edit(data);
  }

  @EventPattern('account.notification-settings.remove', Transport.KAFKA)
  async remove(
    @Payload() data: RemoveNotificationSettingsDto,
    @Ctx() context: KafkaContext,
  ) {
    await Promise.all([this.service.remove(data), commitKafkaOffset(context)]);
  }
}

```

## File: connectfy-account/src/modules/settings/notification-settings/dto/edit.notification-settings.dto.ts
```typescript
import { PartialType } from '@nestjs/mapped-types';
import { BaseNotificationSettingsDto } from './base.notification-settings.dto';
import { FIELD_TYPE, FieldValidator } from 'connectfy-shared';

export class EditNotificationSettingsDto extends PartialType(
  BaseNotificationSettingsDto,
) {
  @FieldValidator({ type: FIELD_TYPE.UUID })
  _id: string;
}

```

## File: connectfy-account/src/modules/settings/notification-settings/dto/find.notification-settings.dto.ts
```typescript
import { BaseFindDto } from 'connectfy-shared';

export class FindNotificationSettingsDto extends BaseFindDto {}

```

## File: connectfy-account/src/modules/settings/notification-settings/dto/remove.notification-settings.dto.ts
```typescript
import { BaseRemoveDto } from 'connectfy-shared';

export class RemoveNotificationSettingsDto extends BaseRemoveDto {}

```

## File: connectfy-account/src/modules/settings/notification-settings/dto/base.notification-settings.dto.ts
```typescript
import {
  FIELD_TYPE,
  NOTIFICATION_CONTENT_MODE,
  NOTIFICATION_SOUND_MODE,
  FieldValidator,
} from 'connectfy-shared';

export class BaseNotificationSettingsDto {
  // <=================  =================>
  @FieldValidator({
    type: FIELD_TYPE.ENUM,
    enumObject: NOTIFICATION_SOUND_MODE,
  })
  notificationSoundMode: NOTIFICATION_SOUND_MODE;

  // <=================  =================>
  @FieldValidator({
    type: FIELD_TYPE.ENUM,
    enumObject: NOTIFICATION_CONTENT_MODE,
  })
  notificationContentMode: NOTIFICATION_CONTENT_MODE;

  // <=================  =================>
  @FieldValidator({ type: FIELD_TYPE.BOOLEAN })
  sendMessageSound: boolean;

  // additional boolean fields (same pattern as sendMessageSound)

  @FieldValidator({ type: FIELD_TYPE.BOOLEAN })
  receiveMessageSound: boolean;


  @FieldValidator({ type: FIELD_TYPE.BOOLEAN })
  privateMessageSound: boolean;

  @FieldValidator({ type: FIELD_TYPE.BOOLEAN })
  groupMessageSound: boolean;

  @FieldValidator({ type: FIELD_TYPE.BOOLEAN })
  systemNotificationSound: boolean;

  @FieldValidator({ type: FIELD_TYPE.BOOLEAN })
  friendshipNotificationSound: boolean;

  @FieldValidator({ type: FIELD_TYPE.BOOLEAN })
  showPrivateMessageNotification: boolean;

  @FieldValidator({ type: FIELD_TYPE.BOOLEAN })
  showGroupMessageNotification: boolean;

  @FieldValidator({ type: FIELD_TYPE.BOOLEAN })
  showFriendshipNotification: boolean;

  @FieldValidator({ type: FIELD_TYPE.BOOLEAN })
  showSystemNotification: boolean;
}

```

## File: connectfy-account/src/modules/settings/notification-settings/dto/add.notification-settings.dto.ts
```typescript
import { FIELD_TYPE, FieldValidator } from 'connectfy-shared';

export class AddNotificationSettingsDto {
  @FieldValidator({ type: FIELD_TYPE.UUID })
  userId: string;
}

```

## File: connectfy-account/src/modules/settings/notification-settings/interface/notification-settings.interface.ts
```typescript
import {
  NOTIFICATION_CONTENT_MODE,
  NOTIFICATION_SOUND_MODE,
} from 'connectfy-shared';

export interface INotificationSettings {
  _id: string;
  userId: string;
  notificationSoundMode: NOTIFICATION_SOUND_MODE;
  notificationContentMode: NOTIFICATION_CONTENT_MODE;
  sendMessageSound: boolean;
  receiveMessageSound: boolean;
  privateMessageSound: boolean;
  groupMessageSound: boolean;
  systemNotificationSound: boolean;
  friendshipNotificationSound: boolean;
  showPrivateMessageNotification: boolean;
  showGroupMessageNotification: boolean;
  showFriendshipNotification: boolean;
  showSystemNotification: boolean;
  createdAt: Date;
  updatedAt: Date;
}

export interface IReturnedNotificationSettings {
  _id: string;
  userId: string;
  notificationSoundMode: NOTIFICATION_SOUND_MODE;
  notificationContentMode: NOTIFICATION_CONTENT_MODE;
  sendMessageSound: boolean;
  receiveMessageSound: boolean;
  privateMessageSound: boolean;
  groupMessageSound: boolean;
  systemNotificationSound: boolean;
  friendshipNotificationSound: boolean;
  showNotification: boolean;
  showNotificationContent: boolean;
  showPrivateMessageNotification: boolean;
  showGroupMessageNotification: boolean;
  showFriendshipNotification: boolean;
  showSystemNotification: boolean;
  createdAt: Date;
  updatedAt: Date;
}

```

## File: connectfy-account/src/modules/settings/notification-settings/repo/notification-settings.repo.ts
```typescript
import { Injectable } from '@nestjs/common';
import { InjectModel } from '@nestjs/mongoose';
import { Model } from 'mongoose';
import { NotificationSettingsDocument } from '../entity/notification-settings.entity';
import { AddNotificationSettingsDto } from '../dto/add.notification-settings.dto';
import { EditNotificationSettingsDto } from '../dto/edit.notification-settings.dto';
import { COLLECTIONS, BaseRepository } from 'connectfy-shared';
import { IReturnedNotificationSettings } from '@modules/settings/notification-settings/interface/notification-settings.interface';

@Injectable()
export class NotificationSettingsRepository extends BaseRepository<
  NotificationSettingsDocument,
  IReturnedNotificationSettings,
  AddNotificationSettingsDto,
  EditNotificationSettingsDto
> {
  constructor(
    @InjectModel(COLLECTIONS.ACCOUNT.SETTINGS.NOTIFICATION)
    protected readonly model: Model<NotificationSettingsDocument>,
  ) {
    super(model);
  }
}

```

## File: connectfy-account/src/modules/settings/notification-settings/entity/notification-settings.entity.ts
```typescript
import { Prop, Schema, SchemaFactory } from '@nestjs/mongoose';
import { INotificationSettings } from '../interface/notification-settings.interface';
import { v4 as uuid, validate } from 'uuid';
import { HydratedDocument } from 'mongoose';
import {
  LANGUAGE,
  NOTIFICATION_CONTENT_MODE,
  NOTIFICATION_SOUND_MODE,
  COLLECTIONS,
} from 'connectfy-shared';
import { t } from 'i18next';

@Schema({
  timestamps: true,
  collection: COLLECTIONS.ACCOUNT.SETTINGS.NOTIFICATION,
  toJSON: { virtuals: true, versionKey: false, getters: true },
  toObject: { virtuals: true, versionKey: false },
  autoIndex: process.env.NODE_ENV !== 'production',
  minimize: false,
  strict: true,
  strictQuery: true,
})
export class NotificationSettingsModel implements INotificationSettings {
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
  })
  userId: string;

  @Prop({
    type: String,
    required: [
      true,
      t('validation_messages.required', {
        lng: LANGUAGE.EN,
        field: 'notificationSoundMode',
      }),
    ],
    enum: {
      values: Object.values(NOTIFICATION_SOUND_MODE),
      message: t('validation_messages.enum', {
        lng: LANGUAGE.EN,
        field: 'notificationSoundMode',
        values: Object.values(NOTIFICATION_SOUND_MODE),
      }),
    },
    default: NOTIFICATION_SOUND_MODE.SOUND,
    index: true,
  })
  notificationSoundMode: NOTIFICATION_SOUND_MODE;

  @Prop({
    type: String,
    required: [
      true,
      t('validation_messages.required', {
        lng: LANGUAGE.EN,
        field: 'notificationContentMode',
      }),
    ],
    enum: {
      values: Object.values(NOTIFICATION_CONTENT_MODE),
      message: t('validation_messages.enum', {
        lng: LANGUAGE.EN,
        field: 'notificationContentMode',
        values: Object.values(NOTIFICATION_CONTENT_MODE),
      }),
    },
    default: NOTIFICATION_CONTENT_MODE.HEADER_AND_CONTENT,
    index: true,
  })
  notificationContentMode: NOTIFICATION_CONTENT_MODE;

  @Prop({
    type: Boolean,
    required: [
      true,
      t('validation_messages.required', {
        lng: LANGUAGE.EN,
        field: 'sendMessageSound',
      }),
    ],
    default: true,
    index: true,
  })
  sendMessageSound: boolean;

  @Prop({
    type: Boolean,
    required: [
      true,
      t('validation_messages.required', {
        lng: LANGUAGE.EN,
        field: 'receiveMessageSound',
      }),
    ],
    default: true,
    index: true,
  })
  receiveMessageSound: boolean;

  @Prop({
    type: Boolean,
    required: [
      true,
      t('validation_messages.required', {
        lng: LANGUAGE.EN,
        field: 'privateMessageSound',
      }),
    ],
    default: true,
  })
  privateMessageSound: boolean;

  @Prop({
    type: Boolean,
    required: [
      true,
      t('validation_messages.required', {
        lng: LANGUAGE.EN,
        field: 'groupMessageSound',
      }),
    ],
    default: true,
  })
  groupMessageSound: boolean;

  @Prop({
    type: Boolean,
    required: [
      true,
      t('validation_messages.required', {
        lng: LANGUAGE.EN,
        field: 'systemNotificationSound',
      }),
    ],
    default: true,
  })
  systemNotificationSound: boolean;

  @Prop({
    type: Boolean,
    required: [
      true,
      t('validation_messages.required', {
        lng: LANGUAGE.EN,
        field: 'friendshipNotificationSound',
      }),
    ],
    default: true,
  })
  friendshipNotificationSound: boolean;

  @Prop({
    type: Boolean,
    required: [
      true,
      t('validation_messages.required', {
        lng: LANGUAGE.EN,
        field: 'showPrivateMessageNotification',
      }),
    ],
    default: true,
  })
  showPrivateMessageNotification: boolean;

  @Prop({
    type: Boolean,
    required: [
      true,
      t('validation_messages.required', {
        lng: LANGUAGE.EN,
        field: 'showGroupMessageNotification',
      }),
    ],
    default: true,
  })
  showGroupMessageNotification: boolean;

  @Prop({
    type: Boolean,
    required: [
      true,
      t('validation_messages.required', {
        lng: LANGUAGE.EN,
        field: 'showFriendshipNotification',
      }),
    ],
    default: true,
  })
  showFriendshipNotification: boolean;

  @Prop({
    type: Boolean,
    required: [
      true,
      t('validation_messages.required', {
        lng: LANGUAGE.EN,
        field: 'showSystemNotification',
      }),
    ],
    default: true,
  })
  showSystemNotification: boolean;

  createdAt: Date;
  updatedAt: Date;
}

export const NotificationSettingsSchema = SchemaFactory.createForClass(
  NotificationSettingsModel,
);

// ================================================
// INDEXES - Performance optimization
// ================================================

// Timestamp indexes
NotificationSettingsSchema.index({ createdAt: -1 });
NotificationSettingsSchema.index({ updatedAt: -1 });

export type NotificationSettingsDocument =
  HydratedDocument<NotificationSettingsModel>;

```

## File: connectfy-account/src/modules/settings/general-settings/general-settings.controller.ts
```typescript
import { Controller } from '@nestjs/common';
import { GeneralSettingsService } from './general-settings.service';
import {
  Ctx,
  EventPattern,
  KafkaContext,
  MessagePattern,
  Payload,
  Transport,
} from '@nestjs/microservices';
import { FindGeneralSettingsDto } from './dto/find.general-settings.dto';
import { AddGeneralSettingsDto } from './dto/add.general-settings.dto';
import { EditGenearalSettingsDto } from './dto/edit.general-settings.dto';
import { RemoveGeneralSettingsDto } from './dto/remove.general-settings.dto';
import { commitKafkaOffset } from 'connectfy-shared';

@Controller('')
export class GeneralSettingsController {
  constructor(private readonly service: GeneralSettingsService) {}

  @MessagePattern('general-settings/get', Transport.TCP)
  async get() {
    return this.service.get();
  }

  @MessagePattern('general-settings/findOne', Transport.TCP)
  async findOne(@Payload() data: FindGeneralSettingsDto) {
    return this.service.findOne(data);
  }

  @MessagePattern('general-settings/create', Transport.TCP)
  async create(@Payload() data: AddGeneralSettingsDto) {
    return this.service.create(data);
  }

  @MessagePattern('general-settings/update', Transport.TCP)
  async update(@Payload() data: EditGenearalSettingsDto) {
    return this.service.edit(data);
  }

  @MessagePattern('general-settings/reset', Transport.TCP)
  async reset() {
    return this.service.reset();
  }

  @EventPattern('account.general-settings.remove', Transport.KAFKA)
  async remove(
    @Payload() data: RemoveGeneralSettingsDto,
    @Ctx() context: KafkaContext,
  ) {
    await Promise.all([this.service.remove(data), commitKafkaOffset(context)]);
  }
}

```

## File: connectfy-account/src/modules/settings/general-settings/general-settings.service.ts
```typescript
import { HttpStatus, Injectable } from '@nestjs/common';
import { GeneralSettingsRepository } from './repo/general-settings.repo';
import { AddGeneralSettingsDto } from './dto/add.general-settings.dto';
import { EditGenearalSettingsDto } from './dto/edit.general-settings.dto';
import { RemoveGeneralSettingsDto } from './dto/remove.general-settings.dto';
import { FindGeneralSettingsDto } from './dto/find.general-settings.dto';
import { ClsService } from 'nestjs-cls';
import {
  CLS_KEYS,
  DATE_FORMAT,
  LANGUAGE,
  NOTIFICATION_CONTENT_MODE,
  NOTIFICATION_SOUND_MODE,
  PRIVACY_SETTINGS_CHOICE,
  STARTUP_PAGE,
  TIME_FORMAT,
  ExceptionMessages,
  BaseException,
  IReturnedUser,
  IResponse,
} from 'connectfy-shared';
import { PrivacySettingRepository } from '../privacy-setting/repo/privacy-settings.repo';
import { NotificationSettingsRepository } from '../notification-settings/repo/notification-settings.repo';
import { IReturnedGeneralSettings } from './interface/general-settings.interface';

@Injectable()
export class GeneralSettingsService {
  constructor(
    private readonly cls: ClsService,
    private readonly repo: GeneralSettingsRepository,
    private readonly privacySettingsRepo: PrivacySettingRepository,
    private readonly notificationSettingsRepo: NotificationSettingsRepository,
  ) {}

  // ========================= GET
  async get(): Promise<IReturnedGeneralSettings> {
    const { _id: userId } = this.cls.get<IReturnedUser>(CLS_KEYS.USER);
    const lang = this.cls.get<LANGUAGE>(CLS_KEYS.LANG);

    const res = await this.repo.findOneByUserId(userId);

    if (!res) {
      throw new BaseException(
        ExceptionMessages.NOT_FOUND_MESSAGE(lang),
        HttpStatus.NOT_FOUND,
      );
    }

    return res;
  }

  // ========================= FIND ONE
  async findOne(
    data: FindGeneralSettingsDto,
  ): Promise<IReturnedGeneralSettings> {
    const language = this.cls.get<LANGUAGE>(CLS_KEYS.LANG);

    const res = await this.repo.findOne(data);

    if (!res)
      throw new BaseException(
        ExceptionMessages.NOT_FOUND_MESSAGE(language),
        HttpStatus.NOT_FOUND,
      );

    return res;
  }

  // ========================= CREATE
  async create(data: AddGeneralSettingsDto): Promise<IReturnedGeneralSettings> {
    const { userId } = data;
    const language = this.cls.get<LANGUAGE>(CLS_KEYS.LANG);

    const isExist = await this.repo.findOneByUserId(userId);

    if (isExist) {
      throw new BaseException(
        ExceptionMessages.ALREADY_EXISTS_MESSAGE('privacySettings', language),
        HttpStatus.CONFLICT,
      );
    }

    return await this.repo.create(data);
  }

  // ========================= UPDATE
  async edit(data: EditGenearalSettingsDto): Promise<IResponse> {
    const { _id } = data;
    const { _id: userId } = this.cls.get<IReturnedUser>(CLS_KEYS.USER);
    const lang = this.cls.get<LANGUAGE>(CLS_KEYS.LANG);

    const isExist = await this.repo.existsByField({
      query: {
        $and: [{ _id }, { userId }],
      },
    });

    if (!isExist) {
      throw new BaseException(
        ExceptionMessages.NOT_FOUND_MESSAGE(lang),
        HttpStatus.NOT_FOUND,
      );
    }

    await this.repo.update({ _id: data._id }, data);

    return { success: true };
  }

  // ========================= REMOVE
  async remove(data: RemoveGeneralSettingsDto): Promise<IResponse> {
    const { _id } = data;
    const { _id: userId } = this.cls.get<IReturnedUser>(CLS_KEYS.USER);
    const lang = this.cls.get<LANGUAGE>(CLS_KEYS.LANG);

    const isExist = await this.repo.existsByField({
      query: { $and: [{ _id }, { userId }] },
    });

    if (!isExist) {
      throw new BaseException(
        ExceptionMessages.NOT_FOUND_MESSAGE(lang),
        HttpStatus.NOT_FOUND,
      );
    }

    await this.repo.removeOne({
      $and: [{ _id }, { userId }],
    });

    return { success: true };
  }

  // RESET SETTINGS TO DEFAULT
  async reset(): Promise<IResponse> {
    const { _id: userId } = this.cls.get<IReturnedUser>(CLS_KEYS.USER);
    const lang = this.cls.get<LANGUAGE>(CLS_KEYS.LANG);

    const [generalSettings, privacySettings, notificationSettings] =
      await Promise.all([
        this.repo.findOneByUserId(userId),
        this.privacySettingsRepo.findOneByUserId(userId),
        this.notificationSettingsRepo.findOneByUserId(userId),
      ]);

    if (!generalSettings || !privacySettings || !notificationSettings)
      throw new BaseException(
        ExceptionMessages.NOT_FOUND_MESSAGE(lang),
        HttpStatus.NOT_FOUND,
      );

    await Promise.all([
      this.repo.update(
        {
          _id: generalSettings._id,
        },
        {
          _id: generalSettings._id,
          startupPage: STARTUP_PAGE.MESSENGER,
          timeZone: {
            timeFormat: TIME_FORMAT.H24,
            dateFormat: DATE_FORMAT.DDMMYYYY,
          },
        },
      ),
      this.privacySettingsRepo.update(
        {
          _id: privacySettings._id,
        },
        {
          _id: privacySettings._id,
          email: PRIVACY_SETTINGS_CHOICE.NOBODY,
          bio: PRIVACY_SETTINGS_CHOICE.EVERYONE,
          gender: PRIVACY_SETTINGS_CHOICE.NOBODY,
          location: PRIVACY_SETTINGS_CHOICE.EVERYONE,
          socialLinks: PRIVACY_SETTINGS_CHOICE.EVERYONE,
          lastSeen: PRIVACY_SETTINGS_CHOICE.EVERYONE,
          avatar: PRIVACY_SETTINGS_CHOICE.EVERYONE,
          messageRequest: PRIVACY_SETTINGS_CHOICE.EVERYONE,
          birthdayDate: PRIVACY_SETTINGS_CHOICE.EVERYONE,
          friendshipRequest: true,
          readReceipts: true,
        },
      ),
      this.notificationSettingsRepo.update(
        {
          _id: notificationSettings._id,
        },
        {
          _id: notificationSettings._id,
          notificationSoundMode: NOTIFICATION_SOUND_MODE.SOUND,
          notificationContentMode: NOTIFICATION_CONTENT_MODE.HEADER_AND_CONTENT,
          sendMessageSound: true,
          receiveMessageSound: true,
          privateMessageSound: true,
          groupMessageSound: true,
          systemNotificationSound: true,
          friendshipNotificationSound: true,
          showPrivateMessageNotification: true,
          showGroupMessageNotification: true,
          showFriendshipNotification: true,
          showSystemNotification: true,
        },
      ),
    ]);

    return { success: true };
  }
}

```

## File: connectfy-account/src/modules/settings/general-settings/general-settings.module.ts
```typescript
import { Module } from '@nestjs/common';
import { GeneralSettingsService } from './general-settings.service';
import { GeneralSettingsController } from './general-settings.controller';
import { MongooseModule } from '@nestjs/mongoose';
import { GeneralSettingSchema } from './entity/general-settings.entity';
import { GeneralSettingsRepository } from './repo/general-settings.repo';
import { PrivacySettingsModule } from '../privacy-setting/privacy-settings.module';
import { NotificationSettingsModule } from '../notification-settings/notification-settings.module';
import { COLLECTIONS } from 'connectfy-shared';

@Module({
  imports: [
    MongooseModule.forFeature([
      {
        name: COLLECTIONS.ACCOUNT.SETTINGS.GENERAL,
        schema: GeneralSettingSchema,
      },
    ]),
    PrivacySettingsModule,
    NotificationSettingsModule,
  ],
  controllers: [GeneralSettingsController],
  providers: [GeneralSettingsService, GeneralSettingsRepository],
  exports: [GeneralSettingsService, GeneralSettingsRepository],
})
export class GeneralSettingsModule {}

```

## File: connectfy-account/src/modules/settings/general-settings/dto/remove.general-settings.dto.ts
```typescript
import { BaseRemoveDto } from 'connectfy-shared';

export class RemoveGeneralSettingsDto extends BaseRemoveDto {}

```

## File: connectfy-account/src/modules/settings/general-settings/dto/base.general-settings.dto.ts
```typescript
import {
  LANGUAGE,
  STARTUP_PAGE,
  THEME,
  FIELD_TYPE,
  FieldValidator,
} from 'connectfy-shared';
import { TimeZoneDto } from './nested/time-zone.dto';

export class BaseGeneralSettingsDto {
  @FieldValidator({ type: FIELD_TYPE.ENUM, enumObject: THEME })
  theme: THEME;

  // <=================  =================>
  @FieldValidator({ type: FIELD_TYPE.ENUM, enumObject: LANGUAGE })
  language: LANGUAGE;

  // <=================  =================>
  @FieldValidator({ type: FIELD_TYPE.ENUM, enumObject: STARTUP_PAGE })
  startupPage: STARTUP_PAGE;

  @FieldValidator({
    type: FIELD_TYPE.OBJECT,
    validateNested: {},
    classType: TimeZoneDto,
  })
  timeZone: TimeZoneDto;
}

```

## File: connectfy-account/src/modules/settings/general-settings/dto/edit.general-settings.dto.ts
```typescript
import { PartialType } from '@nestjs/mapped-types';
import { BaseGeneralSettingsDto } from './base.general-settings.dto';
import { FieldValidator, FIELD_TYPE } from 'connectfy-shared';

export class EditGenearalSettingsDto extends PartialType(
  BaseGeneralSettingsDto,
) {
  @FieldValidator({ type: FIELD_TYPE.UUID })
  _id: string;
}

```

## File: connectfy-account/src/modules/settings/general-settings/dto/find.general-settings.dto.ts
```typescript
import { BaseFindDto } from 'connectfy-shared';

export class FindGeneralSettingsDto extends BaseFindDto {}

```

## File: connectfy-account/src/modules/settings/general-settings/dto/add.general-settings.dto.ts
```typescript
import { FIELD_TYPE, LANGUAGE, THEME, FieldValidator } from 'connectfy-shared';

export class AddGeneralSettingsDto {
  @FieldValidator({ type: FIELD_TYPE.UUID })
  userId: string;

  // <=================  =================>
  @FieldValidator({ type: FIELD_TYPE.ENUM, enumObject: THEME })
  theme: THEME;

  // <=================  =================>
  @FieldValidator({ type: FIELD_TYPE.ENUM, enumObject: LANGUAGE })
  language: LANGUAGE;
}

```

## File: connectfy-account/src/modules/settings/general-settings/dto/nested/time-zone.dto.ts
```typescript
import {
  DATE_FORMAT,
  FIELD_TYPE,
  TIME_FORMAT,
  FieldValidator,
} from 'connectfy-shared';

export class TimeZoneDto {
  // <=================  =================>
  @FieldValidator({ type: FIELD_TYPE.ENUM, enumObject: TIME_FORMAT })
  timeFormat: TIME_FORMAT;

  // <=================  =================>
  @FieldValidator({ type: FIELD_TYPE.ENUM, enumObject: DATE_FORMAT })
  dateFormat: DATE_FORMAT;
}

```

## File: connectfy-account/src/modules/settings/general-settings/interface/general-settings.interface.ts
```typescript
import { LANGUAGE, STARTUP_PAGE, THEME } from 'connectfy-shared';
import { ITimeZone } from './nested/time-zone.interface';

export interface IGeneralSettings {
  _id: string;
  userId: string;
  theme: THEME;
  language: LANGUAGE;
  startupPage: STARTUP_PAGE;
  timeZone: ITimeZone;
  createdAt: Date;
  updatedAt: Date;
}

export interface IReturnedGeneralSettings {
  _id: string;
  userId: string;
  theme: THEME;
  language: LANGUAGE;
  startupPage: STARTUP_PAGE;
  timeZone: ITimeZone;
  createdAt: Date;
  updatedAt: Date;
}

```

## File: connectfy-account/src/modules/settings/general-settings/interface/nested/time-zone.interface.ts
```typescript
import { DATE_FORMAT, TIME_FORMAT } from 'connectfy-shared';

export interface ITimeZone {
  timeFormat: TIME_FORMAT;
  dateFormat: DATE_FORMAT;
}

```

## File: connectfy-account/src/modules/settings/general-settings/repo/general-settings.repo.ts
```typescript
import { Injectable } from '@nestjs/common';
import { GeneralSettingsDocument } from '../entity/general-settings.entity';
import { AddGeneralSettingsDto } from '../dto/add.general-settings.dto';
import { EditGenearalSettingsDto } from '../dto/edit.general-settings.dto';
import { InjectModel } from '@nestjs/mongoose';
import { Model } from 'mongoose';
import { COLLECTIONS, BaseRepository } from 'connectfy-shared';
import { IReturnedGeneralSettings } from '@modules/settings/general-settings/interface/general-settings.interface';

@Injectable()
export class GeneralSettingsRepository extends BaseRepository<
  GeneralSettingsDocument,
  IReturnedGeneralSettings,
  AddGeneralSettingsDto,
  EditGenearalSettingsDto
> {
  constructor(
    @InjectModel(COLLECTIONS.ACCOUNT.SETTINGS.GENERAL)
    protected readonly model: Model<GeneralSettingsDocument>,
  ) {
    super(model);
  }
}

```

## File: connectfy-account/src/modules/settings/general-settings/entity/general-settings.entity.ts
```typescript
import { Prop, Schema, SchemaFactory } from '@nestjs/mongoose';
import { IGeneralSettings } from '../interface/general-settings.interface';
import { v4 as uuid, validate } from 'uuid';
import {
  DATE_FORMAT,
  LANGUAGE,
  STARTUP_PAGE,
  THEME,
  TIME_FORMAT,
  COLLECTIONS,
} from 'connectfy-shared';
import { TimeZone, TimeZoneSchema } from './nested/time-zone.entity';
import { HydratedDocument } from 'mongoose';
import { t } from 'i18next';

@Schema({
  timestamps: true,
  collection: COLLECTIONS.ACCOUNT.SETTINGS.GENERAL,
  toJSON: { virtuals: true, versionKey: false, getters: true },
  toObject: { virtuals: true, versionKey: false },
  autoIndex: process.env.NODE_ENV !== 'production',
  minimize: false,
  strict: true,
  strictQuery: true,
})
export class GeneralSettingsModel implements IGeneralSettings {
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
    trim: true,
    validate: {
      validator: function (
        this: GeneralSettingsModel,
        value: string | null,
      ): boolean {
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
        field: 'theme',
      }),
    ],
    enum: {
      values: Object.values(THEME),
      message: t('validation_messages.enum', {
        lng: LANGUAGE.EN,
        field: 'theme',
        values: Object.values(THEME),
      }),
    },
    default: THEME.LIGHT,
    index: true,
  })
  theme: THEME;

  @Prop({
    type: String,
    required: [
      true,
      t('validation_messages.required', {
        lng: LANGUAGE.EN,
        field: 'language',
      }),
    ],
    enum: {
      values: Object.values(LANGUAGE),
      message: t('validation_messages.enum', {
        lng: LANGUAGE.EN,
        field: 'language',
        values: Object.values(LANGUAGE),
      }),
    },
    default: LANGUAGE.EN,
    index: true,
  })
  language: LANGUAGE;

  @Prop({
    type: String,
    required: [
      true,
      t('validation_messages.required', {
        lng: LANGUAGE.EN,
        field: 'startupPage',
      }),
    ],
    enum: {
      values: Object.values(STARTUP_PAGE),
      message: t('validation_messages.enum', {
        lng: LANGUAGE.EN,
        field: 'startupPage',
        values: Object.values(STARTUP_PAGE),
      }),
    },
    default: STARTUP_PAGE.MESSENGER,
  })
  startupPage: STARTUP_PAGE;

  @Prop({
    type: TimeZoneSchema,
    required: [
      true,
      t('validation_messages.required', {
        lng: LANGUAGE.EN,
        field: 'timeZone',
      }),
    ],
    default: () => ({
      timeFormat: TIME_FORMAT.H24,
      dateFormat: DATE_FORMAT.DDMMYYYY,
    }),
  })
  timeZone: TimeZone;

  createdAt: Date;
  updatedAt: Date;
}

export const GeneralSettingSchema =
  SchemaFactory.createForClass(GeneralSettingsModel);

// ================================================
// INDEXES - Performance optimization
// ================================================

// Compound indexes (tez-tez birlikdə sorğulanan fieldlər)
GeneralSettingSchema.index({ createdAt: -1 });
GeneralSettingSchema.index({ updatedAt: -1 });

export type GeneralSettingsDocument = HydratedDocument<GeneralSettingsModel>;

```

## File: connectfy-account/src/modules/settings/general-settings/entity/nested/time-zone.entity.ts
```typescript
import { DATE_FORMAT, TIME_FORMAT } from 'connectfy-shared';
import { Prop, Schema, SchemaFactory } from '@nestjs/mongoose';

@Schema({ timestamps: false, _id: false })
export class TimeZone {
  @Prop({
    type: String,
    required: true,
    enum: TIME_FORMAT,
    default: TIME_FORMAT.H24,
  })
  timeFormat: TIME_FORMAT;

  @Prop({
    type: String,
    required: true,
    enum: DATE_FORMAT,
    default: DATE_FORMAT.DDMMYYYY,
  })
  dateFormat: DATE_FORMAT;
}

export const TimeZoneSchema = SchemaFactory.createForClass(TimeZone);

```

## File: connectfy-account/src/modules/media/media.service.ts
```typescript
import { Injectable } from '@nestjs/common';
import { MediaRepository } from './repo/media.repo';
import { CreateMediaDto } from './dto/create-media.dto';
import { v4 as uuid } from 'uuid';

@Injectable()
export class MediaService {
  constructor(private readonly mediaRepo: MediaRepository) {}

  async create(data: CreateMediaDto) {
    return this.mediaRepo.create({
      _id: uuid(),
      ...data,
    } as any);
  }

  async findOneByUserId(userId: string, key: string) {
    return this.mediaRepo.findOne({ query: { userId, key } });
  }
}

```

## File: connectfy-account/src/modules/media/media.module.ts
```typescript
import { Module } from '@nestjs/common';
import { MongooseModule } from '@nestjs/mongoose';
import { MediaController } from './media.controller';
import { MediaService } from './media.service';
import { MediaRepository } from './repo/media.repo';
import { MediaModel, MediaSchema } from './entity/media.entity';

@Module({
  imports: [
    MongooseModule.forFeature([{ name: MediaModel.name, schema: MediaSchema }]),
  ],
  controllers: [MediaController],
  providers: [MediaService, MediaRepository],
  exports: [MediaService],
})
export class MediaModule {}

```

## File: connectfy-account/src/modules/media/media.controller.ts
```typescript
import { Controller } from '@nestjs/common';
import { MessagePattern, Payload } from '@nestjs/microservices';
import { MediaService } from './media.service';
import { CreateMediaDto } from './dto/create-media.dto';

@Controller('media')
export class MediaController {
  constructor(private readonly mediaService: MediaService) {}

  @MessagePattern('media.create')
  async create(@Payload() data: CreateMediaDto) {
    return this.mediaService.create(data);
  }

  @MessagePattern('media.findOneByUserId')
  async findOneByUserId(@Payload() data: { userId: string; key: string }) {
    return this.mediaService.findOneByUserId(data.userId, data.key);
  }
}

```

## File: connectfy-account/src/modules/media/dto/create-media.dto.ts
```typescript
import { FieldValidator, FIELD_TYPE } from 'connectfy-shared';

export class CreateMediaDto {
  @FieldValidator({ type: FIELD_TYPE.STRING })
  userId: string;

  @FieldValidator({ type: FIELD_TYPE.STRING })
  key: string;

  @FieldValidator({ type: FIELD_TYPE.URL })
  url: string;

  @FieldValidator({ type: FIELD_TYPE.STRING })
  mimeType: string;

  @FieldValidator({ type: FIELD_TYPE.NUMBER })
  size: number;

  @FieldValidator({ type: FIELD_TYPE.STRING })
  moduleName: string;
}

```

## File: connectfy-account/src/modules/media/interface/media.interface.ts
```typescript
export interface IMedia {
  _id: string;
  userId: string;
  key: string;
  url: string;
  mimeType: string;
  size: number;
  moduleName: string;
  createdAt?: Date;
  updatedAt?: Date;
}

export interface IReturnedMedia extends IMedia {}

```

## File: connectfy-account/src/modules/media/repo/media.repo.ts
```typescript
import { Injectable } from '@nestjs/common';
import { InjectModel } from '@nestjs/mongoose';
import { Model } from 'mongoose';
import { BaseRepository } from 'connectfy-shared';
import { MediaModel, MediaDocument } from '../entity/media.entity';
import { IReturnedMedia } from '../interface/media.interface';
import { CreateMediaDto } from '../dto/create-media.dto';

@Injectable()
export class MediaRepository extends BaseRepository<
  MediaDocument,
  IReturnedMedia,
  CreateMediaDto,
  any
> {
  constructor(@InjectModel(MediaModel.name) protected readonly mediaModel: Model<MediaDocument>) {
    super(mediaModel);
  }
}

```

## File: connectfy-account/src/modules/media/entity/media.entity.ts
```typescript
import { Prop, Schema, SchemaFactory } from '@nestjs/mongoose';
import { HydratedDocument } from 'mongoose';
import { IMedia } from '../interface/media.interface';

@Schema({ timestamps: true })
export class MediaModel implements IMedia {
  @Prop({ type: String, required: true })
  _id: string;

  @Prop({ type: String, required: true, index: true })
  userId: string;

  @Prop({ type: String, required: true })
  key: string;

  @Prop({ type: String, required: true })
  url: string;

  @Prop({ type: String, required: true })
  mimeType: string;

  @Prop({ type: Number, required: true })
  size: number;

  @Prop({ type: String, required: true })
  moduleName: string;

  createdAt?: Date;
  updatedAt?: Date;
}

export const MediaSchema = SchemaFactory.createForClass(MediaModel);
export type MediaDocument = HydratedDocument<MediaModel>;

```

## File: connectfy-account/src/modules/social-link/social-link.service.ts
```typescript
import { Injectable } from '@nestjs/common';
import { SocialLinkRespository } from './repo/social-link.repo';
import { FindSocialLinkDto } from './dto/find.social-link.dto';
import { IReturnedSocialLink } from './interface/social-link.interface';
import { ClsService } from 'nestjs-cls';
import {
  BaseException,
  CLS_KEYS,
  ExceptionMessages,
  FriendshipStatus,
  HttpStatus,
  IFindAllResponse,
  IRemoveAllResponse,
  IResponse,
  IReturnedUser,
  shouldShowField,
} from 'connectfy-shared';
import { AddSocialLinkDto } from './dto/add.social-link.dto';
import {
  EditSocialLinkDto,
  EditSocialLinkRankDto,
} from './dto/edit.social-link.dto';
import {
  RemoveAllSocialLinksDto,
  RemoveSocialLinkDto,
} from './dto/remove.social-link.dto';
import { TcpConnectionService } from '@/src/app-settings/tcp-connections/tcp-connection.service';
import { PrivacySettingsService } from '../settings/privacy-setting/privacy-settings.service';

@Injectable()
export class SocialLinkService {
  constructor(
    private readonly repo: SocialLinkRespository,
    private readonly cls: ClsService,
    private readonly tcpConnectionService: TcpConnectionService,
    private readonly privacySettingsService: PrivacySettingsService,
  ) {}

  // ===========================
  // Find Many
  // ===========================
  async findMany(
    params: FindSocialLinkDto,
  ): Promise<IFindAllResponse<IReturnedSocialLink> | null> {
    const { _id: currentUserId } = this.cls.get<IReturnedUser>(CLS_KEYS.USER);
    const { userId, sort } = params;

    const defaultSort: Record<string, 1 | -1> =
      sort && Object.keys(sort).length ? sort : { createdAt: -1 };

    const [data, totalCount] = await Promise.all([
      this.repo.findMany({
        query: { userId },
        sort: defaultSort,
      }),
      this.repo.count({ userId }),
    ]);

    if (currentUserId === userId) {
      return { data, totalCount };
    }

    const [relRes, privRes] = await Promise.all([
      this.tcpConnectionService.relationship({
        endpoint: 'friendship/findOneInternalWithCount',
        payload: { currentUserId, targetUserId: userId },
      }),
      this.privacySettingsService.findOne({
        query: { userId },
        fields: 'socialLinks',
      }),
    ]);

    const isAcceptedFriend = relRes?.data?.status === FriendshipStatus.Accepted;

    const shouldShow = shouldShowField(
      privRes?.socialLinks,
      isAcceptedFriend,
      false,
    );

    if (!shouldShow) {
      return null;
    }

    return { data, totalCount };
  }

  // ===========================
  // Create
  // ===========================
  async create(data: AddSocialLinkDto): Promise<IReturnedSocialLink> {
    const { _id } = this.cls.get<IReturnedUser>(CLS_KEYS.USER);

    const count = await this.repo.count({
      userId: _id,
    });

    if (count >= 5) {
      throw new BaseException(
        ExceptionMessages.LIMIT_REACHED_MESSAGE(5),
        HttpStatus.BAD_REQUEST,
      );
    }

    const finalData = {
      ...data,
      userId: _id,
      rank: count + 1,
    };

    return await this.repo.create(finalData);
  }

  // ===========================
  // Update
  // ===========================
  async update(data: EditSocialLinkDto): Promise<IResponse> {
    const { _id } = data;
    const { _id: userId } = this.cls.get<IReturnedUser>(CLS_KEYS.USER);

    const isExist = await this.repo.existsByField({
      $and: [{ _id }, { userId }],
    });

    if (!isExist) {
      throw new BaseException(
        ExceptionMessages.NOT_FOUND_MESSAGE(),
        HttpStatus.BAD_REQUEST,
      );
    }

    await this.repo.update({ _id }, data);

    return { success: true };
  }

  // ===========================
  // Update Rank
  // ===========================
  async updateRank(data: EditSocialLinkRankDto): Promise<IResponse> {
    const { _id: userId } = this.cls.get<IReturnedUser>(CLS_KEYS.USER);
    const ids = data.links.map((link) => link._id);

    const allLinks = await this.repo.findMany({
      query: { _id: { $in: ids }, userId },
      fields: '_id rank',
    });

    if (allLinks.length !== data.links.length) {
      throw new BaseException(
        ExceptionMessages.NOT_FOUND_MESSAGE(),
        HttpStatus.BAD_REQUEST,
      );
    }

    await this.repo.bulkUpdateRank(data.links);

    return { success: true };
  }

  // ===========================
  // Remove
  // ===========================
  async remove(data: RemoveSocialLinkDto): Promise<IResponse> {
    const { _id: userId } = this.cls.get<IReturnedUser>(CLS_KEYS.USER);

    const isExist = await this.repo.existsByField({
      $and: [{ _id: data._id }, { userId }],
    });

    if (!isExist) {
      throw new BaseException(
        ExceptionMessages.NOT_FOUND_MESSAGE(),
        HttpStatus.BAD_REQUEST,
      );
    }

    await this.repo.remove(data);

    return { success: true };
  }

  // ===========================
  // Remove Many
  // ===========================
  async removeMany(data: RemoveAllSocialLinksDto): Promise<IRemoveAllResponse> {
    const { _id: userId } = this.cls.get<IReturnedUser>(CLS_KEYS.USER);

    const allLinks = (await this.repo.findMany({
      query: { $and: [{ _id: { $in: data._ids } }, { userId }] },
      fields: '_id',
    })) as IReturnedSocialLink[];

    if (!allLinks.length) {
      return { deletedIds: [], notDeleted: [], deletedCount: 0 };
    }

    const allIds = allLinks.map((link) => link._id);

    const removableIds = allIds.filter((id) => data._ids.includes(id));
    const notDeleted = data._ids.filter((id) => !removableIds.includes(id));

    await this.repo.removeMany({ _ids: removableIds });

    return {
      deletedIds: removableIds,
      notDeleted,
      deletedCount: removableIds.length,
    };
  }
}

```

## File: connectfy-account/src/modules/social-link/social-link.module.ts
```typescript
import { Module } from '@nestjs/common';
import { SocialLinkService } from './social-link.service';
import { SocialLinkController } from './social-link.controller';
import { MongooseModule } from '@nestjs/mongoose';
import { SocialLinkSchema } from './entity/social-link.entity';
import { SocialLinkRespository } from './repo/social-link.repo';
import { COLLECTIONS } from 'connectfy-shared';
import { PrivacySettingsModule } from '../settings/privacy-setting/privacy-settings.module';

@Module({
  imports: [
    MongooseModule.forFeature([
      { name: COLLECTIONS.ACCOUNT.SOCIAL_LINKS, schema: SocialLinkSchema },
    ]),
    PrivacySettingsModule,
  ],
  controllers: [SocialLinkController],
  providers: [SocialLinkService, SocialLinkRespository],
  exports: [SocialLinkService, SocialLinkRespository],
})
export class SocialLinkModule {}

```

## File: connectfy-account/src/modules/social-link/social-link.controller.ts
```typescript
import { Controller } from '@nestjs/common';
import { SocialLinkService } from './social-link.service';
import { MessagePattern, Transport } from '@nestjs/microservices';
import { FindSocialLinkDto } from './dto/find.social-link.dto';
import { AddSocialLinkDto } from './dto/add.social-link.dto';
import {
  EditSocialLinkDto,
  EditSocialLinkRankDto,
} from './dto/edit.social-link.dto';
import {
  RemoveAllSocialLinksDto,
  RemoveSocialLinkDto,
} from './dto/remove.social-link.dto';

@Controller('')
export class SocialLinkController {
  constructor(private readonly service: SocialLinkService) {}

  @MessagePattern('socialLinks/findMany', Transport.TCP)
  async findMany(params: FindSocialLinkDto) {
    return this.service.findMany(params);
  }

  @MessagePattern('socialLinks/create', Transport.TCP)
  async create(data: AddSocialLinkDto) {
    return this.service.create(data);
  }

  @MessagePattern('socialLinks/update', Transport.TCP)
  async update(data: EditSocialLinkDto) {
    return this.service.update(data);
  }

  @MessagePattern('socialLinks/updateRank', Transport.TCP)
  async updateRank(data: EditSocialLinkRankDto) {
    return this.service.updateRank(data);
  }

  @MessagePattern('socialLinks/remove', Transport.TCP)
  async remove(data: RemoveSocialLinkDto) {
    return this.service.remove(data);
  }

  @MessagePattern('socialLinks/removeMany', Transport.TCP)
  async removeMany(data: RemoveAllSocialLinksDto) {
    return this.service.removeMany(data);
  }
}

```

## File: connectfy-account/src/modules/social-link/dto/find.social-link.dto.ts
```typescript
import { FIELD_TYPE, FieldValidator } from 'connectfy-shared';

export class FindSocialLinkDto {
  @FieldValidator({ type: FIELD_TYPE.UUID })
  userId: string;

  @FieldValidator({
    type: FIELD_TYPE.OBJECT,
    isOptional: true,
  })
  sort?: Record<string, 1 | -1>;
}

```

## File: connectfy-account/src/modules/social-link/dto/edit.social-link.dto.ts
```typescript
import { PartialType } from '@nestjs/mapped-types';
import { BaseSocialLinkDto } from './base.social-link.dto';
import { FieldValidator, FIELD_TYPE } from 'connectfy-shared';

export class EditSocialLinkDto extends PartialType(BaseSocialLinkDto) {
  @FieldValidator({ type: FIELD_TYPE.UUID })
  _id: string;
}

export class SocialLinkRankDto {
  @FieldValidator({ type: FIELD_TYPE.UUID })
  _id: string;

  @FieldValidator({ type: FIELD_TYPE.NUMBER })
  rank: number;
}

export class EditSocialLinkRankDto {
  @FieldValidator({
    type: FIELD_TYPE.ARRAY,
    classType: SocialLinkRankDto,
    minSize: 1,
    maxSize: 5,
  })
  links: SocialLinkRankDto[];
}

```

## File: connectfy-account/src/modules/social-link/dto/remove.social-link.dto.ts
```typescript
import { BaseRemoveDto, FIELD_TYPE, FieldValidator } from 'connectfy-shared';

export class RemoveSocialLinkDto extends BaseRemoveDto {}

export class RemoveAllSocialLinksDto {
  @FieldValidator({
    type: FIELD_TYPE.ARRAY,
    minSize: 1,
    maxSize: 5,
    arrayItemType: FIELD_TYPE.UUID,
  })
  _ids: string[];
}

```

## File: connectfy-account/src/modules/social-link/dto/base.social-link.dto.ts
```typescript
import {
  FIELD_TYPE,
  SOCIAL_LINK_PLATFORM,
  FieldValidator,
} from 'connectfy-shared';

export class BaseSocialLinkDto {
  @FieldValidator({ type: FIELD_TYPE.STRING, minLength: 1, maxLength: 50 })
  name: string;

  @FieldValidator({ type: FIELD_TYPE.URL, maxLength: 200 })
  url: string;

  @FieldValidator({ type: FIELD_TYPE.ENUM, enumObject: SOCIAL_LINK_PLATFORM })
  platform: SOCIAL_LINK_PLATFORM;
}

```

## File: connectfy-account/src/modules/social-link/dto/add.social-link.dto.ts
```typescript
import { BaseSocialLinkDto } from './base.social-link.dto';

export class AddSocialLinkDto extends BaseSocialLinkDto {}

```

## File: connectfy-account/src/modules/social-link/interface/social-link.interface.ts
```typescript
import { SOCIAL_LINK_PLATFORM } from 'connectfy-shared';

export interface ISocialLink {
  _id: string;
  userId: string;
  name: string;
  url: string;
  rank: number;
  platform: SOCIAL_LINK_PLATFORM;
  createdAt: Date;
  updatedAt: Date;
}

export interface IReturnedSocialLink {
  _id: string;
  userId: string;
  name: string;
  url: string;
  rank: number;
  platform: SOCIAL_LINK_PLATFORM;
  createdAt: Date;
  updatedAt: Date;
}

```

## File: connectfy-account/src/modules/social-link/repo/social-link.repo.ts
```typescript
import { Injectable } from '@nestjs/common';
import { SocialLinkDocument } from '../entity/social-link.entity';
import { AddSocialLinkDto } from '../dto/add.social-link.dto';
import {
  EditSocialLinkDto,
  SocialLinkRankDto,
} from '../dto/edit.social-link.dto';
import { Model } from 'mongoose';
import { InjectModel } from '@nestjs/mongoose';
import { COLLECTIONS, BaseRepository } from 'connectfy-shared';
import { IReturnedSocialLink } from '@modules/social-link/interface/social-link.interface';

@Injectable()
export class SocialLinkRespository extends BaseRepository<
  SocialLinkDocument,
  IReturnedSocialLink,
  AddSocialLinkDto,
  EditSocialLinkDto
> {
  constructor(
    @InjectModel(COLLECTIONS.ACCOUNT.SOCIAL_LINKS)
    protected readonly model: Model<SocialLinkDocument>,
  ) {
    super(model);
  }

  async bulkUpdateRank(data: SocialLinkRankDto[]): Promise<any> {
    const bulkOps = data.map((item) => ({
      updateOne: {
        filter: { _id: item._id },
        update: { $set: { rank: item.rank } },
      },
    }));

    return this.model.bulkWrite(bulkOps);
  }
}

```

## File: connectfy-account/src/modules/social-link/entity/social-link.entity.ts
```typescript
import { Prop, Schema, SchemaFactory } from '@nestjs/mongoose';
import { ISocialLink } from '../interface/social-link.interface';
import { v4 as uuid, validate } from 'uuid';
import { COLLECTIONS, SOCIAL_LINK_PLATFORM, LANGUAGE } from 'connectfy-shared';
import { HydratedDocument } from 'mongoose';
import { t } from 'i18next';
import { max, min } from 'class-validator';

@Schema({
  timestamps: true,
  collection: COLLECTIONS.ACCOUNT.SOCIAL_LINKS,
  toJSON: { virtuals: true, versionKey: false, getters: true },
  toObject: { virtuals: true, versionKey: false },
  autoIndex: process.env.NODE_ENV !== 'production',
  minimize: false,
  strict: true,
  strictQuery: true,
})
export class SocialLinkModel implements ISocialLink {
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
  })
  userId: string;

  @Prop({
    type: String,
    required: [
      true,
      t('validation_messages.required', {
        lng: LANGUAGE.EN,
        field: 'name',
      }),
    ],
    trim: true,
    minlength: [
      1,
      t('validation_messages.min_length', {
        lng: LANGUAGE.EN,
        field: 'name',
        length: 1,
      }),
    ],
    maxlength: [
      50,
      t('validation_messages.max_length', {
        lng: LANGUAGE.EN,
        field: 'name',
        length: 50,
      }),
    ],
    index: true,
  })
  name: string;

  @Prop({
    type: String,
    required: [
      true,
      t('validation_messages.required', {
        lng: LANGUAGE.EN,
        field: 'url',
      }),
    ],
    trim: true,
    lowercase: true,
    maxlength: [
      200,
      t('validation_messages.max_length', {
        lng: LANGUAGE.EN,
        field: 'url',
        length: 200,
      }),
    ],
    validate: {
      validator: function (value: string): boolean {
        try {
          const url = new URL(value);
          return ['http:', 'https:'].includes(url.protocol);
        } catch {
          return false;
        }
      },
      message: t('validation_messages.invalid_url', {
        lng: LANGUAGE.EN,
        field: 'url',
      }),
    },
  })
  url: string;

  @Prop({
    type: String,
    required: [
      true,
      t('validation_messages.required', {
        lng: LANGUAGE.EN,
        field: 'platform',
      }),
    ],
    enum: {
      values: Object.values(SOCIAL_LINK_PLATFORM),
      message: t('validation_messages.enum', {
        lng: LANGUAGE.EN,
        field: 'platform',
        values: Object.values(SOCIAL_LINK_PLATFORM),
      }),
    },
    default: SOCIAL_LINK_PLATFORM.OTHER,
    index: true,
  })
  platform: SOCIAL_LINK_PLATFORM;

  @Prop({
    type: Number,
    required: [
      true,
      t('validation_messages.required', {
        lng: LANGUAGE.EN,
        field: 'rank',
      }),
    ],
    min: [
      1,
      t('validation_messages.min', { lng: LANGUAGE.EN, field: 'rank', min: 1 }),
    ],
    max: [
      5,
      t('validation_messages.max', { lng: LANGUAGE.EN, field: 'rank', max: 5 }),
    ],
    index: true,
  })
  rank: number;

  createdAt: Date;
  updatedAt: Date;
}

export const SocialLinkSchema = SchemaFactory.createForClass(SocialLinkModel);

// ================================================
// INDEXES - Performance optimization
// ================================================

// Compound indexes (çox istifadə olunan queries üçün)
SocialLinkSchema.index({ userId: 1, platform: 1 });
SocialLinkSchema.index({ userId: 1, createdAt: -1 });
SocialLinkSchema.index({ userId: 1, rank: 1 });
SocialLinkSchema.index({ userId: 1, name: 1 });

// Single field indexes
SocialLinkSchema.index({ createdAt: -1 });
SocialLinkSchema.index({ updatedAt: -1 });

export type SocialLinkDocument = HydratedDocument<SocialLinkModel>;

```

## File: connectfy-account/src/common/constants/environment-variables.ts
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

  // TCP
  AUTH_SERVICE_HOST: process.env.AUTH_SERVICE_HOST,
  AUTH_SERVICE_PORT: Number(process.env.AUTH_SERVICE_PORT),

  MESSENGER_SERVICE_HOST: process.env.MESSENGER_SERVICE_HOST,
  MESSENGER_SERVICE_PORT: Number(process.env.MESSENGER_SERVICE_PORT),

  RELATIONSHIP_SERVICE_HOST: process.env.RELATIONSHIP_SERVICE_HOST,
  RELATIONSHIP_SERVICE_PORT: Number(process.env.RELATIONSHIP_SERVICE_PORT),

  NOTIFICATION_ACTION_HISTORY_SERVICE_HOST:
    process.env.NOTIFICATION_ACTION_HISTORY_SERVICE_HOST,
  NOTIFICATION_ACTION_HISTORY_SERVICE_PORT: Number(
    process.env.NOTIFICATION_ACTION_HISTORY_SERVICE_PORT,
  ),

  FILE_UPLOADER_BASE_URL: process.env.FILE_UPLOADER_BASE_URL || 'http://connectfy-file-uploader-dev:9003',
  INTERNAL_SERVICE_API_KEY: process.env.INTERNAL_SERVICE_API_KEY || 'default-internal-key',
};

```

## File: connectfy-account/src/app-settings/app-settings.module.ts
```typescript
import { Global, Module } from '@nestjs/common';
import { TcpConnectionModule } from './tcp-connections/tcp-connection.module';
import { KafkaConnectionModule } from './kafka-connections/kafka-connection.module';
import { HttpConnectionModule } from './http-connections/http-connection.module';

@Global()
@Module({
  imports: [TcpConnectionModule, KafkaConnectionModule, HttpConnectionModule],
  controllers: [],
  providers: [],
  exports: [TcpConnectionModule, KafkaConnectionModule, HttpConnectionModule],
})
export class AppSettingsModule {}

```

## File: connectfy-account/src/app-settings/kafka-connections/loggin-kafka.server.ts
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

## File: connectfy-account/src/app-settings/kafka-connections/kafka-connection.service.ts
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

## File: connectfy-account/src/app-settings/kafka-connections/kafka-connection.module.ts
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

## File: connectfy-account/src/app-settings/tcp-connections/tcp-connection.service.ts
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

## File: connectfy-account/src/app-settings/tcp-connections/tcp-connection.module.ts
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

## File: connectfy-account/src/app-settings/http-connections/http-connection.service.ts
```typescript
import { HttpService } from '@nestjs/axios';
import { Injectable } from '@nestjs/common';
import { firstValueFrom } from 'rxjs';
import { AxiosRequestConfig } from 'axios';
import { ClsService } from 'nestjs-cls';
import { ENVIRONMENT_VARIABLES } from '@/src/common/constants/environment-variables';

@Injectable()
export class HttpConnectionService {
  constructor(
    private readonly http: HttpService,
    private readonly cls: ClsService,
  ) {}

  async post<T>({
    baseUrl,
    endPoint,
    payload = {},
    config = {},
  }: {
    baseUrl: string;
    endPoint: string;
    payload?: Record<string, any>;
    config?: AxiosRequestConfig;
  }): Promise<T> {
    const url = this.joinBaseUrlAndEndpoint(baseUrl, endPoint);
    const user = this.cls.get('user');

    const data = {
      ...payload,
      _loggedUser: user,
    };

    const finalConfig: AxiosRequestConfig = {
      ...config,
      headers: {
        ...(config.headers ?? {}),
        'x-internal-api-key': ENVIRONMENT_VARIABLES.INTERNAL_SERVICE_API_KEY,
      },
    };

    const response = await firstValueFrom(
      this.http.post<T>(url, data, finalConfig),
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

## File: connectfy-account/src/app-settings/http-connections/http-connection.module.ts
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

