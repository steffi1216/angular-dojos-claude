# Angular Dojo: Module Federation & Micro-Frontends
**Datum:** 2026-06-29
**Dauer:** ~25 Minuten
**Level:** Fortgeschritten

## Lernziel
Du lernst, wie du mit Webpack Module Federation eine Angular-Anwendung in unabhängige Micro-Frontends aufteilst, die zur Laufzeit dynamisch geladen werden – ohne gemeinsame Build-Abhängigkeiten.

## Hintergrund & Theorie

**Module Federation** ist ein Webpack-5-Feature, das es ermöglicht, JavaScript-Module aus separaten Build-Artefakten zur Laufzeit zu laden und zu teilen. In Angular wird es über `@angular-architects/module-federation` genutzt.

**Kernkonzepte:**

- **Shell (Host):** Die Hauptanwendung, die andere Micro-Frontends einbindet.
- **Remote:** Eine eigenständige Angular-App, die Module (z. B. Routes, Components) exportiert.
- **Shared Libraries:** Abhängigkeiten wie `@angular/core` werden nur einmal geladen (Singleton-Strategie), um Konflikte und doppeltes Bundling zu vermeiden.
- **Dynamic Federation:** Statt statischer Konfiguration im Build können Remote-URLs zur Laufzeit aus einem Backend kommen – ideal für skalierbare Plattformen.

**Ablauf:**
1. Shell lädt `remoteEntry.js` des Remotes zur Laufzeit.
2. Webpack handelt Shared Modules ab.
3. Die Shell rendert die Remote-Komponente/Route, als wäre sie lokal.

Das Muster ermöglicht unabhängige Deployments verschiedener Teams und ist die Grundlage moderner Enterprise-Plattformen.

## Aufgabe

Du hast eine Shell-App (`shell`) und eine Remote-App (`mfe-products`). Konfiguriere Module Federation so, dass die Shell die `ProductsModule`-Route der Remote-App dynamisch lädt. Nutze dabei **Dynamic Federation** mit einer zur Laufzeit gelesenen Konfiguration.

### Schritte

1. **Remote konfigurieren:** Passe `webpack.config.js` der `mfe-products`-App an, sodass sie ihr `ProductsModule` als `remoteEntry` exponiert.

2. **Shell konfigurieren:** Konfiguriere die Shell-App so, dass sie den Remote zur Laufzeit via `loadRemoteModule` lädt – URLs kommen aus einer `app-config.json`.

3. **Routing verdrahten:** Füge in der Shell eine Lazy Route hinzu, die via `loadChildren` das Remote-Modul lädt.

4. **Shared Libraries absichern:** Stelle sicher, dass `@angular/core`, `@angular/common` und `@angular/router` als `singleton: true` und `strictVersion: false` geteilt werden.

5. **Dynamic Config laden:** Lade `app-config.json` in einem `APP_INITIALIZER` und speichere die Remote-URLs, bevor die App rendert.

## Hints

<details>
<summary>Hint 1 – Remote webpack.config.js</summary>

```javascript
// mfe-products/webpack.config.js
const { shareAll, withModuleFederationPlugin } = require('@angular-architects/module-federation/webpack');

module.exports = withModuleFederationPlugin({
  name: 'mfeProducts',
  exposes: {
    './ProductsModule': './src/app/products/products.module.ts',
  },
  shared: {
    ...shareAll({
      singleton: true,
      strictVersion: false,
      requiredVersion: 'auto',
    }),
  },
});
```

</details>

<details>
<summary>Hint 2 – Shell Lazy Route mit loadRemoteModule</summary>

```typescript
// shell/src/app/app.routes.ts
import { loadRemoteModule } from '@angular-architects/module-federation';

export const APP_ROUTES = [
  {
    path: 'products',
    loadChildren: () =>
      loadRemoteModule({
        type: 'module',
        remoteEntry: 'http://localhost:4201/remoteEntry.js',
        exposedModule: './ProductsModule',
      }).then((m) => m.ProductsModule),
  },
];
```

Für Dynamic Federation ersetzt du die hartcodierte URL durch eine aus dem `APP_INITIALIZER` geladene Konfiguration.

</details>

<details>
<summary>Hint 3 – APP_INITIALIZER für dynamische Config</summary>

```typescript
// shell/src/app/config.service.ts
import { Injectable } from '@angular/core';
import { HttpClient } from '@angular/common/http';
import { firstValueFrom } from 'rxjs';

export interface MfeConfig {
  mfeProducts: string;
}

@Injectable({ providedIn: 'root' })
export class ConfigService {
  config!: MfeConfig;

  constructor(private http: HttpClient) {}

  async load(): Promise<void> {
    this.config = await firstValueFrom(
      this.http.get<MfeConfig>('/assets/app-config.json')
    );
  }
}

// shell/src/app/app.config.ts
import { ApplicationConfig, APP_INITIALIZER } from '@angular/core';
import { ConfigService } from './config.service';

export function initConfig(cfg: ConfigService) {
  return () => cfg.load();
}

export const appConfig: ApplicationConfig = {
  providers: [
    {
      provide: APP_INITIALIZER,
      useFactory: initConfig,
      deps: [ConfigService],
      multi: true,
    },
  ],
};
```

```json
// shell/src/assets/app-config.json
{
  "mfeProducts": "http://localhost:4201/remoteEntry.js"
}
```

</details>

<details>
<summary>Hint 4 – Shell webpack.config.js</summary>

```javascript
// shell/webpack.config.js
const { shareAll, withModuleFederationPlugin } = require('@angular-architects/module-federation/webpack');

module.exports = withModuleFederationPlugin({
  remotes: {
    // statisch (für lokale Entwicklung):
    // mfeProducts: 'http://localhost:4201/remoteEntry.js',
  },
  shared: {
    ...shareAll({
      singleton: true,
      strictVersion: false,
      requiredVersion: 'auto',
    }),
  },
});
```

Bei Dynamic Federation bleibt `remotes` leer – die URL kommt zur Laufzeit von `loadRemoteModule`.

</details>

## Beispiellösung

```typescript
// mfe-products/src/app/products/products.module.ts
import { NgModule } from '@angular/core';
import { RouterModule } from '@angular/router';
import { CommonModule } from '@angular/common';
import { ProductsComponent } from './products.component';

@NgModule({
  declarations: [ProductsComponent],
  imports: [
    CommonModule,
    RouterModule.forChild([{ path: '', component: ProductsComponent }]),
  ],
})
export class ProductsModule {}

// shell/src/app/config.service.ts
import { Injectable } from '@angular/core';
import { HttpClient } from '@angular/common/http';
import { firstValueFrom } from 'rxjs';

@Injectable({ providedIn: 'root' })
export class ConfigService {
  remotes: Record<string, string> = {};

  constructor(private http: HttpClient) {}

  async load(): Promise<void> {
    this.remotes = await firstValueFrom(
      this.http.get<Record<string, string>>('/assets/app-config.json')
    );
  }
}

// shell/src/app/app.routes.ts
import { Routes } from '@angular/router';
import { loadRemoteModule } from '@angular-architects/module-federation';
import { inject } from '@angular/core';
import { ConfigService } from './config.service';

export const APP_ROUTES: Routes = [
  {
    path: 'products',
    loadChildren: () => {
      const config = inject(ConfigService);
      return loadRemoteModule({
        type: 'module',
        remoteEntry: config.remotes['mfeProducts'],
        exposedModule: './ProductsModule',
      }).then((m) => m.ProductsModule);
    },
  },
  { path: '', redirectTo: 'products', pathMatch: 'full' },
];

// shell/src/app/app.config.ts
import { ApplicationConfig, APP_INITIALIZER } from '@angular/core';
import { provideRouter } from '@angular/router';
import { provideHttpClient } from '@angular/common/http';
import { APP_ROUTES } from './app.routes';
import { ConfigService } from './config.service';

export const appConfig: ApplicationConfig = {
  providers: [
    provideRouter(APP_ROUTES),
    provideHttpClient(),
    {
      provide: APP_INITIALIZER,
      useFactory: (cfg: ConfigService) => () => cfg.load(),
      deps: [ConfigService],
      multi: true,
    },
  ],
};
```

```javascript
// mfe-products/webpack.config.js
const { shareAll, withModuleFederationPlugin } = require('@angular-architects/module-federation/webpack');

module.exports = withModuleFederationPlugin({
  name: 'mfeProducts',
  exposes: {
    './ProductsModule': './src/app/products/products.module.ts',
  },
  shared: {
    ...shareAll({ singleton: true, strictVersion: false, requiredVersion: 'auto' }),
  },
});

// shell/webpack.config.js
const { shareAll, withModuleFederationPlugin } = require('@angular-architects/module-federation/webpack');

module.exports = withModuleFederationPlugin({
  shared: {
    ...shareAll({ singleton: true, strictVersion: false, requiredVersion: 'auto' }),
  },
});
```

```json
// shell/src/assets/app-config.json
{
  "mfeProducts": "http://localhost:4201/remoteEntry.js"
}
```

## Weiterführendes

- **Native Federation:** `@angular-architects/native-federation` nutzt den Browser-nativen Import-Maps-Standard statt Webpack – ideal für Standalone-Apps und Angular ab v17+.
- **Nx Module Federation:** Das Nx-Monorepo-Tool bietet Generatoren für vollständig konfigurierte MFE-Workspaces: `nx g @nx/angular:host shell` und `nx g @nx/angular:remote mfe-products`.
- **Offizieller Guide:** [angular-architects.io/blog/module-federation](https://www.angulararchitects.io/blog/the-microfrontend-revolution-module-federation-in-webpack-5/)
- **Tipp:** Aktiviere in Produktionsumgebungen **Versionsprüfungen** (`strictVersion: true`) nur für kritische Libraries, um subtile Laufzeitfehler durch Versionskonflikte zu erkennen.
