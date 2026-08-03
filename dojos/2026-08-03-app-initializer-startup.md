# Angular Dojo: APP_INITIALIZER & Startup-Konfiguration

**Datum:** 2026-08-03
**Dauer:** ~25 Minuten
**Level:** Fortgeschritten

## Lernziel

Du lernst, wie du mit `APP_INITIALIZER` und `ENVIRONMENT_INITIALIZER` Code garantiert ausführst, bevor Angular die erste Komponente rendert – ideal für das Laden von Remote-Konfiguration, Feature Flags oder Nutzer-Session-Daten beim Bootstrap.

## Hintergrund & Theorie

In Enterprise-Apps gibt es oft Aufgaben, die *vor* dem ersten Rendering abgeschlossen sein müssen: Konfiguration von einem Config-Server laden, Auth-Token aus dem Cookie erneuern, Feature-Flag-Dienst initialisieren. Angular stellt dafür spezielle Injection Tokens bereit:

**`APP_INITIALIZER`**  
Ein Multi-Provider-Token, das eine oder mehrere Factory-Funktionen akzeptiert. Jede Factory muss eine `Promise` oder ein `Observable` zurückgeben. Angular wartet, bis *alle* dieser Promises/Observables abgeschlossen sind, bevor die App gebootstrapped wird. Wirft eine Factory einen Fehler, stoppt der Bootstrap-Prozess komplett.

```typescript
providers: [
  {
    provide: APP_INITIALIZER,
    useFactory: (cfg: ConfigService) => () => cfg.load(),
    deps: [ConfigService],
    multi: true,
  }
]
```

**`ENVIRONMENT_INITIALIZER`**  
Ähnlich wie `APP_INITIALIZER`, aber scoped auf ein Environment (z. B. beim Lazy Loading von Routen). Liefert eine synchrone Funktion und ist vor allem für Side-Effects wie Logging-Setup oder Icon-Registrierung gedacht.

**Wichtige Fallstricke:**
- Die Factory gibt eine *Funktion* zurück, die dann die Promise/Observable zurückgibt – doppelte Pfeilfunktion ist kein Tippfehler.
- Bei einem Observable muss es `complete()` aufrufen, sonst hängt der Bootstrap.
- Lange Initialisierung blockiert den Browser: Zeige einen Splash Screen oder nutze `takeUntilDestroyed()` mit Timeout-Fallback.

## Aufgabe

Erstelle eine Angular-App, die beim Start eine **Remote-Konfiguration** lädt (simuliert mit einem kurzen `delay`). Die geladene Konfiguration soll App-weit via Service verfügbar sein, ohne dass Komponenten wissen müssen, wann sie geladen wurde.

### Schritte

1. Erstelle einen `AppConfigService` mit einer `load()`-Methode, die eine `Promise<void>` zurückgibt. Simuliere darin einen HTTP-Request mit einem 800ms-Delay (`delay`-Operator oder `setTimeout`). Speichere die geladene Konfiguration (z. B. `apiUrl`, `featureFlags`) als Property.

2. Registriere `APP_INITIALIZER` in `app.config.ts` (oder `bootstrapApplication`), der `AppConfigService.load()` aufruft. Stelle sicher, dass der Service als `providedIn: 'root'` oder explizit im Provider-Array registriert ist.

3. Erstelle einen `AppComponent`, der die Konfigurationswerte aus dem Service anzeigt. Vergewissere dich, dass sie bereits vorhanden sind – kein `undefined`, kein Loading-Spinner nötig.

4. **Bonus:** Füge einen zweiten `APP_INITIALIZER` hinzu, der Feature Flags aus dem bereits geladenen Config-Objekt auswertet und einen `FeatureFlagService` initialisiert. Beide Initializer sollen **parallel** laufen (standard, da Angular alle APP_INITIALIZER gleichzeitig startet).

5. **Bonus 2:** Zeige während des Bootstraps einen Splash Screen an, indem du in `index.html` einen `<div id="splash">Loading…</div>` platzierst und ihn am Ende von `AppConfigService.load()` entfernst (`document.getElementById('splash')?.remove()`).

## Hints

<details>
<summary>Hint 1 – Doppelte Pfeilfunktion verstehen</summary>

`APP_INITIALIZER` erwartet eine **Factory**, die eine **Factory** zurückgibt:

```typescript
// Äußere Funktion: wird beim DI-Setup aufgerufen, erhält Deps als Parameter
// Innere Funktion: wird von Angular beim Bootstrap aufgerufen, gibt Promise/Observable zurück
{
  provide: APP_INITIALIZER,
  useFactory: (service: AppConfigService) => () => service.load(),
  //           ^^^^^ DI-Factory                ^^^^^ Init-Funktion
  deps: [AppConfigService],
  multi: true,
}
```

Ohne `multi: true` würde jeder weitere `APP_INITIALIZER`-Provider den vorherigen überschreiben.
</details>

<details>
<summary>Hint 2 – Observable statt Promise</summary>

Möchtest du RxJS nutzen, muss das Observable `complete()` aufrufen. `firstValueFrom` ist der einfachste Weg:

```typescript
import { firstValueFrom, timer } from 'rxjs';
import { tap } from 'rxjs/operators';

load(): Promise<void> {
  return firstValueFrom(
    timer(800).pipe(
      tap(() => {
        this.config = { apiUrl: 'https://api.example.com', featureFlags: { darkMode: true } };
      }),
      map(() => void 0),
    )
  );
}
```

Alternativ gibt `lastValueFrom` den letzten emittierten Wert zurück – bei `timer` identisch mit `firstValueFrom`.
</details>

<details>
<summary>Hint 3 – ENVIRONMENT_INITIALIZER für Lazy Routes</summary>

In einer Lazy-Route-Konfiguration kannst du `ENVIRONMENT_INITIALIZER` nutzen, um Dienste beim Laden des Moduls zu initialisieren:

```typescript
// routes.ts
export const routes: Routes = [
  {
    path: 'admin',
    loadChildren: () => import('./admin/routes').then(m => m.ADMIN_ROUTES),
    providers: [
      {
        provide: ENVIRONMENT_INITIALIZER,
        useValue: () => inject(AdminAnalyticsService).init(),
        multi: true,
      }
    ]
  }
];
```

`ENVIRONMENT_INITIALIZER` bekommt eine **synchrone** Funktion (kein Promise/Observable) – nur für Side-Effects.
</details>

## Beispiellösung

```typescript
// app-config.service.ts
import { Injectable, inject } from '@angular/core';
import { DOCUMENT } from '@angular/common';

export interface AppConfig {
  apiUrl: string;
  featureFlags: Record<string, boolean>;
}

@Injectable({ providedIn: 'root' })
export class AppConfigService {
  private document = inject(DOCUMENT);

  config!: AppConfig;

  load(): Promise<void> {
    return new Promise(resolve => {
      setTimeout(() => {
        // Simulierter HTTP-Response
        this.config = {
          apiUrl: 'https://api.myapp.com/v2',
          featureFlags: {
            darkMode: true,
            betaDashboard: false,
            newCheckout: true,
          },
        };
        this.document.getElementById('splash')?.remove();
        resolve();
      }, 800);
    });
  }
}
```

```typescript
// feature-flag.service.ts
import { Injectable, inject } from '@angular/core';
import { AppConfigService } from './app-config.service';

@Injectable({ providedIn: 'root' })
export class FeatureFlagService {
  private configService = inject(AppConfigService);
  private flags: Record<string, boolean> = {};

  init(): void {
    // config ist hier garantiert geladen, da wir nach APP_INITIALIZER laufen
    this.flags = this.configService.config.featureFlags;
    console.log('[FeatureFlags] initialisiert:', this.flags);
  }

  isEnabled(flag: string): boolean {
    return this.flags[flag] ?? false;
  }
}
```

```typescript
// app.config.ts
import { ApplicationConfig, APP_INITIALIZER } from '@angular/core';
import { provideRouter } from '@angular/router';
import { AppConfigService } from './app-config.service';
import { FeatureFlagService } from './feature-flag.service';
import { routes } from './app.routes';

export const appConfig: ApplicationConfig = {
  providers: [
    provideRouter(routes),
    {
      provide: APP_INITIALIZER,
      useFactory: (cfg: AppConfigService) => () => cfg.load(),
      deps: [AppConfigService],
      multi: true,
    },
    {
      provide: APP_INITIALIZER,
      useFactory: (cfg: AppConfigService, flags: FeatureFlagService) =>
        async () => {
          // Wartet implizit, bis der erste Initializer fertig ist,
          // weil Angular APP_INITIALIZER alle parallel startet –
          // aber cfg.load() wird durch den ersten Provider bereits gestartet.
          // Für eine garantierte Sequenz: cfg.load() hier direkt awaiten.
          await cfg.load(); // Idempotenz sicherstellen oder separaten Status nutzen
          flags.init();
        },
      deps: [AppConfigService, FeatureFlagService],
      multi: true,
    },
  ],
};
```

```typescript
// app.component.ts
import { Component, inject } from '@angular/core';
import { AppConfigService } from './app-config.service';
import { FeatureFlagService } from './feature-flag.service';

@Component({
  selector: 'app-root',
  standalone: true,
  template: `
    <h1>App gestartet!</h1>
    <p>API URL: {{ config.config.apiUrl }}</p>
    <p>Dark Mode: {{ flags.isEnabled('darkMode') ? 'aktiviert' : 'deaktiviert' }}</p>
    <p>Beta Dashboard: {{ flags.isEnabled('betaDashboard') ? 'aktiviert' : 'deaktiviert' }}</p>
  `,
})
export class AppComponent {
  config = inject(AppConfigService);
  flags = inject(FeatureFlagService);
}
```

```html
<!-- index.html – Splash Screen während des Bootstraps -->
<body>
  <div id="splash" style="
    position: fixed; inset: 0; display: grid;
    place-items: center; background: #0f172a; color: #e2e8f0;
    font-family: sans-serif; font-size: 1.5rem; z-index: 9999;
  ">
    Lade Konfiguration…
  </div>
  <app-root></app-root>
</body>
```

## Weiterführendes

- **Parallel vs. sequenziell:** Alle `APP_INITIALIZER`-Provider starten gleichzeitig. Wenn Initializer B von den Ergebnissen von Initializer A abhängt, musst du entweder einen einzigen Initializer mit interner Sequenz bauen oder den zweiten Service so gestalten, dass er von `AppConfigService` (bereits geladen) injiziert und synchron initialisiert wird.
- **Fehlerbehandlung:** Wirft ein Initializer, bricht der Bootstrap komplett ab. Fange Fehler intern ab und zeige dem Nutzer eine aussagekräftige Fehlermeldung an (`ErrorHandler` aus `@angular/core`).
- **`ApplicationRef.isStable`:** Für Tests oder SSR kann es nützlich sein, auf `ApplicationRef.isStable.pipe(filter(Boolean), take(1))` zu warten, um zu wissen, wann die App vollständig initialisiert und stabil ist.
- **Offizielle Doku:** [angular.dev/api/core/APP_INITIALIZER](https://angular.dev/api/core/APP_INITIALIZER)
