---
layout: post
title: "flutter injectable 패키지"
date: 2025-11-23 01:16:00 +0900
description: >
  injectable 패키지의 주요 기능을 정리합니다.
categories: [pick]
tags: [amplify, data, authorization]
---

injectable은 get_it 기반의 DI를 자동화해주는 코드 생성 도구이다.
각 어노테이션이 무엇을 하는지, 그리고 실제로 어떻게 흐름이 구성되는지 단계별로 정리해보자.

## 기본 세팅 및 실행 흐름

### 1. 초기 설정

먼저 pubspec.yaml에 필요한 패키지들을 추가한다.

```yaml
dependencies:
  injectable:
  get_it:

dev_dependencies:
  injectable_generator:
  build_runner:
```

그 다음 DI 설정 파일을 만들고, GetIt 인스턴스를 글로벌로 선언한다.

```dart
import 'injection.config.dart';

final getIt = GetIt.instance;

@InjectableInit()
void configureDependencies() => getIt.init();
```

이제 main 함수에서 configureDependencies()를 호출하면 모든 의존성이 등록된다.

```dart
void main() {
  configureDependencies();
  runApp(MyApp());
}
```

### 2. 코드 생성 명령어

다음 명령어를 실행하면 injectable_generator가 자동으로 의존성 등록 코드를 생성한다.

```bash
flutter pub run build_runner build
# 또는 watch 모드로 자동 갱신
flutter pub run build_runner watch
```

생성된 파일(예: injection.config.dart)에는 모든 등록 로직이 자동으로 구현되어 있다.

---

## 주요 어노테이션 상세 해설

### @injectable - 기본 팩토리 등록

가장 기본적인 어노테이션으로, 클래스를 팩토리 형태로 등록한다.

```dart
@injectable
class ApiClient {
  ApiClient(NetworkService service);
}
```

**생성되는 코드:**

```dart
gh.factory<ApiClient>(() => ApiClient(getIt<NetworkService>()));
```

**동작 방식:**

- getIt.get<ApiClient>()를 호출할 때마다 새로운 인스턴스가 생성된다.
- 생성자의 파라미터는 자동으로 GetIt에서 해결(resolve)된다.

**언제 사용하나:**

- 매번 새로운 객체가 필요한 경우
- 상태를 공유하지 않아야 하는 서비스나 컨트롤러

---

### @singleton - 싱글턴 등록

앱 전체에서 하나의 인스턴스만 유지한다.

```dart
@singleton
class AuthService {
  AuthService(ApiClient client);
}
```

**생성되는 코드:**

```dart
gh.singleton<AuthService>(AuthService(getIt<ApiClient>()));
```

**동작 방식:**

- 앱 시작 시점에 즉시 인스턴스가 생성된다.
- 이후 getIt.get<AuthService>()를 호출하면 항상 동일한 객체를 반환한다.

**signalsReady 옵션:**

```dart
@Singleton(signalsReady: true)
class DatabaseService {}
```

이 옵션을 사용하면 GetIt의 isReady() 메커니즘과 연동된다.

---

### @lazySingleton - 지연 초기화 싱글턴

첫 호출 시점에 인스턴스를 생성하고, 이후 재사용한다.

```dart
@lazySingleton
class LocalStorage {
  LocalStorage();
}
```

**생성되는 코드:**

```dart
gh.lazySingleton<LocalStorage>(() => LocalStorage());
```

**동작 방식:**

- 앱 시작 시에는 인스턴스를 만들지 않는다.
- 처음 getIt.get<LocalStorage>()를 호출할 때 생성된다.
- 이후에는 동일한 인스턴스를 반환한다.

**언제 사용하나:**

- 초기화 비용이 크지만 항상 사용되지는 않는 서비스
- 메모리를 절약하고 싶을 때

---

### @factoryMethod - 생성 메서드 지정

기본 생성자가 아닌 다른 메서드로 객체를 생성할 때 사용한다.

```dart
@injectable
class UserRepository {
  UserRepository._internal(ApiClient client);

  @factoryMethod
  static UserRepository create(ApiClient client) {
    return UserRepository._internal(client);
  }
}
```

**생성되는 코드:**

```dart
gh.factory<UserRepository>(() => UserRepository.create(getIt<ApiClient>()));
```

**named constructor 예시:**

```dart
@injectable
class ConfigService {
  @factoryMethod
  ConfigService.fromEnv(String env);
}
```

이렇게 하면 ConfigService.fromEnv()가 호출된다.

**언제 사용하나:**

- private 생성자를 사용할 때
- 추상 클래스의 static 팩토리 메서드
- 복잡한 초기화 로직이 필요한 경우

---

### @postConstruct - 생성 후 초기화

객체가 생성된 직후 추가 초기화 로직을 실행한다.

```dart
@injectable
class ChatController {
  ChatController(MessageService service);

  @postConstruct
  void init() {
    // WebSocket 연결 초기화
    print('ChatController initialized');
  }
}
```

**동작 방식:**

- 생성자 호출 후 자동으로 init()이 실행된다.
- 동기/비동기 모두 지원한다.

**비동기 초기화:**

```dart
@injectable
class DatabaseController {
  @PostConstruct(preResolve: true)
  Future<void> connect() async {
    await database.connect();
  }
}
```

preResolve: true를 설정하면 Future가 완료될 때까지 대기한 후 등록한다.

---

### @preResolve - 비동기 의존성 사전 해결

비동기로 생성되는 객체를 await 처리한 후 등록한다.

```dart
@module
abstract class AppModule {
  @preResolve
  Future<SharedPreferences> get prefs => SharedPreferences.getInstance();
}
```

**생성되는 코드:**

```dart
Future<GetIt> init() async {
  final sharedPreferences = await registerModule.prefs;
  gh.factory<SharedPreferences>(() => sharedPreferences);
  return this;
}
```

**동작 방식:**

- configureDependencies()가 async 함수가 된다.
- main에서 await configureDependencies()로 호출해야 한다.

```dart
Future<void> main() async {
  WidgetsFlutterBinding.ensureInitialized();
  await configureDependencies();
  runApp(MyApp());
}
```

---

### @factoryParam - 런타임 파라미터 전달

객체 생성 시점에 동적으로 값을 전달할 수 있다.

```dart
@injectable
class ChatRoomController {
  ChatRoomController(
    ApiClient client,
    @factoryParam String roomId,
  );
}
```

**생성되는 코드:**

```dart
gh.factoryParam<ChatRoomController, String, dynamic>(
  (roomId, _) => ChatRoomController(getIt<ApiClient>(), roomId),
);
```

**사용 방법:**

```dart
final controller = getIt<ChatRoomController>(param1: 'room-123');
```

**중요한 제약:**

- 최대 2개의 파라미터까지만 지원된다.
- param1, param2로 순서대로 전달한다.

---

### @disposeMethod - 싱글턴 해제 시 정리

싱글턴이 제거될 때 자동으로 호출할 메서드를 지정한다.

```dart
@singleton
class WebSocketService {
  @disposeMethod
  void dispose() {
    _socket.close();
    print('WebSocket closed');
  }
}
```

**또는 외부 함수로 지정:**

```dart
@Singleton(dispose: disposeWebSocket)
class WebSocketService {}

FutureOr disposeWebSocket(WebSocketService instance) {
  instance.dispose();
}
```

**언제 호출되나:**

- getIt.reset()이나 getIt.resetLazySingleton() 호출 시.

---

### @Named - 동일 타입의 여러 구현체 구분

같은 인터페이스를 구현하는 여러 클래스를 등록할 때 사용한다.

```dart
abstract class PaymentService {}

@Named('card')
@Injectable(as: PaymentService)
class CardPaymentService implements PaymentService {}

@Named('cash')
@Injectable(as: PaymentService)
class CashPaymentService implements PaymentService {}
```

**생성되는 코드:**

```dart
gh.factory<PaymentService>(() => CardPaymentService(), instanceName: 'card');
gh.factory<PaymentService>(() => CashPaymentService(), instanceName: 'cash');
```

**사용 방법:**

```dart
@injectable
class PaymentController {
  PaymentController(@Named('card') PaymentService service);
}

// 또는 직접 호출
final cardService = getIt<PaymentService>(instanceName: 'card');
```

**@named (소문자) - 자동 태깅:**

```dart
@named
@Injectable(as: PaymentService)
class CardPaymentService implements PaymentService {}

@injectable
class Controller {
  Controller(@Named.from(CardPaymentService) PaymentService service);
}
```

이렇게 하면 클래스 이름이 자동으로 instanceName이 된다.

---

### @Environment - 환경별 등록

dev, prod, test 등 환경에 따라 다른 구현체를 주입할 수 있다.

```dart
const dev = Environment('dev');
const prod = Environment('prod');

@dev
@Injectable(as: ApiClient)
class MockApiClient implements ApiClient {}

@prod
@Injectable(as: ApiClient)
class RealApiClient implements ApiClient {}
```

**사용 방법:**

```dart
void configureDependencies() => getIt.init(environment: 'dev');
```

**여러 환경 지정:**

```dart
@Environment.dev
@Environment.test
@injectable
class DebugLogger {}
```

이 경우 dev 또는 test 환경에서 모두 등록된다.

---

### @module - 외부 라이브러리 등록

내가 만들지 않은 서드파티 클래스를 등록할 때 사용한다.

```dart
@module
abstract class ExternalModule {
  @lazySingleton
  Dio get dio => Dio(BaseOptions(baseUrl: 'https://api.example.com'));

  @preResolve
  Future<SharedPreferences> get prefs => SharedPreferences.getInstance();

  @singleton
  FlutterSecureStorage get secureStorage => const FlutterSecureStorage();
}
```

**파라미터를 받는 경우:**

```dart
@module
abstract class ExternalModule {
  @lazySingleton
  Dio provideDio(@Named('baseUrl') String url) {
    return Dio(BaseOptions(baseUrl: url));
  }
}
```

이렇게 하면 다른 의존성을 주입받아 외부 객체를 생성할 수 있다.

---

### @Order - 등록 순서 지정

의존성 등록 순서를 명시적으로 제어한다.

```dart
@Order(-1)
@singleton
class ConfigService {} // 가장 먼저 등록

@injectable
class NormalService {} // 기본 순서 (0)

@Order(1)
@singleton
class CleanupService {} // 가장 나중에 등록
```

**언제 사용하나:**

- 초기화 순서가 중요한 경우
- 특정 서비스가 다른 서비스보다 먼저 준비되어야 할 때

---

### @Scope - 스코프 기반 등록

특정 스코프에만 등록하고, 필요할 때만 초기화한다.

```dart
@Scope('auth')
@injectable
class AuthController {}

@Scope('auth')
@singleton
class TokenStorage {}
```

**사용 방법:**

```dart
void main() async {
  configureDependencies(); // 메인 스코프만 초기화

  // 로그인 성공 시
  await getIt.initAuthScope(); // auth 스코프 초기화

  // 로그아웃 시
  await getIt.resetScope('auth'); // auth 스코프 제거
}
```

**언제 사용하나:**

- 로그인/로그아웃 시 관련 서비스들을 한꺼번에 관리
- 메모리를 절약하고 싶을 때
- 특정 기능이 활성화될 때만 의존성 초기화

---

## 실전 활용 예시

### 전체 흐름 예제

```dart
// injection.dart
import 'package:get_it/get_it.dart';
import 'package:injectable/injectable.dart';
import 'injection.config.dart';

final getIt = GetIt.instance;

@InjectableInit()
Future<void> configureDependencies() async => await getIt.init();

// main.dart
Future<void> main() async {
  WidgetsFlutterBinding.ensureInitialized();
  await configureDependencies();
  runApp(MyApp());
}

// services/api_client.dart
@lazySingleton
class ApiClient {
  final Dio dio;
  ApiClient(this.dio);
}

// services/auth_service.dart
@singleton
class AuthService {
  final ApiClient client;
  final TokenStorage storage;

  AuthService(this.client, this.storage);

  @postConstruct
  Future<void> init() async {
    await loadToken();
  }

  @disposeMethod
  void dispose() {
    print('AuthService disposed');
  }
}

// repositories/user_repository.dart
@injectable
class UserRepository {
  final ApiClient client;

  UserRepository(this.client);

  @factoryMethod
  static UserRepository create(ApiClient client) {
    return UserRepository(client);
  }
}

// modules/external_module.dart
@module
abstract class ExternalModule {
  @preResolve
  Future<SharedPreferences> get prefs => SharedPreferences.getInstance();

  @lazySingleton
  Dio get dio => Dio(BaseOptions(
    baseUrl: 'https://api.example.com',
    connectTimeout: Duration(seconds: 5),
  ));
}
```

---

## 자동 등록 설정

build.yaml 파일을 만들어 패턴 기반 자동 등록도 가능하다.

```yaml
targets:
  $default:
    builders:
      injectable_generator:injectable_builder:
        options:
          auto_register: true
          class_name_pattern: "Service$|Repository$|Controller$"
          file_name_pattern: "_service$|_repository$"
```

이렇게 설정하면 Service, Repository, Controller로 끝나는 모든 클래스가 자동으로 @injectable 처리된다.

---

## 정리

injectable의 어노테이션은 다음과 같은 역할을 한다.

- **@injectable** → 매번 새 인스턴스 생성
- **@singleton** → 앱 시작 시 즉시 생성, 전역 공유
- **@lazySingleton** → 첫 사용 시 생성, 이후 재사용
- **@factoryMethod** → 특정 생성 메서드 지정
- **@postConstruct** → 생성 후 초기화 로직 실행
- **@preResolve** → 비동기 객체 사전 해결
- **@factoryParam** → 런타임 파라미터 전달
- **@disposeMethod** → 싱글턴 해제 시 정리
- **@Named** → 같은 타입의 여러 구현체 구분
- **@Environment** → 환경별 등록
- **@module** → 외부 라이브러리 등록
- **@Order** → 등록 순서 제어
- **@Scope** → 스코프 기반 생명주기 관리

이 어노테이션들을 적절히 조합하면 복잡한 DI 구조도 명확하게 관리할 수 있으며, 테스트와 유지보수가 훨씬 쉬워진다.
