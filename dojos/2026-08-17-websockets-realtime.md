# Angular Dojo: WebSockets & Real-Time Communication
**Datum:** 2026-08-17
**Dauer:** ~25 Minuten
**Level:** Fortgeschritten

## Lernziel
Du lernst, wie du WebSockets in Angular mit RxJS's `webSocket()`-Funktion integrierst und die eingehenden Nachrichten reaktiv mit Signals im UI darstellst – inklusive automatischer Wiederverbindungslogik.

## Hintergrund & Theorie

RxJS bietet mit `webSocket()` (aus `rxjs/webSocket`) eine elegante Abstraktion über die native `WebSocket`-API. Die Funktion gibt ein `WebSocketSubject` zurück, das sowohl Observable (für eingehende Nachrichten) als auch Observer (zum Senden) ist.

**Wichtige Konzepte:**

- `webSocket(url)` – erstellt ein `WebSocketSubject`, das die Verbindung lazy öffnet (erst beim ersten Subscriber)
- `.pipe(retry({ delay: 2000 }))` – automatische Wiederverbindung bei Verbindungsabbruch
- `subject.next(message)` – sendet eine Nachricht an den Server
- `subject.complete()` – schließt die Verbindung sauber
- Teardown über `DestroyRef`/`takeUntilDestroyed()` verhindert Memory Leaks

Seit Angular 17+ lässt sich das Observable einfach mit `toSignal()` in ein Signal umwandeln, was den Boilerplate für `async`-Pipe oder manuelle Subscriptions eliminiert. Für komplexere Szenarien (Reconnect-Status, Fehlerbehandlung) ist ein dedizierter Service mit eigenem Zustand sinnvoll.

Eine wichtige Besonderheit: `WebSocketSubject` teilt sich **eine** Verbindung unter allen Subscribern (kein `share()` nötig), schließt sie aber, sobald der letzte Subscriber sich abmeldet.

## Aufgabe

Baue einen einfachen **Live-Chat** als Standalone-Component. Nutze einen Mock-WebSocket-Service (du kannst `wss://echo.websocket.org` oder einen lokalen Mock verwenden), der gesendete Nachrichten zurückspiegelt.

### Schritte

1. Erstelle einen `ChatService` als `@Injectable({ providedIn: 'root' })` mit:
   - einer `WebSocketSubject` für die Verbindung
   - einer öffentlichen Methode `messages$` (Observable mit `retry`-Logik)
   - einer Methode `send(message: ChatMessage): void`
   - einer Methode `disconnect(): void`

2. Erstelle ein Interface `ChatMessage` mit `{ user: string; text: string; timestamp: number }`.

3. Erstelle eine `ChatComponent` (Standalone) die:
   - `ChatService` per `inject()` bezieht
   - `messages$` mit `toSignal()` in ein Signal umwandelt (Initialwert: `[]`)
   - eingehende Nachrichten in einem `messages`-Array-Signal akkumuliert (via `effect()` oder `scan()`)
   - Ein Eingabefeld und einen Send-Button hat
   - Die Verbindung bei Destroy sauber trennt (via `DestroyRef`)

4. Stelle sicher, dass die Component den `ChatService` korrekt per DI nutzt und keine direkten `WebSocket`-Instanzen erstellt.

## Hints

<details>
<summary>Hint 1 – WebSocketSubject aufbauen</summary>

```typescript
import { webSocket, WebSocketSubject } from 'rxjs/webSocket';

// Im Service:
private socket$: WebSocketSubject<ChatMessage> | null = null;

private getSocket(): WebSocketSubject<ChatMessage> {
  if (!this.socket$ || this.socket$.closed) {
    this.socket$ = webSocket<ChatMessage>('wss://echo.websocket.org');
  }
  return this.socket$;
}

readonly messages$ = this.getSocket().pipe(
  retry({ count: 5, delay: 2000 }),
  share()
);
```
</details>

<details>
<summary>Hint 2 – Nachrichten akkumulieren mit scan()</summary>

Statt `toSignal()` direkt auf ein Array zu mappen, kombiniere mit `scan()`:

```typescript
readonly messages = toSignal(
  this.chatService.messages$.pipe(
    scan((acc, msg) => [...acc, msg], [] as ChatMessage[])
  ),
  { initialValue: [] as ChatMessage[] }
);
```

So wächst das Array reaktiv mit jeder eingehenden Nachricht – ohne manuelles `effect()`.
</details>

<details>
<summary>Hint 3 – Sauber Teardown mit DestroyRef</summary>

```typescript
export class ChatComponent {
  private destroyRef = inject(DestroyRef);
  private chatService = inject(ChatService);

  constructor() {
    this.destroyRef.onDestroy(() => this.chatService.disconnect());
  }
}
```

Alternativ kannst du `takeUntilDestroyed(this.destroyRef)` direkt in der Pipe des Observables nutzen, falls du eine manuelle Subscription verwendest.
</details>

## Beispiellösung

```typescript
// chat.model.ts
export interface ChatMessage {
  user: string;
  text: string;
  timestamp: number;
}

// chat.service.ts
import { Injectable, OnDestroy } from '@angular/core';
import { webSocket, WebSocketSubject } from 'rxjs/webSocket';
import { Observable, retry, share } from 'rxjs';
import { ChatMessage } from './chat.model';

@Injectable({ providedIn: 'root' })
export class ChatService implements OnDestroy {
  private socket$: WebSocketSubject<ChatMessage> | null = null;

  private getSocket(): WebSocketSubject<ChatMessage> {
    if (!this.socket$ || this.socket$.closed) {
      this.socket$ = webSocket<ChatMessage>('wss://echo.websocket.org');
    }
    return this.socket$;
  }

  readonly messages$: Observable<ChatMessage> = new Observable(observer => {
    const sub = this.getSocket()
      .pipe(retry({ count: 5, delay: 2000 }), share())
      .subscribe(observer);
    return () => sub.unsubscribe();
  });

  send(message: ChatMessage): void {
    this.getSocket().next(message);
  }

  disconnect(): void {
    this.socket$?.complete();
    this.socket$ = null;
  }

  ngOnDestroy(): void {
    this.disconnect();
  }
}

// chat.component.ts
import { Component, inject, signal, DestroyRef } from '@angular/core';
import { CommonModule } from '@angular/common';
import { FormsModule } from '@angular/forms';
import { toSignal } from '@angular/core/rxjs-interop';
import { scan } from 'rxjs/operators';
import { ChatService } from './chat.service';
import { ChatMessage } from './chat.model';

@Component({
  selector: 'app-chat',
  standalone: true,
  imports: [CommonModule, FormsModule],
  template: `
    <div class="chat-container">
      <ul class="message-list">
        @for (msg of messages(); track msg.timestamp) {
          <li>
            <strong>{{ msg.user }}</strong>: {{ msg.text }}
            <small>{{ msg.timestamp | date:'HH:mm:ss' }}</small>
          </li>
        }
        @if (messages().length === 0) {
          <li><em>Noch keine Nachrichten...</em></li>
        }
      </ul>

      <div class="input-row">
        <input
          [(ngModel)]="inputText"
          placeholder="Nachricht eingeben..."
          (keyup.enter)="sendMessage()"
        />
        <button (click)="sendMessage()" [disabled]="!inputText()">Senden</button>
      </div>
    </div>
  `,
})
export class ChatComponent {
  private chatService = inject(ChatService);
  private destroyRef = inject(DestroyRef);

  readonly inputText = signal('');

  readonly messages = toSignal(
    this.chatService.messages$.pipe(
      scan((acc, msg) => [...acc, msg], [] as ChatMessage[])
    ),
    { initialValue: [] as ChatMessage[] }
  );

  constructor() {
    this.destroyRef.onDestroy(() => this.chatService.disconnect());
  }

  sendMessage(): void {
    const text = this.inputText().trim();
    if (!text) return;

    this.chatService.send({
      user: 'Me',
      text,
      timestamp: Date.now(),
    });
    this.inputText.set('');
  }
}
```

## Weiterführendes

- **Reconnect-Status als Signal**: Exponiere einen `connectionStatus`-Signal (`'connected' | 'disconnected' | 'reconnecting'`) über `webSocket({ url, openObserver, closeObserver })` – die `openObserver` und `closeObserver`-Optionen feuern bei Verbindungsauf- und -abbau.
- **Authentifizierung**: Sende nach dem Verbindungsaufbau sofort ein Auth-Token mit `openObserver` + `subject.next({ type: 'auth', token })`.
- **Multiplexing**: `WebSocketSubject.multiplex()` erlaubt es, mehrere logische Kanäle über eine physische WebSocket-Verbindung zu routen – ideal für Chat-Räume oder Topic-Subscriptions.
- Offizielle RxJS-Doku: [rxjs.dev/api/webSocket](https://rxjs.dev/api/webSocket/webSocket)
