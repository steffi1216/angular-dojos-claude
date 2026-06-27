# Angular Dojo: Custom Decorators
**Datum:** 2026-06-27
**Dauer:** ~25 Minuten
**Level:** Fortgeschritten

## Lernziel
Du lernst, eigene TypeScript-Dekoratoren für Angular-Klassen, -Methoden und -Properties zu schreiben, um Cross-Cutting Concerns wie Logging, Caching und Timing sauber zu kapseln.

## Hintergrund & Theorie
Dekoratoren sind ein TypeScript-Feature (Stage-3-Proposal), das Angular intern intensiv nutzt (`@Component`, `@Injectable`, `@Input` usw.). Ein Dekorator ist eine Funktion, die zur Laufzeit auf eine Klasse, Methode, Property oder einen Parameter angewendet wird.

Es gibt vier Arten:
- **Class Decorator**: erhält den Konstruktor der Klasse
- **Method Decorator**: erhält `target`, `propertyKey` und `PropertyDescriptor`
- **Property Decorator**: erhält `target` und `propertyKey`
- **Parameter Decorator**: erhält `target`, `propertyKey` und den Index

Typische Anwendungsfälle in Angular-Projekten:
- **`@Log()`** – loggt Methodenaufrufe automatisch
- **`@Memoize()`** – cached Rückgabewerte einer reinen Methode
- **`@Timed()`** – misst die Ausführungszeit
- **`@Confirm()`** – fragt vor einer Aktion nach Bestätigung

Wichtig: In Angular-Projekten muss `experimentalDecorators: true` in der `tsconfig.json` gesetzt sein (ist bei Angular CLI-Projekten standardmäßig der Fall).

## Aufgabe
Erstelle drei wiederverwendbare Dekoratoren und wende sie in einem Angular-Service an:

1. **`@LogMethod()`** – loggt Methodenname, Argumente und Rückgabewert
2. **`@Memoize()`** – cached das Ergebnis einer Methode basierend auf den Argumenten (als JSON-Key)
3. **`@Deprecated(message)`** – gibt eine `console.warn`-Warnung aus, wenn die Methode aufgerufen wird

Teste die Dekoratoren in einem `DataService`, der Daten transformiert.

### Schritte

1. Erstelle eine Datei `src/app/decorators/log-method.decorator.ts` und implementiere `@LogMethod()`.

2. Erstelle `src/app/decorators/memoize.decorator.ts` mit einem Cache-Mechanismus via `Map`.

3. Erstelle `src/app/decorators/deprecated.decorator.ts` mit einer konfigurierbaren Warnmeldung.

4. Erstelle einen `DataService` (`src/app/services/data.service.ts`) und dekoriere mindestens eine Methode mit jedem der drei Dekoratoren.

5. Rufe die Methoden in einer Komponente auf und prüfe die Konsolenausgaben. Verifiziere, dass `@Memoize()` bei gleichen Argumenten nur einmal berechnet wird (zweiter Aufruf liefert den Cache-Wert ohne erneute Berechnung).

## Hints

<details>
<summary>Hint 1 – Struktur eines Method Decorators</summary>

Ein Method Decorator erhält drei Argumente und muss den `descriptor` zurückgeben:

```typescript
export function LogMethod(): MethodDecorator {
  return (target: object, propertyKey: string | symbol, descriptor: PropertyDescriptor) => {
    const originalMethod = descriptor.value;
    descriptor.value = function (...args: unknown[]) {
      console.log(`Calling ${String(propertyKey)} with`, args);
      const result = originalMethod.apply(this, args);
      console.log(`${String(propertyKey)} returned`, result);
      return result;
    };
    return descriptor;
  };
}
```

</details>

<details>
<summary>Hint 2 – Cache-Schlüssel für @Memoize()</summary>

Nutze `JSON.stringify(args)` als Map-Key. Speichere den Cache direkt auf der Instanz, um ihn pro Instanz zu isolieren:

```typescript
export function Memoize(): MethodDecorator {
  return (_target, propertyKey, descriptor: PropertyDescriptor) => {
    const originalMethod = descriptor.value;
    const cacheKey = `__memoize_${String(propertyKey)}`;
    descriptor.value = function (...args: unknown[]) {
      if (!this[cacheKey]) {
        this[cacheKey] = new Map<string, unknown>();
      }
      const key = JSON.stringify(args);
      if (this[cacheKey].has(key)) {
        return this[cacheKey].get(key);
      }
      const result = originalMethod.apply(this, args);
      this[cacheKey].set(key, result);
      return result;
    };
    return descriptor;
  };
}
```

</details>

<details>
<summary>Hint 3 – @Deprecated mit Nachricht</summary>

```typescript
export function Deprecated(message: string): MethodDecorator {
  return (_target, propertyKey, descriptor: PropertyDescriptor) => {
    const originalMethod = descriptor.value;
    descriptor.value = function (...args: unknown[]) {
      console.warn(
        `[DEPRECATED] ${String(propertyKey)} ist veraltet: ${message}`
      );
      return originalMethod.apply(this, args);
    };
    return descriptor;
  };
}
```

</details>

## Beispiellösung

```typescript
// src/app/decorators/log-method.decorator.ts
export function LogMethod(): MethodDecorator {
  return (_target: object, propertyKey: string | symbol, descriptor: PropertyDescriptor) => {
    const originalMethod = descriptor.value;
    descriptor.value = function (...args: unknown[]) {
      console.log(`▶ ${String(propertyKey)}(`, ...args, ')');
      const result = originalMethod.apply(this, args);
      console.log(`◀ ${String(propertyKey)} →`, result);
      return result;
    };
    return descriptor;
  };
}

// src/app/decorators/memoize.decorator.ts
export function Memoize(): MethodDecorator {
  return (_target: object, propertyKey: string | symbol, descriptor: PropertyDescriptor) => {
    const originalMethod = descriptor.value;
    const cacheKey = `__memo_${String(propertyKey)}`;
    descriptor.value = function (...args: unknown[]) {
      const cache: Map<string, unknown> = (this[cacheKey] ??= new Map());
      const key = JSON.stringify(args);
      if (cache.has(key)) {
        console.log(`[Memoize] Cache-Hit für ${String(propertyKey)}`);
        return cache.get(key);
      }
      const result = originalMethod.apply(this, args);
      cache.set(key, result);
      return result;
    };
    return descriptor;
  };
}

// src/app/decorators/deprecated.decorator.ts
export function Deprecated(message: string): MethodDecorator {
  return (_target: object, propertyKey: string | symbol, descriptor: PropertyDescriptor) => {
    const originalMethod = descriptor.value;
    descriptor.value = function (...args: unknown[]) {
      console.warn(`[DEPRECATED] ${String(propertyKey)}: ${message}`);
      return originalMethod.apply(this, args);
    };
    return descriptor;
  };
}

// src/app/services/data.service.ts
import { Injectable } from '@angular/core';
import { LogMethod } from '../decorators/log-method.decorator';
import { Memoize } from '../decorators/memoize.decorator';
import { Deprecated } from '../decorators/deprecated.decorator';

@Injectable({ providedIn: 'root' })
export class DataService {

  @LogMethod()
  @Memoize()
  computeExpensiveValue(input: number): number {
    // Simuliert eine aufwändige Berechnung
    let result = 0;
    for (let i = 0; i < input * 1000; i++) {
      result += Math.sqrt(i);
    }
    return Math.round(result);
  }

  @LogMethod()
  transformData(items: string[]): string[] {
    return items.map(item => item.trim().toUpperCase());
  }

  @Deprecated('Verwende stattdessen transformData()')
  formatItems(items: string[]): string[] {
    return items.map(item => item.toUpperCase());
  }
}

// src/app/app.component.ts (Verwendung)
import { Component, inject, OnInit } from '@angular/core';
import { DataService } from './services/data.service';

@Component({
  selector: 'app-root',
  standalone: true,
  template: `
    <h1>Custom Decorators Demo</h1>
    <p>Ergebnis: {{ result }}</p>
    <p>Daten: {{ data | json }}</p>
  `
})
export class AppComponent implements OnInit {
  private dataService = inject(DataService);

  result = 0;
  data: string[] = [];

  ngOnInit(): void {
    // Erster Aufruf: berechnet den Wert
    this.result = this.dataService.computeExpensiveValue(500);
    // Zweiter Aufruf mit gleichen Args: Cache-Hit
    this.dataService.computeExpensiveValue(500);

    this.data = this.dataService.transformData(['  hello ', ' world ']);

    // Löst Deprecation-Warnung aus
    this.dataService.formatItems(['test']);
  }
}
```

## Weiterführendes
- **Decorator Stacking**: Dekoratoren werden von unten nach oben ausgeführt – bei `@LogMethod() @Memoize()` greift zuerst `Memoize`, dann `Log`. Teste dieses Verhalten bewusst.
- **Reflect Metadata**: Mit dem Paket `reflect-metadata` und `Reflect.defineMetadata` / `Reflect.getMetadata` lassen sich Dekoratoren noch mächtiger gestalten – Angular's DI-System basiert genau darauf.
- **Decorator Factories mit Konfiguration**: Erweitere `@LogMethod()` um ein Options-Objekt `{ logArgs?: boolean; logResult?: boolean; prefix?: string }`.
- **Angular 18+ Native Decorators**: Verfolge das TC39 Stage-3-Proposal für native JS-Dekoratoren, da Angular schrittweise auf den Standard migriert.
