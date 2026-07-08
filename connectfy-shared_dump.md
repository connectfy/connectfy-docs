# connectfy-shared Source Dump

## File: connectfy-shared/src/i18n.ts
```typescript
// src/i18n.ts
import i18next from "i18next";
import { resources } from "connectfy-i18n";

// Optional: avtomatik init etmək istəyirsinizsə (amma side-effect yaradır):
i18next.init({
  resources,
  fallbackLng: "en",
  lng: "en",
  interpolation: { escapeValue: false },
});

// Export həm default həm named olsun — beləcə köhnə importlar işləyər
export default i18next;
export { i18next as i18n };

```

## File: connectfy-shared/src/index.ts
```typescript
export * from "./class";
export * from "./constants";
export * from "./decorators";
export * from "./dto";
export * from "./enums";
export * from "./exception-filters";
export * from "./functions";
export * from "./interfaces";
export * from "./repo";

```

## File: connectfy-shared/src/exception-filters/index.ts
```typescript
export * from "./server/all.filter";

```

## File: connectfy-shared/src/exception-filters/server/all.filter.ts
```typescript
import { Catch, ExceptionFilter, HttpException } from "@nestjs/common";
import { RpcException } from "@nestjs/microservices";
import { throwError, Observable } from "rxjs";
import { ExceptionMessages } from "../../constants/server/exception.messages";
import { HttpStatus } from "@nestjs/common";
import { LANGUAGE } from "../../enums/enum";

@Catch()
export class AllExceptionsFilter implements ExceptionFilter {
  catch(exception: any): Observable<any> {
    let message = ExceptionMessages.INTERNAL_SERVER_ERROR_MESSAGE(LANGUAGE.EN);
    let statusCode = HttpStatus.INTERNAL_SERVER_ERROR;
    let additional: Record<string, any> | undefined;

    // -------------------------------------------------------------------------
    // DIAGNOSTIC LOGGING (You can remove this later)
    // -------------------------------------------------------------------------
    // this.logger.error("Raw Exception Structure", JSON.stringify(exception, null, 2));

    // -------------------------------------------------------------------------
    // 1. Handle RPC / BaseException (Duck Typing)
    // -------------------------------------------------------------------------
    // We check if it acts like an RpcException (has .getError() or .error prop)
    // This fixes the "instanceof" failure across shared libraries.
    const isRpcException =
      exception instanceof RpcException ||
      (exception &&
        typeof exception === "object" &&
        "error" in exception &&
        typeof exception.getError === "function") ||
      (exception &&
        typeof exception === "object" &&
        "error" in exception &&
        typeof exception.error === "object");

    if (isRpcException) {
      // If .getError() exists use it, otherwise access .error directly
      const errorObj =
        typeof exception.getError === "function"
          ? exception.getError()
          : exception.error;

      if (typeof errorObj === "object" && errorObj !== null) {
        message = (errorObj as any)?.message || message;
        statusCode = (errorObj as any)?.statusCode || statusCode;
        additional = (errorObj as { additional?: Record<string, any> })
          ?.additional;
      } else if (typeof errorObj === "string") {
        message = errorObj;
      }
    }

    // -------------------------------------------------------------------------
    // 2. Handle HTTP Exceptions
    // -------------------------------------------------------------------------
    else if (exception instanceof HttpException) {
      const response = exception.getResponse();
      const resObj =
        typeof response === "string"
          ? { message: response }
          : (response as any);

      message = resObj.message || exception.message;
      statusCode = exception.getStatus();
      additional = resObj?.additional;
    }

    // -------------------------------------------------------------------------
    // 3. Handle Standard JS Errors
    // -------------------------------------------------------------------------
    else if (exception instanceof Error) {
      // Only override if we haven't found a better message yet
      if (
        message === ExceptionMessages.INTERNAL_SERVER_ERROR_MESSAGE(LANGUAGE.EN)
      ) {
        // Be careful not to expose sensitive stack traces in production
        message = exception.message;
      }
    }

    const lastResult = {
      status: "error",
      message, // This will now be the correct string from Step 1
      statusCode,
      timestamp: new Date().toISOString(), // Good practice to add
      ...(additional ? { additional } : {}),
    };

    return throwError(() => lastResult);
  }
}

```

## File: connectfy-shared/src/enums/enum.ts
```typescript
export enum GENDER {
  MALE = "MALE",
  FEMALE = "FEMALE",
  OTHER = "OTHER",
}

export enum PRIVACY_SETTINGS_CHOICE {
  EVERYONE = "EVERYONE",
  MY_FRIENDS = "MY_FRIENDS",
  NOBODY = "NOBODY",
}

export enum SOCIAL_LINK_PLATFORM {
  INSTAGRAM = "Instagram",
  FACEBOOK = "Facebook",
  X = "X",
  LINKEDIN = "LinkedIn",
  YOUTUBE = "Youtube",
  GITHUB = "Github",
  REDDIT = "Reddit",
  PINTEREST = "Pinterest",
  TIKTOK = "Tiktok",
  TELEGRAM = "Telegram",
  DISCORD = "Discord",
  SNAPCHAT = "Snapchat",
  WHATSAPP = "Whatsapp",
  MEDIUM = "Medium",
  BEHANCE = "Behance",
  EMAIL = "Email",
  WEBSITE = "Website",
  OTHER = "Other",
}

export enum LANGUAGE {
  EN = "en",
  AZ = "az",
  RU = "ru",
  TR = "tr",
}

export enum THEME {
  DARK = "DARK",
  LIGHT = "LIGHT",
  DEVICE = "DEVICE",
}

export enum STARTUP_PAGE {
  MESSENGER = "MESSENGER",
  GROUPS = "GROUPS",
  CHANNELS = "CHANNELS",
  USERS = "USERS",
  NOTIFICATIONS = "NOTIFICATIONS",
  PROFILE = "PROFILE",
}

export enum TIME_FORMAT {
  H24 = "24h",
  H12 = "12h",
}

export enum DATE_FORMAT {
  DDMMYYYY = "DD/MM/YYYY",
  MMDDYYYY = "MM/DD/YYYY",
}

export enum NOTIFICATION_SOUND_MODE {
  SOUND = "SOUND",
  SILENT = "SILENT",
  DND = "DND",
}

export enum NOTIFICATION_CONTENT_MODE {
  HEADER_AND_CONTENT = "HEADER_AND_CONTENT",
  HEADER_ONLY = "HEADER_ONLY",
  HIDE_NOTIFICATION = "HIDE_NOTIFICATION",
}

export enum ROLE {
  ADMIN = "ADMIN",
  MODERATOR = "MODERATOR",
  USER = "USER",
}

export enum PROVIDER {
  PASSWORD = "PASSWORD",
  GOOGLE = "GOOGLE",
  FACEBOOK = "FACEBOOK",
}

export enum TIME_DIFFERENCE_TYPE {
  NOW = "just_now",
  MINUTE = "minutes_ago",
  HOUR = "hours_ago",
  DAY = "days_ago",
}

export enum CLS_KEYS {
  USER = "user",
  LANG = "lang",
  ACCOUNT = "account",
  SETTINGS = "settings",
  DEVICE_ID = "deviceId",
}

export enum FIELD_TYPE {
  // String əsaslı
  STRING = "string", // @IsString
  UUID = "uuid", // @IsUUID
  EMAIL = "email", // @IsEmail
  URL = "url", // @IsUrl
  ENUM = "enum", // @IsEnum
  PHONE = "phone", // @IsPhoneNumber
  JSON = "json", // @IsJSON
  JWT = "jwt", // @IsJWT
  LOWERCASE = "lowercase", // @IsLowercase
  UPPERCASE = "uppercase", // @IsUppercase
  HEX_COLOR = "hex_color", // @IsHexColor
  BASE64 = "base64", // @IsBase64
  IP = "ip", // @IsIP
  MAC_ADDRESS = "mac_address", // @IsMACAddress

  // Number əsaslı
  NUMBER = "number", // @IsNumber
  INT = "int", // @IsInt
  POSITIVE = "positive", // @IsPositive
  NEGATIVE = "negative", // @IsNegative
  DECIMAL = "decimal", // @IsDecimal
  LATITUDE = "latitude", // @IsLatitude
  LONGITUDE = "longitude", // @IsLongitude

  // Boolean
  BOOLEAN = "boolean", // @IsBoolean

  // Date & Time
  DATE = "date", // @IsDate
  DATE_STRING = "date_string", // @IsDateString

  // Array
  ARRAY = "array", // @IsArray

  // Object
  OBJECT = "object", // @IsObject
}

export enum VALIDATION_TYPE {
  REQUIRED = "required",
  STRING = "string",
  INT = "int",
  NUMBER = "number",
  MIN = "min",
  MAX = "max",
  DATE = "date",
  UUID = "uuid",
  ARRAY = "array",
  ENUM = "enum",
  BOOLEAN = "boolean",
  EXISTS = "exist",
  AVAILABLE = "available",
  NOT_ALLOWED_FIELD = "not_allowed_field",
  TYPE_MISMATCH = "type_mismatch",
  OBJECT = "object",
  EMAIL = "email",
  MISMATCH = "mismatch",
  PASSWORD = "password",
  INVALID_LENGTH = "invalid_length",
  ARRAY_EACH = "array_each",
  PHONE_CODE = "phone_code",
  FULL_PHONE = "full_phone",
  URL = "url",
  PHONE = "phone",
  JSON = "json",
  JWT = "jwt",
  LOWERCASE = "lowercase",
  UPPERCASE = "uppercase",
  HEX_COLOR = "hex_color",
  BASE64 = "base64",
  IP = "ip",
  MAC_ADDRESS = "mac_address",
  POSITIVE = "positive",
  NEGATIVE = "negative",
  DECIMAL = "decimal",
  LATITUDE = "latitude",
  LONGITUDE = "longitude",
  MIN_LENGTH = "min_length",
  MAX_LENGTH = "max_length",
  ARRAY_MIN_SIZE = "array_min_size",
  ARRAY_MAX_SIZE = "array_max_size",
  ARRAY_UNIQUE = "array_unique",
  MIN_DATE = "min_date",
  MAX_DATE = "max_date",
}

export enum USER_STATUS {
  ACTIVE = "ACTIVE",
  INACTIVE = "INACTIVE",
  DELETED = "DELETED",
  BANNED = "BANNED",
}

export enum TOKEN_TYPE {
  PASSWORD_RESET = "PASSWORD_RESET",
  DELETE_ACCOUNT = "DELETE_ACCOUNT",
  RESTORE_ACCOUNT = "RESTORE_ACCOUNT",
  CHANGE_USERNAME = "CHANGE_USERNAME",
  CHANGE_EMAIL = "CHANGE_EMAIL",
  CHANGE_PASSWORD = "CHANGE_PASSWORD",
  CHANGE_PHONE_NUMBER = "CHANGE_PHONE_NUMBER",
  DEACTIVATE_ACCOUNT = "DEACTIVATE_ACCOUNT",
  TWO_FACTOR = "TWO_FACTOR",
}

export enum TWO_FACTOR_ACTION {
  ENABLE = "ENABLE",
  DISABLE = "DISABLE",
}

export enum PHONE_NUMBER_ACTION {
  UPDATE = "UPDATE",
  REMOVE = "REMOVE",
}

export enum IDENTIFIER_TYPE {
  USERNAME = "USERNAME",
  EMAIL = "EMAIL",
  PHONE_NUMBER = "PHONE_NUMBER",
  FACE_DESCRIPTOR = "FACE_DESCRIPTOR",
}

export enum GOOGLE_AUTH_LOGIN_TYPE {
  LOGIN = "LOGIN",
  SIGNUP = "SIGNUP",
}

export enum FORGOT_PASSWORD_IDENTIFIER_TYPE {
  EMAIL = "EMAIL",
  PHONE_NUMBER = "PHONE_NUMBER",
}

export enum DELETE_REASON {
  USER_REQUEST = "USER_REQUEST",
  TERMS_VIOLATION = "TERMS_VIOLATION",
  SPAM = "SPAM",
  FRAUD = "FRAUD",
  SECURITY = "SECURITY",
  INACTIVITY = "INACTIVITY",
  ADMIN_ACTION = "ADMIN_ACTION",
  OTHER = "OTHER",
}

export enum DELETE_REASON_CODE {
  NOT_USEFUL = "NOT_USEFUL",
  PRIVACY_CONCERNS = "PRIVACY_CONCERNS",
  FOUND_ALTERNATIVE = "FOUND_ALTERNATIVE",
  TECHNICAL_ISSUES = "TECHNICAL_ISSUES",
  OTHER = "OTHER",
}

export enum DEVICE_TYPE {
  WEB = "WEB",
  MOBILE = "MOBILE",
  TABLET = "TABLET",
  DESKTOP = "DESKTOP",
  UNKNOWN = "UNKNOWN",
}

export enum BROWSER_TYPE {
  CHROME = "CHROME",
  FIREFOX = "FIREFOX",
  SAFARI = "SAFARI",
  EDGE = "EDGE",
  OPERA = "OPERA",
  BRAVE = "BRAVE",
  SAMSUNG = "SAMSUNG",
  UNKNOWN = "UNKNOWN",
}

export enum OS_TYPE {
  WINDOWS = "WINDOWS",
  MACOS = "MACOS",
  LINUX = "LINUX",
  ANDROID = "ANDROID",
  IOS = "IOS",
  UNKNOWN = "UNKNOWN",
}

export enum CHECK_UNIQUE_FIELD {
  USERNAME = "username",
  EMAIL = "email",
  PHONE_NUMBER = "phone_number",
}

export enum REDUCER_PATH {
  PROFILE = "profilePath",
  AUTH = "authPath",
  GENERAL_SETTINGS = "generalSettingsPath",
  NOTIFICATIONS_SETTINGS = "notificationsSettingsPath",
  PRIVACY_SETTINGS = "privacySettingsPath",
}

export enum TAG_TYPES {
  PROFILE = "Profile",
  AUTH = "Auth",
  GENERAL_SETTINGS = "GeneralSettings",
  NOTIFICATIONS_SETTINGS = "NotificationsSettings",
  PRIVACY_SETTINGS = "PrivacySettings",
}

export enum RESOURCE {
  AUTH = "auth",
  GENERAL_SETTINGS = "general-settings",
  PRIVACY_SETTINGS = "privacy-settings",
  NOTIFICATION_SETTINGS = "notification-settings",
  USER = "user",
  PROFILE = "profile",
  ACCOUNT_SETTINGS = "account-settings",
}

export enum LOCAL_STORAGE_KEYS {
  LANG = "lang",
  ACCESS_TOKEN = "access_token",
  DEVICE_ID = "deviceId",
  APP_THEME = "app-theme",
  AUTH_PAGE = "auth-page",
  LOGIN_MODE = "login-mode",
  FORGOT_PASSWORD_MODE = "forgot-password-mode",
  OTP_EXPIRES_AT = "otp_expires_at",
  SIGNUP_FORM = "signup_form",
}

export enum FriendshipStatus {
  Pending = "PENDING",
  Accepted = "ACCEPTED",
  Blocked = "BLOCKED",
}

export enum HttpStatus {
  CONTINUE = 100,
  SWITCHING_PROTOCOLS = 101,
  PROCESSING = 102,
  EARLYHINTS = 103,
  OK = 200,
  CREATED = 201,
  ACCEPTED = 202,
  NON_AUTHORITATIVE_INFORMATION = 203,
  NO_CONTENT = 204,
  RESET_CONTENT = 205,
  PARTIAL_CONTENT = 206,
  MULTI_STATUS = 207,
  ALREADY_REPORTED = 208,
  CONTENT_DIFFERENT = 210,
  AMBIGUOUS = 300,
  MOVED_PERMANENTLY = 301,
  FOUND = 302,
  SEE_OTHER = 303,
  NOT_MODIFIED = 304,
  TEMPORARY_REDIRECT = 307,
  PERMANENT_REDIRECT = 308,
  BAD_REQUEST = 400,
  UNAUTHORIZED = 401,
  PAYMENT_REQUIRED = 402,
  FORBIDDEN = 403,
  NOT_FOUND = 404,
  METHOD_NOT_ALLOWED = 405,
  NOT_ACCEPTABLE = 406,
  PROXY_AUTHENTICATION_REQUIRED = 407,
  REQUEST_TIMEOUT = 408,
  CONFLICT = 409,
  GONE = 410,
  LENGTH_REQUIRED = 411,
  PRECONDITION_FAILED = 412,
  PAYLOAD_TOO_LARGE = 413,
  URI_TOO_LONG = 414,
  UNSUPPORTED_MEDIA_TYPE = 415,
  REQUESTED_RANGE_NOT_SATISFIABLE = 416,
  EXPECTATION_FAILED = 417,
  I_AM_A_TEAPOT = 418,
  MISDIRECTED = 421,
  UNPROCESSABLE_ENTITY = 422,
  LOCKED = 423,
  FAILED_DEPENDENCY = 424,
  PRECONDITION_REQUIRED = 428,
  TOO_MANY_REQUESTS = 429,
  UNRECOVERABLE_ERROR = 456,
  INTERNAL_SERVER_ERROR = 500,
  NOT_IMPLEMENTED = 501,
  BAD_GATEWAY = 502,
  SERVICE_UNAVAILABLE = 503,
  GATEWAY_TIMEOUT = 504,
  HTTP_VERSION_NOT_SUPPORTED = 505,
  INSUFFICIENT_STORAGE = 507,
  LOOP_DETECTED = 508,
}

export enum REDIS_KEYS {
  REDIS_CLIENT = "REDIS_CLIENT",
  KEYV_CLIENT = "KEYV_CLIENT",
}

export enum FileOwnerModule {
  PROFILE_PHOTO = "PROFILE_PHOTO",
}

export enum FileUploadTopic {
  PROFILE_PHOTO_UPLOADED = "profile.photo.uploaded",
}

export enum ProfilePhotoUpdateAction {
  Remove = "Remove",
  Update = "Update",
  SetDefault = "SetDefault",
}

export enum AvatarFormats {
  Adventurer = "adventurer",
  Avataaars = "avataaars",
  Personas = "personas",
  ToonHead = "toon-head",
  Micah = "micah",
}

export enum FriendshipRequestType {
  Received = "Received",
  Sent = "Sent",
}

export enum NotificationType {
  // Social
  FRIENDSHIP_REQUEST_SENT = "friendship_request_sent",
  FRIENDSHIP_REQUEST_ACCEPTED = "friendship_request_accepted",
  FRIENDSHIP_REQUEST_DECLINED = "friendship_request_declined",
  // Messaging (Phase 2)
  NEW_MESSAGE = "message_new",
  // System
  SYSTEM_ALERT = "system_alert",
  ACCOUNT_WARNING = "account_warning",
}

export enum NotificationStatus {
  Unread = "unread",
  Read = "read",
  Archived = "archived",
  Pending = "pending",
  Accepted = "accepted",
  Declined = "declined",
  Cancelled = "cancelled",
}

export enum NotificationChannel {
  InApp = "in_app",
  Push = "push",
  Both = "both",
}

export enum Push_DELIVERY_STATUS {
  PENDING = "pending",
  DELIVERED = "delivered",
  FAILED = "failed",
  SKIPPED = "skipped", // user preference disabled
}

export enum NotificationResource {
  Friendship = "friendship",
  Message = "message",
  System = "system",
  Account = "account",
}

```

## File: connectfy-shared/src/enums/index.ts
```typescript
export * from "./enum";

```

## File: connectfy-shared/src/dto/index.ts
```typescript
export * from "./server/base.remove.dto";
export * from "./server/base.find.dto";
export * from "./server/populate.option.dto";

```

## File: connectfy-shared/src/dto/server/base.remove.dto.ts
```typescript
import { FieldValidator } from "../../decorators/server/field-validator/field-validator.decorator";
import { FIELD_TYPE } from "../../enums/enum";

export class BaseRemoveDto {
  @FieldValidator({
    type: FIELD_TYPE.UUID,
  })
  _id: string;
}

export class BaseRemoveAllDto {
  @FieldValidator({
    type: FIELD_TYPE.ARRAY,
    minSize: 1,
    arrayItemType: FIELD_TYPE.UUID,
  })
  _ids: string[];
}

```

## File: connectfy-shared/src/dto/server/populate.option.dto.ts
```typescript
import { FieldValidator } from "../../decorators/server/field-validator/field-validator.decorator";
import { FIELD_TYPE } from "../../enums/enum";

export class PopulateOption {
  @FieldValidator({
    type: FIELD_TYPE.STRING,
    isOptional: true,
  })
  path?: string;

  @FieldValidator({
    type: FIELD_TYPE.STRING,
    isOptional: true,
  })
  select?: string;

  @FieldValidator({
    type: FIELD_TYPE.STRING,
    isOptional: true,
  })
  collection?: string;

  @FieldValidator({
    type: FIELD_TYPE.OBJECT,
    isOptional: true,
  })
  match?: Record<string, any>;
}

```

## File: connectfy-shared/src/dto/server/base.find.dto.ts
```typescript
import { FieldValidator } from "../../decorators/server/field-validator/field-validator.decorator";
import { PopulateOption } from "./populate.option.dto";
import { FIELD_TYPE } from "../../enums/enum";

export class BaseFindDto {
  @FieldValidator({
    type: FIELD_TYPE.INT,
    isOptional: true,
    classType: Number,
  })
  limit?: number;

  @FieldValidator({
    type: FIELD_TYPE.INT,
    isOptional: true,
    classType: Number,
  })
  skip?: number;

  @FieldValidator({
    type: FIELD_TYPE.OBJECT,
    isOptional: true,
  })
  sort?: Record<string, 1 | -1>;

  @FieldValidator({
    type: FIELD_TYPE.OBJECT,
    isOptional: true,
  })
  query?: Record<string, any>;

  @FieldValidator({
    type: FIELD_TYPE.STRING,
    isOptional: true,
  })
  fields?: string;

  @FieldValidator({
    type: FIELD_TYPE.ARRAY,
    isOptional: true,
    validateNested: { each: true },
    classType: PopulateOption,
  })
  populate?: PopulateOption[];
}

```

## File: connectfy-shared/src/repo/index.ts
```typescript
export * from "./server/base.repository";

```

## File: connectfy-shared/src/repo/server/base.repository.ts
```typescript
import {
  Document,
  Model,
  PipelineStage,
  PopulateOptions,
  UpdateQuery,
} from "mongoose";
import {
  IBaseRepositoryInterface,
  IBaseRepositoryOptions,
  IBaseRepositoryRemoveOptions,
  IBaseRepositoryUpdateOptions,
} from "../../interfaces/server/interfaces";
import {
  BaseRemoveAllDto,
  BaseRemoveDto,
} from "../../dto/server/base.remove.dto";
import { BaseFindDto } from "../../dto/server/base.find.dto";

export abstract class BaseRepository<
  TDocument extends Document<any, any, any>,
  TInterface extends IBaseRepositoryInterface,
  CreateDto = Partial<TDocument>,
  UpdateDto = any,
> {
  protected constructor(protected readonly model: Model<TDocument>) {}

  // ================================================
  // CREATE AND CREATE MANY FUNCTIONS
  // ================================================
  async create(
    data: CreateDto,
    opts?: IBaseRepositoryOptions,
  ): Promise<TInterface> {
    const virtuals = opts?.lean ?? true;
    const newData = new this.model(data);
    let saved = await newData.save();

    if (opts?.populate) {
      saved = await saved.populate(opts.populate);
    }

    return saved.toObject({
      versionKey: false,
      virtuals,
    }) as unknown as TInterface;
  }

  async createMany(
    data: CreateDto[],
    opts?: IBaseRepositoryOptions,
  ): Promise<TInterface[]> {
    const virtuals = opts?.lean ?? true;

    const inserted = await this.model.insertMany(data, {
      lean: true,
      ordered: false,
    });

    if (opts?.populate) {
      const ids = inserted.map((doc: any) => doc._id);
      const populated = await this.model
        .find({ _id: { $in: ids } })
        .populate(
          opts.populate as PopulateOptions | string[] | PopulateOptions[],
        )
        .lean({ virtuals, versionKey: false })
        .exec();

      return populated as TInterface[];
    }

    if (virtuals) {
      return inserted.map((doc: any) =>
        doc.toObject({ versionKey: false, virtuals }),
      ) as TInterface[];
    }

    return inserted as TInterface[];
  }

  // ================================================
  // UPDATE AND UPDATE MANY FUNCTIONS
  // ================================================
  async update(
    query: Record<string, any>,
    data: UpdateDto,
    opts?: IBaseRepositoryUpdateOptions,
  ): Promise<TInterface> {
    const updateOptions = {
      upsert: opts?.upsert ?? false,
      new: opts?.new ?? true,
      runValidators: opts?.runValidators ?? true,
      populate: opts?.populate ?? [],
      lean:
        (opts?.lean ?? true) ? { virtuals: true, versionKey: false } : false,
    };

    const updatedData = await this.model.findOneAndUpdate(
      query,
      data as UpdateQuery<TDocument>,
      updateOptions,
    );

    return updatedData as TInterface;
  }

  async updateMany(
    query: Record<string, any>,
    data: UpdateDto,
    opts?: IBaseRepositoryUpdateOptions,
  ): Promise<{
    matchedCount: number;
    modifiedCount: number;
    acknowledged: boolean;
  }> {
    const updateOptions = {
      upsert: opts?.upsert ?? false,
      runValidators: opts?.runValidators ?? true,
    };

    const result = await this.model.updateMany(
      query,
      data as UpdateQuery<TDocument>,
      updateOptions,
    );

    return {
      matchedCount: result.matchedCount,
      modifiedCount: result.modifiedCount,
      acknowledged: result.acknowledged,
    };
  }

  // ================================================
  // REMOVE AND REMOVE MANY FUNCTIONS
  // ================================================
  async remove(
    data: BaseRemoveDto,
    opts?: IBaseRepositoryRemoveOptions,
  ): Promise<TInterface> {
    if (opts?.lean) {
      return (await this.model
        .findByIdAndDelete(data._id)
        .lean()
        .exec()) as TInterface;
    }
    return (await this.model.findByIdAndDelete(data._id).exec()) as TInterface;
  }

  async removeOne(
    query: Record<string, any>,
    opts?: IBaseRepositoryRemoveOptions,
  ): Promise<TInterface> {
    if (opts?.lean) {
      return (await this.model
        .findOneAndDelete(query)
        .lean()
        .exec()) as TInterface;
    }
    return (await this.model.findOneAndDelete(query).exec()) as TInterface;
  }

  async removeMany(
    data: BaseRemoveAllDto,
    opts?: IBaseRepositoryRemoveOptions,
  ): Promise<TInterface[]> {
    if (opts?.lean) {
      return (await this.model
        .deleteMany({ _id: { $in: data._ids } })
        .lean()
        .exec()) as unknown as TInterface[];
    }

    return (await this.model
      .deleteMany({ _id: { $in: data._ids } })
      .exec()) as unknown as TInterface[];
  }

  async removeManyByQuery(
    query: Record<string, any>,
    opts?: IBaseRepositoryRemoveOptions,
  ): Promise<TInterface[]> {
    if (opts?.lean) {
      return (await this.model
        .deleteMany(query)
        .lean()
        .exec()) as unknown as TInterface[];
    }

    return (await this.model
      .deleteMany(query)
      .exec()) as unknown as TInterface[];
  }

  // ================================================
  // FINE ONE AND FIND MANY FUNCTIONS
  // ================================================
  async findOne(
    params: BaseFindDto = {},
    opts?: IBaseRepositoryOptions,
  ): Promise<TInterface | TDocument | null> {
    const {
      query = {},
      fields = "",
      sort = { createdAt: -1 },
      populate = [],
    } = params;
    const virtuals = opts?.lean ?? true;

    if (virtuals) {
      return (await this.model
        .findOne(query)
        .select(fields)
        .sort(sort)
        .populate(populate as PopulateOptions[])
        .lean({ virtuals, versionKey: false })
        .exec()) as TInterface | TDocument | null;
    }

    return (await this.model
      .findOne(query)
      .select(fields)
      .sort(sort)
      .populate(populate as PopulateOptions[])
      .exec()) as TInterface | TDocument | null;
  }

  async findOneById(
    _id: string,
    params: BaseFindDto = {},
    opts?: IBaseRepositoryOptions,
  ): Promise<TInterface | TDocument | null> {
    const { fields = "", sort = { createdAt: -1 }, populate = [] } = params;
    const virtuals = opts?.lean ?? true;

    if (virtuals) {
      return (await this.model
        .findOne({ _id })
        .select(fields)
        .sort(sort)
        .populate(populate as PopulateOptions[])
        .lean({ virtuals, versionKey: false })
        .exec()) as TInterface | TDocument | null;
    }

    return (await this.model
      .findOne({ _id })
      .select(fields)
      .sort(sort)
      .populate(populate as PopulateOptions[])
      .exec()) as TInterface | TDocument | null;
  }

  async findOneByUserId(
    userId: string,
    params: BaseFindDto = {},
    opts?: IBaseRepositoryOptions,
  ): Promise<TInterface | TDocument | null> {
    const { fields = "", sort = { createdAt: -1 }, populate = [] } = params;
    const virtuals = opts?.lean ?? true;

    if (virtuals) {
      return (await this.model
        .findOne({ userId })
        .select(fields)
        .sort(sort)
        .populate(populate as PopulateOptions[])
        .lean({ virtuals, versionKey: false })
        .exec()) as TInterface | TDocument | null;
    }

    return (await this.model
      .findOne({ userId })
      .select(fields)
      .sort(sort)
      .populate(populate as PopulateOptions[])
      .exec()) as TInterface | TDocument | null;
  }

  async findMany(
    params: BaseFindDto = {},
    opts?: IBaseRepositoryOptions,
  ): Promise<TInterface[] | TDocument[]> {
    const {
      query = {},
      fields = "",
      sort = { createdAt: -1 },
      populate = [],
      skip = 0,
      limit = 10,
    } = params;
    const virtuals = opts?.lean ?? true;

    if (virtuals) {
      return (await this.model
        .find(query)
        .select(fields)
        .skip(skip)
        .limit(limit)
        .sort(sort)
        .populate(populate as PopulateOptions[])
        .lean({ virtuals, versionKey: false })
        .exec()) as TInterface[] | TDocument[];
    }

    return (await this.model
      .find(query)
      .select(fields)
      .skip(skip)
      .limit(limit)
      .sort(sort)
      .populate(populate as PopulateOptions[])
      .exec()) as TInterface[] | TDocument[];
  }

  async findAll(
    params: BaseFindDto = {},
    opts?: IBaseRepositoryOptions,
  ): Promise<TInterface[] | TDocument[]> {
    const {
      query = {},
      fields = "",
      sort = { createdAt: -1 },
      populate = [],
    } = params;
    const virtuals = opts?.lean ?? true;

    if (virtuals) {
      return (await this.model
        .find(query)
        .select(fields)
        .sort(sort)
        .populate(populate as PopulateOptions[])
        .lean({ virtuals, versionKey: false })
        .exec()) as TInterface[] | TDocument[];
    }

    return (await this.model
      .find(query)
      .select(fields)
      .sort(sort)
      .populate(populate as PopulateOptions[])
      .exec()) as TInterface[] | TDocument[];
  }

  // ================================================
  // EXIST AND COUNT FUNCTIONS
  // ================================================
  async existsByField(query: Record<string, any>): Promise<boolean> {
    const result = await this.model.exists(query);
    return !!result;
  }

  async count(query: Record<string, any>): Promise<number> {
    return await this.model.countDocuments(query).exec();
  }

  // ================================================
  // AGGREGATE FUNCTION
  // ================================================
  async aggregate<R = any>(pipeline: PipelineStage[]): Promise<R[]> {
    return await this.model.aggregate(pipeline).exec();
  }
}

```

## File: connectfy-shared/src/constants/index.ts
```typescript
export * from "./server/constants";
export * from "./server/validation.messages";
export * from "./server/exception.messages";
export * from "./server/emial.messages";

```

## File: connectfy-shared/src/constants/server/emial.messages.ts
```typescript
import { LANGUAGE, TOKEN_TYPE } from "@/src/enums/enum";
import i18n from "@/src/i18n";

const supportMail = "connectfy.team@gmail.com";
const supportMailLink = `mailto:${supportMail}`;

export const signupVerifyMessage = (
  firstName: string,
  lastName: string,
  verifyCode: string,
  lang: LANGUAGE = LANGUAGE.EN,
) => {
  const team = i18n.t("email_messages.signup_verify.team", { lng: lang });
  const ignore = i18n.t("email_messages.signup_verify.ignore", { lng: lang });
  const support = i18n.t("email_messages.signup_verify.support", { lng: lang });
  const subject = i18n.t("email_messages.verify_yourself.subject", {
    lng: lang,
  });
  const greeting = i18n.t("email_messages.signup_verify.greeting", {
    lng: lang,
  });
  const thankYou = i18n.t("email_messages.signup_verify.thank_you", {
    lng: lang,
  });
  const signature = i18n.t("email_messages.signup_verify.signature", {
    lng: lang,
  });
  const instruction = i18n.t("email_messages.signup_verify.instruction", {
    lng: lang,
  });

  return `
    <html>
      <body style="font-family: 'Segoe UI', Arial, sans-serif; line-height: 1.6; color: #334155; background-color: #f8fafc; padding: 20px;">
        <div style="max-width: 500px; margin: 0 auto; background: #ffffff; padding: 32px; border-radius: 12px; border: 1px solid #e2e8f0; box-shadow: 0 4px 6px -1px rgba(0, 0, 0, 0.1);">
          
          <div style="text-align: center; margin-bottom: 24px;">
             <h2 style="color: #1e293b; margin: 0; font-size: 24px;">${subject}</h2>
          </div>

          <p style="font-size: 16px; margin-bottom: 8px;">${greeting} <strong>${firstName} ${lastName}</strong>,</p>
          <p style="font-size: 15px; color: #64748b; margin-top: 0;">${thankYou}</p>
          
          <p style="font-size: 15px; color: #334155;">${instruction}</p>
          
          <div style="text-align: center; margin: 32px 0;">
            <div style="display: inline-block; letter-spacing: 8px; font-size: 36px; font-weight: 800; color: #16a34a; background: #f0fdf4; padding: 16px 32px; border-radius: 12px; border: 2px dashed #bbf7d0;">
              ${verifyCode}
            </div>
          </div>

          <div style="background: #fffbeb; border-left: 4px solid #f59e0b; padding: 12px; margin-bottom: 24px;">
            <p style="font-size: 13px; color: #92400e; margin: 0;">${ignore}</p>
          </div>

          <p style="font-size: 14px; color: #64748b;">${support} 
            <a href="mailto:${supportMail}" style="color: #2ecc71; text-decoration: none; font-weight: 500;">${supportMail}</a>.
          </p>
          
          <hr style="border: 0; border-top: 1px solid #e2e8f0; margin: 24px 0;" />
          
          <p style="font-size: 13px; color: #94a3b8; margin: 0;">${signature}</p>
          <p style="font-size: 14px; font-weight: bold; color: #1e293b; margin: 4px 0 0 0;">${team}</p>
        </div>
      </body>
    </html>
  `;
};

export const forgotPasswordMessage = (
  resetToken: string,
  lang: LANGUAGE = LANGUAGE.EN,
) => {
  const resetUrl = `http://localhost:4800/auth/reset-password?token=${resetToken}`;

  const team = i18n.t("email_messages.forgot_password.team", { lng: lang });
  const intro = i18n.t("email_messages.forgot_password.intro", { lng: lang });
  const expiry = i18n.t("email_messages.forgot_password.expiry", { lng: lang });
  const support = i18n.t("email_messages.forgot_password.support", {
    lng: lang,
  });
  const subject = i18n.t("email_messages.forgot_password.subject", {
    lng: lang,
  });
  const buttonText = i18n.t("email_messages.forgot_password.button", {
    lng: lang,
  });
  const fallback = i18n.t("email_messages.forgot_password.fallback", {
    lng: lang,
  });
  const thankYou = i18n.t("email_messages.forgot_password.thank_you", {
    lng: lang,
  });

  return `
    <html>
      <body style="font-family: Arial, sans-serif; line-height: 1.6; color: #333; background-color: #f9f9f9; padding: 20px;">
        <div style="max-width: 600px; margin: 0 auto; background: #ffffff; padding: 20px; border-radius: 8px; box-shadow: 0 2px 10px rgba(0, 0, 0, 0.1);">
          <h2 style="text-align: center; color: #2ecc71;">${subject}</h2>
          <p>${intro}</p>

          <div style="text-align: center; margin: 20px 0;">
            <a href="${resetUrl}"
               style="display: inline-block; font-size: 18px; font-weight: bold; color: #ffffff; background-color: #2ecc71; padding: 12px 24px; text-decoration: none; border-radius: 4px;">
              ${buttonText}
            </a>
          </div>

          <p>${fallback}</p>
          <a href="${resetUrl}" style="word-wrap: break-word; color: #2ecc71;">${resetUrl}</a>

          <p style="color: #777; font-size: 14px;">${expiry}</p>

          <p>${support} 
            <a href="${supportMailLink}" style="color: #2ecc71; text-decoration: none;">${supportMail}</a>.
          </p>

          <p>${thankYou}</p>
          <p style="text-align: center; font-weight: bold; color: #2ecc71;">${team}</p>
        </div>
      </body>
    </html>
  `;
};

export const emailNotFoundMessage = (
  email: string,
  lang: LANGUAGE = LANGUAGE.EN,
) => {
  const registerUrl = `http://localhost:4800/auth`;

  const team = i18n.t("email_messages.email_not_found.team", { lng: lang });
  const intro = i18n.t("email_messages.email_not_found.intro", { lng: lang });
  const subject = i18n.t("email_messages.email_not_found.subject", {
    lng: lang,
  });
  const support = i18n.t("email_messages.email_not_found.support", {
    lng: lang,
  });
  const buttonText = i18n.t("email_messages.email_not_found.button", {
    lng: lang,
  });
  const thankYou = i18n.t("email_messages.email_not_found.thank_you", {
    lng: lang,
  });
  const instruction = i18n.t("email_messages.email_not_found.instruction", {
    lng: lang,
  });

  return `
    <html>
      <body style="font-family: Arial, sans-serif; line-height: 1.6; color: #333; background-color: #f9f9f9; padding: 20px;">
        <div style="max-width: 600px; margin: 0 auto; background: #ffffff; padding: 20px; border-radius: 8px; box-shadow: 0 2px 10px rgba(0, 0, 0, 0.1);">
          <h2 style="text-align: center; color: #FF5722;">${subject}</h2>
          <p>${intro} <strong>${email}</strong>.</p>
          <p>${instruction}</p>

          <div style="text-align: center; margin: 20px 0;">
            <a href="${registerUrl}"
               style="display: inline-block; font-size: 18px; font-weight: bold; color: #ffffff; background-color: #FF5722; padding: 12px 24px; text-decoration: none; border-radius: 4px;">
              ${buttonText}
            </a>
          </div>

          <p>${support} 
            <a href="${supportMailLink}" style="color: #2ecc71; text-decoration: none;">${supportMail}</a>.
          </p>

          <p>${thankYou}</p>
          <p style="text-align: center; font-weight: bold; color: #2ecc71;">${team}</p>
        </div>
      </body>
    </html>
  `;
};

export const googleSignInMessage = (
  email: string,
  lang: LANGUAGE = LANGUAGE.EN,
) => {
  const loginUrl = `http://localhost:4800/auth`;

  const team = i18n.t("email_messages.google_sign_in.team", { lng: lang });
  const intro = i18n.t("email_messages.google_sign_in.intro", { lng: lang });
  const ignore = i18n.t("email_messages.google_sign_in.ignore", { lng: lang });
  const subject = i18n.t("email_messages.google_sign_in.subject", {
    lng: lang,
  });
  const support = i18n.t("email_messages.google_sign_in.support", {
    lng: lang,
  });
  const buttonText = i18n.t("email_messages.google_sign_in.button", {
    lng: lang,
  });
  const thankYou = i18n.t("email_messages.google_sign_in.thank_you", {
    lng: lang,
  });
  const instruction = i18n.t("email_messages.google_sign_in.instruction", {
    lng: lang,
  });

  return `
    <html>
      <body style="font-family: Arial, sans-serif; line-height: 1.6; color: #333; background-color: #f9f9f9; padding: 20px;">
        <div style="max-width: 600px; margin: 0 auto; background: #ffffff; padding: 20px; border-radius: 8px; box-shadow: 0 2px 10px rgba(0, 0, 0, 0.1);">
          <h2 style="text-align: center; color: #FF9800;">${subject}</h2>
          <p>${intro}</p>
          <p><strong>${email}</strong></p>
          <p>${instruction}</p>

          <div style="text-align: center; margin: 20px 0;">
            <a href="${loginUrl}"
               style="display: inline-block; font-size: 18px; font-weight: bold; color: #ffffff; background-color: #FF9800; padding: 12px 24px; text-decoration: none; border-radius: 4px;">
              ${buttonText}
            </a>
          </div>

          <p>${ignore}</p>

          <p>${support} 
            <a href="${supportMailLink}" style="color: #2ecc71; text-decoration: none;">${supportMail}</a>.
          </p>

          <p>${thankYou}</p>
          <p style="text-align: center; font-weight: bold; color: #2ecc71;">${team}</p>
        </div>
      </body>
    </html>
  `;
};

export const deleteAccountMessage = (
  deleteToken: string,
  lang: LANGUAGE = LANGUAGE.EN,
) => {
  const deleteUrl = `http://localhost:4800/auth/delete-account?token=${deleteToken}`;

  const team = i18n.t("email_messages.delete_account.team", { lng: lang });
  const intro = i18n.t("email_messages.delete_account.intro", { lng: lang });
  const expiry = i18n.t("email_messages.delete_account.expiry", { lng: lang });
  const support = i18n.t("email_messages.delete_account.support", {
    lng: lang,
  });
  const subject = i18n.t("email_messages.delete_account.subject", {
    lng: lang,
  });
  const buttonText = i18n.t("email_messages.delete_account.button", {
    lng: lang,
  });
  const fallback = i18n.t("email_messages.delete_account.fallback", {
    lng: lang,
  });
  const thankYou = i18n.t("email_messages.delete_account.thank_you", {
    lng: lang,
  });

  return `
    <html>
      <body style="font-family: Arial, sans-serif; line-height: 1.6; color: #333; background-color: #f9f9f9; padding: 20px;">
        <div style="max-width: 600px; margin: 0 auto; background: #ffffff; padding: 20px; border-radius: 8px; box-shadow: 0 2px 10px rgba(0, 0, 0, 0.1);">
          <h2 style="text-align: center; color: #f44336;">${subject}</h2>
          <p>${intro}</p>

          <div style="text-align: center; margin: 20px 0;">
            <a href="${deleteUrl}"
               style="display: inline-block; font-size: 18px; font-weight: bold; color: #ffffff; background-color: #f44336; padding: 12px 24px; text-decoration: none; border-radius: 4px;">
              ${buttonText}
            </a>
          </div>

          <p>${fallback}</p>
          <a href="${deleteUrl}" style="word-wrap: break-word; color: #f44336;">${deleteUrl}</a>

          <p style="color: #777; font-size: 14px;">${expiry}</p>

          <p>${support} 
            <a href="${supportMailLink}" style="color: #f44336; text-decoration: none;">${supportMail}</a>.
          </p>

          <p>${thankYou}</p>
          <p style="text-align: center; font-weight: bold; color: #2ecc71;">${team}</p>
        </div>
      </body>
    </html>
  `;
};

export const accountDeletedMessage = (
  restoreToken: string,
  lang: LANGUAGE = LANGUAGE.EN,
) => {
  const restoreUrl = `http://localhost:4800/auth?type=restore&token=${restoreToken}`;

  const team = i18n.t("email_messages.delete_account_completed.team", {
    lng: lang,
  });
  const note = i18n.t("email_messages.delete_account_completed.note", {
    lng: lang,
  });
  const intro = i18n.t("email_messages.delete_account_completed.intro", {
    lng: lang,
  });
  const support = i18n.t("email_messages.delete_account_completed.support", {
    lng: lang,
  });
  const subject = i18n.t("email_messages.delete_account_completed.subject", {
    lng: lang,
  });
  const buttonText = i18n.t("email_messages.delete_account_completed.button", {
    lng: lang,
  });
  const fallback = i18n.t("email_messages.delete_account_completed.fallback", {
    lng: lang,
  });
  const signature = i18n.t(
    "email_messages.delete_account_completed.signature",
    { lng: lang },
  );
  const optionalRestore = i18n.t(
    "email_messages.delete_account_completed.optional_restore",
    { lng: lang },
  );

  return `
    <html>
      <body style="font-family: Arial, sans-serif; line-height: 1.6; color: #333; background-color: #f9f9f9; padding: 20px;">
        <div style="max-width: 600px; margin: 0 auto; background: #ffffff; padding: 20px; border-radius: 8px; box-shadow: 0 2px 10px rgba(0, 0, 0, 0.1);">
          <h2 style="text-align: center; color: #f44336;">${subject}</h2>
          <p>${intro}</p>
          <p>${optionalRestore}</p>

          <div style="text-align: center; margin: 20px 0;">
            <a href="${restoreUrl}"
               style="display: inline-block; font-size: 18px; font-weight: bold; color: #ffffff; background-color: #4CAF50; padding: 12px 24px; text-decoration: none; border-radius: 4px;">
              ${buttonText}
            </a>
          </div>

          <p>${fallback}</p>
          <a href="${restoreUrl}" style="word-wrap: break-word; color: #4CAF50;">${restoreUrl}</a>

          <p style="color: #777; font-size: 14px;">${note}</p>

          <p>${support} 
            <a href="${supportMailLink}" style="color: #2ecc71; text-decoration: none;">${supportMail}</a>.
          </p>

          <p>${signature}</p>
          <p style="text-align: center; font-weight: bold; color: #2ecc71;">${team}</p>
        </div>
      </body>
    </html>
  `;
};

export const verifyYourselfMessage = (
  firstName: string,
  lastName: string,
  verifyCode: string,
  lang: LANGUAGE = LANGUAGE.EN,
) => {
  const team = i18n.t("email_messages.verify_yourself.team", { lng: lang });
  const subject = i18n.t("email_messages.verify_yourself.subject", {
    lng: lang,
  });
  const support = i18n.t("email_messages.verify_yourself.support", {
    lng: lang,
  });
  const greeting = i18n.t("email_messages.verify_yourself.greeting", {
    lng: lang,
  });
  const codeInfo = i18n.t("email_messages.verify_yourself.code_info", {
    lng: lang,
  });
  const signature = i18n.t("email_messages.verify_yourself.signature", {
    lng: lang,
  });
  const instruction = i18n.t("email_messages.verify_yourself.instruction", {
    lng: lang,
  });

  return `
    <html>
      <body style="font-family: Arial, sans-serif; line-height: 1.6; color: #333; background-color: #f9f9f9; padding: 20px;">
        <div style="max-width: 600px; margin: 0 auto; background: #ffffff; padding: 20px; border-radius: 8px; box-shadow: 0 2px 10px rgba(0, 0, 0, 0.1);">
          <h2 style="text-align: center; color: #2ecc71;">${subject}</h2>
          <p>${greeting} <strong>${firstName} ${lastName}</strong>,</p>
          <p>${codeInfo}</p>

          <div style="text-align: center; font-size: 24px; font-weight: bold; color: #2ecc71; background: #f1f8ff; padding: 10px; border-radius: 5px; margin: 20px 0;">
            ${verifyCode}
          </div>

          <p>${instruction}</p>

          <p>${support} 
            <a href="${supportMailLink}" style="color: #2ecc71; text-decoration: none;">${supportMail}</a>.
          </p>

          <p>${signature}</p>
          <p style="text-align: center; font-weight: bold; color: #2ecc71;">${team}</p>
        </div>
      </body>
    </html>
  `;
};

export const changeEmailMessage = (
  token: string,
  lang: LANGUAGE = LANGUAGE.EN,
) => {
  const changeUrl = `http://localhost:4800/settings/account?action=${TOKEN_TYPE.CHANGE_EMAIL}&token=${token}`;

  const team = i18n.t("email_messages.change_email.team", { lng: lang });
  const intro = i18n.t("email_messages.change_email.intro", { lng: lang });
  const subject = i18n.t("email_messages.change_email.subject", { lng: lang });
  const support = i18n.t("email_messages.change_email.support", { lng: lang });
  const buttonText = i18n.t("email_messages.change_email.button", {
    lng: lang,
  });
  const fallback = i18n.t("email_messages.change_email.fallback", {
    lng: lang,
  });
  const thankYou = i18n.t("email_messages.change_email.thank_you", {
    lng: lang,
  });
  const instruction = i18n.t("email_messages.change_email.instruction", {
    lng: lang,
  });
  const expiryNotice = i18n.t("email_messages.change_email.expiry", {
    lng: lang,
  }); // New expiry message

  return `
    <html>
      <body style="font-family: Arial, sans-serif; line-height: 1.6; color: #333; background-color: #f9f9f9; padding: 20px;">
        <div style="max-width: 600px; margin: 0 auto; background: #ffffff; padding: 20px; border-radius: 8px; box-shadow: 0 2px 10px rgba(0, 0, 0, 0.1);">
          <h2 style="text-align: center; color: #FF9800;">${subject}</h2>
          <p>${intro}</p>
          <p>${instruction}</p>

          <div style="text-align: center; margin: 20px 0;">
            <a href="${changeUrl}"
               style="display: inline-block; font-size: 18px; font-weight: bold; color: #ffffff; background-color: #FF9800; padding: 12px 24px; text-decoration: none; border-radius: 4px;">
              ${buttonText}
            </a>
          </div>

          <p>${expiryNotice}</p> <!-- Expiry notice here -->

          <p>${fallback}</p>
          <a href="${changeUrl}" style="word-wrap: break-word; color: #FF9800;">${changeUrl}</a>

          <p>${support} 
            <a href="mailto:${supportMail}" style="color: #FF9800; text-decoration: none;">${supportMail}</a>.
          </p>

          <p>${thankYou}</p>
          <p style="text-align: center; font-weight: bold; color: #2ecc71;">${team}</p>
        </div>
      </body>
    </html>
  `;
};

export const twoFactorVerifyMessage = (
  firstName: string,
  lastName: string,
  verifyCode: string,
  lang: LANGUAGE = LANGUAGE.EN,
) => {
  const team = i18n.t("email_messages.two_factor.team", { lng: lang });
  const warning = i18n.t("email_messages.two_factor.warning", { lng: lang });
  const support = i18n.t("email_messages.two_factor.support", { lng: lang });
  const subject = i18n.t("email_messages.two_factor.subject", { lng: lang });
  const greeting = i18n.t("email_messages.two_factor.greeting", { lng: lang });
  const instruction = i18n.t("email_messages.two_factor.instruction", {
    lng: lang,
  });
  const signature = i18n.t("email_messages.two_factor.signature", {
    lng: lang,
  });

  return `
    <html>
      <body style="font-family: 'Segoe UI', Arial, sans-serif; line-height: 1.6; color: #334155; background-color: #f8fafc; padding: 20px;">
        <div style="max-width: 500px; margin: 0 auto; background: #ffffff; padding: 32px; border-radius: 12px; border: 1px solid #e2e8f0; box-shadow: 0 4px 6px -1px rgba(0, 0, 0, 0.1);">
          <div style="text-align: center; margin-bottom: 24px;">
             <h2 style="color: #1e293b; margin: 0; font-size: 24px;">${subject}</h2>
          </div>
          <p style="font-size: 16px;">${greeting} <strong>${firstName} ${lastName}</strong>,</p>
          <p style="font-size: 15px; color: #64748b;">${instruction}</p>
          
          <div style="text-align: center; margin: 32px 0;">
            <div style="display: inline-block; letter-spacing: 8px; font-size: 36px; font-weight: 800; color: #16a34a; background: #f0fdf4; padding: 16px 32px; border-radius: 12px; border: 2px dashed #bbf7d0;">
              ${verifyCode}
            </div>
          </div>

          <div style="background: #fffbeb; border-left: 4px solid #f59e0b; padding: 12px; margin-bottom: 24px;">
            <p style="font-size: 13px; color: #92400e; margin: 0;">${warning}</p>
          </div>

          <p style="font-size: 14px; color: #64748b;">${support} 
            <a href="${supportMailLink}" style="color: #2ecc71; text-decoration: none; font-weight: 500;">${supportMail}</a>.
          </p>
          
          <hr style="border: 0; border-top: 1px solid #e2e8f0; margin: 24px 0;" />
          
          <p style="font-size: 13px; color: #94a3b8; margin: 0;">${signature}</p>
          <p style="font-size: 14px; font-weight: bold; color: #1e293b; margin: 4px 0 0 0;">${team}</p>
        </div>
      </body>
    </html>
  `;
};

```

## File: connectfy-shared/src/constants/server/constants.ts
```typescript
import { FileOwnerModule, FileUploadTopic } from "@/src/enums";
import { ICountry } from "@/src/interfaces/server/interfaces";

export const MICROSERVICE_NAMES = {
  KAFKA: "KAFKA_CONNECTIONS",
  TCP: {
    ACCOUNT: "ACCOUNT_SERVICE_TCP",
    AUTH: "AUTH_SERVICE_TCP",
    MESSENGER: "MESSENGER_SERVICE_TCP",
    RELATIONSHIP: "RELATIONSHIP_SERVICE_TCP",
    NOTIFICATION_ACTION_HISTORY: "NOTIFICATION_ACTION_HISTORY_SERVICE_TCP",
  },
};

export const COLLECTIONS = {
  AUTH: {
    USER: {
      USERS: "users",
      BANNED: "banned_users",
      DELETED: "deleted_users",
      DEACTIVATED: "deactivated_users",
    },
    TOKEN: {
      TOKENS: "tokens",
      REFRESH_TOKENS: "refresh_tokens",
    },
  },
  ACCOUNT: {
    PROFILES: "profiles",
    SOCIAL_LINKS: "social_links",
    SETTINGS: {
      GENERAL: "general_settings",
      PRIVACY: "privacy_settings",
      NOTIFICATION: "notification_settings",
    },
  },
  RELATIONSHIP: {
    FRIENDSHIPS: "friendships",
    BLOCKS_LIST: "blocks_list",
  },
  NOTIFICATION: {
    NOTIFICATIONS: "notifications",
    DEVICE_TOKENS: "device_tokens",
  },
};

export const EXPIRE_DATES = {
  JWT: {
    ONE_HOUR: "1h",
    ONE_DAY: "1d",
    ONE_MONTH: "30d",
  },
  TOKEN: {
    ONE_MINUTE: 60 * 1000,
    ONE_HOUR: 60 * 60 * 1000,
    ONE_DAY: 24 * 60 * 60 * 1000,
    ONE_MONTH: 30 * 60 * 60 * 1000,
  },
  TTL: {
    ONE_MINUTE: 60,
    ONE_HOUR: 60 * 60,
    ONE_DAY: 24 * 60 * 60,
    ONE_MONTH: 30 * 60 * 60,
  },
};

export const CACHE_KEYS = {
  AUTH: {
    USER: (id: string) => `user:${id}`,
    ACCESS_TOKEN: (token: string) => `access_token:${token}`,
  },
  ACCOUNT: {
    PROFILE: (userId: string) => `profile:${userId}`,
    SETTINGS: {
      GENERAL: (userId: string) => `general_settings:${userId}`,
      NOTIFICATION: (userId: string) => `notification_settings:${userId}`,
      PRIVACY: (userId: string) => `privacy_settings:${userId}`,
    },
    SOCIAL_LINK: (userId: string) => `social_link:${userId}`,
  },
};

export const COUNTRIES: ICountry[] = [
  {
    key: "az",
    name: "azerbaijan",
    flag: "fi fi-az",
    code: "+994",
    numberLength: 9,
  },
  {
    key: "tr",
    name: "türkiye",
    flag: "fi fi-tr",
    code: "+90",
    numberLength: 10,
  },
  { key: "ru", name: "russia", flag: "fi fi-ru", code: "+7", numberLength: 10 },
  { key: "us", name: "usa", flag: "fi fi-us", code: "+1", numberLength: 10 },
  { key: "gb", name: "uk", flag: "fi fi-gb", code: "+44", numberLength: 10 },
  {
    key: "de",
    name: "germany",
    flag: "fi fi-de",
    code: "+49",
    numberLength: 10,
  },
  { key: "fr", name: "france", flag: "fi fi-fr", code: "+33", numberLength: 9 },
  { key: "es", name: "spain", flag: "fi fi-es", code: "+34", numberLength: 9 },
  { key: "it", name: "italy", flag: "fi fi-it", code: "+39", numberLength: 10 },
  { key: "cn", name: "china", flag: "fi fi-cn", code: "+86", numberLength: 11 },
  { key: "jp", name: "japan", flag: "fi fi-jp", code: "+81", numberLength: 10 },
  {
    key: "kr",
    name: "south_korea",
    flag: "fi fi-kr",
    code: "+82",
    numberLength: 10,
  },
  {
    key: "ua",
    name: "ukraine",
    flag: "fi fi-ua",
    code: "+380",
    numberLength: 9,
  },
  {
    key: "ge",
    name: "georgia",
    flag: "fi fi-ge",
    code: "+995",
    numberLength: 9,
  },
  { key: "ir", name: "iran", flag: "fi fi-ir", code: "+98", numberLength: 10 },
  { key: "in", name: "india", flag: "fi fi-in", code: "+91", numberLength: 10 },
  { key: "ca", name: "canada", flag: "fi fi-ca", code: "+1", numberLength: 10 },
  {
    key: "kz",
    name: "kazakhstan",
    flag: "fi fi-kz",
    code: "+7",
    numberLength: 10,
  },
  {
    key: "uz",
    name: "uzbekistan",
    flag: "fi fi-uz",
    code: "+998",
    numberLength: 9,
  },
  { key: "ae", name: "uae", flag: "fi fi-ae", code: "+971", numberLength: 9 },
];

export const MODULE_TO_TOPIC_MAP: Record<FileOwnerModule, FileUploadTopic> = {
  [FileOwnerModule.PROFILE_PHOTO]: FileUploadTopic.PROFILE_PHOTO_UPLOADED,
};

```

## File: connectfy-shared/src/constants/server/validation.messages.ts
```typescript
import i18n from "@/src/i18n";
import { ValidationArguments } from "class-validator";
import { ClsServiceManager } from "nestjs-cls";
import { IValidateMessageOptions, CLS_KEYS, LANGUAGE } from "@/src";

export const getLang = (args: ValidationArguments): LANGUAGE => {
  // 1.️CLS
  try {
    const cls = ClsServiceManager.getClsService();
    const langInCls = cls?.get<LANGUAGE | undefined | null>(CLS_KEYS.LANG);
    if (langInCls) return langInCls;
  } catch {
    // CLS context yoxdursa → ignore
  }

  // 2. _lang inside DTO
  const dto = args?.object as any;
  const langInDto: LANGUAGE | undefined | null = dto?.lang;
  if (langInDto) return langInDto;

  // 3. Language inside _loggedUser
  const userLang: LANGUAGE | undefined =
    dto?._loggedUser?.settings?.generalSettings?.language;
  if (userLang) return userLang;

  // 4️. Final (Default)
  return LANGUAGE.EN;
};

export const validationMessage = ({
  args,
  type,
  params = {},
}: IValidateMessageOptions): string => {
  return i18n.t(`validation_messages.${type}`, {
    lng: getLang(args),
    field: args.property,
    ...params,
  }) as string;
};

```

## File: connectfy-shared/src/constants/server/exception.messages.ts
```typescript
import { LANGUAGE } from "@/src";
import i18n from "@/src/i18n";

export const ExceptionMessages = {
  INTERNAL_SERVER_ERROR_MESSAGE: (lang: LANGUAGE = LANGUAGE.EN) =>
    i18n.t("exception_messages.internal_server_error_message", { lng: lang }),

  BAD_REQUEST_MESSAGE: (lang: LANGUAGE = LANGUAGE.EN) =>
    i18n.t("exception_messages.bad_request_message", { lng: lang }),

  UNAUTHORIZED_MESSAGE: (lang: LANGUAGE = LANGUAGE.EN) =>
    i18n.t("exception_messages.unauthorized_message", { lng: lang }),

  FORBIDDEN_MESSAGE: (lang: LANGUAGE = LANGUAGE.EN) =>
    i18n.t("exception_messages.forbidden_message", { lng: lang }),

  NOT_FOUND_MESSAGE: (lang: LANGUAGE = LANGUAGE.EN) =>
    i18n.t("exception_messages.not_found_message", { lng: lang }),

  CONFLICT_MESSAGE: (lang: LANGUAGE = LANGUAGE.EN) =>
    i18n.t("exception_messages.conflict_message", { lng: lang }),

  UNPROCESSABLE_ENTITY_MESSAGE: (lang: LANGUAGE = LANGUAGE.EN) =>
    i18n.t("exception_messages.unprocessable_entity_message", { lng: lang }),

  GONE_MESSAGE: (lang: LANGUAGE = LANGUAGE.EN) =>
    i18n.t("exception_messages.gone_message", { lng: lang }),

  TOO_MANY_REQUESTS_MESSAGE: (lang: LANGUAGE = LANGUAGE.EN) =>
    i18n.t("exception_messages.too_many_requests_message", { lng: lang }),

  ACCESS_TOKEN_EXPIRED: (lang: LANGUAGE = LANGUAGE.EN) =>
    i18n.t("exception_messages.access_token_expired", { lng: lang }),

  TOKEN_EXPIRED: (lang: LANGUAGE = LANGUAGE.EN) =>
    i18n.t("exception_messages.token_expired", { lng: lang }),

  ALREADY_EXISTS_MESSAGE: (field: string, lang: LANGUAGE = LANGUAGE.EN) =>
    i18n.t("exception_messages.already_exists_message", { field, lng: lang }),

  INVALID_CREDENTIALS: (lang: LANGUAGE = LANGUAGE.EN) =>
    i18n.t("exception_messages.invalid_credentials", { lng: lang }),

  BANNED_MESSAGE: (bannedToDate: Date, lang: LANGUAGE = LANGUAGE.EN) =>
    i18n.t("exception_messages.banned_message", {
      bannedToDate: bannedToDate?.toISOString?.() ?? String(bannedToDate),
      lng: lang,
    }),

  SAME_DATA: (field: string, lang: LANGUAGE = LANGUAGE.EN) =>
    i18n.t("exception_messages.same_data", { field, lng: lang }),

  STRING_OR_NULL_MESSAGE: (field: string, lang: LANGUAGE = LANGUAGE.EN) =>
    i18n.t("exception_messages.string_or_null", { field, lng: lang }),

  NUMBER_OR_NULL_MESSAGE: (field: string, lang: LANGUAGE = LANGUAGE.EN) =>
    i18n.t("exception_messages.number_or_null", { field, lng: lang }),

  ARRAY_TYPE_MESSAGE: (field: string, lang: LANGUAGE = LANGUAGE.EN) =>
    i18n.t("exception_messages.array_type", { field, lng: lang }),

  ARRAY_MIN_ONE_MESSAGE: (field: string, lang: LANGUAGE = LANGUAGE.EN) =>
    i18n.t("exception_messages.array_min_one", { field, lng: lang }),

  OBJECT_TYPE_MESSAGE: (field: string, lang: LANGUAGE = LANGUAGE.EN) =>
    i18n.t("exception_messages.object_type", { field, lng: lang }),

  OBJECT_MIN_ONE_PROPERTY_MESSAGE: (
    field: string,
    lang: LANGUAGE = LANGUAGE.EN,
  ) =>
    i18n.t("exception_messages.object_min_one_property", {
      field,
      lng: lang,
    }),

  DATE_OR_NULL_MESSAGE: (field: string, lang: LANGUAGE = LANGUAGE.EN) =>
    i18n.t("exception_messages.date_or_null", { field, lng: lang }),

  BOOLEAN_TYPE_MESSAGE: (field: string, lang: LANGUAGE = LANGUAGE.EN) =>
    i18n.t("exception_messages.boolean_type", { field, lng: lang }),

  BOOLEAN_INVALID_MESSAGE: (field: string, lang: LANGUAGE = LANGUAGE.EN) =>
    i18n.t("exception_messages.boolean_invalid", {
      field,
      lng: lang,
    }),

  ENUM_INVALID_MESSAGE: (
    field: string,
    validValues: any[],
    lang: LANGUAGE = LANGUAGE.EN,
  ) =>
    i18n.t("exception_messages.enum_invalid", {
      field,
      validValues: validValues.join(", "),
      lng: lang,
    }),

  INVALID_LENGTH_MESSAGE: (field: string, lang: LANGUAGE = LANGUAGE.EN) =>
    i18n.t("exception_messages.invalid_length", { field, lng: lang }),

  LIMIT_REACHED_MESSAGE: (limit: number, lang: LANGUAGE = LANGUAGE.EN) =>
    i18n.t("exception_messages.limit_reached", { limit, lng: lang }),
};

```

## File: connectfy-shared/src/functions/index.ts
```typescript
export * from "./server/transform";
export * from "./server/crypto";
export * from "./server/microservice-request-helper";
export * from "./server/kafka-commit.helper";
export * from "./server/privacy-settings.helper";

```

## File: connectfy-shared/src/functions/server/microservice-request-helper.ts
```typescript
import { lastValueFrom } from "rxjs";
import { ClsServiceManager } from "nestjs-cls";
import {
  IEmitTcpWithContextClientParams,
  IEmitWithContextClientParams,
  ISendWithContextClientParams,
} from "../../interfaces/server/interfaces";

/**
 * Request/response wrapper
 */
export async function sendWithContext<TResponse = any>({
  client,
  endpoint,
  payload = {},
}: ISendWithContextClientParams): Promise<TResponse> {
  const cls = ClsServiceManager.getClsService();
  const user = cls?.get("user") || null;

  const enrichedPayload = {
    ...payload,
    _loggedUser: user,
  };

  return await lastValueFrom(client.send(endpoint, enrichedPayload));
}

/**
 * Send event with context
 */
export async function emitWithContextTcp({
  client,
  endpoint,
  payload = {},
}: IEmitTcpWithContextClientParams): Promise<void> {
  const cls = ClsServiceManager.getClsService();
  const user = cls?.get("user") || null;

  const enrichedPayload = {
    ...payload,
    _loggedUser: user,
  };

  client.emit(endpoint, enrichedPayload);
}

/**
 * Fire-and-forget event wrapper (Kafka emit, RMQ emit, etc.)
 * Automatically attaches `_loggedUser` from CLS.
 */
export function emitWithContext({
  client,
  topic,
  payload = {},
}: IEmitWithContextClientParams) {
  const cls = ClsServiceManager.getClsService();
  const user = cls?.get("user") || null;

  const enrichedPayload = {
    ...payload,
    _loggedUser: user,
  };

  return client.emit(topic, enrichedPayload);
}

```

## File: connectfy-shared/src/functions/server/privacy-settings.helper.ts
```typescript
import { PRIVACY_SETTINGS_CHOICE } from "../../enums";

export function shouldShowField(
  setting: PRIVACY_SETTINGS_CHOICE | undefined,
  isFriend: boolean,
  isOwnProfile: boolean,
): boolean {
  // 1. Öz profilidirsə hər şeyi görə bilər
  if (isOwnProfile) return true;

  // 2. Setting yoxdursa və ya EVERYONE-dırsa icazə ver
  if (!setting || setting === PRIVACY_SETTINGS_CHOICE.EVERYONE) return true;

  // 3. Yalnız dostlar üçündürsə, dost olub-olmadığını yoxla
  if (setting === PRIVACY_SETTINGS_CHOICE.MY_FRIENDS) return isFriend;

  // 4. NOBODY
  return false;
}

```

## File: connectfy-shared/src/functions/server/transform.ts
```typescript
import { ExceptionMessages, BaseException, HttpStatus } from "@/src";

export function stringTransform({ value, key }: { value: any; key: string }) {
  if (value === undefined) return null;
  if (value === null) return null;
  if (typeof value !== "string") {
    throw new BaseException(
      (lang) => ExceptionMessages.STRING_OR_NULL_MESSAGE(key, lang),
      HttpStatus.BAD_REQUEST,
    );
  }
  const trimmedValue = value.trim();
  if (trimmedValue === "") {
    return null;
  }
  return trimmedValue;
}

export function numberTransform({ value, key }: { value: any; key: string }) {
  if (value === undefined || value === null) {
    return null;
  }

  if (typeof value === "number") {
    if (isNaN(value)) {
      throw new BaseException(
        (lang) => ExceptionMessages.NUMBER_OR_NULL_MESSAGE(key, lang),
        HttpStatus.BAD_REQUEST,
      );
    }
    return value;
  }

  if (typeof value === "string") {
    const trimmedValue = value.trim();
    if (trimmedValue === "") return null;
    const num = Number(trimmedValue);
    if (isNaN(num)) {
      throw new BaseException(
        (lang) => ExceptionMessages.NUMBER_OR_NULL_MESSAGE(key, lang),
        HttpStatus.BAD_REQUEST,
      );
    }
    return num;
  }

  throw new BaseException(
    (lang) => ExceptionMessages.NUMBER_OR_NULL_MESSAGE(key, lang),
    HttpStatus.BAD_REQUEST,
  );
}

export function arrayTransform({ value, key }: { value: any; key: string }) {
  if (value === undefined || value === null) {
    return null;
  }

  if (!Array.isArray(value)) {
    throw new BaseException(
      (lang) => ExceptionMessages.ARRAY_TYPE_MESSAGE(key, lang),
      HttpStatus.BAD_REQUEST,
    );
  }

  if (value.length === 0) {
    throw new BaseException(
      (lang) => ExceptionMessages.ARRAY_MIN_ONE_MESSAGE(key, lang),
      HttpStatus.BAD_REQUEST,
    );
  }

  return value;
}

export function objectTransform({ value, key }: { value: any; key: string }) {
  if (value === undefined || value === null) {
    return null;
  }

  if (typeof value !== "object" || Array.isArray(value)) {
    throw new BaseException(
      (lang) => ExceptionMessages.OBJECT_TYPE_MESSAGE(key, lang),
      HttpStatus.BAD_REQUEST,
    );
  }

  if (Object.keys(value).length === 0) {
    throw new BaseException(
      (lang) => ExceptionMessages.OBJECT_MIN_ONE_PROPERTY_MESSAGE(key, lang),
      HttpStatus.BAD_REQUEST,
    );
  }

  return value;
}

export function dateTransform({ value, key }: { value: any; key: string }) {
  if (value === undefined || value === null) {
    return null;
  }

  if (value instanceof Date) {
    if (isNaN(value.getTime())) {
      throw new BaseException(
        (lang) => ExceptionMessages.DATE_OR_NULL_MESSAGE(key, lang),
        HttpStatus.BAD_REQUEST,
      );
    }
    return value;
  }

  const parsedValue = !isNaN(Number(value));

  if (parsedValue) {
    throw new BaseException(
      (lang) => ExceptionMessages.DATE_OR_NULL_MESSAGE(key, lang),
      HttpStatus.BAD_REQUEST,
    );
  }

  if (typeof value === "string") {
    const trimmed = value.trim();
    if (trimmed === "") return null;

    if (!isNaN(Number(trimmed))) {
      throw new BaseException(
        (lang) => ExceptionMessages.DATE_OR_NULL_MESSAGE(key, lang),
        HttpStatus.BAD_REQUEST,
      );
    }

    const date = new Date(trimmed);

    if (isNaN(date.getTime())) {
      throw new BaseException(
        (lang) => ExceptionMessages.DATE_OR_NULL_MESSAGE(key, lang),
        HttpStatus.BAD_REQUEST,
      );
    }

    return date;
  }

  throw new BaseException(
    (lang) => ExceptionMessages.DATE_OR_NULL_MESSAGE(key, lang),
    HttpStatus.BAD_REQUEST,
  );
}

export function booleanTransform({ value, key }: { value: any; key: string }) {
  if (value === undefined || value === null) {
    throw new BaseException(
      (lang) => ExceptionMessages.BOOLEAN_INVALID_MESSAGE(key, lang),
      HttpStatus.BAD_REQUEST,
    );
  }

  if (typeof value === "boolean") {
    return value;
  }

  if (typeof value === "string") {
    const trimmed = value.trim().toLowerCase();
    if (trimmed === "true") return true;
    if (trimmed === "false") return false;

    throw new BaseException(
      (lang) => ExceptionMessages.BOOLEAN_INVALID_MESSAGE(key, lang),
      HttpStatus.BAD_REQUEST,
    );
  }

  throw new BaseException(
    (lang) => ExceptionMessages.BOOLEAN_TYPE_MESSAGE(key, lang),
    HttpStatus.BAD_REQUEST,
  );
}

export function enumTransform<T extends object>({
  value,
  key,
  enumObject,
}: {
  value: any;
  key: string;
  enumObject: T;
}): T[keyof T] | null {
  if (value === undefined || value === null) {
    return null;
  }

  const enumValues = Object.values(enumObject);
  if (enumValues.includes(value)) {
    return value;
  }

  throw new BaseException(
    (lang) => ExceptionMessages.ENUM_INVALID_MESSAGE(key, enumValues, lang),
    HttpStatus.BAD_REQUEST,
  );
}

export function upperCaseTranform(value: any) {
  if (value === undefined || value === null) return null;

  if (typeof value !== "string")
    throw new BaseException(
      (lang) => ExceptionMessages.STRING_OR_NULL_MESSAGE(value, lang),
      HttpStatus.BAD_REQUEST,
    );

  return value.toUpperCase();
}

export function lowerCaseTranform(value: any) {
  if (value === undefined || value === null) return null;

  if (typeof value !== "string")
    throw new BaseException(
      (lang) => ExceptionMessages.STRING_OR_NULL_MESSAGE(value, lang),
      HttpStatus.BAD_REQUEST,
    );

  return value.toLowerCase();
}

```

## File: connectfy-shared/src/functions/server/crypto.ts
```typescript
import { randomBytes, createCipheriv, createDecipheriv } from 'crypto';

const ALGORITHM = 'aes-256-gcm';
const IV_LENGTH = 12;
const AUTH_TAG_LENGTH = 16;

export function encryptPayload(plaintext: string, base64Key: string): string {
  const iv = randomBytes(IV_LENGTH);
  const key = Buffer.from(base64Key, 'base64');

  const cipher = createCipheriv(ALGORITHM, key, iv, {
    authTagLength: AUTH_TAG_LENGTH,
  });

  const encrypted = Buffer.concat([
    cipher.update(plaintext, 'utf-8'),
    cipher.final(),
  ]);

  const tag = cipher.getAuthTag();

  return Buffer.concat([iv, tag, encrypted]).toString('base64');
}

export function decryptPayload(payload: string, base64Key: string): string {
  const data = Buffer.from(payload, 'base64');
  const iv = data.slice(0, IV_LENGTH);
  const tag = data.slice(IV_LENGTH, IV_LENGTH + AUTH_TAG_LENGTH);
  const encrypted = data.slice(IV_LENGTH + AUTH_TAG_LENGTH);

  const key = Buffer.from(base64Key, 'base64');

  const decipher = createDecipheriv(ALGORITHM, key, iv, {
    authTagLength: AUTH_TAG_LENGTH,
  });

  decipher.setAuthTag(tag);

  const decrypted = Buffer.concat([
    decipher.update(encrypted),
    decipher.final(),
  ]);

  return decrypted.toString('utf-8');
}

```

## File: connectfy-shared/src/functions/server/kafka-commit.helper.ts
```typescript
import { BaseException } from "@/src";
import { HttpStatus } from "@nestjs/common";
import { KafkaContext } from "@nestjs/microservices";

export async function commitKafkaOffset(context: KafkaContext) {
  const topic = context.getTopic();
  const partition = context.getPartition();
  const originalMessage = context.getMessage();
  const offset = originalMessage.offset;

  console.log(
    `Committing offset ${offset} for topic ${topic}, partition ${partition}`,
  );

  try {
    await context
      .getConsumer()
      .commitOffsets([
        { topic, partition, offset: (Number(offset) + 1).toString() },
      ]);
    console.log("✅ Offset committed successfully");
  } catch (error) {
    // Let this fail properly as BaseException
    throw new BaseException(
      "❌ Failed to commit Kafka offset",
      HttpStatus.INTERNAL_SERVER_ERROR,
    );
  }
}

```

## File: connectfy-shared/src/class/index.ts
```typescript
export * from "./server/base.exception";

```

## File: connectfy-shared/src/class/server/base.exception.ts
```typescript
import { RpcException } from "@nestjs/microservices";
import { ExceptionMessages, HttpStatus } from "@/src";

export class BaseException extends RpcException {
  constructor(
    messageOrFunc:
      | string
      | string[]
      | ((
          ...args: any[]
        ) =>
          | string
          | string[]) = ExceptionMessages.INTERNAL_SERVER_ERROR_MESSAGE,
    statusCode = HttpStatus.INTERNAL_SERVER_ERROR,
    additional?: Record<string, any>,
  ) {
    // RESOLVE THE MESSAGE HERE
    const message =
      typeof messageOrFunc === "function" ? messageOrFunc() : messageOrFunc;

    super({ message, statusCode, additional });
  }
}

```

## File: connectfy-shared/src/decorators/index.ts
```typescript
export * from "./server/index";

```

## File: connectfy-shared/src/decorators/server/index.ts
```typescript
export * from "./field-validator/field-transformer.registry";
export * from "./field-validator/field-validator.decorator";
export * from "./field-validator/field-validator.registry";

```

## File: connectfy-shared/src/decorators/server/field-validator/field-validator.decorator.ts
```typescript
import {
  IsNotEmpty,
  IsOptional,
  Matches,
  ValidateIf,
  ValidateNested,
} from "class-validator";
import { Type } from "class-transformer";
import { applyDecorators } from "@nestjs/common";
import {
  FIELD_TYPE,
  VALIDATION_TYPE,
  validationMessage,
  FieldValidatorOptions,
  FIELD_TRANSFORMER_REGISTRY,
  FIELD_VALIDATOR_REGISTRY,
} from "@/src";

export function FieldValidator(options: FieldValidatorOptions) {
  const decorators: PropertyDecorator[] = [];

  /****
     1) İlk olaraq @Transform işə düşür. Ona görədə ilk o əlavə olunur
     2) Optional bir field olarsa @IsOptional üst tərəfdə olmalıdır
     3) Hər hansı bir şərtdən asılı olarsa @ValidateIf yazılacaq
     4) @ValidateIf true olarsa və ya @ValidateIf olmasa belə type-a uyğun decoratorlar gələcək
     5) Type prop-u gəldikdə @Type decoratoru işə düşür
  ****/

  // Transform decorator
  const transform = FIELD_TRANSFORMER_REGISTRY[options.type];
  decorators.push(transform(options));

  // Optional
  if (options.isOptional) {
    decorators.push(IsOptional());
  }

  // Validate if for conditional situations
  if (options.validateIf) {
    decorators.push(ValidateIf(options.validateIf));
  }

  // Validate nested
  if (options.validateNested) {
    decorators.push(ValidateNested(options.validateNested));
  }

  // Nested types validator (@Types decorator)
  if (options.classType) {
    decorators.push(Type(() => options.classType!));
  }

  // Type decorators
  const validator = FIELD_VALIDATOR_REGISTRY[options.type];

  decorators.push(...validator(options));

  if (options.type === FIELD_TYPE.ARRAY && options.arrayItemType) {
    const itemValidatorFactory =
      FIELD_VALIDATOR_REGISTRY[options.arrayItemType];

    const itemOptions = {
      ...options,
      type: options.arrayItemType,
      validationOptions: {
        ...options.validationOptions,
        each: true,
      },
    } as any;

    decorators.push(...itemValidatorFactory(itemOptions));
  }

  // Required
  if (!options.isOptional) {
    decorators.push(
      IsNotEmpty({
        message: (args) =>
          validationMessage({
            args,
            type: VALIDATION_TYPE.REQUIRED,
          }),
      }),
    );
  }

  if (
    (options.type === FIELD_TYPE.NUMBER ||
      options.type === FIELD_TYPE.STRING) &&
    options.matches
  ) {
    decorators.push(
      Matches(options.matches.regexp, {
        message: (args) =>
          validationMessage({
            args,
            type: options.matches!.message.type,
            params: options.matches?.message.params,
          }),
      }),
    );
  }

  return applyDecorators(...decorators);
}

```

## File: connectfy-shared/src/decorators/server/field-validator/field-transformer.registry.ts
```typescript
import { Transform } from "class-transformer";
import {
  arrayTransform,
  booleanTransform,
  dateTransform,
  enumTransform,
  numberTransform,
  objectTransform,
  stringTransform,
  FIELD_TYPE,
  FieldValidatorOptions,
  IEnumFieldOptions,
} from "@/src";

type TransformFactory = (opts: FieldValidatorOptions) => PropertyDecorator;

const STRING_TYPES = [
  FIELD_TYPE.STRING,
  FIELD_TYPE.EMAIL,
  FIELD_TYPE.UUID,
  FIELD_TYPE.URL,
  FIELD_TYPE.PHONE,
  FIELD_TYPE.JSON,
  FIELD_TYPE.JWT,
  FIELD_TYPE.LOWERCASE,
  FIELD_TYPE.UPPERCASE,
  FIELD_TYPE.HEX_COLOR,
  FIELD_TYPE.BASE64,
  FIELD_TYPE.IP,
  FIELD_TYPE.MAC_ADDRESS,
];

const NUMBER_TYPES = [
  FIELD_TYPE.NUMBER,
  FIELD_TYPE.INT,
  FIELD_TYPE.POSITIVE,
  FIELD_TYPE.NEGATIVE,
  FIELD_TYPE.DECIMAL,
  FIELD_TYPE.LATITUDE,
  FIELD_TYPE.LONGITUDE,
];

// ====================================================
// Transform decorator selector from class-transformer
// ====================================================
const make =
  (transformFn: (args: { key: string; value: any }) => any): TransformFactory =>
  () =>
    Transform(({ key, value }) => transformFn({ key, value }));

export const FIELD_TRANSFORMER_REGISTRY: Record<FIELD_TYPE, TransformFactory> =
  Object.fromEntries([
    ...STRING_TYPES.map((t) => [t, make(stringTransform)]),
    ...NUMBER_TYPES.map((t) => [t, make(numberTransform)]),
    [FIELD_TYPE.BOOLEAN, make(booleanTransform)],
    [FIELD_TYPE.DATE, make(dateTransform)],
    [FIELD_TYPE.DATE_STRING, make(stringTransform)],
    [FIELD_TYPE.ARRAY, make(arrayTransform)],
    [
      FIELD_TYPE.ENUM,
      (opts: IEnumFieldOptions) =>
        Transform(({ key, value }) =>
          enumTransform({ key, value, enumObject: opts.enumObject }),
        ),
    ],
    [FIELD_TYPE.OBJECT, make(objectTransform)],
  ]) as unknown as Record<FIELD_TYPE, TransformFactory>;

```

## File: connectfy-shared/src/decorators/server/field-validator/field-validator.registry.ts
```typescript
import {
  IsString,
  IsUUID,
  IsEmail,
  IsInt,
  IsNumber,
  IsBoolean,
  IsArray,
  IsEnum,
  IsDate,
  IsUrl,
  IsPhoneNumber,
  IsJSON,
  IsJWT,
  IsLowercase,
  IsUppercase,
  IsHexColor,
  IsBase64,
  IsIP,
  IsMACAddress,
  IsPositive,
  IsNegative,
  IsDecimal,
  IsLatitude,
  IsLongitude,
  IsDateString,
  IsObject,
  MinDate,
  MaxDate,
  ArrayUnique,
  ArrayMaxSize,
  ArrayMinSize,
  Max,
  Min,
  MaxLength,
  MinLength,
  ValidationOptions,
} from "class-validator";

import {
  FieldValidatorOptions,
  IArrayFieldOptions,
  IDateFieldOptions,
  IEnumFieldOptions,
  INumberFieldOptions,
  IStringFieldOptions,
  FIELD_TYPE,
  VALIDATION_TYPE,
  validationMessage,
} from "@/src";

type DecoratorFactory = (opts: FieldValidatorOptions) => PropertyDecorator[];

/**
 * Helper function to create validation message options
 */
const createValidationOptions = (
  type: VALIDATION_TYPE,
  baseOptions?: ValidationOptions,
  params?: Record<string, any>,
) => ({
  message: (args: any) => validationMessage({ args, type, params }),
  ...baseOptions,
});

/**
 * Conditionally add decorator to array
 */
const addDecorator = (
  decorators: PropertyDecorator[],
  condition: boolean,
  decorator: PropertyDecorator,
) => {
  if (condition) decorators.push(decorator);
};

/**
 * Add string length validators (minLength, maxLength)
 */
const addStringLengthValidators = (
  decorators: PropertyDecorator[],
  opts: IStringFieldOptions,
) => {
  addDecorator(
    decorators,
    opts.minLength !== undefined,
    MinLength(
      opts.minLength!,
      createValidationOptions(
        VALIDATION_TYPE.MIN_LENGTH,
        opts.validationOptions,
        {
          minLength: opts.minLength,
        },
      ),
    ),
  );

  addDecorator(
    decorators,
    opts.maxLength !== undefined,
    MaxLength(
      opts.maxLength!,
      createValidationOptions(
        VALIDATION_TYPE.MAX_LENGTH,
        opts.validationOptions,
        {
          maxLength: opts.maxLength,
        },
      ),
    ),
  );
};

/**
 * Add number range validators (min, max)
 */
const addNumberRangeValidators = (
  decorators: PropertyDecorator[],
  opts: INumberFieldOptions,
) => {
  addDecorator(
    decorators,
    opts.min !== undefined,
    Min(
      opts.min!,
      createValidationOptions(VALIDATION_TYPE.MIN, opts.validationOptions, {
        min: opts.min,
      }),
    ),
  );

  addDecorator(
    decorators,
    opts.max !== undefined,
    Max(
      opts.max!,
      createValidationOptions(VALIDATION_TYPE.MAX, opts.validationOptions, {
        max: opts.max,
      }),
    ),
  );
};

/**
 * Add array size validators (minSize, maxSize, unique)
 */
const addArraySizeValidators = (
  decorators: PropertyDecorator[],
  opts: IArrayFieldOptions,
) => {
  addDecorator(
    decorators,
    opts.minSize !== undefined,
    ArrayMinSize(
      opts.minSize!,
      createValidationOptions(
        VALIDATION_TYPE.ARRAY_MIN_SIZE,
        opts.validationOptions,
        {
          minSize: opts.minSize,
        },
      ),
    ),
  );

  addDecorator(
    decorators,
    opts.maxSize !== undefined,
    ArrayMaxSize(
      opts.maxSize!,
      createValidationOptions(
        VALIDATION_TYPE.ARRAY_MAX_SIZE,
        opts.validationOptions,
        {
          maxSize: opts.maxSize,
        },
      ),
    ),
  );

  addDecorator(
    decorators,
    !!opts.unique,
    ArrayUnique(
      createValidationOptions(
        VALIDATION_TYPE.ARRAY_UNIQUE,
        opts.validationOptions,
      ),
    ),
  );
};

/**
 * Add date range validators (minDate, maxDate)
 */
const addDateRangeValidators = (
  decorators: PropertyDecorator[],
  opts: IDateFieldOptions,
) => {
  addDecorator(
    decorators,
    !!opts.minDate,
    MinDate(
      opts.minDate!,
      createValidationOptions(
        VALIDATION_TYPE.MIN_DATE,
        opts.validationOptions,
        {
          minDate: opts.minDate,
        },
      ),
    ),
  );

  addDecorator(
    decorators,
    !!opts.maxDate,
    MaxDate(
      opts.maxDate!,
      createValidationOptions(
        VALIDATION_TYPE.MAX_DATE,
        opts.validationOptions,
        {
          maxDate: opts.maxDate,
        },
      ),
    ),
  );
};

/**
 * Factory to create string field validators
 */
const createStringValidator = (
  type: VALIDATION_TYPE,
  extraDecorator?: (opts: IStringFieldOptions) => PropertyDecorator,
): DecoratorFactory => {
  return (opts: IStringFieldOptions) => {
    const decorators: PropertyDecorator[] = [
      IsString(
        createValidationOptions(VALIDATION_TYPE.STRING, opts.validationOptions),
      ),
    ];

    if (extraDecorator) {
      decorators.push(extraDecorator(opts));
    }

    addStringLengthValidators(decorators, opts);
    return decorators;
  };
};

/**
 * Factory to create number field validators
 */
const createNumberValidator = (
  validatorDecorator: (opts: INumberFieldOptions) => PropertyDecorator,
): DecoratorFactory => {
  return (opts: INumberFieldOptions) => {
    const decorators: PropertyDecorator[] = [validatorDecorator(opts)];
    addNumberRangeValidators(decorators, opts);
    return decorators;
  };
};

/**
 * Type decorator selector from class-validator
 */
export const FIELD_VALIDATOR_REGISTRY: Record<FIELD_TYPE, DecoratorFactory> = {
  // ================= STRING TYPES =================
  [FIELD_TYPE.STRING]: createStringValidator(VALIDATION_TYPE.STRING),

  [FIELD_TYPE.EMAIL]: createStringValidator(VALIDATION_TYPE.EMAIL, (opts) =>
    IsEmail(
      { ...opts.emailOptions },
      createValidationOptions(VALIDATION_TYPE.EMAIL, opts.validationOptions),
    ),
  ),

  [FIELD_TYPE.UUID]: createStringValidator(VALIDATION_TYPE.UUID, (opts) =>
    IsUUID(
      opts?.uuidVersion || "4",
      createValidationOptions(VALIDATION_TYPE.UUID, opts.validationOptions),
    ),
  ),

  [FIELD_TYPE.URL]: createStringValidator(VALIDATION_TYPE.URL, (opts) =>
    IsUrl(
      { ...opts.urlOptions },
      createValidationOptions(VALIDATION_TYPE.URL, opts.validationOptions),
    ),
  ),

  [FIELD_TYPE.PHONE]: createStringValidator(VALIDATION_TYPE.PHONE, (opts) =>
    IsPhoneNumber(
      opts?.region as any,
      createValidationOptions(VALIDATION_TYPE.PHONE, opts.validationOptions),
    ),
  ),

  [FIELD_TYPE.JSON]: createStringValidator(VALIDATION_TYPE.JSON, (opts) =>
    IsJSON(
      createValidationOptions(VALIDATION_TYPE.JSON, opts.validationOptions),
    ),
  ),

  [FIELD_TYPE.JWT]: createStringValidator(VALIDATION_TYPE.JWT, (opts) =>
    IsJWT(createValidationOptions(VALIDATION_TYPE.JWT, opts.validationOptions)),
  ),

  [FIELD_TYPE.LOWERCASE]: createStringValidator(
    VALIDATION_TYPE.LOWERCASE,
    (opts) =>
      IsLowercase(
        createValidationOptions(
          VALIDATION_TYPE.LOWERCASE,
          opts.validationOptions,
        ),
      ),
  ),

  [FIELD_TYPE.UPPERCASE]: createStringValidator(
    VALIDATION_TYPE.UPPERCASE,
    (opts) =>
      IsUppercase(
        createValidationOptions(
          VALIDATION_TYPE.UPPERCASE,
          opts.validationOptions,
        ),
      ),
  ),

  [FIELD_TYPE.HEX_COLOR]: createStringValidator(
    VALIDATION_TYPE.HEX_COLOR,
    (opts) =>
      IsHexColor(
        createValidationOptions(
          VALIDATION_TYPE.HEX_COLOR,
          opts.validationOptions,
        ),
      ),
  ),

  [FIELD_TYPE.BASE64]: createStringValidator(VALIDATION_TYPE.BASE64, (opts) =>
    IsBase64(
      { ...opts.base64Options },
      createValidationOptions(VALIDATION_TYPE.BASE64, opts.validationOptions),
    ),
  ),

  [FIELD_TYPE.IP]: createStringValidator(VALIDATION_TYPE.IP, (opts) =>
    IsIP(
      opts.ipVersion,
      createValidationOptions(VALIDATION_TYPE.IP, opts.validationOptions),
    ),
  ),

  [FIELD_TYPE.MAC_ADDRESS]: createStringValidator(
    VALIDATION_TYPE.MAC_ADDRESS,
    (opts) =>
      IsMACAddress(
        createValidationOptions(
          VALIDATION_TYPE.MAC_ADDRESS,
          opts.validationOptions,
        ),
      ),
  ),

  // ================= NUMBER TYPES =================
  [FIELD_TYPE.NUMBER]: createNumberValidator((opts: INumberFieldOptions) =>
    IsNumber(
      { ...opts.numberOptions },
      createValidationOptions(VALIDATION_TYPE.NUMBER, opts.validationOptions),
    ),
  ),

  [FIELD_TYPE.INT]: createNumberValidator((opts: INumberFieldOptions) =>
    IsInt(createValidationOptions(VALIDATION_TYPE.INT, opts.validationOptions)),
  ),

  [FIELD_TYPE.POSITIVE]: createNumberValidator((opts: INumberFieldOptions) =>
    IsPositive(
      createValidationOptions(VALIDATION_TYPE.POSITIVE, opts.validationOptions),
    ),
  ),

  [FIELD_TYPE.NEGATIVE]: createNumberValidator((opts: INumberFieldOptions) =>
    IsNegative(
      createValidationOptions(VALIDATION_TYPE.NEGATIVE, opts.validationOptions),
    ),
  ),

  [FIELD_TYPE.DECIMAL]: createNumberValidator((opts: INumberFieldOptions) =>
    IsDecimal(
      { ...opts.decimalOptions },
      createValidationOptions(VALIDATION_TYPE.DECIMAL, opts.validationOptions),
    ),
  ),

  [FIELD_TYPE.LATITUDE]: createNumberValidator((opts: INumberFieldOptions) =>
    IsLatitude(
      createValidationOptions(VALIDATION_TYPE.LATITUDE, opts.validationOptions),
    ),
  ),

  [FIELD_TYPE.LONGITUDE]: createNumberValidator((opts: INumberFieldOptions) =>
    IsLongitude(
      createValidationOptions(
        VALIDATION_TYPE.LONGITUDE,
        opts.validationOptions,
      ),
    ),
  ),

  // ================= BOOLEAN =================
  [FIELD_TYPE.BOOLEAN]: (opts) => [
    IsBoolean(
      createValidationOptions(VALIDATION_TYPE.BOOLEAN, opts.validationOptions),
    ),
  ],

  // ================= DATE TYPES =================
  [FIELD_TYPE.DATE]: (opts: IDateFieldOptions) => {
    const decorators: PropertyDecorator[] = [
      IsDate(
        createValidationOptions(VALIDATION_TYPE.DATE, opts.validationOptions),
      ),
    ];
    addDateRangeValidators(decorators, opts);
    return decorators;
  },

  [FIELD_TYPE.DATE_STRING]: (opts: IDateFieldOptions) => [
    IsDateString(
      { ...opts.dateStringOptions },
      createValidationOptions(VALIDATION_TYPE.DATE, opts.validationOptions),
    ),
  ],

  // ================= ARRAY =================
  [FIELD_TYPE.ARRAY]: (opts: IArrayFieldOptions) => {
    const decorators: PropertyDecorator[] = [
      IsArray(
        createValidationOptions(VALIDATION_TYPE.ARRAY, opts.validationOptions),
      ),
    ];
    addArraySizeValidators(decorators, opts);
    return decorators;
  },

  // ================= ENUM =================
  [FIELD_TYPE.ENUM]: (opts: IEnumFieldOptions) => [
    IsEnum(
      opts.enumObject,
      createValidationOptions(VALIDATION_TYPE.ENUM, opts.validationOptions, {
        values: Object.values(opts.enumObject),
      }),
    ),
  ],

  // ================= OBJECT =================
  [FIELD_TYPE.OBJECT]: (opts) => [
    IsObject(
      createValidationOptions(VALIDATION_TYPE.OBJECT, opts.validationOptions),
    ),
  ],
};

```

## File: connectfy-shared/src/interfaces/index.ts
```typescript
export * from "./server/interfaces";
export * from "./server/request.interface";
export * from "./server/response.interface";
export * from "./server/validation.interface";

```

## File: connectfy-shared/src/interfaces/server/response.interface.ts
```typescript
export interface IRemoveAllResponse {
  deletedCount: number;
  notDeleted: string[];
  deletedIds: string[];
}

export interface IResponse {
  success: boolean;
  [key: string]: any;
}

export interface IFindAllResponse<T> {
  data: T[];
  totalCount?: number;
  limit?: number;
  currentPage?: number;
  totalPages?: number;
}

```

## File: connectfy-shared/src/interfaces/server/interfaces.ts
```typescript
import { PopulateOptions } from "mongoose";
import { ClientKafka, ClientProxy } from "@nestjs/microservices";
import { ClsService } from "nestjs-cls";

export interface ICountry {
  key: string;
  name: string;
  flag: string;
  code: string;
  numberLength: number;
}

export interface ISendWithContextClientParams {
  client: ClientProxy;
  cls?: ClsService;
  endpoint: string;
  payload?: any;
}

export interface IEmitTcpWithContextClientParams extends ISendWithContextClientParams {}

export interface IEmitWithContextClientParams {
  client: ClientKafka;
  cls?: ClsService;
  topic: string;
  payload?: any;
}

export interface IBaseRepositoryOptions {
  lean?: boolean;
  populate?: string | PopulateOptions | string[] | PopulateOptions[];
}

export interface IBaseRepositoryUpdateOptions extends IBaseRepositoryOptions {
  new?: boolean;
  upsert?: boolean;
  runValidators?: boolean;
}

export interface IBaseRepositoryRemoveOptions extends IBaseRepositoryOptions {}

export interface IBaseRepositoryInterface {
  _id: string;
  [key: string]: any;
}

```

## File: connectfy-shared/src/interfaces/server/request.interface.ts
```typescript
import {
  AvatarFormats,
  DATE_FORMAT,
  GENDER,
  LANGUAGE,
  NOTIFICATION_CONTENT_MODE,
  NOTIFICATION_SOUND_MODE,
  PRIVACY_SETTINGS_CHOICE,
  PROVIDER,
  ROLE,
  STARTUP_PAGE,
  THEME,
  TIME_FORMAT,
} from "../../enums/enum";

export interface ILoggedUser extends IReturnedUser {
  language: LANGUAGE;
  avatar: IAvatar | null;
  defaultAvatar: IDefaultAvatar;
}

export interface IReturnedUser {
  _id: string;
  username: string;
  email: string;
  role: ROLE;
  provider: PROVIDER;
  password: string;
  phoneNumber: IPhoneNumber;
  createdAt: Date;
  updatedAt: Date;
}

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

export interface IPhoneNumber {
  countryCode: string | null;
  number: string | null;
  fullPhoneNumber: string | null;
}

export interface IAccount {
  _id: string;
  userId: string;
  firstName: string;
  lastName: string;
  gender: GENDER;
  bio: string | null;
  location: string | null;
  avatar: string | null;
  lastSeen: Date;
  birthdayDate: Date;
  createdAt: Date;
  updatedAt: Date;
}

export interface IGeneralSettings {
  _id: string;
  userId: string;
  theme: THEME;
  language: LANGUAGE;
  startupPage: STARTUP_PAGE;
  timeZone: ITimeZone;
}

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
}

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
  friendshipRequest: boolean;
}

export interface ITimeZone {
  timeFormat: TIME_FORMAT;
  dateFormat: DATE_FORMAT;
}

```

## File: connectfy-shared/src/interfaces/server/validation.interface.ts
```typescript
import { VALIDATION_TYPE, FIELD_TYPE } from "@/src";
import {
  IsIpVersion,
  IsNumberOptions,
  ValidationArguments,
  ValidationOptions,
} from "class-validator";
import { ClassConstructor } from "class-transformer";

export interface IValidateMessageOptions {
  args: ValidationArguments;
  type: VALIDATION_TYPE;
  [key: string]: any;
}

export interface IBaseFieldOptions {
  type: FIELD_TYPE;
  isOptional?: boolean;
  customMessage?: string;
  validateIf?: (object: any, value: any) => boolean;
  classType?: ClassConstructor<any>;
  validateNested?: ValidationOptions;
  validationOptions?: ValidationOptions;
}

export interface IStringFieldOptions extends IBaseFieldOptions {
  type:
    | FIELD_TYPE.STRING
    | FIELD_TYPE.UUID
    | FIELD_TYPE.EMAIL
    | FIELD_TYPE.URL
    | FIELD_TYPE.PHONE
    | FIELD_TYPE.JSON
    | FIELD_TYPE.JWT
    | FIELD_TYPE.LOWERCASE
    | FIELD_TYPE.UPPERCASE
    | FIELD_TYPE.HEX_COLOR
    | FIELD_TYPE.BASE64
    | FIELD_TYPE.IP
    | FIELD_TYPE.MAC_ADDRESS;
  minLength?: number;
  maxLength?: number;

  // Regex for patterns
  matches?: {
    regexp: RegExp;
    message: {
      type: VALIDATION_TYPE;
      params?: Record<string, any>;
    };
  };

  // Phone specific
  region?: string; // 'AZ', 'US', və s.

  // UUID specific
  uuidVersion?: validator.UUIDVersion;

  // IP specific
  ipVersion?: IsIpVersion;

  // Email spesific
  emailOptions?: validator.IsEmailOptions;

  // URL spesific
  urlOptions?: validator.IsURLOptions;

  // Base64 spesific
  base64Options?: validator.IsBase64Options;
}

export interface INumberFieldOptions extends IBaseFieldOptions {
  type:
    | FIELD_TYPE.NUMBER
    | FIELD_TYPE.INT
    | FIELD_TYPE.POSITIVE
    | FIELD_TYPE.NEGATIVE
    | FIELD_TYPE.DECIMAL
    | FIELD_TYPE.LATITUDE
    | FIELD_TYPE.LONGITUDE;
  min?: number;
  max?: number;

  // Decimal specific
  decimalOptions?: validator.IsDecimalOptions;

  // Regex for patterns
  matches?: {
    regexp: RegExp;
    message: {
      type: VALIDATION_TYPE;
      params?: Record<string, any>;
    };
  };

  // Number spesific
  numberOptions?: IsNumberOptions;
}

export interface IEnumFieldOptions extends IBaseFieldOptions {
  type: FIELD_TYPE.ENUM;
  enumObject: object;
  enumValues?: string[];
}

export interface IDateFieldOptions extends IBaseFieldOptions {
  type: FIELD_TYPE.DATE | FIELD_TYPE.DATE_STRING;
  minDate?: Date;
  maxDate?: Date;

  // DateString spesific
  dateStringOptions?: validator.IsISO8601Options;
}

export interface IArrayFieldOptions extends IBaseFieldOptions {
  type: FIELD_TYPE.ARRAY;
  minSize?: number;
  maxSize?: number;
  arrayItemType?: FIELD_TYPE;
  unique?: boolean;
}

export interface IObjectFieldOptions extends IBaseFieldOptions {
  type: FIELD_TYPE.OBJECT;
}

export interface IBooleanFieldOptions extends IBaseFieldOptions {
  type: FIELD_TYPE.BOOLEAN;
}

export type FieldValidatorOptions =
  | IStringFieldOptions
  | INumberFieldOptions
  | IEnumFieldOptions
  | IDateFieldOptions
  | IArrayFieldOptions
  | IObjectFieldOptions
  | IBooleanFieldOptions;

```

