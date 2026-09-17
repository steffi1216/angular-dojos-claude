# Angular Dojo: Meta & Title Service – SEO in Angular Apps
**Datum:** 2026-09-17
**Dauer:** ~25 Minuten
**Level:** Fortgeschritten

## Lernziel
Du lernst, wie du mit Angulars `Title`- und `Meta`-Service dynamische, routenbasierte SEO-Metadaten (Page Title, Description, Open Graph, Twitter Cards) implementierst – sauber, reactive und SSR-kompatibel.

## Hintergrund & Theorie

Angular bietet zwei eingebaute Services für SEO-relevante HTML-`<head>`-Manipulation:

- **`Title`** (`@angular/platform-browser`): Setzt den `<title>`-Tag des Dokuments.
- **`Meta`** (`@angular/platform-browser`): Erlaubt das Hinzufügen, Aktualisieren und Entfernen von `<meta>`-Tags (z. B. `description`, `og:title`, `twitter:card`).

Beide Services sind SSR-kompatibel: Sie funktionieren auf dem Server (Angular Universal / SSR) genauso wie im Browser, was für Search-Engine-Indexierung entscheidend ist.

Das Muster ist meist dasselbe: Man hört auf `NavigationEnd`-Events des Routers, liest Metadaten aus dem aktivierten Route-Snapshot (über `data`) und aktualisiert `<head>` entsprechend. Mit dem neueren `RouterInputBindings`-Feature und Signals lässt sich das noch eleganter lösen.

**Typischer Ansatz:**
1. Route-`data` enthält `{ title, description, ogImage }` pro Route.
2. Ein Root-Service/Effect hört auf `NavigationEnd` und liest rekursiv den letzten aktivierten Child-Snapshot.
3. `Title.setTitle()` und `Meta.updateTag()` werden mit den Daten befüllt.

**Wichtig für SSR:** Da der Server keinen echten DOM hat, arbeiten die Services über `DOCUMENT`-Injection intern, und Angular regelt das automatisch – du musst nichts extra tun.

## Aufgabe

Erstelle einen `SeoService`, der automatisch bei jeder Navigation die Seiten-Metadaten aktualisiert. Verdrahte ihn mit der App-Konfiguration und demonstriere das Verhalten mit zwei verschiedenen Routen.

### Schritte

1. **`SeoService` erstellen**: Injiziere `Router`, `Title` und `Meta`. Subscribiere auf `NavigationEnd`-Events und extrahiere die `data` des tiefsten aktivierten Child-Routes.

2. **Route-Daten definieren**: Ergänze die Route-Definitionen mit einem `data`-Objekt vom Typ `SeoData`:
   ```typescript
   interface SeoData {
     title: string;
     description?: string;
     ogImage?: string;
     noIndex?: boolean;
   }
   ```

3. **Meta-Tags setzen**: Im `SeoService` nach jedem `NavigationEnd`:
   - `Title.setTitle()` mit Suffix (z. B. `"Seitentitel | MeineApp"`)
   - `meta.updateTag({ name: 'description', content: '...' })`
   - `meta.updateTag({ property: 'og:title', content: '...' })`
   - `meta.updateTag({ property: 'og:description', content: '...' })`
   - `meta.updateTag({ property: 'og:image', content: '...' })` (falls vorhanden)
   - `noIndex`-Route: `meta.updateTag({ name: 'robots', content: 'noindex, nofollow' })` bzw. entfernen wenn nicht gesetzt.

4. **Service initialisieren**: Den `SeoService` im Root-Injector starten (via `APP_INITIALIZER` oder indem er in `app.component.ts` injiziert wird). Alternativ: `inject(SeoService)` direkt in `app.config.ts` als Side-Effect.

5. **Testen**: Prüfe mit den Browser DevTools (`<head>`-Element inspizieren), dass sich Titel und Meta-Tags beim Navigieren zwischen Routen korrekt ändern.

## Hints

<details>
<summary>Hint 1 – NavigationEnd filtern und Route-Data extrahieren</summary>

```typescript
import { Router, NavigationEnd, ActivatedRoute } from '@angular/router';
import { filter } from 'rxjs/operators';

// Im Service-Konstruktor:
this.router.events.pipe(
  filter(event => event instanceof NavigationEnd)
).subscribe(() => {
  let route = this.activatedRoute.root;
  while (route.firstChild) {
    route = route.firstChild;
  }
  const data = route.snapshot.data as SeoData;
  this.updateSeo(data);
});
```
</details>

<details>
<summary>Hint 2 – Service im Root starten ohne APP_INITIALIZER</summary>

In `app.config.ts` kannst du den Service ohne `APP_INITIALIZER` starten, indem du `inject` in einer Factory verwendest:

```typescript
// app.config.ts
export const appConfig: ApplicationConfig = {
  providers: [
    provideRouter(routes),
    {
      provide: ENVIRONMENT_INITIALIZER,
      useValue: () => inject(SeoService).init(),
      multi: true,
    },
  ],
};
```

Oder einfacher: den `SeoService` in `AppComponent` injizieren – das reicht für die meisten Apps.
</details>

<details>
<summary>Hint 3 – Meta-Tags mit updateTag vs. addTag</summary>

`Meta.updateTag()` überschreibt einen bestehenden Tag oder erstellt ihn neu (idempotent). `addTag()` würde hingegen einen Duplicate erzeugen. Verwende immer `updateTag` für dynamische Werte.

Für `og:`-Properties nutze den `property`-Selektor, nicht `name`:
```typescript
this.meta.updateTag({ property: 'og:title', content: title });
// Entfernen:
this.meta.removeTag("name='robots'");
```
</details>

## Beispiellösung

```typescript
// seo.service.ts
import { Injectable, inject } from '@angular/core';
import { Router, NavigationEnd, ActivatedRoute } from '@angular/router';
import { Title, Meta } from '@angular/platform-browser';
import { filter } from 'rxjs/operators';

export interface SeoData {
  title?: string;
  description?: string;
  ogImage?: string;
  noIndex?: boolean;
}

const APP_NAME = 'MeineApp';

@Injectable({ providedIn: 'root' })
export class SeoService {
  private router = inject(Router);
  private activatedRoute = inject(ActivatedRoute);
  private titleService = inject(Title);
  private meta = inject(Meta);

  init(): void {
    this.router.events.pipe(
      filter(e => e instanceof NavigationEnd)
    ).subscribe(() => {
      let route = this.activatedRoute.root;
      while (route.firstChild) {
        route = route.firstChild;
      }
      this.applyMetadata(route.snapshot.data as SeoData);
    });
  }

  private applyMetadata(data: SeoData): void {
    const title = data.title ? `${data.title} | ${APP_NAME}` : APP_NAME;
    const description = data.description ?? '';
    const ogImage = data.ogImage ?? '';

    this.titleService.setTitle(title);

    this.meta.updateTag({ name: 'description', content: description });
    this.meta.updateTag({ property: 'og:title', content: title });
    this.meta.updateTag({ property: 'og:description', content: description });

    if (ogImage) {
      this.meta.updateTag({ property: 'og:image', content: ogImage });
    } else {
      this.meta.removeTag("property='og:image'");
    }

    if (data.noIndex) {
      this.meta.updateTag({ name: 'robots', content: 'noindex, nofollow' });
    } else {
      this.meta.removeTag("name='robots'");
    }
  }
}

// routes.ts
export const routes: Routes = [
  {
    path: '',
    component: HomeComponent,
    data: {
      title: 'Startseite',
      description: 'Willkommen auf MeineApp – die beste Plattform für Angular-Entwickler.',
      ogImage: 'https://example.com/og-home.png',
    } satisfies SeoData,
  },
  {
    path: 'ueber-uns',
    component: AboutComponent,
    data: {
      title: 'Über uns',
      description: 'Erfahre mehr über unser Team und unsere Mission.',
    } satisfies SeoData,
  },
  {
    path: 'intern',
    component: InternalComponent,
    data: {
      title: 'Interner Bereich',
      noIndex: true,
    } satisfies SeoData,
  },
];

// app.component.ts
@Component({ selector: 'app-root', template: '<router-outlet />' })
export class AppComponent {
  // SeoService hier injizieren startet den Listener
  private seoService = inject(SeoService);

  constructor() {
    this.seoService.init();
  }
}
```

## Weiterführendes

- **Canonical URLs**: Ergänze den `SeoService` um `<link rel="canonical">` via `DOCUMENT`-Injection und `document.head.querySelector('link[rel="canonical"]')` – wichtig bei paginierten Seiten oder mehreren URLs für denselben Inhalt.
- **Structured Data (JSON-LD)**: Injiziere ein `<script type="application/ld+json">`-Element dynamisch für Rich Results in Google Search.
- **Angular SSR**: Kombiniere mit `TransferState` (Dojo 2026-09-11), um SSR-gerenderte Meta-Tags im Browser nicht doppelt zu rendern.
- **Offizielle Docs**: [angular.dev – Title service](https://angular.dev/api/platform-browser/Title), [Meta service](https://angular.dev/api/platform-browser/Meta)
