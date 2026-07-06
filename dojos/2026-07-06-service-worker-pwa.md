# Angular Dojo: Service Worker & Progressive Web App (PWA)
**Datum:** 2026-07-06
**Dauer:** ~25 Minuten
**Level:** Fortgeschritten

## Lernziel
Du lernst, wie du eine Angular-App mit `@angular/service-worker` in eine vollwertige PWA verwandelst: Offline-Fähigkeit, Caching-Strategien und App-Update-Handling.

## Hintergrund & Theorie

Ein **Service Worker** ist ein JavaScript-Skript, das im Browser-Hintergrund läuft und als Netzwerk-Proxy zwischen App und Server fungiert. Angular's `@angular/service-worker` kapselt die komplexe Service-Worker-Logik und bietet:

- **Precaching**: Statische Assets beim Install cachen (App Shell)
- **Runtime Caching**: Dynamische API-Antworten per Strategie cachen
- **Caching-Strategien**:
  - `freshness` (Network-first): Netzwerk bevorzugen, Cache als Fallback
  - `performance` (Cache-first): Cache bevorzugen, Netzwerk nur bei Miss
- **App-Updates**: Der `SwUpdate`-Service ermöglicht kontrollierte Updates mit `checkForUpdate()` / `activateUpdate()`
- **Push Notifications**: Via `SwPush`-Service können Web-Push-Benachrichtigungen abonniert werden

Die Konfiguration erfolgt in `ngsw-config.json`. Angular's Build-Prozess generiert daraus automatisch den fertigen Service Worker (`ngsw-worker.js`).

Wichtig: Service Workers laufen **nur in Production-Builds** (`ng build`) und erfordern HTTPS (außer `localhost`).

## Aufgabe

Rüste eine bestehende Angular Standalone-App mit einem Service Worker aus und implementiere ein sauberes **Update-Handling**, das den Nutzer über neue Versionen informiert und einen Reload anbietet.

### Schritte

1. **PWA-Support hinzufügen**
   ```bash
   ng add @angular/pwa --project <dein-projekt>
   ```
   Das erstellt `ngsw-config.json`, aktualisiert `app.config.ts` und `index.html`.

2. **`ngsw-config.json` anpassen**: Füge eine `dataGroups`-Sektion für API-Calls hinzu:
   ```json
   {
     "dataGroups": [
       {
         "name": "api-cache",
         "urls": ["/api/**"],
         "cacheConfig": {
           "strategy": "freshness",
           "maxSize": 100,
           "maxAge": "1h",
           "timeout": "3s"
         }
       }
     ]
   }
   ```

3. **`UpdateService` erstellen**: Implementiere einen Injectable-Service, der:
   - Im Konstruktor mit `interval()` alle 6 Stunden auf Updates prüft
   - Auf `SwUpdate.versionUpdates` hört und bei `VERSION_READY` einen Reload anbietet
   - Eine `activateAndReload()`-Methode bereitstellt

4. **Update-Banner-Komponente**: Erstelle eine Standalone-Komponente, die:
   - Den `UpdateService` injiziert
   - Ein Signal `updateAvailable` hält
   - Bei verfügbarem Update ein Banner anzeigt mit einem "Jetzt aktualisieren"-Button

5. **Production-Build testen**:
   ```bash
   ng build
   npx http-server dist/<projekt-name>/browser -p 8080
   ```
   Öffne DevTools → Application → Service Workers und verifiziere die Registrierung.

## Hints

<details>
<summary>Hint 1 – SwUpdate im Service nutzen</summary>

```typescript
import { Injectable, inject } from '@angular/core';
import { SwUpdate, VersionReadyEvent } from '@angular/service-worker';
import { interval } from 'rxjs';
import { filter } from 'rxjs/operators';

@Injectable({ providedIn: 'root' })
export class UpdateService {
  private swUpdate = inject(SwUpdate);

  constructor() {
    if (!this.swUpdate.isEnabled) return;

    // Periodisch auf Updates prüfen
    interval(6 * 60 * 60 * 1000).subscribe(() => {
      this.swUpdate.checkForUpdate();
    });
  }

  readonly updateAvailable$ = this.swUpdate.versionUpdates.pipe(
    filter((event): event is VersionReadyEvent => event.type === 'VERSION_READY')
  );

  async activateAndReload(): Promise<void> {
    await this.swUpdate.activateUpdate();
    document.location.reload();
  }
}
```

</details>

<details>
<summary>Hint 2 – Update-Banner mit Signal</summary>

```typescript
import { Component, inject, signal, OnInit } from '@angular/core';
import { UpdateService } from './update.service';

@Component({
  selector: 'app-update-banner',
  standalone: true,
  template: `
    @if (updateAvailable()) {
      <div class="update-banner">
        <span>Eine neue Version ist verfügbar!</span>
        <button (click)="update()">Jetzt aktualisieren</button>
      </div>
    }
  `,
  styles: [`
    .update-banner {
      position: fixed;
      bottom: 0;
      width: 100%;
      background: #1976d2;
      color: white;
      padding: 12px;
      display: flex;
      justify-content: space-between;
      align-items: center;
      z-index: 9999;
    }
  `]
})
export class UpdateBannerComponent implements OnInit {
  private updateService = inject(UpdateService);
  updateAvailable = signal(false);

  ngOnInit(): void {
    this.updateService.updateAvailable$.subscribe(() => {
      this.updateAvailable.set(true);
    });
  }

  update(): void {
    this.updateService.activateAndReload();
  }
}
```

</details>

<details>
<summary>Hint 3 – SwPush für Web Push Notifications</summary>

```typescript
import { Injectable, inject } from '@angular/core';
import { SwPush } from '@angular/service-worker';

const VAPID_PUBLIC_KEY = 'DEIN_VAPID_PUBLIC_KEY';

@Injectable({ providedIn: 'root' })
export class PushNotificationService {
  private swPush = inject(SwPush);

  async subscribeToNotifications(): Promise<PushSubscription> {
    return this.swPush.requestSubscription({
      serverPublicKey: VAPID_PUBLIC_KEY
    });
  }

  // Push-Nachrichten empfangen
  readonly messages$ = this.swPush.messages;
}
```

</details>

## Beispiellösung

```typescript
// ngsw-config.json (Auszug)
{
  "$schema": "./node_modules/@angular/service-worker/config/schema.json",
  "index": "/index.html",
  "assetGroups": [
    {
      "name": "app",
      "installMode": "prefetch",
      "resources": {
        "files": ["/favicon.ico", "/index.html", "/manifest.webmanifest", "/*.css", "/*.js"]
      }
    },
    {
      "name": "assets",
      "installMode": "lazy",
      "updateMode": "prefetch",
      "resources": {
        "files": ["/assets/**", "/*.(svg|cur|jpg|jpeg|png|apng|webp|avif|gif|otf|ttf|woff|woff2)"]
      }
    }
  ],
  "dataGroups": [
    {
      "name": "api-freshness",
      "urls": ["/api/**"],
      "cacheConfig": {
        "strategy": "freshness",
        "maxSize": 100,
        "maxAge": "1h",
        "timeout": "3s"
      }
    }
  ]
}

// app.config.ts
import { ApplicationConfig } from '@angular/core';
import { provideServiceWorker } from '@angular/service-worker';
import { isDevMode } from '@angular/core';

export const appConfig: ApplicationConfig = {
  providers: [
    provideServiceWorker('ngsw-worker.js', {
      enabled: !isDevMode(),
      registrationStrategy: 'registerWhenStable:30000'
    })
  ]
};

// update.service.ts
import { Injectable, inject } from '@angular/core';
import { SwUpdate, VersionReadyEvent } from '@angular/service-worker';
import { interval, EMPTY } from 'rxjs';
import { filter, switchMap } from 'rxjs/operators';

@Injectable({ providedIn: 'root' })
export class UpdateService {
  private swUpdate = inject(SwUpdate);

  constructor() {
    if (!this.swUpdate.isEnabled) return;
    interval(6 * 60 * 60 * 1000).subscribe(() => this.swUpdate.checkForUpdate());
  }

  readonly updateAvailable$ = this.swUpdate.isEnabled
    ? this.swUpdate.versionUpdates.pipe(
        filter((e): e is VersionReadyEvent => e.type === 'VERSION_READY')
      )
    : EMPTY;

  async activateAndReload(): Promise<void> {
    await this.swUpdate.activateUpdate();
    document.location.reload();
  }
}

// update-banner.component.ts
import { Component, inject, signal, OnInit } from '@angular/core';
import { UpdateService } from './update.service';

@Component({
  selector: 'app-update-banner',
  standalone: true,
  template: `
    @if (updateAvailable()) {
      <div class="update-banner">
        <span>Neue Version verfügbar!</span>
        <button (click)="update()">Jetzt aktualisieren</button>
      </div>
    }
  `,
  styles: [`
    .update-banner {
      position: fixed; bottom: 0; left: 0; right: 0;
      background: #1976d2; color: white;
      padding: 12px 24px;
      display: flex; justify-content: space-between; align-items: center;
      z-index: 9999; box-shadow: 0 -2px 8px rgba(0,0,0,.2);
    }
    button { background: white; color: #1976d2; border: none;
             padding: 6px 16px; border-radius: 4px; cursor: pointer; font-weight: 600; }
  `]
})
export class UpdateBannerComponent implements OnInit {
  private updateService = inject(UpdateService);
  updateAvailable = signal(false);

  ngOnInit(): void {
    this.updateService.updateAvailable$.subscribe(() => this.updateAvailable.set(true));
  }

  update(): void { this.updateService.activateAndReload(); }
}
```

## Weiterführendes

- **Offizielle Docs**: [angular.dev/ecosystem/service-workers](https://angular.dev/ecosystem/service-workers) – vollständige API-Referenz für `SwUpdate`, `SwPush` und `ngsw-config.json`
- **Lighthouse Audit**: Führe in Chrome DevTools einen Lighthouse-PWA-Audit durch, um Installierbarkeit und Offline-Fähigkeit zu messen
- **Workbox**: Für komplexere Caching-Strategien jenseits von Angular's Service Worker lohnt sich ein Blick auf [Workbox](https://developer.chrome.com/docs/workbox/) als Low-Level-Alternative
- **Vertiefung**: Implementiere einen `unrecoverableState$`-Handler für den Fall, dass der Service Worker in einen inkonsistenten Zustand gerät (`SwUpdate.unrecoverable`)
