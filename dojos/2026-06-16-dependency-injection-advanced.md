# Angular Dojo: Dependency Injection (Advanced)
**Datum:** 2026-06-16
**Dauer:** ~25 Minuten
**Level:** Fortgeschritten

## Lernziel
Du lernst, wie du mit `InjectionToken`, `useFactory` und Multi-Providern flexible, austauschbare Abhängigkeiten gestaltest – nützlich für Konfiguration, Plugin-Systeme und Testbarkeit.

## Hintergrund & Theorie
Angulars DI-System geht weit über `@Injectable()` und Klassen-Tokens hinaus. Wenn du primitive Werte (Strings, Objekte, Funktionen) injizieren willst, brauchst du ein **`InjectionToken`**, da TypeScript-Interfaces zur Laufzeit nicht existieren:

```typescript
export interface AppConfig { apiUrl: string; }
export const APP_CONFIG = new InjectionToken<AppConfig>('app.config');
```

Mit **`useFactory`** kannst du den Wert eines Providers zur Laufzeit berechnen – z. B. abhängig von anderen injizierten Services:

```typescript
{
  provide: APP_CONFIG,
  useFactory: (env: EnvironmentService) => ({ apiUrl: env.isProd ? '/api' : '/api-dev' }),
  deps: [EnvironmentService],
}
```

Mit **`multi: true`** registrierst du mehrere Provider für denselben Token – Angular injiziert dann ein **Array** aller Werte. Das ist die Grundlage für `HTTP_INTERCEPTORS` und eignet sich für Plugin-/Validator-Listen, die aus verschiedenen Modulen befüllt werden, ohne dass diese sich gegenseitig kennen müssen.

## Aufgabe
Baue ein kleines Plugin-System für eine "Greeting"-Komponente: Mehrere unabhängige Provider sollen Begrüßungstexte beisteuern, die per Multi-Provider gesammelt und angezeigt werden – plus eine konfigurierbare App-Einstellung per `useFactory`.

### Schritte
1. Erstelle ein `InjectionToken<string[]>` namens `GREETING_PROVIDERS` (multi).
2. Registriere mindestens zwei separate Provider für diesen Token (z. B. `useValue: 'Hallo!'` und `useFactory`, der den Wochentag einbaut).
3. Erstelle einen zweiten `InjectionToken<AppConfig>` namens `APP_CONFIG`, dessen Wert per `useFactory` aus einem injizierten `EnvironmentService` berechnet wird.
4. Injiziere beide Tokens in eine Standalone-Komponente und rendere alle Begrüßungen sowie die `apiUrl` aus der Config im Template.
5. Teste das Setup: Füge in einer zweiten Provider-Registrierung (z. B. in einer "Feature"-Datei) einen dritten Greeting-Provider hinzu, ohne die bestehenden Provider zu verändern.

## Hints
<details>
<summary>Hint 1</summary>

Multi-Provider werden so registriert – Angular sammelt automatisch alle Einträge für denselben Token in ein Array, unabhängig davon, wo sie deklariert sind:

```typescript
{ provide: GREETING_PROVIDERS, useValue: 'Hallo!', multi: true }
```
</details>
<details>
<summary>Hint 2</summary>

Bei `useFactory` mit Abhängigkeiten brauchst du immer `deps`, da Angular sonst nicht weiß, was es in die Factory-Funktion injizieren soll:

```typescript
{
  provide: GREETING_PROVIDERS,
  useFactory: () => `Heute ist ${new Date().toLocaleDateString('de-DE', { weekday: 'long' })}!`,
  multi: true,
}
```
</details>

## Beispiellösung
```typescript
import { Component, InjectionToken, inject } from '@angular/core';
import { bootstrapApplication } from '@angular/platform-browser';

// --- Tokens ---
interface AppConfig {
  apiUrl: string;
}

export const GREETING_PROVIDERS = new InjectionToken<string[]>('greeting.providers');
export const APP_CONFIG = new InjectionToken<AppConfig>('app.config');

// --- Service für die Factory ---
class EnvironmentService {
  isProd = false;
}

// --- Komponente ---
@Component({
  selector: 'app-greeting',
  standalone: true,
  template: `
    <h2>Begrüßungen:</h2>
    <ul>
      <li *ngFor="let greeting of greetings">{{ greeting }}</li>
    </ul>
    <p>API URL: {{ config.apiUrl }}</p>
  `,
})
export class GreetingComponent {
  greetings = inject(GREETING_PROVIDERS);
  config = inject(APP_CONFIG);
}

// --- Provider-Registrierung (z. B. in app.config.ts) ---
bootstrapApplication(GreetingComponent, {
  providers: [
    EnvironmentService,
    { provide: GREETING_PROVIDERS, useValue: 'Hallo!', multi: true },
    {
      provide: GREETING_PROVIDERS,
      useFactory: () =>
        `Heute ist ${new Date().toLocaleDateString('de-DE', { weekday: 'long' })}!`,
      multi: true,
    },
    // dritter Provider, z. B. aus einem Feature-Modul, unabhängig hinzugefügt:
    { provide: GREETING_PROVIDERS, useValue: 'Willkommen im Dojo!', multi: true },
    {
      provide: APP_CONFIG,
      useFactory: (env: EnvironmentService) => ({
        apiUrl: env.isProd ? '/api' : '/api-dev',
      }),
      deps: [EnvironmentService],
    },
  ],
});
```

## Weiterführendes
- Schau dir an, wie Angular selbst `HTTP_INTERCEPTORS` als Multi-Provider-Token nutzt – das ist exakt dasselbe Pattern wie in dieser Aufgabe.
- Offizielle Doku: [Dependency injection in Angular](https://angular.dev/guide/di) und [`InjectionToken` API](https://angular.dev/api/core/InjectionToken)
