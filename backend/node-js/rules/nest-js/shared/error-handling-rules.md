# Error Handling Rules (NestJS)

> NestJS-specific error handling using AppException, error code enums, i18n localization, and HttpExceptionFilter.

## AppException

- Use `AppException` with module-specific error enums — **never** NestJS built-in exceptions:

```ts
// common/exceptions/app-exception.ts
export class AppException extends HttpException {
  constructor(
    public readonly code: AppExceptionCode,
    status: HttpStatus = HttpStatus.BAD_REQUEST,
  ) {
    super({ code }, status);
  }
}
```

```ts
// ❌ Incorrect — NestJS built-in exceptions
throw new NotFoundException(`User ${id} not found`);
throw new BadRequestException("Invalid email");
throw new UnauthorizedException("Invalid credentials");

// ✅ Correct — AppException with error enums
throw new AppException(UserError.NOT_FOUND, HttpStatus.NOT_FOUND);
throw new AppException(AuthError.INVALID_CREDENTIALS, HttpStatus.UNAUTHORIZED);
throw new AppException(AuthError.PERMISSION_DENIED, HttpStatus.FORBIDDEN);
```

## Module Error Enums

- Each module defines its own error enum with a `MODULE_` prefix:

```ts
// modules/user/enums/user-error.enum.ts
export enum UserError {
  ALREADY_EXISTS = "USER_ALREADY_EXISTS",
  NOT_FOUND = "USER_NOT_FOUND",
  INACTIVE = "USER_INACTIVE",
  SELF_UPDATE_FORBIDDEN = "USER_SELF_UPDATE_FORBIDDEN",
  SELF_DELETE_FORBIDDEN = "USER_SELF_DELETE_FORBIDDEN",
}

// modules/auth/enums/auth-error.enum.ts
export enum AuthError {
  INVALID_CREDENTIALS = "AUTH_INVALID_CREDENTIALS",
  TOKEN_INVALID_OR_EXPIRED = "AUTH_TOKEN_INVALID_OR_EXPIRED",
  OTP_LIMIT_EXCEEDED = "AUTH_OTP_LIMIT_EXCEEDED",
  EMAIL_ALREADY_EXISTS = "AUTH_EMAIL_ALREADY_EXISTS",
  PERMISSION_DENIED = "AUTH_PERMISSION_DENIED",
  ACCESS_TOKEN_INVALID_OR_EXPIRED = "AUTH_ACCESS_TOKEN_INVALID_OR_EXPIRED",
  COMPANY_KEY_MISMATCH = "AUTH_COMPANY_KEY_MISMATCH",
  COMPANY_KEY_REQUIRED = "AUTH_COMPANY_KEY_REQUIRED",
}

// common/enums/global-error.enum.ts
export enum GlobalError {
  VALIDATION_FAILED = "VALIDATION_FAILED",
  UNKNOWN_ERROR = "UNKNOWN_ERROR",
  EMAIL_SEND_FAILED = "EMAIL_SEND_FAILED",
}
```

## AppExceptionCode Union Type

- Union all module error enums into a single type:

```ts
// common/localization/type.ts
export type AppExceptionCode =
  | GlobalError
  | AuthError
  | UserError
  | CompanyError
  | EmployeeError
  | CandidateError
  | FileError;

export interface LocalizedString {
  uz: string;
  ru: string;
  en: string;
  cyr: string;
}
```

## i18n Error Messages

- Every error code has translations in 4 languages (uz, ru, en, cyr):

```ts
// common/localization/error-messages.ts
export const ERROR_MESSAGES: Record<AppExceptionCode, LocalizedString> = {
  VALIDATION_FAILED: {
    uz: "Maʼlumotlarni tekshirishda xatolik yuz berdi.",
    ru: "Произошла ошибка при проверке данных.",
    en: "An error occurred while validating the data.",
    cyr: "Маълумотларни текширишда хатолик юз берди.",
  },
  USER_NOT_FOUND: {
    uz: "Foydalanuvchi topilmadi yoki mavjud emas.",
    ru: "Пользователь не найден или не существует.",
    en: "User not found or does not exist.",
    cyr: "Фойдаланувчи топилмади ёки мавжуд эмас.",
  },
  AUTH_INVALID_CREDENTIALS: {
    uz: "Tizimga kirish ma'lumotlari noto'g'ri.",
    ru: "Неверные учетные данные.",
    en: "Invalid login credentials. Please try again.",
    cyr: "Тизимга кириш маълумотлари нотўғри.",
  },
  // ... all other error codes
};
```

- When adding a new error code:
  1. Add to the module's error enum.
  2. Add to `AppExceptionCode` union type.
  3. Add i18n translations to `ERROR_MESSAGES`.

## HttpExceptionFilter

- The global exception filter handles `AppException`, `BadRequestException` (validation), and unexpected errors:

```ts
@Catch(HttpException)
export class HttpExceptionFilter implements ExceptionFilter {
  catch(exception: HttpException, host: ArgumentsHost) {
    const ctx = host.switchToHttp();
    const response = ctx.getResponse<Response>();
    const request = ctx.getRequest<Request>();

    const statusCode = exception.getStatus?.() ?? 500;
    const locale = getLocaleFromRequest(request); // from Accept-Language header
    let message = ERROR_MESSAGES.UNKNOWN_ERROR[locale];
    let code: AppExceptionCode = "UNKNOWN_ERROR" as AppExceptionCode;
    let details: ValidationErrorDetail[] | undefined;

    if (exception instanceof AppException) {
      code = exception.code;
      message = ERROR_MESSAGES[code][locale];
    } else if (exception instanceof BadRequestException) {
      // Validation errors from ValidationPipe
      const body = exception.getResponse() as { message: ValidationErrorDetail[] };
      message = ERROR_MESSAGES.VALIDATION_FAILED[locale];
      code = GlobalError.VALIDATION_FAILED;
      details = body.message;
    }

    response.status(statusCode).json(
      new ErrorResponseDto({ statusCode, message, code, locale, path: request.url, details }),
    );
  }
}
```

- Register globally in `main.ts`:

```ts
app.useGlobalFilters(new HttpExceptionFilter());
```

## Error Response DTO

```ts
export class ErrorResponseDto {
  statusCode: number;
  message: string;    // Localized user-friendly message
  code: string;       // Machine-readable error code
  locale: string;     // Response language (uz, ru, en, cyr)
  path: string;       // Request URL
  timestamp: string;  // ISO 8601
  details?: ValidationErrorDetail[];

  constructor(params: Partial<ErrorResponseDto>) {
    Object.assign(this, { timestamp: new Date().toISOString(), locale: "uz", ...params });
  }
}

export class ValidationErrorDetail {
  field: string;
  errors: string[];
}
```

## Locale Detection

- Extract locale from `Accept-Language` header (defaults to `uz`):

```ts
export function getLocaleFromRequest(request: Request): "uz" | "ru" | "en" | "cyr" {
  const lang = request.headers["accept-language"];
  if (lang && ["uz", "ru", "en", "cyr"].includes(lang)) return lang;
  return "uz";
}
```

## Validation Errors

- Use `ValidationPipe` globally — validation errors are caught by `HttpExceptionFilter`:

```ts
// main.ts
app.useGlobalPipes(
  new ValidationPipe({
    whitelist: true,
    forbidNonWhitelisted: true,
    transform: true,
  }),
);
```

- DTO with class-validator:

```ts
export class UserCreateDto {
  @IsEmail({}, { message: "Invalid email format" })
  email: string;

  @IsString()
  @MinLength(2, { message: "Name must be at least 2 characters" })
  @MaxLength(150)
  name: string;

  @IsString()
  @MinLength(8)
  password: string;
}
```

## Service Layer Error Pattern

- Services throw `AppException` — controllers never contain error logic:

```ts
@Injectable()
export class UserService {
  async findOne(id: number): Promise<UserResponseDto> {
    const user = await this.userRepository.findOneBy({ id });
    if (!user) throw new AppException(UserError.NOT_FOUND, HttpStatus.NOT_FOUND);
    return UserMapper.toDto(user);
  }

  async createUser(dto: UserCreateDto): Promise<UserResponseDto> {
    const exists = await this.userRepository.findByUsername(dto.email);
    if (exists) throw new AppException(UserError.ALREADY_EXISTS, HttpStatus.CONFLICT);

    const entity = UserMapper.toCreateEntity(dto, UserRole.USER);
    const saved = await this.userRepository.save(entity);
    return UserMapper.toDto(saved);
  }

  async deleteUser(id: number): Promise<void> {
    const user = await this.userRepository.findOneBy({ id });
    if (!user) throw new AppException(UserError.NOT_FOUND, HttpStatus.NOT_FOUND);
    await this.userRepository.softDelete(id);
  }
}
```

## TypeORM Error Handling

- Catch PostgreSQL-specific errors and convert to `AppException`:

```ts
async createUser(dto: UserCreateDto): Promise<UserResponseDto> {
  try {
    const entity = UserMapper.toCreateEntity(dto, UserRole.USER);
    const saved = await this.userRepository.save(entity);
    return UserMapper.toDto(saved);
  } catch (error) {
    if (error.code === "23505") {
      throw new AppException(UserError.ALREADY_EXISTS, HttpStatus.CONFLICT);
    }
    throw error;
  }
}
```

## Rules

- **Never** use NestJS built-in exceptions — always use `AppException` with error enums.
- Every module defines its own error enum with `MODULE_` prefix.
- Every error code has i18n translations in all 4 languages.
- New error codes require 3 changes: enum → union type → translations.
- `HttpExceptionFilter` handles all exceptions — register globally.
- Services throw exceptions — controllers delegate to services.
- Log unexpected errors in the filter with full context.
- Never expose stack traces or internal details in responses.
- Locale is determined by `Accept-Language` header (default: `uz`).
