# Angular Dojo: Cross-Tab Kommunikation mit BroadcastChannel API
**Datum:** 2026-11-23
**Dauer:** ~25 Minuten
**Level:** Fortgeschritten

## Lernziel
Du lernst, wie du mit der nativen `BroadcastChannel` Web-API einen Angular-Service baust, der Zustandsänderungen in Echtzeit zwischen mehreren Browser-Tabs synchronisiert – integriert mit Signals und RxJS.

## Hintergrund & Theorie
Die `BroadcastChannel` API ist eine native Web-API, die es ermöglicht, Nachrichten zwischen verschiedenen Browser-Tabs, Fenstern oder iFrames desselben Origins auszutauschen – ohne Server, ohne WebSockets, ohne Polling.

```typescript
const channel = new BroadcastChannel('my-channel');
channel.postMessage({ type: 'UPDATE', payload: data });
channel.onmessage = (event) => console.log(event.data);
```

Typische Anwendungsfälle:
- **Login/Logout-Synchronisation**: Nutzer meldet sich in Tab A ab, Tab B leitet automatisch zum Login weiter.
- **Warenkorb/Favoriten**: Änderungen in Tab A erscheinen sofort in Tab B.
- **Notification Deduplication**: Nur ein Tab zeigt eine Push-Benachrichtigung an.
- **Theme/Settings-Sync**: Einstellungen sofort über alle Tabs hinweg aktualisieren.

Wichtige Einschränkungen:
- Funktioniert nur innerhalb desselben Origins (Protokoll + Domain + Port).
- Tabs im selben BroadcastChannel-Namen empfangen Nachrichten voneinander – nicht von sich selbst.
- Der Channel muss explizit via `.close()` geschlossen werden, um Memory Leaks zu vermeiden.

In Angular ist das Ziel, die Lebenszyklus-Verwaltung über `DestroyRef` oder `takeUntilDestroyed` zu kapseln und die eingehenden Nachrichten als Signal oder Observable bereitzustellen.

## Aufgabe
Erstelle einen generischen `BroadcastChannelService<T>`, der:
1. Einen benannten Channel öffnet und beim Zerstören automatisch schließt.
2. Nachrichten als `Signal<T | null>` bereitstellt.
3. Nachrichten als `Observable<T>` per RxJS bereitstellt.
4. Eine `send(message: T)`-Methode anbietet.

Nutze den Service in einer Komponente, um den **User-Login-Status** (simuliert) über mehrere Tabs zu synchronisieren.

### Schritte
1. **Service erstellen**: `BroadcastChannelService<T>` als generischer Service mit `inject(DestroyRef)` für automatisches `close()`.
2. **Observable erstellen**: Wandle `channel.onmessage` via `fromEventPattern` oder `new Observable()` in ein RxJS `Observable<T>` um.
3. **Signal ableiten**: Nutze `toSignal()` aus `@angular/core/rxjs-interop`, um aus dem Observable ein Signal zu machen.
4. **Komponente bauen**: Eine `AuthSyncComponent`, die den Login-Status anzeigt und Änderungen broadcastet.
5. **Testen**: Öffne die App in zwei Browser-Tabs und klicke in Tab A auf "Login/Logout" – beobachte, wie sich Tab B automatisch aktualisiert.

## Hints
<details>
<summary>Hint 1 – BroadcastChannel als Observable wrappen</summary>

`fromEvent` funktioniert nicht direkt mit `BroadcastChannel`. Nutze stattdessen `fromEventPattern` oder erstelle ein Observable manuell:

```typescript
import { fromEventPattern } from 'rxjs';

const messages$ = fromEventPattern<MessageEvent<T>>(
  (handler) => (channel.onmessage = handler),
  () => (channel.onmessage = null)
);
```

Oder mit dem `new Observable()`-Konstruktor:

```typescript
const messages$ = new Observable<T>((subscriber) => {
  channel.onmessage = (event) => subscriber.next(event.data);
  return () => { channel.onmessage = null; };
});
```

</details>

<details>
<summary>Hint 2 – DestroyRef für automatisches close()</summary>

Nutze `DestroyRef` statt `ngOnDestroy`, um den Channel lifecycle-sicher zu schließen:

```typescript
export class BroadcastChannelService<T> {
  private channel: BroadcastChannel;

  constructor() {
    this.channel = new BroadcastChannel('app-state');
    inject(DestroyRef).onDestroy(() => this.channel.close());
  }
}
```

Oder erstelle eine Factory-Funktion für den Service und übergebe den Channel-Namen:

```typescript
export function provideBroadcastChannel<T>(name: string) {
  return new BroadcastChannelService<T>(name);
}
```

</details>

<details>
<summary>Hint 3 – toSignal mit initialValue</summary>

`toSignal()` benötigt einen Initialwert, wenn keine `requireSync`-Option gesetzt ist:

```typescript
readonly lastMessage = toSignal(this.messages$, { initialValue: null });
```

Da `BroadcastChannel` keine synchronen Werte emittiert, ist `initialValue: null` der korrekte Ansatz.

</details>

## Beispiellösung

```typescript
// broadcast-channel.service.ts
import { Injectable, inject, DestroyRef } from '@angular/core';
import { Observable, Subject } from 'rxjs';
import { toSignal } from '@angular/core/rxjs-interop';

@Injectable()
export class BroadcastChannelService<T> {
  private channel: BroadcastChannel;
  private messageSubject = new Subject<T>();

  readonly messages$: Observable<T> = this.messageSubject.asObservable();
  readonly lastMessage = toSignal(this.messages$, { initialValue: null });

  constructor(channelName: string) {
    this.channel = new BroadcastChannel(channelName);
    this.channel.onmessage = (event: MessageEvent<T>) => {
      this.messageSubject.next(event.data);
    };
    inject(DestroyRef).onDestroy(() => {
      this.messageSubject.complete();
      this.channel.close();
    });
  }

  send(message: T): void {
    this.channel.postMessage(message);
  }
}

// auth-sync.component.ts
import { Component, signal, computed, inject, OnInit } from '@angular/core';
import { BroadcastChannelService } from './broadcast-channel.service';
import { takeUntilDestroyed } from '@angular/core/rxjs-interop';

interface AuthMessage {
  type: 'LOGIN' | 'LOGOUT';
  userId?: string;
  timestamp: number;
}

@Component({
  selector: 'app-auth-sync',
  standalone: true,
  providers: [
    {
      provide: BroadcastChannelService,
      useFactory: () => new BroadcastChannelService<AuthMessage>('auth-channel'),
    },
  ],
  template: `
    <div class="auth-sync">
      <h2>Auth Status (Tab-Sync)</h2>

      <div [class.logged-in]="isLoggedIn()">
        Status: {{ isLoggedIn() ? '✅ Eingeloggt als ' + userId() : '❌ Ausgeloggt' }}
      </div>

      <div class="last-event">
        Letztes Ereignis: {{ lastEventInfo() }}
      </div>

      <button (click)="login()">Login simulieren</button>
      <button (click)="logout()">Logout simulieren</button>
    </div>
  `,
})
export class AuthSyncComponent implements OnInit {
  private authChannel = inject(BroadcastChannelService<AuthMessage>);

  readonly isLoggedIn = signal(false);
  readonly userId = signal<string | null>(null);
  readonly lastEventInfo = signal('Noch keine Ereignisse');

  ngOnInit(): void {
    this.authChannel.messages$
      .pipe(takeUntilDestroyed())
      .subscribe((msg) => this.applyAuthMessage(msg));
  }

  login(): void {
    const userId = `user-${Math.random().toString(36).slice(2, 7)}`;
    const msg: AuthMessage = { type: 'LOGIN', userId, timestamp: Date.now() };
    this.authChannel.send(msg);
    this.applyAuthMessage(msg); // auch lokal anwenden
  }

  logout(): void {
    const msg: AuthMessage = { type: 'LOGOUT', timestamp: Date.now() };
    this.authChannel.send(msg);
    this.applyAuthMessage(msg);
  }

  private applyAuthMessage(msg: AuthMessage): void {
    if (msg.type === 'LOGIN') {
      this.isLoggedIn.set(true);
      this.userId.set(msg.userId ?? null);
      this.lastEventInfo.set(
        `LOGIN (${msg.userId}) um ${new Date(msg.timestamp).toLocaleTimeString()}`
      );
    } else {
      this.isLoggedIn.set(false);
      this.userId.set(null);
      this.lastEventInfo.set(
        `LOGOUT um ${new Date(msg.timestamp).toLocaleTimeString()}`
      );
    }
  }
}
```

## Weiterführendes
- **SharedWorker als Alternative**: Ein `SharedWorker` ermöglicht noch mächtigere Cross-Tab-Kommunikation mit persistentem State – ideal für komplexere Szenarien.
- **Storage Event**: Für einfache Key-Value-Synchronisation kann `window.addEventListener('storage', ...)` eine leichtgewichtigere Alternative zu BroadcastChannel sein.
- **NgRx + BroadcastChannel**: In größeren Apps bietet es sich an, `BroadcastChannel`-Nachrichten direkt als NgRx Actions zu dispatchen, um den globalen Store über Tabs hinweg zu synchronisieren.
- Offizielle MDN-Doku: [BroadcastChannel API](https://developer.mozilla.org/en-US/docs/Web/API/BroadcastChannel)
