# Angular Dojo: Server-Sent Events (SSE) in Angular
**Datum:** 2026-09-06
**Dauer:** ~25 Minuten
**Level:** Fortgeschritten

## Lernziel
Du lernst, wie du Server-Sent Events (SSE) mit der nativen `EventSource`-API in Angular kapselst, einen typisierten `SseService` als `Observable` baust und den Stream sauber in die Signal-Architektur integrierst.

## Hintergrund & Theorie
Server-Sent Events (SSE) ermöglichen einen unidirektionalen Echtzeit-Datenstrom vom Server zum Client über eine persistente HTTP-Verbindung. Im Unterschied zu WebSockets sind SSE-Verbindungen HTTP-basiert, nur vom Server zum Client gerichtet und werden vom Browser bei Verbindungsabbruch automatisch wiederhergestellt.

Die native `EventSource`-API ist einfach zu nutzen, passt aber nicht direkt in Angulars reaktives Modell. Die Lösung: `EventSource` in ein `Observable` wrappen. So erhält man automatisches Cleanup beim Unsubscribe, Fehlerweiterleitung und volle RxJS-Kompatibilität – inklusive `pipe(takeUntilDestroyed())`.

Typische Anwendungsfälle sind KI-Chatbot-Streaming-Antworten, Live-Logs, Echtzeit-Benachrichtigungen und Live-Dashboards. Mit `toSignal()` lässt sich der Stream direkt in die Signal-Welt übersetzen.

Ein wichtiger Unterschied zu WebSockets: SSE unterstützt nur Text-Daten (üblicherweise JSON-serialisiert), keine Binary-Daten, und ausschließlich Server-zu-Client-Kommunikation.

## Aufgabe
Erstelle einen `SseService`, der SSE-Verbindungen als typisierte `Observable<T>` kapselt, und einen `LiveLogComponent`, der einen Log-Stream in Echtzeit anzeigt.

### Schritte

1. Erstelle `sse.service.ts` mit einer generischen `stream<T>(url, options)` Methode. Die Methode soll eine `EventSource`-Verbindung öffnen, Events parsen und als `Observable<T>` zurückgeben. Stelle sicher, dass beim Unsubscribe die Verbindung via `eventSource.close()` getrennt wird.

2. Definiere ein `LogEntry`-Interface (`timestamp`, `level: 'INFO' | 'WARN' | 'ERROR'`, `message`) und erstelle `LiveLogComponent` als Standalone Component. Nutze ein `signal<LogEntry[]>([])` als State und abonniere den Stream mit `takeUntilDestroyed()`.

3. Füge einen `computed()`-Wert für die Fehleranzahl hinzu (`errorCount`) und zeige ihn im Template an. Begrenze die angezeigten Log-Einträge auf maximal 200 (älteste werden verworfen).

4. Implementiere einen "Pause/Resume"-Button: Beim Pausieren wird die aktuelle Subscription beendet (`eventSource.close()` via Unsubscribe), beim Fortsetzen wird eine neue Subscription geöffnet. Nutze dafür ein `isStreaming`-Signal und bedingte Subscription-Logik.

## Hints

<details>
<summary>Hint 1: EventSource als Observable kapseln</summary>

```typescript
stream<T>(url: string, eventType = 'message'): Observable<T> {
  return new Observable<T>(observer => {
    const eventSource = new EventSource(url);

    const handler = (event: MessageEvent) => {
      try {
        observer.next(JSON.parse(event.data) as T);
      } catch {
        observer.error(new Error(`Parse error: ${event.data}`));
      }
    };

    eventSource.addEventListener(eventType, handler);

    eventSource.onerror = () => {
      if (eventSource.readyState === EventSource.CLOSED) {
        observer.complete();
      } else {
        observer.error(new Error('SSE connection error'));
        eventSource.close();
      }
    };

    // Cleanup bei Unsubscribe
    return () => {
      eventSource.removeEventListener(eventType, handler);
      eventSource.close();
    };
  });
}
```
</details>

<details>
<summary>Hint 2: Pause/Resume mit Subscription-Management</summary>

```typescript
export class LiveLogComponent {
  private subscription: Subscription | null = null;
  isStreaming = signal(false);

  start(): void {
    if (this.subscription) return;
    this.isStreaming.set(true);
    this.subscription = this.sseService
      .stream<LogEntry>('/api/logs/stream', { eventType: 'log' })
      .subscribe(entry => this.logs.update(l => [...l, entry].slice(-200)));
  }

  pause(): void {
    this.subscription?.unsubscribe(); // löst das cleanup() im Observable aus
    this.subscription = null;
    this.isStreaming.set(false);
  }

  toggleStream(): void {
    this.isStreaming() ? this.pause() : this.start();
  }
}
```
</details>

## Beispiellösung

```typescript
// sse.service.ts
import { Injectable } from '@angular/core';
import { Observable } from 'rxjs';

export interface SseOptions {
  withCredentials?: boolean;
  eventType?: string;
}

@Injectable({ providedIn: 'root' })
export class SseService {
  stream<T>(url: string, options: SseOptions = {}): Observable<T> {
    const { withCredentials = false, eventType = 'message' } = options;

    return new Observable<T>(observer => {
      const eventSource = new EventSource(url, { withCredentials });

      const handler = (event: MessageEvent) => {
        try {
          observer.next(JSON.parse(event.data) as T);
        } catch {
          observer.error(new Error(`SSE parse error: ${event.data}`));
        }
      };

      eventSource.addEventListener(eventType, handler);

      eventSource.onerror = () => {
        if (eventSource.readyState === EventSource.CLOSED) {
          observer.complete();
        } else {
          observer.error(new Error('SSE connection error'));
          eventSource.close();
        }
      };

      return () => {
        eventSource.removeEventListener(eventType, handler);
        eventSource.close();
      };
    });
  }
}

// live-log.component.ts
import {
  Component, inject, signal, computed,
  DestroyRef, OnInit, OnDestroy
} from '@angular/core';
import { DatePipe, NgClass } from '@angular/common';
import { Subscription } from 'rxjs';
import { SseService } from './sse.service';

interface LogEntry {
  timestamp: string;
  level: 'INFO' | 'WARN' | 'ERROR';
  message: string;
}

@Component({
  selector: 'app-live-log',
  standalone: true,
  imports: [DatePipe, NgClass],
  template: `
    <div class="log-container">
      <div class="controls">
        <button (click)="toggleStream()">
          {{ isStreaming() ? 'Pause' : 'Resume' }}
        </button>
        <button (click)="clearLogs()">Clear</button>
        <span>{{ logs().length }} Einträge | Fehler: {{ errorCount() }}</span>
      </div>

      <div class="log-list">
        @for (entry of logs(); track entry.timestamp) {
          <div class="log-entry" [ngClass]="entry.level.toLowerCase()">
            <span class="timestamp">{{ entry.timestamp | date:'HH:mm:ss.SSS' }}</span>
            <span class="level">[{{ entry.level }}]</span>
            <span class="message">{{ entry.message }}</span>
          </div>
        } @empty {
          <p class="empty">Kein Log-Einträge – Stream starten.</p>
        }
      </div>
    </div>
  `,
  styles: [`
    .log-list { height: 400px; overflow-y: auto; font-family: monospace; font-size: 13px; }
    .log-entry { display: flex; gap: 8px; padding: 2px 4px; }
    .info  { color: #333; }
    .warn  { color: #b45309; background: #fef3c7; }
    .error { color: #b91c1c; background: #fee2e2; }
    .timestamp { opacity: 0.6; flex-shrink: 0; }
    .level { font-weight: bold; width: 50px; }
  `],
})
export class LiveLogComponent implements OnInit, OnDestroy {
  private sseService = inject(SseService);
  private destroyRef = inject(DestroyRef);
  private subscription: Subscription | null = null;

  logs = signal<LogEntry[]>([]);
  isStreaming = signal(false);

  errorCount = computed(
    () => this.logs().filter(l => l.level === 'ERROR').length
  );

  ngOnInit(): void {
    this.start();
    this.destroyRef.onDestroy(() => this.pause());
  }

  ngOnDestroy(): void {
    this.pause();
  }

  private start(): void {
    if (this.subscription) return;
    this.isStreaming.set(true);
    this.subscription = this.sseService
      .stream<LogEntry>('/api/logs/stream', { eventType: 'log' })
      .subscribe({
        next: entry => this.logs.update(l => [...l, entry].slice(-200)),
        error: err => console.error('SSE-Fehler:', err),
        complete: () => this.isStreaming.set(false),
      });
  }

  private pause(): void {
    this.subscription?.unsubscribe();
    this.subscription = null;
    this.isStreaming.set(false);
  }

  toggleStream(): void {
    this.isStreaming() ? this.pause() : this.start();
  }

  clearLogs(): void {
    this.logs.set([]);
  }
}
```

## Weiterführendes
- **`fetch()` mit `ReadableStream`**: Für SSE über POST-Requests (z.B. beim Senden eines Prompts an eine KI-API) kann `fetch()` mit `response.body.getReader()` genutzt werden – `EventSource` unterstützt nur GET ohne Custom-Headers.
- **Automatisches Reconnect mit RxJS**: Füge `retry({ delay: (_, count) => timer(Math.min(Math.pow(2, count) * 1000, 30_000)) })` in die Pipe ein, um bei Verbindungsabbruch exponentielles Backoff zu implementieren.
- **`toSignal()` Integration**: Der gesamte Stream lässt sich mit `toSignal(this.sseService.stream(...), { initialValue: [] })` direkt als Signal nutzen – ideal für read-only Displays ohne Pause-Logik.
