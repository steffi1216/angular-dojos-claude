# Angular Dojo: Application Builder (esbuild & Vite)
**Datum:** 2026-10-23
**Dauer:** ~25 Minuten
**Level:** Fortgeschritten

## Lernziel
Du lernst, wie du eine Angular-App vom alten Webpack-basierten `browser`-Builder auf den modernen `application`-Builder (esbuild + Vite Dev Server) migrierst, die wichtigsten Konfigurationsunterschiede verstehst und die Performance-Vorteile im Alltag nutzt.

## Hintergrund & Theorie
Ab Angular 17 ist `@angular-devkit/build-angular:application` der Standard-Builder für neue Projekte. Er ersetzt den alten `browser`-Builder und bringt zwei wesentliche Verbesserungen:

**esbuild als Bundler:** esbuild ist 10–100× schneller als Webpack. Production-Builds sind deutlich kürzer, weil esbuild alle Transformationen (TypeScript, SCSS, Tree-Shaking) nativ in Go implementiert.

**Vite für den Dev Server:** Das HMR (Hot Module Replacement) im Development-Modus nutzt den Vite-Dev-Server. Statt ein vollständiges Bundle zu erzeugen, liefert Vite Module on-demand als native ES Modules aus — das führt zu nahezu sofortigen Reload-Zeiten.

**Weitere Features des `application`-Builders:**
- SSR und SSG direkt konfigurierbar (`server`, `prerender`)
- `outputMode: 'server'` | `'static'` | `'client'`
- `budgets` wie gewohnt
- `define` für globale Konstanten (ähnlich webpack `DefinePlugin`)
- `externalDependencies` für CDN-Dependencies
- Polyfills als Array statt `polyfills.ts`

Die Migration ist für die meisten Projekte unkompliziert; Sonderfälle entstehen bei Custom Webpack-Configs (`extra-webpack.config.js`) oder speziellen Asset-Setups.

## Aufgabe
Du hast eine bestehende Angular-App, die noch den alten `browser`-Builder verwendet. Migriere sie auf `application` und konfiguriere folgende fortgeschrittene Features:

1. **Migration:** Wechsle in `angular.json` von `browser` auf `application` und passe die Konfiguration an.
2. **`define`-Konstanten:** Injiziere zur Build-Zeit eine globale Konstante `APP_VERSION` (aus `package.json`), die im Code ohne Import verfügbar ist.
3. **`externalDependencies`:** Konfiguriere `lodash-es` als externe Dependency, die per CDN geladen wird, und füge die entsprechende `importmap` in `index.html` ein.
4. **Bundle-Analyse:** Nutze `--stats-json` und `webpack-bundle-analyzer` (bzw. `esbuild-bundle-analyzer`) zur Analyse des Outputs.

### Schritte
1. Öffne `angular.json`. Suche den `build`-Target deines Projekts. Ändere `"builder"` von `@angular-devkit/build-angular:browser` auf `@angular-devkit/build-angular:application`.
2. Passe die Optionen an: Der `application`-Builder kennt kein `main`, sondern `browser`. Benenne `"main": "src/main.ts"` in `"browser": "src/main.ts"` um. Entferne die `"vendorChunk"`- und `"buildOptimizer"`-Optionen (nicht mehr nötig). Polyfills werden als Array angegeben: `"polyfills": ["zone.js"]`.
3. Füge `"define"` hinzu:
   ```json
   "define": {
     "APP_VERSION": "JSON.stringify(require('./package.json').version)"
   }
   ```
   Deklariere den Typ in `src/typings.d.ts`:
   ```typescript
   declare const APP_VERSION: string;
   ```
   Verwende ihn in einer Komponente: `console.log('Version:', APP_VERSION);`
4. Füge `"externalDependencies": ["lodash-es"]` zu den Build-Optionen hinzu. Ergänze in `src/index.html` eine `<script type="importmap">` vor dem closing `</head>`:
   ```html
   <script type="importmap">
     {
       "imports": {
         "lodash-es": "https://cdn.jsdelivr.net/npm/lodash-es@4.17.21/lodash.js"
       }
     }
   </script>
   ```
5. Führe `ng build --stats-json` aus und analysiere `dist/stats.json` mit einem Analyzer deiner Wahl.

## Hints
<details>
<summary>Hint 1: Automatische Migration per Schematic</summary>

Angular bietet eine offizielle Schematic für die Migration:
```bash
ng update @angular/cli
# oder manuell:
ng generate @angular/cli:app-builder-migration
```
Diese erledigt die meisten `angular.json`-Änderungen automatisch. Custom Webpack Configs werden als Kommentar markiert, da sie nicht übertragbar sind.
</details>

<details>
<summary>Hint 2: SSR mit dem application-Builder aktivieren</summary>

Im `application`-Builder wird SSR direkt in denselben Build-Target integriert — kein separater `server`-Target mehr:
```json
"options": {
  "browser": "src/main.ts",
  "server": "src/main.server.ts",
  "prerender": true,
  "ssr": {
    "entry": "server.ts"
  }
}
```
`prerender: true` führt SSG für alle bekannten Routen aus. Einzelne Routen können per `"routesFile"` angegeben werden.
</details>

<details>
<summary>Hint 3: TypeScript-Fehler durch fehlende Typdeklaration</summary>

Wenn TypeScript `APP_VERSION` nicht kennt, obwohl `define` konfiguriert ist, prüfe:
1. Ist `src/typings.d.ts` in `tsconfig.app.json` unter `"include"` aufgeführt (oder liegt es im `rootDir`)?
2. Ist die `declare const`-Syntax korrekt (kein `export`)?
3. Nach Änderungen an `tsconfig` muss der Dev Server neugestartet werden.
</details>

## Beispiellösung

**`angular.json` (Ausschnitt):**
```json
{
  "architect": {
    "build": {
      "builder": "@angular-devkit/build-angular:application",
      "options": {
        "outputPath": "dist/my-app",
        "index": "src/index.html",
        "browser": "src/main.ts",
        "polyfills": ["zone.js"],
        "tsConfig": "tsconfig.app.json",
        "assets": ["src/favicon.ico", "src/assets"],
        "styles": ["src/styles.scss"],
        "scripts": [],
        "define": {
          "APP_VERSION": "JSON.stringify(require('./package.json').version)"
        },
        "externalDependencies": ["lodash-es"]
      },
      "configurations": {
        "production": {
          "budgets": [
            { "type": "initial", "maximumWarning": "500kB", "maximumError": "1MB" }
          ],
          "outputHashing": "all"
        },
        "development": {
          "optimization": false,
          "extractLicenses": false,
          "sourceMap": true
        }
      },
      "defaultConfiguration": "production"
    }
  }
}
```

**`src/typings.d.ts`:**
```typescript
declare const APP_VERSION: string;
```

**`src/app/app.component.ts`:**
```typescript
import { Component, OnInit } from '@angular/core';

@Component({
  selector: 'app-root',
  standalone: true,
  template: `<h1>App v{{ version }}</h1>`,
})
export class AppComponent implements OnInit {
  version = APP_VERSION; // zur Build-Zeit eingebettet

  ngOnInit() {
    console.log('Build version:', APP_VERSION);
  }
}
```

**`src/index.html` (Importmap für externes lodash-es):**
```html
<!doctype html>
<html lang="en">
  <head>
    <meta charset="utf-8" />
    <title>My App</title>
    <base href="/" />
    <meta name="viewport" content="width=device-width, initial-scale=1" />
    <script type="importmap">
      {
        "imports": {
          "lodash-es": "https://cdn.jsdelivr.net/npm/lodash-es@4.17.21/lodash.js"
        }
      }
    </script>
  </head>
  <body>
    <app-root></app-root>
  </body>
</html>
```

**Bundle-Analyse:**
```bash
# esbuild-Metafile erzeugen (Angular CLI schreibt stats.json)
ng build --stats-json

# Mit esbuild Bundle Analyzer analysieren
npx esbuild-bundle-analyzer dist/my-app/browser/stats.json

# Alternativ: source-map-explorer
npx source-map-explorer dist/my-app/browser/*.js
```

## Weiterführendes
- [Angular Docs: Application Builder](https://angular.dev/tools/cli/build) — offizielle Dokumentation aller Optionen des `application`-Builders inkl. SSR-Konfiguration
- Experimentiere mit `outputMode: 'server'` für eine vollständige SSR-Anwendung ohne separaten Buildschritt und vergleiche die Bundle-Größen zwischen `browser`- und `application`-Builder an einem realen Projekt.
