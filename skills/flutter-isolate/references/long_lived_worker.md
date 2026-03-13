# Long-lived Worker Isolate

A production-ready `Worker` class that handles multiple concurrent requests via `Completer`-based sequencing, error propagation, and clean port shutdown.

## When to Use This Pattern

- You send many requests to the same isolate over the lifetime of the app (avoids repeated spawn overhead).
- You need the isolate to stay alive and accumulate state between calls.
- You need concurrent requests (fire multiple `parseJson` calls without waiting for each one).

## Complete Implementation

```dart
import 'dart:async';
import 'dart:convert';
import 'dart:isolate';

class Worker {
  final SendPort _commands;
  final ReceivePort _responses;
  final Map<int, Completer<Object?>> _activeRequests = {};
  int _idCounter = 0;
  bool _closed = false;

  // --- Public API ---

  /// Spawns the background isolate and returns a ready Worker instance.
  static Future<Worker> spawn() async {
    // RawReceivePort is used for the handshake only — it has no type overhead.
    final initPort = RawReceivePort();
    final connection = Completer<(ReceivePort, SendPort)>.sync();

    // The very first message from the worker isolate is its SendPort.
    initPort.handler = (initialMessage) {
      final commandPort = initialMessage as SendPort;
      connection.complete((
        ReceivePort.fromRawReceivePort(initPort),
        commandPort,
      ));
    };

    try {
      await Isolate.spawn(_startRemoteIsolate, initPort.sendPort);
    } on Object {
      initPort.close();
      rethrow;
    }

    final (ReceivePort receivePort, SendPort sendPort) =
        await connection.future;
    return Worker._(receivePort, sendPort);
  }

  /// Sends a JSON string to the worker, returns the decoded value.
  Future<Object?> parseJson(String message) async {
    if (_closed) throw StateError('Worker is closed');
    final completer = Completer<Object?>.sync();
    final id = _idCounter++;
    _activeRequests[id] = completer;
    // Send as a Dart record — bundles id + payload in a single message.
    _commands.send((id, message));
    return completer.future;
  }

  /// Shuts down the worker gracefully after all pending requests complete.
  void close() {
    if (_closed) return;
    _closed = true;
    _commands.send('shutdown');
    if (_activeRequests.isEmpty) _responses.close();
  }

  // --- Private ---

  Worker._(this._responses, this._commands) {
    _responses.listen(_handleResponsesFromIsolate);
  }

  void _handleResponsesFromIsolate(dynamic message) {
    final (int id, Object? response) = message as (int, Object?);
    final completer = _activeRequests.remove(id)!;

    if (response is RemoteError) {
      completer.completeError(response);
    } else {
      completer.complete(response);
    }

    // Close the response port once all requests are done and we're shutting down.
    if (_closed && _activeRequests.isEmpty) _responses.close();
  }

  // --- Worker isolate entry points (must be static/top-level) ---

  static void _startRemoteIsolate(SendPort sendPort) {
    final receivePort = ReceivePort();
    // Send our SendPort back so the main isolate can reach us.
    sendPort.send(receivePort.sendPort);
    _handleCommandsToIsolate(receivePort, sendPort);
  }

  static void _handleCommandsToIsolate(
    ReceivePort receivePort,
    SendPort sendPort,
  ) {
    receivePort.listen((message) {
      if (message == 'shutdown') {
        receivePort.close();
        return;
      }
      final (int id, String jsonText) = message as (int, String);
      try {
        final jsonData = jsonDecode(jsonText);
        sendPort.send((id, jsonData));
      } catch (e) {
        sendPort.send((id, RemoteError(e.toString(), '')));
      }
    });
  }
}
```

## Usage

```dart
void main() async {
  final worker = await Worker.spawn();

  // Sequential calls
  print(await worker.parseJson('{"key":"value"}'));
  print(await worker.parseJson('"banana"'));

  // Concurrent calls — responses matched correctly by id
  final results = await Future.wait([
    worker.parseJson('"yes"'),
    worker.parseJson('"no"'),
  ]);
  print(results);

  worker.close();
}
```

## Key Design Decisions

| Decision | Reason |
|---|---|
| `RawReceivePort` for handshake | Separates startup from message-passing logic; lower overhead |
| `Completer.sync()` | Response is delivered synchronously on the event loop, avoiding extra microtask delay |
| Dart Record `(id, message)` | Bundles two values into a single `SendPort.send()` call cleanly |
| `'shutdown'` sentinel string | Tells the worker to `close()` its own `ReceivePort`, ending the isolate naturally |
| `RemoteError` propagation | Surfaces worker-side exceptions back to the caller's `await` as real errors |

## Adapting to Your Domain

Replace `jsonDecode` with your CPU-intensive operation (image processing, encryption, etc.):

```dart
// In _handleCommandsToIsolate, swap the computation:
final result = await heavyComputation(payload);
sendPort.send((id, result));
```

The `Worker` class shell stays unchanged — only the computation logic inside `_handleCommandsToIsolate` needs to change.
