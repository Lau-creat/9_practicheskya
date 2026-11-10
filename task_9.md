---
title: "Практическая работа #9: Асинхронное программирование в Dart"
discipline: "Разработка мобильных приложений"
tags:
  [
    dart,
    async,
    future,
    stream,
    stream-controller,
    isolates,
    event-loop,
    flutter,
  ]
---

# Практическая работа: Асинхронное программирование в Dart (Event Loop, Future, Stream, Isolates)

> [!NOTE]
> **Цель работы:** Глубоко освоить неблокирующую модель выполнения программ в Dart. Изучить архитектуру цикла событий (Event Loop), очереди микротасок и событий, работу с отложенными результатами (`Future`, `async/await`), реактивную обработку непрерывных потоков данных (`Stream`, `StreamController`, `async*`), а также параллельные вычисления в изолированных потоках памяти (`Isolates`).

---

## 📋 Содержание

1. [Модель параллелизма Dart: однопоточность и Event Loop](#1-модель-параллелизма-dart-однопоточность-и-event-loop)
2. [Тема 1. Отложенные вычисления: Future API и синтаксис async/await](#тема-1-отложенные-вычисления-future-api-и-синтаксис-asyncawait)
3. [Тема 2. Непрерывные потоки: Stream API и генераторы async\*](#тема-2-непрерывные-потоки-stream-api-и-генераторы-async)
4. [Тема 3. Реактивное управление: StreamController и StreamTransformer](#тема-3-реактивное-управление-streamcontroller-и-streamtransformer)
5. [Тема 4. Истинная многопоточность: Изоляты (Isolates) и Isolate.run](#тема-4-истинная-многопоточность-изоляты-isolates-и-isolaterun)
6. [50 Смешанных заданий по асинхронной разработке (Mobile Async Focus)](#50-смешанных-заданий-по-асинхронной-разработке)

---

## 1. Модель параллелизма Dart: однопоточность и Event Loop

По умолчанию код Dart выполняется в **одном потоке** (Main Isolate). Это исключает гонки данных (Race Conditions) и необходимость в блокирующих мьютексах, но требует неблокирующего подхода при операциях ввода-вывода (сеть, файлы, таймеры).

### Архитектура цикла событий (Event Loop)

```
                            СТАРТ ПРОГРАММЫ (main())
                                       │
                                       ▼
                     ┌───────────────────────────────────┐
                     │ Очередь микрозадач (Microtask)    │ ◄── Высший приоритет
                     │ (Внутренние системные колбэки)    │
                     └─────────────────┬─────────────────┘
                                       │ Пуста?
                                       ▼ Да
                     ┌───────────────────────────────────┐
                     │ Очередь событий (Event Queue)     │ ◄── Стандартный приоритет
                     │ (I/O, Таймеры, Тапы по экрану)    │
                     └─────────────────┬─────────────────┘
                                       │
                                       ▼
                             Исполнение задачи
                                       │
                                       └──────► Цикл повторяется
```

### Приоритет очередей

1. **Синхронный код:** Выполняется немедленно от первой до последней строчки.
2. **Microtask Queue:** Высокоприоритетные короткие системные задачи (`scheduleMicrotask()`). Очередь событий не начнет обрабатываться, пока очередь микрозадач не опустеет.
3. **Event Queue:** События пользовательского ввода (тапы, жесты), срабатывание таймеров (`Timer`), ответы сетевых сокетов и завершение чтения файлов.

---

## Тема 1. Отложенные вычисления: Future API и синтаксис async/await

`Future<T>` представляет собой контейнер для значения типа `T`, которое станет доступно через некоторое время, либо завершится ошибкой.

### Состояния Future

- **Uncompleted (В процессе):** Асинхронная операция еще выполняется.
- **Completed with Data (Успех):** Операция завершилась, вернув результат.
- **Completed with Error (Ошибка):** Во время выполнения выброшено исключение.

### Пример кода (Future, цепочки и параллельное ожидание)

```dart
// Имитация сетевого запроса к каталогу
Future<List<String>> fetchProductCatalog() async {
  await Future.delayed(const Duration(seconds: 2)); // Неблокирующая задержка

  bool hasNetworkError = false;
  if (hasNetworkError) {
    throw Exception("Ошибка 503: Сервер каталога временно недоступен");
  }

  return ["Смартфон Galaxy S26", "Планшет Tab Ultra", "Наушники Buds Pro"];
}

// Имитация загрузки профиля пользователя
Future<String> fetchUserProfile() async {
  await Future.delayed(const Duration(seconds: 1));
  return "Алексей Смирнов";
}

void main() async {
  print("1. Инициализация экрана...");

  try {
    // 1. Последовательное ожидание через async/await
    String user = await fetchUserProfile();
    print("2. Пользователь загружен: $user");

    // 2. Параллельный запуск нескольких Future через Future.wait()
    print("3. Старт параллельной загрузки дополнительных данных...");
    var results = await Future.wait([
      fetchProductCatalog(),
      Future.delayed(const Duration(milliseconds: 500), () => "Курсы валют OK"),
    ]);

    List<String> products = results[0] as List<String>;
    print("4. Товары получены: $products");

  } catch (error) {
    print("Перехвачена ошибка: $error");
  } finally {
    print("5. Завершение работы, скрытие ProgressBar");
  }

  // 3. Таймаут выполнения операции
  try {
    await fetchProductCatalog().timeout(
      const Duration(milliseconds: 500),
      onTimeout: () => throw Exception("Таймаут соединения! Сервер не ответил вовремя."),
    );
  } catch (e) {
    print("Контроль таймаута: $e");
  }
}
```

### Задания для закрепления

1. **Симулятор авторизации:** Напишите функцию `Future<bool> login(String user, String pass)`, которая через 1.5 секунды возвращает `true`, если пароль равен `"1234"`, и выбрасывает `AuthException` в противном случае.


<img width="1920" height="1080" alt="image" src="https://github.com/user-attachments/assets/b01fc483-f94b-4f4e-8e90-d7aab5a33b1a" />

```dart
import 'dart:async';

class AuthException implements Exception {
  final String message;
  AuthException(this.message);
  @override
  String toString() => 'AuthException: $message';
}

Future<bool> login(String user, String pass) async {
  await Future.delayed(const Duration(milliseconds: 1500));
  if (pass == "1234") {
    return true;
  } else {
    throw AuthException("Неверный пароль");
  }
}

void main() async {
  try {
    final result = await login("admin", "1234");
    print("Успешный вход: $result");
  } catch (e) {
    print(e);
  }

  try {
    final result = await login("admin", "wrong");
    print("Успешный вход: $result");
  } catch (e) {
    print(e);
  }
}
```

2. **Параллельное чтение файлов конфигурации:** Запустите параллельно три вызова `Future` с разным временем задержки с помощью `Future.wait()` и объедините их результаты в одну карту.


<img width="1920" height="1080" alt="image" src="https://github.com/user-attachments/assets/a07e188d-4e4d-44c2-af7f-7a940f154ce6" />

```dart
import 'dart:async';

class AuthException implements Exception {
  final String message;
  AuthException(this.message);
  @override
  String toString() => 'AuthException: $message';
}

Future<Map<String, String>> readConfig(String fileName, int delayMs) async {
  await Future.delayed(Duration(milliseconds: delayMs));
  return {fileName: "Данные из $fileName"};
}

void main() async {
  final results = await Future.wait([
    readConfig("config1.json", 1000),
    readConfig("config2.yaml", 1500),
    readConfig("config3.xml", 500),
  ]);

  final Map<String, String> combinedConfig = {};
  for (final map in results) {
    combinedConfig.addAll(map);
  }

  print(combinedConfig);
}
```

3. **Гонка асинхронных задач (Future.any):** Запустите запрос к двум зеркалам одного сервера (задержка 800 мс и 1200 мс) и получите ответ от самого быстрого с помощью `Future.any()`.


<img width="1920" height="1080" alt="image" src="https://github.com/user-attachments/assets/413f0b80-ea28-4066-be7e-170c964cb831" />

```dart
import 'dart:async';

class AuthException implements Exception {
  final String message;
  AuthException(this.message);
  @override
  String toString() => 'AuthException: $message';
}

Future<String> fetchServer(String mirror, int delayMs) async {
  await Future.delayed(Duration(milliseconds: delayMs));
  return "Ответ от $mirror";
}

void main() async {
  final result = await Future.any([
    fetchServer("Зеркало 1", 800),
    fetchServer("Зеркало 2", 1200),
  ]);
  print(result);
}

```

4. **Повторные попытки с задержкой (Retry Pattern):** Напишите функцию `retry(Future Function() task, int maxAttempts)`, которая при падении задачи повторяет вызов до 3 раз с паузой в 1 секунду.


<img width="1920" height="1080" alt="image" src="https://github.com/user-attachments/assets/ab0ca645-d962-47ec-9df1-463337dd9caa" />

```dart
import 'dart:async';

class AuthException implements Exception {
  final String message;
  AuthException(this.message);
  @override
  String toString() => 'AuthException: $message';
}

Future<T> retry<T>(Future<T> Function() task, int maxAttempts) async {
  int attempts = 0;
  while (true) {
    try {
      attempts++;
      return await task();
    } catch (e) {
      if (attempts >= maxAttempts) {
        rethrow;
      }
      await Future.delayed(const Duration(seconds: 1));
    }
  }
}

void main() async {
  int attemptCount = 0;
  Future<String> unstableTask() async {
    attemptCount++;
    if (attemptCount < 3) {
      print("Попытка $attemptCount провалилась");
      throw Exception("Ошибка сети");
    }
    return "Успех с попытки $attemptCount";
  }

  try {
    final result = await retry(unstableTask, 3);
    print(result);
  } catch (e) {
    print("Задача окончательно провалена: $e");
  }
}

```

5. **Цепочка промисов через then/catchError:** Реализуйте аналогичную логику авторизации без использования ключевых слов `async/await`, используя только методы `.then()`, `.catchError()` и `.whenComplete()`.


<img width="1920" height="1080" alt="image" src="https://github.com/user-attachments/assets/1af2cdc6-e6b6-4c2e-8646-9188d5effbae" />

```dart
class AuthException implements Exception {
  final String message;
  AuthException(this.message);
  
  @override
  String toString() => 'AuthException: $message';
}

// Функция авторизации (реализована без async/await для чистоты эксперимента)
Future<bool> login(String user, String pass) {
  return Future.delayed(const Duration(milliseconds: 1500), () {
    if (pass == "1234") {
      return true;
    } else {
      throw AuthException("Неверный пароль для пользователя $user");
    }
  });
}

void main() {
  print("Начало авторизации...");

  // Цепочка промисов
  login("admin", "1234") // Попробуйте поменять пароль на неправильный, чтобы проверить catchError
      .then((bool isSuccess) {
        // Блок выполняется при успешном завершении Future
        if (isSuccess) {
          print("Успех: Пользователь авторизован.");
        }
      })
      .catchError((Object error) {
        // Блок выполняется при возникновении ошибки (исключения)
        if (error is AuthException) {
          print("Ошибка авторизации: ${error.message}");
        } else {
          print("Произошла неизвестная ошибка: $error");
        }
      })
      .whenComplete(() {
        // Блок выполняется в любом случае (и при успехе, и при ошибке)
        print("Завершение процесса авторизации (скрытие лоадера и т.д.).");
      });
}
```

---

## Тема 2. Непрерывные потоки: Stream API и генераторы async\*

Если `Future` возвращает **однократное** значение, то `Stream<T>` поставляет **последовательность** асинхронных событий во времени (клики, показания датчиков GPS, сообщения сокета).

### Два типа потоков

1. **Single-Subscription Stream (Одиночный поток):** Может иметь только одного слушателя за все время жизни (например, чтение файла по кускам). Повторный вызов `.listen()` приведет к исключению `StateError`.
2. **Broadcast Stream (Широковещательный поток):** Позволяет подключать множество независимых подписчиков одновременно (события UI, шина аналитики). Создается через `stream.asBroadcastStream()`.

### Пример кода (Генератор async\* и обработка потока)

```dart
// Асинхронный генератор: возвращает непрерывный поток данных
Stream<int> countdownTimer(int fromSeconds) async* {
  for (int i = fromSeconds; i >= 0; i--) {
    await Future.delayed(const Duration(seconds: 1));
    yield i; // Отправка значения в поток
  }
}

void main() async {
  print("Старт таймера:");

  Stream<int> timerStream = countdownTimer(5);

  // Функциональные операторы над потоком (подобно коллекциям)
  StreamSubscription<String> subscription = timerStream
      .where((seconds) => seconds % 2 == 0) // Фильтруем только четные секунды
      .map((seconds) => "Осталось четных секунд: $seconds")
      .listen(
        (data) => print("[DATA] $data"),
        onError: (err) => print("[ERROR] $err"),
        onDone: () => print("[DONE] Отсчет завершен!"),
        cancelOnError: false,
      );

  // Отмена подписки через 3.5 секунды для предотвращения утечки ресурсов
  await Future.delayed(const Duration(milliseconds: 3500));
  print("Принудительная отмена подписки слушателем!");
  await subscription.cancel();
}
```

### Задания для закрепления

1. **Поток тиков секундомера:** Создайте функцию `Stream<int> stopWatch() async*`, непрерывно генерирующую инкрементный счетчик каждую секунду.


<img width="1920" height="1080" alt="image" src="https://github.com/user-attachments/assets/bcec5386-9ba5-4d17-9e88-4d3ad3622383" />

```dart
Stream<int> stopwatch() async* {
  int counter = 0;
  while (true) {
    await Future.delayed(const Duration(seconds: 1));
    yield counter++;
  }
}

void main() async {
  final sub = stopwatch().listen((tick) {
    print('Тик: $tick');
  });
  await Future.delayed(const Duration(seconds: 5));
  await sub.cancel();
}
```

2. **Поток GPS-координат (Имитация):** Сгенерируйте поток кортежей `(double lat, double lon)`, слегка изменяющих координаты каждые 500 мс.


<img width="1920" height="1080" alt="image" src="https://github.com/user-attachments/assets/b020d62c-3913-461d-86d9-97c7fccb4ce0" />

```dart
Stream<(double, double)> gpsStream() async* {
  double lat = 55.7558;
  double lon = 37.6173;
  while (true) {
    await Future.delayed(const Duration(milliseconds: 500));
    lat += 0.0001;
    lon += 0.0001;
    yield (lat, lon);
  }
}

void main() async {
  final sub = gpsStream().listen((coords) {
    print('GPS: (${coords.$1}, ${coords.$2})');
  });
  await Future.delayed(const Duration(seconds: 2));
  await sub.cancel();
}

```

3. **Широковещательный стрим котировок:** Создайте однократный поток цен, преобразуйте его в Broadcast через `.asBroadcastStream()` и подключите двух разных слушателей: UI-виджет и логгер аналитики.


<img width="1920" height="1080" alt="image" src="https://github.com/user-attachments/assets/fdf9b717-9934-4798-b6a7-85118a03ac31" />

```dart
void main() async {
  final singleStream = Stream.fromIterable([100.0, 105.5, 102.3, 110.0]);
  final broadcastStream = singleStream.asBroadcastStream();

  broadcastStream.listen((price) {
    print('UI-виджет: Обновление цены - $price');
  });

  broadcastStream.listen((price) {
    print('Логгер аналитики: Зафиксирована цена - $price');
  });

  await Future.delayed(const Duration(milliseconds: 100));
}
```

4. **Агрегация потока (fold/reduce):** Посчитайте сумму чисел, поступающих из потока из 10 элементов, с помощью метода `stream.fold()`.


<img width="1920" height="1080" alt="image" src="https://github.com/user-attachments/assets/21a43283-822e-4812-b7a9-de5b586e5d14" />

```dart
void main() async {
  final stream = Stream.fromIterable(List.generate(10, (index) => index + 1));
  final sum = await stream.fold(0, (previous, element) => previous + element);
  print('Сумма элементов потока: $sum');
}
```

5. **Делегирование генератора через yield\*:** Напишите функцию, которая объединяет два потока последовательно с помощью синтаксиса `yield* subStream`.


<img width="1920" height="1080" alt="image" src="https://github.com/user-attachments/assets/6afd49b9-e501-4dc0-8de3-6c0f19614c2f" />

```dart
Stream<int> subStream1() async* {
  yield 1;
  yield 2;
  yield 3;
}

Stream<int> subStream2() async* {
  yield 4;
  yield 5;
  yield 6;
}

Stream<int> combinedStream() async* {
  yield* subStream1();
  yield* subStream2();
}

void main() async {
  await for (final value in combinedStream()) {
    print('Значение: $value');
  }
}
```
---

## Тема 3. Реактивное управление: StreamController и StreamTransformer

`StreamController<T>` — это программный пульт управления потоком, разделенный на две части:

- **`Sink` (Входной порт):** Место, куда события отправляются (`sink.add()`, `sink.addError()`).
- **`Stream` (Выходной порт):** Поток, на который подписываются клиенты для чтения данных.

```
                           STREAM CONTROLLER
        ┌─────────────────────────────────────────────────────┐
        │                                                     │
Events ─┼──► [ Sink (add / addError) ]                        │
        │                  │                                  │
        │                  ▼                                  │
        │         Внутренний буфер                            │
        │                  │                                  │
        │                  ▼                                  │
        │         [ Stream (listen) ] ──► Subscribers (UI)    │
        │                                                     │
        └─────────────────────────────────────────────────────┘
```

### Пример кода (Простейший реактивный BLoC счетчика без сторонних библиотек)

```dart
import 'dart:async';

class CounterBloc {
  int _counter = 0;

  // Контроллер с поддержкой нескольких слушателей (Broadcast)
  final _stateController = StreamController<int>.broadcast();

  // Публичный поток только для чтения UI-компонентами
  Stream<int> get counterStream => _stateController.stream;

  void increment() {
    _counter++;
    _stateController.sink.add(_counter); // Отправка нового стейта
  }

  void reset() {
    _counter = 0;
    _stateController.sink.add(_counter);
  }

  // Обязательное закрытие контроллера при уничтожении экрана
  void dispose() {
    _stateController.close();
  }
}

void main() async {
  final bloc = CounterBloc();

  final sub1 = bloc.counterStream.listen((val) => print("UI Экран получил: $val"));
  final sub2 = bloc.counterStream.listen((val) => print("Аналитика зафиксировала: $val"));

  bloc.increment(); // 1
  bloc.increment(); // 2
  bloc.reset();     // 0

  await Future.delayed(const Duration(milliseconds: 100));
  await sub1.cancel();
  await sub2.cancel();
  bloc.dispose();
}
```

### Задания для закрепления

1. **Кастомный StreamTransformer:** Реализуйте трансформер строк через `StreamTransformer.fromHandlers()`, переводящий все входящие сообщения в верхний регистр.


<img width="1920" height="1080" alt="image" src="https://github.com/user-attachments/assets/fbc25b1a-6dd7-4689-a1f4-7ad368bee6da" />

```dart
import 'dart:async';

void main() async {
  final transformer = StreamTransformer<String, String>.fromHandlers(
    handleData: (data, sink) {
      sink.add(data.toUpperCase());
    },
  );

  final stream = Stream.fromIterable(['hello', 'dart', 'streams']);
  
  await for (final value in stream.transform(transformer)) {
    print(value);
  }
}
```

2. **Контроллер поискового ввода:** Реализуйте класс `SearchControllerWrapper`, принимающий посимвольный ввод пользователя в Sink и отдающий очищенную строку в Stream.


<img width="1920" height="1080" alt="image" src="https://github.com/user-attachments/assets/5ddf2895-aff9-4820-af94-29c2bde25954" />

```dart
import 'dart:async';

class SearchControllerWrapper {
  final _controller = StreamController<String>();
  
  Sink<String> get sink => _controller.sink;
  
  Stream<String> get stream => _controller.stream.transform(
    StreamTransformer.fromHandlers(
      handleData: (data, sink) {
        sink.add(data.trim().toLowerCase());
      },
    ),
  );
  
  void dispose() {
    _controller.close();
  }
}

void main() {
  final wrapper = SearchControllerWrapper();
  
  wrapper.stream.listen((cleaned) {
    print('Очищенная строка: "$cleaned"');
  });
  
  wrapper.sink.add(' ПриВет МиР ');
  wrapper.sink.add(' FlUtTeR DaRt ');
}
```

3. **Обработка ошибок в контроллере:** Отправьте ошибку в контроллер с помощью `sink.addError(Exception("Сбой датчика"))` и перехватите её в секции `onError` слушателя.


<img width="1920" height="1080" alt="image" src="https://github.com/user-attachments/assets/cd944263-8b61-474f-9ac4-6ffecd4c5362" />

```dart
import 'dart:async';

void main() {
  final controller = StreamController<String>();
  
  controller.stream.listen(
    (data) {
      print('Данные: $data');
    },
    onError: (error) {
      print('Перехвачена ошибка: $error');
    },
  );
  
  controller.sink.add('Нормальное значение');
  controller.sink.addError(Exception("Сбой датчика"));
  controller.sink.add('Следующее значение');
  
  controller.close();
}
```

4. **Контроль состояния закрытия:** Напишите метод, проверяющий свойство `controller.isClosed` перед попыткой добавить новое событие, предотвращая падение с ошибкой.


<img width="1920" height="1080" alt="image" src="https://github.com/user-attachments/assets/cbca7afc-d375-4dc7-968c-962617c01f41" />

```dart
import 'dart:async';

void safeAddEvent(StreamController controller, dynamic data) {
  if (!controller.isClosed) {
    controller.add(data);
    print('Событие "$data" успешно добавлено');
  } else {
    print('Ошибка: Невозможно добавить "$data", контроллер закрыт');
  }
}

void main() {
  final controller = StreamController<String>();
  
  safeAddEvent(controller, 'Событие 1');
  
  controller.close();
  
  safeAddEvent(controller, 'Событие 2');
}
```

5. **Счетчик активных подписчиков:** Сконфигурируйте `StreamController(onListen: ..., onCancel: ...)` для отслеживания момента появления и ухода подписчиков.


<img width="1920" height="1080" alt="image" src="https://github.com/user-attachments/assets/933fc0df-4a7b-4505-bfd7-36f5f7680852" />

```dart
import 'dart:async';

void main() async {
  int activeListeners = 0;
  
  final controller = StreamController<int>.broadcast(
    onListen: () {
      activeListeners++;
      print('Появился подписчик. Всего активных: $activeListeners');
    },
    onCancel: () {
      activeListeners--;
      print('Подписчик ушел. Всего активных: $activeListeners');
    },
  );
  
  final sub1 = controller.stream.listen((data) => print('Sub1: $data'));
  final sub2 = controller.stream.listen((data) => print('Sub2: $data'));
  
  await Future.delayed(const Duration(milliseconds: 100));
  
  await sub1.cancel();
  
  await Future.delayed(const Duration(milliseconds: 100));
  
  await sub2.cancel();
  
  controller.close();
}
```

---

## Тема 4. Истинная многопоточность: Изоляты (Isolates) и Isolate.run

Если задача требует непрерывных тяжелых математических расчетов (например, парсинг 50-мегабайтного JSON, фильтрация графических пикселей, шифрование больших баз данных), она заблокирует Event Loop главного потока, вызвав зависание мобильного экрана (Jank).

Для тяжелых фоновых вычислений Dart использует **Изоляты** (Isolates).

### Отличие изолятов от традиционных потоков Java/C++

- Изоляты **НЕ имеют общей разделяемой памяти** (Shared Memory отсутствует).
- Нет блокировок, взаимных дедлоков и мьютексов.
- Общение между изолятами происходит исключительно через передачу сообщений по системным портам (`SendPort` и `ReceivePort`), либо через метод `Isolate.run()` (начиная с Dart 2.19+).

```
   MAIN ISOLATE (UI & Event Loop)             BACKGROUND WORKER ISOLATE
┌─────────────────────────────────┐       ┌─────────────────────────────────┐
│ Своя изолированная память       │       │ Своя изолированная память       │
│                                 │       │                                 │
│ [SendPort] ─── Message (Copy) ──┼──────►│ [ReceivePort]                   │
│                                 │       │                                 │
│                                 │       │ Тяжелый расчет:                 │
│                                 │       │ парсинг JSON / сжатие фото      │
│                                 │       │                                 │
│ [ReceivePort] ◄── Result ───────┼───────┤ [SendPort]                      │
└─────────────────────────────────┘       └─────────────────────────────────┘
```

### Пример кода: Isolate.run() против низкоуровневых портов

```dart
import 'dart:isolate';

// Тяжелая функция верхнего уровня (Top-level или static)
int heavyCalculation(int iterations) {
  int sum = 0;
  for (int i = 0; i < iterations; i++) {
    sum += (i % 3);
  }
  return sum;
}

void main() async {
  print("1. Главный поток свободен, UI отрисовывается со скоростью 120 FPS");

  // Способ 1: Современный компактный Isolate.run() (Dart 2.19 / 3+)
  print("2. Запуск тяжелого расчета в параллельном изоляте...");
  int result = await Isolate.run(() => heavyCalculation(500000000));
  print("3. Результат из Isolate.run: $result");

  // Способ 2: Низкоуровневые порты для длительного двустороннего обмена
  final receivePort = ReceivePort();
  await Isolate.spawn(_workerEntryPoint, receivePort.sendPort);

  // Ожидаем ответ от запущенного воркера
  final workerMessage = await receivePort.first;
  print("4. Сообщение от постоянного воркера: $workerMessage");
}

void _workerEntryPoint(SendPort sendPort) {
  // Выполнение фоновой задачи
  int calculatedValue = heavyCalculation(100000);
  sendPort.send("Готово! Вычислено: $calculatedValue");
}
```

### Задания для закрепления

1. **Параллельный парсер JSON:** Напишите функцию, принимающую строку из 10 000 фиктивных записей и десериализующую ее в объекты через `Isolate.run()`.


<img width="1920" height="1080" alt="image" src="https://github.com/user-attachments/assets/1c076bbd-82aa-41c6-b928-7281a3202d26" />

```dart
import 'dart:async';
import 'dart:convert';
import 'dart:isolate';

List<dynamic> parseJsonData(String jsonStr) {
  return jsonDecode(jsonStr);
}

void main() async {
  final data = List.generate(10000, (i) => {'id': i, 'name': 'User $i'});
  final jsonString = jsonEncode(data);
  
  print('Начало парсинга...');
  final result = await Isolate.run(() => parseJsonData(jsonString));
  print('Успешно распарсено записей: ${result.length}');
}
```

2. **Поиск простых чисел в фоне:** Запустите вычисление всех простых чисел до 1 000 000 в фоновом изоляте, параллельно выводя в консоль главного потока точки каждую секунду.


<img width="1920" height="1080" alt="image" src="https://github.com/user-attachments/assets/43c66847-7846-469c-8d8a-8b3071e69664" />

```dart
import 'dart:async';
import 'dart:convert';
import 'dart:isolate';

List<int> findPrimes(int max) {
  final primes = <int>[];
  for (int i = 2; i <= max; i++) {
    bool isPrime = true;
    for (int j = 2; j * j <= i; j++) {
      if (i % j == 0) {
        isPrime = false;
        break;
      }
    }
    if (isPrime) primes.add(i);
  }
  return primes;
}

void main() async {
  print('Запуск поиска простых чисел...');
  final timer = Timer.periodic(const Duration(seconds: 1), (_) => print('.'));
  
  final primes = await Isolate.run(() => findPrimes(1000000));
  
  timer.cancel();
  print('\nНайдено простых чисел: ${primes.length}');
}
```

3. **Двусторонний обмен через порты:** Создайте изолят-калькулятор, который слушает команды по `SendPort`, выполняет операции сложения и отправляет ответ обратно.


<img width="1920" height="1080" alt="image" src="https://github.com/user-attachments/assets/b18ea99e-7106-4791-9a10-54c2e9cb3faf" />

```dart
import 'dart:async';
import 'dart:convert';
import 'dart:isolate';

void calculatorIsolate(SendPort mainSendPort) {
  final receivePort = ReceivePort();
  mainSendPort.send(receivePort.sendPort);
  
  receivePort.listen((message) {
    if (message is List && message.length == 2) {
      final a = message[0] as int;
      final b = message[1] as int;
      mainSendPort.send(a + b);
    }
  });
}

void main() async {
  final mainReceivePort = ReceivePort();
  await Isolate.spawn(calculatorIsolate, mainReceivePort.sendPort);
  
  SendPort? childSendPort;
  
  mainReceivePort.listen((message) {
    if (message is SendPort) {
      childSendPort = message;
      print('Порт изолята-калькулятора получен. Отправляем команду: 15 + 25');
      childSendPort!.send([15, 25]);
    } else if (message is int) {
      print('Ответ от изолята: $message');
      mainReceivePort.close();
    }
  });
}

```

4. **Замер падения FPS:** Напишите скрипт, запускающий цикл на 2 млрд итераций сначала синхронно в `main()`, а затем в `Isolate.run()`, сравнив отзывчивость таймеров.


<img width="1920" height="1080" alt="image" src="https://github.com/user-attachments/assets/7f5647d7-3448-4606-8b2f-db361cedd9fb" />

```dart
import 'dart:async';
import 'dart:convert';
import 'dart:isolate';

void heavyLoop() {
  int sum = 0;
  for (int i = 0; i < 2000000000; i++) {
    sum += i;
  }
}

void main() async {
  print('--- Синхронный запуск (главный поток заблокирован) ---');
  final timer1 = Timer.periodic(const Duration(milliseconds: 100), (_) => print('Тик синхронный'));
  
  heavyLoop();
  
  timer1.cancel();
  print('Синхронный цикл завершен\n');
  
  print('--- Асинхронный запуск (главный поток свободен) ---');
  final timer2 = Timer.periodic(const Duration(milliseconds: 100), (_) => print('Тик асинхронный'));
  
  await Isolate.run(heavyLoop);
  
  timer2.cancel();
  print('Асинхронный цикл завершен');
}
```

5. **Принудительное уничтожение изолята:** Запустите бесконечный фоновый изолят через `Isolate.spawn` и принудительно завершите его работу методом `isolate.kill(priority: Isolate.immediate)`.


<img width="1920" height="1080" alt="image" src="https://github.com/user-attachments/assets/db8edb0c-3077-4f00-b8d3-92df34f0c5ef" />

```dart
import 'dart:async';
import 'dart:convert';
import 'dart:isolate';

void infiniteBackgroundTask() {
  int counter = 0;
  while (true) {
    if (counter % 50000000 == 0) {
      print('Фоновый изолят активен. Счетчик: $counter');
    }
    counter++;
  }
}

void main() async {
  print('Запуск бесконечного изолята...');
  final isolate = await Isolate.spawn(infiniteBackgroundTask, null);
  
  await Future.delayed(const Duration(seconds: 2));
  
  print('Принудительное уничтожение изолята...');
  isolate.kill(priority: Isolate.immediate);
  print('Изолят успешно уничтожен.');
}
```

---

## 50 Смешанных заданий по асинхронной разработке

Задания охватывают реальные сценарии клиентской мобильной разработки (REST API, WebSocket, фоновые очереди, кэширование, датчики, архитектура MVI/BLoC).

### Блок 1: Сетевые запросы, таймеры и Future

1. **Имитатор HTTP GET запроса:** Напишите функцию `fetchData(String url)`, возвращающую JSON через 1200 мс с обработкой статусов 200, 404 и 500.


```dart

```

2. **Экспоненциальная задержка (Exponential Backoff):** Реализуйте повтор сетевого запроса с прогрессирующей паузой: 1с, 2с, 4с, 8с при возникновении `SocketException`.


```dart

```

3. **Параллельная загрузка медиафайлов:** Скачайте метаданные для 5 картинок одновременно через `Future.wait()`, ограничив параллелизм до 2 одновременных вызовов.


```dart

```

4. **Валидатор доступности сервера (Ping):** Отправьте пинг-запросы к трем зеркалам и верните URL первого ответившего сервера (`Future.any`).


```dart

```

5. **Прерываемый запрос по таймауту:** Оберните вызов длительного сохранения профиля в таймаут 3 секунды с аварийным прерыванием операции.


```dart

```

6. **Очередь последовательного выполнения (Async Queue):** Спроектируйте класс, выполняющий поступающие асинхронные задачи строго по очереди друг за другом (FIFO).


```dart

```

7. **Асинхронный замер времени выполнения:** Напишите утилиту `benchmark(Future Function() task)`, замеряющую точное время выполнения асинхронного блока в миллисекундах (`Stopwatch`).


```dart

```

8. **Ленивый асинхронный синглтон:** Реализуйте инициализацию конфигурации приложения `AppConfig.getInstance()`, выполняющую чтение данных из асинхронного источника ровно один раз.


```dart

```

9. **Параллельное слияние локального и сетевого кэша:** Запустите чтение локальной базы SQLite и сетевой вызов одновременно, вернув локальные данные, если сеть не успела ответить за 300 мс.


```dart

```

10. **Асинхронный дебаунсер (Debounce):** Напишите функцию, откладывающую выполнение сетевого поиска на 400 мс и сбрасывающую таймер при каждом новом нажатии клавиши.


```dart

```

### Блок 2: Потоковые данные, датчики и Stream API

11. **Слушатель котировок криптовалют:** Смоделируйте WebSocket-поток, передающий случайную цену биткоина раз в 200 мс, и отфильтруйте всплески более 5%.


```dart

```

12. **Стриминг прогресса скачивания файла:** Напишите асинхронный генератор `async*`, передающий процент загрузки от 0% до 100% с шагом в 10% каждые 300 мс.


```dart

```

13. **Объединение двух потоков (Merge):** Создайте функцию, объединяющую события из потока геолокации и потока компаса в единый поток навигации.


```dart

```

14. **Дросселирование потока (Throttle):** Реализуйте оператор, пропускающий не более одного клика по кнопке в секунду из непрерывного потока нажатий.


```dart

```

15. **Поток шагов акселерометра:** Сгенерируйте поток показаний ускорения по трем осям, подсчитывающий число шагов при превышении порогового значения.


```dart

```

16. **Автоматическое переподключение потока (Reconnection):** Напишите слушатель стрима чата, который при возникновении ошибки автоматически пересоздает подписку через 2 секунды.


```dart

```

17. **Буферизация потока пачками (Buffer / Chunk):** Накапливайте события кликов в список и отдавайте их пачками по 5 штук или раз в 3 секунды.


```dart

```

18. **Поток обратного отсчета для OTP-кода:** Создайте стрим таймера на 60 секунд для экрана ввода SMS с уведомлением о завершении срока жизни кода.


```dart

```

19. **Потоковый калькулятор скользящего среднего:** Спроектируйте стрим, принимающий числа и отдающий скользящее среднее значение последних 5 измерений.


```dart

```

20. **Преобразование Stream в Future:** Извлеките последнее значение из конечного потока с помощью метода `stream.last`.


```dart

```

### Блок 3: Реактивные шины событий и StreamController

21. **Шина глобальных событий (EventBus):** Создайте синглтон `EventBus` на базе широковещательного `StreamController` для обмена событиями между несвязанными экранами.


```dart

```

22. **Реактивная корзина интернет-магазина:** Реализуйте класс `CartStore`, транслирующий в стрим текущее количество товаров и итоговую сумму при вызовах `add()` и `remove()`.


```dart

```

23. **Счетчик непрочитанных сообщений:** Спроектируйте контроллер бейджа уведомлений, увеличивающий счетчик при новых событиях и сбрасывающий его в ноль при прочтении.


```dart

```

24. **Управление состоянием аудиоплеера:** Реализуйте `AudioPlayerBloc` со стримом состояний: `PlayerStopped`, `PlayerPlaying(track)`, `PlayerPaused`.


```dart

```

25. **Контроллер бесконечной ленты (Pagination Stream):** Спроектируйте стрим, подгружающий следующую страницу новостей при передаче события `LoadNextPage`.


```dart

```

26. **Реактивный переключатель темной темы:** Создайте менеджер темы приложения, отдающий `ThemeMode.dark` или `ThemeMode.light` через `BehaviorSubject`-подобный контроллер.


```dart

```

27. **Слушатель сетевого статуса (Connectivity Stream):** Контроллер, транслирующий изменения подключения (`WiFi`, `Cellular`, `None`) во все подписанные модули.


```dart

```

28. **Защита контроллера от утечек памяти (Auto-Dispose):** Напишите класс с механизмом проверки наличия подписчиков и автоматическим вызовом `close()` при обнулении слушателей.


```dart

```

29. **Валидация формы регистрации на стримах:** Создайте два потока для полей Email и Пароль, объедините их через `Rx.combineLatest`-подобную логику и отдавайте флаг активности кнопки `isSubmitEnabled`.


```dart

```

30. **Потоковый логгер крашей:** Направляйте все неотловленные ошибки приложения через `StreamController` в фоновый модуль отправки аналитики.


```dart

```

### Блок 4: Изоляты и ресурсоемкие параллельные задачи

31. **Фоновая генерация миниатюр (Thumbnail Generator):** Обработайте массив байтов изображения в `Isolate.run()`, изменив его размер и вернув сжатый массив.


```dart

```

32. **Шифрование базы заметок:** Выполните шифрование большого текстового файла по алгоритму XOR / AES в параллельном изоляте без просадки FPS.


```dart

```

33. **Полнотекстовый поиск по 100 000 документов:** Передайте поисковый запрос и список документов в фоновый изолят, вернув 10 наиболее релевантных совпадений.


```dart

```

34. **Парсинг тяжелого GeoJSON карты города:** Распарсите координаты полигонов районов города в фоновом потоке памяти и верните готовые структуры данных.


```dart

```

35. **Сжатие логов в ZIP-архив:** Смоделируйте упаковку текстовых журналов в сжатый архив через изолят перед отправкой в техподдержку.


```dart

```

36. **Двусторонний пул воркеров (Worker Pool):** Создайте менеджер из двух постоянно работающих изолятов, распределяющий между ними математические задачи.


```dart

```

37. **Генерация PDF-отчета:** Сформируйте текстовую структуру финансового отчета за год в параллельном изоляте.


```dart

```

38. **Сортировка массива из 1 000 000 чисел:** Сравните время сортировки массива в главном потоке и в отдельном изоляте.


```dart

```

39. **Конвертер аудио-семплов:** Смоделируйте преобразование массива сырых аудио-семплов в изоляте.


```dart

```

40. **Безопасная передача больших объектов (TransferableTypedData):** Передайте большой бинарный буфер между изолятами без затратного копирования памяти.


```dart

```

### Блок 5: Сложные асинхронные архитектуры и отказоустойчивость

41. **Отмена сетевой операции через CancelToken:** Реализуйте класс токена отмены, выбрасывающий `CancellationException` при прерывании пользователем ухода со страницы.


```dart

```

42. **Репликация базы данных с разрешением конфликтов:** Синхронизируйте две асинхронные очереди изменений с временными метками (Timestamps).


```dart

```

43. **Асинхронная очередь сохранения офлайн-заказов:** Сохраняйте оформленные без интернета заказы в локальную очередь и автоматически отправляйте их по одному при восстановлении сети.


```dart

```

44. **Паттерн Circuit Breaker:** Реализуйте защиту от падения сервера: если 5 запросов подряд завершились ошибкой, переведите сервис в режим «Разрыв цепи» на 30 секунд.


```dart

```

45. **Асинхронный семафор (Rate Limiter):** Ограничьте число одновременных сетевых обращений к API до 3 параллельных потоков.


```dart

```

46. **Гонка состояний при быстром вводе (Race Condition Fix):** Защитите поисковую строку: гарантируйте, что старый медленный ответ сервера не перезапишет новый быстрый ответ.


```dart

```

47. **Слушатель жизненного цикла приложения:** Свяжите события сворачивания приложения с мгновенной приостановкой всех активных `StreamSubscription`.


```dart

```

48. **Конечный автомат асинхронной синхронизации:** Реализуйте переключение стейтов `Idle` -> `Syncing(progress)` -> `Success` -> `Error` с защитой от повторного запуска во время синхронизации.


```dart

```

49. **Многоуровневый кэш (Memory -> Disk -> Network):** Спроектируйте функцию, последовательно проверяющую быструю оперативную память, затем локальный диск и только потом идущую в сеть.


```dart

```

50. **Итоговый реактивный модуль финансового терминала (Production Architecture):**
    - Подключите непрерывный поток биржевых котировок через WebSocket-имитатор.
    - Передайте расчет скользящей волатильности в параллельный изолят (`Isolate.run`).
    - Ограничьте частоту обновления графического экрана до 60 кадров/сек с помощью троттлинга.
    - Реализуйте автоматическое восстановление соединения при обрыве и безопасный вызов `dispose()` для всех контроллеров и подписок.


```dart

```

---
