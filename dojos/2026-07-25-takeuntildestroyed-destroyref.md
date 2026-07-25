# Angular Dojo: takeUntilDestroyed & DestroyRef
**Datum:** 2026-07-25
**Dauer:** ~25 Minuten
**Level:** Fortgeschritten

## Lernziel
Du lernst, wie du mit `takeUntilDestroyed` und `DestroyRef` (Angular 16+) sauber RxJS-Subscriptions und andere Ressourcen beim Zerstören einer Komponente oder eines Services automatisch aufräumst – ohne manuelles `ngOnDestroy`.

## Hintergrund & Theorie

Lange Zeit war das Standardmuster in Angular, Subscriptions manuell über `ngOnDestroy` und einen `Subject` zu beenden:

```typescript
private destroy$ = new Subject<void>();

ngOnDestroy() {
  this.destroy$.next();
  this.destroy$.complete();
}
```

Seit **Angular 16** gibt es zwei elegantere Alternativen:

**`DestroyRef`**: Ein injizierbarer Service, der einen `onDestroy`-Callback registriert. Er ist an den Lifecycle des aktuellen Injection-Kontexts gebunden (Komponente, Direktive, Service, etc.).

**`takeUntilDestroyed`**: Ein RxJS-Operator, der intern `DestroyRef` nutzt und eine Observable automatisch beendet, sobald der Kontext zerstört wird. Er kann direkt im Injection-Kontext (z.B. im Konstruktor oder als Feldinitializer) verwendet werden – oder mit einem explizit injizierten `DestroyRef` ausserhalb des Injection-Kontexts.

Beide Ansätze reduzieren Boilerplate erheblich und funktionieren auch in **Standalone Components**, **Services**, **Direktiven** und sogar in Funktionen, die während der Initialisierung aufgerufen werden.

Wichtig: `takeUntilDestroyed()` ohne Argument muss im Injection-Kontext aufgerufen werden (Konstruktor, Feldinitializer). Ausserhalb des Kontexts – z.B. in `ngOnInit` – muss `DestroyRef` explizit injiziert und übergeben werden.

## Aufgabe

Baue eine `UserDashboardComponent`, die drei verschiedene Datenquellen per RxJS abonniert und alle Subscriptions sauber mit `takeUntilDestroyed` beendet. Refaktoriere danach einen bestehenden Service, der `DestroyRef` für das Aufräumen eines Timers nutzt.

### Schritte

1. **Komponente erstellen**: Erstelle `UserDashboardComponent` als Standalone Component. Injiziere drei verschiedene Services (`UserService`, `NotificationService`, `ActivityService`), die jeweils Observables bereitstellen (kannst du als Stubs implementieren).

2. **Subscriptions mit `takeUntilDestroyed`**: Abonniere alle drei Observables im Konstruktor (oder als Feldinitializer) mit `takeUntilDestroyed()` ohne expliziten Parameter. Gib empfangene Werte in Signals oder einfache Properties.

3. **`takeUntilDestroyed` in `ngOnInit`**: Verschiebe eine der Subscriptions in `ngOnInit`. Da dies ausserhalb des Injection-Kontexts ist, injiziere `DestroyRef` explizit und übergib ihn an `takeUntilDestroyed(this.destroyRef)`.

4. **`DestroyRef` für Timer-Cleanup**: Erstelle einen `PollingService` (providedIn: 'root' oder Component-scoped), der in seinem Konstruktor ein `setInterval` startet. Nutze `DestroyRef.onDestroy()`, um das Interval beim Zerstören zu stoppen.

5. **Testen**: Überprüfe in der Browser-Konsole (oder mit einem Unit-Test), dass beim Navigieren weg von der Komponente keine Subscriptions mehr feuern und der Timer gestoppt wird.

## Hints

<details>
<summary>Hint 1 – Grundstruktur mit takeUntilDestroyed im Konstruktor</summary>

```typescript
import { takeUntilDestroyed } from '@angular/core/rxjs-interop';

@Component({ /* ... */ })
export class UserDashboardComponent {
  // Im Konstruktor = Injection-Kontext, kein explizites DestroyRef nötig
  constructor(private userService: UserService) {
    this.userService.getUser$().pipe(
      takeUntilDestroyed()
    ).subscribe(user => this.user = user);
  }
}
```

</details>

<details>
<summary>Hint 2 – takeUntilDestroyed in ngOnInit (ausserhalb Injection-Kontext)</summary>

```typescript
import { DestroyRef, inject } from '@angular/core';
import { takeUntilDestroyed } from '@angular/core/rxjs-interop';

@Component({ /* ... */ })
export class UserDashboardComponent implements OnInit {
  private destroyRef = inject(DestroyRef);

  ngOnInit() {
    // Explizites DestroyRef nötig, da ngOnInit kein Injection-Kontext ist
    this.notificationService.getNotifications$().pipe(
      takeUntilDestroyed(this.destroyRef)
    ).subscribe(n => this.notifications = n);
  }
}
```

</details>

<details>
<summary>Hint 3 – DestroyRef für manuelle Cleanup-Logik</summary>

```typescript
import { DestroyRef, inject, Injectable } from '@angular/core';

@Injectable()
export class PollingService {
  private destroyRef = inject(DestroyRef);

  constructor() {
    const intervalId = setInterval(() => {
      console.log('Polling...');
    }, 3000);

    this.destroyRef.onDestroy(() => {
      clearInterval(intervalId);
      console.log('Polling stopped.');
    });
  }
}
```

</details>

## Beispiellösung

```typescript
// user-dashboard.component.ts
import { Component, OnInit, inject, signal } from '@angular/core';
import { DestroyRef } from '@angular/core';
import { takeUntilDestroyed } from '@angular/core/rxjs-interop';
import { interval, of } from 'rxjs';
import { map, switchMap } from 'rxjs/operators';

// Stub-Services
import { Injectable } from '@angular/core';

@Injectable({ providedIn: 'root' })
export class UserService {
  getUser$() {
    return interval(2000).pipe(map(i => ({ id: i, name: `User ${i}` })));
  }
}

@Injectable({ providedIn: 'root' })
export class NotificationService {
  getNotifications$() {
    return interval(5000).pipe(map(i => [`Notification ${i}`]));
  }
}

@Injectable({ providedIn: 'root' })
export class ActivityService {
  getActivity$() {
    return interval(3000).pipe(map(i => `Activity event ${i}`));
  }
}

@Injectable()
export class PollingService {
  private destroyRef = inject(DestroyRef);

  constructor() {
    const intervalId = setInterval(() => {
      console.log('[PollingService] Polling data...');
    }, 3000);

    this.destroyRef.onDestroy(() => {
      clearInterval(intervalId);
      console.log('[PollingService] Cleaned up interval.');
    });
  }
}

@Component({
  selector: 'app-user-dashboard',
  standalone: true,
  providers: [PollingService],
  template: `
    <h2>Dashboard</h2>
    <p>User: {{ user() | json }}</p>
    <p>Notifications: {{ notifications() | json }}</p>
    <p>Last Activity: {{ lastActivity() }}</p>
  `,
})
export class UserDashboardComponent implements OnInit {
  // Explizit für ngOnInit
  private destroyRef = inject(DestroyRef);

  // Services
  private userService = inject(UserService);
  private notificationService = inject(NotificationService);
  private activityService = inject(ActivityService);

  // PollingService: Lifecycle an Component gebunden
  private pollingService = inject(PollingService);

  // State als Signals
  user = signal<{ id: number; name: string } | null>(null);
  notifications = signal<string[]>([]);
  lastActivity = signal<string>('');

  constructor() {
    // Im Konstruktor: kein explizites DestroyRef nötig
    this.userService.getUser$().pipe(
      takeUntilDestroyed()
    ).subscribe(u => {
      this.user.set(u);
      console.log('[UserDashboardComponent] User updated:', u);
    });

    this.activityService.getActivity$().pipe(
      takeUntilDestroyed()
    ).subscribe(a => this.lastActivity.set(a));
  }

  ngOnInit() {
    // Ausserhalb des Injection-Kontexts: DestroyRef explizit übergeben
    this.notificationService.getNotifications$().pipe(
      takeUntilDestroyed(this.destroyRef)
    ).subscribe(n => this.notifications.set(n));
  }
}
```

```typescript
// app.component.ts – Toggle-Test
@Component({
  selector: 'app-root',
  standalone: true,
  imports: [UserDashboardComponent, NgIf],
  template: `
    <button (click)="show.set(!show())">Toggle Dashboard</button>
    @if (show()) {
      <app-user-dashboard />
    }
  `,
})
export class AppComponent {
  show = signal(true);
}
```

## Weiterführendes

- **Feldinitializer-Pattern**: `data$ = this.service.get$().pipe(takeUntilDestroyed())` direkt als Klassenfeld – noch weniger Boilerplate als im Konstruktor.
- **`toSignal()` mit `DestroyRef`**: `toSignal(this.data$, { injector: this.injector })` ermöglicht das Erstellen von Signals aus Observables auch ausserhalb des Injection-Kontexts.
- **Offizielle Doku**: [angular.dev/guide/components/lifecycle](https://angular.dev/guide/components/lifecycle) und [angular.dev/api/core/rxjs-interop/takeUntilDestroyed](https://angular.dev/api/core/rxjs-interop/takeUntilDestroyed)
- **Testing**: Mit `TestBed.inject(DestroyRef)` und `.onDestroy()` lassen sich Cleanup-Callbacks in Unit-Tests überprüfen.
