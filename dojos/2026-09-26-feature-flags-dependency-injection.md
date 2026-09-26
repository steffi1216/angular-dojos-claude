# Angular Dojo: Feature Flags mit Dependency Injection
**Datum:** 2026-09-26
**Dauer:** ~25 Minuten
**Level:** Fortgeschritten

## Lernziel
Du lernst, wie du Feature Flags in Angular sauber über Dependency Injection implementierst – typsicher, testbar und ohne globale Variablen oder direkte Environment-Zugriffe.

## Hintergrund & Theorie

Feature Flags (auch Feature Toggles) ermöglichen es, Funktionen zur Laufzeit ein- oder auszuschalten, ohne neuen Code auszurollen. In Angular bietet sich eine DI-basierte Lösung an, weil sie:

- **Typsicherheit** garantiert (kein `any`-Casting),
- **einfach testbar** ist (Flags können pro Test überschrieben werden),
- **entkoppelt** ist von konkreten Implementierungen (HTTP, LocalStorage, Env-Files).

Das Kernmuster:

1. Ein `InjectionToken<FeatureFlags>` definiert die Form der Flags.
2. Eine Factory-Funktion (`useFactory`) lädt die Werte aus einer Quelle (z. B. Environment).
3. Services und Komponenten injizieren das Token, statt direkt auf `environment` zuzugreifen.
4. Guards und Direktiven nutzen das Token, um Zugang zu steuern.

Das Muster lässt sich später auf asynchrone Quellen (HTTP-API) erweitern, indem man einen `APP_INITIALIZER` oder `provideAppInitializer` einsetzt.

## Aufgabe

Implementiere ein vollständiges Feature-Flag-System für eine fiktive Anwendung mit zwei Flags:
- `betaDashboard` – aktiviert ein neues Dashboard
- `experimentalSearch` – aktiviert eine experimentelle Suchfunktion

### Schritte

1. **Definiere ein typsicheress Interface** `FeatureFlags` und erstelle ein `InjectionToken<FeatureFlags>`.

2. **Registriere den Provider** in `app.config.ts` mittels `useFactory`, der die Flags aus dem `environment`-Objekt liest.

3. **Erstelle eine `FeatureFlagService`-Klasse**, die das Token injiziert und eine Methode `isEnabled(flag: keyof FeatureFlags): boolean` anbietet.

4. **Erstelle eine strukturelle Direktive `*appIfFeature`**, die ein Template nur rendert, wenn das angegebene Flag aktiv ist.

5. **Schreibe einen Unit-Test** für den `FeatureFlagService`, bei dem du die Flags per `TestBed.overrideProvider` überschreibst.

## Hints

<details>
<summary>Hint 1 – InjectionToken und Interface</summary>

```typescript
// feature-flags.token.ts
export interface FeatureFlags {
  betaDashboard: boolean;
  experimentalSearch: boolean;
}

export const FEATURE_FLAGS = new InjectionToken<FeatureFlags>('FEATURE_FLAGS');
```

In `app.config.ts`:
```typescript
import { environment } from '../environments/environment';

export const appConfig: ApplicationConfig = {
  providers: [
    {
      provide: FEATURE_FLAGS,
      useFactory: () => environment.featureFlags,
    },
  ],
};
```

In `environment.ts`:
```typescript
export const environment = {
  production: false,
  featureFlags: {
    betaDashboard: true,
    experimentalSearch: false,
  },
};
```
</details>

<details>
<summary>Hint 2 – FeatureFlagService und *appIfFeature Direktive</summary>

```typescript
// feature-flag.service.ts
@Injectable({ providedIn: 'root' })
export class FeatureFlagService {
  private flags = inject(FEATURE_FLAGS);

  isEnabled(flag: keyof FeatureFlags): boolean {
    return this.flags[flag];
  }
}
```

Für die strukturelle Direktive:
```typescript
// if-feature.directive.ts
@Directive({ selector: '[appIfFeature]', standalone: true })
export class IfFeatureDirective {
  private templateRef = inject(TemplateRef);
  private vcr = inject(ViewContainerRef);
  private featureFlags = inject(FeatureFlagService);

  @Input() set appIfFeature(flag: keyof FeatureFlags) {
    this.vcr.clear();
    if (this.featureFlags.isEnabled(flag)) {
      this.vcr.createEmbeddedView(this.templateRef);
    }
  }
}
```
</details>

## Beispiellösung

```typescript
// feature-flags.token.ts
import { InjectionToken } from '@angular/core';

export interface FeatureFlags {
  betaDashboard: boolean;
  experimentalSearch: boolean;
}

export const FEATURE_FLAGS = new InjectionToken<FeatureFlags>('FEATURE_FLAGS');


// environments/environment.ts
export const environment = {
  production: false,
  featureFlags: {
    betaDashboard: true,
    experimentalSearch: false,
  } satisfies FeatureFlags,
};


// app.config.ts
import { ApplicationConfig } from '@angular/core';
import { FEATURE_FLAGS } from './feature-flags.token';
import { environment } from '../environments/environment';

export const appConfig: ApplicationConfig = {
  providers: [
    {
      provide: FEATURE_FLAGS,
      useFactory: () => environment.featureFlags,
    },
  ],
};


// feature-flag.service.ts
import { Injectable, inject } from '@angular/core';
import { FEATURE_FLAGS, FeatureFlags } from './feature-flags.token';

@Injectable({ providedIn: 'root' })
export class FeatureFlagService {
  private flags = inject(FEATURE_FLAGS);

  isEnabled(flag: keyof FeatureFlags): boolean {
    return this.flags[flag];
  }
}


// if-feature.directive.ts
import { Directive, Input, TemplateRef, ViewContainerRef, inject } from '@angular/core';
import { FeatureFlagService } from './feature-flag.service';
import { FeatureFlags } from './feature-flags.token';

@Directive({
  selector: '[appIfFeature]',
  standalone: true,
})
export class IfFeatureDirective {
  private templateRef = inject(TemplateRef);
  private vcr = inject(ViewContainerRef);
  private featureFlagService = inject(FeatureFlagService);

  @Input() set appIfFeature(flag: keyof FeatureFlags) {
    this.vcr.clear();
    if (this.featureFlagService.isEnabled(flag)) {
      this.vcr.createEmbeddedView(this.templateRef);
    }
  }
}


// Verwendung im Template
// app.component.html
// <app-beta-dashboard *appIfFeature="'betaDashboard'" />
// <app-experimental-search *appIfFeature="'experimentalSearch'" />


// feature-flag.service.spec.ts
import { TestBed } from '@angular/core/testing';
import { FEATURE_FLAGS } from './feature-flags.token';
import { FeatureFlagService } from './feature-flag.service';

describe('FeatureFlagService', () => {
  function setup(flags: Partial<FeatureFlags>) {
    TestBed.configureTestingModule({
      providers: [
        {
          provide: FEATURE_FLAGS,
          useValue: { betaDashboard: false, experimentalSearch: false, ...flags },
        },
      ],
    });
    return TestBed.inject(FeatureFlagService);
  }

  it('returns true when flag is enabled', () => {
    const service = setup({ betaDashboard: true });
    expect(service.isEnabled('betaDashboard')).toBe(true);
  });

  it('returns false when flag is disabled', () => {
    const service = setup({ betaDashboard: false });
    expect(service.isEnabled('betaDashboard')).toBe(false);
  });
});
```

## Weiterführendes

- **Asynchrone Flags via HTTP**: Nutze `provideAppInitializer` (Angular 19+) oder `APP_INITIALIZER`, um Flags beim App-Start von einer API zu laden und das Token mit dem Ergebnis zu befüllen – so kannst du Flags ohne Redeploy ändern.
- **Signals-Integration**: Verwandle die Flags in ein `Signal<FeatureFlags>`, um reaktive UI-Updates zu ermöglichen, wenn Flags sich zur Laufzeit ändern (z. B. nach Login).
- **Offizielle Docs zu `InjectionToken`**: https://angular.dev/api/core/InjectionToken
